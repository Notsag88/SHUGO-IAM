# UC-08: Self-Service Profile - Benchmarks, SLAs y Estrategia de Rendimiento

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-08 - Self-Service Profile (User-Initiated Profile Management) |
| **Documento:** | 12_UC_Specific_Performance_Benchmarks.md |
| **Metodología:** | Performance Engineering + SRE (SLI/SLO/SLA) + Load Testing (k6) + Capacity Planning + Async Processing |
| **Estándares:** | ISO/IEC 25010 (Performance Efficiency), Google SRE Workbook, OWASP Performance Project |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 09/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | [09/09/2026] | [Gastón Ruiz] | **Creación inicial.** Definición de SLAs, SLOs, estrategia de Load Testing (enfocada en actualizaciones de perfil, upload de avatares y detección de No-Op), y planificación de capacidad para el endpoint de gestión self-service. | Documento de referencia para SREs y Arquitectos de Rendimiento. Establece cómo mantener la latencia sub-100ms para actualizaciones de texto, manejar la carga CPU del procesamiento de imágenes, y garantizar la integridad de los logs de auditoría bajo alta concurrencia. |

---

## 1. Introducción y Propósito

A diferencia de los UCs de lectura (UC-04) o los UCs administrativos de baja frecuencia, el **UC-08 (Self-Service Profile)** es una operación de **escritura de alta frecuencia, impulsada por el usuario final**, con características de rendimiento únicas:

1. **Operaciones Mixtas (CPU vs I/O):** La actualización de texto (display_name, preferences) es I/O bound (DB), pero el upload y procesamiento de avatares es intensivo en CPU (redimensionado, re-encoding).
2. **Detección de No-Op:** Una optimización crítica que debe medir su efectacidad para evitar writes innecesarios en la base de datos.
3. **Auditoría Granular Write-Heavy:** Cada cambio de campo genera una fila en `profile_audit_logs`. Bajo picos de uso, esto puede saturar el almacenamiento si no se particiona correctamente.
4. **Validación de Seguridad (RLS & JWT):** La prevención de IDOR mediante Row-Level Security (RLS) en PostgreSQL añade un overhead mínimo pero medible que debe ser monitoreado.

Este documento aborda estos desafíos definiendo SLAs realistas, estrategias de optimización (como el procesamiento asíncrono de imágenes) y pruebas de carga específicas.

---

## 2. Service Level Indicators (SLIs) y Objectives (SLOs)

### 2.1. SLIs Específicos para UC-08

| SLI | Descripción | Fórmula / Medición |
| :--- | :--- | :--- |
| **Profile Update Latency (p95)** | Tiempo para actualizar campos de texto (sin avatar). | `histogram_quantile(0.95, rate(kagami_profile_update_latency_bucket[5m]))` |
| **Avatar Upload & Process Latency (p95)** | Tiempo total desde que el usuario sube la imagen hasta que recibe la URL. | `histogram_quantile(0.95, rate(kagami_avatar_process_latency_bucket[5m]))` |
| **No-Op Detection Rate** | Porcentaje de requests que no resultan en cambios en DB. | `rate(kagami_profile_no_op_total[5m]) / rate(kagami_profile_update_attempts_total[5m])` |
| **Audit Log Insert Latency (p95)** | Tiempo para persistir los logs granulares de cambios. | `histogram_quantile(0.95, rate(kagami_audit_log_insert_latency_bucket[5m]))` |
| **IDOR Attempt Rate** | Frecuencia de intentos de acceso a perfiles ajenos bloqueados. | `rate(kagami_security_idor_blocked_total[5m])` |
| **Profile Cache Hit Ratio** | Efectividad de la caché Redis para lecturas de perfil (UC-04). | `rate(kagami_profile_cache_hits_total[5m]) / rate(kagami_profile_cache_requests_total[5m])` |

### 2.2. SLOs y SLAs

| Métrica | SLO (Objetivo Interno - 95%) | SLA (Contrato Enterprise - 99%) | Justificación |
| :--- | :--- | :--- | :--- |
| **Latencia Actualización Texto (p95)** | **< 100 ms** | **< 200 ms** | Operación simple de UPDATE + Audit INSERT. Debe ser casi instantánea. |
| **Latencia Procesamiento Avatar (p95)** | **< 1.500 ms** | **< 3.000 ms** | Incluye upload a S3, redimensionado (CPU) y invalidación de CDN. |
| **Latencia Validación Ownership (RLS)** | **< 5 ms** | **< 10 ms** | Overhead de RLS en PostgreSQL debe ser mínimo gracias a índices en `identity_id`. |
| **Throughput Actualizaciones** | **500 req/seg** | **300 req/seg** | Suficiente para picos donde muchos usuarios actualizan preferencias simultáneamente. |
| **Throughput Upload Avatares** | **50 req/seg** | **30 req/seg** | Limitado por CPU de procesamiento de imágenes y ancho de banda de S3. |
| **Disponibilidad** | **99.99%** | **99.9%** | Crítico para la UX. Si el perfil no carga o no guarda, el usuario se frustra. |
| **Tasa de No-Op** | **> 20%** | **N/A** | Indicador de buena UX (el sistema no guarda basura) o de clientes ruidosos. |

---

## 3. Cuellos de Botella y Estrategias de Optimización

### 3.1. Cuello de Botella #1: Procesamiento de Imágenes (CPU Bound)

**Problema:** Redimensionar y re-encodear imágenes (para mitigar T-48 Malicious Avatar) consume mucha CPU. Hacerlo síncronamente en el request del usuario bloquea el thread y aumenta la latencia.

**Estrategia de Optimización: Patrón de Procesamiento Asíncrono**

1.  **Upload Directo a S3 (Presigned URL):** El cliente sube la imagen directamente a un bucket temporal de S3 usando una URL firmada, liberando al servidor de la carga de red.
2.  **Trigger Serverless:** La subida a S3 dispara una función AWS Lambda (o Azure Function) que realiza el re-encoding, redimensionado y validación de Magic Bytes.
3.  **Notificación vía Evento:** La Lambda guarda la imagen final en el bucket público y publica un evento `Avatar.Processed` en Kafka/SQS.
4.  **Actualización de Perfil:** El `ProfileHandler` consume el evento y actualiza la URL en la tabla `profiles`.

*Nota: Para mantener la simplicidad en la Fase 1, se puede hacer síncrono usando librerías optimizadas como `ImageSharp` con límites estrictos de concurrencia (SemaphoreSlim).*

### 3.2. Cuello de Botella #2: Auditoría Granular (Write-Heavy)

**Problema:** Si un usuario actualiza 5 campos, generamos 5 INSERTs en `profile_audit_logs`. Esto multiplica la carga de escritura en la base de datos.

**Estrategia de Optimización: Batch Insert & Partitioning**

1.  **Particionamiento por Tiempo:** La tabla `profile_audit_logs` está particionada por mes (como se definió en el Doc 10). Esto mantiene los índices pequeños y las inserciones rápidas.
2.  **Batching en Aplicación:** Si se detectan múltiples cambios, el `AuditLogger` los agrupa en una sola operación de `INSERT` múltiple o usa `SqlBulkCopy` / `COPY` de Postgres si el volumen es masivo.
3.  **Outbox Pattern:** Los logs de auditoría se insertan en la misma transacción que el perfil, pero la replicación a sistemas de BI/Compliance se hace asíncronamente vía Debezium o Kafka Connect.

### 3.3. Cuello de Botella #3: Parsing de JSON en Preferencias

**Problema:** Validar y deserializar el campo `preferences` (JSONB) en cada request puede ser costoso si el JSON es profundo o malformado.

**Estrategia de Optimización: Validación Estricta en el Gateway**

1.  **Schema Validation:** Usar `JsonSchema` (ej. NJsonSchema) para validar la estructura antes de deserializar.
2.  **Límites de Profundidad:** Configurar `JsonDocumentOptions.MaxDepth = 5` para prevenir ataques de "JSON Billion Laughs" o recursión infinita.
3.  **No Indexar Todo:** Solo crear índices GIN en `preferences` si realmente necesitamos buscar usuarios por preferencias (ej. "todos los usuarios con theme='dark'"). Si no, el overhead de mantener el índice GIN en cada UPDATE no vale la pena.

---

## 4. Estrategia de Load Testing (Pruebas de Carga)

Utilizamos **k6** para simular escenarios realistas. Es crucial probar no solo el éxito, sino también los mecanismos de seguridad (IDOR) y optimización (No-Op).

### 4.1. Escenario 1: Actualizaciones de Perfil Mixtas (Texto y No-Op)

**Objetivo:** Validar la latencia sub-100ms y la efectividad de la detección de No-Op bajo carga.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    profile_updates: {
      executor: 'constant-vus',
      vus: 50,
      duration: '5m',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<100'], // SLO estricto para texto
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  // 50% de las requests son No-Op (mismos datos)
  const isNoOp = Math.random() < 0.5;
  
  const payload = JSON.stringify({
    display_name: isNoOp ? "Usuario Estático" : `Usuario_${__VU}_${Date.now()}`,
    preferences: { theme: "dark", notifications: { email: true } }
  });

  const res = http.patch(`${__ENV.BASE_URL}/v1/profile`, payload, {
    headers: { 
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${__ENV.USER_TOKEN}`
    },
  });

  check(res, {
    'status is 200': (r) => r.status === 200,
    'message correct': (r) => r.json('message') !== undefined,
  });

  sleep(0.1); // 10 req/seg por VU
}
```

### 4.2. Escenario 2: Upload de Avatares (Stress Test de CPU/Storage)

**Objetivo:** Validar el procesamiento de imágenes y la subida a S3.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { SharedArray } from 'k6/data';

// Cargar imágenes de prueba (1MB, 5MB)
const images = new SharedArray('images', function () {
  return [
    open('./test_images/1mb_valid.jpg', 'b'),
    open('./test_images/5mb_valid.png', 'b'),
  ];
});

export const options = {
  scenarios: {
    avatar_upload: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 10 },
        { duration: '2m', target: 20 }, // Pico moderado
        { duration: '1m', target: 0 },
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<2000'], // 2 segundos para upload + proceso
  },
};

export default function () {
  const image = images[Math.floor(Math.random() * images.length)];
  
  const res = http.post(`${__ENV.BASE_URL}/v1/profile/avatar`, {
    file: http.file(image, 'avatar.jpg', 'image/jpeg'),
  }, {
    headers: { 'Authorization': `Bearer ${__ENV.USER_TOKEN}` },
  });

  check(res, {
    'avatar uploaded (200)': (r) => r.status === 200,
    'has avatar_url': (r) => r.json('avatar_url') !== undefined,
  });

  sleep(1);
}
```

### 4.3. Escenario 3: Simulación de Ataque IDOR

**Objetivo:** Medir el overhead de la validación de seguridad (JWT + RLS) cuando es bombardeada con IDs aleatorios.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { randomUUID } from 'https://jslib.k6.io/k6-utils/1.4.0/index.js';

export const options = {
  scenarios: {
    idor_attack: {
      executor: 'constant-vus',
      vus: 20,
      duration: '2m',
    },
  },
  thresholds: {
    // Esperamos 403 Forbidden, pero la latencia debe ser baja (<20ms)
    http_req_duration: ['p(95)<50'], 
  },
};

export default function () {
  // Intentar actualizar el perfil de un UUID aleatorio (víctima)
  const victimProfileId = randomUUID();
  
  const res = http.patch(`${__ENV.BASE_URL}/v1/profile/${victimProfileId}`, {
    display_name: "Hacked"
  }, {
    headers: { 
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${__ENV.ATTACKER_TOKEN}` // Token de usuario normal
    },
  });

  check(res, {
    'blocked by security (403)': (r) => r.status === 403,
    'error code is IDOR_ATTEMPT': (r) => r.json('error.code') === 'IDOR_ATTEMPT',
  });

  sleep(0.05); // Alta frecuencia de ataque
}
```

---

## 5. Capacity Planning y Escalabilidad

### 5.1. Almacenamiento de Avatares (S3/Blob)

*   **Cálculo:** Si tenemos 1 millón de usuarios activos, y el 50% sube avatar (promedio 200KB tras compresión).
*   **Espacio:** 500,000 * 200KB = ~100 GB.
*   **Costo:** Mínimo en S3 Standard.
*   **Estrategia:** Usar **S3 Lifecycle Rules** para mover avatares de usuarios inactivos (>1 año sin login) a S3 Glacier o eliminarlos si no son críticos.

### 5.2. Base de Datos (PostgreSQL)

*   **Tabla `profiles`:** Crecimiento lento (1 row por usuario). Índices pequeños.
*   **Tabla `profile_audit_logs`:** Crecimiento rápido. Si cada usuario actualiza su perfil 10 veces al mes, y cambian 3 campos cada vez:
    *   1,000,000 usuarios * 10 updates * 3 fields = 30,000,000 rows/mes.
    *   **Requisito:** Particionamiento mensual obligatorio. Archive a almacenamiento frío después de 12 meses.
*   **IOPS:** Pico de 500 updates/seg * 4 writes (profile + 3 audit logs) = 2,000 IOPS de escritura. Un `db.r5.large` con GP3 (3000 IOPS base) es suficiente.

### 5.3. CDN (CloudFront / Azure CDN)

*   **Cache Hit Ratio:** Debe ser >95% para avatares.
*   **Invalidación:** Usar **Cache Busting** en la URL (ej. `avatar.jpg?v=12345`) en lugar de invalidaciones manuales de CDN, que son lentas y costosas. El `ProfileHandler` actualiza la versión en la DB al subir nuevo avatar.

---

## 6. Observabilidad y Profiling en Producción

### 6.1. Distributed Tracing (OpenTelemetry)

El Trace de una actualización de perfil debe descomponerse para identificar cuellos de botella:

```
HTTP PATCH /profile (Root Span)
├── JWT Validation & Ownership Check (Auth)
├── No-Op Detection (Cache/DB Read)
├── [Condicional] Avatar Upload & Processing (External/CPU)
├── DB Transaction (PostgreSQL)
│   ├── UPDATE profiles
│   └── INSERT profile_audit_logs (Batch)
└── Cache Invalidation (Redis)
```

**Spans Críticos a Monitorear:**
1.  **No-Op Detection:** Si este span es lento, la caché de Redis está fallando o la query de perfil no está optimizada.
2.  **Avatar Processing:** Si >1s, el servidor de aplicaciones está saturado de CPU. Migrar a Lambda/Serverless.
3.  **Audit Log Insert:** Si >50ms, la tabla de auditoría necesita mantenimiento (VACUUM) o el particionamiento no está funcionando.

### 6.2. Dashboards de Grafana (Específicos para UC-08)

#### Panel 1: Eficacia de No-Op y UX
*   **Métricas:** `no_op_detected_total` vs `profile_updates_total`.
*   **Objetivo:** Si la tasa de No-Op es >80%, los clientes (frontend) están enviando requests "guardar" sin que el usuario cambie nada. Se debe corregir en la UI.

#### Panel 2: Salud de Seguridad (IDOR)
*   **Métricas:** `idor_attempts_total` agrupado por `source_ip` y `user_id`.
*   **Alerta:** Si un `user_id` específico genera >50 intentos IDOR en 5 minutos, bloquear su cuenta automáticamente (posible atacante interno o script malicioso).

#### Panel 3: Rendimiento de Avatares
*   **Métricas:** `avatar_upload_latency` (p50, p95, p99) y `avatar_processing_errors_total`.
*   **Objetivo:** Detectar si ciertos tipos de imágenes (ej. PNGs gigantes) están saturando el procesador.

### 6.3. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Audit Log Lag** | `audit_log_insert_latency_p95 > 100ms` | **Alta** | La tabla de auditoría está contendiada. Revisar particiones o locks en DB. |
| **Avatar Queue Buildup** | `avatars_pending_processing > 100` | Media | Si se usa procesamiento asíncrono, los workers no dan abasto. Escalar workers. |
| **High IDOR Rate** | `idor_attempts_total > 100` en 5 min | **Crítica** | Posible ataque de enumeración o bug en el frontend que envía IDs incorrectos. |
| **Cache Miss Spike** | `profile_cache_hit_ratio < 0.80` | Media | Redis está expulsando claves o el TTL es muy corto. Ajustar memoria de Redis. |

---

## 7. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los objetivos de rendimiento iniciales (<100ms p95 para texto) que este documento formaliza. |
| `08_Final_Class_Design.md` | Las clases `ProfileHandler` y `AvatarValidator` deben implementar las optimizaciones de No-Op y procesamiento asíncrono definidas aquí. |
| `10_UC_Specific_Data_Model.md` | El particionamiento de `profile_audit_logs` y los índices GIN definidos allí son críticos para cumplir los SLAs de escritura. |
| `11_UC_Specific_Security_Threats.md` | Las mitigaciones de "IDOR Prevention" (RLS) tienen un costo de rendimiento que se valida en el Escenario 3 de Load Testing. |
| `SHUGO_Performance_Testing_Strategy.md` (Transversal) | Este documento aplica la estrategia global al contexto específico de operaciones de escritura de alta frecuencia. |

---

## 8. Anexos: Configuración de Rendimiento

### 8.1. Configuración de PostgreSQL para JSONB y Auditoría

```ini
# postgresql.conf - Ajustes para UC-08

# Optimización para escritura intensiva (Audit Logs)
wal_buffers = 64MB
checkpoint_completion_target = 0.9
synchronous_commit = on # Crítico para auditoría, no perder logs

# Optimización para JSONB (Preferencias)
# Si no se hacen búsquedas complejas dentro de JSON, desactivar estadísticas extendidas
# default_statistics_target = 100 

# Autovacuum agresivo para tablas particionadas
autovacuum_max_workers = 5
autovacuum_vacuum_scale_factor = 0.01
```

### 8.2. Configuración de S3 Lifecycle para Avatares

```json
{
  "Rules": [
    {
      "ID": "ArchiveInactiveAvatars",
      "Status": "Enabled",
      "Filter": { "Prefix": "avatars/" },
      "Transitions": [
        {
          "Days": 365,
          "StorageClass": "STANDARD_IA" // Infrequent Access
        },
        {
          "Days": 730,
          "StorageClass": "GLACIER" // Archivo frío
        }
      ],
      "Expiration": {
        "Days": 2555 // 7 años (Cumplimiento legal)
      }
    }
  ]
}
```

### 8.3. Configuración de Rate Limiting (Redis)

```csharp
// Configuración para prevenir abuso de actualizaciones
services.AddStackExchangeRedisCache(options => {
    options.Configuration = "redis-cluster:6379";
    options.InstanceName = "shugo_profile_";
});

// Regla: 10 updates por minuto por usuario
var rateLimitOptions = new RateLimitOptions {
    Endpoint = "/v1/profile",
    Period = "1m",
    Limit = 10,
    QuotaExceededMessage = "Too many profile updates. Please try again later."
};
```

---
*Fin del Documento 12_UC_Specific_Performance_Benchmarks para UC-08*