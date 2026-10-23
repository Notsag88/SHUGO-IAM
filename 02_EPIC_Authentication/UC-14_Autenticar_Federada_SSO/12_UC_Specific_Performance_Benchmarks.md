# UC-14: Autenticar Federada (SSO) - Benchmarks, SLAs y Estrategia de Rendimiento

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-14 - Autenticar Federada (SSO - OIDC/OAuth2 + SAML 2.0) |
| **Documento:** | 12_UC_Specific_Performance_Benchmarks.md |
| **Metodología:** | SRE (Site Reliability Engineering) + Performance Engineering + Load Testing (k6) + Chaos Engineering + Capacity Planning |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 23/10/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 23/10/2026 | Gastón Ruiz | **Creación inicial.** Definición de SLAs, SLOs y estrategia de Load Testing para la autenticación federada. Modela los seis perfiles de carga críticos: ola matutina federada mixta (OIDC+SAML), comparativa de costo criptográfico XML-DSig vs JWT, tormenta JIT de primeros logins, flood SAML malicioso (DoS de XML-DSig), chaos con IdP degradado / metadata envenenada / Redis caído, y soak de 24h con drift de keyspaces. Incluye capacity planning de PostgreSQL (escrituras por callback + subset JIT), Redis (cuatro keyspaces de federación), pools HTTP por host de IdP, y dimensionamiento de CPU por mezcla de protocolos. | Documento de referencia para SREs y Arquitectos de Rendimiento. Cierra el UC-14 garantizando que la federación —con su dependencia externa heterogénea y su criptografía XML costosa— mantenga SLOs locales estrictos, que el costo del atacante permanezca acotado por rechazos pre-firma, y que los vuelos legítimos nunca compitan por recursos con artefactos hostiles. |

---

## 1. Introducción y Propósito

El **UC-14 (Autenticar Federada)** presenta el perfil de rendimiento más heterogéneo del motor KAGI, por cuatro razones estructurales:

1. **Dos criptografías con costos asimétricos:** Validar un `id_token` EdDSA cuesta ~1 ms de CPU; validar una SAML Assertion con XML-DSig exclusive-c14n cuesta ~8-10 ms (canonicalización domina). La mezcla de protocolos por tenant determina el costo de CPU por callback, y el flood SAML es el vector de DoS asimétrico natural.
2. **Una dependencia externa no controlable:** El IdP domina la latencia del token exchange (front-channel y back-channel). El SLO local debe **excluir** lo no controlable y **acotar** lo controlable: timeouts, breakers y estados de confianza.
3. **Un camino cross-context costoso:** El JIT Provisioning invoca a KAGAMI (UC-01) con transacción identidad+link, añadiendo ~120 ms en el subconjunto de primeros logins — un costo que aparece solo en olas de onboarding.
4. **Cuatro keyspaces Redis efímeros:** Tx State cifrado, Replay Cache, FederateCode y Trust State compiten por memoria y ops con los keyspaces de UC-11/12/13 en la misma instancia `noeviction`.

Este documento aborda estos desafíos con SLOs por fase y protocolo, benchmarks comparativos, y escenarios de carga que separan tráfico legítimo de hostil en la medición.

---

## 2. Service Level Indicators (SLIs) y Objectives (SLOs)

### 2.1. SLIs Específicos para UC-14

| SLI | Descripción | Fórmula / Medición (OpenTelemetry) |
| :--- | :--- | :--- |
| **Start Latency (p95)** | Latencia local de `/federate/start` (realm discovery + whitelist + tx save + redirect build). | `histogram_quantile(0.95, rate(kagi_auth_federation_duration_bucket{phase="start"}[5m]))` |
| **Callback Local Latency (p95/p99)** | Procesamiento local del callback **excluyendo** front-channel; **incluye** token exchange back-channel al IdP. | `histogram_quantile(0.95, rate(kagi_auth_federation_duration_bucket{phase="callback_local"}[5m]))` |
| **Artifact Validate Duration (p95)** | Validación criptográfica+semántica del artefacto, por protocolo. | `histogram_quantile(0.95, rate(kagi_auth_federation_id_token_validate_duration_bucket[5m]))` (OIDC) / `..._assertion_validate_duration_bucket` (SAML) |
| **IdP Token Exchange (p95)** | Round-trip back-channel al token endpoint del IdP, por conexión. | `histogram_quantile(0.95, rate(kagi_auth_federation_idp_token_exchange_duration_bucket{connection_id}[5m]))` |
| **JIT Gateway Latency (p95)** | Comando cross-context a KAGAMI (identidad+link atómicos). | `histogram_quantile(0.95, rate(kagi_auth_federation_jit_provision_duration_bucket[5m]))` |
| **JWKS/Metadata Cache Hit Ratio** | Proporción de validaciones servidas desde cache sin refetch. | `1 - rate(jwks_refetch_total[1h]) / rate(callback_total[1h])` |
| **Redeem Latency (p95)** | Canje de `federate_code` (GETDEL + mint de tokens). | `histogram_quantile(0.95, rate(kagi_auth_federation_duration_bucket{phase="redeem"}[5m]))` |
| **Back-Channel Latency (p95)** | Validación de logout_token + revocación de familias + ack. | `histogram_quantile(0.95, rate(kagi_auth_federation_duration_bucket{phase="backchannel"}[5m]))` |
| **User Journey Duration (p50/p95)** | Duración del viaje front-channel completo (start→callback), correlacionado por `transaction_id`. | `histogram_quantile(0.95, rate(kagi_auth_federation_user_journey_duration_bucket[24h]))` |
| **CPU por Callback (media)** | Costo de CPU segregado por protocolo (detección de asimetría de ataque). | `rate(container_cpu_usage_seconds{pod=~"kagi-federation.*"}[5m]) / rate(callback_total[5m])` |

### 2.2. SLOs y SLAs

| Métrica | SLO (Objetivo Interno) | SLA (Contrato Enterprise) | Justificación |
| :--- | :--- | :--- | :--- |
| **Start Latency (p95)** | **< 50 ms** | **< 100 ms** | Todo local: realm GIN scan + whitelist + SET Redis + build redirect. |
| **Callback Local (p95)** | **< 200 ms** | **< 400 ms** | Suma de spans back-channel: consume 2 + exchange 45 + validate 1-10 + replay 4 + link 8 + risk 30 + family 12 + code 2 + outbox 6 ≈ 110-120 ms con margen. |
| **Callback Local (p99)** | **< 400 ms** | **< 800 ms** | Cola de JIT gateway y contención de pools HTTP por conexión. |
| **OIDC Validate (p95)** | **< 3 ms** | **< 8 ms** | Verificación EdDSA/RS256 + claims; JWKS en cache. |
| **SAML Validate (p95)** | **< 15 ms** | **< 30 ms** | XML-DSig exclusive-c14n sobre assertion ≤ 64 KB con rechazos pre-firma. |
| **JIT Gateway (p95)** | **< 500 ms** | **< 1 s** | Transacción identidad+link en KAGAMI + mapeo de roles. |
| **Redeem (p95)** | **< 50 ms** | **< 100 ms** | GETDEL Redis + mint de par de tokens (UC-11/13). |
| **Back-Channel (p95)** | **< 50 ms** | **< 100 ms** | Validación de firma + lookup por `idp_session_id` (índice parcial) + revocación UC-12. |
| **Cache Hit Ratio** | **> 99%** | **> 98%** | Refetch solo en rollover legítimo o sonda de ataque. |
| **Throughput federado (pico)** | **500 callbacks/seg + 200 starts/seg** | **250 + 100** | Apertura de jornada enterprise con mezcla 80/20 OIDC/SAML. |
| **Disponibilidad local** | **99.99%** | **99.95%** | La dependencia del IdP se aísla por conexión (breaker + DEGRADED_IDP). |
| **Tokens emitidos con clave no pineada** | **0 (siempre)** | **0 (siempre)** | Invariante de seguridad: cuarentena antes que emisión (T-98). |

---

## 3. Cuellos de Botella y Estrategias de Optimización

### 3.1. Cuello de Botella #1: Asimetría de Costo XML-DSig vs JWT

**Problema:** Validar una Assertion SAML (canonicalización exclusiva + verificación RSA-SHA256 sobre nodo firmado) cuesta ~8-10 ms de CPU vs ~1 ms de un id_token. Con 20% de tráfico SAML en el pico (100 callbacks/seg), son ~1 core dedicado solo a validación XML — y un flood SAML multiplica ese costo por request hostil.

**Estrategias de Optimización:**

#### A. Rechazo Pre-Firma (Costo del Atacante Acotado)
Cap de 64 KB, conteo de assertions, presencia de firma hija y parser estricto sin DTD se evalúan **antes** de cualquier canonicalización (doc 11, T-97/T-104). El costo por request hostil queda en ~0.2 ms de parse, no 10 ms de firma.

#### B. Reutilización de Configuración de Parser y Certificados
- `XmlReaderSettings` y `SignedXml` contexts se construyen **una vez por conexión** y se poolan; la cadena de certificados del metadata se pre-valida y cachea junto al metadata (no en cada validación).
- La canonicalización opera sobre el nodo firmado localizado por ID, no sobre el documento completo.

#### C. Mezcla de Protocolos como Variable de Capacidad
El HPA y el capacity plan usan la **mezcla observada** (`callback_total{protocol}`): un tenant que migra de SAML a OIDC libera CPU sin cambiar volumen. La recomendación arquitectónica para conexiones nuevas es OIDC; SAML queda para IdPs legacy, y su costo está medido y alertado (`assertion_validate_duration` p95 > 15 ms = regresión de parser o assertions anormalmente grandes).

### 3.2. Cuello de Botella #2: Token Exchange contra IdPs Heterogéneos

**Problema:** El exchange back-channel es el único round-trip IdP dentro del SLO observacional. IdPs lentos (200-800 ms en horas pico del proveedor) elevan el p99 de callback y consumen sockets del pool HTTP por conexión.

**Estrategias de Optimización:**
- **Pool HTTP por host:** `SocketsHttpHandler` con `MaxConnectionsPerServer = 64` por IdP; concurrencia esperada = arrival × latencia = 500/seg × 45 ms ≈ 23 sockets en uso medio, 64 absorbe picos de 3x sin queueing.
- **Timeout 5 s sin retry** (política global `Retries = 0`): un IdP lento degrada a `idp_unavailable` fail-secure en lugar de acumular sockets colgados.
- **Breaker por conexión:** 5 fallos/60 s abren el breaker y transicionan a `DEGRADED_IDP`; el tráfico de **otras conexiones** no se afecta (aislamiento por tenant/IdP).
- **Observabilidad por conexión:** `idp_token_exchange_duration{connection_id}` permite negociar SLAs con owners de IdP y detectar degradación del proveedor antes que los usuarios.

### 3.3. Cuello de Botella #3: JIT Gateway en Olas de Onboarding

**Problema:** En olas de incorporación (inicio de trimestre, adquisiciones), el subconjunto de primeros logins puede llegar al 30-60% de los callbacks (vs ~5% en régimen). Cada JIT añade ~120 ms de gateway cross-context con transacción identidad+link en KAGAMI.

**Estrategias de Optimización:**
- **SLO propio y segregado:** `jit_provision_duration` p95 < 500 ms se mide aparte; el SLO de callback_local se evalúa **por resultado** (`result=jit` vs `result=linked`), evitando que el onboarding masque degradación del camino caliente.
- **Índices de KAGAMI para el gateway:** unicidad de email y creación de identidad usan índices existentes de UC-01; el link se crea en la misma transacción (ADR-14-01), sin round-trips adicionales desde KAGI.
- **Backpressure natural:** El gateway respeta `KagamiOperation = 200 ms` de timeout; bajo tormenta JIT, los excesos fallan como `federate_failed` genérico con reintento del usuario (el vuelo de federación puede re-iniciarse), nunca encolados sin límite.

### 3.4. Cuello de Botella #4: Outbox bajo Volumen Federado

**Problema:** Cada callback exitoso inserta 1+ eventos (`FederatedSucceeded` + opcionalmente `JitProvisioned`/`Account.Linked`); una ola de 500/seg con 5% JIT son ~535 inserts/seg de outbox adicionales a los de UC-11/12/13.

**Estrategias de Optimización:**
- **Inserción en la misma TX que `family_create`** (atomicidad no negociable); el costo es 1 write adicional por commit ya agrupado.
- **Workers con batch 100 y poll 500 ms:** 2 workers drenan 400 eventos/seg sostenidos; bajo pico, el lag crece transitoriamente y se drena en <60 s (SLO de outbox lag heredado de UC-12/13).
- **Particionamiento mensual** de `auth_outbox` (UC-11): el scan del worker toca solo la partición actual vía índice parcial `idx_ao_unprocessed`.

### 3.5. Cuello de Botella #5: Cifrado AEAD del Transaction State

**Problema:** El payload de `federate:tx:*` se cifra/descifra AES-256-GCM en cada start/callback (doc 10, §4.2).

**Estrategia:** El costo medido es ~0.05 ms por seal/unseal con claves KMS cacheadas en memoria (data keys locales con rotación), irrelevante frente al SLO de 50 ms de start. Se monitorea `tx_seal_duration` solo para detectar regresiones de KMS (ej. data key expirada que force fetch remoto por operación: p99 > 2 ms alerta).

---

## 4. Estrategia de Load Testing (k6)

### 4.1. Escenario 1: Ola Matutina Federada Mixta (500 callbacks/seg)

**Objetivo:** Validar SLOs de start/callback/redeem con mezcla 80% OIDC / 20% SAML y mock de IdP a 40 ms, manteniendo `trust_state == 0` y `replay_rejected == 0`.

```javascript
import http from 'k6/http';
import { check } from 'k6';
import { Trend, Rate } from 'k6/metrics';

const startLatency = new Trend('federation_start', true);
const callbackLocal = new Trend('federation_callback_local', true);
const errors = new Rate('errors');

export const options = {
  scenarios: {
    federated_morning_wave: {
      executor: 'ramping-arrival-rate',
      startRate: 100, timeUnit: '1s',
      preAllocatedVUs: 200, maxVUs: 1200,
      stages: [
        { target: 250, duration: '2m' },
        { target: 500, duration: '4m' },   // pico de apertura (callbacks)
        { target: 500, duration: '4m' },
        { target: 0,   duration: '2m' },
      ],
      exec: 'federatedWave',
    },
  },
  thresholds: {
    federation_start: ['p(95)<50'],
    federation_callback_local: ['p(95)<200', 'p(99)<400'],
    errors: ['rate<0.005'],
    // Externos post-run: trust_state{connection_id=*} == 0; replay_rejected_total == 0
  },
};

export function federatedWave() {
  // 1) start → Location + tx cookie
  const s = http.post(`${__ENV.BASE_URL}/v1/auth/federate/start`,
    JSON.stringify({ email: `wave${__VU}@khcs.com`, redirect_uri: __ENV.REDIRECT }));
  startLatency.add(s.timings.duration);

  // 2) Mock IdP firma artefacto (OIDC 80% / SAML 20%) con latencia 40ms
  const artifact = mockIdpIssue(s.headers['Location'], __VU);

  // 3) callback → medir solo procesamiento local
  const t0 = Date.now();
  const cb = postCallback(artifact);            // GET code+state o POST SAMLResponse
  callbackLocal.add(Date.now() - t0);

  if (check(cb, { 'callback 302': (r) => r.status === 302 })) {
    // 4) redeem del federate_code
    const code = extractCode(cb.headers['Location']);
    const rd = http.post(`${__ENV.BASE_URL}/v1/auth/federate/token`,
      JSON.stringify({ code }), { headers: { 'Content-Type': 'application/json' } });
    check(rd, { 'redeem 200': (r) => r.status === 200 });
  } else errors.add(1);
}
```

### 4.2. Escenario 2: Comparativa de Costo Criptográfico (SAML vs OIDC)

**Objetivo:** A igual tasa (200/seg por protocolo), medir `assertion_validate` vs `id_token_validate` y CPU por callback; verificar que el rechazo pre-firma mantiene el costo hostil acotado.

```javascript
export const options = {
  scenarios: {
    oidc_only: { executor: 'constant-arrival-rate', rate: 200, timeUnit: '1s',
                 duration: '5m', exec: 'oidcFlow', preAllocatedVUs: 100, maxVUs: 400 },
    saml_only: { executor: 'constant-arrival-rate', rate: 200, timeUnit: '1s',
                 duration: '5m', exec: 'samlFlow', preAllocatedVUs: 100, maxVUs: 400 },
  },
  thresholds: {
    // Externos por corrida (Prometheus):
    // oidc:  id_token_validate p95 < 3ms
    // saml:  assertion_validate p95 < 15ms
    // cpu_per_callback{saml} / cpu_per_callback{oidc} < 10x   (asimetría esperada acotada)
  },
};
```

### 4.3. Escenario 3: Tormenta JIT (Onboarding Masivo)

**Objetivo:** 60% de primeros logins (JIT) a 300 callbacks/seg; validar `jit_provision_duration` p95 < 500 ms y que el camino `linked` (no-JIT) concurrente no degrade (> p95 200 ms).

```javascript
export const options = {
  scenarios: {
    jit_storm:   { executor: 'constant-arrival-rate', rate: 180, timeUnit: '1s',
                   duration: '8m', exec: 'firstLoginFlow', preAllocatedVUs: 150, maxVUs: 500 },
    linked_base: { executor: 'constant-arrival-rate', rate: 120, timeUnit: '1s',
                   duration: '8m', exec: 'returningFlow', preAllocatedVUs: 100, maxVUs: 300 },
  },
  thresholds: {
    jit_gateway: ['p(95)<500'],
    returning_callback_local: ['p(95)<200'],   // el camino caliente no se contamina
  },
};
```

### 4.4. Escenario 4: Flood SAML Hostil + Tráfico Legítimo Mezclado

**Objetivo:** 2.000 req/seg de SAMLResponses hostiles (oversized, DOCTYPE, multi-assertion) contra el ACS, mezcladas con 200/seg legítimas. Verificar: (a) CPU por request hostil < 0.5 ms (rechazo pre-firma), (b) callbacks legítimos mantienen p95 < 200 ms, (c) WAF y validador contabilizan rechazos sin cuarentena falsa de la conexión legítima.

```javascript
export const options = {
  scenarios: {
    hostile_flood:  { executor: 'constant-arrival-rate', rate: 2000, timeUnit: '1s',
                      duration: '6m', exec: 'hostileSaml', preAllocatedVUs: 300, maxVUs: 800 },
    legit_mixed:    { executor: 'constant-arrival-rate', rate: 200, timeUnit: '1s',
                      duration: '6m', exec: 'returningFlow', preAllocatedVUs: 100, maxVUs: 300 },
  },
  thresholds: {
    legit_callback_local: ['p(95)<200'],      // legítimos aislados del flood
    // Externos: cpu_per_request_hostile < 0.5ms; trust_state(conexión legítima) == 0
    // (un flood NO debe cuarentenar la conexión: los rechazos son pre-firma, no fallos de firma)
  },
};
```

**Nota de diseño crítica:** El flood hostil produce rechazos `response_too_large` / `assertion_count_*`, **no** fallos de firma post-refetch; por tanto la máquina `ConnectionTrust` permanece en `TRUSTED`. Cuarentenar por flood sería un auto-DoS: la distinción entre "artefacto estructuralmente inválido" (ruido/ataque) y "firma que no valida tras refetch" (integridad) es lo que mantiene la cuarentena como señal de compromiso, no de volumen.

### 4.5. Escenario 5: Chaos - IdP Degradado, Metadata Envenenada, Redis Caído

| Inyección | Ventana | Expectativas verificadas |
| :--- | :--- | :--- |
| Latencia IdP token exchange = 6 s (> timeout 5 s) | min 4-9 | 503 `idp_unavailable` fail-secure; `DEGRADED_IDP`; sesiones vigentes y UC-13 refresh intactos; back-channel operativo con cache |
| JWKS sirviendo clave del atacante (poisoning) | min 9-12 | Cuarentena en el **primer** artefacto con kid desconocido que no re-valida; `trust_state=3`; **cero familias emitidas** con clave atacante (verificación externa sobre `refresh_tokens`); salida solo por re-pin manual |
| Redis down | min 12-16 | Starts y callbacks rechazados fail-secure; sin tx residuales al recuperar (TTL natural); sin reconciliación requerida |
| Risk Engine con latencia 300 ms | min 16-19 | Tenants `fail_mode=stepup`: step-ups; tenants `open`: rotaciones con `risk_timeouts_total{open}` |

### 4.6. Escenario 6: Soak 24h (Drift de Keyspaces y Salud de Vacuum)

**Objetivo:** A 100 callbacks/seg sostenidos, verificar: memoria Redis de `federate:*`/`replay:*` dentro de ±20% de la curva teórica; `n_dead_tup/n_live_tup` de `refresh_tokens` < 0.20 (churn de familias federadas); outbox lag < 60 s; `trust_state` estable en 0; cero `replay_rejected` espurios.

---

## 5. Capacity Planning y Escalabilidad

### 5.1. PostgreSQL (Escrituras por Callback)

| Operación por callback exitoso | Count | TPS en pico (500 cb/seg) |
| :--- | :--- | :--- |
| SELECT link por `(issuer, subject)` (índice único) | 1 | 500 |
| INSERT `refresh_tokens` (familia) + INSERT outbox | 2 | 1.000 |
| Subset JIT (5%): gateway KAGAMI (identidad+link, TX remota) | 0.05 | 25 |
| SELECT familias por `idp_session_id` (back-channel, pico aparte) | — | ≤ 50 |
| **Total escrituras locales** | **2** | **~1.000 writes/seg + 500 commits/seg** |

**Recomendación:** Comparte instancia con UC-11/12/13 (`db.r6g.xlarge`, GP3 6.000 IOPS): el incremento federado es <20% del writes/seg combinado del motor KAGI. Autovacuum de `refresh_tokens` ya agresivo (UC-13) absorbe el churn adicional de familias federadas.

### 5.2. Redis Auth (Keyspaces de Federación + Ops)

| Keyspace | Entradas concurrentes pico | Memoria | Ops/seg pico |
| :--- | :--- | :--- | :--- |
| `federate:tx:*` | 200 starts/seg × 600 s = 120.000 | ~55 MB | 200 SET + 500 GETDEL |
| `replay:*` | 500 cb/seg × 600 s = 300.000 | ~36 MB | 500 SETNX |
| `federate:code:*` | 500/seg × 60 s = 30.000 | ~9 MB | 500 SET + 500 GETDEL |
| `federate:trust:*` | = conexiones (cientos) | < 1 MB | CAS esporádicos |
| **Total federación** | — | **~100 MB** | **~2.700 ops/seg** |

Sumado a UC-11/12/13 (~130 MB en ola combinada): **< 15% de la instancia de 2 GB `noeviction`**, con headroom de ops >10x (Redis sirve >100k ops/seg por nodo).

### 5.3. CPU de Pods KAGI-Federation por Mezcla de Protocolos

| Componente por callback | OIDC (80%) | SAML (20%) |
| :--- | :--- | :--- |
| Validación criptográfica | ~1 ms | ~9 ms |
| Parse/estructura | ~0.2 ms | ~1.5 ms |
| Resto (serialización, AEAD tx, mint) | ~2 ms | ~2 ms |
| **CPU total por callback** | **~3.2 ms** | **~12.5 ms** |

**Cálculo de pods:** 500 cb/seg × (0.8×3.2 + 0.2×12.5) ms ≈ 500 × 5.1 ms = **2.55 cores** de CPU pura + overhead de red/serialización (~2x) ≈ **5-6 vCPUs** → 6 pods × 2 vCPU (min 4, max 24 vía HPA del doc 09). Un shift de mezcla a 50% SAML elevaría el requerimiento a ~8 cores: la métrica `cpu_per_callback{protocol}` es el leading indicator para renegociar HPA o migrar conexiones legacy a OIDC.

### 5.4. Pools HTTP por Host de IdP

Concurrencia media de exchanges = 500/seg × 45 ms ≈ 23 sockets; `MaxConnectionsPerServer = 64` por host con `SocketsHttpHandler` multiplexado absorbe picos de 3x sin queueing. IdPs con latencia p95 de 300 ms elevarían la concurrencia a ~150: el breaker por conexión (5 fallos/60 s) protege el pool antes que el queueing degrade a todos.

---

## 6. Observabilidad y Profiling en Producción

### 6.1. Traces por Protocolo

```text
POST /v1/auth/federate/callback — OIDC (Root) [Total local: 85ms]
├── redis.tx.consume [2ms]
├── idp.token_exchange [45ms]
├── trust.jwks.resolve [3ms]          ← cache hit; refetch = +200ms y sonda visible
├── oidc.id_token.validate [1ms]
├── redis.replay.check_write [4ms]
├── kagami.link_lookup [8ms]
├── risk.evaluate [30ms]
├── kagami.family_create [12ms]       ← misma TX que outbox.insert
├── redis.federate_code.set [2ms]
└── outbox.insert [6ms]

POST /v1/auth/federate/callback — SAML (Root) [Total local: 96ms]
├── redis.tx.consume [2ms]
├── saml.structural_precheck [1ms]    ← rechazos pre-firma terminan aquí (~0.3ms)
├── trust.saml_metadata.resolve [3ms]
├── saml.xmldsig.validate [9ms]       ← exclusive-c14n + RSA-SHA256 + cobertura de nodos
├── saml.semantics.validate [1ms]     ← Destination/Audience/Recipient/Conditions/InResponseTo
├── redis.replay.check_write [4ms]
├── kagami.link_lookup [8ms]
├── risk.evaluate [30ms]
├── kagami.family_create [12ms]
├── redis.federate_code.set [2ms]
└── outbox.insert [6ms]
```

### 6.2. Dashboards de Grafana (Específicos para UC-14)

#### Panel 1: Costo Criptográfico por Protocolo
*   **Métricas:** `id_token_validate_duration` vs `assertion_validate_duration` (p50/p95) + `cpu_per_callback{protocol}`.
*   **Lectura:** Una subida de `assertion_validate` sin subida de tamaño de assertion indica regresión de parser; una subida de tamaño indica IdP emitiendo assertions gordas (grupos embebidos) → revisar mapping.

#### Panel 2: Salud de IdPs por Conexión
*   **Métricas:** `idp_token_exchange_duration{connection_id}` p95 + `trust_state{connection_id}` como banda de fondo.
*   **Lectura:** Permite negociar SLAs con owners de IdP y anticipar `DEGRADED_IDP` antes del breaker.

#### Panel 3: Keyspaces de Federación vs Curva Teórica
*   **Métricas:** memoria de `federate:tx/replay/code` vs `rate(starts|callbacks) × TTL × bytes/entrada`.
*   **Lectura:** Divergencia >20% = fugas de TTL o keyspace contaminado (auditar PX/NX de los adapters).

#### Panel 4: JIT y Onboarding
*   **Métricas:** `jit_provision_duration` p95 + `jit_provisioned_total` rate + proporción `callback_total{result=jit}`.
*   **Lectura:** Distingue ola de onboarding legítima (proporción alta sostenida) de tormenta anómala (spike abrupto → alerta de doc 09).

#### Panel 5: Timeline de Confianza
*   **Métricas:** `trust_state` por conexión + `jwks_refetch_total{outcome}` apilado.
*   **Lectura:** Historia completa de rollovers, sondas y cuarentenas por conexión: el artefacto de post-incidente para Runbook 1.

### 6.3. Alertas de Rendimiento

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **SAML Validate Regression** | `assertion_validate_duration_p95 > 15ms` por 5 min | Alta | Regresión de parser o assertions anormalmente grandes; revisar mezcla y tamaños; verificar que rechazos pre-firma sigan activos. |
| **IdP Exchange Degraded** | `idp_token_exchange_duration_p95{conn} > 2s` por 5 min | Alta | Anticipa breaker/DEGRADED_IDP; notificar owner del IdP; evaluar `local_fallback`. |
| **JIT Gateway Slow** | `jit_provision_duration_p95 > 500ms` por 5 min | Alta | Contención en KAGAMI o mapping pesado; revisar TPS de gateway y locks de unicidad de email. |
| **Callback Local Breach** | `callback_local_p95 > 200ms` por 5 min | Alta | Descomponer por span en Panel 1/6.1: identificar si es validate, exchange, risk o family_create. |
| **Cache Hit Ratio Low** | `jwks_cache_hit_ratio < 0.99` por 10 min | Alta | Refetches anormales: sondeo de kids falsos o misconfig de cache; cruzar con `jwks_refetch_total{outcome}`. |
| **CPU Asymmetry Spike** | `cpu_per_callback{saml} / cpu_per_callback{oidc} > 10` | Media | Posible flood pre-firma no bloqueado por WAF; verificar reglas de tamaño/DOCTYPE. |

---

## 7. Configuración de Rendimiento

### 7.1. HTTP y Parsers

```csharp
// SocketsHttpHandler por host de IdP (exchange back-channel y refetch de metadata)
var handler = new SocketsHttpHandler
{
    MaxConnectionsPerServer = 64,
    PooledConnectionLifetime = TimeSpan.FromMinutes(5),   // DNS rotation del IdP
    PooledConnectionIdleTimeout = TimeSpan.FromMinutes(2),
    EnableMultipleHttp2Connections = true,
    ConnectTimeout = TimeSpan.FromSeconds(2)
};

// Parser SAML: settings inmutables y pool de contextos de validación
public static readonly XmlReaderSettings StrictSamlSettings = new()
{
    DtdProcessing = DtdProcessing.Prohibit,
    XmlResolver = null,
    MaxCharactersInDocument = 65_536,
    ConformanceLevel = ConformanceLevel.Document
};
```

### 7.2. Feature Flags de Rendimiento

| Feature Flag | Descripción | Default |
| :--- | :--- | :--- |
| `kagi.auth.federation.saml_max_response_bytes` | Cap pre-parse de SAMLResponse | `65536` |
| `kagi.auth.federation.http_max_connections_per_idp` | Pool por host de IdP | `64` |
| `kagi.auth.federation.jit_gateway_timeout_ms` | Timeout del gateway cross-context | `200` |
| `kagi.auth.federation.outbox_worker_batch` | Batch del worker de outbox | `100` |
| `kagi.auth.federation.tx_payload_encryption` | AEAD del payload de tx (no desactivable en prod) | `true` (bloqueado) |

### 7.3. HPA (Reafirmación con Mezcla de Protocolos)

```yaml
# Del doc 09, con anotación de operación:
# Si cpu_per_callback{saml} sostenido > 12ms por cambio de mezcla de tenants,
# escalar minReplicas de 4 → 6 o migrar conexiones legacy a OIDC (decisión de arquitectura).
metrics:
  - type: Pods
    pods:
      metric: { name: kagi_auth_federation_duration_p95 }
      target: { type: AverageValue, averageValue: "200" }
  - type: Resource
    resource: { name: cpu, target: { type: Utilization, averageUtilization: 65 } }
```

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Los objetivos de rendimiento (start <50ms, callback <200ms, JIT <500ms, cache >99%) se formalizan aquí como SLOs medibles por fase y protocolo. |
| `07_Sequence_Diagram.md` | El presupuesto de timeouts y los spans por canal (§7.1 de ese doc) son la base de los traces y SLOs de este documento. |
| `09_Modern_Artifacts.md` | Las métricas `kagi.auth.federation.*`, el HPA y los escenarios k6 preliminares se extienden aquí con comparativa de protocolos, tormenta JIT y flood hostil mezclado. |
| `10_UC_Specific_Data_Model.md` | Los keyspaces Redis y sus capacidades (~100 MB federación) se validan aquí bajo ola; los índices de links y proveniencia soportan los lookups medidos. |
| `11_UC_Specific_Security_Threats.md` | El rechazo pre-firma (T-97/T-104) se valida empíricamente en el Escenario 4: costo hostil <0.5 ms y **sin cuarentena falsa** de la conexión legítima. |
| `UC-13/12_UC_Specific_Performance_Benchmarks.md` | Documento hermano: el soak de `refresh_tokens` (vacuum, generations) ahora incluye el churn de familias federadas con proveniencia. |

---

## 9. Cierre del Paquete de Diseño (UC-14)

Con este documento, completamos los **12 archivos del Paquete de Diseño Profundo (Tier 1 + Tier 2)** para el UC-14:

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

### 9.1. Resumen de lo Logrado en UC-14

Hemos diseñado a SHUGO como **Relying Party / Service Provider de grado enterprise**:
- **Confianza delimitada y observable:** Metadata pineada con sonda de una bala, cuarentena sin auto-recuperación, y estado de confianza por conexión como métrica y dashboard de primer nivel.
- **Dos protocolos, un artefacto canónico:** OIDC y SAML convergen en `IdpArtifact`; la asimetría de costo criptográfico está medida, alertada y acotada por rechazos pre-firma.
- **Federación segura por orden y por canal:** Consumo single-use pre-round-trip, replay post-validación fail-secure, step-up pre-familia, tokens fuera de URLs, y back-channel con contrato 200 idempotente.
- **Identidades desde fuera con privilegios cerrados:** JIT con mapping declarativo sin admin, linking por email verificado con notificación al titular, y dominios por conexión.
- **Resiliencia asimétrica documentada:** IdP (fail-secure + fallback por política), Redis (fail-secure absoluto), Risk (fail-mode por tenant), metadata (cuarentena manual) — cada degradación con comportamiento, alerta y runbook propios.
- **Rendimiento verificado bajo hostilidad:** Flood SAML mezclado con tráfico legítimo que prueba el aislamiento de costos, y chaos que valida cuarentena, fail-secure y recuperación sin estado residual.

### 9.2. Estado del EPIC-02 (Authentication)

| UC | Nombre | Estado | Documentos |
| :--- | :--- | :--- | :--- |
| **UC-11** | Autenticar Credenciales Locales | ✅ Completo | 12/12 |
| **UC-12** | Cerrar Sesión (Logout) | ✅ Completo | 12/12 |
| **UC-13** | Renovar Sesión (Refresh Token) | ✅ Completo | 12/12 |
| **UC-14** | Autenticar Federada (SSO) | ✅ Completo | 12/12 |
| **UC-15** | Multi-Factor Authentication (MFA) | 📋 Pendiente | 0/12 |
| **UC-16** | Passwordless Authentication | 📋 Pendiente | 0/12 |
| **UC-17** | Risk-Based Authentication | 📋 Pendiente | 0/12 |
| **UC-18** | Recuperación de Credenciales | 📋 Pendiente | 0/12 |
| **UC-19** | Cambio de Password (Self-Service) | 📋 Pendiente | 0/12 |

**Total: 48/108 documentos completados en el EPIC-02.** Los cuatro pilares de entrada de identidad quedan cerrados como sistema coherente: credenciales locales (UC-11), federación (UC-14), rotación y detección de abuso (UC-13), y terminación de sesiones con propagación bidireccional (UC-12).

---
*Fin del Documento 12_UC_Specific_Performance_Benchmarks para UC-14*