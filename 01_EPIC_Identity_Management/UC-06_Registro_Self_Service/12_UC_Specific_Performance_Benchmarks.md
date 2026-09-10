# UC-06: Registro Self-Service - Benchmarks, SLAs y Estrategia de Rendimiento

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-06 - Registro Self-Service (Self-Service Sign-Up) |
| **Documento:** | 12_UC_Specific_Performance_Benchmarks.md |
| **Metodología:** | Performance Engineering + SRE (SLI/SLO/SLA) + Load Testing (k6) + Capacity Planning |
| **Estándares:** | ISO/IEC 25010 (Performance Efficiency), Google SRE Workbook, OWASP Performance Project |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 04/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 04/09/2026 | Gastón Ruiz | **Creación inicial.** Definición de SLAs, SLOs, estrategia de Load Testing (enfocada en picos de tráfico, degradación de dependencias externas y coste computacional de Argon2id), y planificación de capacidad para el endpoint público de registro. | Documento de referencia para SREs y Arquitectos de Rendimiento. Establece cómo mantener la latencia controlada y la disponibilidad alta a pesar de la sobrecarga inherente del hashing criptográfico (Argon2id), las llamadas a APIs externas (HIBP) y el envío de emails transaccionales. |

---

## 1. Introducción y Propósito

A diferencia de los UCs de lectura (UC-04) o los UCs administrados (UC-01 a UC-03), el **UC-06 (Registro Self-Service)** es una operación de **escritura pública, no autenticada y computacionalmente costosa**.

Sus características de rendimiento son únicas y desafiantes:
1. **Dependencias Externas Críticas:** La latencia total depende de servicios fuera de nuestro control (Have I Been Pwned API, Servidor SMTP).
2. **Coste Computacional Alto:** El hashing de contraseñas con Argon2id es intencionalmente lento y consume mucha memoria (CPU/Memory bound).
3. **Patrón de Compensación:** Si el email falla, debemos ejecutar un `DELETE` adicional, duplicando la carga de escritura en la base de datos para ese request.
4. **Tráfico Impredecible:** Campañas de marketing o ataques de Sybil pueden generar picos de tráfico repentinos (Spike Traffic).

Este documento aborda estos desafíos definiendo SLAs realistas, estrategias de optimización y pruebas de carga específicas para garantizar que el onboarding de usuarios sea rápido, seguro y escalable.

---

## 2. Service Level Indicators (SLIs) y Objectives (SLOs)

### 2.1. SLIs Específicos para UC-06

| SLI | Descripción | Fórmula / Medición |
| :--- | :--- | :--- |
| **Registration Latency (p95)** | Tiempo total desde que el usuario envía el formulario hasta que recibe la confirmación (incluye HIBP y SMTP). | `histogram_quantile(0.95, rate(kagami_registration_latency_bucket[5m]))` |
| **Argon2id Hashing Latency** | Tiempo específico dedicado al hashing de la contraseña. | `histogram_quantile(0.95, rate(kagami_argon2_latency_bucket[5m]))` |
| **HIBP API Latency** | Tiempo de respuesta de la consulta a Have I Been Pwned (incluyendo caché). | `histogram_quantile(0.95, rate(kagami_hibp_latency_bucket[5m]))` |
| **SMTP Latency** | Tiempo de aceptación del email por el servidor SMTP. | `histogram_quantile(0.95, rate(kagami_smtp_latency_bucket[5m]))` |
| **Registration Success Rate** | Porcentaje de registros que completan el flujo sin errores 5xx. | `rate(kagami_registration_success_total[5m]) / rate(kagami_registration_attempts_total[5m])` |
| **Compensation Rate** | Porcentaje de registros que fallan en el envío de email y requieren compensación (DELETE). | `rate(kagami_registration_compensation_total[5m]) / rate(kagami_registration_attempts_total[5m])` |

### 2.2. SLOs y SLAs

| Métrica | SLO (Objetivo Interno - 95%) | SLA (Contrato Enterprise - 99%) | Justificación |
| :--- | :--- | :--- | :--- |
| **Latencia Total (p95)** | **< 1.500 ms** | **< 3.000 ms** | Incluye Argon2id (~200ms), HIBP (~100ms caché), SMTP (~500ms) y DB (~50ms). |
| **Latencia Argon2id (p95)** | **< 250 ms** | **< 500 ms** | Parámetros balanceados: 64MB RAM, 3 iteraciones. |
| **Latencia HIBP (p95)** | **< 50 ms** | **< 200 ms** | Asumiendo Hit en Caché Redis. Si es Miss, puede subir a 800ms. |
| **Throughput (RPS)** | **100 req/seg** | **50 req/seg** | Limitado por el coste de CPU de Argon2id y colas de SMTP. |
| **Disponibilidad** | **99.9%** | **99.5%** | Permite caídas breves de HIBP (usando Circuit Breaker) o SMTP (con reintentos). |
| **Tasa de Error (5xx)** | **< 0.5%** | **< 1.0%** | Errores de DB, SMTP o HIBP no manejados. |
| **Tasa de Conversión** | **> 80%** | **> 70%** | Usuarios que inician registro y llegan a la pantalla de "Verifica tu email". |

---

## 3. Cuellos de Botella y Estrategias de Optimización

### 3.1. Cuello de Botella #1: Argon2id (CPU & Memory Bound)

**Problema:** Argon2id está diseñado para ser lento. Con los parámetros estándar (64MB RAM, 3 iteraciones), un solo core de CPU puede procesar ~4 hashes por segundo. Para 100 RPS, necesitamos ~25 cores dedicados solo a hashing.

**Estrategia de Optimización:**

#### A. Tuning de Parámetros por Entorno
No usar los mismos parámetros en Dev que en Prod.
```csharp
public class Argon2Config
{
    // Producción: Seguridad máxima
    public static Argon2Config Production => new Argon2Config {
        MemorySize = 65536, // 64 MB
        Iterations = 3,
        DegreeOfParallelism = 4 // Usar 4 hilos por hash
    };

    // Desarrollo/Testing: Velocidad máxima
    public static Argon2Config Development => new Argon2Config {
        MemorySize = 1024, // 1 MB
        Iterations = 1,
        DegreeOfParallelism = 1
    };
}
```

#### B. Connection Pooling de Hilos de Hashing
Limitar la concurrencia de hashing para evitar agotar la memoria del servidor (OOM Killer).
```csharp
// SemaphoreSlim para limitar hashing concurrente a N tareas
private static readonly SemaphoreSlim HashingSemaphore = new SemaphoreSlim(10, 10);

public async Task<string> HashPasswordAsync(string password)
{
    await HashingSemaphore.WaitAsync();
    try
    {
        return await Task.Run(() => Argon2id.Hash(password, config));
    }
    finally
    {
        HashingSemaphore.Release();
    }
}
```

### 3.2. Cuello de Botella #2: Have I Been Pwned (Network Bound)

**Problema:** La API de HIBP tiene rate limits estrictos y latencia de red variable.

**Estrategia de Optimización:**

#### A. Caché Agresiva con k-Anonymity
Solo consultamos a HIBP si no está en caché. La clave de caché es el prefijo SHA1 (5 chars).
```csharp
// Clave de caché: "hibp:pwd:{prefix}"
// TTL: 24 horas (las listas de breaches no cambian tan rápido)
var cacheKey = $"hibp:pwd:{hashPrefix}";
var cachedResult = await _cache.GetStringAsync(cacheKey);

if (cachedResult != null) return ParseCachedResult(cachedResult);

// Call HIBP API...
```

#### B. Fallback "Fail-Open" vs "Fail-Closed"
*   **Decisión de Seguridad:** Si HIBP está caído, **bloqueamos el registro** (Fail-Closed). Es preferible perder un registro legítimo que permitir el uso de una contraseña comprometida.
*   **Mitigación de Rendimiento:** Usar un Circuit Breaker con un timeout muy corto (200ms). Si HIBP tarda más de 200ms, asumimos que está caído y bloqueamos.

### 3.3. Cuello de Botella #3: Envío de Email (I/O Bound)

**Problema:** El patrón de compensación requiere que el envío de email sea síncrono. Si SMTP tarda 5 segundos, el usuario espera 5 segundos.

**Estrategia de Optimización:**

#### A. Timeouts Estrictos y Conexiones Persistentes
```csharp
services.AddFluentEmail("noreply@shugo.io")
    .AddSmtpSender(new SmtpClient("smtp.sendgrid.net") {
        Timeout = 2000, // 2 segundos máximo. Si falla, compensamos.
        EnableSsl = true,
        ServicePoint.MaxIdleTime = 1000 // Mantener conexión viva
    });
```

#### B. Optimización del Payload
No enviar HTML complejo en el primer email. Usar texto plano o HTML minimalista para reducir el tiempo de serialización y transmisión.

---

## 4. Estrategia de Load Testing (Pruebas de Carga)

Utilizamos **k6** para simular escenarios realistas. Es crucial mockear HIBP y SMTP en las pruebas de estrés para aislar el rendimiento de nuestra aplicación, pero también debemos tener pruebas de integración realistas.

### 4.1. Escenario 1: Spike Test (Campaña de Marketing)

**Objetivo:** Validar que el sistema soporta un pico de 500 req/seg durante 2 minutos sin caer, degradando elegantemente si es necesario.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  scenarios: {
    spike: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '30s', target: 100 },  // Calentamiento
        { duration: '1m', target: 500 },   // Pico repentino
        { duration: '1m', target: 500 },   // Mantener pico
        { duration: '30s', target: 0 },    // Enfriamiento
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<2000'], // SLO de 2s
    errors: ['rate<0.05'],             // Menos del 5% de errores
  },
};

export default function () {
  const payload = JSON.stringify({
    email: `user_${__VU}_${__ITER}@khcs.com`, // Emails únicos para evitar colisiones
    password: 'SecureP@ssw0rd!123',
    display_name: `User ${__VU}`,
    accept_terms: true,
    terms_version: 'v2.1',
    privacy_policy_version: 'v1.3'
  });

  const res = http.post(`${__ENV.BASE_URL}/v1/auth/register`, payload, {
    headers: { 'Content-Type': 'application/json' },
  });

  const success = check(res, {
    'status is 201 or 400': (r) => r.status === 201 || r.status === 400,
  });

  if (!success) errorRate.add(1);

  sleep(0.1); // 10 req/seg por VU
}
```

### 4.2. Escenario 2: Degradación de Dependencias (Resiliencia)

**Objetivo:** Validar que si HIBP o SMTP se vuelven lentos, el sistema no colapsa (Circuit Breaker y Timeouts).

```javascript
// Simulación: SMTP tarda 5 segundos (configurado en el mock del servidor)
export const options = {
  scenarios: {
    slow_smtp: {
      executor: 'constant-vus',
      vus: 50,
      duration: '2m',
    },
  },
  thresholds: {
    // Esperamos que el timeout de 2s corte la conexión y ejecute la compensación
    http_req_duration: ['p(95)<2500'], 
  },
};

// ... (mismo script que arriba, pero el backend está configurado para tener un SMTP lento)
```

**Resultado Esperado:**
*   La latencia se dispara a ~2.100ms (2s timeout + overhead).
*   La tasa de compensación (DELETE) sube al 100%.
*   El sistema **no se cae** ni agota conexiones de DB (gracias al timeout).

---

## 5. Capacity Planning y Escalabilidad

### 5.1. Cálculo de Recursos para Argon2id

Para soportar **100 RPS** con parámetros de producción (64MB RAM, 250ms por hash):

1.  **Memoria:** 100 hashes simultáneos * 64MB = **6.4 GB de RAM** solo para el proceso de hashing.
2.  **CPU:** 250ms por hash significa que 1 core puede hacer 4 hashes/seg. Para 100 RPS, necesitamos **25 cores** dedicados.

**Recomendación de Infraestructura:**
*   **Opción A (Vertical):** Nodos grandes (ej. AWS c5.4xlarge - 16 vCPUs, 32GB RAM). Necesitamos ~2 instancias.
*   **Opción B (Horizontal + Queue):** Usar una cola (RabbitMQ/SQS) para desacoplar el hashing. El usuario recibe "Registro en proceso" y se le notifica por email cuando termine. *Nota: Esto reduce la UX, por lo que se recomienda Opción A para registro síncrono.*

### 5.2. Cálculo de IOPS de Base de Datos

Por cada registro exitoso:
1.  `SELECT` (Unicidad email)
2.  `INSERT` (Identity)
3.  `INSERT` (PasswordCredential)
4.  `INSERT` (ConsentRecord)
5.  `INSERT` (AuditLog)
6.  `DELETE` (Solo si falla email - ~5% de los casos)

Total: ~5 escrituras + 1 lectura por registro.
Para 100 RPS = **500 IOPS de escritura + 100 IOPS de lectura**.

**Recomendación:**
*   PostgreSQL con almacenamiento GP3 (AWS) configurado para **3.000 IOPS** y **250 MB/s throughput** para tener margen de seguridad.

---

## 6. Observabilidad y Profiling en Producción

### 6.1. Distributed Tracing (OpenTelemetry)

El Trace de un registro debe descomponerse para identificar dónde se pierde el tiempo:

```
HTTP POST /auth/register (Root Span)
├── Rate Limit Check (Redis)
├── Argon2id Hashing (CPU Intensive)
── HIBP Check (External API)
│   └── [Cache Hit] or [Cache Miss -> HTTP Call]
── DB Transaction (PostgreSQL)
│   ├── Insert Identity
│   ├── Insert Consent
│   └── Insert Audit Log
└── SMTP Send (External I/O)
    └── [Success] or [Failure -> DB Delete Compensation]
```

**Spans Críticos a Monitorear:**
1.  **Argon2id Hashing:** Si > 300ms, revisar carga de CPU del nodo.
2.  **HIBP Check:** Si > 100ms, revisar hit rate de caché Redis.
3.  **SMTP Send:** Si > 2000ms, el Circuit Breaker/Timeout debe actuar.

### 6.2. Dashboards de Grafana (Específicos para UC-06)

#### Panel 1: Funnel de Conversión de Registro
*   **Métricas:** `Attempts` -> `Validations Passed` -> `DB Committed` -> `Email Sent`.
*   **Objetivo:** Identificar en qué paso se pierden los usuarios (ej. si muchos fallan en "Validations Passed", las políticas de password son muy estrictas).

#### Panel 2: Coste de Argon2id vs Throughput
*   **Eje X:** RPS (Requests Per Second).
*   **Eje Y:** CPU Usage % y Latencia p95.
*   **Alerta:** Si la CPU supera el 80% y la latencia sube exponencialmente, estamos saturando los hilos de hashing.

#### Panel 3: Tasa de Compensación (Health Check)
*   **Métrica:** `rate(kagami_registration_compensation_total[5m])`.
*   **Alerta:** Si > 10%, el servicio SMTP está fallando intermitentemente. Revisar logs de SendGrid/SMTP.

---

## 7. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los objetivos de rendimiento iniciales (< 2s p95) que este documento formaliza. |
| `09_Modern_Artifacts.md` | Define las métricas de OpenTelemetry; este documento explica cómo usarlas para validar SLAs de dependencias externas. |
| `10_UC_Specific_Data_Model.md` | Los índices únicos y la estructura de tablas definidos aquí son críticos para cumplir los SLOs de latencia de DB. |
| `11_UC_Specific_Security_Threats.md` | Las mitigaciones de "Argon2id DoS" y "Sybil Attack" tienen un costo de rendimiento que se gestiona mediante Rate Limiting y Tuning. |
| `SHUGO_Performance_Testing_Strategy.md` (Transversal) | Este documento aplica la estrategia global al contexto específico de operaciones públicas de alto volumen. |

---

## 8. Anexos: Configuración de Rendimiento

### 8.1. Configuración de Redis para Caché HIBP

```ini
# redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru
save "" # Deshabilitar persistencia (es caché)
timeout 300
```

### 8.2. Configuración de PgBouncer para Registro

```ini
[databases]
shugo_kagami = host=postgres-primary port=5432 dbname=shugo_kagami

[pgbouncer]
pool_mode = transaction
max_client_conn = 2000
default_pool_size = 100
reserve_pool_size = 50 # Reserva alta para picos de registro
reserve_pool_timeout = 2
```

---
*Fin del Documento 12_UC_Specific_Performance_Benchmarks para UC-06*