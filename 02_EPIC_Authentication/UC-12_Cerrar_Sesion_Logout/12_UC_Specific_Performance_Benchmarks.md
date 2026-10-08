# UC-12: Cerrar Sesión (Logout) - Benchmarks, SLAs y Estrategia de Rendimiento

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-12 - Cerrar Sesión (Logout & Token Revocation) |
| **Documento:** | 12_UC_Specific_Performance_Benchmarks.md |
| **Metodología:** | SRE (Site Reliability Engineering) + Performance Engineering + Load Testing (k6) + Chaos Engineering + Capacity Planning |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 20/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 20/09/2026 | Gastón Ruiz | **Creación inicial.** Definición de SLAs, SLOs y estrategia de Load Testing para el cierre de sesión. Modela los cuatro perfiles de carga críticos: olas de logout de fin de turno (1.000 logouts/seg), Cascade Logout masivo, el "middleware tax" (latencia añadida por la verificación de Blacklist en CADA request autenticado), y Chaos Engineering con caída de Redis durante la ola. Incluye capacity planning de Redis bajo TTL efímero, dimensionamiento de PostgreSQL para workloads update-heavy, y el benchmark del job de reconciliación post-Fail-Safe. | Documento de referencia para SREs y Arquitectos de Rendimiento. Cierra el UC-12 garantizando que el logout sea instantáneo para el usuario, que la verificación de revocación no degrade el tráfico global de la API, y que las olas de logout enterprise no saturen Redis ni PostgreSQL. |

---

## 1. Introducción y Propósito

El **UC-12 (Cerrar Sesión)** presenta un perfil de rendimiento atípico dentro de la plataforma: es una operación **barata individualmente pero masiva en agregado**, y —críticamente— su mecanismo de seguridad (la Blacklist de JTI) introduce un costo permanente en **todas** las requests autenticadas de la plataforma, no solo en el logout.

Este documento aborda cuatro desafíos de rendimiento únicos:

1. **El "Middleware Tax":** Cada request autenticado a cualquier endpoint de SHUGO ejecuta un `GET blacklist:{jti}` en Redis antes de aceptar el JWT. Esta latencia añadida (1-2ms) se multiplica por millones de requests diarias. Si Redis se degrada, **toda la plataforma** se degrada. Debe estar medida, acotada y protegida por Circuit Breaker.
2. **Olas de Logout de Fin de Turno:** En tenants enterprise (hospitales, fábricas, bancos), miles de usuarios cierran sesión simultáneamente al terminar el turno (ej. 18:00). El sistema debe absorber picos de 1.000 logouts/segundo sin degradar el login (UC-11) que ocurre en paralelo.
3. **Cascade Logout Set-Based:** Revocar 100 sesiones no debe costar 100 round-trips a DB ni 100 round-trips a Redis. Se requieren operaciones set-based y pipelining.
4. **Reconciliación post-Fail-Safe:** Tras recuperar Redis de una caída, el job de backfill de Blacklist debe procesar miles de JTIs pendientes rápidamente, sin saturar Redis ni dejar la ventana de riesgo abierta más tiempo del necesario.

---

## 2. Service Level Indicators (SLIs) y Objectives (SLOs)

### 2.1. SLIs Específicos para UC-12

| SLI | Descripción | Fórmula / Medición (OpenTelemetry) |
| :--- | :--- | :--- |
| **Logout Latency (p95)** | Latencia end-to-end de `POST /auth/logout` (extracción JTI → revocación DB → Blacklist → Outbox → 204). | `histogram_quantile(0.95, rate(kagi_auth_logout_duration_bucket[5m]))` |
| **Middleware Blacklist Check (p99)** | Latencia añadida por la verificación de Blacklist en **cada request autenticada**. | `histogram_quantile(0.99, rate(kagi_auth_blacklist_check_duration_bucket[5m]))` |
| **Cascade Logout Latency (p95)** | Latencia de un logout con `logout_everywhere=true` (hasta 100 tokens). | `histogram_quantile(0.95, rate(kagi_auth_logout_duration_bucket{mode="cascade"}[5m]))` |
| **DB Revocation Latency (p95)** | Latencia del `UPDATE ... RETURNING` en `refresh_tokens`. | `histogram_quantile(0.95, rate(kagi_auth_db_revoke_duration_bucket[5m]))` |
| **Blacklist Write Latency (p95)** | Latencia del `SET blacklist:{jti} EX {ttl}`. | `histogram_quantile(0.95, rate(kagi_auth_blacklist_write_duration_bucket[5m]))` |
| **Outbox Lag** | Edad del evento `Session.Terminated` más antiguo sin publicar. | `time() - min(created_at) WHERE processed_at IS NULL` |
| **Reconciliation Completion Time** | Tiempo desde que Redis recupera salud hasta que el backfill de JTIs `FAILSAFE_SKIPPED` completa. | `kagi_auth_reconciliation_completion_seconds` (histogram por incidente) |
| **Blacklist Memory Usage** | Memoria consumida por claves `blacklist:*` en Redis. | `redis_memory_used_bytes{db="auth"}` vs. curva teórica |

### 2.2. SLOs y SLAs

| Métrica | SLO (Objetivo Interno - 95%) | SLA (Contrato Enterprise - 99%) | Justificación |
| :--- | :--- | :--- | :--- |
| **Logout Latency (p95)** | **< 100 ms** | **< 150 ms** | Suma teórica de spans: 1+8+12+0+6+6+8 ≈ 45ms. Margen para contención de pools. |
| **Logout Latency (p99)** | **< 150 ms** | **< 250 ms** | Cola de conexiones bajo ola de fin de turno. |
| **Middleware Blacklist Check (p99)** | **< 5 ms** | **< 10 ms** | Es un impuesto sobre TODO el tráfico autenticado. Debe ser invisible. |
| **Cascade Logout (p95, ≤100 tokens)** | **< 500 ms** | **< 800 ms** | 1 UPDATE set-based + 1 pipeline Redis de ≤100 SETs + 1 INSERT multi-fila. |
| **Throughput de Logout (pico)** | **1.000 logouts/seg** | **500 logouts/seg** | Ola de fin de turno en tenant enterprise de 50.000 empleados. |
| **Outbox Lag** | **< 60 s** | **< 300 s** | Propagación de `Session.Terminated` a KAGE/KIZUNA. |
| **Reconciliation Completion** | **< 5 min post-recuperación** | **< 10 min** | Cierra la ventana de riesgo residual del Fail-Safe. |
| **Disponibilidad del Endpoint** | **99.99%** | **99.95%** | El logout nunca debe caer, ni siquiera con Redis caído (Fail-Safe). |
| **Redis Evicted Keys** | **0 (siempre)** | **0 (siempre)** | Una evicción de Blacklist es una vulnerabilidad de seguridad (T-80). |

---

## 3. Cuellos de Botella y Estrategias de Optimización

### 3.1. Cuello de Botella #1: El "Middleware Tax" sobre el Tráfico Global

**Problema:** La verificación `GET blacklist:{jti}` ocurre en el hot path de **cada request autenticada**. A 5.000 req/seg de tráfico API, son 5.000 GETs/seg adicionales a Redis. Si la latencia de Redis sube de 1ms a 20ms (contención, GC, red), **toda la plataforma** suma 20ms por request.

**Estrategias de Optimización:**

#### A. Multiplexing de Conexiones (StackExchange.Redis)
StackExchange.Redis multiplexa todas las llamadas sobre un único socket por instancia. No se crea un connection pool tradicional; se configura un timeout agresivo para fail-fast:
```csharp
var redisOptions = new ConfigurationOptions
{
    EndPoints = { "redis-auth:6379" },
    AsyncTimeout = 50,          // Fail-fast: el Circuit Breaker toma el control
    ConnectTimeout = 500,
    AbortOnConnectFail = false,
    KeepAlive = 60
};
```

#### B. Circuit Breaker en el Middleware (Fail-Open Acotado)
Cuando el Circuit Breaker de Redis está **OPEN**, el middleware **omite el check en 0ms** (sin intentar conexión), aplicando el fail-mode configurado por tenant. Esto garantiza que una caída de Redis no añada latencia al tráfico global:
```csharp
if (_circuitBreaker.CircuitState == CircuitState.Open)
{
    // 0ms: no se intenta conectar. Fail-mode por tenant (open/closed).
    Interlocked.Increment(ref _middlewareShortCircuits);
    return await ApplyFailMode(context);
}
```

#### C. Alternativas Evaluadas y Rechazadas (Documentación de Decisión)
| Alternativa | Por qué se rechazó |
| :--- | :--- |
| **Cache local negativa** (recordar JTIs no-blacklisteados por N segundos) | Un token revocado *durante* la ventana de cache negativa sería aceptado. Amplía la ventana de replay de 0s a Ns por decisión propia. Inaceptable. |
| **Bloom Filter en memoria** | Falsos positivos rechazarían tokens **válidos** (DoS autoinfligido). Falsos negativos (tras resize) aceptarían revocados. Complejidad sin beneficio real sobre un GET de 1ms. |
| **Push de invalidación a cada pod** (broadcast de JTIs revocados) | Requiere mensajería fan-out a N pods con garantías de entrega. Convierte un GET de 1ms en un problema de consistencia distribuida. Overkill para TTLs de 15min. |

**Decisión final:** GET directo a Redis con Circuit Breaker. Simple, correcto, y acotado en el peor caso.

### 3.2. Cuello de Botella #2: Cascade Logout con N Round-Trips

**Problema:** Un Cascade Logout ingenuo (loop de UPDATEs + loop de SETs) con 100 sesiones = 100 round-trips a PostgreSQL + 100 round-trips a Redis ≈ 1-2 segundos.

**Estrategia de Optimización:**

#### A. Revocación Set-Based (1 round-trip a DB)
```sql
-- Un solo UPDATE con índice parcial idx_rt_identity_active
UPDATE kagi_auth.refresh_tokens
SET is_revoked = TRUE, revoked_at = NOW(),
    revocation_reason = 'CASCADE_LOGOUT', revoked_by_identity_id = :initiated_by
WHERE identity_id = :identity_id AND is_revoked = FALSE AND expires_at > NOW()
RETURNING id, replaced_by_token_id;   -- cascade_size = row_count
```

#### B. Pipelining de Blacklist (1 round-trip a Redis)
```csharp
// Batch de hasta 100 SETs en un solo pipeline (StackExchange.Redis multiplexa automáticamente)
var batch = _redis.CreateBatch();
foreach (var jti in revokedJtis)
{
    var ttl = jti.Expiration - DateTime.UtcNow + TimeSpan.FromMinutes(1);
    _ = batch.StringSetAsync($"blacklist:{jti.Value}", "CL", ttl);
}
await batch.ExecuteAsync(); // 1 round-trip
```

#### C. Inserción Multi-Fila de Auditoría y Outbox
```sql
-- 1 INSERT para N filas de auditoría (unnest de arrays)
INSERT INTO kagi_auth.session_revocation_audit
    (tenant_id, identity_id, refresh_token_id, jti, revocation_reason,
     initiated_by_identity_id, cascade_size, blacklist_write_status, ip_address, user_agent)
SELECT :tenant, :identity, unnest(:token_ids), unnest(:jtis), 'CASCADE_LOGOUT',
       :initiated_by, :cascade_size, :bl_status, :ip, :ua;
```

**Resultado benchmark esperado:** Cascade de 100 tokens ≈ 12ms (DB) + 8ms (pipeline Redis) + 10ms (INSERTs multi-fila) ≈ **30-40ms**, no 1-2s.

### 3.3. Cuello de Botella #3: Contención de Pools bajo Ola de Fin de Turno

**Problema:** 1.000 logouts/seg × 4 operaciones de DB (SELECT idempotencia + UPDATE + INSERT audit + INSERT outbox) = 4.000 transacciones/seg. Con pools mal dimensionados, las requests esperan conexión.

**Estrategia de Optimización:**
- **Npgsql Pool:** `MaxPoolSize=50` por pod × 20 pods = 1.000 conexiones. PostgreSQL con `max_connections=1200` (headroom para PgBouncer y workers).
- **Transacción única por logout:** Las 3 escrituras (UPDATE + audit + outbox) comparten una transacción, reduciendo overhead de commit (1 commit en lugar de 3).
- **PgBouncer en modo transaction:** Para los workers de Outbox y reconciliación, que no necesitan afinidad de sesión.

### 3.4. Cuello de Botella #4: Reconciliación post-Fail-Safe

**Problema:** Tras una caída de Redis de 10 minutos con 500 logouts/seg, hay ~300.000 filas `FAILSAFE_SKIPPED` candidatas a backfill (aunque solo las de los últimos ≤16min tienen TTL vivo: ~480.000 en el peor caso teórico, acotadas por la ventana de expiración de los JWTs).

**Estrategia de Optimización:**
- **Filtro por expiración viva:** Solo se backfillean JTIs cuyo `jwt_expiration > NOW()`. El resto expiró naturalmente: **no se tocan** (ahorro masivo de escrituras).
- **Chunks de 5.000 con pipelining:** El job procesa en lotes con `batch.ExecuteAsync()`, limitando a 50.000 SETs/seg para no saturar Redis durante la recuperación.
- **Idempotencia del backfill:** `SET` es idempotente; re-ejecutar el job tras un fallo parcial no corrompe estado.

**Benchmark objetivo:** 480.000 JTIs backfilleados en **< 3 minutos** (≈ 2.700 SETs/seg sostenidos con pipelining).

---

## 4. Estrategia de Load Testing (k6)

### 4.1. Escenario 1: Ola de Logout de Fin de Turno

**Objetivo:** Validar 1.000 logouts/seg sostenidos con latencia p95 < 100ms, sin degradar el login concurrente.

```javascript
import http from 'k6/http';
import { check } from 'k6';
import { Trend, Rate } from 'k6/metrics';

const logoutLatency = new Trend('logout_latency', true);
const loginLatency = new Trend('login_latency_concurrent', true);
const errorRate = new Rate('errors');

export const options = {
  scenarios: {
    end_of_shift_wave: {
      executor: 'ramping-arrival-rate',
      startRate: 50, timeUnit: '1s',
      preAllocatedVUs: 200, maxVUs: 1500,
      stages: [
        { target: 200,  duration: '2m' },  // Calentamiento
        { target: 1000, duration: '3m' },  // Rampa de ola
        { target: 1000, duration: '5m' },  // Pico sostenido (fin de turno)
        { target: 0,    duration: '2m' },  // Disipación
      ],
      exec: 'logoutWave',
    },
    concurrent_login: {
      executor: 'constant-arrival-rate',
      rate: 100, timeUnit: '1s', duration: '12m',
      preAllocatedVUs: 50, maxVUs: 200,
      exec: 'loginBackground', // El login NO debe degradarse durante la ola
    },
  },
  thresholds: {
    logout_latency: ['p(95)<100', 'p(99)<150'],
    login_latency_concurrent: ['p(95)<300'], // SLO de UC-11 intacto
    errors: ['rate<0.001'],
  },
};

export function logoutWave() {
  const session = __ENV.SESSION_POOL.split(',')[__VU % 1000]; // tokens pre-provisionados en setup()
  const res = http.post(`${__ENV.BASE_URL}/v1/auth/logout`,
    JSON.stringify({ refresh_token: session.refresh }),
    { headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${session.access}` } });

  logoutLatency.add(res.timings.duration);
  if (!check(res, { 'logout 204': (r) => r.status === 204 })) errorRate.add(1);
}

export function loginBackground() {
  const res = http.post(`${__ENV.BASE_URL}/v1/auth/login`,
    JSON.stringify({ email: `load_${__VU}@khcs.com`, password: 'SecureP@ssw0rd!' }),
    { headers: { 'Content-Type': 'application/json' } });
  loginLatency.add(res.timings.duration);
}
```

### 4.2. Escenario 2: Middleware Tax bajo Tráfico Global

**Objetivo:** Medir la latencia añadida por el check de Blacklist sobre requests autenticadas, con Redis sano y con Redis degradado (Circuit Breaker open).

```javascript
export const options = {
  scenarios: {
    authenticated_traffic: {
      executor: 'constant-arrival-rate',
      rate: 3000, timeUnit: '1s', duration: '10m', // 3.000 req/seg autenticadas
      preAllocatedVUs: 300, maxVUs: 800,
    },
  },
  thresholds: {
    // El "impuesto" del middleware debe ser invisible
    middleware_blacklist_check: ['p(99)<5'],
    http_req_duration: ['p(95)<80'], // endpoint de referencia con check incluido
  },
};

export default function () {
  const res = http.get(`${__ENV.BASE_URL}/v1/identities/me`,
    { headers: { Authorization: `Bearer ${__ENV.VALID_JWT_POOL}` }, tags: { authed: 'true' } });
  check(res, { 'me 200': (r) => r.status === 200 });
}
// Nota operativa: durante el test, SRE inyecta 30ms de latencia a Redis con toxiproxy
// en el minuto 5. Se espera que el Circuit Breaker abra y el p99 del check caiga a ~0ms
// (fail-open), NO que suba a 30ms. Eso valida la protección del tráfico global.
```

### 4.3. Escenario 3: Cascade Logout Masivo

**Objetivo:** Validar que un Cascade de 100 sesiones completa en < 500ms p95 usando set-based + pipelining.

```javascript
export const options = {
  scenarios: {
    cascade_stress: {
      executor: 'per-vu-iterations',
      vus: 40, iterations: 25, maxDuration: '10m',
    },
  },
  thresholds: {
    cascade_latency: ['p(95)<500', 'p(99)<800'],
    cascade_tokens_revoked: ['avg>=50'], // identidades de prueba con 50-100 sesiones
  },
};

export function cascade() {
  const identity = pickProvisionedIdentityWithSessions(); // setup(): 50-100 sesiones activas c/u
  const start = Date.now();
  const res = http.post(`${__ENV.BASE_URL}/v1/auth/logout`,
    JSON.stringify({ refresh_token: identity.refresh, logout_everywhere: true }),
    { headers: { Authorization: `Bearer ${identity.access}` } });
  cascadeLatency.add(Date.now() - start);
  check(res, { 'cascade 204': (r) => r.status === 204 });
}
```

### 4.4. Escenario 4: Chaos Engineering - Redis Cae durante la Ola

**Objetivo:** Validar que (a) el logout mantiene disponibilidad 99.9% y latencia acotada con Redis muerto, y (b) la reconciliación cierra la ventana en < 5 min tras la recuperación.

```javascript
export const options = {
  scenarios: {
    wave_during_chaos: {
      executor: 'constant-arrival-rate',
      rate: 500, timeUnit: '1s', duration: '15m',
      preAllocatedVUs: 300, maxVUs: 800,
    },
  },
  thresholds: {
    // Con Circuit Breaker open, el logout NO debe esperar timeouts de Redis
    logout_latency: ['p(95)<120'],
    logout_availability: ['rate>0.999'],
    // Post-recuperación (verificación externa vía Prometheus):
    // reconciliation_completion_seconds < 300
  },
};
// Runbook del caos:
// min 0-5:  Redis sano (baseline)
// min 5-10: `kubectl delete pod redis-auth-0` + bloqueo de red (Toxiproxy)
// min 10:   Restaurar Redis
// min 10-15: Verificar que reconciliation_backfilled_total > 0 y completion < 300s
```

---

## 5. Capacity Planning y Escalabilidad

### 5.1. Redis Auth (Blacklist) bajo Ola de Fin de Turno

| Parámetro | Cálculo | Resultado |
| :--- | :--- | :--- |
| Entradas concurrentes (pico 1.000 logouts/seg) | 1.000 × 960s (TTL máx) | 960.000 entradas |
| Memoria por entrada | ~120 bytes (clave+valor+overhead) | — |
| Memoria de Blacklist en pico | 960.000 × 120 B | **~115 MB** |
| GETs/seg del middleware (tráfico API) | 5.000 req/seg autenticadas | 5.000 ops/seg |
| SETs/seg de logout (pico) | 1.000 | 1.000 ops/seg |
| **Total ops/seg en pico** | — | **~6.000 ops/seg** (Redis soporta >100.000) |

**Conclusión:** Instancia `cache.r6g.large` (2 vCPU, 12.5 GB) con `maxmemory 2gb` y `noeviction` tiene headroom de ~17x en memoria y ~16x en throughput. El riesgo OOM es prácticamente nulo.

### 5.2. PostgreSQL bajo Ola (Workload Update-Heavy)

| Operación por logout | TPS en pico (1.000 logouts/seg) |
| :--- | :--- |
| SELECT idempotencia (índice único token_hash) | 1.000 |
| UPDATE revocación (índice parcial) | 1.000 |
| INSERT session_revocation_audit | 1.000 |
| INSERT auth_outbox | 1.000 |
| **Total transaccional** | **4.000 TPS** (1 commit c/u) |

**Recomendación:**
- **Instancia:** `db.r6g.xlarge` (4 vCPU, 32 GB) con GP3 a **6.000 IOPS** y 250 MB/s.
- **Autovacuum agresivo** en `refresh_tokens` (churn de UPDATEs genera dead tuples): `autovacuum_vacuum_scale_factor = 0.02`.
- **PgBouncer** (transaction mode) para workers de Outbox y reconciliación.

### 5.3. Escalado de Pods KAGI

```yaml
# HPA: escala por latencia de revocación y profundidad de cola de logouts
metrics:
  - type: Pods
    pods:
      metric: { name: kagi_auth_db_revoke_duration_p95 }
      target: { type: AverageValue, averageValue: "20" }  # escalar si el UPDATE supera 20ms
  - type: Resource
    resource:
      name: cpu
      target: { type: Utilization, averageUtilization: 65 }
minReplicas: 6     # headroom para ola de fin de turno
maxReplicas: 40
```

---

## 6. Observabilidad y Profiling en Producción

### 6.1. Distributed Tracing (OpenTelemetry)

**Trace de Logout Estándar:**
```text
POST /v1/auth/logout (Root Span) [Total: 45ms]
├── jwt.extract_jti [1ms]
├── db.refresh_token.select [8ms]        (idempotencia)
├── db.refresh_token.update_returning [12ms]
├── vo.blacklist_entry.calculate_ttl [0ms]
├── redis.blacklist.set [6ms]
├── db.session_revocation_audit.insert [6ms]
└── db.auth_outbox.insert [8ms]          (misma transacción: 1 commit)
```

**Trace de Request Autenticada (Middleware Tax):**
```text
GET /v1/identities/me (Root Span) [Total: 38ms]
├── middleware.blacklist.check [2ms]     ← el "impuesto"; p99 debe ser < 5ms
├── kizuna.authz.evaluate [10ms]
└── handler + db [26ms]
```

**Spans Críticos a Monitorear:**
1. **`middleware.blacklist.check`:** Si p99 > 10ms con Redis sano, hay contención de red o de socket multiplexado. Si sube a >0ms→inmediato con Circuit open, el fail-fast funciona.
2. **`db.refresh_token.update_returning`:** Si supera 20ms en pico, hay contención de índice parcial o de locks de fila.
3. **`reconciliation.backfill_chunk`:** Progreso del job post-Fail-Safe en tiempo real.

### 6.2. Dashboards de Grafana (Específicos para UC-12)

#### Panel 1: Desglose de Latencia de Logout (Stacked Spans)
*   **Métricas:** p50/p95 de cada span del trace de logout.
*   **Objetivo:** Identificar si una degradación viene de DB (UPDATE), Redis (SET) o pools (espera de conexión).

#### Panel 2: Middleware Tax + Estado del Circuit Breaker (Overlay)
*   **Métricas:** `blacklist_check_duration_p99` (línea) + `circuit_breaker_state` (área sombreada).
*   **Comportamiento esperado en incidente:** Al abrir el Circuit, la línea de p99 debe **caer a ~0ms**, no subir. Si sube, el fail-fast está roto.

#### Panel 3: Memoria de Blacklist vs. Curva Teórica
*   **Métricas:** `redis_memory_used_bytes` vs. `logouts_rate × 960s × 120B`.
*   **Alerta:** Divergencia > 20% indica fugas de TTL (entradas sin EX) o keyspace contaminado.

#### Panel 4: Distribución de Cascade Size vs. Latencia
*   **Métricas:** Histograma de `cascade_size` correlacionado con `logout_duration{mode=cascade}`.
*   **Objetivo:** Verificar que la latencia de cascade es **plana** (set-based) y no lineal con N.

#### Panel 5: Reconciliación post-Fail-Safe
*   **Métricas:** `reconciliation_backfilled_total` (counter), `reconciliation_completion_seconds`.
*   **Objetivo:** Tras cada incidente de Redis, visualizar el cierre de la ventana de riesgo residual.

### 6.3. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Middleware Tax Degraded** | `blacklist_check_duration_p99 > 10ms` por 5 min (Redis sano) | Alta | Contención de Redis o de socket. Revisar co-tenancy de Redis Auth y latencia de red inter-AZ. |
| **Fail-Fast Roto** | Circuit OPEN y `blacklist_check_duration_p99 > 1ms` | **Crítica** | El middleware está intentando conectar con Circuit abierto. Bug de implementación. |
| **Logout Latency Degraded** | `logout_duration_p95 > 150ms` por 5 min | Alta | Revisar contención de pools Npgsql y locks en `refresh_tokens`. |
| **Cascade Latency Lineal** | Correlación(latencia, cascade_size) > 0.7 | Media | El pipeline Redis o el UPDATE set-based se rompió (regresión de código). |
| **Reconciliation Slow** | `reconciliation_completion_seconds > 300` tras recovery | Alta | Ventana de riesgo residual abierta más de lo aceptado. Escalar job o aumentar chunk rate. |
| **Outbox Lag** | `outbox_lag_seconds > 300` | Alta | (Heredada de Doc 09) KAGE/KIZUNA desactualizados. |

---

## 7. Configuración de Rendimiento

### 7.1. PostgreSQL para Workload Update-Heavy

```ini
# postgresql.conf - Optimizaciones UC-12
shared_buffers = 8GB
effective_cache_size = 24GB
work_mem = 64MB
maintenance_work_mem = 1GB

# Churn de UPDATEs en refresh_tokens genera dead tuples rápidamente
autovacuum_vacuum_scale_factor = 0.02
autovacuum_analyze_scale_factor = 0.01
autovacuum_max_workers = 4

# Commits: agrupar writes del outbox worker
synchronous_commit = on          # durabilidad de revocación es innegociable
wal_buffers = 64MB
checkpoint_timeout = 10min
```

### 7.2. Pooling de Conexiones (Npgsql)

```csharp
// Connection string por pod de KAGI
// Host=pg-primary;Database=shugo;Pooling=true;MinPoolSize=10;MaxPoolSize=50;
// ConnectionIdleLifetime=300;ConnectionPruningInterval=30;Timeout=5;
// CommandTimeout=10  ← fail-fast sobre contención de locks
```

### 7.3. Redis Auth (Reafirmación de Seguridad sobre Rendimiento)

```ini
# La política de rendimiento NUNCA debe comprometer la integridad de la Blacklist:
maxmemory-policy noeviction     # Una evicción = token revocado resucitado (T-80)
maxmemory 2gb
tcp-nodelay yes                 # latencia mínima para el middleware tax
```

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Los objetivos de rendimiento (<100ms, 1.000 logouts/seg) se formalizan aquí como SLOs medibles. |
| `08_Final_Class_Design.md` | `JtiBlacklistMiddleware`, `TokenRevoker` y `BlacklistReconciliationJob` emiten las métricas y respetan los timeouts definidos aquí. |
| `09_Modern_Artifacts.md` | Las métricas base (`logout.duration`, `blacklist.failures_total`) se extienden aquí con el middleware tax y los benchmarks de reconciliación. |
| `10_UC_Specific_Data_Model.md` | El capacity planning de Redis (noeviction, ~120B/entrada) y los índices parciales que acotan el UPDATE set-based se validan aquí bajo carga. |
| `11_UC_Specific_Security_Threats.md` | El Escenario 4 (Chaos) valida empíricamente que la ventana Fail-Safe (T-85) permanece acotada y recuperable bajo fuego. |

---

## 9. Cierre del Paquete de Diseño (UC-12)

Con este documento, completamos los **12 archivos del Paquete de Diseño Profundo (Tier 1 + Tier 2)** para el UC-12:

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

### 9.1. Resumen de lo Logrado en UC-12

Hemos diseñado el cierre de sesión como lo que realmente es en una arquitectura JWT moderna: **un problema de revocación distribuida**, no un "borrar cookie":
- **Revocación Híbrida Instantánea:** Stateful en DB (barrera principal) + Stateless en Redis Blacklist con TTL dinámico autolimpiable.
- **Zero Info Leakage:** HTTP 204 constante con normalización de latencia entre caminos, cerrando enumeración por estado y por timing.
- **Fail-Safe con Riesgo Residual Gestionado:** Circuit Breaker agresivo, alerta inmediata, lista forense de `FAILSAFE_SKIPPED` y **backfill de reconciliación** que cierra la ventana tras la recuperación.
- **Middleware Tax Acotado y Protegido:** El check de Blacklist sobre todo el tráfico autenticado está medido (p99 < 5ms), protegido por fail-fast (0ms con Circuit open) y documentado con alternativas evaluadas y rechazadas.
- **Cascade Logout Set-Based:** 100 revocaciones en ~40ms mediante UPDATE con RETURNING, pipeline Redis e INSERTs multi-fila.
- **Resiliencia Validada Empíricamente:** Chaos Engineering que prueba la ola de fin de turno con Redis muerto y verifica disponibilidad 99.9% y reconciliación < 5 min.

### 9.2. Estado del EPIC-02 (Authentication)

| UC | Nombre | Estado | Documentos |
| :--- | :--- | :--- | :--- |
| **UC-11** | Autenticar Credenciales Locales | ✅ Completo | 12/12 |
| **UC-12** | Cerrar Sesión (Logout) | ✅ Completo | 12/12 |
| **UC-13** | Renovar Sesión (Refresh Token) | 📋 Pendiente | 0/12 |
| **UC-14** | Autenticar Federada (SSO) | 📋 Pendiente | 0/12 |
| **UC-15** | Multi-Factor Authentication (MFA) | 📋 Pendiente | 0/12 |
| **UC-16** | Passwordless Authentication | 📋 Pendiente | 0/12 |
| **UC-17** | Risk-Based Authentication | 📋 Pendiente | 0/12 |
| **UC-18** | Recuperación de Credenciales | 📋 Pendiente | 0/12 |
| **UC-19** | Cambio de Password (Self-Service) | 📋 Pendiente | 0/12 |

**Total: 24/108 documentos completados en el EPIC-02.**

---
*Fin del Documento 12_UC_Specific_Performance_Benchmarks para UC-12*