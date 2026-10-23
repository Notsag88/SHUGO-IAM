# UC-14: Autenticar Federada (SSO) - Artefactos Modernos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-14 - Autenticar Federada (SSO - OIDC/OAuth2 + SAML 2.0) |
| **Documento:** | 09_Modern_Artifacts.md |
| **Metodología:** | API-First + Observability (OpenTelemetry) + Resilience Patterns + Contract Testing (WireMock) + DevSecOps |
| **Estándares:** | NIST SP 800-63C (Federation & Assertions), OIDC Core 1.0, OIDC Back-Channel Logout 1.0, RFC 6749/7636 (PKCE), SAML 2.0 Core/Bindings/Profiles, OWASP SAML & OAuth2 Cheat Sheets, GDPR Art. 5(1)(c) & 32, SOC 2 CC6.1/CC7.2, ISO 27001:2022 A.5.18/A.8.15 |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 22/10/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 22/10/2026 | Gastón Ruiz | **Creación inicial.** Definición de los artefactos modernos para UC-14: contratos OpenAPI 3.1 de los cuatro endpoints de federación (`/start`, `/callback`, `/token`, `/backchannel-logout`) con matriz de colapso Zero Info Leakage por `FederateStage`, catálogo de métricas OpenTelemetry (confianza de conexión, cuarentenas, replays, JIT, step-ups, back-channel), patrones de resiliencia con presupuesto de timeouts y cero reintentos, Feature Flags de política por conexión, estrategia de Contract Testing con IdP simulado (WireMock) incluyendo rotación de claves y metadata envenenada, chaos engineering de IdP degradado, y runbooks de incidente por estado de confianza. | Cierra el paquete de diseño Tier 1 funcional del UC-14, proporcionando todos los artefactos necesarios para implementar, operar, monitorear y auditar a SHUGO como Relying Party / Service Provider de grado enterprise frente a IdPs heterogéneos. |

---

## 1. Introducción y Propósito

Este documento define los **Artefactos Modernos** necesarios para operar el UC-14 (Autenticar Federada) en producción. A diferencia de los UCs de autenticación local, el UC-14 introduce una dependencia externa crítica y heterogénea (el IdP), y una superficie de configuración por tenant (conexiones, mapeos, políticas). Sus artefactos deben satisfacer cuatro demandas simultáneas:

1. **Contratos estables y opacos:** Cuatro endpoints con semánticas distintas (inicio front-channel, callback anónimo atacable, canje back-channel, y logout server-to-server), todos colapsando sus diez stages de fallo a respuestas genéricas que no filtren estado de tenants, conexiones ni identidades.
2. **Observabilidad de la confianza:** Las métricas no solo miden latencia y volumen: exponen el **estado operacional de cada conexión** (TRUSTED / ROLLOVER_PROBE / SUSPECT / DEGRADED_IDP), haciendo visible cuarentenas, rollovers confirmados y replays rechazados.
3. **Resiliencia asimétrica por dependencia:** IdP (fail-secure con fallback por política), Redis (fail-secure absoluto), Risk Engine (fail-mode configurable), metadata (refetch único con cuarentena). Cada dependencia degrada de forma distinta y documentada.
4. **Verificación continua contra IdPs reales:** Contract Testing con IdP simulado (WireMock) que reproduce rotación de claves, metadata envenenada y artefactos malformados, porque en federación **el proveedor externo es parte del sistema bajo prueba**.

---

## 2. Contratos OpenAPI Completos

### 2.1. Especificación OpenAPI 3.1

```yaml
openapi: 3.1.0
info:
  title: SHUGO KAGI - Federated Authentication API (SSO)
  description: |
    Endpoints de autenticación federada donde SHUGO actúa como
    Relying Party (OIDC) / Service Provider (SAML 2.0).
    
    ## Características Clave
    - Authorization Code + PKCE S256 obligatorio (OIDC) y Web Browser SSO (SAML).
    - Transaction State server-side single-use (state + nonce + verifier, TTL 10 min).
    - Account Linking / JIT Provisioning gobernados por política de conexión.
    - Assurancia derivada de acr/amr; step-up MFA local si es insuficiente.
    - Sesiones locales con proveniencia federada (auth_method, idp, idp_session_id).
    - Back-Channel Logout (OIDC) y SAML SLO con revocación de familias locales.
    
    ## Seguridad
    - Todos los stages de fallo retornan `federate_failed` genérico (Zero Info Leakage).
    - Los tokens NUNCA viajan en URLs de redirección: solo un federate_code de 60s.
    - IdP-Initiated SAML: default deny; habilitable por conexión con controles reforzados.
  version: 1.0.0

servers:
  - url: https://auth.shugo.io/v1
    description: Producción

paths:
  /auth/federate/start:
    post:
      operationId: startFederatedAuth
      summary: Iniciar autenticación federada (SP-initiated)
      tags: [Federation]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/StartRequest' }
      responses:
        '302':
          description: |
            Redirect al authorization_endpoint del IdP (OIDC con PKCE)
            o auto-POST de SAMLRequest (SAML). El Location nunca contiene tokens.
          headers:
            Location:
              schema: { type: string }
            Cache-Control:
              schema: { type: string }
              description: "no-store"
        '400':
          description: Conexión no resuelta, redirect_uri fuera de whitelist o federación deshabilitada
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ErrorResponse' }

  /auth/federate/callback:
    get:
      operationId: oidcCallback
      summary: Callback OIDC (authorization code + state)
      tags: [Federation]
      parameters:
        - { name: code,  in: query, required: true,  schema: { type: string } }
        - { name: state, in: query, required: true,  schema: { type: string } }
        - { name: error, in: query, required: false, schema: { type: string } }
      responses:
        '302':
          description: |
            Redirect al redirect_uri del cliente con federate_code single-use (60s)
            y client_state original. O redirect de step-up (mfa_required).
        '400':
          description: federate_failed genérico (state, validation, replay, idp_initiated, risk)
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ErrorResponse' }
        '403':
          description: linking_requires_verification (mensaje neutro anti-oracle)
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ErrorResponse' }
        '503':
          description: idp_unavailable (fail-secure; Retry-After)
          headers:
            Retry-After: { schema: { type: integer } }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ErrorResponse' }
    post:
      operationId: samlAcs
      summary: Assertion Consumer Service SAML 2.0 (HTTP-POST binding)
      tags: [Federation]
      requestBody:
        required: true
        content:
          application/x-www-form-urlencoded:
            schema:
              type: object
              required: [SAMLResponse]
              properties:
                SAMLResponse: { type: string, description: "Assertion base64" }
                RelayState:   { type: string, description: "transaction_id + client_state" }
      responses:
        '302': { description: "Idem GET: federate_code o step-up" }
        '400': { description: "federate_failed genérico" }
        '403': { description: "linking_requires_verification" }
        '503': { description: "idp_unavailable" }

  /auth/federate/token:
    post:
      operationId: redeemFederateCode
      summary: Canjear federate_code por tokens de sesión federada
      tags: [Federation]
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: '#/components/schemas/RedeemRequest' }
      responses:
        '200':
          description: Tokens de la familia federada
          headers:
            Cache-Control: { schema: { type: string }, description: "no-store" }
          content:
            application/json:
              schema: { $ref: '#/components/schemas/TokenResponse' }
        '400':
          description: federate_failed genérico (code inválido, expirado o ya canjeado)
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ErrorResponse' }

  /auth/federate/backchannel-logout:
    post:
      operationId: backChannelLogout
      summary: Back-Channel Logout (OIDC logout_token / SAML LogoutRequest)
      description: |
        Endpoint server-to-server. La autenticación del caller ES la firma
        del artefacto contra la metadata pineada de la conexión.
        Contrato OIDC Back-Channel: 200 OK siempre tras validación exitosa,
        incluso si no existen familias locales (idempotencia).
      tags: [Federation]
      requestBody:
        required: true
        content:
          application/x-www-form-urlencoded:
            schema:
              type: object
              properties:
                logout_token: { type: string, description: "JWT firmado por el IdP (OIDC)" }
                SAMLRequest:  { type: string, description: "LogoutRequest firmado (SAML SLO)" }
      responses:
        '200': { description: "Ack contractual (con o sin familias revocadas)" }
        '400': { description: "Artefacto inválido (firma/iss/aud); el IdP reintentará" }

components:
  schemas:
    StartRequest:
      type: object
      properties:
        email:         { type: string, format: email, description: "Home Realm Discovery" }
        connection_id: { type: string, format: uuid, description: "Alternativa explícita a email" }
        redirect_uri:  { type: string, format: uri, description: "Debe estar en la whitelist de la conexión" }
        client_state:  { type: string, description: "State opaco del cliente, devuelto en el redirect final" }
      required: [redirect_uri]

    RedeemRequest:
      type: object
      required: [code]
      properties:
        code: { type: string, description: "federate_code single-use (TTL 60s)" }

    TokenResponse:
      type: object
      properties:
        token_type:    { type: string, enum: [Bearer] }
        access_token:  { type: string }
        refresh_token: { type: string }
        expires_in:    { type: integer, example: 900 }
        session_meta:
          type: object
          properties:
            auth_method:    { type: string, enum: [federated] }
            idp:            { type: string, example: "https://khcs.okta.com" }
            idp_session_id: { type: string }
            auth_level:     { type: string, enum: [AAL1, AAL2] }

    ErrorResponse:
      type: object
      properties:
        error:
          type: object
          properties:
            code:
              type: string
              enum: [federate_failed, linking_requires_verification, idp_unavailable, invalid_request]
            message: { type: string }
```

### 2.2. Matriz de Comportamiento (Colapso Zero Info Leakage por Stage)

| `FederateOutcome` / Stage interno | HTTP | Cuerpo | Métrica (`stage` / `result`) |
| :--- | :--- | :--- | :--- |
| `RedirectToIdp` | **302** | Location al IdP | `start_total{protocol}` |
| `RedirectToClient` | **302** | Location con `federate_code` | `callback_total{result=redirect_client}` |
| `TokensIssued` | **200** | TokenResponse + `session_meta` | `callback_total{result=redeemed}` |
| `MfaStepUp` | **200** | `{ mfa_required, mfa_token }` | `stepup_total{reason}` |
| `LinkingConflict` | **403** | `linking_requires_verification` (neutro) | `linking_conflicts_total` |
| `IdpUnavailable` | **503** | `idp_unavailable` + Retry-After | `callback_total{result=idp_unavailable}` |
| `Failed(RealmResolution)` | **400** | `federate_failed` | `failed_total{stage=realm}` |
| `Failed(State)` | **400** | `federate_failed` | `failed_total{stage=state}` |
| `Failed(Validation)` | **400** | `federate_failed` | `failed_total{stage=validation}` |
| `Failed(Replay)` | **400** | `federate_failed` | `failed_total{stage=replay}` + `replay_rejected_total` |
| `Failed(IdpInitiated)` | **400** | `federate_failed` | `failed_total{stage=idp_initiated}` + `unsolicited_total{decision=denied}` |
| `Failed(RiskRevoked)` | **400** | `federate_failed` | `failed_total{stage=risk_revoked}` |
| `Failed(Redeem)` | **400** | `federate_failed` | `failed_total{stage=redeem}` |
| `Failed(Internal)` | **503** | `service_unavailable` | `failed_total{stage=internal}` |
| Back-channel válido | **200** | — | `backchannel_received_total{protocol}` |
| Back-channel inválido | **400** | — | `failed_total{stage=backchannel}` |

---

## 3. Catálogo de Métricas OpenTelemetry

### 3.1. Métricas de Negocio y Federación

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagi.auth.federation.start_total` | Counter | operations | Inicios de vuelo federado | `tenant_id`, `connection_id`, `protocol` |
| `kagi.auth.federation.callback_total` | Counter | operations | Callbacks procesados por resultado | `tenant_id`, `protocol`, `result` |
| `kagi.auth.federation.failed_total` | Counter | operations | Rechazos por stage interno | `tenant_id`, `stage`, `protocol` |
| `kagi.auth.federation.jit_provisioned_total` | Counter | identities | Identidades creadas vía JIT | `tenant_id`, `connection_id` |
| `kagi.auth.federation.account_linked_total` | Counter | links | Links creados por método | `tenant_id`, `method` (email_verified, admin, consent) |
| `kagi.auth.federation.linking_conflicts_total` | Counter | conflicts | Conflictos de linking (flujo neutro 403) | `tenant_id` |
| `kagi.auth.federation.stepup_total` | Counter | stepups | Step-ups por assurance insuficiente o riesgo | `tenant_id`, `reason` (acr_insufficient, risk) |
| `kagi.auth.federation.unsolicited_total` | Counter | responses | Respuestas SAML unsolicited por decisión | `connection_id`, `decision` (denied, accepted) |
| `kagi.auth.federation.backchannel_received_total` | Counter | logouts | Back-channel logouts recibidos | `connection_id`, `protocol`, `families_terminated_bucket` |

### 3.2. Métricas de Confianza y Seguridad Operacional

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagi.auth.federation.trust_state` | Gauge | state | Estado de confianza por conexión (0=TRUSTED, 1=ROLLOVER_PROBE, 2=DEGRADED_IDP, 3=SUSPECT) | `connection_id` |
| `kagi.auth.federation.quarantine_total` | Counter | events | Cuarentenas SUSPECT activadas | `connection_id`, `reason` |
| `kagi.auth.federation.rollover_confirmed_total` | Counter | rollovers | Rollovers de clave confirmados por refetch | `connection_id` |
| `kagi.auth.federation.jwks_refetch_total` | Counter | refetches | Refetches de metadata/JWKS | `connection_id`, `outcome` (confirmed, quarantined, unreachable) |
| `kagi.auth.federation.jwks_cache_hit_ratio` | Gauge | ratio | Hit ratio de cache de metadata (SLO > 0.99) | `connection_id` |
| `kagi.auth.federation.replay_rejected_total` | Counter | replays | Artefactos rechazados por replay cache | `protocol`, `replay_kind` (nonce, assertion, code) |
| `kagi.auth.federation.risk_timeouts_total` | Counter | timeouts | Timeouts del Risk Engine con fail-mode aplicado | `tenant_id`, `fail_mode` |

### 3.3. Métricas de Rendimiento

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagi.auth.federation.duration` | Histogram | ms | Latencia local por fase (excluye front-channel) | `phase` (start, callback_local, redeem, backchannel) |
| `kagi.auth.federation.idp_token_exchange_duration` | Histogram | ms | Round-trip back-channel al token endpoint del IdP | `connection_id` |
| `kagi.auth.federation.id_token_validate_duration` | Histogram | ms | Validación de firma/claims (incluye JWKS resolve) | `protocol` |
| `kagi.auth.federation.jit_provision_duration` | Histogram | ms | Gateway JIT hacia KAGAMI (identidad+link atómicos) | `tenant_id` |
| `kagi.auth.federation.user_journey_duration` | Histogram | min | Duración del viaje front-channel completo (start→callback), correlacionado por `transaction_id` | `protocol` |

### 3.4. Distributed Tracing (Spans por Canal)

```text
FRONT-CHANNEL (sin SLO: mediado por humano/IdP; span de journey por transaction_id)
  federate.start (302) ──► [browser → IdP → browser] ──► federate.callback_in

BACK-CHANNEL CALLBACK (Root Span, SLO local < 200ms p95)
POST /v1/auth/federate/callback [Total local: 85ms]
├── redis.tx.consume [2ms]             (GETDEL single-use)
├── idp.token_exchange [45ms]          (único round-trip IdP observado; timeout 5s)
├── trust.jwks.resolve [3ms]           (cache hit; refetch añadiría ~200ms + sonda)
├── oidc.id_token.validate [1ms]       (firma + iss/aud/exp/nonce + allowlist alg)
├── redis.replay.check_write [4ms]     (post-validación, fail-secure)
├── kagami.link_lookup [8ms]
├── kagami.jit_gateway [120ms]         (solo si NeedsJit; TX identidad+link en KAGAMI)
├── risk.evaluate [30ms]               (timeout 100ms, fail-mode por política)
├── kagami.family_create [12ms]        (CreateRoot con proveniencia; misma TX que outbox)
├── redis.federate_code.set [2ms]
└── outbox.insert [6ms]

BACK-CHANNEL LOGOUT (Root Span independiente)
POST /v1/auth/federate/backchannel-logout [Total: 25ms]
├── trust.metadata.validate [3ms]
├── kagami.family_lookup_by_sid [7ms]
├── uc12.family_revoke [12ms]          (IDP_SESSION_TERMINATED + blacklist JTIs)
└── outbox.insert [3ms]
```

### 3.5. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Connection Quarantined** | `trust_state == 3` (SUSPECT) | **Crítica** | Ejecutar Runbook 1: posible inyección de claves o IdP comprometido. Vuelos nuevos rechazados; familias existentes intactas. |
| **Replay Rejected Spike** | `rate(replay_rejected_total[5m]) > 5` | **Crítica** | Artefactos IdP siendo replaysados: investigar fuga de tokens/codes en cliente, correlacionar IPs, evaluar revocación de familias afectadas (UC-21). |
| **IdP Token Exchange Degraded** | `idp_token_exchange_duration_p95 > 2s` por 5 min | Alta | IdP degradado: anticipar `idp_unavailable`; revisar política `local_fallback` del tenant y comunicar al owner del IdP. |
| **JWKS Cache Hit Ratio Low** | `jwks_cache_hit_ratio < 0.99` por 10 min | Alta | Refetches anormales: posible sondeo de kids falsos (ataque) o misconfig de cache. Revisar `jwks_refetch_total{outcome}`. |
| **JIT Provisioning Storm** | `rate(jit_provisioned_total[10m]) > 10x baseline` | Alta | Cambio masivo de grupos/dominios en IdP o misconfig de mapeo. Auditar identidades creadas; considerar `jit_enabled=false` temporal. |
| **Back-Channel Logout Storm** | `rate(backchannel_received_total[5m]) > 20x baseline` | Alta | Incidente o administración masiva en el IdP (ej. revocación global). Verificar que las familias terminadas coincidan y notificar a usuarios afectados. |
| **Unsolicited Denied Spike** | `rate(unsolicited_total{decision="denied"}[10m]) > 0` | Media | IdP mal configurado enviando flujos IdP-initiated, o sondeo externo. Validar configuración con owner del IdP. |
| **Federation 503 Storm** | `rate(callback_total{result="idp_unavailable"}[2m]) > 0` | **Crítica** | IdP caído: usuarios no pueden federar. Ejecutar Runbook 2; evaluar fallback por política; sesiones vigentes intactas. |

---

## 4. Patrones de Resiliencia

### 4.1. Presupuesto de Timeouts y Cero Reintentos (Política Global)

```csharp
public sealed record FederationTimeouts
{
    public TimeSpan IdpTokenExchange { get; init; } = TimeSpan.FromSeconds(5);
    public TimeSpan MetadataRefetch  { get; init; } = TimeSpan.FromSeconds(2);
    public TimeSpan RedisOperation   { get; init; } = TimeSpan.FromMilliseconds(50);
    public TimeSpan KagamiOperation  { get; init; } = TimeSpan.FromMilliseconds(200);
    public TimeSpan RiskEvaluation   { get; init; } = TimeSpan.FromMilliseconds(100);
    public int Retries               { get; init; } = 0;  // Política global inmutable
}
```

**Justificación de `Retries = 0`:** En federación, las operaciones críticas (consumo de tx, canje de code, emisión de familia, write de replay cache) son **no idempotentes o single-use**. Un retry server-side crearía efectos duplicados o consumiría ventanas anti-replay legítimas. El reintento correcto vive en el cliente (backoff con jitter ante 503) o en el IdP (back-channel).

### 4.2. Circuit Breakers Diferenciados por Dependencia

| Dependencia | Breaker | Comportamiento al Abrir | Recuperación |
| :--- | :--- | :--- | :--- |
| **IdP HTTP (por conexión)** | 5 fallos / 60s open | `IdpUnavailable` (503) + transición a `DEGRADED_IDP`; fallback local solo si `local_fallback=true` | Health probe cada 30s; medio-open con un vuelo de prueba |
| **Redis (Tx/Replay/Code)** | 3 fallos / 30s open | **Fail-secure absoluto:** callbacks y redeems rechazados (sin tx no hay vuelo; sin replay no hay garantía) | Automática al cerrar; sin estado residual que reconciliar (las tx vivas simplemente expiran) |
| **Risk Engine (UC-17)** | Timeout 100ms | Fail-mode por política: `open` (continúa con warning) o `stepup` (degrada a MFA) | Inmediata al siguiente evaluación exitosa |
| **Metadata/JWKS** | No breaker: sonda única | `ROLLOVER_PROBE` → `SUSPECT` (cuarentena) o `DEGRADED_IDP` | Cuarentena: manual (re-pin UC-25). Degradada: automática |

### 4.3. Cuarentena como Estado Operativo de Primer Nivel

La cuarentena (`SUSPECT`) no es un error transitorio: es un **estado de conexión persistido** (`federate:trust:{connection_id}`) con efectos explícitos:
- `/start` de la conexión rechaza con `federate_failed` (sin revelar motivo interno).
- Callbacks rechazan sin refetch adicional.
- Back-channel logout **sigue operativo** con la metadata cacheada vigente (terminar sesiones nunca debe bloquearse por sospecha de la conexión que las creó).
- Salida exclusiva: re-pin administrativo de metadata (UC-25) con incidente cerrado (Runbook 1).

### 4.4. Idempotencia y Contratos Externos

- **Back-Channel Logout:** 200 OK siempre tras validación, con o sin familias locales. El IdP no reintenta en bucle contra un SHUGO saludable.
- **Redeem de `federate_code`:** GETDEL atómico; el segundo canje (retry de cliente tras timeout donde el primero sí ocurrió) falla genérico, pero la sesión del cliente ya está establecida con sus tokens en memoria.
- **Replay cache SET NX:** idempotente por construcción; re-registros dentro de la ventana retornan `false` y se clasifican como replay, no como error de infraestructura.

---

## 5. Feature Flags y Política por Conexión

### 5.1. Flags Globales

| Feature Flag | Descripción | Default |
| :--- | :--- | :--- |
| `kagi.auth.federation.enabled` | Kill switch global de federación | `true` |
| `kagi.auth.federation.tx_ttl_minutes` | Vida del Transaction State | `10` |
| `kagi.auth.federation.federate_code_ttl_seconds` | Vida del código de canje | `60` |
| `kagi.auth.federation.max_clock_skew_seconds` | Skew tolerado en exp/Conditions | `60` |
| `kagi.auth.federation.allowed_algorithms` | Allowlist de firma | `RS256,ES256,EdDSA` (**`none`/`HS256` no añadibles**: validador rechaza la configuración al arranque) |

### 5.2. Política por Conexión (configurada en UC-25, consumida como Snapshot)

| Parámetro | Descripción | Default |
| :--- | :--- | :--- |
| `protocol` | `OIDC` / `SAML` | — |
| `allow_idp_initiated` | Habilita SAML unsolicited con controles reforzados | `false` (RN-30) |
| `local_fallback` | Fallback a credenciales locales ante IdP caído | `false` (RN-31) |
| `jit.enabled` / `jit.allowed_domains` / `jit.allow_admin_roles` | Política de JIT Provisioning | `false` / `[]` / **`false` bloqueado salvo aprobación compliance** |
| `linking.auto_link_email_verified` | Auto-link por email verificado | `true` |
| `assurance.required_level` / `assurance.acr_mfa_map` | Assurance exigido y equivalencias acr→MFA | `AAL1` / mapa por vendor |
| `risk.policy` / `risk.fail_mode` | Decisión ante score alto y ante timeout | `stepup` / `open` |

---

## 6. Estrategia de Testing

### 6.1. Contract Testing con IdP Simulado (WireMock)

El IdP es parte del sistema bajo prueba. Cada release ejecuta la suite de contratos contra un IdP simulado que reproduce comportamientos reales y hostiles:

```json
// WireMock stub: rotación de claves legítima (kid desconocido → refetch → éxito)
{
  "request": { "method": "GET", "url": "/oidc/jwks", "queryParameters": { "refetch": { "equalTo": "1" } } },
  "response": {
    "status": 200,
    "jsonBody": { "keys": [ { "kid": "key-2026-09", "kty": "OKP", "crv": "Ed25519", "x": "..." } ] }
  }
}
```

| Caso de Contrato | Simulación | Expectativa SHUGO |
| :--- | :--- | :--- |
| Happy path OIDC | id_token válido EdDSA, nonce correcto | 302 con federate_code; familia con proveniencia |
| Happy path SAML | Assertion firmada, InResponseTo correcto, Audience OK | Ídem vía ACS |
| **Key rollover legítimo** | Primer JWKS sin el kid; refetch lo contiene | Rollover confirmado; `rollover_confirmed_total++`; vuelo continúa |
| **Metadata envenenada** | Refetch retorna JWKS con clave del atacante; firma no re-valida | **Cuarentena SUSPECT**; `quarantine_total++`; vuelo rechazado; segundo vuelo sin refetch |
| id_token con `alg=none` | Token sin firma | Rechazo stage=validation; alerta de allowlist |
| Audience mismatch | `aud` = client_id de otro tenant | Rechazo stage=validation |
| Nonce replay | Mismo nonce dos veces | Segundo rechazo stage=replay |
| SAML unsolicited con policy deny | SAMLResponse sin InResponseTo | Rechazo stage=idp_initiated; `unsolicited_total{denied}` |
| Back-channel con firma válida sin familias | logout_token correcto, sid desconocido | 200 OK idempotente |

### 6.2. Load Testing (k6) con IdP Simulado

```javascript
import http from 'k6/http';
import { check } from 'k6';
import { Trend, Rate } from 'k6/metrics';

const callbackLocal = new Trend('federation_callback_local', true);
const errors = new Rate('errors');

export const options = {
  scenarios: {
    federation_wave: {
      executor: 'ramping-arrival-rate',
      startRate: 50, timeUnit: '1s',
      preAllocatedVUs: 150, maxVUs: 900,
      stages: [
        { target: 200, duration: '2m' },
        { target: 500, duration: '5m' },   // apertura de jornada federada
        { target: 500, duration: '3m' },
        { target: 0,   duration: '2m' },
      ],
    },
  },
  thresholds: {
    federation_callback_local: ['p(95)<200', 'p(99)<400'],  // excluye front-channel
    errors: ['rate<0.005'],
    // Externos: trust_state == 0 durante toda la corrida; replay_rejected == 0
  },
};

export default function () {
  // 1) start → captura Location y transaction cookie
  const start = http.post(`${__ENV.BASE_URL}/v1/auth/federate/start`,
    JSON.stringify({ email: `load${__VU}@khcs.com`, redirect_uri: __ENV.REDIRECT }));
  // 2) IdP simulado responde code+state inmediatamente (latencia inyectada 40ms)
  const cb = mockIdpAuthorize(start.headers['Location']);
  // 3) callback → medir SOLO procesamiento local
  const t0 = Date.now();
  const res = http.get(`${__ENV.BASE_URL}/v1/auth/federate/callback?code=${cb.code}&state=${cb.state}`);
  callbackLocal.add(Date.now() - t0);
  if (!check(res, { 'callback 302': (r) => r.status === 302 })) errors.add(1);
}
```

### 6.3. Chaos Engineering: IdP Degradado y Metadata Hostil

| Inyección | Duración | Expectativa |
| :--- | :--- | :--- |
| Latencia IdP token endpoint = 6s (> timeout 5s) | 5 min | `idp_unavailable` 503 con Retry-After; `DEGRADED_IDP`; sesiones vigentes y back-channel intactos; cero familias emitidas |
| JWKS sirviendo clave del atacante (poisoning) | permanente hasta detección | Cuarentena en el primer artefacto con kid desconocido que no re-valida; `trust_state=3`; alerta crítica; **cero tokens emitidos con clave atacante** |
| Redis down | 5 min | Callbacks y redeems rechazados fail-secure; `/start` rechaza (sin tx no hay vuelo); al recuperar, sin estado residual que reconciliar |
| Risk Engine con latencia 300ms | 5 min | Tenants `fail_mode=stepup`: step-ups; tenants `open`: rotaciones con `risk_timeouts_total{open}` |

---

## 7. Deployment y Operaciones

### 7.1. Kubernetes HPA (Escalado por Fase Local, no por Front-Channel)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: kagi-federation-hpa }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: kagi-federation }
  minReplicas: 4
  maxReplicas: 24
  metrics:
    - type: Pods
      pods:
        metric: { name: kagi_auth_federation_duration_p95 }
        target: { type: AverageValue, averageValue: "200" }   # fase callback_local
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 65 } }
```

### 7.2. Runbook de Operaciones

**Runbook 1: Conexión en Cuarentena (SUSPECT)**
1. Confirmar alerta `Connection Quarantined` y leer `quarantine_total{reason}` (SECOND_SIGNATURE_FAILURE_AFTER_REFETCH / KEY_NOT_PRESENT_AFTER_REFETCH).
2. **No** re-pinear de inmediato: preservar forense (JWKS servido, artefactos con `payloadHash` en KAGE).
3. Contactar al owner del IdP: ¿hubo rotación de claves real? Comparar fingerprints de claves servidas vs publicadas por el vendor.
4. Si es rotación legítima mal servida (CDN/cache del IdP): corregir con vendor, luego re-pin vía UC-25 con incidente cerrado.
5. Si es inyección de claves: escalar a incidente de plataforma; rotar credenciales de la conexión; revisar accesos a infraestructura del IdP.
6. Verificar que `trust_state` vuelva a 0 y que `start_total` de la conexión recupere baseline.

**Runbook 2: IdP Caído (DEGRADED_IDP)**
1. Verificar `idp_token_exchange_duration` y errores TLS/5xx del IdP.
2. Comunicar a usuarios: login federado indisponible; sesiones vigentes y refresh (UC-13) **no afectados**.
3. Evaluar con owner de tenant la activación temporal de `local_fallback` (solo identidades con credencial local).
4. Al recuperar: health probe cierra breaker; monitorear pico de `/start` reprimido.

**Runbook 3: Replay Spike**
1. Correlacionar `replay_rejected_total{replay_kind}` con IPs y `transaction_id` en KAGE.
2. Si nonce/assertion replays desde una IP: bloquear en WAF y revisar fuga de codes en cliente (logs de proxy, extensiones).
3. Si replays distribuidos: asumir artefactos interceptados; revocar familias afectadas vía UC-21 y forzar re-autenticación; rotar claves de firma de SHUGO si el vector lo justifica.

**Runbook 4: JIT Provisioning Storm**
1. Verificar `jit_provisioned_total` vs baseline y cambios recientes en grupos/dominios del IdP.
2. Auditar roles asignados a las identidades provisionadas (¿algún admin vía mapping erróneo?).
3. Si misconfig: desactivar `jit.enabled`, corregir mapping en UC-25, y remediar identidades sobre-privilegiadas.
4. Si ataque (IdP comprometido emitiendo claims): cuarentena manual de la conexión + incidente.

---

## 8. Compliance y Auditoría

### 8.1. NIST SP 800-63C (Federation and Assertions)
*   **Requisito:** El RP debe validar aserciones (firma, audiencia, expiración, replay) y registrar el evento de federación con el assurance recibido.
*   **Implementación UC-14:** Validación estricta por protocolo (INV-90/91), replay cache post-validación, y `Authentication.FederatedSucceeded` con `acr/amr/auth_level` y `payloadHash` forense.

### 8.2. OIDC Core 1.0 + RFC 7636 (PKCE) + OIDC Back-Channel Logout 1.0
*   **Requisito:** nonce en id_token, PKCE para code flow, y ack 200 en back-channel logout.
*   **Implementación UC-14:** `nonce` single-use por transaction, `code_verifier` server-side, y contrato 200 idempotente del endpoint de back-channel.

### 8.3. SAML 2.0 Core / Bindings / Profiles + OWASP SAML Cheat Sheet
*   **Requisito:** Firma de assertion, validación de Destination/Audience/Recipient/Conditions, y protección contra unsigned/wrapped assertions.
*   **Implementación UC-14:** Validación XML-DSig exclusiva sobre el nodo firmado con certificado pineado, `InResponseTo` single-use, y default-deny de unsolicited (RN-30).

### 8.4. GDPR Art. 5(1)(c) y Art. 32
*   **Requisito:** Minimización de datos y seguridad del tratamiento.
*   **Implementación UC-14:** Solo claims mapeados se persisten (RN-29); el forense usa `payloadHash` sin retener PII cruda; degradaciones siempre fail-secure.

### 8.5. SOC 2 CC6.1 / CC7.2 e ISO 27001:2022 A.5.18 / A.8.15
*   **Requisito:** Control de acceso lógico federado y monitoreo de eventos de seguridad.
*   **Implementación UC-14:** Roles JIT por mapping declarativo cerrado (sin admin por defecto), cuarentenas y replays alertados al SIEM, y auditoría inmutable vía Outbox/KAGE.

---

## 9. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | La matriz de colapso implementa RN-25-equivalente de federación (stages genéricos); las métricas de JIT/linking/step-up monitorean INV-92 a INV-94. |
| `07_Sequence_Diagram.md` | Los spans por canal y el presupuesto de timeouts (§4.1) son la operacionalización de las fases y latencias del diagrama de secuencia. |
| `08_Final_Class_Design.md` | `FederationController.Map`, `ConnectionTrustManager`, `FederationMetrics` y `FederationTimeouts` emiten exactamente las métricas y respetan los breakers definidos aquí. |
| `10_UC_Specific_Data_Model.md` | Los keyspaces Redis (`federate:tx:*`, `replay:*`, `federate:code:*`, `federate:trust:*`) y las tablas de links/conexiones soportan métricas y runbooks. |
| `11_UC_Specific_Security_Threats.md` | Las alertas de cuarentena, replay y metadata poisoning operacionalizan las mitigaciones de key injection, assertion replay y IdP impersonation. |
| `12_UC_Specific_Performance_Benchmarks.md` | Los SLOs de `callback_local` y `idp_token_exchange` se validan con los escenarios k6 y chaos definidos aquí. |

---

## 10. Cierre del Paquete de Diseño Funcional (UC-14)

Con este documento, completamos los **9 archivos del Paquete de Diseño Profundo Funcional (Tier 1)** para el UC-14:

1. ✅ `01_Requirements_Specification.md` - Federación OIDC/SAML con JIT, linking, proveniencia y back-channel logout.
2. ✅ `02_Use_Case_Diagram.md` - Seis casos núcleo, cuatro validaciones obligatorias y seis extensiones de política.
3. ✅ `03_Activity_Diagram.md` - Flujo SP-initiated con swimlanes y flujo back-channel server-to-server.
4. ✅ `04_Initial_Domain_Model.md` - `FederationTransaction`, `IdpArtifact` canónico, políticas como VOs y seis Domain Services.
5. ✅ `05_State_Machine_Diagram.md` - Máquinas de transacción, confianza de conexión y federate_code.
6. ✅ `06_Robustness_Diagram.md` - Colaboración ICONIX con gatekeeper único de metadata y punto único de emisión.
7. ✅ `07_Sequence_Diagram.md` - Orden temporal con canales front/back diferenciados y presupuesto de timeouts.
8. ✅ `08_Final_Class_Design.md` - Strategy de protocolos, Typed Result con colapso ZIL compilable y gatekeeper CAS.
9. ✅ `09_Modern_Artifacts.md` - OpenAPI de 4 endpoints, métricas de confianza, resiliencia asimétrica, contract testing y runbooks.

### 10.1. Resumen de lo Logrado

Hemos diseñado a SHUGO como **Relying Party / Service Provider de grado enterprise**:
- **Confianza delimitada y observable:** El IdP autentica; SHUGO autoriza, sesiona y audita. El estado de confianza de cada conexión es una métrica de primer nivel con cuarentena como estado operativo.
- **Un artefacto canónico, dos protocolos:** OIDC y SAML convergen en `IdpArtifact`; las reglas comunes de seguridad existen una sola vez.
- **Federación segura por orden:** consumo single-use pre-round-trip, replay post-validación fail-secure, step-up pre-familia, tokens fuera de URLs.
- **Privilegios cerrados:** JIT con mapping declarativo sin roles admin, linking por email verificado con flujo de conflicto neutro.
- **Círculo completo de sesiones:** proveniencia federada en la familia local y back-channel logout que la termina por `idp_session_id`.
- **Verificación continua:** Contract Testing con IdP simulado que reproduce rotación de claves y metadata envenenada en cada release.

### 10.2. Próximos Pasos (Documentos Técnicos Específicos)

Para el UC-14 aún faltan los 3 documentos de la Fase 2:

1. **`10_UC_Specific_Data_Model.md`** - Tablas `federated_connections` y `federated_identity_links`, keyspaces Redis de federación, índices de proveniencia en `refresh_tokens` y retención forense.
2. **`11_UC_Specific_Security_Threats.md`** - Threat modeling de assertion wrapping, key injection vía JWKS, open-redirect, JIT privilege escalation, token binding confusion y IdP impersonation.
3. **`12_UC_Specific_Performance_Benchmarks.md`** - SLAs de fases locales, capacidad de Tx State bajo ola federada, costo de validación XML-DSig vs JWT, y benchmarks de cuarentena/recovery.

---
*Fin del Documento 09_Modern_Artifacts para UC-14*