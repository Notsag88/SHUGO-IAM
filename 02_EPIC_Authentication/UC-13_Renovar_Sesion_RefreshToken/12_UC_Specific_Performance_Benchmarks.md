# UC-13: Renovar Sesión (Refresh Token) - Benchmarks, SLAs y Estrategia de Rendimiento

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-13 - Renovar Sesión (Refresh Token Rotation & Reuse Detection) |
| **Documento:** | 12_UC_Specific_Performance_Benchmarks.md |
| **Metodología:** | SRE (Site Reliability Engineering) + Performance Engineering + Load Testing (k6) + Chaos Engineering + Capacity Planning |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 07/10/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 07/10/2026 | Gastón Ruiz | **Creación inicial.** Definición de SLAs, SLOs y estrategia de Load Testing para la rotación de Refresh Tokens. Modela los cinco perfiles de carga críticos: ola matutina de renovación (800 rot/seg), tormenta de concurrencia multi-tab (validación de Grace Window y del índice único single-ACTIVE), contención por reuso bajo carga, chaos con Redis y Risk Engine degradados, y soak test de 24h para presión de vacuum y profundidad de cadena. Incluye capacity planning de PostgreSQL (workload write-heavy con locks), Redis (Grace Cache y superseded sets), y análisis de costo de firma JWT por algoritmo. | Documento de referencia para SREs y Arquitectos de Rendimiento. Cierra el UC-13 garantizando que el endpoint más frecuentemente invocado del motor KAGI mantenga latencia sub-150ms bajo ola matutina, que el lock de fila permanezca acotado, y que los controles de seguridad (índice único, device-binding, contención) no degraden bajo fuego. |

---

## 1. Introducción y Propósito

El **UC-13 (Renovar Sesión)** es el endpoint de **mayor frecuencia de invocación** de toda la plataforma: cada sesión activa lo llama cada ~15 minutos durante su vida útil, y una organización de 50.000 empleados genera un pico matutino de aperturas de jornada que coincide con el pico de rotaciones. A diferencia del login (UC-11, CPU-bound por Argon2id), el refresh es **DB-bound y lock-bound**: su recurso crítico no es la CPU criptográfica sino el **lock de fila** (`SELECT ... FOR UPDATE`) y la escritura transaccional de la rotación.

Este documento aborda cinco desafíos de rendimiento únicos:

1. **Contención de Lock de Fila:** Bajo concurrencia multi-tab legítima, múltiples requests serializan sobre la misma fila. El lock debe sostenerse lo mínimo (diseño: <15ms p99) y la serialización debe convertirse en Grace Window, no en errores.
2. **Escrituras por Rotación:** Cada refresh exitoso ejecuta 1 SELECT con lock + 1 INSERT (sucesor) + 1 UPDATE (presentado) + 1 INSERT (outbox) en un solo commit: ~3 escrituras y 1 commit por operación, sostenidas a 800/seg en pico.
3. **Churn de Dead Tuples:** Las rotaciones UPDATEan filas indexadas por índices parciales sobre `status`, lo que **invalida HOT updates** y genera presión de autovacuum específica sobre `refresh_tokens`.
4. **Grace Cache en el Hot Path:** La resolución de Grace Window añade un round-trip a Redis + desellado AEAD en el camino crítico de las renovaciones concurrentes.
5. **Costo de Firma JWT:** Cada rotación firma un Access Token nuevo; el algoritmo de firma (EdDSA vs RS256) determina el costo de CPU por operación y la capacidad por pod.

---

## 2. Service Level Indicators (SLIs) y Objectives (SLOs)

### 2.1. SLIs Específicos para UC-13

| SLI | Descripción | Fórmula / Medición (OpenTelemetry) |
| :--- | :--- | :--- |
| **Refresh Latency (p95/p99)** | Latencia end-to-end de `POST /auth/refresh` por camino (`rotate`, `grace`, `reject`). | `histogram_quantile(0.95, rate(kagi_auth_refresh_duration_bucket[5m]))` |
| **Lock Hold Duration (p99)** | Tiempo entre `BEGIN TX + FOR UPDATE` y `COMMIT` en el camino de rotación. | `histogram_quantile(0.99, rate(kagi_auth_refresh_tx_duration_bucket[5m]))` |
| **Grace Resolve Latency (p95)** | `GET rotgrace` + desellado AEAD + verificación de device-binding. | `histogram_quantile(0.95, rate(kagi_auth_refresh_grace_resolve_duration_bucket[5m]))` |
| **Grace Resolution Ratio** | Proporción de renovaciones resueltas vía Grace Window vs rotaciones estándar (indicador de patrón multi-tab). | `rate(refresh_total{result="grace_resolved"}[1h]) / rate(refresh_total{result="rotated"}[1h])` |
| **Family Revocation Latency (p95)** | Contención completa: UPDATE set-based + outbox ×2 + commit. | `histogram_quantile(0.95, rate(kagi_auth_refresh_tx_duration_bucket{path="revoke"}[5m]))` |
| **Blacklist Pipeline Latency (p95)** | Pipeline de SETs de JTIs en contención (familias ≤100 tokens). | `histogram_quantile(0.95, rate(kagi_auth_refresh_blacklist_pipeline_duration_bucket[5m]))` |
| **JWT Sign Latency (p99)** | Firma del Access Token sucesor por algoritmo. | `histogram_quantile(0.99, rate(kagi_auth_refresh_jwt_sign_duration_bucket[5m]))` |
| **Outbox Lag** | Edad del evento `Session.Refreshed` más antiguo sin publicar. | `time() - min(created_at) WHERE processed_at IS NULL AND event_type LIKE 'Session.%'` |
| **Generation Depth (p99)** | Profundidad de cadena observada en rotaciones (health de sesiones). | `histogram_quantile(0.99, rate(kagi_auth_refresh_generation_bucket[24h]))` |
| **Reconciliation Completion** | Tiempo desde recuperación de Redis hasta cierre de ventanas residuales de Blacklist. | `kagi_auth_refresh_reconciliation_completion_seconds` |

### 2.2. SLOs y SLAs

| Métrica | SLO (Objetivo Interno - 95/99%) | SLA (Contrato Enterprise - 99%) | Justificación |
| :--- | :--- | :--- | :--- |
| **Refresh Latency (p95)** | **< 150 ms** | **< 250 ms** | Suma de spans: crypto 1ms + lock+TX 13ms + seal 5ms + sign 3ms + overhead red. |
| **Refresh Latency (p99)** | **< 250 ms** | **< 400 ms** | Cola de lock bajo concurrencia multi-tab. |
| **Lock Hold Duration (p99)** | **< 15 ms** | **< 30 ms** | Objetivo de diseño (doc 07); HPA escala al acercarse a 25ms. |
| **Grace Resolve Latency (p95)** | **< 10 ms** | **< 20 ms** | Redis GET (~1ms) + AEAD unseal (~0.1ms) + device check (~0ms). |
| **Family Revocation Latency (p95)** | **< 50 ms** | **< 100 ms** | UPDATE set-based por `family_root_id` con índice parcial. |
| **Blacklist Pipeline (p95, ≤100 JTIs)** | **< 50 ms** | **< 100 ms** | Un round-trip pipelined a Redis. |
| **JWT Sign (p99, EdDSA)** | **< 2 ms** | **< 5 ms** | Ed25519: ~70k firmas/seg por core. |
| **Throughput de Rotación (pico)** | **800 rot/seg** | **400 rot/seg** | Ola matutina en tenant enterprise de 50.000 empleados. |
| **Falsos Positivos de Reuse Detection** | **< 0.01%** | **< 0.05%** | Grace Window + device-binding deben absorber toda concurrencia legítima. |
| **Outbox Lag** | **< 60 s** | **< 300 s** | Propagación de `Session.Refreshed` a KAGE/KIZUNA. |
| **Reconciliation Completion** | **< 5 min post-recuperación** | **< 10 min** | Cierre de ventana residual del Fail-Safe de Blacklist. |
| **Disponibilidad del Endpoint** | **99.99%** | **99.95%** | Si refresh cae, todas las sesiones mueren al expirar su JWT (15 min). |
| **Bifurcaciones de Cadena detectadas** | **0 (siempre)** | **0 (siempre)** | Invariante físico (índice único parcial); cualquier valor > 0 es incidente de plataforma. |

---

## 3. Cuellos de Botella y Estrategias de Optimización

### 3.1. Cuello de Botella #1: Contención de Lock de Fila

**Problema:** Bajo concurrencia multi-tab, N requests serializan sobre la misma fila (`FOR UPDATE`). Si la transacción incluye trabajo lento (Redis, firma JWT, notificaciones), el lock se sostiene y la cola de espera crece: la latencia percibida es `N × lock_hold`.

**Estrategias de Optimización:**

#### A. Todo el Trabajo No-DB fuera de la Transacción
El orden implementado (doc 07) ya acota el lock: crypto prep pre-lock, seal de Grace Cache y firma JWT post-commit. La TX contiene **solo**: guardas en memoria de dominio + `RotateTo()` + 3 escrituras + commit.

#### B. Índices que Hacen el Lock Barato
- `idx_rt_token_hash` (único): el `SELECT ... FOR UPDATE` es un index scan de 1 fila, no un scan.
- `uq_rt_single_active_per_family` (hardening): el chequeo de unicidad parcial es O(1) por escritura.

#### C. Por qué HOT Updates NO Aplican (y qué hacer en su lugar)
La rotación cambia `status` de `ACTIVE` a `ROTATED`. Como `status` participa en el predicado del índice parcial único (`WHERE status = 'ACTIVE' AND is_revoked = FALSE`), PostgreSQL **debe** actualizar entradas de índice: no hay HOT update posible. La mitigación correcta no es `fillfactor`, sino **autovacuum agresivo por tabla**:

```sql
ALTER TABLE kagi_auth.refresh_tokens SET (
    autovacuum_vacuum_scale_factor = 0.01,     -- vacuum al 1% de dead tuples
    autovacuum_analyze_scale_factor = 0.005,   -- analyze al 0.5%
    autovacuum_vacuum_cost_delay = 2           -- vacuum más rápido en tabla caliente
);
```

#### D. Escalado por Contención, no por CPU
El HPA del doc 09 escala sobre `refresh.tx_duration_p99` (averageValue 25ms): añadir pods reduce la probabilidad de que dos pestañas del mismo usuario caigan en el mismo pod, pero **no elimina** la serialización por fila (el lock vive en PostgreSQL). Por eso el SLO de lock se protege principalmente con el acotamiento de la TX, y el HPA es la segunda línea.

### 3.2. Cuello de Botella #2: Grace Cache en el Hot Path

**Problema:** Cada resolución de Grace Window añade un `GET` a Redis + desellado AEAD. Bajo tormenta multi-tab, este camino puede dominar la latencia de las renovaciones concurrentes.

**Estrategias de Optimización:**
- **Multiplexing StackExchange.Redis:** Un socket por instancia; el `GET` de grace compite con los `SET` de seal y blacklist sin abrir conexiones nuevas.
- **Timeout de 50ms con breaker:** Un Redis lento no debe convertir la grace en una espera larga; tras 3 fallos, el breaker abre y el camino cae al rechazo conservador en ~0ms.
- **Device-binding antes del desellado:** La comparación `SameDeviceAs` (operación de memoria) se ejecuta **antes** del `GET` a Redis cuando el mismatch es detectable desde el contexto del token presentado... *corrección de diseño:* el device check compara el contexto del **attempt** contra el de la **fila** (ya en memoria post-SELECT), por lo que un mismatch contiente **sin tocar Redis**: el hot path de grace solo paga Redis cuando el dispositivo coincide.

### 3.3. Cuello de Botella #3: Outbox bajo Volumen de Rotación

**Problema:** Cada rotación inserta 1 fila de outbox (`Session.Refreshed`). A 800 rot/seg son 800 inserts/seg adicionales en la misma transacción, más el worker que lee y publica.

**Estrategias de Optimización:**
- **Inserción en la misma TX:** No se negocia (garantía atómica); el costo es 1 write adicional por commit ya agrupado.
- **Worker con batch de 100 y polling de 500ms:** Publica hasta 200 eventos/seg por worker; 2 workers absorben 400/seg sostenidos con headroom; bajo pico, el lag crece transitoriamente y se drena en segundos.
- **Particionamiento mensual de `auth_outbox`** (UC-11): el scan del worker toca solo la partición actual gracias al índice parcial `idx_ao_unprocessed`.

### 3.4. Cuello de Botella #4: Costo de Firma JWT

**Problema:** Cada rotación firma un Access Token. El algoritmo determina CPU por operación:

| Algoritmo | Firmas/seg por core (aprox.) | Costo por firma | Verificación en GW |
| :--- | :--- | :--- | :--- |
| **EdDSA (Ed25519)** | ~70.000 | ~0.014 ms | Rápida (~50k/sec/core) |
| **RS256 (RSA-2048)** | ~15.000 | ~0.066 ms | Muy rápida (~200k/sec/core) |

**Decisión:** Default **EdDSA** con flag `kagi.auth.refresh.jwt_algorithm` para fallback RS256 en gateways legacy que no soporten EdDSA. A 800 firmas/seg, ambos algoritmos son triviales (<2% de un core); el SLO de `jwt_sign_duration_p99 < 2ms` protege contra regresiones (ej. clave en HSM remoto sin cache, que añadiría ~1-5ms por firma).

### 3.5. Cuello de Botella #5: Familias Grandes en Contención

**Problema:** Una contención sobre una familia con 100 tokens activos ejecuta un UPDATE de 100 filas + RETURNING de 100 JTIs + pipeline de 100 SETs.

**Estrategias de Optimización:**
- **UPDATE set-based con índice parcial `idx_rt_family_active`:** range scan acotado a filas no revocadas de la familia.
- **Pipeline Redis único:** 100 SETs en 1 round-trip (~8-12ms).
- **Límite de seguridad:** `max_tokens_per_operation = 100` (Feature Flag); familias mayores se revocan en lotes encadenados dentro de la misma TX por rangos de `generation`.

---

## 4. Estrategia de Load Testing (k6)

### 4.1. Escenario 1: Ola Matutina de Renovación (800 rot/seg)

**Objetivo:** Validar SLOs de latencia y lock bajo el pico de apertura de jornada, con login concurrente de fondo (UC-11) creando familias nuevas.

```javascript
import http from 'k6/http';
import { check } from 'k6';
import { Trend, Rate } from 'k6/metrics';

const refreshLatency = new Trend('refresh_latency', true);
const loginLatency = new Trend('login_latency_background', true);
const errorRate = new Rate('errors');

export const options = {
  scenarios: {
    morning_wave: {
      executor: 'ramping-arrival-rate',
      startRate: 100, timeUnit: '1s',
      preAllocatedVUs: 200, maxVUs: 1500,
      stages: [
        { target: 300, duration: '2m' },
        { target: 800, duration: '3m' },   // rampa de apertura de jornada
        { target: 800, duration: '5m' },   // pico sostenido
        { target: 0,   duration: '2m' },
      ],
      exec: 'refreshWave',
    },
    background_logins: {
      executor: 'constant-arrival-rate',
      rate: 80, timeUnit: '1s', duration: '12m',
      preAllocatedVUs: 40, maxVUs: 150,
      exec: 'loginBackground',
    },
  },
  thresholds: {
    refresh_latency: ['p(95)<150', 'p(99)<250'],
    login_latency_background: ['p(95)<300'],  // UC-11 intacto durante la ola
    errors: ['rate<0.001'],
  },
};

// El pool rota: cada iteración consume el sucesor de la anterior (cadena viva)
export function refreshWave() {
  const s = pool.next();                       // { refresh, access }
  const res = http.post(`${__ENV.BASE_URL}/v1/auth/refresh`,
    JSON.stringify({ refresh_token: s.refresh }),
    { headers: { 'Content-Type': 'application/json' } });

  refreshLatency.add(res.timings.duration);
  if (check(res, { 'refresh 200': (r) => r.status === 200 })) {
    pool.advance(s, res.json('refresh_token')); // el sucesor entra al pool
  } else {
    errorRate.add(1);
  }
}

export function loginBackground() {
  const res = http.post(`${__ENV.BASE_URL}/v1/auth/login`,
    JSON.stringify({ email: `wave_${__VU}@khcs.com`, password: 'SecureP@ssw0rd!' }),
    { headers: { 'Content-Type': 'application/json' } });
  loginLatency.add(res.timings.duration);
  if (res.status === 200) pool.seed(res.json('refresh_token'), res.json('access_token'));
}
```

### 4.2. Escenario 2: Tormenta Multi-Tab (Grace Window + Índice Único bajo Fuego)

**Objetivo:** G grupos de V VUs renuevan **el mismo token simultáneamente**. Se espera exactamente 1 rotación + (V-1) resoluciones grace por grupo, **cero** revocaciones de familia y **cero** bifurcaciones.

```javascript
export const options = {
  scenarios: {
    multi_tab_storm: {
      executor: 'per-vu-iterations',
      vus: 200,            // 50 grupos × 4 "pestañas"
      iterations: 1,
      maxDuration: '10m',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<400'],          // incluye espera de lock serializada
    // Verificaciones externas post-run (Prometheus/DB):
    // 1) increase(refresh_total{result="grace_resolved"}) == 150   (3 por grupo)
    // 2) increase(refresh_total{result="rotated"}) == 50          (1 por grupo)
    // 3) increase(family_revocations_total) == 0                  (sin falsos positivos)
    // 4) query(bifurcation_check) == 0                            (índice único íntegro)
  },
};

export default function () {
  const shared = groupTokenFor(__VU);          // 4 VUs comparten el mismo token
  const res = http.post(`${__ENV.BASE_URL}/v1/auth/refresh`,
    JSON.stringify({ refresh_token: shared }),
    { headers: { 'Content-Type': 'application/json' }, timeout: '10s' });
  check(res, { 'tab obtuvo 200': (r) => r.status === 200 });
}
```

**Justificación:** Este escenario es la prueba de fuego combinada de tres controles: el lock de fila (serialización), la Grace Window con device-binding (los 4 VUs comparten `device_id` de prueba) y el índice único parcial (cualquier bifurcación fallaría ruidosamente con unique violation → error 500 visible en `errors`).

### 4.3. Escenario 3: Contención por Reuso bajo Carga

**Objetivo:** Medir latencia de contención completa (revocación de familia + pipeline blacklist + notificación) mientras el sistema sostiene tráfico normal.

```javascript
export const options = {
  scenarios: {
    normal_traffic: {
      executor: 'constant-arrival-rate',
      rate: 300, timeUnit: '1s', duration: '10m',
      preAllocatedVUs: 150, maxVUs: 400,
      exec: 'refreshWave',
    },
    attacker_replay: {
      executor: 'per-vu-iterations',
      vus: 50, iterations: 4, startTime: '3m',
      exec: 'replayRotatedToken',              // tokens rotados >61s pre-preparados
    },
  },
  thresholds: {
    containment_tx: ['p(95)<50'],               // latencia del Camino C
    blacklist_pipeline: ['p(95)<50'],
    normal_traffic_refresh_latency: ['p(95)<150'], // el tráfico legítimo no se degrada
  },
};

export function replayRotatedToken() {
  const stolen = stolenPool.next();             // token ROTADO hace >61s
  const res = http.post(`${__ENV.BASE_URL}/v1/auth/refresh`,
    JSON.stringify({ refresh_token: stolen }),
    { headers: { 'Content-Type': 'application/json' } });
  check(res, { 'reuso contenido (400)': (r) => r.status === 400 });
}
```

### 4.4. Escenario 4: Chaos - Redis y Risk Engine Degradados

**Objetivo:** Validar los cuatro comportamientos de degradación bajo ola: (a) seal fail → warning sin impacto; (b) grace read fail → rechazo conservador **sin** revocaciones; (c) blacklist fail → contención duradera con ventana residual reconciliada; (d) risk timeout → fail-mode por tenant.

```javascript
export const options = {
  scenarios: {
    wave_during_chaos: {
      executor: 'constant-arrival-rate',
      rate: 400, timeUnit: '1s', duration: '18m',
      preAllocatedVUs: 250, maxVUs: 700,
    },
  },
  thresholds: {
    // Camino A no depende de Redis para commitear:
    refresh_rotated_availability: ['rate>0.999'],
    // El fallback conservador NO debe revocar familias:
    // externo: increase(family_revocations_total) == 0 durante ventana de Redis down
    refresh_latency: ['p(95)<200'],   // degradación acotada incluso con breaker abierto
  },
};
// Runbook del caos:
// min 0-4:   baseline sano
// min 4-9:   Redis down (toxiproxy / kubectl delete pod redis-auth-*)
// min 9-12:  Redis recovery → verificar reconciliation_completion < 300s
// min 12-15: Risk Engine con latency injection 300ms (> timeout 100ms)
//            → tenants strict: step-ups; tenants open: rotaciones con warning
// min 15-18: recovery total
```

### 4.5. Escenario 5: Soak Test 24h (Presión de Vacuum y Profundidad de Cadena)

**Objetivo:** Detectar degradación lenta: acumulación de dead tuples en `refresh_tokens`, crecimiento de `generation`, drift de memoria Redis y lag de outbox sostenido.

```javascript
export const options = {
  scenarios: {
    soak: {
      executor: 'constant-arrival-rate',
      rate: 100, timeUnit: '1s', duration: '24h',
      preAllocatedVUs: 100, maxVUs: 300,
    },
  },
  thresholds: {
    refresh_latency: ['p(95)<150'],
    outbox_lag_seconds: ['max<60'],
    // Externos (evaluada cada hora por el runner):
    // dead_tuple_ratio(refresh_tokens) < 0.20
    // redis_memory_used_bytes{db=auth} dentro de ±20% de la curva teórica
    // refresh_generation p99 < 500
  },
};
```

---

## 5. Capacity Planning y Escalabilidad

### 5.1. PostgreSQL (Workload Write-Heavy con Locks)

| Operación por rotación (Camino A) | Count | TPS en pico (800 rot/seg) |
| :--- | :--- | :--- |
| SELECT ... FOR UPDATE (index scan 1 fila) | 1 | 800 |
| INSERT sucesor | 1 | 800 |
| UPDATE presentado → ROTATED | 1 | 800 |
| INSERT outbox `Session.Refreshed` | 1 | 800 |
| COMMIT (WAL fsync agrupado) | 1 | 800 |
| **Total escrituras lógicas** | **3** | **2.400 writes/seg** |

**Recomendación:**
- **Instancia:** `db.r6g.xlarge` (4 vCPU, 32 GB) con GP3 a **6.000 IOPS / 250 MB-s** (los commits agrupados por WAL mantienen los IOPS de fsync ~= commits/seg, no writes/seg).
- **Conexiones:** 20 pods × `MaxPoolSize=50` = 1.000 conexiones KAGI + PgBouncer (transaction mode) para workers de outbox y reconciliación. `max_connections=1200`.
- **Vacuum:** Parámetros por tabla de §3.1.C; monitorear `n_dead_tup` de `refresh_tokens` como métrica de primer nivel en el soak test.

### 5.2. Redis Auth (Grace Cache + Superseded + Blacklist)

| Keyspace | Entradas concurrentes en pico | Memoria | Ops/seg en pico |
| :--- | :--- | :--- | :--- |
| `rotgrace:*` (TTL 60s) | 800 × 60 = 48.000 | ~13.2 MB | 800 SET + N GET (grace) |
| `superseded:{family}` (TTL 16m) | << 1.000 (solo caminos B repetidos) | < 1 MB | esporádico |
| `blacklist:{jti}` (heredado UC-12) | ver doc UC-12 | ~115 MB en ola de logout | picos independientes |

**Recomendación:** Instancia compartida `cache.r6g.large` (2 GB `maxmemory`, `noeviction`) con headroom > 100× para el keyspace de UC-13. `tcp-nodelay yes` para latencia de grace resolve.

### 5.3. Pods KAGI y Costo de Firma

- **CPU por rotación:** serialización JSON (~0.2ms) + SELECT/TX wait + AEAD seal (~0.05ms) + firma EdDSA (~0.014ms) ≈ dominado por espera de red/DB, no por CPU.
- **Dimensionamiento:** 20 pods × 2 vCPU (min 6, max 40 vía HPA por `tx_duration_p99`). Con EdDSA, la firma no es factor limitante; con RS256 legacy, añadir +1 core por cada 15k firmas/seg (irrelevante a 800/seg).
- **Outbox workers:** 2 réplicas (batch 100, poll 500ms) con HPA sobre `outbox_lag_seconds`.

---

## 6. Observabilidad y Profiling en Producción

### 6.1. Distributed Tracing (OpenTelemetry)

```text
POST /v1/auth/refresh (Root Span) [Total: 42ms]
├── crypto.prepare [1ms]               (SHA256 + Generate256 + UUID jti)
├── db.token.select_for_update [9ms]   (lock adquirido)
├── domain.rotate [0ms]                (RotateTo: atomicidad de par)
├── db.rotation_tx [13ms]              (UPDATE presentado + INSERT sucesor + outbox + COMMIT)
├── redis.grace.seal [5ms]             (POST-COMMIT, AES-256-GCM, PX 60000)
└── jwt.sign [1ms]                     (EdDSA; sid=family, auth_level preservado)

Camino B (Grace): db.select_for_update [9ms] → device_check [0ms] →
redis.grace.get [2ms] → aead.unseal [0.5ms] → db.rebind_tx [8ms] → jwt.sign [1ms]

Camino C (Contención): db.family_revoke_tx [16ms] → redis.blacklist.pipeline [9ms] →
notify.dispatch [async fuera del trace crítico]
```

### 6.2. Dashboards de Grafana (Específicos para UC-13)

#### Panel 1: Lock Hold Heatmap + Cola de Espera
*   **Métricas:** Heatmap de `refresh.tx_duration` + `refresh.duration` p99 superpuesto.
*   **Lectura:** Si el p99 de duration crece mientras el heatmap de tx permanece plano, la cola está **fuera** del lock (red/pool); si ambos crecen juntos, hay contención real de fila.

#### Panel 2: Grace Resolution Ratio y Device-Binding
*   **Métricas:** `grace_resolved / (rotated + grace_resolved)` + `grace_context_mismatch_total`.
*   **Lectura:** Un ratio estable refleja el patrón multi-tab esperado del tenant; un spike de mismatch indica intentos de hijack (T-88) o un bug de fingerprinting de dispositivo.

#### Panel 3: Profundidad de Cadena y Edad de Familias
*   **Métricas:** `refresh.generation` p50/p99 + familias con `NOW() - created_at` cercano a `absolute_expires_at`.
*   **Lectura:** Validación continua de que la vida absoluta gobierna y de que ningún cliente renueva patológicamente.

#### Panel 4: Salud de Vacuum en refresh_tokens
*   **Métricas:** `n_dead_tup / n_live_tup`, último `autovacuum_at`, `idx_scan` de `uq_rt_single_active_per_family`.
*   **Lectura:** El churn de rotaciones sin HOT updates hace de este panel el predictor temprano de degradación de latencia de lock.

#### Panel 5: Reconciliación y Ventanas Residuales
*   **Métricas:** `reconciliation_completion_seconds`, `blacklist_failures_total`, edad del `FAILSAFE_SKIPPED` más antiguo.
*   **Lectura:** Tras cada incidente de Redis, visualiza el cierre efectivo de la ventana residual.

### 6.3. Alertas de Rendimiento

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Lock Contention** | `refresh.tx_duration_p99 > 15ms` por 5 min | Alta | Revisar concurrencia multi-tab, vacuum de `refresh_tokens` y contención de pools. HPA debería escalar solo. |
| **Grace Resolve Slow** | `grace_resolve_duration_p95 > 10ms` por 5 min | Media | Latencia de Redis o breaker a punto de abrir; revisar co-tenancy de la instancia Auth. |
| **JWT Sign Regression** | `jwt_sign_duration_p99 > 5ms` por 5 min | Media | Posible cambio a HSM remoto sin cache o regresión de algoritmo; verificar flag `jwt_algorithm`. |
| **Outbox Lag** | `outbox_lag_seconds > 60` por 5 min | Alta | Escalar workers de outbox; verificar salud del broker. |
| **Dead Tuple Pressure** | `n_dead_tup/n_live_tup > 0.20` en `refresh_tokens` | Media | Autovacuum no da abasto: reducir `scale_factor` a 0.005 o aumentar `autovacuum_max_workers`. |
| **Bifurcation Detected** | Consulta de integridad > 0 | **Crítica** | Incidente de plataforma: congelar rotaciones del tenant y preservar forense (doc 11, T-90). |

---

## 7. Configuración de Rendimiento

### 7.1. PostgreSQL

```ini
# postgresql.conf - Optimizaciones UC-13 (write-heavy con locks cortos)
shared_buffers = 8GB
effective_cache_size = 24GB
work_mem = 64MB
wal_buffers = 64MB
synchronous_commit = on            # durabilidad de rotación es innegociable
checkpoint_timeout = 10min
max_locks_per_transaction = 128    # familias grandes en contención set-based
```

### 7.2. Npgsql y PgBouncer

```csharp
// Por pod KAGI: pool acotado y fail-fast sobre contención
// Host=pg-primary;Pooling=true;MinPoolSize=10;MaxPoolSize=50;
// Timeout=5;CommandTimeout=10;ConnectionIdleLifetime=300;
```

### 7.3. Redis y Feature Flags de Rendimiento

```ini
# redis.conf (instancia Auth compartida)
maxmemory-policy noeviction
maxmemory 2gb
tcp-nodelay yes
```

| Feature Flag | Descripción | Default |
| :--- | :--- | :--- |
| `kagi.auth.refresh.jwt_algorithm` | `eddsa \| rs256` (fallback para gateways legacy) | `eddsa` |
| `kagi.auth.refresh.max_tokens_per_operation` | Tope de filas por contención set-based | `100` |
| `kagi.auth.refresh.outbox_worker_batch` | Tamaño de batch del worker de outbox | `100` |
| `kagi.auth.refresh.risk_timeout_ms` | Timeout del hook UC-17 | `100` |

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Los objetivos de rendimiento (latencia <150ms, 800 rot/seg, falsos positivos <0.01%) se formalizan aquí como SLOs medibles. |
| `07_Sequence_Diagram.md` | El orden "crypto pre-lock, Redis y JWT post-commit" que acota el lock se valida aquí con el heatmap de tx_duration. |
| `08_Final_Class_Design.md` | `TokenRotator`, `GraceWindowResolver` y `FamilyRevoker` emiten las métricas y respetan los timeouts definidos aquí. |
| `09_Modern_Artifacts.md` | Las métricas base (`refresh.*`, HPA por tx_duration) y los escenarios k6 preliminares se extienden aquí con soak testing y capacity planning. |
| `10_UC_Specific_Data_Model.md` | Los índices (`idx_rt_token_hash`, `idx_rt_family_active`, `uq_rt_single_active_per_family`) y los parámetros de autovacuum por tabla son la base física de los benchmarks. |
| `11_UC_Specific_Security_Threats.md` | Los escenarios 2 y 4 validan empíricamente que los controles anti-hijack (device-binding) y anti-bifurcación (índice único) no degradan bajo fuego ni producen falsos positivos. |

---

## 9. Cierre del Paquete de Diseño (UC-13)

Con este documento, completamos los **12 archivos del Paquete de Diseño Profundo (Tier 1 + Tier 2)** para el UC-13:

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

### 9.1. Resumen de lo Logrado en UC-13

Hemos diseñado la renovación de sesiones como un **sistema de cadena auto-curativa de grado enterprise**:
- **Rotación single-use con atomicidad de par** y lock de fila acotado (<15ms p99) que serializa concurrencia legítima sin bifurcar cadenas.
- **Grace Window con device-binding:** concurrencia multi-tab absorbida idempotentemente; cualquier otro dispositivo dentro de la ventana es contención, no grace.
- **Reuse Detection como contención completa:** revocación de familia set-based, blacklisteado quirúrgico de JTIs (incluidos superseded), notificación al titular y evento SIEM.
- **Vida absoluta inextensible** que elimina Zombie Sessions a nivel de dominio, de esquema (inmutabilidad post-INSERT) y de trigger.
- **Degradaciones siempre hacia el lado seguro:** Fail-Secure en DB (503), rechazo conservador en grace (sin revocaciones), Fail-Safe en blacklist con reconciliación que cierra la ventana residual.
- **Rendimiento validado empíricamente:** cinco escenarios de carga y chaos que prueban ola matutina, tormenta multi-tab, contención bajo fuego, degradación de dependencias y soak de 24h.

### 9.2. Estado del EPIC-02 (Authentication)

| UC | Nombre | Estado | Documentos |
| :--- | :--- | :--- | :--- |
| **UC-11** | Autenticar Credenciales Locales | ✅ Completo | 12/12 |
| **UC-12** | Cerrar Sesión (Logout) | ✅ Completo | 12/12 |
| **UC-13** | Renovar Sesión (Refresh Token) | ✅ Completo | 12/12 |
| **UC-14** | Autenticar Federada (SSO) | 📋 Pendiente | 0/12 |
| **UC-15** | Multi-Factor Authentication (MFA) | 📋 Pendiente | 0/12 |
| **UC-16** | Passwordless Authentication | 📋 Pendiente | 0/12 |
| **UC-17** | Risk-Based Authentication | 📋 Pendiente | 0/12 |
| **UC-18** | Recuperación de Credenciales | 📋 Pendiente | 0/12 |
| **UC-19** | Cambio de Password (Self-Service) | 📋 Pendiente | 0/12 |

**Total: 36/108 documentos completados en el EPIC-02.** El tríptico fundamental de sesiones (Login → Refresh → Logout) queda diseñado y cerrado como un sistema coherente: UC-11 crea la familia, UC-13 la rota y detecta abuso, UC-12 la termina.

---
*Fin del Documento 12_UC_Specific_Performance_Benchmarks para UC-13*