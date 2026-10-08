# UC-13: Renovar Sesión (Refresh Token) - Artefactos Modernos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-13 - Renovar Sesión (Refresh Token Rotation & Reuse Detection) |
| **Documento:** | 09_Modern_Artifacts.md |
| **Metodología:** | API-First + Observability (OpenTelemetry) + Resilience Patterns + DevSecOps + Chaos Engineering |
| **Estándares:** | RFC 6749 §1.5/§6, RFC 6819 §5.2.2.3 (Reuse Detection), NIST SP 800-63B, OWASP ASVS 4.0 (V3), SOC 2 CC6.1/CC7.2, ISO 27001:2022 A.5.18/A.8.15, GDPR Art. 32 |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 07/10/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 07/10/2026 | Gastón Ruiz | **Creación inicial.** Definición de los artefactos modernos para UC-13: contrato OpenAPI 3.1 de `POST /auth/refresh` con matriz de colapso Zero Info Leakage, catálogo de métricas OpenTelemetry (rotaciones, resoluciones de Grace Window, detecciones de reuso, revocaciones de familia, duración de lock de fila), patrones de resiliencia (Fail-Secure en DB, Circuit Breakers diferenciados para Grace Cache y Blacklist, reconciliación de Blacklist desde eventos), Feature Flags de política por tenant, estrategia de testing con concurrencia multi-tab y simulación de reuso, y runbooks de incidente. | Cierra el paquete de diseño Tier 1 funcional del UC-13, proporcionando todos los artefactos necesarios para implementar, operar, monitorear y auditar la rotación de Refresh Tokens con detección de reuso de grado enterprise. |

---

## 1. Introducción y Propósito

Este documento define los **Artefactos Modernos** necesarios para operar el UC-13 (Renovar Sesión) en producción. El endpoint de refresh es el **más frecuentemente invocado del motor KAGI** (cada cliente lo llama cada ~15 minutos por sesión activa), y además es el mecanismo de detección de robo de sesiones. Por tanto, sus artefactos deben satisfacer tres demandas simultáneas:

1. **Contrato estable y opaco:** La respuesta debe ser idéntica para siete rechazos internos distintos (Zero Info Leakage), mientras las métricas —no los códigos HTTP— cuentan la verdad operativa.
2. **Observabilidad de la cadena:** Métricas que distingan rotaciones estándar, resoluciones de Grace Window, step-ups y contenciones, y que expongan la **duración del lock de fila** (el recurso contencioso crítico del UC).
3. **Resiliencia diferenciada por dependencia:** KAGAMI DB (Fail-Secure: 503), Grace Cache de Redis (fail-open conservador: rechazo sin revocación), Blacklist de Redis (Fail-Safe: contención duradera con ventana residual alertada), y Risk Engine (fail-open configurable por tenant).

---

## 2. Contratos OpenAPI Completos

### 2.1. Especificación OpenAPI 3.1

```yaml
openapi: 3.1.0
info:
  title: SHUGO KAGI - Session Refresh API
  description: |
    Endpoint de renovación de sesión con Refresh Token Rotation y Reuse Detection.
    
    ## Características Clave
    - Rotación single-use: cada Refresh Token se usa exactamente una vez.
    - Token Family Chain: los sucesores heredan family_root_id (claim sid).
    - Grace Window (60s): concurrencia legítima multi-tab resuelta idempotentemente.
    - Reuse Detection: reuso fuera de ventana revoca la familia completa (RFC 6819 §5.2.2.3).
    - Vida absoluta (30d): inextensible por rotación; fuerza re-autenticación interactiva.
    
    ## Seguridad
    - Todos los rechazos de negocio retornan 400 invalid_grant (Zero Info Leakage).
    - El Access Token sucesor preserva auth_level y amr: la renovación nunca eleva privilegios.
  version: 1.0.0

servers:
  - url: https://auth.shugo.io/v1
    description: Producción

paths:
  /auth/refresh:
    post:
      operationId: refreshSession
      summary: Renovar sesión (rotación de Refresh Token)
      description: |
        Intercambia un Refresh Token vigente por un par nuevo (Access Token + sucesor).
        El token presentado queda ROTATED y no puede usarse como credencial primaria.
      tags:
        - Session Management
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RefreshRequest'
      responses:
        '200':
          description: Renovación exitosa o step-up de MFA requerido
          headers:
            Cache-Control:
              schema: { type: string }
              description: "no-store (los tokens nunca deben cachearse)"
            Pragma:
              schema: { type: string }
              description: "no-cache"
          content:
            application/json:
              schema:
                oneOf:
                  - $ref: '#/components/schemas/RefreshResponse'
                  - $ref: '#/components/schemas/MfaRequiredResponse'
        '400':
          description: |
            Rechazo genérico. Cubre: token inexistente, revocado, expirado (sliding o
            absoluto), grace indisponible, reuso detectado o revocación por riesgo.
            El motivo interno NUNCA se expone (Zero Info Leakage).
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '429':
          description: Rate limit por identidad o IP excedido
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '503':
          description: |
            Fail-Secure: Token Store (KAGAMI DB) no disponible.
            No se emiten tokens sin validar y rotar contra la fuente de verdad.
          headers:
            Retry-After:
              schema: { type: integer }
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

components:
  schemas:
    RefreshRequest:
      type: object
      required: [refresh_token]
      properties:
        refresh_token:
          type: string
          description: "Refresh Token opaco (256 bits, Base64url). Nunca JWT."
          example: "o39xKd82mQwZr7LpVbNcXtYhFjA5sEuG..."

    RefreshResponse:
      type: object
      properties:
        token_type:
          type: string
          enum: [Bearer]
        access_token:
          type: string
          description: "JWT de 15 min con sid = family_root_id y jti nuevo."
        expires_in:
          type: integer
          example: 900
        refresh_token:
          type: string
          description: "Sucesor opaco. En Grace Window, es el MISMO sucesor previo."

    MfaRequiredResponse:
      type: object
      properties:
        mfa_required:
          type: boolean
          example: true
        mfa_token:
          type: string
          description: "Token de step-up (UC-15). La cadena NO se rota aún."
        allowed_methods:
          type: array
          items: { type: string, enum: [totp, webauthn, push] }

    ErrorResponse:
      type: object
      properties:
        error:
          type: object
          properties:
            code:
              type: string
              enum: [invalid_request, invalid_grant, rate_limit_exceeded, service_unavailable]
            message:
              type: string
              example: "The provided refresh token is not valid."
```

### 2.2. Matriz de Comportamiento (Colapso Zero Info Leakage)

| Outcome Interno | Estado del Token | HTTP | Cuerpo | Métrica (`result` label) |
| :--- | :--- | :--- | :--- | :--- |
| `Rotated` | ACTIVE → ROTADO + sucesor | **200** | Tokens nuevos | `rotated` |
| `GraceResolved` | ROTADO en ventana | **200** | Mismo sucesor + JWT nuevo | `grace_resolved` |
| `MfaStepUp` | ACTIVE (sin rotar) | **200** | `{ mfa_required }` | `mfa_stepup` |
| `Reject(NotFound)` | — | **400** | `invalid_grant` | `rejected_not_found` |
| `Reject(Revoked)` | REVOKED | **400** | `invalid_grant` | `rejected_revoked` |
| `Reject(SlidingExpired)` | EXPIRED (sliding) | **400** | `invalid_grant` | `rejected_sliding` |
| `Reject(AbsoluteExpired)` | Familia ≥ 30d | **400** | `invalid_grant` | `rejected_absolute` |
| `Reject(GraceUnavailable)` | ROTADO + Redis caído | **400** | `invalid_grant` | `rejected_grace_unavailable` |
| `Reject(ReuseDetected)` | Familia REVOKED | **400** | `invalid_grant` | `rejected_reuse` |
| `Reject(RiskRevoked)` | Familia REVOKED | **400** | `invalid_grant` | `rejected_risk` |
| `InfrastructureException` | DB caída | **503** | `service_unavailable` | `db_unavailable` |

---

## 3. Catálogo de Métricas OpenTelemetry

### 3.1. Métricas de Negocio y Cadena

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagi.auth.refresh.total` | Counter | operations | Total de renovaciones procesadas | `tenant_id`, `result` (ver matriz) |
| `kagi.auth.refresh.reuse_detections_total` | Counter | events | **Detecciones de reuso** (compromiso confirmado) | `tenant_id` |
| `kagi.auth.refresh.family_revocations_total` | Counter | families | Familias revocadas (reuse o risk) | `tenant_id`, `reason` |
| `kagi.auth.refresh.generation` | Histogram | generations | Profundidad de cadena (`generation`) en cada rotación | `tenant_id` |
| `kagi.auth.refresh.absolute_expirations_total` | Counter | events | Familias que alcanzaron vida absoluta (re-login forzado) | `tenant_id` |

### 3.2. Métricas de Rendimiento y Contención

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagi.auth.refresh.duration` | Histogram | ms | Latencia end-to-end del endpoint | `tenant_id`, `result` |
| `kagi.auth.refresh.tx_duration` | Histogram | ms | **Duración del lock de fila** (BEGIN→COMMIT) | `tenant_id`, `path` (rotate, grace, revoke) |
| `kagi.auth.refresh.grace_resolve_duration` | Histogram | ms | Lectura + desellado AEAD de Grace Cache | `tenant_id` |
| `kagi.auth.refresh.blacklist_pipeline_duration` | Histogram | ms | Pipeline de blacklisteado de JTIs en contención | `tenant_id` |
| `kagi.auth.refresh.jwt_sign_duration` | Histogram | ms | Firma del Access Token sucesor | `algorithm` |

### 3.3. Métricas de Resiliencia

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagi.auth.refresh.grace_seal_failures_total` | Counter | failures | Seals de Grace Cache fallidos post-commit (warning) | `tenant_id` |
| `kagi.auth.refresh.grace_unavailable_total` | Counter | rejections | Rechazos conservadores por Grace Cache ilegible | `tenant_id` |
| `kagi.auth.refresh.blacklist_failures_total` | Counter | failures | Blacklists fallidos durante contención (ventana residual abierta) | `tenant_id` |
| `kagi.auth.refresh.db_failures_total` | Counter | failures | Fail-Secure 503 emitidos (Token Store caído) | `tenant_id` |
| `kagi.auth.refresh.risk_evaluator_timeouts_total` | Counter | timeouts | Timeouts del Risk Engine (fail-open/step-up según política) | `tenant_id`, `fail_mode` |

### 3.4. Distributed Tracing (Spans del Refresh)

```text
POST /v1/auth/refresh (Root Span) [Total: 42ms]
├── crypto.prepare [1ms]              (SHA256 + Generate256 + UUID jti)
├── db.token.select_for_update [9ms]  (lock adquirido)
├── domain.rotate [0ms]               (RotateTo: atomicidad de par)
├── db.rotation_tx [13ms]             (INSERT sucesor + UPDATE presentado + outbox + COMMIT)
├── redis.grace.seal [5ms]            (POST-COMMIT, AES-256-GCM, TTL 60s)
└── jwt.sign [3ms]                    (EdDSA, sid=family, auth_level preservado)

Camino B (Grace):
├── redis.grace.get [2ms] + aead.unseal [1ms]
├── db.rebind_jti_tx [8ms]
└── jwt.sign [3ms]

Camino C (Contención):
├── db.family_revoke_tx [16ms]        (UPDATE set-based + outbox ×2 + COMMIT)
├── redis.blacklist.pipeline [9ms]    (POST-COMMIT, TTL residual)
└── notify.dispatch [async]           (fuera del trace crítico)
```

### 3.5. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Reuse Detection Spike** | `rate(reuse_detections_total[10m]) > 3x baseline` | **Crítica** | Campaña de robo de tokens probable. Correlacionar con SIEM, verificar notificaciones a titulares, evaluar revocación tenant-wide (UC-21). |
| **Lock Contention** | `refresh.tx_duration_p99 > 25ms` por 5 min | Alta | Contención en `refresh_tokens`. Revisar índices, vacuum y picos de concurrencia multi-tab. |
| **Grace Unavailable Spike** | `rate(grace_unavailable_total[5m]) > 10` | Alta | Redis Grace Cache degradado. Usuarios concurrentes reciben rechazos conservadores. Verificar breaker y salud de Redis. |
| **Fail-Secure Storm** | `rate(db_failures_total[2m]) > 0` | **Crítica** | Token Store caído: los refreshes retornan 503 y los usuarios caerán al expirar su JWT (15 min). Escalar a DBA inmediatamente. |
| **Blacklist Residual Window** | `rate(blacklist_failures_total[5m]) > 0` | **Crítica** | Contención efectiva en DB pero Access Tokens vivos ≤15 min sin blacklisear. Ejecutar reconciliación desde `SessionFamilyRevoked`. |
| **Generation Depth Anomaly** | `refresh.generation_p99 > 500` | Media | Sesiones anormalmente largas o cliente que renueva en exceso. Verificar que la vida absoluta se cumple y revisar comportamiento del cliente. |

---

## 4. Patrones de Resiliencia

### 4.1. Fail-Secure en Token Store (Sin Retries Automáticos)

**Decisión:** No se reintentan transacciones `FOR UPDATE` fallidas automáticamente.

**Justificación:** Un retry sobre una transacción que sostiene locks de fila puede amplificar la contención exactamente cuando la DB está degradada (tormenta de retries → locks en cola → caída total). El cliente recibe `503 + Retry-After` y aplica backoff con jitter, que es el lugar correcto para reintentar.

```csharp
// RefreshHandler: única política de reintento = la del cliente
catch (InfrastructureException ex)
{
    _logger.LogCritical(ex, "Token store unavailable during refresh.");
    return new ObjectResult(ErrorResponse.ServiceUnavailable)
    {
        StatusCode = 503,
        Headers = { RetryAfter = 5 }   // backoff del cliente, no del servidor
    };
}
```

### 4.2. Circuit Breakers Diferenciados para Redis

| Dependencia | Política | Comportamiento al Abrir | Justificación |
| :--- | :--- | :--- | :--- |
| **Grace Cache (lectura/escritura)** | Breaker tras 3 fallos, 30s open | Lectura → `GraceResolution.Unavailable()` (rechazo conservador **sin** revocar familia); escritura → skip con warning | La indisponibilidad no es evidencia de abuso; revocar familias por outage sería un auto-DoS. |
| **Blacklist de JTIs (contención)** | Breaker tras 3 fallos, 30s open | Skip con `LogCritical` + alerta `Blacklist Residual Window` | La contención principal (revocación en DB) ya es duradera; la ventana residual ≤15 min es aceptada y alertada. |

### 4.3. Reconciliación de Blacklist desde Eventos (Cierre de Ventana Residual)

El evento `SessionFamilyRevoked` ya transporta `jtisBlacklisted` en su payload. Un job de reconciliación consume el **Outbox/event store** (no Redis) y re-intenta el blacklisteado pendiente tras la recuperación:

```csharp
public class FamilyBlacklistReconciliationJob : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await foreach (var recovered in _breakerMonitor.OnRedisRecovered(ct))
        {
            // Eventos de contención de los últimos 20 min (ventana máx de Access Tokens)
            var pending = await _eventStore.GetUnacknowledgedAsync<SessionFamilyRevoked>(
                since: DateTime.UtcNow.AddMinutes(-20), ct);

            foreach (var evt in pending)
            {
                var jtis = evt.JtisBlacklisted
                    .Select(j => new JtiWithExpiration(j, evt.OccurredAt.AddMinutes(15)))
                    .Where(j => j.Expiration > DateTime.UtcNow)   // solo JTIs aún vivos
                    .ToList();

                if (jtis.Count > 0)
                    await _jtiBlacklist.BlacklistAsync(jtis, DateTime.UtcNow, ct); // idempotente (SET)
            }
        }
    }
}
```

**Justificación:** Convierte la ventana residual del Fail-Safe en una **ventana cerrable y verificable**, reutilizando el mismo patrón de reconciliación del UC-12 pero alimentado desde el evento de dominio en lugar de una tabla forense.

### 4.4. Risk Engine con Fail-Mode Configurable

```csharp
// Timeout estricto de 100ms para el hook de riesgo
using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
cts.CancelAfter(TimeSpan.FromMilliseconds(100));

try
{
    return await _riskClient.EvaluateContextDeltaAsync(oldCtx, newCtx, cts.Token);
}
catch (OperationCanceledException)
{
    _metrics.RiskEvaluatorTimeouts.Add(1, tags);
    // fail_mode=open  → continuar rotación (score 0) con warning
    // fail_mode=stepup → degradar a step-up MFA (nunca fail-open silencioso en tenants strict)
    return policy.RiskFailMode == RiskFailMode.StepUp
        ? RiskEvaluation.ForceStepUp()
        : RiskEvaluation.PassWithWarning();
}
```

---

## 5. Feature Flags y Política por Tenant

| Feature Flag | Descripción | Default | ¿Desactivable en incidente? |
| :--- | :--- | :--- | :--- |
| `kagi.auth.refresh.enabled` | Kill switch global del endpoint | `true` | Sí (último recurso) |
| `kagi.auth.refresh.sliding_lifetime_days` | Expiración deslizante por rotación | `30` | Sí |
| `kagi.auth.refresh.absolute_lifetime_days` | Vida absoluta de la familia | `30` | Sí (solo aumentar; reducirla expulsa sesiones) |
| `kagi.auth.refresh.grace_window_seconds` | Ventana de concurrencia legítima | `60` | Sí (rango 5-300 validado por `RefreshPolicy`) |
| `kagi.auth.refresh.mfa_freshness_hours` | Step-up programado por frescura MFA | `12` | Sí (`null` desactiva) |
| `kagi.auth.refresh.risk_policy` | Decisión ante score alto: `off \| stepup \| revoke` | `stepup` | Sí |
| `kagi.auth.refresh.risk_fail_mode` | Comportamiento ante timeout del Risk Engine: `open \| stepup` | `open` | Sí |
| `kagi.auth.refresh.reuse_containment_enabled` | Contención por reuso (revocación de familia) | `true` | **NO — bloqueado por diseño** |

**Nota de Seguridad:** `reuse_containment_enabled` **no puede desactivarse** por configuración: la revocación de familia ante reuso confirmado es un control mandated por RFC 6819 §5.2.2.3. El flag existe únicamente como documentación ejecutable de que la contención es siempre-on; cualquier intento de registro con valor `false` falla en el validador de configuración al arranque.

---

## 6. Estrategia de Testing

### 6.1. Escenario 1: Ola de Renovación Matutina

**Objetivo:** 800 refresh/seg sostenidos con latencia p95 < 150ms y lock p99 < 25ms.

```javascript
import http from 'k6/http';
import { check } from 'k6';
import { Trend, Rate } from 'k6/metrics';

const refreshLatency = new Trend('refresh_latency', true);
const errorRate = new Rate('errors');

export const options = {
  scenarios: {
    morning_wave: {
      executor: 'ramping-arrival-rate',
      startRate: 100, timeUnit: '1s',
      preAllocatedVUs: 200, maxVUs: 1200,
      stages: [
        { target: 400, duration: '2m' },
        { target: 800, duration: '5m' },  // apertura de jornada
        { target: 800, duration: '3m' },
        { target: 0,   duration: '2m' },
      ],
    },
  },
  thresholds: {
    refresh_latency: ['p(95)<150', 'p(99)<250'],
    errors: ['rate<0.001'],
  },
};

export default function () {
  const session = nextSessionFromPool(); // tokens pre-rotados en setup()
  const res = http.post(`${__ENV.BASE_URL}/v1/auth/refresh`,
    JSON.stringify({ refresh_token: session.refresh }),
    { headers: { 'Content-Type': 'application/json' } });

  refreshLatency.add(res.timings.duration);
  if (!check(res, { 'refresh 200': (r) => r.status === 200 })) errorRate.add(1);
  else rotatePoolSession(session, res.json('refresh_token')); // el sucesor entra al pool
}
```

### 6.2. Escenario 2: Concurrencia Multi-Tab (Grace Window bajo Fuego)

**Objetivo:** Dos VUs renuevan **el mismo token simultáneamente**; se espera exactamente 1 rotación + 1 resolución grace, **cero** revocaciones de familia.

```javascript
export const options = {
  scenarios: {
    multi_tab: {
      executor: 'per-vu-iterations',
      vus: 2, iterations: 1,   // par de VUs por token compartido
      maxDuration: '2m',
    },
  },
  thresholds: {
    // Verificación externa vía Prometheus post-run:
    // increase(kagi_auth_refresh_family_revocations_total) == 0
    // increase(kagi_auth_refresh_total{result="grace_resolved"}) == 1
    // increase(kagi_auth_refresh_total{result="rotated"}) == 1
    http_req_duration: ['p(95)<300'],
  },
};

export default function () {
  const shared = __ENV.SHARED_REFRESH_TOKEN; // mismo valor inyectado a ambos VUs
  const res = http.post(`${__ENV.BASE_URL}/v1/auth/refresh`,
    JSON.stringify({ refresh_token: shared }),
    { headers: { 'Content-Type': 'application/json' } });
  check(res, { 'ambas pestañas obtienen 200': (r) => r.status === 200 });
}
```

### 6.3. Escenario 3: Simulación de Robo (Reuse Detection End-to-End)

**Objetivo:** Validar el ciclo completo de contención: rotación → espera >60s → replay del token viejo → familia revocada → sucesor también rechazado → notificación emitida.

```javascript
// Test de integración orquestado (no k6 puro): valida efectos colaterales
// 1. Login (UC-11) → tokens T0
// 2. Refresh con T0 → T1 (T0 = ROTATED)
// 3. sleep(61s)
// 4. Replay de T0 → expect 400 invalid_grant
// 5. Assert: reuse_detections_total +1; family_revocations_total +1
// 6. Refresh con T1 → expect 400 (familia revocada)
// 7. Assert: blacklist contiene issued_jti de T1 (GET interno de verificación)
// 8. Assert: notificación al titular despachada (mock de Notification Service)
```

### 6.4. Escenario 4: Chaos - Redis Caído durante Ola de Refresh

**Objetivo:** Con Redis muerto: (a) las rotaciones (Camino A) mantienen disponibilidad 100%; (b) los rechazos conservadores aumentan pero `family_revocations_total` permanece **plano**; (c) al recuperar, la reconciliación cierra ventanas residuales.

```javascript
export const options = {
  scenarios: {
    wave_during_redis_outage: {
      executor: 'constant-arrival-rate',
      rate: 300, timeUnit: '1s', duration: '12m',
      preAllocatedVUs: 200, maxVUs: 600,
    },
  },
  thresholds: {
    // Camino A no depende de Redis para commitear:
    refresh_rotated_availability: ['rate>0.999'],
    // Verificación externa: family_revocations_total == 0 durante el outage
  },
};
// Runbook del caos: min 0-3 baseline | min 3-8 Redis down | min 8-12 recovery + reconciliación
```

---

## 7. Deployment y Operaciones

### 7.1. Kubernetes HPA (Escalado por Contención, no solo CPU)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kagi-refresh-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kagi-refresh
  minReplicas: 6
  maxReplicas: 40
  metrics:
    - type: Pods
      pods:
        metric: { name: kagi_auth_refresh_tx_duration_p99 }
        target: { type: AverageValue, averageValue: "25" }   # contención de lock
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 65 }
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies: [{ type: Percent, value: 100, periodSeconds: 30 }]
```

### 7.2. Runbook de Operaciones

**Escenario 1: Reuse Detection Spike (Campaña de Robo)**
1. Verificar alerta `Reuse Detection Spike` y correlacionar IPs/UA en `Session.ReuseDetected` (KAGE).
2. Confirmar que las notificaciones a titulares fueron despachadas (cola de Notification Service).
3. Si el patrón es multi-tenant o multi-identidad: escalar a incidente de plataforma; evaluar revocación tenant-wide (UC-21) y rotación de claves de firma JWT.
4. Post-incidente: revisar si los tokens robados provenían de un cliente específico (fuga de almacenamiento local) y emitir guidance de hardening.

**Escenario 2: Grace Unavailable Spike (Redis Degradado)**
1. Verificar `grace_unavailable_total` y estado del breaker de Grace Cache.
2. Confirmar que `family_revocations_total` permanece plano (los rechazos conservadores **no** deben revocar familias).
3. Comunicar impacto esperado: usuarios multi-tab recibirán 400 y reintentarán; sin pérdida de sesión para la pestaña con el sucesor legítimo.
4. Tras recuperación, verificar que `grace_resolved` vuelve a baseline.

**Escenario 3: Fail-Secure Storm (Token Store Caído)**
1. Verificar `db_failures_total` y salud de PostgreSQL/PgBouncer.
2. Comunicar al SOC: **ningún token se emite** durante la ventana (comportamiento correcto); los usuarios caerán a login al expirar su JWT (≤15 min).
3. Tras recuperación, monitorear pico de logins (UC-11) y de `absolute_expirations` si la ventana fue larga.

---

## 8. Compliance y Auditoría

### 8.1. RFC 6819 §5.2.2.3 (OAuth 2.0 Threat Model - Refresh Token Reuse)
*   **Requisito:** El servidor debe revocar el token y, ante reuso, revocar el contexto completo.
*   **Implementación UC-13:** Detección de reuso fuera de Grace Window → revocación de familia + blacklisteado de JTIs + notificación. Control siempre-on (flag no desactivable).

### 8.2. NIST SP 800-63B
*   **Requisito:** Los refresh tokens deben estar vinculados al contexto y exigir re-autenticación periódica.
*   **Implementación UC-13:** `SessionContext` binding evaluado en cada renovación (hook UC-17) + vida absoluta inextensible que fuerza re-login interactivo.

### 8.3. OWASP ASVS 4.0 - V3 (Session Management)
*   **Requisitos:** Binding de sesión, ciclo de vida con timeouts, invalidación efectiva.
*   **Implementación UC-13:** Rotación single-use (V3.3), binding por device/UA/IP (V3.2), invalidación en cascada por familia.

### 8.4. SOC 2 CC6.1 / CC7.2 e ISO 27001 A.5.18 / A.8.15
*   **Requisito:** Retiro de derechos de acceso y monitoreo de eventos de seguridad.
*   **Implementación UC-13:** `Session.FamilyRevoked` y `Session.ReuseDetected` auditados inmutablemente vía Outbox; alertas de spike conectadas al SIEM.

### 8.5. GDPR Art. 32
*   **Requisito:** Seguridad del tratamiento con resiliencia de sistemas.
*   **Implementación UC-13:** Degradaciones diferenciadas y documentadas (Fail-Secure, rechazo conservador, Fail-Safe con reconciliación) que mantienen la seguridad base bajo fallo de cualquier dependencia.

---

## 9. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | La matriz de colapso implementa RN-25 (Zero Info Leakage); las métricas de reuso y familia monitorean INV-83/INV-84. |
| `08_Final_Class_Design.md` | `RefreshHandler.Match` y los puertos (`IGraceWindowCache`, `IJtiBlacklist`, `IRiskEvaluator`) emiten exactamente las métricas y respetan los breakers definidos aquí. |
| `10_UC_Specific_Data_Model.md` | Los índices de familia y `issued_jti` soportan las consultas de reconciliación y las métricas de generation depth. |
| `11_UC_Specific_Security_Threats.md` | Las alertas de reuse spike y blacklist residual operationalizan las mitigaciones de Token Theft y Zombie Sessions. |
| `12_UC_Specific_Performance_Benchmarks.md` | Los SLOs de latencia y lock duration se validan con los escenarios k6 y chaos definidos aquí. |

---

## 10. Cierre del Paquete de Diseño Funcional (UC-13)

Con este documento, completamos los **9 archivos del Paquete de Diseño Profundo Funcional (Tier 1)** para el UC-13:

1. ✅ `01_Requirements_Specification.md` - Rotación, Grace Window, Reuse Detection y vida absoluta con BDD completo.
2. ✅ `02_Use_Case_Diagram.md` - Tres caminos mutuamente excluyentes con validaciones `<<include>>` y step-ups `<<extend>>`.
3. ✅ `03_Activity_Diagram.md` - Flujo con guardas fail-fast, límites transaccionales y caminos degradados.
4. ✅ `04_Initial_Domain_Model.md` - `RotateTo` con atomicidad de par, `GraceWindowEntry` AEAD y `FamilyRevoker` set-based.
5. ✅ `05_State_Machine_Diagram.md` - Máquina del token (persistente) y máquina derivada de la familia (conceptual).
6. ✅ `06_Robustness_Diagram.md` - Colaboración ICONIX con lock de fila exclusivo y commit-before-cache.
7. ✅ `07_Sequence_Diagram.md` - Orden temporal exacto con JWT post-commit y serialización multi-tab.
8. ✅ `08_Final_Class_Design.md` - Typed Result con Match exhaustivo y Zero Info Leakage compilable.
9. ✅ `09_Modern_Artifacts.md` - OpenAPI, métricas, resiliencia, flags, testing y compliance.

### 10.1. Resumen de lo Logrado

Hemos diseñado la renovación de sesiones como un **sistema de cadena auto-curativa**:
- **Rotación single-use con atomicidad de par:** imposible rotar sin sucesor o crear sucesor sin rotar.
- **Concurrencia legítima absorbida:** Grace Window cifrada de 60s que no bifurca cadenas ni amplía superficie de ataque.
- **Robo convertido en alarma:** reuso fuera de ventana = contención automática de familia + blacklisteado quirúrgico + notificación al titular + evento SIEM.
- **Zombie Sessions eliminadas:** vida absoluta inextensible que fuerza re-autenticación interactiva.
- **Degradaciones sin sorpresas:** cuatro fallos de infraestructura con comportamientos distintos, alertados y reconciliables.
- **Zero Info Leakage de extremo a extremo:** del contrato OpenAPI al `Match` exhaustivo en compile-time.

### 10.2. Próximos Pasos (Documentos Técnicos Específicos)

Para el UC-13 aún faltan los 3 documentos de la Fase 2:

1. **`10_UC_Specific_Data_Model.md`** - Migración de `refresh_tokens` con columnas de familia (`family_root_id`, `generation`, `absolute_expires_at`, `issued_jti`), índices de cadena y keyspace Redis (`rotgrace:*`, `blacklist:*`).
2. **`11_UC_Specific_Security_Threats.md`** - Threat modeling de Token Theft paralelo, Grace Window abuse, cadena bifurcada, Zombie Sessions y supresión de contención.
3. **`12_UC_Specific_Performance_Benchmarks.md`** - SLAs de lock duration, throughput de rotación, capacidad de Grace Cache y benchmarks de reconciliación.

---
*Fin del Documento 09_Modern_Artifacts para UC-13*