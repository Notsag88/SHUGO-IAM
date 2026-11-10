# UC-16: Passwordless Authentication - Artefactos Modernos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-16 - Passwordless Authentication (Passkeys + Magic Links + OTP Restringido) |
| **Documento:** | 09_Modern_Artifacts.md |
| **Metodología:** | API-First + Observability (OpenTelemetry) + Resilience Patterns + Virtual Authenticator Testing + Email Sink Contract Testing + DevSecOps |
| **Estándares:** | NIST SP 800-63B (Verifier Impersonation Resistance, Restricted Authenticators), W3C WebAuthn Level 2 / FIDO2, RFC 8176 (amr), OWASP ASVS 4.0 (V2) + OWASP Passwordless Authentication Cheat Sheet, PSD2 SCA RTS, GDPR Art. 5/32, SOC 2 CC6.1/CC7.2, ISO 27001:2022 A.5.18/A.8.15 |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 10/11/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 10/11/2026 | Gastón Ruiz | **Creación inicial.** Definición de los artefactos modernos para UC-16: contratos OpenAPI 3.1 de los nueve endpoints passwordless con matriz de colapso de los diez `PasswordlessOutcome` y dieciocho `FailReason`, catálogo de métricas OpenTelemetry con funnel completo del magic link (issued → validated → consumed), señales de seguridad como métricas de primer nivel (clonación, replay, stuffing, agotamiento), patrones de resiliencia con el único retry del UC en el despacho de email, Feature Flags por tenant con invariantes bloqueadas, estrategia de testing con authenticators virtuales (UV apagable, counter manipulable) y email sink para E2E de enlaces, y runbooks por tipo de incidente. | Cierra el paquete de diseño Tier 1 funcional del UC-16, proporcionando todos los artefactos necesarios para operar la autenticación sin contraseña como un sistema medible: cada canal tiene su funnel, cada ataque su señal, cada degradación su comportamiento documentado, y el contrato HTTP jamás revela la decisión interna de despacho. |

---

## 1. Introducción y Propósito

Este documento define los **Artefactos Modernos** necesarios para operar el UC-16 (Passwordless Authentication) en producción. A diferencia de los UCs de autenticación con secretos, el passwordless opera sobre artefactos efímeros mediados por canales externos (email, authenticators), y sus artefactos deben satisfacer cuatro demandas simultáneas:

1. **Contratos opacos con riqueza interna:** Nueve endpoints que colapsan dieciocho motivos de fallo a cuerpos genéricos, mientras las métricas conservan el motivo exacto — el funnel del magic link y las señales de abuso son la verdad operativa.
2. **Observabilidad de flujos mediados por humano:** El magic link tiene un tramo fuera de nuestro control (entrega de email + decisión del usuario) que se mide como **funnel de conversión**, no como latencia; el OTP se mide por intentos y agotamiento.
3. **Resiliencia asimétrica por criticidad:** Redis/KAGAMI caídos → fail-secure absoluto; email caído → métrica con supersede; riesgo caído → fail-mode por política; pause broker caído con AAL2+ → bloqueo de emisión insuficiente.
4. **Verificación con adversario simulado:** Los tests ejercitan authenticators con UV apagado y counters manipulados, emails con prefetch agresivo, y campañas de stuffing — porque el passwordless se ataca por los canales, no por la criptografía.

---

## 2. Contratos OpenAPI Completos

### 2.1. Especificación OpenAPI 3.1

```yaml
openapi: 3.1.0
info:
  title: SHUGO KAGI - Passwordless Authentication API
  description: |
    Endpoints de autenticación sin contraseña del motor KAGI.
    
    ## Características Clave
    - Passkeys: WebAuthn como primer factor con userVerification obligatorio (AAL2/3).
    - Magic Links: token 256 bits single-use, TTL ≤10 min, consumo en dos pasos
      (GET valida sin consumir; POST confirma y consume), handoff cross-device opcional.
    - Email OTP: 8 dígitos, TTL 5 min, 5 intentos, 3 emisiones/hora, default OFF.
    - Emisión silenciosa: las solicitudes OOB retornan 202 idéntico exista o no la identidad.
    - Session code: puente one-time de 60s; los tokens jamás viajan en URLs.
    
    ## Seguridad
    - Los 18 motivos internos de fallo convergen a `passwordless_failed` genérico.
    - El handoff code viaja solo en la respuesta al dispositivo origen.
    - Bajo require_phishing_resistant, solo passkeys son verificables.
  version: 1.0.0

servers:
  - url: https://auth.shugo.io/v1
    description: Producción

paths:
  /auth/passwordless/webauthn/options:
    post:
      operationId: passkeyOptions
      summary: Iniciar ceremonia passkey (options)
      tags: [Passkey]
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                username_hint: { type: string }
      responses:
        '200':
          description: PublicKeyCredentialRequestOptions (UV required)
          headers: { Cache-Control: { schema: { type: string }, description: "no-store" } }
        '503': { $ref: '#/components/responses/Unavailable' }

  /auth/passwordless/webauthn/verify:
    post:
      operationId: passkeyVerify
      summary: Completar ceremonia passkey (escalera de guardas + emisión)
      tags: [Passkey]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/WebAuthnAssertion' }
      responses:
        '200': { description: "session_code (60s) o mfa_required (solo riesgo)", content: { application/json: { schema: { $ref: '#/components/schemas/SessionCodeOrMfa' } } } }
        '400': { $ref: '#/components/responses/Failed' }
        '503': { $ref: '#/components/responses/Unavailable' }

  /auth/passwordless/magic-link:
    post:
      operationId: magicLinkRequest
      summary: Solicitar magic link (emisión silenciosa)
      tags: [Magic Link]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [email]
              properties:
                email: { type: string, format: email }
                redirect_uri: { type: string, format: uri }
      responses:
        '202':
          description: "Respuesta idéntica exista o no la identidad"
          content:
            application/json:
              schema:
                type: object
                properties:
                  message: { type: string, example: "If the account exists and this sign-in method is enabled, a sign-in link has been sent." }
                  handoff_code: { type: string, description: "Solo si cross_device_handoff está habilitado" }
        '503': { $ref: '#/components/responses/Unavailable' }

  /auth/passwordless/link:
    get:
      operationId: magicLinkValidate
      summary: Validar magic link (GET puro: NO consume)
      tags: [Magic Link]
      parameters:
        - { name: token, in: query, required: true, schema: { type: string } }
      responses:
        '200': { description: "Vista de confirmación: ready_same_device | ready_handoff | device_mismatch | invalid" }

  /auth/passwordless/link/confirm:
    post:
      operationId: magicLinkConfirm
      summary: Confirmar magic link (gate → consumo → emisión/pausa)
      tags: [Magic Link]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [token]
              properties:
                token: { type: string }
                handoff_code: { type: string }
      responses:
        '200': { description: "session_code o mfa_required (pausa AAL2+)" }
        '302': { description: "redirect_uri?session_code=... (clientes web)" }
        '400': { $ref: '#/components/responses/Failed' }
        '503': { $ref: '#/components/responses/Unavailable' }

  /auth/passwordless/otp/request:
    post:
      operationId: otpRequest
      summary: Solicitar OTP de email (emisión silenciosa, canal restringido)
      tags: [Email OTP]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [email]
              properties:
                email: { type: string, format: email }
      responses:
        '202': { description: "Respuesta idéntica exista o no la identidad" }
        '503': { $ref: '#/components/responses/Unavailable' }

  /auth/passwordless/otp/verify:
    post:
      operationId: otpVerify
      summary: Verificar OTP (5 intentos, single-use, pausa AAL2+)
      tags: [Email OTP]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [email, code]
              properties:
                email: { type: string, format: email }
                code: { type: string, pattern: '^\d{8}$' }
      responses:
        '200': { description: "session_code o mfa_required" }
        '400': { $ref: '#/components/responses/Failed' }
        '503': { $ref: '#/components/responses/Unavailable' }

  /auth/passwordless/token:
    post:
      operationId: passwordlessRedeem
      summary: Canjear session_code por tokens (único camino de entrega)
      tags: [Session]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [session_code]
              properties:
                session_code: { type: string }
      responses:
        '200': { description: "Access + Refresh tokens con session_meta (auth_method passwordless_*, amr, auth_level)" }
        '400': { $ref: '#/components/responses/Failed' }

  /auth/passwordless/webauthn/enroll:
    post:
      operationId: passkeyEnroll
      summary: Enrollment de passkey (sesión autenticada; step-up según política)
      tags: [Passkey]
      security: [ { BearerAuth: [] } ]
      requestBody:
        content:
          application/json:
            schema:
              type: object
              properties:
                attestation: { type: object, description: "Segunda llamada: resultado de la ceremonia de registro" }
      responses:
        '200': { description: "RegistrationOptions (1ª llamada) o enrollment activo (2ª llamada)" }
        '403': { description: "step_up_required: la sesión no alcanza la assurance exigida" }

components:
  securitySchemes:
    BearerAuth: { type: http, scheme: bearer, bearerFormat: JWT }
  responses:
    Failed:
      description: "passwordless_failed genérico (18 motivos internos colapsados)"
      content:
        application/json:
          schema:
            type: object
            properties:
              error:
                type: object
                properties:
                  code: { type: string, enum: [passwordless_failed] }
                  message: { type: string, example: "Passwordless verification could not be completed." }
    Unavailable:
      description: "Fail-secure (Redis/KAGAMI/PauseBroker caído); sin efectos laterales"
      headers: { Retry-After: { schema: { type: integer } } }
      content:
        application/json:
          schema:
            type: object
            properties:
              error:
                type: object
                properties:
                  code: { type: string, enum: [passwordless_unavailable] }
                  message: { type: string }
  schemas:
    WebAuthnAssertion:
      type: object
      required: [assertion]
      properties:
        assertion:
          type: object
          properties:
            id: { type: string }
            response:
              type: object
              properties:
                clientDataJSON: { type: string }
                authenticatorData: { type: string }
                signature: { type: string }
                userHandle: { type: string }
    SessionCodeOrMfa:
      type: object
      properties:
        session_code: { type: string }
        expires_in: { type: integer, example: 60 }
        mfa_required: { type: boolean }
        mfa_token: { type: string }
```

### 2.2. Matriz de Comportamiento (Colapso de Outcomes)

| `PasswordlessOutcome` | HTTP | Cuerpo | Métrica principal |
| :--- | :--- | :--- | :--- |
| `CeremonyOptions` | **200** | PublicKey options (no-store) | `options_total{channel=passkey}` |
| `IssuanceAccepted` | **202** | Mensaje genérico (+ handoff si aplica) | `issuance_requests_total{channel}` |
| `LinkView` | **200** | Vista de confirmación (sin consumo) | `magiclink_validated_total{view}` |
| `SessionIssued` | **200/302** | session_code o redirect con code | `sessions_total{channel}` |
| `PausedForMfa` | **200** | `{ mfa_required, mfa_token }` | `pause_by_assurance_total{channel}` o `risk_stepup_total` |
| `StepUpRequired` | **403** | `step_up_required` | `enrollment_stepup_total` |
| `Enrolled` | **200** | Credencial activa | `passkey_enrolled_total` |
| `TokensIssued` | **200** | Tokens + `session_meta` | `redeem_total{result=succeeded}` |
| `Failed(reason)` | **400** | `passwordless_failed` (único) | `failed_total{reason}` (18 labels) |
| `ServiceUnavailable(dep)` | **503** | `passwordless_unavailable` + Retry-After | `unavailable_total{dependency}` |

---

## 3. Catálogo de Métricas OpenTelemetry

### 3.1. Métricas de Canal y Funnel

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagi.auth.passwordless.verify_total` | Counter | verificaciones | Verificaciones por canal y resultado | `channel`, `result` |
| `kagi.auth.passwordless.verify_duration` | Histogram | ms | Latencia local de verify por canal | `channel` |
| `kagi.auth.passwordless.issuance_requests_total` | Counter | solicitudes | Solicitudes OOB (magic link / OTP) | `channel`, `tenant_id` |
| `kagi.auth.passwordless.magiclink_issued_total` | Counter | emisiones | Enlaces efectivamente emitidos (despacho iniciado) | `tenant_id` |
| `kagi.auth.passwordless.magiclink_validated_total` | Counter | GETs | Validaciones puras (incluye prefetch) | `view` (ready/handoff/mismatch/invalid) |
| `kagi.auth.passwordless.magiclink_consumed_total` | Counter | consumos | Confirmaciones exitosas | `cross_device` |
| `kagi.auth.passwordless.magiclink_failed_total` | Counter | fallos | Por stage interno | `stage` (expired/consumed/device_mismatch/handoff_invalid/risk_denied) |
| `kagi.auth.passwordless.magiclink_ttl_elapsed` | Histogram | s | Tiempo solicitud→consumo (UX de entrega) | `tenant_id` |
| `kagi.auth.passwordless.otp_attempts_total` | Counter | intentos | Intentos de verificación OTP | `result` |
| `kagi.auth.passwordless.otp_exhausted_total` | Counter | agotamientos | Emisiones invalidadas por 5 fallos | `tenant_id` |
| `kagi.auth.passwordless.otp_superseded_total` | Counter | supersedes | Emisiones reemplazadas por re-solicitud | `tenant_id` |

### 3.2. Métricas de Seguridad (Detecciones de Primer Nivel)

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagi.auth.passwordless.clone_detected_total` | Counter | eventos | **Clonación de passkey confirmada** (counter regression) | `tenant_id` |
| `kagi.auth.passwordless.replay_signals_total` | Counter | señales | Replays de enlace consumido / ceremony challenge | `channel` |
| `kagi.auth.passwordless.unknown_email_total` | Counter | solicitudes | Solicitudes OOB sin identidad (interno, jamás expuesto) | `channel`, `tenant_realm` |
| `kagi.auth.passwordless.rate_suspended_total` | Counter | suspensiones | Rate limit silencioso aplicado | `channel`, `dimension` (email/ip) |
| `kagi.auth.passwordless.uv_missing_total` | Counter | rechazos | Assertions sin userVerification | `tenant_id` |
| `kagi.auth.passwordless.handoff_failures_total` | Counter | fallos | Handoff codes incorrectos | `tenant_id` |

### 3.3. Métricas de Despacho y Disponibilidad

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagi.auth.passwordless.dispatch_failures_total` | Counter | fallos | Despachos de email fallidos (tras 1 retry) | `channel` |
| `kagi.auth.passwordless.dispatch_duration` | Histogram | ms | Latencia del transporte de email (async) | `channel` |
| `kagi.auth.passwordless.pause_by_assurance_total` | Counter | pausas | Emisiones AAL1 pausadas hacia UC-15 | `channel`, `tenant_id` |
| `kagi.auth.passwordless.unavailable_total` | Counter | 503s | Fail-secure por dependencia | `dependency` (redis/kagami/pause_broker) |
| `kagi.auth.passwordless.sessions_total` | Counter | sesiones | Sesiones passwordless emitidas (adopción vs password) | `channel`, `auth_level` |
| `kagi.auth.passwordless.passkey_enrolled_total` / `passkey_revoked_total` | Counter | credenciales | Ciclo de vida de credenciales | `reason` |

### 3.4. Distributed Tracing (Spans por Canal)

```text
POST /passwordless/webauthn/verify (Root) [~55ms]
├── redis.challenge.getdel [2ms]
├── guard.origin_uv [0.2ms]
├── kagami.credential.fetch [3ms]
├── webauthn.verify_signature [1ms]
├── kagami.counter.cas [4ms]          ← fail-secure: bloquea emisión si falla
├── risk.evaluate [30ms]
└── issuer.complete [15ms]            ← familia + session_code + outbox

POST /passwordless/link/confirm (Root) [~70ms]
├── redis.link.peek [2ms]
├── gate.binding_handoff [0.5ms]
├── redis.link.getdel [2ms]           ← single-writer
├── risk.evaluate [30ms]
└── issuer.or_pause [20ms]

POST /passwordless/magic-link (Root) [~18ms — respuesta ANTES del despacho]
├── abuse.evaluate [2ms]
├── kagami.identity.lookup [5ms]
├── redis.token.set [2ms]
├── outbox.insert [4ms]
└── email.dispatch [async, fuera del span raíz]
```

### 3.5. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Passkey Clone Detected** | `clone_detected_total` ≥ 1 | **Crítica** | Runbook 1: authenticator clonado. Verificar con el titular, revocar sesiones (UC-21), preservar forense de counters. |
| **Magic Link Replay Spike** | `rate(replay_signals_total{channel=magic_link}[10m]) > 3` | **Crítica** | Posible intercepción de enlaces. Runbook 2: correlacionar IPs de confirmación vs solicitud, evaluar suspensión temporal del canal. |
| **Stuffing Campaign** | `rate(rate_suspended_total[10m]) > 5x baseline` o `unknown_email_total` concentrado por realm | Alta | Runbook 3: campaña de enumeración/credential stuffing. Escalar a WAF; la respuesta genérica ya opera. |
| **OTP Exhaustion Storm** | `rate(otp_exhausted_total[1h]) > 5x baseline` | Alta | Ataques de fuerza bruta sobre OTP o UX rota. Verificar distribuciones de intentos por identidad. |
| **Dispatch Failure Storm** | `rate(dispatch_failures_total[5m]) > 0.05 × issuance_requests` | Alta | Runbook 4: proveedor de email degradado. Los tokens persisten; comunicar re-solicitud; passkeys/federación intactos. |
| **Magic Link Funnel Collapse** | `consumed/issued < 0.5` sostenido 30 min (baseline ~0.8) | Media | Email no llega (spam, proveedor) o UX de confirmación rota; cruzar con `dispatch_failures` y `magiclink_ttl_elapsed` p95. |
| **UV Missing Spike** | `rate(uv_missing_total[10m]) > 0` | Media | Cliente enviando assertions sin UV (bug de integración o sonda). Verificar versiones de SDK. |
| **Passwordless Unavailable** | `rate(unavailable_total[2m]) > 0` | **Crítica** | Runbook 5 según `dependency`. Fail-secure activo: ninguna emisión en curso. |
| **Verify Latency Regression** | `verify_duration_p95{channel}` sobre SLO por 5 min | Alta | Descomponer por span; típicamente KAGAMI o risk. |

---

## 4. Patrones de Resiliencia

### 4.1. Presupuesto de Timeouts y Política de Reintentos

```csharp
public sealed record PasswordlessTimeouts
{
    public TimeSpan RedisOperation  { get; init; } = TimeSpan.FromMilliseconds(50);
    public TimeSpan KagamiOperation { get; init; } = TimeSpan.FromMilliseconds(200);
    public TimeSpan EmailDispatch   { get; init; } = TimeSpan.FromSeconds(2);
    public TimeSpan RiskEvaluation  { get; init; } = TimeSpan.FromMilliseconds(100);
    public TimeSpan ChallengeTtl    { get; init; } = TimeSpan.FromMinutes(5);
    public TimeSpan MagicLinkTtl    { get; init; } = TimeSpan.FromMinutes(10);   // tope INV-106
    public TimeSpan OtpTtl          { get; init; } = TimeSpan.FromMinutes(5);    // tope RN-38
    public TimeSpan SessionCodeTtl  { get; init; } = TimeSpan.FromSeconds(60);
    public int EmailDispatchRetries { get; init; } = 1;   // ÚNICO retry del UC
    public int Retries              { get; init; } = 0;   // GETDEL/SET/CAS no reintentables
}
```

**Justificación del retry único:** El despacho de email es la única operación idempotente y no crítica del UC (el token ya está persistido; la re-solicitud supersede). Todo lo demás es single-use o CAS: reintentar produciría dobles consumos o transiciones contradictorias.

### 4.2. Comportamientos de Fallo por Dependencia

| Dependencia | Al fallar | Comportamiento | Recuperación |
| :--- | :--- | :--- | :--- |
| **Redis** (tokens/challenges/codes/rate) | Breaker tras 3 fallos | 503 fail-secure; ningún artefacto consumido ni emitido; solicitudes OOB retornan 503 | Automática; TTLs naturales limpian estado |
| **KAGAMI** (identidades/credenciales/familias) | Breaker tras 5 fallos | 503 fail-secure; counter CAS fallido bloquea emisión passkey | Automática |
| **Proveedor de email** | Timeout 2s + 1 retry | Token persistido; `dispatch_failures_total`; el usuario re-solicita (supersede) | Inmediata al siguiente despacho |
| **UC-17 Risk** | Timeout 100ms | Fail-mode por política: `open` (continúa con warning) o `stepup` (degrada a MFA) | Inmediata |
| **UC-15 Pause Broker** | Error en pausa | **503 si `required_level ≥ AAL2`**: nunca emisión insuficiente por indisponibilidad | Automática; passkeys no afectados |
| **KAGE/Outbox** | Error de inserción | Completitud continúa; eventos reintentados por el worker | Entrega eventual |

### 4.3. Idempotencias y Contratos de Operación

- **GET de validación:** Lectura pura; repetible infinitamente (prefetch-proof por contrato de puerto).
- **Confirmación doble:** GETDEL → un ganador; el perdedor recibe `LinkConsumed` + señal de replay.
- **Re-solicitud OOB:** Supersede atómico (SET que reemplaza) — sin coexistencia de artefactos.
- **Canje doble de session_code:** GETDEL → segundo canje falla genérico.
- **Enrollment duplicado:** Índice único por `(tenant, credentialId)` + inserción condicional.

### 4.4. Degradación del Usuario Final

Cuando un canal falla, el sistema **ofrece los demás canales habilitados del usuario** en lugar de bloquear: passkey con despacho email caído funciona; magic link con Redis caído no (fail-secure), pero el usuario puede usar passkey o password según política del tenant. La regla: **la caída de un canal nunca invalida los otros**.

---

## 5. Feature Flags y Política por Tenant

### 5.1. Flags Globales

| Feature Flag | Descripción | Default | ¿Desactivable en incidente? |
| :--- | :--- | :--- | :--- |
| `kagi.auth.passwordless.enabled` | Kill switch global | `true` | Sí (degrada a password/federación) |
| `kagi.auth.passwordless.email_dispatch_retries` | Reintentos de transporte | `1` | Sí |
| `kagi.auth.passwordless.dummy_compare_enabled` | Dummy hash anti-timing en OTP | `true` | **NO — bloqueado** |
| `kagi.auth.passwordless.session_code_ttl_seconds` | Vida del puente de canje | `60` | Sí (rango 30-120) |

### 5.2. Política por Tenant (UC-25, consumida como snapshot)

| Parámetro | Descripción | Default |
| :--- | :--- | :--- |
| `channels.passkey` | Passkeys habilitados | `true` |
| `channels.magic_link` | Magic link habilitado | `false` (opt-in) |
| `channels.email_otp` | OTP restringido habilitado | `false` |
| `required_level` | Assurance exigido (AAL1/AAL2/AAL3) | `AAL2` |
| `require_phishing_resistant` | Solo passkeys verificables | `false` |
| `cross_device_handoff` | Handoff de 6 dígitos para magic link | `false` |
| `magic_link_ttl_minutes` | TTL del enlace (≤10, validado) | `10` |
| `otp_max_per_hour` | Emisiones OTP por identidad/hora | `3` |
| `attestation_policy` | none / indirect / direct para passkeys | `indirect` |
| `risk_fail_mode` | open / stepup ante timeout de UC-17 | `open` |

### 5.3. Invariantes Bloqueadas por Diseño (No Configurables)

| Invariante | Razón del bloqueo |
| :--- | :--- |
| UV obligatorio en passkeys (INV-105) | Sin UV no hay evidencia de usuario presente; el canal dejaría de ser AAL2 |
| Magic link TTL ≤ 10 min (INV-106) | Ventana de intercepción acotada por diseño |
| Consumo en dos pasos (RN-37) | El GET nunca consume; inmunidad a prefetch estructural |
| Respuesta 202 idéntica (INV-109) | La anti-enumeración no es negociable por tenant |
| Contención de clonación siempre activa | Un counter regression es evidencia de compromiso, no configuración |
| Emisión insuficiente imposible | Pause broker caído + política AAL2 → 503, nunca AAL1 |
| Recovery de identidad preexistente (INV-111) | Passwordless no crea identidades; eso es provisioning |

---

## 6. Estrategia de Testing

### 6.1. Authenticators Virtuales y Email Sink

El adversario y el usuario legítimo se modelan con los mismos instrumentos:

| Instrumento | Capacidades | Casos que ejercita |
| :--- | :--- | :--- |
| `VirtualPasskeyAuthenticator` | Firma assertions con clave de test; **UV conmutable**, **counter manipulable**, origin configurable | Happy path, UV ausente, counter stale (clonación), origin foráneo |
| `EmailCaptureSink` | Captura emails del dispatcher en cola de test; extrae token/código | E2E de magic link y OTP, prefetch simulado, expiración por TTL acelerado |
| `VirtualHuman` | Retardos de apertura/clic distribuidos; denegaciones | Funnel de magic link, TTL elapsed, device mismatch |
| `ProbeGenerator` | Solicitudes OOB masivas sobre listas de emails | Stuffing, rate silencioso, respuesta idéntica |

### 6.2. Contract Testing E2E del Magic Link

```javascript
// Flujo completo con EmailCaptureSink:
// 1. POST /magic-link → 202 genérico
// 2. sink.awaitEmail(identity) → extraer token del enlace
// 3. GET /link?token → vista ready_same_device (NO consume)
// 4. GET /link?token (2ª vez, simulando scanner) → misma vista, token intacto
// 5. POST /link/confirm → session_code
// 6. POST /link/confirm (replay) → passwordless_failed + replay_signals_total++
// 7. POST /passwordless/token → tokens con amr=["otk"], auth_level=AAL1
```

### 6.3. Load Testing (k6) por Canal

```javascript
export const options = {
  scenarios: {
    passkey_wave: {
      executor: 'ramping-arrival-rate',
      startRate: 50, timeUnit: '1s',
      preAllocatedVUs: 150, maxVUs: 600,
      stages: [{ target: 300, duration: '5m' }, { target: 300, duration: '5m' }, { target: 0, duration: '2m' }],
      exec: 'passkeyCeremony',   // options + verify con VirtualPasskeyAuthenticator
    },
    magiclink_funnel: {
      executor: 'constant-arrival-rate', rate: 50, timeUnit: '1s', duration: '15m',
      exec: 'magicLinkWithVirtualHuman',   // request → sink → GET → delay humano → confirm
    },
    stuffing_probe: {
      executor: 'constant-arrival-rate', rate: 100, timeUnit: '1s', duration: '5m',
      startTime: '5m', exec: 'probeEmailList',   // 80% emails inexistentes
    },
  },
  thresholds: {
    'verify_duration{channel:passkey}': ['p(95)<80'],
    'verify_duration{channel:otp}': ['p(95)<50'],
    // Externos: todas las respuestas de probe == 202 idéntico (comparación de cuerpos byte-a-byte)
    // consumed/issued > 0.75 en funnel; replay_signals == 0 en tráfico legítimo
  },
};
```

### 6.4. Escenarios de Abuso y Chaos

| Escenario | Inyección | Expectativas verificadas |
| :--- | :--- | :--- |
| **Clonación** | Assertion válida con counter ≤ almacenado | `clone_detected_total == 1`; credencial REVOKED; cero sesiones con esa credencial; otros métodos del usuario activos |
| **Prefetch agresivo** | 100 GETs al enlace antes del confirm | Token intacto; el confirm posterior emite normalmente |
| **Carrera de confirmaciones** | 2 POST simultáneos del mismo token | Exactamente 1 sesión; 1 `LinkConsumed` + señal de replay |
| **Stuffing masivo** | 10k solicitudes OOB (90% inexistentes) | 100% respuestas 202 idénticas; `rate_suspended_total` > 0; despachos ≤ límite por email |
| **Redis caído** | Kill de Redis Auth durante ola | 503 `unavailable{redis}`; **ningún** artefacto consumido ni emitido; al recuperar, operación normal sin reconciliación |
| **KAGAMI caído tras firma** | Error en `SaveCounterCasAsync` | Verify → 503; cero sesiones emitidas; credencial intacta con counter antiguo |
| **Proveedor email caído** | Fallo de despacho sostenido | `dispatch_failures_total` crece; tokens persistidos; re-solicitud supersede; passkeys sin impacto |
| **Pause Broker caído (tenant AAL2)** | Error en `PauseForMfaAsync` | Magic link/OTP → 503; **cero emisiones AAL1**; passkeys continúan emitiendo AAL2 |

---

## 7. Deployment y Operaciones

### 7.1. Kubernetes HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: kagi-passwordless-hpa }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: kagi-passwordless }
  minReplicas: 4
  maxReplicas: 20
  metrics:
    - type: Pods
      pods:
        metric: { name: kagi_auth_passwordless_verify_duration_p95 }
        target: { type: AverageValue, averageValue: "30" }   # mezcla dominada por passkey
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 65 } }
```

### 7.2. Runbook de Operaciones

**Runbook 1: Clonación de Passkey (Crítica)**
1. Leer `PasskeyCloneDetected` en KAGE: identityId, credentialId, counters (almacenado vs presentado), IP del verify.
2. Contactar al titular: ¿migró de dispositivo con backup del authenticator? (causa benigna rara) ¿reconoce el verify?
3. Si no es legítimo: revocar sesiones del usuario (UC-21), revisar otras credenciales de la identidad, forzar re-enrollment con attestation directa si el tenant lo permite.
4. Preservar forense: la firma válida con counter stale es evidencia de clonación de clave, no de robo de sesión.

**Runbook 2: Intercepción/Replay de Magic Links (Crítica)**
1. Confirmar `replay_signals_total{channel=magic_link}` y leer `MagicLinkFailed{stage=consumed}` con IPs de confirmación.
2. Correlacionar IP/geo de solicitud vs confirmación en los eventos `MagicLinkConsumed`: divergencia sistemática = enlaces abiertos en otro contexto.
3. Evaluar suspensión temporal del canal por tenant (`channels.magic_link=false`) y comunicación a usuarios afectados.
4. Verificar infraestructura de email (headers de los correos entregados): ¿reenvío automático? ¿scanner que sigue enlaces con session del usuario?

**Runbook 3: Campaña de Stuffing (Alta)**
1. Verificar `rate_suspended_total{dimension}` y `unknown_email_total{tenant_realm}`: concentración por realm indica targeting.
2. La respuesta genérica ya opera — no cambiar nada visible. Escalar límites al WAF si el volumen es DDoS.
3. Cruzar con UC-11: si la misma lista de emails aparece en intentos de password, es credential stuffing combinado; activar UC-17 en modo estricto para el tenant.

**Runbook 4: Tormenta de Fallos de Despacho (Alta)**
1. Verificar `dispatch_failures_total` y cuota/estado del proveedor (SES).
2. Los tokens están persistidos: el impacto es UX (el usuario re-solicita y supersede), no seguridad.
3. Si la caída es prolongada: comunicar a usuarios, recordar que passkeys y federación siguen operativos; nunca degradar la política de assurance por indisponibilidad de email.

**Runbook 5: Passwordless Unavailable (Crítica)**
1. Identificar `unavailable_total{dependency}`: redis / kagami / pause_broker.
2. **Redis/KAGAMI:** fail-secure activo; ninguna emisión en curso; los usuarios reintentan tras recuperación sin pérdida de primer factor.
3. **Pause broker:** los canales OOB de tenants AAL2+ están bloqueados por diseño (nunca emisión insuficiente); los passkeys siguen emitiendo. Escalar a UC-15; no bajar `required_level` como "solución".

---

## 8. Compliance y Auditoría

### 8.1. NIST SP 800-63B (Verificadores y Autenticadores)
*   **Requisito:** Resistencia a verifier impersonation para assurance alta; clasificaciónde authenticators restringidos.
*   **Implementación UC-16:** Passkeys con UV como verificadores phishing-resistant (AAL2, AAL3 con hardware+attestation directa); magic link y email OTP clasificados AAL1 restringidos que pausan hacia segundo factor bajo política AAL2+.

### 8.2. W3C WebAuthn Level 2 / FIDO2
*   **Requisito:** Vinculación de origen, verificación de UV, manejo de signature counter, políticas de attestation.
*   **Implementación UC-16:** Escalera de guardas origin→UV→registro→firma→counter; CAS condicional de counter; attestation por política de tenant; contención de clonación transaccional.

### 8.3. RFC 8176 (Authentication Method References)
*   **Requisito:** Evidencia del método de autenticación en claims.
*   **Implementación UC-16:** `amr=["hwk","user"]` (passkey), `["otk"]` (magic link), `["otp"]` (email OTP) grabados en la proveniencia de cada familia.

### 8.4. OWASP ASVS 4.0 (V2) + Passwordless Cheat Sheet
*   **Requisitos:** Verificadores de un solo uso con TTL corto, sin tokens en logs/URLs, resistencia a enumeración.
*   **Implementación UC-16:** Single-use GETDEL en todos los artefactos; session_code puente de 60s; dummy hash y 202 convergente; tokens jamás en URLs finales.

### 8.5. PSD2 SCA RTS (Independencia de Factores)
*   **Requisito:** Los factores deben ser independientes; la posesión no debe derivar del conocimiento.
*   **Implementación UC-16:** Passkey = posesión criptográfica + inherencia (biometría local UV); magic link = posesión de buzón (canal independiente del password, que no existe en este flujo).

### 8.6. GDPR Art. 5/32 / SOC 2 CC6.1-CC7.2 / ISO 27001 A.5.18-A.8.15
*   **Implementación UC-16:** Emails con contenido mínimo (enlace/código sin PII adicional); eventos con `emailHash` en lugar de email crudo; tokens almacenados por hash; zeroización implícita por TTL; señales de abuso inmutables vía Outbox/KAGE.

---

## 9. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | La matriz de colapso implementa INV-105 a INV-111 y RN-36 a RN-39; las métricas de funnel y detección monitorean los escenarios 4.1-4.10. |
| `07_Sequence_Diagram.md` | Los spans por canal y el presupuesto de timeouts (§8 de ese doc) son la base de los SLOs y traces de este documento. |
| `08_Final_Class_Design.md` | `PasswordlessController.Map`, `PasswordlessMetrics`, `PasswordlessTimeouts` y los adaptadores Redis/KAGAMI emiten exactamente las métricas y respetan los breakers definidos aquí. |
| `10_UC_Specific_Data_Model.md` | Los keyspaces `passwordless:*` y la tabla `passkey_credentials` con CAS condicional soportan las métricas y runbooks. |
| `11_UC_Specific_Security_Threats.md` | Las alertas de clonación, replay, stuffing y funnel collapse operacionalizan esas amenazas. |
| `12_UC_Specific_Performance_Benchmarks.md` | Los SLOs por canal y los escenarios de authenticators virtuales se validan con los scripts k6 definidos aquí. |

---

## 10. Cierre del Paquete de Diseño Funcional (UC-16)

Con este documento, completamos los **9 archivos del Paquete de Diseño Profundo Funcional (Tier 1)** para el UC-16:

1. ✅ `01_Requirements_Specification.md` - Tres canales con clasificación de assurance, dos pasos anti-prefetch y emisión silenciosa.
2. ✅ `02_Use_Case_Diagram.md` - Siete casos núcleo, cuatro validaciones obligatorias y seis extensiones.
3. ✅ `03_Activity_Diagram.md` - Cuatro flujos con swimlanes: ceremonia, magic link, OTP y enrollment.
4. ✅ `04_Initial_Domain_Model.md` - ARs por canal con validación pura, supersede y proveniencia única.
5. ✅ `05_State_Machine_Diagram.md` - Cinco máquinas: estado implícito del enlace, OTP con cuatro terminales, passkey con clonación, session code y convergencia silenciosa.
6. ✅ `06_Robustness_Diagram.md` - Colaboración ICONIX con Peek/GETDEL separados y puerta única de assurance.
7. ✅ `07_Sequence_Diagram.md` - Escalera de guardas, gate pre-consumo, supersede y presupuesto de timeouts.
8. ✅ `08_Final_Class_Design.md` - Typed Result de diez casos con ZIL compilable, CAS de counter y fail-secure asimétrico.
9. ✅ `09_Modern_Artifacts.md` - OpenAPI de nueve endpoints, funnel observable, resiliencia por dependencia, testing con adversario simulado y runbooks.

### 10.1. Resumen de lo Logrado

Hemos diseñado la autenticación sin contraseña como **tres canales honestos sobre una sola maquinaria**:
- **Assurance sin ficciones:** Passkeys emiten AAL2/3 directo; los canales OOB emiten AAL1 y pausan cuando la política exige más — la clasificación es mecánica, no promocional.
- **Anti-enumeración estructural:** Emisión silenciosa con convergencia pública, rate silencioso y dummy hash: el atacante no distingue existencia de identidad en ninguna respuesta, timing o límite.
- **Consumo como átomo:** Dos pasos para el magic link (GET puro, POST GETDEL), supersede para OTP, single-use para challenges y session codes — cada artefacto tiene exactamente una muerte.
- **Ataques como señales:** Clonación, replay, stuffing y fatiga son métricas de primer nivel con alertas y runbooks propios.
- **Degradación sin traición:** Ningún fallo de dependencia produce emisión insuficiente, consumo espurio o fuga de estado; el email caído es UX, no seguridad.

### 10.2. Próximos Pasos (Documentos Técnicos Específicos)

Para el UC-16 aún faltan los 3 documentos de la Fase 2:

1. **`10_UC_Specific_Data_Model.md`** - Tabla `passkey_credentials` con CAS, keyspaces Redis de magic link/OTP/challenges/codes/rate, y columnas de proveniencia passwordless en `refresh_tokens`.
2. **`11_UC_Specific_Security_Threats.md`** - Threat modeling de intercepción de enlaces, prefetch abuse, stuffing, clonación de passkeys, OTP brute-force y abuso de handoff.
3. **`12_UC_Specific_Performance_Benchmarks.md`** - SLAs por canal, funnel de magic link bajo carga, costo de la escalera de guardas, y benchmarks de clonación/stuffing bajo fuego.

---
*Fin del Documento 09_Modern_Artifacts para UC-16*