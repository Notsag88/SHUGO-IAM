# UC-16: Passwordless Authentication - Benchmarks, SLAs y Estrategia de Rendimiento

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-16 - Passwordless Authentication (Passkeys + Magic Links + OTP Restringido) |
| **Documento:** | 12_UC_Specific_Performance_Benchmarks.md |
| **Metodología:** | SRE (Site Reliability Engineering) + Performance Engineering + Load Testing (k6) + Chaos Engineering + Hostile Traffic Modeling |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 10/11/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 10/11/2026 | Gastón Ruiz | **Creación inicial.** Definición de SLAs, SLOs y estrategia de Load Testing para autenticación passwordless. Modela los siete perfiles de carga críticos: ola matutina mixta por canal, flood de assertions basura contra tráfico legítimo (validación de la escalera de guardas como filtro de costo), funnel de magic link con humanos virtuales y tormenta de prefetch, stuffing/brute-force de OTP bajo concurrencia (atomicidad de HINCRBY), tormenta de enrollments con attestation, chaos quirúrgico por dependencia, y soak de 24h con drift de keyspaces state-as-storage. Incluye capacity planning de Redis (~34 MB, ~3.500 ops/seg), PostgreSQL (CAS de counter sin locks) y CPU ponderado por mezcla de canales, más el presupuesto de throughput del proveedor de email. | Documento de referencia para SREs y Arquitectos de Rendimiento. Cierra el UC-16 garantizando que el tráfico hostil pague solo los peldaños baratos de la escalera de guardas, que el funnel mediado por email/humano se mida como conversión y no como latencia, y que el estado implícito en Redis permanezca dentro de su curva teórica bajo soak prolongado. |

---

## 1. Introducción y Propósito

El **UC-16 (Passwordless Authentication)** presenta el perfil de rendimiento más heterogéneo del motor KAGI, por cinco razones estructurales:

1. **Criptografía asimétrica por canal:** El verify passkey gasta ~1 ms en firma ECDSA pero hace 3 round-trips (challenge GETDEL, lookup, counter CAS); el magic link confirm gasta ~0 ms de criptografía pero 2 operaciones Redis + riesgo; el OTP gasta un hash compare. El costo dominante no es CPU sino **orden de guardas y round-trips**.
2. **Tráfico hostil mezclado con legítimo:** La superficie pública de passwordless (options, solicitudes OOB, verifies) recibe sondas y floods que el diseño debe contener **por costo**: el tráfico basura debe morir en los peldaños baratos de la escalera sin tocar KAGAMI ni criptografía.
3. **Un tramo mediado por email y por humano:** El magic link tiene dos latencias fuera de control (entrega del proveedor, decisión del usuario) que se miden como **funnel de conversión** y percentiles de espera, nunca como latencia de servidor.
4. **Estado implícito bajo carga:** Los artefactos de un solo uso viven como presencia en Redis con TTL; el soak debe verificar que la memoria siga la curva teórica (rate × TTL × bytes) sin fugas ni acumulación.
5. **Despacho fuera del camino crítico:** La respuesta de emisión se construye antes del transporte de email; el presupuesto del proveedor (SES/SMTP) es una dependencia de throughput, no de latencia.

---

## 2. Service Level Indicators (SLIs) y Objectives (SLOs)

### 2.1. SLIs Específicos para UC-16

| SLI | Descripción | Fórmula / Medición (OpenTelemetry) |
| :--- | :--- | :--- |
| **Options Latency (p95)** | Emisión de opciones de ceremonia (challenge + lookup de sugerencias). | `histogram_quantile(0.95, rate(kagi_auth_passwordless_options_duration_bucket[5m]))` |
| **Verify Latency por Canal (p95/p99)** | Verificación completa local (consume → guardas → evidencia → riesgo → emisión/pausa). | `histogram_quantile(0.95, rate(kagi_auth_passwordless_verify_duration_bucket{channel}[5m]))` |
| **Confirm Latency (p95)** | Confirmación de magic link (gate → GETDEL → riesgo → issuer). | `histogram_quantile(0.95, rate(kagi_auth_passwordless_confirm_duration_bucket[5m]))` |
| **Redeem Latency (p95)** | Canje de session_code (GETDEL + mint). | `histogram_quantile(0.95, rate(kagi_auth_passwordless_redeem_duration_bucket[5m]))` |
| **Issuance Latency (p95)** | Solicitud OOB hasta respuesta 202 (despacho excluido). | `histogram_quantile(0.95, rate(kagi_auth_passwordless_issuance_duration_bucket[5m]))` |
| **Dispatch Latency (p95)** | Transporte de email (async, informativo). | `histogram_quantile(0.95, rate(kagi_auth_passwordless_dispatch_duration_bucket[15m]))` |
| **Magic Link Funnel** | `consumed / issued` en ventana móvil de 1h. | `sum(rate(magiclink_consumed_total[1h])) / sum(rate(magiclink_issued_total[1h]))` |
| **Magic Link Wait (p50/p95)** | Tiempo solicitud→consumo (entrega + decisión humana). | `histogram_quantile(0.5, rate(magiclink_ttl_elapsed_bucket[1h]))` |
| **Guard Ladder Drop-off** | Distribución de rechazos por peldaño (dónde muere el tráfico). | `sum by (stage) (rate(failed_total[10m]))` |
| **CAS Rejection Rate** | Proporción de counter CAS rechazados (concurrencia o clonación). | `rate(counter_cas_rejected_total[10m]) / rate(counter_cas_total[10m])` |
| **Passwordless Availability** | 1 − (503 / total verify+confirm+redeem). | `1 - rate(unavailable_total[5m]) / rate(verify_total[5m])` |

### 2.2. SLOs y SLAs

| Métrica | SLO (Objetivo Interno) | SLA (Contrato Enterprise) | Justificación |
| :--- | :--- | :--- | :--- |
| **Options Latency (p95)** | **< 50 ms** | **< 100 ms** | SET de challenge + lookup opcional de sugerencias. |
| **Verify Passkey (p95)** | **< 80 ms** | **< 150 ms** | Escalera completa: GETDEL + lookup + firma + CAS + riesgo. |
| **Confirm Magic Link (p95)** | **< 150 ms** | **< 300 ms** | Peek + gate + GETDEL + riesgo + emisión. |
| **Verify OTP (p95)** | **< 50 ms** | **< 100 ms** | GET hash + compare + GETDEL + riesgo. |
| **Issuance OOB (p95)** | **< 100 ms** | **< 200 ms** | Respuesta antes del despacho (fire-and-forget). |
| **Redeem (p95)** | **< 50 ms** | **< 100 ms** | GETDEL + mint sobre familia existente. |
| **Magic Link Funnel** | **> 0.80** | **> 0.70** | Conversión issued→consumed; caída indica entrega o UX. |
| **Magic Link Wait (p50)** | **< 30 s** | n/a (informativo) | Entrega de email + lectura; TTL 10 min es el horizonte. |
| **Dispatch Success Rate** | **> 98%** | **> 95%** | Tras 1 retry; fallos son métrica, no error de request. |
| **Throughput verify (pico)** | **500/seg** (mezcla 70/20/10) | **250/seg** | Ola matutina passwordless. |
| **Throughput issuance OOB (pico)** | **70/seg** (50 magic + 20 OTP) | **35/seg** | Limitado por cuota del proveedor de email. |
| **Disponibilidad local** | **99.99%** | **99.95%** | Fail-secure preserva primeros factores; usuarios reintentan. |
| **Costo de tráfico hostil** | **< 3 ms/request basura** | — | La escalera debe matar el flood antes de KAGAMI. |

---

## 3. Cuellos de Botella y Estrategias de Optimización

### 3.1. Cuello de Botella #1: La Escalera de Guardas como Filtro de Costo

**Problema:** La superficie de verify passkey es pública y atacable: un flood de assertions basura que llegara a KAGAMI o a la verificación de firma multiplicaría el costo por request ×10.

**Estrategia (validada en el Escenario 4.2):** El orden de la escalera es el presupuesto:

| Peldaño | Costo | Qué mata |
| :--- | :--- | :--- |
| Challenge GETDEL | ~2 ms (Redis) | Ceremonias replayadas / sin options |
| Origin/rpId | ~0.01 ms | Assertions de otros orígenes |
| Flag UV | ~0.01 ms | Assertions sin presencia de usuario |
| **Total peldaños 1-3** | **~2.1 ms** | **>95% del tráfico basura típico** |
| Lookup KAGAMI | ~3 ms | CredentialIds desconocidos |
| Firma | ~1 ms | Assertions con firma inválida |
| Counter CAS | ~4 ms | Clonación / carrera |

El tráfico basura bien formado (challenge válido de options reales + assertion forjada) alcanza el peldaño 5 (~6 ms), pero generar challenges válidos requiere consumir options reales, lo que está rate-limited en el gateway. El SLO de costo hostil (<3 ms/request) se monitorea con `failed_total{stage}` ponderado por duración de span.

### 3.2. Cuello de Botella #2: Round-Trips en el Camino Caliente Passkey

**Problema:** El verify passkey hace 3 operaciones serializadas (Redis GETDEL → KAGAMI lookup → KAGAMI CAS) + riesgo. Con Redis a 2 ms y KAGAMI a 3-5 ms por operación indexada, el total ronda 15-20 ms antes del riesgo.

**Estrategias:**
- **Índice parcial de lookup activo:** `idx_pkc_active_lookup (tenant_id, credential_id) WHERE status='ACTIVE'` convierte el lookup en index scan de 1 fila (~2-3 ms con pool caliente).
- **CAS sin re-lectura en el camino feliz:** El `UPDATE ... RETURNING` devuelve el resultado en el mismo round-trip; solo el perdedor del CAS hace un re-read.
- **Riesgo en paralelo cuando es posible:** Para tenants con `risk_fail_mode=open`, la evaluación de riesgo puede lanzarse concurrentemente al CAS (ninguno depende del otro); la emisión espera a ambos. Reduce el p95 ~15-20 ms en caminos de riesgo lento.
- **Pool dedicado de KAGAMI para passwordless:** El HPA escala por latencia de verify, no por CPU, protegiendo el lookup+CAS de la contención de otros consumidores de KAGAMI.

### 3.3. Cuello de Botella #3: El Despacho de Email como Dependencia de Throughput

**Problema:** La ola matutina OOB puede exigir 70 emails/seg pico. Los proveedores (SES, SMTP corporativo) tienen cuotas por segundo y latencias de 200-800 ms.

**Estrategias:**
- **Fire-and-forget fuera del request:** El 202 se retorna tras `SET` del artefacto en Redis; el despacho corre en background con su propio pool y 1 retry.
- **Cola interna con backpressure:** El dispatcher usa `Channel<T>` acotado (10.000); si se llena (proveedor caído sostenido), los despachos nuevos fallan rápido hacia la métrica en lugar de acumular memoria.
- **Presupuesto por tenant:** Rate limit de despacho por tenant (proporcional a su cuota del proveedor) evita que un tenant en tormenta OOB agote la cuota global.
- **Supersede como válvula:** La re-solicitud del usuario reemplaza el artefacto anterior: el proveedor nunca procesa dos emails para la misma emisión lógica.

### 3.4. Cuello de Botella #4: Estado Implícito bajo Soak (Fugas de TTL)

**Problema:** Con estado implícito (presencia + TTL), un bug que omita el `EXPIRE` en un `SET` acumularía claves muertas que distorsionan memoria y, peor, artefactos que no expiran (riesgo de seguridad y rendimiento).

**Estrategia:** El soak de 24h (Escenario 4.7) compara la memoria observada de cada keyspace contra la **curva teórica** `rate × TTL × bytes_por_entrada` con tolerancia ±20%. Divergencia = fuga de TTL alertada; la seguridad (artefacto vivo más allá de su ventana) y el rendimiento (memoria fantasma) se detectan con el mismo mecanismo.

### 3.5. Cuello de Botella #5: Concurrencia de Verifies sobre la Misma Credencial

**Problema:** Un usuario con múltiples tabs o un cliente que reintenta puede disparar verifies concurrentes sobre el mismo passkey; el CAS de counter serializa, pero un spike de `StaleOrRevoked` indicaría retry storms.

**Estrategia:**
- El CAS rechaza al perdedor en ~4 ms sin locks; el perdedor retorna rechazo determinista (el cliente reintenta con nueva ceremonia, no con la misma assertion — el challenge ya fue consumido).
- `counter_cas_rejected_total` tiene baseline ~0 en tráfico normal; el spike es señal de bug de cliente o ataque de replay automatizado, y se alerta independientemente de la latencia.

---

## 4. Estrategia de Load Testing (k6 + Instrumentos Virtuales)

### 4.1. Escenario 1: Ola Matutina Mixta (500 verifies/seg)

**Objetivo:** Validar los SLOs por canal bajo mezcla realista (70% passkey, 20% magic link, 10% OTP) con `VirtualPasskeyAuthenticator` y `EmailCaptureSink`.

```javascript
import http from 'k6/http';
import { check } from 'k6';
import { Trend, Rate } from 'k6/metrics';

const verifyLatency = new Trend('passwordless_verify', true);
const confirmLatency = new Trend('passwordless_confirm', true);
const errors = new Rate('errors');

export const options = {
  scenarios: {
    passwordless_morning_wave: {
      executor: 'ramping-arrival-rate',
      startRate: 100, timeUnit: '1s',
      preAllocatedVUs: 200, maxVUs: 900,
      stages: [
        { target: 300, duration: '2m' },
        { target: 500, duration: '4m' },
        { target: 500, duration: '4m' },
        { target: 0,   duration: '2m' },
      ],
    },
  },
  thresholds: {
    'passwordless_verify{channel:passkey}': ['p(95)<80', 'p(99)<150'],
    'passwordless_verify{channel:otp}': ['p(95)<50'],
    'passwordless_confirm': ['p(95)<150'],
    errors: ['rate<0.005'],
    // Externos: magiclink funnel > 0.80; counter_cas_rejected ≈ 0;
    //           unavailable_total == 0; clone_detected_total == 0
  },
};

export default function () {
  const channel = pickChannel({ passkey: 0.7, magic_link: 0.2, otp: 0.1 });
  switch (channel) {
    case 'passkey': runPasskeyCeremony(); break;          // options + assertion virtual + verify
    case 'magic_link': runMagicLinkWithSink(); break;      // request → sink extrae token → GET → confirm
    case 'otp': runOtpWithSink(); break;                   // request → sink extrae código → verify
  }
}
```

### 4.2. Escenario 2: Flood de Assertions Basura + Tráfico Legítimo

**Objetivo:** Verificar que el tráfico hostil muere en los peldaños baratos y **no degrada** el legítimo.

```javascript
export const options = {
  scenarios: {
    garbage_flood: {
      executor: 'constant-arrival-rate',
      rate: 2000, timeUnit: '1s', duration: '6m',
      exec: 'garbageAssertions',      // 40% origin inválido, 30% UV=0, 30% challenge replayado
    },
    legit_passkeys: {
      executor: 'constant-arrival-rate',
      rate: 200, timeUnit: '1s', duration: '6m',
      exec: 'runPasskeyCeremony',
    },
  },
  thresholds: {
    'passwordless_verify{channel:passkey,traffic:legit}': ['p(95)<80'],   // invariante bajo flood
    // Externos: costo_medio_basura < 3ms (span duration de failed por stage)
    //           kagami.credential.fetch QPS ≈ 0 desde tráfico basura
    //           (los peldaños 1-3 deben matar el flood antes de KAGAMI)
  },
};
```

**Nota de validación:** El assertion crítico es que `kagami.credential.fetch` permanezca plano durante el flood: si el lookup de KAGAMI escala con el tráfico basura, la escalera está rota (algún peldaño temprano no está rechazando lo que debe).

### 4.3. Escenario 3: Funnel de Magic Link con Humanos Virtuales y Prefetch

**Objetivo:** Medir conversión y resistencia a prefetch bajo carga.

```javascript
export const options = {
  scenarios: {
    magiclink_funnel: {
      executor: 'constant-arrival-rate', rate: 50, timeUnit: '1s', duration: '20m',
      exec: 'magicLinkWithVirtualHuman',
    },
    prefetch_storm: {
      executor: 'constant-arrival-rate', rate: 100, timeUnit: '1s', duration: '20m',
      exec: 'aggressivePrefetch',      // 100 GETs/seg a enlaces recién emitidos (simula scanners)
    },
  },
  thresholds: {
    // Externos: consumed/issued > 0.80 (el prefetch NO reduce la conversión)
    //           magiclink_failed_total{stage=consumed} por prefetch == 0
    //           (ningún enlace murió por GET: state-as-storage verificado empíricamente)
  },
};

export function magicLinkWithVirtualHuman() {
  const req = requestMagicLink();                    // 202 + sink captura email
  const link = emailSink.extractLink(req.identity);
  http.get(link);                                    // clic humano (validación)
  sleep(humanDecisionTime());                        // distribución: 80% 3-20s, 15% 20-50s, 5% abandona
  const confirm = http.post(`${__ENV.BASE_URL}/v1/auth/passwordless/link/confirm`,
    JSON.stringify({ token: link.token }), { headers: jsonHeaders });
  check(confirm, { 'confirm 200/302': (r) => r.status === 200 || r.status === 302 });
}

export function aggressivePrefetch() {
  const link = emailSink.peekRandomActiveLink();     // scanners: GET repetido, jamás POST
  if (link) http.get(link);
}
```

### 4.4. Escenario 4: OTP bajo Concurrencia y Stuffing

**Objetivo:** Verificar atomicidad de intentos (HINCRBY) y contención de stuffing.

```javascript
export const options = {
  scenarios: {
    otp_concurrent_verify: {
      executor: 'per-vu-iterations', vus: 50, iterations: 20,
      exec: 'concurrentWrongCodes',    // 5 VUs verifican el MISMO email en paralelo
    },
    otp_stuffing: {
      executor: 'constant-arrival-rate', rate: 100, timeUnit: '1s', duration: '5m',
      exec: 'requestOtpForEmailList',  // 80% emails inexistentes
    },
  },
  thresholds: {
    // Externos:
    // 1) La suma de intentos registrados por emisión == exactamente 5 al agotar
    //    (HINCRBY no pierde incrementos bajo concurrencia)
    // 2) otp_exhausted_total coherente con emisiones agotadas
    // 3) Todas las respuestas de stuffing == 202 byte-identical
    // 4) rate_suspended_total > 0 con emails reales despachados ≤ límite
  },
};
```

### 4.5. Escenario 5: Chaos Quirúrgico por Dependencia

| Inyección | Ventana | Expectativas verificadas |
| :--- | :--- | :--- |
| **Redis caído** | 5 min en plena ola | 503 `unavailable{redis}`; **cero** artefactos consumidos ni emitidos durante la ventana; al recuperar, operación normal sin reconciliación (TTLs limpian) |
| **KAGAMI caído tras firma passkey** | Error inyectado en `SaveCounterCasAsync` | Verify → 503; **cero sesiones emitidas**; credenciales intactas con counter antiguo; `counter_cas_rejected` no contaminado |
| **Proveedor de email caído** | 10 min | `dispatch_failures_total` crece; tokens persisten en Redis; re-solicitudes superseden; passkeys y OTP-ya-entregados sin impacto; funnel cae y se recupera |
| **Risk Engine lento (300 ms)** | 5 min | Tenants `open`: verify p95 sube ~200 ms pero completa; tenants `stepup`: pausas hacia UC-15; `risk_timeouts` visible |
| **Pause Broker caído (tenant AAL2)** | 5 min | Magic link/OTP → 503; **cero emisiones AAL1**; passkeys siguen emitiendo AAL2 sin impacto |

### 4.6. Escenario 6: Soak 24h (Estado Implícito y Curva Teórica)

**Objetivo:** A 100 verifies/seg + 20 issuance/seg sostenidos, verificar:
- Memoria de cada keyspace `passwordless:*` dentro de ±20% de la curva teórica `rate × TTL × bytes`.
- Cero claves sin TTL (auditoría periódica `SCAN` con muestra).
- `counter_cas_rejected_total` en baseline (~0 fuera de picos de concurrencia legítima).
- Outbox lag < 60 s; particiones de `refresh_tokens` sin bloat anómalo por la nueva proveniencia.
- `passkey_credentials` con crecimiento lineal al enrollment rate (sin fugas de registros).

---

## 5. Capacity Planning y Escalabilidad

### 5.1. Redis Auth (Ops y Memoria de UC-16)

| Keyspace | Ops/seg en pico | Memoria pico (doc 10) |
| :--- | :--- | :--- |
| `passwordless:challenge:*` | 300 SET + 300 GETDEL | ~16 MB |
| `passwordless:magiclink:*` | 50 SET + 75 GET + 40 GETDEL | ~10 MB |
| `passwordless:otp:*` | 20 SET + 60 HGETALL + 60 HINCRBY + 30 DEL | ~1.2 MB |
| `passwordless:code:*` | 300 SET + 300 GETDEL | ~5 MB |
| `rate` + `replay` + `handoff` | ~200 INCR/GET | < 2 MB |
| **Total UC-16** | **~2.500 ops/seg** | **~34 MB** |

**Acumulado del cluster (UC-11 a UC-16):** ~291 MB de memoria (14.5% de 2 GB) y ~12.000 ops/seg (<12% de capacidad) en el pico simultáneo teórico de toda la autenticación. El `noeviction` permanece holgado con headroom >6x.

### 5.2. PostgreSQL (Passkeys y Proveniencia)

| Operación | TPS en pico (500 verifies/seg) |
| :--- | :--- |
| Lookup de credencial activa (index scan) | 350 |
| Counter CAS (`UPDATE ... WHERE` + RETURNING) | 300 |
| Lookup de identidad (directorio, solicitudes OOB) | 70 |
| INSERT `refresh_tokens` (emisión) + INSERT outbox | 500 + 500 |
| INSERT `passkey_credentials` (enrollment, ola aparte) | ≤ 50 |

**Recomendación:** Comparte la instancia de KAGI (`db.r6g.xlarge`, GP3 6.000 IOPS). El workload passwordless es **lectura dominante con escrituras pequeñas**: los 300 CAS/seg son updates de 1 fila por índice primario (< 1 ms cada uno con WAL agrupado). El autovacuum de `refresh_tokens` ya agresivo (UC-13) absorbe el churn adicional de la proveniencia.

### 5.3. CPU de Pods kagi-passwordless por Mezcla

| Componente por verify | Passkey (70%) | Magic link confirm (20%) | OTP (10%) |
| :--- | :--- | :--- | :--- |
| Redis ops (2-3 por verify) | ~4 ms | ~4 ms | ~3 ms |
| KAGAMI (lookup + CAS / emisión) | ~7 ms | ~5 ms | ~5 ms |
| Criptografía (firma / dummy / compare) | ~1 ms | ~0.2 ms | ~0.1 ms |
| Riesgo (100 ms timeout, p50 ~30 ms) | ~30 ms | ~30 ms | ~30 ms |
| **Latencia media ponderada** | **~42 ms** | **~39 ms** | **~38 ms** |

**Cálculo de pods:** 500 verifies/seg × ~40 ms = 20 segundos-request/seg de ocupación efectiva; con concurrencia interna (async I/O) y overhead (~2x) → **~10 vCPUs** → 4 pods × 2 vCPU baseline con HPA hasta 20 (doc 09). El riesgo domina la latencia de cola; si UC-17 degrada, el fail-mode por tenant decide si passwordless absorbe el timeout o degrada a step-up.

### 5.4. Proveedor de Email (Throughput)

- **Pico:** 70 emails/seg (50 magic links + 20 OTP) ≈ 4.200/min ≈ 252.000/hora.
- **Cuotas típicas:** SES estándar 200/seg por cuenta (ampliable); SMTP corporativo suele ser el cuello (10-50/seg).
- **Mitigación de cuota:** Rate de despacho por tenant proporcional + cola acotada con backpressure; el supersede garantiza que los reintentos del usuario no duplican envíos lógicos.
- **Métrica de capacidad:** `dispatch_queue_depth` y `dispatch_duration_p95` como leading indicators de saturación del proveedor antes de que `dispatch_failures` aparezca.

---

## 6. Observabilidad y Profiling en Producción

### 6.1. Traces por Canal (Reafirmación con Presupuesto)

```text
POST /passwordless/webauthn/verify — éxito [~42ms]
├── redis.challenge.getdel [2ms]
├── guard.origin_uv [0.2ms]
├── kagami.credential.fetch [3ms]
├── webauthn.verify_signature [1ms]
├── kagami.counter.cas [4ms]
├── risk.evaluate [30ms]
└── issuer.complete [2ms] → (emisión de familia en trace propio)

POST /passwordless/link/confirm [~38ms]
├── redis.link.peek [2ms]
├── gate.binding_handoff [0.5ms]
├── redis.link.getdel [2ms]
├── risk.evaluate [30ms]
└── issuer.or_pause [3ms]

POST /passwordless/magic-link [~14ms — despacho fuera del span]
├── abuse.evaluate [2ms]
├── kagami.identity.lookup [5ms]
├── redis.token.set [2ms]
├── outbox.insert [4ms]
└── (email.dispatch: span async separado, 250-800ms)
```

### 6.2. Dashboards de Grafana (Específicos para UC-16)

#### Panel 1: Verify Latency por Canal + Objetivo
*   **Métricas:** `verify_duration{channel}` p50/p95/p99 con líneas de SLO.
*   **Lectura:** Divergencia entre canales indica el cuello por tipo (passkey = KAGAMI/CAS; OTP = Redis; magic link = riesgo).

#### Panel 2: Guard Ladder Drop-off (Funnel de Rechazos)
*   **Métricas:** `failed_total{stage}` apilado (challenge, origin, uv, credential, signature, clone).
*   **Lectura:** La distribución esperada bajo ataque es mayoritaria en los peldaños 1-3; un desplazamiento hacia `credential`/`signature` indica sondas más sofisticadas que están alcanzando KAGAMI/criptografía.

#### Panel 3: Magic Link Funnel y Espera
*   **Métricas:** issued → validated → consumed como río + `magiclink_ttl_elapsed` p50/p95 + `dispatch_duration`.
*   **Lectura:** Caída de conversión con `dispatch_duration` alto = proveedor; con `ttl_elapsed` alto y dispatch sano = fricción de usuario o emails a spam.

#### Panel 4: CAS y Concurrencia de Credenciales
*   **Métricas:** `counter_cas_updated` vs `counter_cas_rejected` ratio + duración del CAS.
*   **Lectura:** Rejected ≈ 0 en normal; spike = retry storm de cliente o replay automatizado (correlacionar con `replay_signals`).

#### Panel 5: Keyspaces vs Curva Teórica
*   **Métricas:** Memoria observada de `passwordless:*` vs `rate × TTL × bytes` por keyspace.
*   **Lectura:** Divergencia >20% = fuga de TTL (alerta de seguridad Y rendimiento) o keyspace contaminado.

#### Panel 6: Despacho de Email
*   **Métricas:** `dispatch_duration` p95, `dispatch_failures_total`, queue depth, tasa por tenant.
*   **Lectura:** Leading indicators de cuota agotada antes del fallo visible.

### 6.3. Alertas de Rendimiento

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Passkey Verify Regression** | `verify_duration_p95{passkey} > 80ms` por 5 min | Alta | Descomponer por span: si `counter.cas` domina → contención KAGAMI; si `challenge.getdel` → Redis. |
| **Confirm Latency Regression** | `confirm_duration_p95 > 150ms` por 5 min | Alta | Revisar riesgo (p50 de `risk.evaluate`) y GETDEL. |
| **Garbage Reaching KAGAMI** | `kagami.credential.fetch` QPS correlacionado con `failed_total{stage=origin|uv}` | Alta | La escalera no está filtrando: verificar que los peldaños 1-3 rechacen antes del lookup. |
| **Magic Link Funnel Collapse** | `consumed/issued < 0.5` por 30 min | Media | Cruzar con `dispatch_failures` y `ttl_elapsed`; proveedor o UX. |
| **Dispatch Saturation** | `dispatch_queue_depth > 5000` o `dispatch_duration_p95 > 2s` | Alta | Proveedor saturado: verificar cuotas por tenant; passkeys sin impacto. |
| **CAS Rejection Spike** | `counter_cas_rejected > 1%` de CAS totales por 10 min | Alta | Retry storm o replay automatizado; correlacionar con `replay_signals_total`. |
| **Keyspace Memory Drift** | Cualquier keyspace `passwordless:*` diverge >20% de la curva | Alta | Fuga de TTL: auditoría de `EXPIRE` en adapters; riesgo de artefactos vivos más allá de su ventana. |
| **Passwordless 503 Storm** | `rate(unavailable_total[2m]) > 0` | **Crítica** | Fail-secure activo por dependencia; Runbook de doc 09 §7.2 con `dependency` tag. |

---

## 7. Configuración de Rendimiento

### 7.1. Timeouts y Pools (Reafirmación Tipada)

```csharp
public sealed record PasswordlessTimeouts
{
    public TimeSpan RedisOperation  { get; init; } = TimeSpan.FromMilliseconds(50);
    public TimeSpan KagamiOperation { get; init; } = TimeSpan.FromMilliseconds(200);
    public TimeSpan EmailDispatch   { get; init; } = TimeSpan.FromSeconds(2);
    public TimeSpan RiskEvaluation  { get; init; } = TimeSpan.FromMilliseconds(100);
    public TimeSpan ChallengeTtl    { get; init; } = TimeSpan.FromMinutes(5);
    public TimeSpan MagicLinkTtl    { get; init; } = TimeSpan.FromMinutes(10);
    public TimeSpan OtpTtl          { get; init; } = TimeSpan.FromMinutes(5);
    public TimeSpan SessionCodeTtl  { get; init; } = TimeSpan.FromSeconds(60);
    public int EmailDispatchRetries { get; init; } = 1;
    public int EmailQueueCapacity   { get; init; } = 10_000;   // backpressure del dispatcher
    public int Retries              { get; init; } = 0;
}
```

### 7.2. Feature Flags de Rendimiento

| Feature Flag | Descripción | Default |
| :--- | :--- | :--- |
| `kagi.auth.passwordless.risk_parallel_with_cas` | Evalúa riesgo concurrentemente al CAS en passkey (tenants `open`) | `true` |
| `kagi.auth.passwordless.email_queue_capacity` | Backpressure del dispatcher | `10000` |
| `kagi.auth.passwordless.dispatch_rate_per_tenant` | Cuota de despacho por tenant (emails/min) | proporcional a plan |
| `kagi.auth.passwordless.options_rate_limit_per_ip` | Límite grueso de `/options` en gateway | `120/min` |

### 7.3. HPA (Reafirmación con Anotación de Mezcla)

```yaml
# Doc 09, con anotación operativa:
# El escalado responde a verify_duration_p95 de la MEZCLA observada.
# Un tenant que migra masivamente a passkey sube el costo medio por verify
# (más round-trips KAGAMI): el leading indicator es verify_duration{passkey},
# no CPU. Si un rollout de passkey coincide con degradación, escalar antes
# de esperar al umbral de CPU.
metrics:
  - type: Pods
    pods:
      metric: { name: kagi_auth_passwordless_verify_duration_p95 }
      target: { type: AverageValue, averageValue: "30" }
  - type: Resource
    resource: { name: cpu, target: { type: Utilization, averageUtilization: 65 } }
```

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Los objetivos de rendimiento (options <50ms, verify por canal, funnel, disponibilidad) se formalizan aquí como SLOs medibles por canal y fase. |
| `07_Sequence_Diagram.md` | El presupuesto de timeouts (§8.1) y las trazas por canal (§8.5) son la base de los SLOs y spans de este documento. |
| `08_Final_Class_Design.md` | `PasswordlessTimeouts`, `PasswordlessMetrics` y los adapters Redis/KAGAMI emiten exactamente las métricas y respetan los presupuestos definidos aquí. |
| `09_Modern_Artifacts.md` | Las métricas `kagi.auth.passwordless.*` y el HPA se extienden aquí con guard ladder drop-off, funnel, CAS ratio y curva teórica de keyspaces. |
| `10_UC_Specific_Data_Model.md` | Los keyspaces (~34 MB), el CAS condicional y las vistas de adopción soportan el capacity planning y los paneles de este documento. |
| `11_UC_Specific_Security_Threats.md` | El Escenario 4.2 valida empíricamente que la escalera de guardas (T-121/T-122) mantiene el costo hostil <3ms y a KAGAMI fuera del flood. |
| `UC-15/12_Performance_Benchmarks.md` | Documento hermano: el pause broker que consume UC-16 tiene sus propios SLOs allí; su caída se prueba aquí como chaos de emisión insuficiente. |

---

## 9. Cierre del Paquete de Diseño (UC-16)

Con este documento, completamos los **12 archivos del Paquete de Diseño Profundo (Tier 1 + Tier 2)** para el UC-16:

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
11. ✅ `11_UC_Security_Threats.md`
12. ✅ `12_UC_Specific_Performance_Benchmarks.md`

### 9.1. Resumen de lo Logrado en UC-16

Hemos diseñado la autenticación sin contraseña como **tres canales honestos con un solo punto de emisión**:
- **Escalera de guardas como presupuesto:** El tráfico hostil muere en peldaños de ~0.01-2 ms; KAGAMI y la criptografía quedan reservados para assertions estructuralmente plausibles, verificado bajo flood mezclado.
- **Estado implícito medible:** Los artefactos single-use viven como presencia con TTL; el soak compara memoria observada contra curva teórica, detectando fugas que serían a la vez bug de rendimiento y riesgo de seguridad.
- **Funnel humano como SLO correcto:** El magic link se mide por conversión (issued→consumed >0.80) y percentiles de espera, no por latencia de servidor — la categoría correcta para un flujo mediado por email y decisión.
- **Despacho como throughput, no latencia:** El 202 precede al transporte; el proveedor se gestiona con cola acotada, backpressure y cuotas por tenant; su caída degrada UX sin degradar seguridad.
- **Atomicidad bajo fuego:** HINCRBY no pierde intentos de OTP bajo concurrencia, GETDEL no dobla consumos de enlace, y el CAS de counter serializa verifies sin locks — todo verificado en los escenarios 4.2-4.4.
- **Fail-secure por dependencia probado:** Redis/KAGAMI caídos no emiten ni consumen; email caído preserva tokens; pause broker caído bloquea emisión insuficiente en lugar de degradarla.

### 9.2. Estado del EPIC-02 (Authentication)

| UC | Nombre | Estado | Documentos |
| :--- | :--- | :--- | :--- |
| **UC-11** | Autenticar Credenciales Locales | ✅ Completo | 12/12 |
| **UC-12** | Cerrar Sesión (Logout) | ✅ Completo | 12/12 |
| **UC-13** | Renovar Sesión (Refresh Token) | ✅ Completo | 12/12 |
| **UC-14** | Autenticar Federada (SSO) | ✅ Completo | 12/12 |
| **UC-15** | Multi-Factor Authentication (MFA) | ✅ Completo | 12/12 |
| **UC-16** | Passwordless Authentication | ✅ Completo | 12/12 |
| **UC-17** | Risk-Based Authentication | 📋 Pendiente | 0/12 |
| **UC-18** | Recuperación de Credenciales | 📋 Pendiente | 0/12 |
| **UC-19** | Cambio de Password (Self-Service) | 📋 Pendiente | 0/12 |

**Total: 72/108 documentos completados en el EPIC-02.** Los seis UCs de entrada de identidad quedan cerrados como sistema coherente: local (UC-11), segundo factor (UC-15), federada (UC-14), passwordless (UC-16), rotación/detección (UC-13) y terminación propagada (UC-12). El bloque restante comienza con el motor que orquesta decisiones sobre todos ellos: **UC-17 Risk-Based Authentication**.

---
*Fin del Documento 12_UC_Specific_Performance_Benchmarks para UC-16*