# UC-01: Provisionar Nueva Identidad - Benchmarks, SLAs y Estrategia de Rendimiento

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-01 - Provisionar Nueva Identidad |
| **Documento:** | 12_UC_Specific_Performance_Benchmarks.md |
| **Metodología:** | Performance Engineering + Capacity Planning + SLO/SLA Definition |
| **Estándares:** | ISO/IEC 25010 (Performance Efficiency), Google SRE (SLI/SLO/SLA) |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 30/08/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 30/08/2026 | Gastón Ruiz | **Creación inicial.** Definición de SLAs, SLOs, estrategia de Load Testing (k6), planificación de capacidad (10k a 1M+ identidades) y técnicas de optimización específicas para el endpoint de provisionamiento. | Documento de referencia para SREs, DevOps y Arquitectos de Rendimiento. Establece los límites operativos del sistema y cómo validarlos antes de llegar a producción. |

---

## 1. Introducción y Propósito

El UC-01 (Provisionar Nueva Identidad) es una operación de **escritura crítica y sensible a la latencia** debido a su dependencia de componentes externos de seguridad (KMS para cifrado de PII) y la base de datos (transacciones atómicas + Outbox).

Este documento define los **Service Level Objectives (SLOs)** que el equipo de ingeniería debe cumplir, la **estrategia de pruebas de carga** para validarlos, y el **Capacity Planning** necesario para escalar desde startups (10k identidades) hasta enterprises (1M+ identidades).

---

## 2. Service Level Agreements (SLAs) y Objectives (SLOs)

### 2.1. Service Level Indicators (SLIs)

Los SLIs son las métricas brutas que medimos en producción mediante OpenTelemetry.

| SLI | Descripción | Fórmula / Medición |
| :--- | :--- | :--- |
| **Latency** | Tiempo transcurrido desde que el API Gateway recibe la request hasta que envía la respuesta HTTP. | `histogram_quantile(0.95, rate(kagami_identity_provisioning_latency_bucket[5m]))` |
| **Throughput** | Número de requests de provisionamiento procesados por segundo. | `rate(kagami_identity_provisioned_total[1m])` |
| **Error Rate** | Porcentaje de requests que resultan en HTTP 5xx (Errores de servidor). *Excluye 4xx (errores de cliente).* | `rate(kagami_identity_provisioning_failed_total{code=~"5.."}[5m]) / rate(kagami_identity_provisioning_total[5m])` |
| **KMS Latency** | Tiempo específico que tarda la llamada al servicio de cifrado (KMS). | `histogram_quantile(0.95, rate(kagami_pii_encryption_latency_bucket[5m]))` |

### 2.2. SLOs (Objetivos Internos) y SLAs (Contratos con el Cliente)

| Métrica | SLO (Objetivo Interno - 95%) | SLA (Contrato Enterprise - 99%) | Justificación |
| :--- | :--- | :--- | :--- |
| **Latencia (p95)** | **< 200 ms** | **< 500 ms** | Incluye validación, cifrado KMS (~50ms), y transacción DB (~50ms). |
| **Latencia (p99)** | **< 500 ms** | **< 1000 ms** | Casos extremos con GC pausas o cold starts de DB. |
| **Throughput** | **100 req/seg** (sostenido) | **50 req/seg** (garantizado) | Suficiente para onboarding masivo (360k identidades/hora). |
| **Disponibilidad** | **99.99%** | **99.95%** | 4.38 min vs 21.9 min de downtime permitido al mes. |
| **Tasa de Error (5xx)** | **< 0.1%** | **< 0.5%** | Fallos críticos (KMS caído, DB full). |

---

## 3. Estrategia de Load Testing (Pruebas de Carga)

Utilizamos **k6** (basado en Go) por su capacidad de generar alta concurrencia con bajo consumo de recursos y su integración nativa con CI/CD.

### 3.1. Escenarios de Prueba

1. **Baseline Test:** 10 usuarios virtuales durante 5 minutos. Establece la latencia mínima sin carga.
2. **Stress Test (Spike):** Aumento repentino de 0 a 500 usuarios virtuales en 1 minuto. Valida el auto-scaling y el Rate Limiting.
3. **Soak Test (Endurance):** 50 usuarios virtuales durante 2 horas. Detecta memory leaks en el servicio o en el driver de PostgreSQL.
4. **KMS Failure Test:** Simulación de latencia de 500ms en el KMS (usando Toxiproxy o mocks). Valida el Circuit Breaker y el timeout del Handler.

### 3.2. Script de Prueba (k6 - Extracto)

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

// Métrica personalizada para errores de servidor
const serverErrors = new Rate('server_errors');

export const options = {
  scenarios: {
    stress_test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 100 }, // Ramp up to 100 VUs
        { duration: '5m', target: 100 }, // Stay at 100 VUs
        { duration: '2m', target: 500 }, // Spike to 500 VUs
        { duration: '5m', target: 500 }, // Stay at 500 VUs
        { duration: '2m', target: 0 },   // Ramp down
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<200', 'p(99)<500'], // SLOs
    server_errors: ['rate<0.001'], // < 0.1% 5xx errors
  },
};

export default function () {
  // Generar datos únicos para evitar DuplicateEmailException (409)
  const uniqueEmail = `user_${Date.now()}_${Math.random().toString(36).substring(7)}@test.com`;
  
  const payload = JSON.stringify({
    type: "USER",
    profile: {
      email: uniqueEmail,
      display_name: "Load Test User",
      custom_attributes: { department: "QA" }
    }
  });

  const params = {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${__ENV.JWT_TOKEN}`,
    },
  };

  const res = http.post(`${__ENV.BASE_URL}/v1/identities`, payload, params);

  const success = check(res, {
    'status is 201': (r) => r.status === 201,
    'status is 409 (Expected duplicate in stress)': (r) => r.status === 409,
    'status is not 5xx': (r) => r.status < 500,
  });

  if (!success) {
    serverErrors.add(res.status >= 500);
  }

  sleep(0.1); // 10 requests per second per VU
}
```

---

## 4. Capacity Planning y Escalabilidad

Estimación de recursos necesarios en Kubernetes (AWS EKS / GCP GKE) y PostgreSQL (RDS/Cloud SQL) para diferentes escalas de operación.

### 4.1. Escenarios de Escala

| Escala | Identidades Totales | Throughput Pico | Recursos KAGAMI (API) | Recursos PostgreSQL (DB) | Estrategia de Escalado |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Startup** | 10,000 | 10 req/seg | 2 Pods (1 vCPU, 2GB RAM) | db.t3.medium (2 vCPU, 4GB RAM, 50GB SSD) | Manual / HPA básico |
| **Scale-up** | 100,000 | 50 req/seg | 5 Pods (2 vCPU, 4GB RAM) | db.r5.large (2 vCPU, 16GB RAM, 200GB SSD) | HPA basado en CPU |
| **Enterprise** | 1,000,000+ | 200 req/seg | 15+ Pods (4 vCPU, 8GB RAM) | db.r5.xlarge (4 vCPU, 32GB RAM, 1TB SSD) + Read Replicas | **Particionamiento DB + HPA avanzado** |

### 4.2. Cuellos de Botella Predictivos

1. **Conexiones de Base de Datos:** PostgreSQL tiene un límite práctico de ~1000 conexiones concurrentes.
   *   *Solución:* Implementar **PgBouncer** (Connection Pooler) en modo *Transaction*. Permite que 50 pods de KAGAMI compartan 100 conexiones reales a la DB.
2. **Límites de KMS:** AWS KMS tiene un límite por defecto de 1000 requests/segundo por clave (Request Quota).
   *   *Solución:* **Envelope Encryption** (definido en Doc 11). Solo ciframos la DEK (32 bytes) en el KMS, no el payload completo. Reduce la cuota consumida en un 90%.
3. **Outbox Lag:** Si se crean 200 identidades/seg, el Outbox Processor debe publicar 200 eventos/seg a Kafka.
   *   *Solución:* Escalar el `OutboxProcessor` horizontalmente usando **Consumer Groups** de Kafka, particionando los eventos por `tenant_id`.

---

## 5. Técnicas de Optimización Específicas para UC-01

### 5.1. Optimización de Base de Datos (PostgreSQL)

*   **JSONB GIN Index:** Para búsquedas rápidas en `custom_attributes`.
    ```sql
    CREATE INDEX idx_profiles_custom_attrs_gin ON profiles USING GIN (custom_attributes);
    ```
*   **Partial Indexes para Outbox:** Reduce drásticamente el tamaño del índice y acelera el `SELECT` del Processor.
    ```sql
    CREATE INDEX idx_outbox_pending ON outbox_events (created_at) WHERE status = 'PENDING';
    ```
*   **Batch Inserts para Bulk API:** Si se usa el endpoint `/identities/bulk`, usar `COPY` o `UNNEST` en lugar de múltiples `INSERT`s individuales. Reduce el overhead de red y WAL (Write-Ahead Logging).

### 5.2. Optimización de Aplicación (.NET 8)

*   **Async/Await Stricto:** Todas las llamadas I/O (DB, KMS, Kafka) deben ser asíncronas para liberar threads del ThreadPool.
*   **Object Pooling:** Reutilizar instancias de `HttpClient` (para KMS) y `KafkaProducer` usando `IHttpClientFactory` y inyección de dependencias singleton.
*   **Span<T> y Memory<T>:** Usar para el procesamiento de strings (emails) y serialización JSON, reduciendo la presión sobre el Garbage Collector (GC).

---

## 6. Observabilidad y Profiling en Producción

### 6.1. Distributed Tracing (OpenTelemetry)

Cada request de provisionamiento genera un Trace con los siguientes Spans críticos para identificar cuellos de botella:

1.  `HTTP POST /identities` (Root Span)
2.  `DB Query: Check Email Uniqueness` (Debe ser < 10ms)
3.  `KMS: Encrypt Email` (Debe ser < 50ms. Si es > 100ms, alerta de degradación de KMS).
4.  `DB Transaction: Insert Identity + Profile + Outbox` (Debe ser < 50ms).
5.  `Kafka: Publish Event` (Asíncrono, no bloquea el Root Span).

### 6.2. Dashboards de Grafana (Específicos para UC-01)

1.  **Provisioning Funnel:** Muestra cuántas requests llegan, cuántas pasan validación, cuántas fallan por cifrado, y cuántas se persisten exitosamente.
2.  **KMS vs DB Latency:** Gráfico comparativo. Si la línea de KMS cruza la de DB, el problema es externo (AWS/Azure), no de nuestra base de datos.
3.  **Outbox Lag:** Diferencia entre `MAX(created_at)` de eventos `PUBLISHED` vs `PENDING`. Si la diferencia es > 1 minuto, el sistema de eventos está roto.

---

## 7. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los objetivos de rendimiento iniciales (< 200ms p95) que este documento formaliza en SLOs. |
| `09_Modern_Artifacts.md` | Define las métricas de OpenTelemetry; este documento explica cómo usarlas para validar SLAs. |
| `10_UC_Specific_Data_Model.md` | Los índices (GIN, Partial) definidos aquí son críticos para cumplir los SLOs de latencia de lectura/escritura. |
| `11_UC_Specific_Security_Threats.md` | La mitigación "Envelope Encryption" (Doc 11) es la base para cumplir el SLO de Throughput sin saturar el KMS. |
| `SHUGO_Performance_Testing_Strategy.md` (Transversal) | Este documento aplica la estrategia global al contexto específico de UC-01. |

---

## 8. Anexos: Configuración de PgBouncer (Connection Pooling)

Para soportar el escalado horizontal de KAGAMI sin agotar las conexiones de PostgreSQL.

```ini
[databases]
shugo_kagami = host=postgres-primary port=5432 dbname=shugo_kagami

[pgbouncer]
listen_port = 6432
listen_addr = 0.0.0.0
auth_type = md5
pool_mode = transaction # CRÍTICO: Permite multiplexar conexiones por transacción, no por sesión.
max_client_conn = 1000
default_pool_size = 50
min_pool_size = 10
reserve_pool_size = 10
reserve_pool_timeout = 3
```

---
*Fin del Documento 12_UC_Specific_Performance_Benchmarks para UC-01*