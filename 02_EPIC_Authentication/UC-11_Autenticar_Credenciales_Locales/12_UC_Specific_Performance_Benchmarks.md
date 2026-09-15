# UC-11: Autenticar Credenciales Locales - Benchmarks, SLAs y Estrategia de Rendimiento

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-11 - Autenticar Credenciales Locales (Local Authentication) |
| **Documento:** | 12_UC_Specific_Performance_Benchmarks.md |
| **Metodología:** | SRE (Site Reliability Engineering) + Performance Engineering + Load Testing (k6) + Capacity Planning |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 15/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 15/09/2026 | Gastón Ruiz | **Creación inicial.** Definición de SLAs, SLOs, estrategia de Load Testing (enfocada en resistencia a Crypto Exhaustion y Credential Stuffing), y planificación de capacidad para el endpoint de autenticación. Incluye benchmarks específicos para la operación criptográfica Argon2id y el patrón Outbox. | Documento de referencia para SREs y Arquitectos de Rendimiento. Establece cómo mantener la latencia de login sub-300ms bajo carga, cómo escalar horizontalmente ante ataques DoS dirigidos a la CPU, y cómo garantizar la entrega de eventos de auditoría. |

---

## 1. Introducción y Propósito

A diferencia de los UCs de gestión de identidad (UC-01 a UC-05), el **UC-11 (Autenticar Credenciales Locales)** presenta un desafío de rendimiento único y paradójico: **la seguridad es el cuello de botella**. 

El uso intencional de **Argon2id** (un algoritmo "memory-hard" y "CPU-intensive") para hashear contraseñas consume ~150ms por request. Esto es excelente para prevenir ataques de cracking offline, pero convierte al endpoint `/v1/auth/login` en un objetivo primario para ataques de **Denegación de Servicio (DoS) por agotamiento criptográfico (Crypto Exhaustion)**.

Este documento aborda estos desafíos definiendo:
1. **SLAs realistas** que aceptan la latencia criptográfica pero protegen la experiencia de usuario legítima.
2. **Estrategias de escalado horizontal** basadas en métricas de latencia criptográfica, no solo en CPU general.
3. **Pruebas de carga específicas** para simular ataques de Credential Stuffing y Crypto DoS.
4. **Optimizaciones** para que las operaciones no criptográficas (Redis Lockout, DB Read Model, Outbox) añadan menos de 20ms de overhead.

---

## 2. Service Level Indicators (SLIs) y Objectives (SLOs)

### 2.1. SLIs Específicos para UC-11

| SLI | Descripción | Fórmula / Medición (OpenTelemetry) |
| :--- | :--- | :--- |
| **Login Latency (p95)** | Tiempo end-to-end desde que el cliente envía el POST hasta que recibe la respuesta (incluye Argon2id). | `histogram_quantile(0.95, rate(kagi_auth_login_duration_bucket[5m]))` |
| **Argon2id Duration (p95)** | Tiempo puro de ejecución del hashing criptográfico (Real o Dummy). | `histogram_quantile(0.95, rate(kagi_crypto_argon2_duration_bucket[5m]))` |
| **Redis Lockout Latency (p95)** | Tiempo de las operaciones atómicas en Redis (`GET`, `INCR`, `DEL`). | `histogram_quantile(0.95, rate(kagi_cache_lockout_duration_bucket[5m]))` |
| **DB Read Model Latency (p95)** | Tiempo de consulta al `identity_auth_snapshot` en PostgreSQL. | `histogram_quantile(0.95, rate(kagi_db_identity_fetch_duration_bucket[5m]))` |
| **Outbox Lag** | Tiempo transcurrido desde que se crea el evento de auditoría hasta que se marca como `processed_at`. | `time() - min(created_at) WHERE processed_at IS NULL` |
| **Authentication Success Rate** | Porcentaje de logins exitosos vs total de intentos. | `rate(kagi_auth_login_total{result="success"}[5m]) / rate(kagi_auth_login_total[5m])` |

### 2.2. SLOs y SLAs

| Métrica | SLO (Objetivo Interno - 95%) | SLA (Contrato Enterprise - 99%) | Justificación |
| :--- | :--- | :--- | :--- |
| **Login Latency (p95)** | **< 250 ms** | **< 400 ms** | Argon2id consume ~150ms. El overhead de red, DB y Redis debe ser < 100ms. |
| **Argon2id Duration (p95)** | **100 - 180 ms** | **< 250 ms** | Rango objetivo para balancear seguridad (resistencia a GPU) y UX. |
| **Redis Lockout Latency** | **< 5 ms** | **< 10 ms** | Redis en la misma VPC/AZ debe responder en sub-milisegundos. |
| **DB Read Model Latency** | **< 10 ms** | **< 20 ms** | Consulta optimizada por índice único en `normalized_email`. |
| **Outbox Lag** | **< 60 segundos** | **< 5 minutos** | Garantiza que KAGE (Auditoría) reciba eventos casi en tiempo real. |
| **Throughput (Por vCPU)** | **~6.5 logins/seg** | **N/A** | Cálculo teórico: 1000ms / 150ms (Argon2id) = 6.66 req/seg por core. |
| **Disponibilidad del Endpoint** | **99.99%** | **99.95%** | El endpoint de login es crítico. Caídas afectan a todos los usuarios. |

---

## 3. Cuellos de Botella y Estrategias de Optimización

### 3.1. Cuello de Botella #1: Crypto Exhaustion (CPU Saturation)

**Problema:** Argon2id bloquea el thread de ejecución. Bajo un ataque de Credential Stuffing (ej. 500 req/seg), un servidor de 4 vCPUs se saturará en segundos, denegando el servicio a usuarios legítimos.

**Estrategia de Optimización:**

#### A. Escalado Horizontal Basado en Latencia Criptográfica (Kubernetes HPA)
No escalamos por CPU general (que puede estar alta por otras razones), sino específicamente por la latencia de Argon2id.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kagi-auth-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kagi-auth
  minReplicas: 3 # Alta disponibilidad mínima
  maxReplicas: 30
  metrics:
    - type: Pods
      pods:
        metric:
          name: kagi_crypto_argon2_duration_p95
        target:
          type: AverageValue
          averageValue: "180" # Escalar si el hashing supera los 180ms
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30 # Escalar agresivamente ante ataques
      policies:
        - type: Percent
          value: 100 # Duplicar capacidad cada 30s
          periodSeconds: 30
```

#### B. Offloading a Workers Asíncronos (Opcional/Avanzado)
Si la latencia de red es crítica, se puede mover la operación `Argon2id.Verify` a un pool de threads dedicado o un sidecar en Rust/C++ optimizado para instrucciones SIMD, liberando el thread principal de ASP.NET Core.

### 3.2. Cuello de Botella #2: Redis Lockout under High Concurrency

**Problema:** Miles de intentos fallidos simultáneos contra la misma cuenta pueden generar contención en Redis.

**Estrategia de Optimización:**
- **Operaciones Atómicas:** Uso estricto de `INCR` y `EXPIRE` en un solo pipeline de Redis cuando sea posible.
- **Connection Pooling:** Configurar `StackExchange.Redis` con `MinPoolSize=50` y `MaxPoolSize=200` para evitar el overhead de crear conexiones TCP bajo carga.

### 3.3. Cuello de Botella #3: Outbox Pattern Throughput

**Problema:** La tabla `auth_outbox` crece rápidamente (1 fila por login). El worker que lee y publica eventos a KAGE puede convertirse en un cuello de botella.

**Estrategia de Optimización:**
- **Batch Processing:** El worker de Outbox no procesa fila por fila. Lee lotes de 100 eventos no procesados (`WHERE processed_at IS NULL LIMIT 100`) y los publica en batch a Kafka/RabbitMQ.
- **Particionamiento:** Como se definió en el Doc 10, la tabla está particionada por mes. El worker solo escanea la partición actual.

---

## 4. Estrategia de Load Testing (Pruebas de Carga)

Utilizamos **k6** para simular escenarios realistas. Es crucial probar no solo el rendimiento normal, sino la resiliencia bajo ataques.

### 4.1. Escenario 1: Login Normal (Happy Path)

**Objetivo:** Validar que el sistema maneja 50 logins/segundo manteniendo la latencia p95 < 250ms.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Trend, Rate } from 'k6/metrics';

const loginLatency = new Trend('login_latency', true);
const errorRate = new Rate('errors');

export const options = {
  scenarios: {
    normal_load: {
      executor: 'constant-vus',
      vus: 10,
      duration: '5m',
    },
  },
  thresholds: {
    login_latency: ['p(95)<250'], // SLO de 250ms
    errors: ['rate<0.01'],
  },
};

export default function () {
  const payload = JSON.stringify({
    email: `user_${__VU}@khcs.com`,
    password: 'SecureP@ssw0rd!',
    device_id: 'test-device-123'
  });

  const res = http.post(`${__ENV.BASE_URL}/v1/auth/login`, payload, {
    headers: { 
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${__ENV.API_KEY}` // Si aplica para el gateway
    },
  });

  loginLatency.add(res.timings.duration);

  check(res, {
    'login successful (200)': (r) => r.status === 200,
    'has access_token': (r) => r.json('access_token') !== undefined,
  });

  if (res.status !== 200) errorRate.add(1);

  sleep(1); // 1 req/seg por VU
}
```

### 4.2. Escenario 2: Simulación de Credential Stuffing (Alta Concurrencia de Fallos)

**Objetivo:** Validar que el Rate Limiting del API Gateway y el Lockout de Redis protegen el sistema cuando el 99% de los logins fallan.

```javascript
import http from 'k6/http';
import { check } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');
const lockoutRate = new Rate('lockouts');

export const options = {
  scenarios: {
    stuffing_attack: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 100 }, // Ramp up
        { duration: '4m', target: 100 }, // Sostener ataque
        { duration: '1m', target: 0 },   // Ramp down
      ],
    },
  },
  thresholds: {
    // El sistema debe seguir respondiendo (aunque sea con 401/429) en < 300ms
    http_req_duration: ['p(95)<300'], 
    errors: ['rate<0.05'], // Aceptamos algunos 503 por saturación controlada
  },
};

export default function () {
  // Atacante prueba passwords aleatorios contra una cuenta existente
  const payload = JSON.stringify({
    email: 'victim@khcs.com',
    password: `RandomPassword_${Math.random().toString(36).substring(7)}`,
  });

  const res = http.post(`${__ENV.BASE_URL}/v1/auth/login`, payload, {
    headers: { 'Content-Type': 'application/json' },
  });

  if (res.status === 423) lockoutRate.add(1); // Cuenta bloqueada
  if (res.status !== 200 && res.status !== 401 && res.status !== 423 && res.status !== 429) {
    errorRate.add(1);
  }
  
  // No sleep: atacar lo más rápido posible
}
```

### 4.3. Escenario 3: Stress Test de Crypto Exhaustion (DoS)

**Objetivo:** Determinar el punto de quiebre del sistema y validar que el HPA escala correctamente.

```javascript
// Configuración para saturar CPU
export const options = {
  scenarios: {
    dos_attack: {
      executor: 'constant-arrival-rate',
      rate: 500, // 500 requests por segundo
      timeUnit: '1s',
      duration: '10m',
      preAllocatedVUs: 200,
      maxVUs: 500,
    },
  },
};
// ... (resto del script similar al Escenario 2)
```

---

## 5. Capacity Planning y Escalabilidad

### 5.1. Cálculo de Recursos para el Motor KAGI (Auth)

Dado que Argon2id es **CPU-bound**, el dimensionamiento se basa en núcleos de CPU, no en memoria.

1. **Costo por Login:** ~150ms de CPU pura (Argon2id) + ~20ms de overhead (Redis, DB, JWT) = ~170ms.
2. **Capacidad por vCPU:** 1000ms / 170ms ≈ **5.8 logins/segundo por vCPU**.
3. **Escenario de Pico:** 200 logins/segundo (pico de mañana laboral).
4. **vCPUs Requeridos:** 200 / 5.8 ≈ **34.5 vCPUs**.

**Recomendación de Infraestructura:**
- **Pods de KAGI Auth:** 10 pods con 4 vCPUs cada uno (Total 40 vCPUs).
- **Memoria:** 512MB por pod (Argon2id usa 64MB por operación, pero se libera inmediatamente. 512MB permite ~8 operaciones concurrentes por pod sin OOM).
- **Redis:** Instancia `cache.r6g.large` (2 vCPUs, 12.5GB RAM) es más que suficiente para lockouts (claves pequeñas con TTL).

### 5.2. Cálculo de Throughput de Base de Datos (PostgreSQL)

- **Lecturas:** 1 lectura por login en `identity_auth_snapshot`.
- **Escrituras:** 1 escritura en `refresh_tokens` (solo en éxito) + 1 escritura en `auth_outbox` (siempre).
- **IOPS Estimados:** Para 200 logins/seg -> ~200 IOPS de lectura + ~400 IOPS de escritura.
- **Recomendación:** PostgreSQL `db.r6g.large` con almacenamiento GP3 (3000 IOPS base) es suficiente. El Read Model está optimizado con índices parciales.

---

## 6. Observabilidad y Profiling en Producción

### 6.1. Distributed Tracing (OpenTelemetry)

El Trace de una autenticación exitosa debe descomponerse en spans granulares para identificar cuellos de botella:

```text
POST /v1/auth/login (Root Span) [Total: 185ms]
├── cache.lockout.check [4ms] (Redis GET)
├── db.identity.fetch [12ms] (KAGAMI Read Model)
├── crypto.argon2.verify [155ms] (CPU Bound - Real o Dummy)
│   └── crypto.argon2.dummy [155ms] (Si el usuario no existe)
├── cache.lockout.reset [3ms] (Redis DEL)
── token.jwt.issue [4ms] (RSA/EdDSA signing)
├── db.refresh_token.insert [5ms] (PostgreSQL)
└── audit.outbox.insert [2ms] (PostgreSQL Outbox)
```

**Spans Críticos a Monitorear:**
1. **`crypto.argon2.verify`:** Si el p95 supera los 200ms, el HPA debe escalar. Si hay divergencia entre `type=real` y `type=dummy`, hay un riesgo de Timing Attack.
2. **`db.identity.fetch`:** Si supera los 20ms, revisar el índice en `normalized_email` o la carga de la Read Replica.

### 6.2. Dashboards de Grafana (Específicos para UC-11)

#### Panel 1: Salud Criptográfica y Anti-Enumeración
*   **Métricas:** `kagi_crypto_argon2_duration_p95` (línea azul para `type=real`, línea roja para `type=dummy`).
*   **Objetivo:** Ambas líneas deben estar superpuestas (~150ms).
*   **Alerta:** Si la línea roja cae a <50ms, el Dummy Hash está roto y el sistema es vulnerable a User Enumeration.

#### Panel 2: Efectividad de Lockout y Rate Limiting
*   **Métricas:** `kagi_auth_lockout_triggered_total` vs `kagi_auth_login_total{result=fail}`.
*   **Objetivo:** Visualizar picos de fallos y cómo el lockout los corta abruptamente.

#### Panel 3: Outbox Lag (Auditoría)
*   **Métricas:** `time() - min(auth_outbox.created_at) WHERE processed_at IS NULL`.
*   **Objetivo:** Mantener el lag < 60 segundos.
*   **Alerta:** Si el lag > 5 minutos, el worker de Outbox está caído o KAGE no está consumiendo.

### 6.3. Alertas Sugeridas (SRE & Security)

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Crypto DoS Inminente** | `kagi_crypto_argon2_duration_p95 > 250ms` por 2 min | **Crítica** | El HPA no está escalando lo suficientemente rápido. Activar CAPTCHA obligatorio en el API Gateway. |
| **Timing Attack Detected** | `abs(avg(argon2_real) - avg(argon2_dummy)) > 50ms` | Alta | Revisar implementación del `PasswordHasher`. Posible vulnerabilidad de enumeración. |
| **Outbox Stuck** | `auth_outbox_lag_seconds > 300` | Alta | El sistema de auditoría (KAGE) no está recibiendo eventos. Revisar worker de Outbox. |
| **Redis Connection Pool Exhausted** | `redis_pool_waiting_connections > 10` | Media | Aumentar `MaxPoolSize` en la configuración de StackExchange.Redis. |

---

## 7. Configuración de Rendimiento

### 7.1. Configuración de la Aplicación (appsettings.json)

```json
{
  "KagiAuth": {
    "Argon2id": {
      "MemoryCostKB": 65536, // 64 MB
      "Iterations": 3,
      "Parallelism": 4,
      "HashLength": 32
    },
    "Lockout": {
      "MaxFailures": 5,
      "DurationMinutes": 15
    },
    "Tokens": {
      "AccessTokenLifetimeMinutes": 15,
      "RefreshTokenLifetimeDays": 30
    }
  },
  "Redis": {
    "ConnectionString": "redis-cluster:6379",
    "MinPoolSize": 50,
    "MaxPoolSize": 200,
    "ConnectTimeout": 1000,
    "SyncTimeout": 2000
  }
}
```

### 7.2. Configuración de PostgreSQL para el Read Model y Outbox

```ini
# postgresql.conf - Optimizaciones para UC-11

# El Read Model es 99% lecturas. La Outbox es 100% escrituras.
# Si están en la misma instancia, balancear:
shared_buffers = 4GB
effective_cache_size = 12GB
work_mem = 64MB # Suficiente para ordenar eventos de outbox

# Autovacuum agresivo para la tabla Outbox (muchos inserts/deletes)
autovacuum_max_workers = 3
autovacuum_vacuum_scale_factor = 0.05
autovacuum_analyze_scale_factor = 0.02

# Logging de queries lentas (para detectar problemas en el Read Model)
log_min_duration_statement = 50 # Log queries > 50ms
```

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los objetivos de rendimiento iniciales (<300ms p95) que este documento formaliza. |
| `08_Final_Class_Design.md` | Las clases `PasswordHasher` y `LockoutManager` deben implementar las optimizaciones de pooling y manejo de memoria definidas aquí. |
| `10_UC_Specific_Data_Model.md` | Los índices parciales y el particionamiento de la Outbox definidos allí son críticos para cumplir los SLAs de latencia. |
| `11_UC_Specific_Security_Threats.md` | Las mitigaciones de "Crypto Exhaustion" (T-73) se validan mediante los escenarios de Load Testing y el HPA definido aquí. |
| `SHUGO_Performance_Testing_Strategy.md` (Transversal) | Este documento aplica la estrategia global al contexto específico de operaciones CPU-bound y de alta seguridad. |

---

## 9. Cierre del Paquete de Diseño (UC-11)

Con este documento, completamos los **12 archivos del Paquete de Diseño Profundo (Tier 1 + Tier 2)** para el UC-11:

1. ✅ `01_Requirements_Specification.md`
2. ✅ `02_Use_Case_Diagram.md`
3. ✅ `03_Activity_Diagram.md`
4. ✅ `04_Initial_Domain_Model.md`
5. ✅ `05_State_Machine_Diagram.md`
6. ✅ `06_Robustness_Diagram.md`
7. ✅ `07_Sequence_Diagram.md`
8. ✅ `08_Final_Class_Design.md`
9. ✅ `09_Modern_Artifacts.md`
10. ✅ `10_UC_Specific_Data_Model.md`
11. ✅ `11_UC_Specific_Security_Threats.md`
12. ✅ `12_UC_Specific_Performance_Benchmarks.md`

### 9.1. Resumen de lo Logrado

Hemos diseñado el endpoint de autenticación local más seguro, observable y resiliente del mercado IAM:
- **Seguridad Criptográfica de Elite:** Argon2id con Dummy Hash para prevenir Timing Attacks.
- **Resiliencia Fail-Secure:** Si Redis cae, el sistema deniega el acceso para proteger al usuario.
- **Escalado Inteligente:** HPA basado en latencia criptográfica, no solo en CPU general.
- **Auditoría Garantizada:** Patrón Outbox con particionamiento para asegurar que ningún login (éxito o fallo) deje de registrarse.
- **Cumplimiento Normativo:** Mapeo directo a NIST SP 800-63B, OWASP ASVS y SOC 2.

---
*Fin del Documento 12_UC_Specific_Performance_Benchmarks para UC-11*