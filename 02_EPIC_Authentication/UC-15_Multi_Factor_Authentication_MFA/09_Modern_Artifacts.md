# UC-15: Multi-Factor Authentication (MFA) - Artefactos Modernos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-15 - Multi-Factor Authentication (MFA - TOTP, WebAuthn/FIDO2, Push, Recovery) |
| **Documento:** | 09_Modern_Artifacts.md |
| **Metodología:** | API-First + Observability (OpenTelemetry) + Resilience Patterns + Virtual Authenticator Testing + DevSecOps |
| **Estándares:** | NIST SP 800-63B (AAL2/AAL3, Restricted Authenticators, Verifier Impersonation Resistance), RFC 6238 (TOTP), W3C WebAuthn Level 2 / FIDO2, OWASP ASVS 4.0 (V2) + OWASP MFA Cheat Sheet, PSD2 SCA RTS (independencia de canales), GDPR Art. 32, SOC 2 CC6.1/CC7.2, ISO 27001:2022 A.5.18/A.8.15 |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 23/10/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 23/10/2026 | Gastón Ruiz | **Creación inicial.** Definición de los artefactos modernos para UC-15: contratos OpenAPI 3.1 de los nueve endpoints MFA (challenge, verify, stepup, push status, enrollments, activate, recovery regenerate, y el inbound push approve/deny) con matriz de colapso Zero Info Leakage de trece `FailReason`, catálogo de métricas OpenTelemetry (detecciones de seguridad como señales de primer nivel: clonación, replay, fatiga, reuso de token), patrones de resiliencia con cero reintentos salvo despacho push, Feature Flags de política por tenant con flags bloqueados por diseño, estrategia de testing con authenticators virtuales (manipulación de counter para clones, simulador push con denegaciones), chaos de Redis/KAGAMI/FCM, y runbooks por tipo de incidente. | Cierra el paquete de diseño Tier 1 funcional del UC-15, proporcionando todos los artefactos necesarios para operar el segundo factor como un sistema de evidencia: los ataques a los métodos (clonación, replay, fatiga, reciclaje de token) son métricas alertables, las degradaciones son asimétricas y documentadas, y el contrato HTTP jamás revela el motivo interno de un fallo. |

---

## 1. Introducción y Propósito

Este documento define los **Artefactos Modernos** necesarios para operar el UC-15 (Multi-Factor Authentication) en producción. A diferencia de los UCs de autenticación primaria, el MFA opera sobre una superficie donde **el adversario ataca a los métodos mismos** (clonar authenticators, replays de códigos, fatigar aprobaciones, reciclar tokens de pausa), por lo que sus artefactos deben satisfacer cuatro demandas simultáneas:

1. **Contratos opacos con riqueza interna:** Nueve endpoints con semánticas distintas que colapsan trece motivos de fallo a un único `mfa_failed`, mientras las métricas conservan el motivo exacto para detección de ataques.
2. **Observabilidad de las detecciones:** Clonación, replay TOTP, reuso de mfa_token, fatiga push y uso de recovery no son errores: son **señales de seguridad de primer nivel** con alertas propias.
3. **Resiliencia asimétrica por criticidad:** Redis/KAGAMI caídos → fail-secure sin invalidar pausas; counter WebAuthn no persistido → completitud rechazada; canal push caído → métodos alternativos; trusted device caído → completitud preservada.
4. **Verificación con adversario activo:** Los tests no solo ejercitan happy paths: manipulan counters para forzar detecciones de clonación, agotan umbrales de throttling y cortan dependencias en puntos quirúrgicos.

---

## 2. Contratos OpenAPI Completos

### 2.1. Especificación OpenAPI 3.1

```yaml
openapi: 3.1.0
info:
  title: SHUGO KAGI - Multi-Factor Authentication API
  description: |
    Endpoints de segundo factor del motor KAGI.
    
    ## Características Clave
    - mfa_token single-use (TTL ≤5 min): challenge valida sin consumir; verify consume.
    - Cinco métodos con clasificación de assurance (webauthn PR; totp/push; recovery break-glass; email_otp restringido).
    - Completitud context-aware: reanuda pausas de UC-11/13/14 o emite elevación de step-up.
    - Push con number matching: el dispositivo escribe el estado (endpoint inbound); el cliente solo pollea.
    - Enrollment con activación por prueba y recovery codes generados atómicamente en la primera activación.
    
    ## Seguridad
    - Los 13 motivos internos de fallo convergen a `mfa_failed` genérico (Zero Info Leakage).
    - Recovery codes se muestran UNA vez con Cache-Control: no-store; nunca se loguean.
    - Bajo require_phishing_resistant: recovery y email_otp no son verificables.
  version: 1.0.0

servers:
  - url: https://auth.shugo.io/v1
    description: Producción

paths:
  /auth/mfa/challenge:
    post:
      operationId: mfaChallenge
      summary: Emitir desafío de segundo factor (valida el contexto sin consumirlo)
      tags: [MFA]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/ChallengeRequest' }
      responses:
        '200':
          description: Desafío emitido u oferta de enrollment (neutra)
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ChallengeResponse' }
        '400': { $ref: '#/components/responses/MfaFailed' }
        '503': { $ref: '#/components/responses/MfaUnavailable' }

  /auth/mfa/verify:
    post:
      operationId: mfaVerify
      summary: Verificar prueba y completar el flujo pausado (consume el mfa_token)
      tags: [MFA]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/VerifyRequest' }
      responses:
        '200':
          description: Completitud (tokens finales o elevación) con evidencia en amr/acr/auth_time
          headers:
            Cache-Control: { schema: { type: string }, description: "no-store" }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/VerifyResponse' }
        '400': { $ref: '#/components/responses/MfaFailed' }
        '503': { $ref: '#/components/responses/MfaUnavailable' }

  /auth/mfa/stepup:
    post:
      operationId: mfaStepUp
      summary: Step-up en sesión viva (sid); reuso de frescura solo sin riesgo forzado
      tags: [MFA]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/StepUpRequest' }
      responses:
        '200': { description: "Elevación emitida o challenge de step-up" }
        '400': { $ref: '#/components/responses/MfaFailed' }

  /auth/mfa/push/status/{challengeId}:
    get:
      operationId: mfaPushStatus
      summary: Observación del challenge push (el cliente nunca escribe estado)
      tags: [MFA]
      parameters:
        - { name: challengeId, in: path, required: true, schema: { type: string } }
      responses:
        '200': { description: "ISSUED | APPROVED | DENIED | EXPIRED" }

  /auth/mfa/enrollments:
    post:
      operationId: mfaBeginEnrollment
      summary: Iniciar enrollment (PENDING, TTL 10 min)
      tags: [MFA Enrollment]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/EnrollmentRequest' }
      responses:
        '200': { description: "Payload de enrollment pendiente (otpauth URI / creation options / confirmación push)" }
        '403': { description: "Método no permitido por política (motivo neutro)" }
    get:
      operationId: mfaListEnrollments
      summary: Listar enrollments del usuario autenticado
      tags: [MFA Enrollment]
      responses:
        '200': { description: "Enrollments por método y estado" }
  /auth/mfa/enrollments/{id}/activate:
    post:
      operationId: mfaActivateEnrollment
      summary: Activar enrollment por prueba (3 fallos cancelan)
      tags: [MFA Enrollment]
      parameters:
        - { name: id, in: path, required: true, schema: { type: string } }
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/ActivateRequest' }
      responses:
        '200':
          description: "Enrollment ACTIVE + recovery codes (primera activación, visualización única)"
          headers:
            Cache-Control: { schema: { type: string }, description: "no-store" }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ActivateResponse' }
        '400': { $ref: '#/components/responses/MfaFailed' }
  /auth/mfa/enrollments/{id}:
    delete:
      operationId: mfaRevokeEnrollment
      summary: Revocar enrollment (el propio usuario; step-up si política lo exige)
      tags: [MFA Enrollment]
      parameters:
        - { name: id, in: path, required: true, schema: { type: string } }
      responses:
        '204': { description: "Revocado" }

  /auth/mfa/recovery-codes/regenerate:
    post:
      operationId: mfaRegenerateRecoveryCodes
      summary: Regenerar lote break-glass (requiere step-up; supersede el lote anterior)
      tags: [MFA Enrollment]
      responses:
        '200': { description: "Nuevo lote de 10 códigos (visualización única, no-store)" }

  /auth/mfa/push/approve:
    post:
      operationId: mfaPushApprove
      summary: "Inbound del dispositivo: aprobación con number matching + attestation"
      tags: [MFA Push Inbound]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/PushApproveRequest' }
      responses:
        '200': { description: "Ack (CAS resuelto; AlreadyResolved si la carrera la ganó una denegación)" }
  /auth/mfa/push/deny:
    post:
      operationId: mfaPushDeny
      summary: "Inbound del dispositivo: denegación explícita (3 seguidas → cooldown)"
      tags: [MFA Push Inbound]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/PushDenyRequest' }
      responses:
        '200': { description: "Ack" }

components:
  responses:
    MfaFailed:
      description: "mfa_failed genérico (13 motivos internos colapsados; motivo real solo en auditoría)"
      content:
        application/json:
          schema:
            type: object
            properties:
              error:
                type: object
                properties:
                  code: { type: string, enum: [mfa_failed] }
                  message: { type: string, example: "Multi-factor verification could not be completed." }
    MfaUnavailable:
      description: "Fail-secure (Redis/KAGAMI caído); la pausa NO se invalida"
      headers:
        Retry-After: { schema: { type: integer } }
      content:
        application/json:
          schema:
            type: object
            properties:
              error:
                type: object
                properties:
                  code: { type: string, enum: [mfa_unavailable] }
                  message: { type: string }

  schemas:
    ChallengeRequest:
      type: object
      properties:
        mfa_token: { type: string, description: "Pausa de UC-11/13/14" }
        sid: { type: string, description: "Alternativa para step-up en sesión viva" }
        method: { type: string, enum: [totp, webauthn, push, email_otp], description: "Opcional: preferencia del usuario" }
    ChallengeResponse:
      type: object
      properties:
        challenge_id: { type: string }
        method: { type: string }
        expires_in: { type: integer, example: 60 }
        public_key: { type: object, description: "Solo webauthn: PublicKeyCredentialRequestOptions" }
        enrollment_required: { type: boolean, description: "Oferta neutra de enrollment si no hay método compatible" }
    VerifyRequest:
      type: object
      required: [mfa_token, method, proof]
      properties:
        mfa_token: { type: string }
        method: { type: string, enum: [totp, webauthn, push, recovery_code, email_otp] }
        challenge_id: { type: string, description: "Requerido para webauthn y push" }
        proof:
          oneOf:
            - { type: object, description: "totp/email_otp: { code }" }
            - { type: object, description: "webauthn: { assertion }" }
            - { type: object, description: "push: { challenge_id } (el estado lo escribe el dispositivo)" }
            - { type: object, description: "recovery_code: { code }" }
        trusted_opt_in: { type: boolean, default: false }
    VerifyResponse:
      type: object
      properties:
        token_type: { type: string, enum: [Bearer] }
        access_token: { type: string }
        refresh_token: { type: string, description: "Solo completitud de pausa de login" }
        expires_in: { type: integer, example: 900 }
        session_meta:
          type: object
          properties:
            auth_level: { type: string, enum: [AAL2, AAL3] }
            amr: { type: array, items: { type: string, enum: [pwd, otp, hwk, push, otk] } }
            auth_time: { type: integer }
            step_up_at: { type: integer, description: "Solo elevaciones" }
            trusted_device: { type: boolean }
    EnrollmentRequest:
      type: object
      required: [method]
      properties:
        method: { type: string, enum: [totp, webauthn, push] }
        device_name: { type: string }
    ActivateRequest:
      type: object
      required: [proof]
      properties:
        proof: { type: object, description: "Código inicial / ceremony / ack push según método" }
    ActivateResponse:
      type: object
      properties:
        enrollment_id: { type: string }
        status: { type: string, enum: [active, retry_remaining, cancelled] }
        retry_remaining: { type: integer }
        recovery_codes: { type: array, items: { type: string }, description: "SOLO en la primera activación; visualización única" }
    PushApproveRequest:
      type: object
      required: [challenge_id, selected_number, device_attestation]
      properties:
        challenge_id: { type: string }
        selected_number: { type: integer, minimum: 0, maximum: 99 }
        device_attestation: { type: string }
    PushDenyRequest:
      type: object
      required: [challenge_id]
      properties:
        challenge_id: { type: string }
    StepUpRequest:
      type: object
      required: [requested_scope]
      properties:
        requested_scope: { type: string, example: "payroll:write" }
        preferred_method: { type: string, enum: [totp, webauthn, push] }
```

### 2.2. Matriz de Comportamiento (Colapso Zero Info Leakage)

| `MfaOutcome` / `FailReason` interno | HTTP | Cuerpo | Métrica (`result` label) |
| :--- | :--- | :--- | :--- |
| `ChallengeIssued` | **200** | ChallengePayload por método | `challenge_total{method,context}` |
| `EnrollmentRequired` | **200** | Oferta neutra de enrollment | `challenge_total{result=enrollment_required}` |
| `Verified` (completitud de pausa) | **200** | Tokens + `session_meta` con evidencia | `verify_total{method,context,result=succeeded}` |
| `Verified` (elevación step-up) | **200** | Token elevado con `step_up_at` | `stepup_completed_total` / `stepup_reused_total` |
| `EnrollmentActivated` | **200** | ACTIVE + recovery codes (una vez, no-store) | `enrollment_activated_total{method}` |
| `PushStatusReturned` | **200** | ISSUED/APPROVED/DENIED/EXPIRED | `push_status_queries_total` |
| `ServiceUnavailable` | **503** | `mfa_unavailable` + Retry-After | `mfa_unavailable_total{dependency}` |
| `Failed(InvalidContext)` | **400** | `mfa_failed` | `failed_total{reason=invalid_context}` |
| `Failed(TokenReuse)` | **400** | `mfa_failed` | `token_reuse_total` |
| `Failed(Throttled)` | **400** | `mfa_failed` | `throttled_total{dimension}` |
| `Failed(NoEnrollment)` | **400** | `mfa_failed` | `failed_total{reason=no_enrollment}` |
| `Failed(InvalidCode)` | **400** | `mfa_failed` | `failed_total{reason=invalid_code}` |
| `Failed(TotpReplay)` | **400** | `mfa_failed` | `totp_replay_total` |
| `Failed(WebAuthnInvalid)` | **400** | `mfa_failed` | `failed_total{reason=webauthn_invalid}` |
| `Failed(Clone)` | **400** | `mfa_failed` | `clone_detected_total` |
| `Failed(PushNotApproved / PushNumberMismatch)` | **400** | `mfa_failed` | `failed_total{reason=push_*}` |
| `Failed(CodeConsumed / BatchSuperseded)` | **400** | `mfa_failed` | `recovery_failed_total{reason}` |
| `Failed(EnrollmentExhausted)` | **400** | `mfa_failed` | `enrollment_cancelled_total{reason=exhausted}` |

---

## 3. Catálogo de Métricas OpenTelemetry

### 3.1. Métricas de Detección de Seguridad (Primer Nivel)

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagi.auth.mfa.clone_detected_total` | Counter | eventos | **Clonación WebAuthn confirmada** (counter regression con firma válida) | `tenant_id` |
| `kagi.auth.mfa.totp_replay_total` | Counter | eventos | Replay de step TOTP ya aceptado (no consume intento) | `tenant_id` |
| `kagi.auth.mfa.token_reuse_total` | Counter | eventos | Segundo consumo de mfa_token (pausa invalidada) | `tenant_id`, `origin_uc` |
| `kagi.auth.mfa.push_fatigue_total` | Counter | eventos | Cooldowns de fatiga push | `tenant_id`, `trigger` (rate, denials) |
| `kagi.auth.mfa.recovery_used_total` | Counter | usos | Consumos de break-glass (cada uso = revisión forzada) | `tenant_id` |
| `kagi.auth.mfa.throttled_total` | Counter | bloqueos | Umbrales de intento alcanzados | `tenant_id`, `dimension` (token, user_hour) |

### 3.2. Métricas de Negocio y Adopción

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagi.auth.mfa.challenge_total` | Counter | desafíos | Desafíos emitidos por método y contexto | `tenant_id`, `method`, `context` |
| `kagi.auth.mfa.verify_total` | Counter | verificaciones | Verificaciones por resultado | `tenant_id`, `method`, `context`, `result` |
| `kagi.auth.mfa.enrollment_activated_total` | Counter | activaciones | Enrollments activados por método | `tenant_id`, `method` |
| `kagi.auth.mfa.enrollment_cancelled_total` | Counter | cancelaciones | Pendings cancelados | `tenant_id`, `reason` (exhausted, expired) |
| `kagi.auth.mfa.stepup_completed_total` | Counter | elevaciones | Step-ups completados con evidencia nueva | `tenant_id`, `origin` (scope, risk) |
| `kagi.auth.mfa.stepup_reused_total` | Counter | reutilizaciones | Reuso de elevación dentro de frescura (nunca riesgo forzado) | `tenant_id` |
| `kagi.auth.mfa.trusted_device_issued_total` | Counter | emisiones | Trusted devices emitidos (solo login pause) | `tenant_id` |
| `kagi.auth.mfa.active_enrollments` | Gauge | enrollments | Distribución de métodos activos por tenant | `tenant_id`, `method` |

### 3.3. Métricas de Rendimiento y Disponibilidad

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagi.auth.mfa.verify_duration` | Histogram | ms | Latencia local del verify por método | `method` |
| `kagi.auth.mfa.completion_duration` | Histogram | ms | Completitud del resolutor de pausa | `pause_kind` |
| `kagi.auth.mfa.push_approval_duration` | Histogram | s | Tiempo de decisión del usuario en push (informativa, mediada por humano) | `tenant_id` |
| `kagi.auth.mfa.push_poll_duration` | Histogram | ms | Latencia del endpoint de status | — |
| `kagi.auth.mfa.push_dispatch_failures_total` | Counter | fallos | Despachos FCM/APNs fallidos (fallback a otros métodos) | `tenant_id` |
| `kagi.auth.mfa.unavailable_total` | Counter | 503s | Fail-secure por dependencia caída | `dependency` (redis, kagami) |

### 3.4. Distributed Tracing (Spans por Método)

```text
POST /v1/auth/mfa/verify — TOTP (Root) [9ms]
├── redis.token.consume [2ms]          (GETDEL mfa_token)
├── throttle.acquire [2ms]
├── kagami.enrollment.fetch [3ms]      (secreto AES-GCM → memoria)
├── totp.compute_window [0.3ms]        (3 HMAC-SHA1, ventana ±1)
├── redis.replay.mark [1ms]            (SET NX por step, post-verificación)
└── completer.resolve [1ms]            → login_pause resolver (trace propio de UC-11)

POST /v1/auth/mfa/verify — WebAuthn (Root) [12ms]
├── redis.token.consume [2ms]
├── throttle.acquire [2ms]
├── redis.challenge.getdel [2ms]
├── webauthn.verify_signature [1ms]
├── kagami.counter.update [4ms]        ← fail-secure: bloquea completitud si falla
└── completer.resolve [1ms]

POST /auth/mfa/push/approve (Root) [14ms]
├── kagami.attestation.validate [6ms]
├── redis.challenge.read [2ms]
├── challenge.try_approve [0.5ms]      (número + attestation)
└── redis.cas.approved [2ms]           (Lua CAS; AlreadyResolved si perdió la carrera)
```

### 3.5. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Clone Detected** | `clone_detected_total` ≥ 1 | **Crítica** | Runbook 1: authenticator clonado. Verificar con el titular, revisar IPs, evaluar revocación de sesiones (UC-21) y re-enrollment. |
| **TOTP Replay Spike** | `rate(totp_replay_total[10m]) > 5` | Alta | Posible sesión compartida o código interceptado. Correlacionar device fingerprints e IPs del tenant. |
| **Token Reuse Spike** | `rate(token_reuse_total[10m]) > 3x baseline` | Alta | Posible interceptación de mfa_token: revisar emisión en UC-11/13/14, TTLs vigentes, y clientes comprometidos. |
| **Push Fatigue Campaign** | `rate(push_fatigue_total[10m]) > 3x baseline` | Alta | Ataque de fatiga activo. Identificar dispositivos/tenants; considerar bloqueo temporal del canal push del tenant. |
| **Recovery Code Storm** | `rate(recovery_used_total[1h]) > 5x baseline` | Alta | Pérdida masiva de dispositivos o campaña de takeover. Verificar notificaciones entregadas y activar revisión de cuentas. |
| **MFA Unavailable** | `rate(unavailable_total[2m]) > 0` | **Crítica** | Fail-secure activo: los usuarios no pueden completar segundos factores. Escalar según dependencia (Redis/KAGAMI). |
| **Verify Latency Regression** | `verify_duration_p95{method=totp} > 30ms` o `{webauthn} > 80ms` por 5 min | Media | Regresión de validación o contención de pools; descomponer por span. |
| **Push Approval Friction** | `push_approval_duration_p95 > 45s` (TTL 60s) por 10 min | Media | Usuarios llegando al límite del challenge: revisar UX del número matching o latencia de FCM/APNs. |
| **Enrollment Cancellation Spike** | `rate(enrollment_cancelled_total[1h]) > 3x baseline` | Media | UX de activación rota o sondeo de pendings; revisar motivos y conexiones afectadas. |

---

## 4. Patrones de Resiliencia

### 4.1. Presupuesto de Timeouts y Política de Reintentos

```csharp
public sealed record MfaTimeouts
{
    public TimeSpan RedisOperation       { get; init; } = TimeSpan.FromMilliseconds(50);
    public TimeSpan KagamiOperation      { get; init; } = TimeSpan.FromMilliseconds(200);
    public TimeSpan PushDispatch         { get; init; } = TimeSpan.FromSeconds(2);
    public TimeSpan ChallengeTtlPush     { get; init; } = TimeSpan.FromSeconds(60);
    public TimeSpan ChallengeTtlWebAuthn { get; init; } = TimeSpan.FromMinutes(5);
    public TimeSpan EnrollmentPendingTtl { get; init; } = TimeSpan.FromMinutes(10);
    public int PushDispatchRetries       { get; init; } = 1;   // ÚNICO retry del UC: canal no crítico
    public int Retries                   { get; init; } = 0;   // Política global: cero reintentos server-side
}
```

**Justificación de `Retries = 0`:** Las operaciones de verificación son single-use (GETDEL de mfa_token y challenges) y las escrituras de estado son CAS: reintentar server-side produciría `AlreadyResolved`/`TokenReuse` espurios o, peor, doble contabilización de throttling. El reintento correcto vive en el cliente (nuevo challenge) o en el dispositivo (nueva aprobación).

### 4.2. Circuit Breakers Diferenciados por Dependencia

| Dependencia | Comportamiento al Fallar | Recuperación |
| :--- | :--- | :--- |
| **Redis** (token/challenges/replay/throttle) | Fail-secure: verify/challenge rechazados con 503; **la pausa NO se invalida** (el token puede no haberse consumido) | Automática al cerrar; sin estado residual que reconciliar (TTLs naturales) |
| **KAGAMI** (enrollments/counter/activation) | 503 fail-secure; si falla el **counter update**, la completitud se rechaza aunque la firma sea válida | Automática; sin degradación intermedia aceptable (la detección de clonación no se negocia) |
| **Push Channel** (FCM/APNs) | Challenge retorna `push_unavailable` + ofrece los otros métodos del usuario | Inmediata al siguiente despacho exitoso |
| **Notification Service** | Completitud continúa; alertas de clonación/recovery encoladas vía Outbox con retry | Entrega eventual; nunca bloquea evidencia verificada |
| **UC-17 Risk** | El verify continúa (las señales de riesgo son aumento, no gate); se degrada la telemetría | Inmediata |

### 4.3. Asimetría Deliberada: Detección vs Conveniencia

Dos fallos de infraestructura tienen consecuencias **opuestas** por diseño:

| Fallo | Consecuencia | Justificación |
| :--- | :--- | :--- |
| Counter WebAuthn no persistido | **Completitud rechazada (503)** | Sin counter, la próxima verificación no detectaría clonación: seguridad dura. |
| Trusted device no emitido | **Completitud preservada** | Es conveniencia: la evidencia ya está verificada y auditada; degradar no aporta seguridad. |

### 4.4. Idempotencias y Contratos Externos

- **Invalidación de pausa idempotente:** `SETNX mfa:pause:invalid:{hash}` garantiza que el segundo y tercer reuso del token no dupliquen efectos.
- **CAS de challenge push:** aprobación y denegación concurrentes → exactamente un ganador; el perdedor recibe `AlreadyResolved` (200 al dispositivo, contrato push).
- **CAS de activación de enrollment:** `UPDATE ... WHERE status = 'PENDING'` hace imposible doble activación del mismo pending.
- **Regeneración de recovery:** supersede atómico del lote anterior; nunca coexisten dos lotes activos.
- **Polling del cliente:** lectura pura sin mutación; intervalo guía 2-3 s con jitter y backoff ante 429/503.

---

## 5. Feature Flags y Política por Tenant

### 5.1. Flags Globales

| Feature Flag | Descripción | Default | ¿Desactivable en incidente? |
| :--- | :--- | :--- | :--- |
| `kagi.auth.mfa.enabled` | Kill switch global del segundo factor | `true` | Sí (último recurso; degrada a AAL1) |
| `kagi.auth.mfa.email_otp_allowed` | Habilita el método restringido | `false` | Sí |
| `kagi.auth.mfa.totp_skew_steps` | Ventana de skew (0 o 1) | `1` | Sí (rango validado) |
| `kagi.auth.mfa.max_attempts_per_token` | Throttling por mfa_token | `5` | Sí |
| `kagi.auth.mfa.max_attempts_per_user_hour` | Throttling por usuario/hora | `10` | Sí |
| `kagi.auth.mfa.push_max_challenges_per_window` | Rate push por dispositivo | `3` | Sí |
| `kagi.auth.mfa.push_window_minutes` / `push_cooldown_minutes` | Ventana y cooldown de fatiga | `10` / `30` | Sí |
| `kagi.auth.mfa.challenge_ttl_push_seconds` / `challenge_ttl_webauthn_minutes` | TTLs de challenges | `60` / `5` | Sí |
| `kagi.auth.mfa.enrollment_pending_ttl_minutes` | Vida del pending | `10` | Sí |

### 5.2. Política por Tenant (configurada en UC-25, consumida como snapshot)

| Parámetro | Descripción | Default |
| :--- | :--- | :--- |
| `required_level` | Assurance exigido (AAL2 / AAL3) | `AAL2` |
| `require_phishing_resistant` | Solo WebAuthn verificable (recovery y email_otp excluidos) | `false` |
| `trusted_device_bypass_enabled` | Bypass de login (nunca step-up; incompatible con PR) | `false` |
| `trusted_device_ttl_days` | Vida del bypass | `14` |
| `step_up_freshness_minutes` | Reutilización de elevación | `10` |
| `allow_email_otp` | Método restringido por tenant (requiere flag global) | `false` |

### 5.3. Flags Bloqueados por Diseño (No Configurables)

| Invariante | Razón del bloqueo |
| :--- | :--- |
| Contención de clonación siempre activa | Un enrollment con counter regression es evidencia de compromiso; no existe modo "observar sin revocar". |
| Counter update fail-secure | La detección futura de clonación depende del counter persistido. |
| Riesgo forzado sin reuso de frescura | Un step-up disparado por UC-17 exige evidencia nueva; la frescura solo aplica a step-ups de scope. |
| Recovery no verificable bajo PR | El break-glass no es phishing-resistant; la recuperación exige re-enroll PR o vía administrativa (UC-21). |
| Trusted device nunca en step-up | INV-104: la fricción reducida no degrada operaciones sensibles. |

---

## 6. Estrategia de Testing

### 6.1. Authenticators Virtuales (Contract Testing)

El adversario y el usuario legítimo se modelan con los mismos instrumentos: una suite de authenticators virtuales que cubre comportamiento correcto y hostil.

| Authenticator Virtual | Capacidades | Casos que ejercita |
| :--- | :--- | :--- |
| `VirtualTotpAuthenticator` | Genera códigos RFC 6238 con reloj controlable | Happy path, skew ±1, replay del mismo step, expiración de ventana |
| `VirtualWebAuthnAuthenticator` | Firma assertions con clave de test; **counter manipulable** | Firma válida, counter creciente, **counter stale (clonación)**, origin/rpId inválidos, challenge reutilizado |
| `PushDeviceSimulator` | Aprueba/deniega con retardo configurable y número seleccionable | Number matching correcto, número incorrecto, attestation inválida, carrera approve/deny, denegaciones consecutivas |
| `RecoveryCodeWallet` | Guarda el lote emitido en activación | Consumo único, segundo uso, lote superseded |

```json
// Ejemplo: test de clonación con counter stale (k6 + VirtualWebAuthnAuthenticator)
{
  "scenario": "clone_detection",
  "steps": [
    { "action": "verify_webauthn", "counter": 10, "expect": "succeeded" },
    { "action": "verify_webauthn", "counter": 10, "expect": "failed_clone" },
    { "assert_metric": "kagi.auth.mfa.clone_detected_total == 1" },
    { "assert_db": "enrollment.status == 'REVOKED' AND reason == 'CloneDetected'" },
    { "assert_db": "otros enrollments del usuario == 'ACTIVE'" }
  ]
}
```

### 6.2. Load Testing (k6) con Mezcla de Métodos

```javascript
import http from 'k6/http';
import { check } from 'k6';
import { Trend, Rate } from 'k6/metrics';

const verifyLatency = new Trend('mfa_verify', true);
const errors = new Rate('errors');

export const options = {
  scenarios: {
    mfa_wave: {
      executor: 'ramping-arrival-rate',
      startRate: 100, timeUnit: '1s',
      preAllocatedVUs: 150, maxVUs: 700,
      stages: [
        { target: 300, duration: '2m' },
        { target: 500, duration: '5m' },   // ola de logins matutinos completando MFA
        { target: 500, duration: '3m' },
        { target: 0,   duration: '2m' },
      ],
    },
  },
  thresholds: {
    mfa_verify: ['p(95)<30', 'p(99)<80'],   // mezcla 70% TOTP / 20% WebAuthn / 10% push pre-aprobado
    errors: ['rate<0.005'],
    // Externos: clone_detected_total == 0; token_reuse_total == 0; throttled_total ≈ 0
  },
};

export default function () {
  const session = nextPausedSession();               // mfa_tokens pre-emitidos por setup (UC-11 mock)
  const method = pickMethod(0.7, 0.2, 0.1);          // totp / webauthn / push

  const challenge = http.post(`${__ENV.BASE_URL}/v1/auth/mfa/challenge`,
    JSON.stringify({ mfa_token: session.mfaToken, method }),
    { headers: { 'Content-Type': 'application/json' } });

  const proof = buildProof(method, challenge.json('challenge_id')); // authenticator virtual
  const t0 = Date.now();
  const verify = http.post(`${__ENV.BASE_URL}/v1/auth/mfa/verify`,
    JSON.stringify({ mfa_token: session.mfaToken, method, proof }),
    { headers: { 'Content-Type': 'application/json' } });
  verifyLatency.add(Date.now() - t0);

  if (!check(verify, { 'verify 200': (r) => r.status === 200 })) errors.add(1);
}
```

### 6.3. Escenarios de Abuso y Chaos

| Escenario | Inyección | Expectativas verificadas |
| :--- | :--- | :--- |
| **Throttling** | 6 códigos TOTP incorrectos con el mismo mfa_token | Quinto fallo → pausa invalidada (`throttled_total{dimension=token}`); sexto verify → `token_reuse`/`throttled`, nunca brute-force ilimitado |
| **Fatiga push** | 4 challenges push al mismo dispositivo en 10 min / 3 denegaciones seguidas | Cuarto challenge → cooldown 30 min + `push_fatigue_total{trigger=rate}`; denegaciones → `{trigger=denials}`; otros métodos del usuario siguen verificables |
| **Clonación** | Assertion firmada válida con counter ≤ almacenado | `clone_detected_total=1`; enrollment revocado; otros enrollments activos; SIEM alertada; cero completitudes con esa credencial |
| **Redis caído** | Kill de Redis Auth durante ola | 503 `mfa_unavailable{dependency=redis}`; **pausas no invalidadas**; al recuperar, verificación normal sin reconciliación |
| **KAGAMI caído tras firma WebAuthn** | Inyección de error en `counter.update` | Verify → 503; `Completer` sin llamadas (fail-secure del counter); ningún token elevado emitido |
| **FCM/APNs caído** | Fallo de despacho push | Challenge → `push_unavailable`; el verify ofrece TOTP/WebAuthn del usuario; `push_dispatch_failures_total` incrementa |

---

## 7. Deployment y Operaciones

### 7.1. Kubernetes HPA (Escalado por Latencia de Verificación)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: kagi-mfa-hpa }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: kagi-mfa }
  minReplicas: 4
  maxReplicas: 20
  metrics:
    - type: Pods
      pods:
        metric: { name: kagi_auth_mfa_verify_duration_p95 }
        target: { type: AverageValue, averageValue: "30" }   # mezcla dominada por TOTP
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 65 } }
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies: [{ type: Percent, value: 100, periodSeconds: 30 }]
```

### 7.2. Runbook de Operaciones

**Runbook 1: Clonación de Authenticator (Critical)**
1. Confirmar `clone_detected_total` y leer `MfaCloneDetected` en KAGE: identityId, enrollmentId, counters (almacenado vs presentado), IP.
2. Verificar con el titular: ¿cambió de dispositivo? ¿compartió la llave? Un counter stale legítimo (ej. restore de backup del authenticator) es raro pero posible.
3. Si no es legítimo: tratar como compromiso — revocar sesiones del usuario (UC-21), revisar otros enrollments, forzar re-enrollment con método PR si el tenant lo exige.
4. Preservar forense: la firma válida con counter regression es evidencia de clonación, no de robo de clave.

**Runbook 2: Campaña de Push Fatigue (High)**
1. Correlacionar `push_fatigue_total{trigger}` por dispositivo/tenant con IPs de verify fallidos.
2. Si es ataque: el cooldown ya está activo (30 min); evaluar bloqueo temporal del canal push del tenant (`kagi.auth.mfa.push_max_challenges_per_window=0` por tenant).
3. Comunicar al usuario objetivo: **nunca aprobar un push que no inició**; el number matching existe exactamente para esto.
4. Si hay aprobaciones sospechosas previas al cooldown: revocar sesiones y forzar re-autenticación.

**Runbook 3: Spike de Token Reuse (High)**
1. Verificar `token_reuse_total{origin_uc}`: ¿la interceptación viene de login (UC-11), refresh (UC-13) o federación (UC-14)?
2. Revisar TTLs vigentes de mfa_token y clientes que loguean el token (nunca debería persistirse en logs de proxy).
3. Si el patrón es multi-tenant: sospechar de cliente comprometido o extensión maliciosa; considerar reducir `mfa_token` TTL globalmente.
4. Las pausas ya están invalidadas (SETNX idempotente): el daño está contenido; el objetivo es encontrar el vector de interceptación.

**Runbook 4: Tormenta de Recovery Codes (High)**
1. Verificar `recovery_used_total` vs baseline y `MfaRecovery.Used` en KAGE (codeIndex, remaining, IP).
2. Cada uso ya disparó `mfa_review_required` + notificación: confirmar entrega de notificaciones.
3. Si usos distribuidos en múltiples identidades: posible campaña de takeover con códigos filtrados; activar revisión manual de cuentas afectadas.
4. Si uso único legítimo (pérdida de dispositivo): acompañar al usuario en el re-enrollment; el flujo ya exige revisión.

**Runbook 5: MFA Unavailable (Critical)**
1. Identificar dependencia en `unavailable_total{dependency}` (redis/kagami).
2. **No invalidar pausas manualmente:** el fail-secure las preserva; los usuarios reintentan tras la recuperación.
3. Evaluar con CISO si procede el kill switch `kagi.auth.mfa.enabled=false` (degrada a AAL1): solo en outage prolongado con impacto de negocio crítico, y con registro de la decisión.
4. Al recuperar: verificar que `challenge_total` y `verify_total` retornan a baseline sin picos de `throttled` (los reintentos no deben quemar umbrales: los fallos 503 no cuentan como intentos).

---

## 8. Compliance y Auditoría

### 8.1. NIST SP 800-63B (AAL2/AAL3 y Restricted Authenticators)
*   **Requisito:** MFA con verificadores aprobados; resistencia a verifier impersonation para AAL3; restricciones a authenticators OOB débiles.
*   **Implementación UC-15:** Catálogo con clasificación por método (WebAuthn PR → AAL3 con hardware+UV; TOTP/Push → AAL2; Email OTP restringido y default OFF); evidencia de método en `amr` y `auth_level` derivados, nunca declarados por el cliente.

### 8.2. RFC 6238 (TOTP)
*   **Requisito:** Secretos con entropía suficiente, ventana de tolerancia acotada.
*   **Implementación UC-15:** Secretos ≥160 bits cifrados AEAD en reposo; skew ≤1 step validado en constructor de `MfaPolicy`; replay guard por step.

### 8.3. W3C WebAuthn Level 2 / FIDO2
*   **Requisito:** Vinculación de origen (origin binding), verificación de attestation/assertion, manejo de signature counter.
*   **Implementación UC-15:** Origin/rpId validados contra el contexto; challenges single-use; counter monotónico con detección de clonación y revocación atómica de la credencial afectada.

### 8.4. OWASP ASVS 4.0 (V2) + OWASP MFA Cheat Sheet
*   **Requisito:** Resistencia a brute-force de segundos factores, verificación de posesión en enrollments, sin oracle de estado.
*   **Implementación UC-15:** Throttling en dos dimensiones + activación por prueba + colapso de trece motivos a respuesta idéntica.

### 8.5. PSD2 SCA RTS (Independencia de Canales)
*   **Requisito:** Los factores deben ser independientes (la compromisión de uno no compromete el otro).
*   **Implementación UC-15:** El segundo factor nunca deriva del primero: el secreto TOTP, la llave WebAuthn y el dispositivo push son independientes de la contraseña; el trusted device bypass aplica solo al MFA de login y es revocable.

### 8.6. GDPR Art. 32 / SOC 2 CC6.1-CC7.2 / ISO 27001 A.5.18-A.8.15
*   **Requisito:** Seguridad del tratamiento, control de acceso lógico, monitoreo y logging protegido.
*   **Implementación UC-15:** Recovery codes y secretos TOTP nunca en logs (hashes/ciphertext + `Clear()` de memoria); eventos de detección inmutables vía Outbox/KAGE; alertas de clonación/replay/fatiga conectadas al SIEM.

---

## 9. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | La matriz de colapso implementa los trece motivos de las excepciones 4.1-4.7; las métricas de detección monitorean INV-97 a INV-104. |
| `07_Sequence_Diagram.md` | Los spans por método (§7.5) y el presupuesto de timeouts (§7.1) son la operacionalización de las fases y latencias del diagrama de secuencia. |
| `08_Final_Class_Design.md` | `MfaController.Map`, `MfaMetrics`, `MfaTimeouts` y los adaptadores Redis/KAGAMI emiten exactamente las métricas y respetan los breakers definidos aquí. |
| `10_UC_Specific_Data_Model.md` | Los keyspaces `mfa:token:*`, `mfa:challenge:*`, `mfa:totp:laststep:*`, `mfa:throttle:*`, `mfa:push:cooldown:*` y las tablas de enrollments/recovery soportan métricas y runbooks. |
| `11_UC_Specific_Security_Threats.md` | Las alertas de clonación, replay, fatiga y token reuse operacionalizan las mitigaciones de esas amenazas. |
| `12_UC_Specific_Performance_Benchmarks.md` | Los SLOs de verify por método y los escenarios de authenticators virtuales se validan con los scripts k6 y chaos definidos aquí. |

---

## 10. Cierre del Paquete de Diseño Funcional (UC-15)

Con este documento, completamos los **9 archivos del Paquete de Diseño Profundo Funcional (Tier 1)** para el UC-15:

1. ✅ `01_Requirements_Specification.md` - Catálogo de métodos con clasificación de assurance, mfa_token single-use y cuatro completitudes.
2. ✅ `02_Use_Case_Diagram.md` - Siete casos núcleo, cuatro validaciones obligatorias y siete extensiones de contención/conveniencia.
3. ✅ `03_Activity_Diagram.md` - Challenge→Verify→Completitud, push inbound y enrollment con swimlanes.
4. ✅ `04_Initial_Domain_Model.md` - Enrollments con detecciones propias, challenges con CAS, evidencia única y cinco estrategias.
5. ✅ `05_State_Machine_Diagram.md` - Cinco máquinas: enrollment, challenge, mfa_token, dispositivo push y batch de recovery.
6. ✅ `06_Robustness_Diagram.md` - Colaboración ICONIX con fuente de verdad invertida en push y punto único de completitud.
7. ✅ `07_Sequence_Diagram.md` - Orden GETDEL→throttling→verificación→completitud, carrera approve/deny y enrollment con break-glass atómico.
8. ✅ `08_Final_Class_Design.md` - Typed Result con colapso ZIL compilable, contención atómica de clonación y asimetría counter/trusted-device.
9. ✅ `09_Modern_Artifacts.md` - OpenAPI de nueve endpoints, métricas de detección, resiliencia asimétrica, authenticators virtuales y runbooks.

### 10.1. Resumen de lo Logrado

Hemos diseñado el segundo factor como un **sistema de evidencia con adversario activo**:
- **Evidencia única, métodos heterogéneos:** Cinco estrategias convergen en `AssuranceEvidence`; la completitud consume evidencia, nunca pruebas crudas.
- **Ataques como métricas de primer nivel:** Clonación, replay, fatiga, reuso de token y break-glass son señales alertables con runbooks propios.
- **Degradaciones asimétricas justificadas:** Fail-secure para detección (counter, replay), best-effort para conveniencia (trusted device, notificaciones), y fallback de canal (push caído → otros métodos).
- **Concurrencia resuelta por construcción:** GETDEL, CAS y UPDATEs condicionales eliminan doble completitud, doble aprobación y doble activación.
- **Opacidad del contrato:** Trece motivos internos colapsados a `mfa_failed`; la riqueza vive en auditoría y métricas, nunca en la respuesta.
- **Cumplimiento verificable:** NIST 800-63B, RFC 6238, WebAuthn L2, OWASP ASVS V2, PSD2 SCA e ISO 27001 mapeados a implementaciones concretas.

### 10.2. Próximos Pasos (Documentos Técnicos Específicos)

Para el UC-15 aún faltan los 3 documentos de la Fase 2:

1. **`10_UC_Specific_Data_Model.md`** - Tablas `mfa_enrollments` (secretos cifrados, credentials WebAuthn con counter), `recovery_codes` (hashes), trusted devices, y keyspaces Redis de challenges/replay/throttle/cooldown.
2. **`11_UC_Specific_Security_Threats.md`** - Threat modeling de brute-force de TOTP, replay, clonación de authenticators, push fatigue/phishing, robo de recovery codes, reciclaje de mfa_token y abuso de trusted devices.
3. **`12_UC_Specific_Performance_Benchmarks.md`** - SLAs de verify por método, capacidad bajo ola de MFA matutina, costo de aprobación push mediada por humano, y benchmarks de contención (clone/replay/fatigue) bajo fuego.

---
*Fin del Documento 09_Modern_Artifacts para UC-15*