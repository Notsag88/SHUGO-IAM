# UC-15: Multi-Factor Authentication (MFA) - Benchmarks, SLAs y Estrategia de Rendimiento

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-15 - Multi-Factor Authentication (MFA - TOTP, WebAuthn/FIDO2, Push, Recovery) |
| **Documento:** | 12_UC_Specific_Performance_Benchmarks.md |
| **Metodología:** | SRE (Site Reliability Engineering) + Performance Engineering + Load Testing (k6) + Chaos Engineering + Virtual Authenticators |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 23/10/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | [23/10/2026] | Gastón Ruiz | **Creación inicial.** Definición de SLAs, SLOs y estrategia de Load Testing para el segundo factor. Modela los siete perfiles de carga críticos: ola matutina de MFA con mezcla de métodos, costo criptográfico por método (incluido el dummy HMAC anti-enumeración), tormenta de ceremonies WebAuthn con presión de counter updates, soak de push con latencia humana simulada y carga de polling, comportamiento de throttling/fatiga a escala, chaos con fallo quirúrgico de counter update, y soak de 24h con drift de keyspaces y particiones. Incluye capacity planning de CPU ponderado por mezcla de métodos y el tratamiento SRE de la aprobación push como métrica de funnel UX, no de latencia. | Documento de referencia para SREs y Arquitectos de Rendimiento. Cierra el UC-15 garantizando que el cuello de completitud de toda la autenticación (login, refresh, federación pausada) verifique en <30ms TOTP / <80ms WebAuthn bajo ola, que la detección de seguridad (clone, replay, fatigue) no degrade el camino caliente, y que la interacción humana de push se mida como funnel, no como latencia de servidor. |

---

## 1. Introducción y Propósito

El **UC-15 (Multi-Factor Authentication)** tiene el perfil de rendimiento más singular del motor KAGI por cinco razones estructurales:

1. **Es el cuello de completitud de toda la autenticación:** Cada login pausado (UC-11), renovación pausada (UC-13) y federación pausada (UC-14) termina aquí. La latencia del verify se suma directamente a la percepción de "el login es lento".
2. **Costos criptográficos heterogéneos y pequeños:** TOTP cuesta ~0.3 ms (3 HMAC), WebAuthn ~1 ms (ECDSA verify), recovery ~0.1 ms (SHA256 compare). El presupuesto dominante no es CPU sino **round-trips** (Redis GETDEL, KAGAMI enrollment fetch, counter update).
3. **Un tramo mediado por humano:** La aprobación push no tiene SLO de latencia — tiene SLO de **funnel** (porcentaje aprobadas antes de expirar). Medirla como latencia de servidor sería un error de categoría.
4. **El dummy HMAC anti-enumeración es costo permanente:** Cada verificación sin enrollment paga un HMAC de igualación (T-116). Debe verificarse que el costo del camino hostil no difiera del legítimo ni abra asimetrías explotables.
5. **Alta disponibilidad = alta disponibilidad del login:** Si MFA cae, los flujos pausados no se completan. El fail-secure preserva pausas (los usuarios reintentan), pero la disponibilidad local debe ser 99.99%.

---

## 2. Service Level Indicators (SLIs) y Objectives (SLOs)

### 2.1. SLIs Específicos para UC-15

| SLI | Descripción | Fórmula / Medición (OpenTelemetry) |
| :--- | :--- | :--- |
| **Challenge Latency (p95)** | Emisión de desafío local (validación de contexto + política + store de challenge). | `histogram_quantile(0.95, rate(kagi_auth_mfa_challenge_duration_bucket[5m]))` |
| **Verify Latency por Método (p95/p99)** | Verificación local completa (consume → throttle → criptografía → evidence), excluida interacción humana. | `histogram_quantile(0.95, rate(kagi_auth_mfa_verify_duration_bucket{method}[5m]))` |
| **Completion Duration por PauseKind (p95)** | Latencia del resolutor de pausa (emisión de tokens/elevación) tras evidencia. | `histogram_quantile(0.95, rate(kagi_auth_mfa_completion_duration_bucket{pause_kind}[5m]))` |
| **Push Approval Funnel** | % de challenges push aprobados antes de expirar; distribución del tiempo de decisión humana. | `approved_total / issued_total`; `histogram_quantile(0.5, push_approval_duration_bucket)` |
| **Push Status Poll Latency (p95)** | Endpoint de observación del challenge (lectura pura). | `histogram_quantile(0.95, rate(kagi_auth_mfa_push_poll_duration_bucket[5m]))` |
| **Enrollment Activation (p95)** | Verificación de prueba + CAS PENDING→ACTIVE + lote de recovery. | `histogram_quantile(0.95, rate(kagi_auth_mfa_enrollment_activation_duration_bucket[5m]))` |
| **MFA Availability** | 1 − (503s / verifies totales); pausa preservada en fallos. | `1 - rate(unavailable_total[5m]) / rate(verify_total[5m])` |
| **Security Detection Rate** | Detecciones por hora (clone, replay, token_reuse, fatigue) — señal, no error. | `rate(clone_detected_total + totp_replay_total + token_reuse_total + push_fatigue_total[1h])` |

### 2.2. SLOs y SLAs

| Métrica | SLO (Objetivo Interno) | SLA (Contrato Enterprise) | Justificación |
| :--- | :--- | :--- | :--- |
| **Challenge Latency (p95)** | **< 50 ms** | **< 100 ms** | Peek de contexto + lookup de enrollments + store de challenge. |
| **Verify TOTP (p95)** | **< 30 ms** | **< 60 ms** | 2 GETDEL Redis + fetch KAGAMI + 3 HMAC + replay mark. |
| **Verify WebAuthn (p95)** | **< 80 ms** | **< 150 ms** | GETDEL challenge + firma + **counter update KAGAMI** (fail-secure). |
| **Verify Push consume (p95)** | **< 20 ms** | **< 50 ms** | Estado ya APPROVED por el dispositivo; solo GETDEL + binding check. |
| **Verify Recovery (p95)** | **< 40 ms** | **< 80 ms** | Hash compare + UPDATE condicional + flag de revisión. |
| **Completion (p95, por PauseKind)** | **< 50 ms** | **< 100 ms** | Medido separado del verify: es la latencia del flujo original reanudado. |
| **Push Approval Funnel** | **> 90% aprobadas antes de expirar** | **> 80%** | UX: un funnel bajo indica fricción de number matching o despacho lento. |
| **Push Approval p50 (humano)** | **< 15 s** | n/a (informativo) | Tiempo de decisión del usuario; TTL 60s es el horizonte duro. |
| **Push Poll Latency (p95)** | **< 20 ms** | **< 50 ms** | Lectura pura de hash Redis; debe soportar miles de pollers concurrentes. |
| **Enrollment Activation (p95)** | **< 100 ms** | **< 200 ms** | Prueba + CAS + lote de recovery (primera activación). |
| **Throughput de verify (pico)** | **500 verifies/seg** | **250 verifies/seg** | Ola matutina con mezcla 70/20/10 (TOTP/WebAuthn/push). |
| **Disponibilidad local** | **99.99%** | **99.95%** | Fail-secure preserva pausas; los usuarios reintentan, no pierden el primer factor. |
| **Detecciones sin degradación** | **p95 de verify invariante ante spikes de clone/replay/fatigue** | — | La contención es por credencial/dispositivo, nunca global. |

---

## 3. Cuellos de Botella y Estrategias de Optimización

### 3.1. Cuello de Botella #1: Costo Criptográfico por Método y el Dummy Anti-Enumeración

**Problema:** Los costos criptográficos son pequeños pero distintos por método, y el camino "sin enrollment" (sondas de enumeración) debe costar lo mismo que el camino real para no filtrar estado por timing.

**Costos medidos de referencia:**

| Camino | Operaciones | CPU aprox. |
| :--- | :--- | :--- |
| Verify TOTP (éxito) | 3 HMAC-SHA1 (ventana ±1) + fixed-time compare | ~0.3 ms |
| Verify TOTP (sin enrollment) | 1 HMAC dummy + fixed-time compare | ~0.15 ms |
| Verify WebAuthn | 1 ECDSA/EdDSA verify + validación de campos | ~1.0 ms |
| Verify Push (consume) | 0 criptografía (estado ya aprobado) | ~0.05 ms |
| Verify Recovery | 1 SHA256 + fixed-time compare | ~0.1 ms |

**Estrategias de Optimización:**
- **El dummy no es opcional ni negociable:** El costo de igualación (~0.15 ms) está dentro del presupuesto TOTP; se monitorea que `verify_duration{reason=no_enrollment}` converja con `{reason=succeeded}` (alerta de divergencia del doc 11).
- **Criptografía en memoria, sin asignaciones por request:** Los buffers de HMAC se reutilizan por pool; el plaintext del secreto TOTP vive solo dentro del `try/finally` con `Clear()` (seguridad y GC a la vez).
- **Bajo ataque de enumeración:** Las sondas hostiles consumen el mismo CPU que verifies legítimos y están limitadas por throttling por usuario/hora + WAF; el costo por sonda es idéntico al legítimo → no hay asimetría explotable ni DoS barato.

### 3.2. Cuello de Botella #2: Round-Trips a KAGAMI en el Camino Caliente

**Problema:** El verify hace 2-3 operaciones KAGAMI (fetch enrollment, counter update/fetch batch, escritura de completitud). Cada una tiene presupuesto de 200 ms; la suma domina la latencia de WebAuthn.

**Estrategias de Optimización:**
- **Índices de lookup directo:** `idx_enr_identity_active (identity_id, method) WHERE status='ACTIVE'` convierte el fetch en index scan de 1 fila (~2-3 ms con pool caliente).
- **Counter update en la misma TX que el outbox de contención** (doc 08, ADR-15-02): un solo commit para el camino de clonación; el camino sano paga un UPDATE + COMMIT (~4-6 ms).
- **Pool de conexiones dedicado:** Los pods `kagi-mfa` comparten el pool con el resto de KAGI pero el HPA escala por latencia de verify, no por CPU — la señal correcta es la contención de pool percibida por el verify.

### 3.3. Cuello de Botella #3: Redis como Spine del Verify

**Problema:** Cada verify toca Redis 2-4 veces (GETDEL token, throttle GET/INCR, replay mark o challenge GETDEL). A 500 verifies/seg son ~1.500-2.000 ops/seg.

**Estrategias de Optimización:**
- **Multiplexing StackExchange.Redis:** Un socket por instancia; las operaciones del verify no abren conexiones nuevas.
- **Timeout de 50 ms fail-fast:** Redis lento no debe convertir el verify en una cola; el breaker de Redis (doc 09) abre a los 3 fallos y retorna 503 preservando la pausa.
- **Throttle con INCR+EXPIRE en un solo comando** (Lua o `INCR` + `EXPIRE` pipelined): evita condiciones donde el contador vive sin TTL.
- **Capacidad:** ~2.000 ops/seg es <2% del throughput de la instancia; el límite real es la latencia, no el volumen (ver §5.2).

### 3.4. Cuello de Botella #4: Push — Carga de Polling y Funnel Humano

**Problema:** Miles de clientes pueden estar polleando `push/status` simultáneamente mientras esperan aprobaciones humanas; y el despacho FCM/APNs tiene latencia propia.

**Estrategias de Optimización:**
- **Polling como lectura pura:** El endpoint de status es un `HGETALL` a Redis (~1-2 ms); a 2.000 pollers concurrentes con intervalo 2.5 s son ~800 GET/seg — triviales.
- **Jitter obligatorio en clientes:** El SDK guía intervalos 2-3 s con jitter y backoff ante 429/503, evitando thundering herd al expirar challenges masivamente.
- **Despacho fire-and-forget con 1 retry:** El dispatch no está en el camino crítico del challenge (se retorna `challenge_id` inmediatamente); si falla, el challenge retorna `push_unavailable` y ofrece otros métodos.
- **Funnel como SLO de UX:** Se mide `issued → approved/denied/expired` con percentiles del tiempo de decisión; un funnel < 90% dispara revisión de UX del number matching o de la latencia de despacho (no de código).

### 3.5. Cuello de Botella #5: Tormentas de Enrollment (Onboarding)

**Problema:** Las incorporaciones masivas (inicio de trimestre, rollout de MFA) concentran activaciones: cada una verifica una prueba, ejecuta un CAS, y la primera por identidad genera un lote de 10 recovery codes.

**Estrategias de Optimización:**
- **Activación por CAS condicional:** `UPDATE ... WHERE status='PENDING'` evita locks pesados; las activaciones concurrentes del mismo pending se serializan naturalmente.
- **Lote de recovery en la misma TX:** La primera activación paga 10 inserts de hash + 1 insert de batch (~8-10 ms adicionales), una sola vez por identidad.
- **SLO separado del verify:** Las activaciones se miden en su propio histograma; una tormenta de onboarding no puede degradar el verify (pools y HPA absorben; el verify tiene prioridad de pool por ser cuello de completitud).

### 3.6. Cuello de Botella #6: La Completitud Delega en Flujos Ajenos

**Problema:** `MfaCompleter` reanuda pausas de UC-11/13/14 o emite elevaciones de KIZUNA: esa latencia no es de MFA, pero el usuario la percibe como parte del verify.

**Estrategia:** Se mide **separada** como `completion_duration{pause_kind}`: el SLO de verify cubre hasta la evidencia; el SLO de completitud cubre la reanudación. Cuando `completion_duration{login_pause}` se degrada, el diagnóstico va al trace de UC-11, no al de MFA — la separación de métricas evita atribuciones erróneas en incidentes.

---

## 4. Estrategia de Load Testing (k6 + Virtual Authenticators)

### 4.1. Escenario 1: Ola Matutina MFA (Mezcla de Métodos)

**Objetivo:** Validar los SLOs de verify por método bajo 500 verifies/seg con mezcla realista, manteniendo las detecciones de seguridad en cero.

```javascript
import http from 'k6/http';
import { check } from 'k6';
import { Trend, Rate } from 'k6/metrics';

const verifyLatency = new Trend('mfa_verify', true);
const completionLatency = new Trend('mfa_completion', true);
const errors = new Rate('errors');

export const options = {
  scenarios: {
    mfa_morning_wave: {
      executor: 'ramping-arrival-rate',
      startRate: 100, timeUnit: '1s',
      preAllocatedVUs: 200, maxVUs: 800,
      stages: [
        { target: 250, duration: '2m' },
        { target: 500, duration: '4m' },   // pico de completitud de logins pausados
        { target: 500, duration: '4m' },
        { target: 0,   duration: '2m' },
      ],
    },
  },
  thresholds: {
    // Por método (tags): totp p95<30ms, webauthn p95<80ms, push p95<20ms
    'mfa_verify{method:totp}': ['p(95)<30'],
    'mfa_verify{method:webauthn}': ['p(95)<80'],
    'mfa_verify{method:push}': ['p(95)<20'],
    'mfa_completion': ['p(95)<50'],
    errors: ['rate<0.005'],
    // Externos post-run:
    // clone_detected_total == 0; totp_replay_total == 0; token_reuse_total == 0
  },
};

export default function () {
  const paused = nextPausedSession();                    // mfa_tokens pre-emitidos (UC-11 mock)
  const method = pickMethod({ totp: 0.7, webauthn: 0.2, push: 0.1 });

  const challenge = http.post(`${__ENV.BASE_URL}/v1/auth/mfa/challenge`,
    JSON.stringify({ mfa_token: paused.mfaToken, method }),
    { headers: { 'Content-Type': 'application/json' }, tags: { method } });

  const proof = buildProof(method, challenge.json('challenge_id'));  // authenticator virtual
  const t0 = Date.now();
  const verify = http.post(`${__ENV.BASE_URL}/v1/auth/mfa/verify`,
    JSON.stringify({ mfa_token: paused.mfaToken, method, proof, challenge_id: challenge.json('challenge_id') }),
    { headers: { 'Content-Type': 'application/json' }, tags: { method } });
  verifyLatency.add(Date.now() - t0);

  if (!check(verify, { 'verify 200': (r) => r.status === 200 })) errors.add(1);
}
```

### 4.2. Escenario 2: Costo del Dummy Anti-Enumeración bajo Sondeo Hostil

**Objetivo:** Verificar que las sondas a identidades sin enrollment cuestan lo mismo que verifies legítimos (sin asimetría explotable) y que el throttling las contiene.

```javascript
export const options = {
  scenarios: {
    legit_traffic: {
      executor: 'constant-arrival-rate', rate: 300, timeUnit: '1s', duration: '8m',
      exec: 'legitTotpFlow',
    },
    hostile_probes: {
      executor: 'constant-arrival-rate', rate: 100, timeUnit: '1s', duration: '8m',
      exec: 'probeNoEnrollment',   // mfa_tokens de identidades SIN TOTP enrollado
    },
  },
  thresholds: {
    // Externo: verify_duration_p50{reason=no_enrollment} ≈ verify_duration_p50{reason=succeeded}
    // (divergencia < 5ms → dummy efectivo; > 10ms → alerta T-116)
    // hostile probes: tras 5 fallos por token y 10 por usuario/hora, 100% throttled
  },
};
```

### 4.3. Escenario 3: Tormenta de Ceremonies WebAuthn (Presión de Counter Updates)

**Objetivo:** 200 verifies WebAuthn/seg sostenidos para estresar el camino más caro (GETDEL challenge + firma + counter update + TX de contención).

```javascript
export const options = {
  scenarios: {
    webauthn_storm: {
      executor: 'constant-arrival-rate', rate: 200, timeUnit: '1s', duration: '10m',
      preAllocatedVUs: 150, maxVUs: 400,
    },
  },
  thresholds: {
    'mfa_verify{method:webauthn}': ['p(95)<80', 'p(99)<150'],
    // Externos: kagami counter update p95 < 10ms; sin CloneDetected (counters estrictamente crecientes)
  },
};
// El authenticator virtual incrementa el counter monótonamente;
// una variante del test fuerza counter stale y exige EXACTAMENTE:
// 1 clone_detected + enrollment REVOKED + completitudes con esa credential == 0
```

### 4.4. Escenario 4: Soak Push con Humanos Virtuales y Carga de Polling

**Objetivo:** Modelar el funnel push con distribución realista de decisión humana y la carga de polling concurrente.

```javascript
import { randomIntBetween } from 'https://jslib.k6.io/k6-utils/1.4.0/index.js';

export const options = {
  scenarios: {
    push_funnel: {
      executor: 'constant-arrival-rate', rate: 50, timeUnit: '1s', duration: '20m',
      exec: 'pushFlowWithVirtualHuman',
    },
    poll_load: {
      executor: 'constant-vus', vus: 2000, duration: '20m',
      exec: 'pollStatusEvery2_5sWithJitter',
    },
  },
  thresholds: {
    push_poll: ['p(95)<20'],
    // Externos: funnel approved/(issued) > 0.90; approval p50 < 15s
    // push_fatigue_total == 0 (los humanos virtuales aprueban al primer intento)
  },
};

export function pushFlowWithVirtualHuman() {
  const ch = issuePushChallenge();                      // POST challenge → dispatch simulado
  const decisionMs = humanDecisionTime();               // distribución: 80% 3-20s, 15% 20-50s, 5% deniega
  sleep(decisionMs / 1000);
  deviceApprove(ch.challenge_id, ch.number);            // POST inbound simulando el dispositivo
  verifyPush(ch);
}
```

### 4.5. Escenario 5: Chaos Quirúrgico

| Inyección | Ventana | Expectativas verificadas |
| :--- | :--- | :--- |
| **Redis caído** | 5 min en plena ola | 503 `unavailable{redis}`; **pausas NO invalidadas**; al recuperar, verifies normales sin reconciliación; `token_reuse_total` plano |
| **KAGAMI caído tras firma WebAuthn** | Error inyectado en `counter.update` | Verify → 503; `Completer` sin llamadas (fail-secure); **cero tokens elevados emitidos**; al recuperar, el enrollment sigue ACTIVE con counter antiguo |
| **FCM/APNs caído** | 10 min | Challenges push → `push_unavailable`; los usuarios completan con TOTP/WebAuthn; funnel push cae a 0 sin errores 5xx |
| **Notification caído** | 10 min | Clonaciones y recovery codes se completan igual; alertas encoladas en Outbox; backlog drena <60s tras recuperación |
| **Tormenta de throttling** | 100 ataques de 6 intentos | 100% de pausas invalidadas al 5º; `throttled_total{dimension=token}` = 100; el tráfico legítimo paralelo mantiene p95 |

### 4.6. Escenario 6: Soak 24h (Higiene de Keyspaces y Particiones)

**Objetivo:** A 100 verifies/seg sostenidos verificar: memoria de `mfa:*` dentro de ±20% de la curva teórica (~27 MB); expiración correcta de `mfa:pending:*`, `mfa:challenge:*` y throttle por hora; crecimiento y drop de particiones de `mfa_step_ups`; job de higiene de trusted devices; y cero `unavailable_total` espurios.

---

## 5. Capacity Planning y Escalabilidad

### 5.1. CPU de Pods kagi-mfa por Mezcla de Métodos

| Componente por verify | TOTP (70%) | WebAuthn (20%) | Push (10%) |
| :--- | :--- | :--- | :--- |
| Redis ops (GETDEL×2 + throttle + replay/challenge) | ~4 ms | ~4 ms | ~3 ms |
| KAGAMI (fetch + counter/batch + completitud) | ~5 ms | ~9 ms | ~5 ms |
| Criptografía | ~0.3 ms | ~1.0 ms | ~0.05 ms |
| **Latencia media por verify** | **~9.3 ms** | **~14 ms** | **~8 ms** |

**Cálculo de pods:** 500 verifies/seg × (0.7×9.3 + 0.2×14 + 0.1×8) ms ≈ 500 × 10.1 ms = **5.05 segundos-core/seg**, más overhead de red/serialización (~1.5x) ≈ **7.5 vCPUs** → 4 pods × 2 vCPU en baseline con HPA hasta 20 (doc 09). Un shift hacia WebAuthn-dominante (migración PR) elevaría el requerimiento ~20% — dentro del headroom del HPA.

### 5.2. Redis Auth (Ops y Memoria de MFA)

| Keyspace | Ops/seg en pico | Memoria pico |
| :--- | :--- | :--- |
| `mfa:token:*` | 500 GETDEL + 500 GET (challenge peek) | ~15 MB |
| `mfa:challenge:*` | ~100 SET + ~100 GETDEL + ~800 HGET (polling) | ~7 MB |
| `mfa:totp:laststep:*` | ~350 SET NX | ~3 MB |
| `mfa:throttle:*` / `mfa:push:*` | ~1.000 INCR/GET | < 2 MB |
| **Total MFA** | **~3.350 ops/seg** | **~27 MB** |

La instancia Redis Auth acumula UC-11/12/13 (~130 MB) + UC-14 (~100 MB) + UC-15 (~27 MB) ≈ **260 MB bajo el peor pico simultáneo: <13% de los 2 GB con `noeviction`**. El throughput (~9.000 ops/seg combinados) es <10% de la capacidad de un nodo Redis moderno.

### 5.3. PostgreSQL (Volumen de MFA)

| Tabla | Crecimiento | Carga en pico |
| :--- | :--- | :--- |
| `mfa_enrollments` | ~1.5 filas/identidad (multi-método); 75k filas para 50k usuarios | 500 SELECT indexados/seg + ~20 counter updates/seg |
| `recovery_codes` | 10 filas por batch; batches ≈ identidades con MFA | Consumos esporádicos (UPDATE condicional) |
| `trusted_devices` | ~0.5 filas/identidad con bypass | Validaciones por login bypass (~10% del tráfico) |
| `mfa_step_ups` | Particionada mensual; retención 90 días | Inserciones = tasa de step-up (scope + riesgo) |

**Conclusión:** MFA es un workload de **lectura dominante con escrituras pequeñas y frecuentes**; comparte la instancia de KAGI sin presión de IOPS (todo cabe en cache + WAL). El índice parcial de activos y el CAS de activación son las únicas estructuras sensibles a contención, y ambas operan por fila individual.

### 5.4. Carga de Polling Push

Con 5.000 challenges push concurrentes y polling de 2.5 s con jitter: ~2.000 `HGETALL`/seg al endpoint de status. Cada uno es una lectura Redis de ~1-2 ms sin mutación — el endpoint escala horizontalmente sin estado y no compite con el verify por recursos de escritura.

---

## 6. Observabilidad y Profiling en Producción

### 6.1. Traces por Método (Reafirmación con Presupuesto)

```text
POST /v1/auth/mfa/verify — TOTP [9ms]
├── redis.token.consume [2ms]
├── throttle.acquire [2ms]
├── kagami.enrollment.fetch [3ms]      ← índice parcial de activos
├── totp.compute_window [0.3ms]
├── redis.replay.mark [1ms]
└── completer.resolve [1ms] → resolver de pausa (trace propio del UC origen)

POST /v1/auth/mfa/verify — WebAuthn [12ms]
├── redis.token.consume [2ms]
├── throttle.acquire [2ms]
├── redis.challenge.getdel [2ms]
├── webauthn.verify_signature [1ms]
├── kagami.counter.update [4ms]        ← fail-secure: bloquea completitud si falla
└── completer.resolve [1ms]

POST /auth/mfa/push/approve [14ms]     (inbound del dispositivo)
├── kagami.attestation.validate [6ms]
├── redis.challenge.read [2ms]
├── challenge.try_approve [0.5ms]
└── redis.cas.approved [2ms]
```

### 6.2. Dashboards de Grafana (Específicos para UC-15)

#### Panel 1: Verify Latency por Método + Dummy Convergence
*   **Métricas:** `verify_duration{method}` p50/p95 + `verify_duration{reason}` (succeeded vs no_enrollment).
*   **Lectura:** La convergencia de `no_enrollment` con `succeeded` es la verificación continua del dummy anti-enumeración; una divergencia es la alerta T-116.

#### Panel 2: Push Funnel (UX, no latencia)
*   **Métricas:** `challenge_issued → approved/denied/expired` como porcentajes apilados + `push_approval_duration` p50/p95.
*   **Lectura:** Un descenso del funnel con approval duration alto indica fricción de number matching; con duration bajo y expirados altos, indica despacho lento o push no entregado.

#### Panel 3: Seguridad como Señal Operativa
*   **Métricas:** `clone_detected_total`, `totp_replay_total`, `token_reuse_total`, `push_fatigue_total`, `recovery_used_total` apiladas por tenant.
*   **Lectura:** Estas métricas son el pulso de ataque del segundo factor; baseline cero con spikes investigables. El p95 de verify debe permanecer plano durante los spikes (contención aislada).

#### Panel 4: Disponibilidad y Degradaciones
*   **Métricas:** `unavailable_total{dependency}` + pausas preservadas vs invalidadas.
*   **Lectura:** Durante un fail-secure, las pausas preservadas deben superar el 99% (las invalidadas son solo token_reuse/throttled legítimos).

#### Panel 5: Enrollment Funnel y Onboarding
*   **Métricas:** `enrollment_activated_total{method}` vs `enrollment_cancelled_total{reason}` + duración de activación.
*   **Lectura:** Una tasa de cancelación alta durante rollouts indica UX de activación rota, no ataque.

### 6.3. Alertas de Rendimiento

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Verify TOTP Regression** | `verify_duration_p95{method=totp} > 30ms` por 5 min | Alta | Descomponer por span: Redis (GETDEL/replay) vs KAGAMI (fetch); verificar pool y latencia de red. |
| **Verify WebAuthn Regression** | `verify_duration_p95{method=webauthn} > 80ms` por 5 min | Alta | Aislar `counter.update`: si domina, contención en KAGAMI; si domina `challenge.getdel`, Redis. |
| **Completion Slow por PauseKind** | `completion_duration_p95{pause_kind=X} > 50ms` | Alta | El problema está en el flujo origen (UC-11/13/14), no en MFA: escalar al owner del resolutor. |
| **Push Funnel Degraded** | Funnel approved < 90% por 15 min | Media | Revisar latencia de despacho FCM/APNs y UX del number matching; no es un problema de servidor. |
| **Poll Error Rate** | `push_poll` errores > 1% | Media | Presión sobre el endpoint de status o Redis degradado; verificar jitter de clientes. |
| **Dummy Divergence** | `verify_duration_p50{reason=no_enrollment}` diverge > 10ms de `{reason=succeeded}` | Alta | El dummy HMAC no está igualando costos: posible oráculo de enumeración emergente (T-116). |
| **Dispatch Failure Rate** | `push_dispatch_failures_total` > 2% del issued | Alta | FCM/APNs degradado; los usuarios están cayendo a métodos alternativos — verificar que el fallback funcione. |

---

## 7. Configuración de Rendimiento

### 7.1. Timeouts y Pools (Reafirmación Tipada)

```csharp
public sealed record MfaTimeouts
{
    public TimeSpan RedisOperation       { get; init; } = TimeSpan.FromMilliseconds(50);
    public TimeSpan KagamiOperation      { get; init; } = TimeSpan.FromMilliseconds(200);
    public TimeSpan PushDispatch         { get; init; } = TimeSpan.FromSeconds(2);
    public TimeSpan ChallengeTtlPush     { get; init; } = TimeSpan.FromSeconds(60);
    public TimeSpan ChallengeTtlWebAuthn { get; init; } = TimeSpan.FromMinutes(5);
    public TimeSpan EnrollmentPendingTtl { get; init; } = TimeSpan.FromMinutes(10);
    public int PushDispatchRetries       { get; init; } = 1;   // único retry del UC
    public int Retries                   { get; init; } = 0;   // GETDEL/CAS no son reintentables
}
```

### 7.2. Feature Flags de Rendimiento

| Feature Flag | Descripción | Default |
| :--- | :--- | :--- |
| `kagi.auth.mfa.polling_interval_ms_min` | Intervalo mínimo de polling push aceptado del cliente | `2000` |
| `kagi.auth.mfa.dummy_verification_enabled` | Dummy HMAC anti-enumeración | `true` (**bloqueado**: no desactivable) |
| `kagi.auth.mfa.push_dispatch_retries` | Reintentos de despacho FCM/APNs | `1` |
| `kagi.auth.mfa.completion_pool_priority` | Prioridad de pool para verifies sobre activaciones | `true` |

### 7.3. HPA (Reafirmación con Anotación de Método)

```yaml
# Doc 09, con anotación operativa:
# El escalado responde a verify_duration_p95 de la MEZCLA observada.
# Si un tenant migra a WebAuthn-dominante (PR), el costo medio por verify
# sube ~20%: el HPA lo absorbe; la métrica de CPU por método (§5.1) es el
# leading indicator para renegociar minReplicas en rollouts PR masivos.
metrics:
  - type: Pods
    pods:
      metric: { name: kagi_auth_mfa_verify_duration_p95 }
      target: { type: AverageValue, averageValue: "30" }
  - type: Resource
    resource: { name: cpu, target: { type: Utilization, averageUtilization: 65 } }
```

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Los objetivos de rendimiento (challenge <50ms, verify por método, activación <100ms, funnel push) se formalizan aquí como SLOs medibles por método y por fase. |
| `07_Sequence_Diagram.md` | El presupuesto de timeouts y los traces por método (§7.5 de ese doc) son la base de los SLOs y spans de este documento. |
| `08_Final_Class_Design.md` | `MfaTimeouts`, `MfaMetrics` y los adaptadores Redis/KAGAMI emiten exactamente las métricas y respetan los presupuestos definidos aquí. |
| `09_Modern_Artifacts.md` | Las métricas `kagi.auth.mfa.*`, el HPA y los escenarios k6 preliminares se extienden aquí con dummy convergence, funnel push y chaos quirúrgico. |
| `10_UC_Specific_Data_Model.md` | Los keyspaces Redis (~27 MB) y el volumen de tablas (enrollments, recovery, trusted, step-ups) validan el capacity planning bajo ola. |
| `11_UC_Specific_Security_Threats.md` | El costo del dummy (T-116), el throttling bajo fuego (T-107) y la contención de clonación sin degradación (T-109) se verifican empíricamente en los escenarios 2, 3 y 5. |
| `UC-11/UC-13/UC-14 (12_Performance)` | `completion_duration{pause_kind}` delega el diagnóstico en los SLOs de los flujos origen: MFA mide hasta la evidencia; cada UC mide su reanudación. |

---

## 9. Cierre del Paquete de Diseño (UC-15)

Con este documento, completamos los **12 archivos del Paquete de Diseño Profundo (Tier 1 + Tier 2)** para el UC-15:

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

### 9.1. Resumen de lo Logrado en UC-15

Hemos diseñado el segundo factor como un **sistema de evidencia con adversario activo y presupuesto medido**:
- **Cinco métodos heterogéneos, una evidencia única:** Estrategias `IMethodVerifier` que convergen en `AssuranceEvidence`; la completitud consume evidencia, nunca pruebas crudas.
- **Seguridad observable sin degradación:** Clonación, replay, fatiga, token reuse y recovery son señales con métricas, alertas y contención aislada que no tocan el p95 del camino caliente.
- **Orden de controles como contrato:** GETDEL → throttling → criptografía → completitud, con replay post-verificación y counter pre-completitud, verificado por tests de orden.
- **Push medido como funnel humano:** La aprobación push tiene SLO de funnel (>90% antes de expirar) y percentiles de decisión, no SLO de latencia de servidor — la categoría correcta para un tramo mediado por humano.
- **Fail-secure diferenciado:** Counter no persistido → completitud rechazada; trusted device no emitido → completitud preservada; Redis caído → pausa preservada para reintento.
- **Capacidad holgada y verificada:** ~260 MB de Redis combinado (<13%), ~3.350 ops/seg de MFA, y 4-20 pods escalando por latencia de verify, no por CPU.

### 9.2. Estado del EPIC-02 (Authentication)

| UC | Nombre | Estado | Documentos |
| :--- | :--- | :--- | :--- |
| **UC-11** | Autenticar Credenciales Locales | ✅ Completo | 12/12 |
| **UC-12** | Cerrar Sesión (Logout) | ✅ Completo | 12/12 |
| **UC-13** | Renovar Sesión (Refresh Token) | ✅ Completo | 12/12 |
| **UC-14** | Autenticar Federada (SSO) | ✅ Completo | 12/12 |
| **UC-15** | Multi-Factor Authentication (MFA) | ✅ Completo | 12/12 |
| **UC-16** | Passwordless Authentication | 📋 Pendiente | 0/12 |
| **UC-17** | Risk-Based Authentication | 📋 Pendiente | 0/12 |
| **UC-18** | Recuperación de Credenciales | 📋 Pendiente | 0/12 |
| **UC-19** | Cambio de Password (Self-Service) | 📋 Pendiente | 0/12 |

**Total: 60/108 documentos completados en el EPIC-02.** Los cinco pilares de autenticación fuerte quedan cerrados: credenciales locales (UC-11), segundo factor (UC-15), federación (UC-14), rotación/detección (UC-13) y terminación propagada (UC-12). El siguiente bloque natural es la autenticación sin contraseña (UC-16) o el motor de riesgo que orquesta step-ups en todos ellos (UC-17).

---
*Fin del Documento 12_UC_Specific_Performance_Benchmarks para UC-15*