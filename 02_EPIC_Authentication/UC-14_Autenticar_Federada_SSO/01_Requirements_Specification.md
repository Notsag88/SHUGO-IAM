# UC-14: Autenticar Federada (SSO) - Especificación de Requisitos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-14 - Autenticar Federada (SSO - OIDC/OAuth2 + SAML 2.0) |
| **Documento:** | 01_Requirements_Specification.md |
| **Metodología:** | ICONIX (Deep Dive) + DDD + BDD + Federation Security (NIST SP 800-63C) |
| **Estándares:** | NIST SP 800-63C (Federation & Assertions), NIST SP 800-63B (AAL), OAuth 2.0 (RFC 6749), OIDC Core 1.0, RFC 7636 (PKCE), RFC 7009/Back-Channel Logout, SAML 2.0 Core/Bindings/Profiles, OWASP SAML & OAuth2 Cheat Sheets, GDPR Art. 5 & 32, SOC 2 CC6.1 |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 22/10/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 22/10/2026 | Gastón Ruiz | **Creación inicial.** Definición completa de la especificación analítica para la Autenticación Federada (SSO) en el motor KAGI. Cubre OIDC Authorization Code + PKCE y SAML 2.0 Web Browser SSO (SP-initiated e IdP-initiated con política explícita), validación estricta de assertions y tokens de IdP (firma, iss/aud/exp, nonce, InResponseTo, replay cache), Home Realm Discovery, Account Linking con política de email verificado, JIT Provisioning con mapeo de roles desde grupos del IdP, emisión de sesiones locales con proveniencia federada (auth_method, idp, acr/amr), y propagación de logout vía Back-Channel Logout y SAML SLO. | Documento base para el paquete de diseño Tier 1 del UC-14. Es el caso de uso que convierte a SHUGO en un **Relying Party / Service Provider de grado enterprise**, delegando la autenticación primario en IdPs externos (Okta, Entra ID, Google, GitHub) sin ceder el control de la sesión local, la autorización ni la auditoría. |

---

## 1. Introducción y Alcance

Este documento detalla la especificación de requisitos para el Caso de Uso **UC-14**, que representa la operación de **Autenticación Federada (SSO)** en el motor **KAGI** de SHUGO.

En entornos enterprise, la contraseña local es la excepción, no la regla: los empleados autentican contra el IdP corporativo (Okta, Microsoft Entra ID, Ping, ADFS) y SHUGO debe **confiar correctamente** en esas aserciones sin convertirse en un eslabón débil de la federación. El UC-14 modela a SHUGO como **Relying Party (OIDC) / Service Provider (SAML)** con las siguientes responsabilidades intransferibles:

1. **Validar criptográfica y semánticamente** todo artefacto del IdP (id_token, SAML Assertion) contra metadata pineada, con protección anti-replay y anti-CSRF.
2. **Traducir identidad federada a identidad local** mediante Account Linking o JIT Provisioning, respetando políticas de tenant (email verificado, mapeo de roles, conflictos).
3. **Emitir sesiones locales con proveniencia:** la familia de tokens de SHUGO registra `auth_method=federated`, el IdP de origen, y el nivel de assurance heredado (acr/amr), de modo que KIZUNA y UC-17 puedan exigir step-up según la confianza del IdP.
4. **Propagar el ciclo de vida de sesión del IdP:** Back-Channel Logout (OIDC) y Single Logout (SAML) terminan las familias locales correspondientes, cerrando el círculo de federación de sesiones.

### 1.1. Importancia Estratégica

El UC-14 es **el caso de uso de adopción enterprise**: sin SSO, un tenant de 50.000 empleados es inviable. Pero la federación multiplica la superficie de ataque:

- **El IdP se convierte en raíz de confianza:** una validación laxa (firmas no verificadas, audience ignorado, replay de assertions) compromete **todos** los tenants federados.
- **Los flujos de redirección son vectores de open-redirect y CSRF:** state, nonce, PKCE y whitelist de redirect_uris no son opcionales.
- **El JIT Provisioning es creación de identidades desde fuera:** un mapeo de roles mal configurado puede convertir un grupo de IdP en privilegios SHUGO no intencionados.
- **La sesión local debe sobrevivir independientemente del IdP:** una caída del IdP no debe tumbar sesiones vigentes, y un logout en el IdP **sí** debe terminarlas.

### 1.2. Relación con Otros Casos de Uso

| UC | Relación | Descripción |
| :--- | :--- | :--- |
| **UC-11 (Login Local)** | Complementario / Fallback | Mismo mecanismo de emisión de familia de tokens (`CreateRoot`). Política de fallback a credenciales locales ante caída de IdP (default: deny). |
| **UC-12 (Logout)** | Consumidor / Extensión | Back-Channel Logout y SAML SLO invocan la revocación de familia de UC-12 con motivo `IDP_SESSION_TERMINATED`. El logout RP-initiated de UC-12 propaga al `end_session_endpoint` / SLO del IdP. |
| **UC-13 (Refresh)** | Consumidor | Las familias federadas rotan idénticamente; la vida absoluta aplica igual (re-autenticación contra el IdP al expirar). |
| **UC-01 (Provisionar)** | Consumidor interno | El JIT Provisioning ejecuta un comando interno de provisioning con atributos mapeados desde claims/grupos del IdP. |
| **UC-17 (Risk-Based)** | Consumidor | El delta de contexto y el `acr` del IdP alimentan la evaluación de riesgo; un `acr` bajo puede forzar step-up local. |
| **UC-24/UC-25 (Tenant Mgmt)** | Proveedor de configuración | Las conexiones de IdP (metadata, client_id, mapeo de claims, políticas de linking/JIT) se administran en Tenant Management. |
| **UC-15 (MFA)** | Integración | Si el `acr` del IdP no satisface la política del tenant, se exige step-up MFA local tras la federación. |

### 1.3. Objetivos de Rendimiento

| Métrica | Objetivo | Justificación |
| :--- | :--- | :--- |
| **Latencia `/federate/start` (p95)** | < 50 ms | Solo local: resolución de conexión + generación de state/nonce/PKCE. |
| **Latencia de procesamiento de callback (p95)** | < 200 ms | Validación de token/assertion + linking + creación de familia. Excluye round-trips del IdP (fuera de control). |
| **Cache hit ratio de JWKS/metadata** | > 99% | Discovery y metadata cacheadas con TTL + refetch on unknown kid. |
| **Latencia JIT Provisioning (p95)** | < 500 ms | Creación de identidad + link + mapeo de roles en una transacción. |
| **Disponibilidad del procesamiento local** | 99.99% | La dependencia del IdP se aísla: sesiones vigentes no requieren al IdP. |

---

## 2. Descripción Analítica (ICONIX Clásico)

| Campo | Detalle |
| :--- | :--- |
| **ID Y NOMBRE:** | UC-14 - Autenticar Federada (SSO - OIDC/OAuth2 + SAML 2.0) |
| **ESTADO:** | Preliminar |
| **DESCRIPCIÓN:** | El usuario inicia sesión mediante un Identity Provider externo confiado. SHUGO (como RP/SP) construye una request de autorización firmada/estado-protegida, redirige al IdP, recibe el artefacto de regreso (authorization code + id_token, o SAML Response), lo valida criptográfica y semánticamente contra la metadata pineada de la conexión, resuelve la identidad local mediante Account Linking o JIT Provisioning según política del tenant, y emite una familia de tokens local con proveniencia federada (auth_method, idp, acr/amr, idp_session_id). Los ciclos de logout del IdP se propagan a las familias locales vía Back-Channel Logout / SAML SLO. |
| **ACTOR PRINCIPAL:** | Usuario (Resource Owner) autenticándose contra su IdP corporativo o social. |
| **ACTOR SECUNDARIO:** | Identity Provider externo (Okta, Entra ID, Google, GitHub, IdP SAML), API Gateway, KAGAMI (Identity Repo), Redis (state/nonce/replay cache), KAGE (Audit), TORII (Webhooks). |
| **PRECONDICIONES:** | 1. El tenant tiene al menos una conexión de IdP habilitada y con metadata válida (configurada en UC-25).<br>2. El redirect_uri del cliente está en la whitelist de la conexión.<br>3. La metadata/JWKS del IdP es alcanzable o está en cache vigente. |
| **PUNTOS DE EXTENSIÓN:** | JIT Provisioning en primer login; Account Linking por email verificado; step-up MFA si el `acr` del IdP no satisface la política; fallback a login local ante caída de IdP (política default deny); IdP-initiated SSO (SAML) solo si la política lo habilita explícitamente. |
| **CONDICIÓN:** | El usuario posee una familia de tokens local con proveniencia federada; el IdP nunca recibe credenciales de SHUGO ni tokens de sesión local. |

### 2.1. Escenario Principal (Happy Path) - OIDC SP-Initiated con JIT Provisioning

1. El usuario selecciona "Continuar con {IdP}" o ingresa su email corporativo (Home Realm Discovery).
2. El cliente envía `POST /auth/federate/start` con `{ tenant_id | email, connection_id?, redirect_uri, client_state }`.
3. KAGI resuelve tenant y conexión (por `connection_id` o por dominio del email → home realm), y valida `redirect_uri` contra la whitelist de la conexión.
4. KAGI genera `state`, `nonce` y `code_verifier` (PKCE S256); persiste el **transaction state** en Redis (`federate:tx:{transaction_id}`) con TTL 10 min: `{state, nonce, code_verifier, redirect_uri, tenant_id, connection_id, client_state}`.
5. KAGI responde `302` al `authorization_endpoint` del IdP con `response_type=code`, `client_id`, `redirect_uri` (callback de KAGI), `scope=openid profile email groups`, `state`, `nonce`, `code_challenge=S256(verifier)`, `acr_values` según política.
6. El usuario autentica en el IdP (fuera del alcance y control de SHUGO).
7. El IdP redirige a `GET /auth/federate/callback?code=...&state=...`.
8. KAGI valida `state` contra el transaction state (consumo single-use) y recupera el contexto.
9. KAGI intercambia el `code` en el `token_endpoint` del IdP (back-channel TLS) presentando el `code_verifier` (PKCE).
10. KAGI valida el `id_token`: firma contra JWKS pineado (refetch on unknown kid), `iss` exacto, `aud` = client_id, `exp`/`iat` con clock skew ≤ 60s, `nonce` igual al emitido, y algoritmo permitido (RS256/ES256; **nunca** `none` ni HS256 en conexión asimétrica).
11. KAGI extrae claims mapeados: `sub`, `email`, `email_verified`, `name`, `groups`, `acr`, `amr`, `sid` (sesión IdP).
12. **Resolución de identidad:** busca `federated_identity_link` por `(issuer, sub)`.
    - Si existe: identidad local resuelta.
    - Si no existe: aplica política de linking (email verificado → link automático) o **JIT Provisioning**: crea identidad en KAGAMI con atributos mapeados, crea el link, y asigna roles según mapeo de grupos de la conexión.
13. KAGI crea la familia de tokens local (`CreateRoot`) con claims de proveniencia: `auth_method=federated`, `idp={issuer}`, `idp_session_id={sid}`, `acr`, `amr` heredados; `auth_level` derivado del assurance del IdP (acr con MFA → AAL2).
14. KAGI emite un `federate_code` de un solo uso (60s) vinculado al transaction y redirige al `redirect_uri` del cliente: `302 {redirect_uri}?code={federate_code}&state={client_state}`.
15. El cliente intercambia: `POST /auth/federate/token { code: federate_code }` → recibe Access Token + Refresh Token (familia federada).
16. Se publican eventos: `Authentication.FederatedSucceeded`, `Identity.JitProvisioned` (si aplicó), `Account.Linked` (si aplicó).

### 2.2. Escenarios Alternativos

**2.2.1 SAML 2.0 SP-Initiated (Web Browser SSO, HTTP-POST Binding)**
1-4. Análogos, pero el transaction state incluye el `AuthnRequestID` y el destino SSO URL del metadata.
5. KAGI responde `302`/auto-POST con `SAMLRequest` (AuthnRequest deflate+base64) al `SingleSignOnService` del IdP.
7. El IdP POSTea `SAMLResponse` a `/auth/federate/callback` (ACS).
8-10. KAGI valida: firma XML contra certificado del metadata (Signature exclusive-c14n), `Destination` = ACS URL, `InResponseTo` = AuthnRequestID almacenado (single-use), `AudienceRestriction` = SP EntityID, `SubjectConfirmation.Recipient` = ACS, ventanas `NotBefore/NotOnOrAfter` con skew ≤ 60s, y **replay cache** del `AssertionID` en Redis con TTL = resto de `NotOnOrAfter`.
11-16. Idénticos al camino OIDC, usando `NameID` + `Issuer` como clave de link y atributos/grupos mapeados.

**2.2.2 SAML IdP-Initiated (Unsolicited Response)**
1. El IdP POSTea una `SAMLResponse` **sin** `InResponseTo` (el flujo no partió de SHUGO).
2. Política default: **DENY** (RN-30). Si la conexión habilita `allow_idp_initiated=true`:
   a. Se exigen todas las validaciones del escenario 2.2.1 excepto `InResponseTo`, más replay cache de AssertionID y firma válida.
   b. Se registra el evento con flag `unsolicited=true` para auditoría y scoring de riesgo (UC-17).

**2.2.3 Account Linking por Email Verificado**
1. No existe link por `(issuer, sub)`, pero existe identidad local con el mismo email.
2. Si `email_verified=true` en el IdP **y** la política de la conexión permite auto-linking: se crea el link y se audita `Account.Linked{method=email_verified}`.
3. Si el email no está verificado o la política lo prohíbe: se retorna un flujo de conflicto (consentimiento explícito con re-autenticación local o intervención de admin), **nunca** link silencioso.

**2.2.4 Back-Channel Logout (OIDC) / Single Logout (SAML)**
1. El IdP POSTea un `logout_token` firmado (OIDC Back-Channel Logout) o un `LogoutRequest` (SAML SLO) al endpoint dedicado de KAGI.
2. KAGI valida firma/iss/aud y extrae `sid` (sesión IdP) o `sub`.
3. KAGI localiza las familias locales con `idp_session_id` coincidente y las revoca vía UC-12 con motivo `IDP_SESSION_TERMINATED`.
4. Se responde `200 OK` al IdP (OIDC exige 200 aunque no haya sesiones) y se publica `IdpBackchannelLogoutReceived`.

### 2.3. Flujos de Excepción

**[4.1] State Inválido o Ausente (CSRF / Replay)**
- 4.1.1 El `state` del callback no existe en Redis o ya fue consumido.
- 4.1.2 Retornar error genérico de federación al cliente (`federate_failed`) sin detalles; auditar `FederatedAuth.Failed{stage=state}`.

**[4.2] Validación de id_token / Assertion Fallida**
- 4.2.1 Firma inválida, `iss`/`aud`/`exp`/`nonce`/`InResponseTo`/`Destination` incorrectos, o algoritmo prohibido.
- 4.2.2 En `unknown kid`: un único refetch de JWKS (key rollover); si persiste el fallo, rechazar.
- 4.2.3 Retornar `federate_failed` genérico; auditar con motivo interno; alertar si la tasa de fallos de firma sube (posible ataque o misconfiguración de IdP).

**[4.3] Replay de Assertion o Code**
- 4.3.1 Authorization code ya canjeado, nonce reutilizado, o AssertionID en replay cache.
- 4.3.2 Rechazar y auditar como evento de seguridad (`FederatedAuth.Failed{stage=replay}`); SIEM alerta.

**[4.4] Conflicto de JIT Provisioning**
- 4.4.1 El email del claim ya existe pero la política de auto-linking está desactivada, o el email está en dominio no permitido por el tenant.
- 4.4.2 Retornar flujo de conflicto (no crear identidad, no linkear); auditar `JitProvisioning.Conflict`.

**[4.5] IdP No Alcanzable (Discovery/Token Endpoint caído)**
- 4.5.1 Timeout o error TLS en endpoints del IdP durante start o callback.
- 4.5.2 Retornar `idp_unavailable` (503-equivalente de federación). Si la política de fallback del tenant lo permite **y** la identidad tiene credenciales locales, ofrecer login local (UC-11); default: deny.

**[4.6] Redirect URI no en Whitelist**
- 4.6.1 El `redirect_uri` del `/start` o del callback no coincide exactamente (string match, sin wildcards) con la whitelist de la conexión.
- 4.6.2 Rechazar inmediatamente (400); auditar como posible intento de open-redirect / token exfiltration.

### 2.4. Postcondiciones

**De Éxito:**
1. Existe una familia de tokens local con proveniencia federada (`auth_method=federated`, `idp`, `idp_session_id`, `acr/amr`).
2. Existe (o se reutilizó) un `federated_identity_link` `(issuer, sub) → identity_id`.
3. Si fue primer login con JIT: identidad creada en KAGAMI con roles mapeados y evento `Identity.JitProvisioned`.
4. Eventos de auditoría publicados con hash del payload IdP (sin persistir claims crudos más allá del mapeo, RN-29).

**De Falla:**
1. No se emiten tokens ni se crean links/identidades.
2. El transaction state se consume o expira (single-use), impidiendo reintentos del callback.
3. El fallo queda auditado con stage interno; el cliente recibe un error genérico.

---

## 3. Reglas de Negocio e Invariantes de Dominio (DDD)

| ID | Invariante / Regla | Descripción Técnica |
| :--- | :--- | :--- |
| **INV-89** | PKCE + State Obligatorios | Todo flujo OIDC usa Authorization Code + PKCE S256 y `state` single-use en Redis (TTL 10 min). Callback sin state válido se rechaza antes de tocar el IdP. |
| **INV-90** | Validación Estricta de Artefactos IdP | id_token: firma JWKS pineada, `iss` exacto, `aud`=client_id, `exp/iat` skew ≤60s, `nonce` igual, algoritmo en allowlist (RS256/ES256/EdDSA). SAML: firma XML del metadata, `Destination`, `AudienceRestriction`, `Recipient`, `Conditions`, `InResponseTo` (salvo IdP-initiated habilitado). |
| **INV-91** | Anti-Replay de Code, Nonce y Assertion | Authorization code single-use; nonce single-use por transaction; `AssertionID` y `sid` SAML en replay cache Redis con TTL = ventana `NotOnOrAfter`. |
| **INV-92** | Account Linking con Política de Email Verificado | Auto-link por email solo si `email_verified=true` en el IdP y la conexión lo permite. Sin verificación: flujo de conflicto con consentimiento o admin. |
| **INV-93** | JIT Provisioning con Mapeo Declarativo | La creación JIT usa exclusivamente el mapeo de claims/grupos declarado en la conexión (UC-25). Grupos no mapeados no otorgan roles. Emite `Identity.JitProvisioned`. |
| **INV-94** | Proveniencia Federada en la Sesión Local | La familia local registra `auth_method=federated`, `idp`, `idp_session_id`, `acr`, `amr`. `auth_level` se deriva del assurance del IdP (acr con MFA → AAL2); nunca se asume AAL2 sin evidencia en `acr/amr`. |
| **INV-95** | Redirect URI Whitelist Exacta | Comparación exacta de strings contra la whitelist de la conexión en `/start` y en callback. Sin wildcards, sin normalizaciones permisivas. |
| **INV-96** | Propagación de Logout del IdP | Back-Channel Logout (OIDC) y SAML SLO revocan las familias locales con `idp_session_id` coincidente vía UC-12 (motivo `IDP_SESSION_TERMINATED`). |
| **RN-27** | Home Realm Discovery sin Fugas | La resolución tenant/conexión por dominio de email retorna errores genéricos; no revela qué dominios tienen federación configurada. |
| **RN-28** | Metadata Cache con Rotación de Claves | JWKS/metadata cacheados (TTL 1h + ETag). Ante `unknown kid` o certificado desconocido: un único refetch; si falla, rechazo. Cambios de `issuer` o EntityID nunca se aceptan dinámicamente. |
| **RN-29** | Minimización de Claims (GDPR Art. 5) | Solo se persisten claims mapeados por la conexión. El payload crudo del IdP no se almacena; se audita un hash SHA256 del payload para forense sin retener PII extra. |
| **RN-30** | IdP-Initiated Default Deny | Las respuestas SAML unsolicited se rechazan salvo `allow_idp_initiated=true` en la conexión, con replay cache y flag `unsolicited=true` en auditoría y riesgo. |
| **RN-31** | Fallback a Local Auth por Política | Ante IdP caído, default deny. Solo si la conexión declara `local_fallback=true` y la identidad tiene credencial local activa, se ofrece UC-11. Nunca fallback silencioso. |

---

## 4. Criterios de Aceptación (BDD / Gherkin)

### Escenario 4.1: OIDC SP-Initiated con JIT Provisioning (primer login)
```gherkin
DADO que el tenant "khcs-001" tiene una conexión OIDC "okta-corp" habilitada
Y el usuario "new.hire@khcs.com" NO existe como identidad local
Y el IdP retorna un id_token válido con email_verified=true y groups=["engineering"]
CUANDO se completa el flujo /federate/start → IdP → /federate/callback → /federate/token
ENTONCES se crea una identidad local con display_name mapeado y rol "engineer" (según mapeo de grupos)
Y se crea un federated_identity_link (issuer=okta, sub=...) → identity_id
Y se emite una familia de tokens con auth_method=federated, idp=okta, acr/amr heredados
Y se publican "Identity.JitProvisioned" y "Authentication.FederatedSucceeded"
```

### Escenario 4.2: Usuario recurrente con link existente
```gherkin
DADO que existe un federated_identity_link (issuer=okta, sub=abc) → identity "user-123"
CUANDO el usuario completa el flujo federado con sub=abc
ENTONCES no se crea ninguna identidad nueva
Y se emite una familia de tokens para "user-123" con idp_session_id del IdP
Y se publica "Authentication.FederatedSucceeded" con jitProvisioned=false
```

### Escenario 4.3: SAML SP-Initiated con validación completa
```gherkin
DADO una conexión SAML "adfs-corp" con metadata vigente
CUANDO el IdP POSTea una SAMLResponse firmada con InResponseTo igual al AuthnRequestID emitido
Y AudienceRestriction = SP EntityID y Recipient = ACS URL y NotOnOrAfter vigente
ENTONCES la autenticación procede y se resuelve la identidad por NameID+Issuer
Y el AssertionID queda en replay cache hasta NotOnOrAfter
```

### Escenario 4.4: SAML IdP-Initiated con política default deny
```gherkin
DADO que la conexión "adfs-corp" tiene allow_idp_initiated=false (default)
CUANDO el IdP POSTea una SAMLResponse sin InResponseTo (unsolicited)
ENTONCES el sistema rechaza la autenticación con error genérico
Y se audita "FederatedAuth.Failed" con stage=idp_initiated_denied
```

### Escenario 4.5: State ausente o reutilizado (CSRF/Replay)
```gherkin
DADO un transaction state ya consumido por un callback previo
CUANDO un segundo callback presenta el mismo state
ENTONCES el sistema rechaza con error genérico federate_failed
Y se audita el evento con stage=state_reuse
```

### Escenario 4.6: Key rollover del IdP (unknown kid)
```gherkin
DADO que el IdP rota su clave de firma y el JWKS cacheado no contiene el nuevo kid
CUANDO se valida un id_token firmado con la clave nueva
ENTONCES el sistema realiza UN refetch del JWKS
Y la validación prosigue exitosamente con la clave nueva
Y el cache se actualiza con el nuevo JWKS
```

### Escenario 4.7: Email no verificado → sin auto-link
```gherkin
DADO un id_token con email "user@khcs.com" y email_verified=false
Y existe una identidad local con ese email pero sin link federado
CUANDO se completa el callback
ENTONCES el sistema NO crea link automático ni JIT
Y retorna el flujo de conflicto de linking (consentimiento/admin)
Y se audita "AccountLinking.Conflict"
```

### Escenario 4.8: Back-Channel Logout del IdP termina familias locales
```gherkin
DADO una familia local federada con idp_session_id="idp-sess-77"
CUANDO el IdP POSTea un logout_token válido con sid="idp-sess-77"
ENTONCES todas las familias locales con ese idp_session_id son revocadas (motivo IDP_SESSION_TERMINATED)
Y se responde HTTP 200 al IdP
Y se publica "IdpBackchannelLogoutReceived" con familiesTerminated
```

### Escenario 4.9: IdP caído con política fallback deny
```gherkin
DADO que el token_endpoint del IdP retorna timeout durante el callback
Y la conexión tiene local_fallback=false (default)
CUANDO se procesa el callback
ENTONCES el sistema retorna idp_unavailable sin emitir tokens
Y NO se ofrece login local automático
```

---

## 5. Eventos de Dominio Generados

### Evento 1: `Authentication.FederatedSucceeded`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Authentication.FederatedSucceeded",
  "timestamp": "2026-09-07T08:30:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "connectionId": "okta-corp",
    "protocol": "OIDC",
    "issuer": "https://khcs.okta.com",
    "idpSessionId": "idp-sess-77",
    "acr": "urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport",
    "amr": ["pwd", "mfa"],
    "authLevel": "AAL2",
    "jitProvisioned": false,
    "linked": false,
    "unsolicited": false,
    "payloadHash": "sha256:9f2c..."
  }
}
```
**Consumidores:** KAGE (audit), KIZUNA (contexto de autorización por proveniencia), UC-17 (baseline de comportamiento), TORII (webhooks de login enterprise).

### Evento 2: `Identity.JitProvisioned`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Identity.JitProvisioned",
  "timestamp": "2026-09-07T08:30:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-999",
    "sourceIssuer": "https://khcs.okta.com",
    "sourceSubject": "okta-sub-555",
    "mappedRoles": ["engineer"],
    "mappedAttributes": ["display_name", "department"],
    "initiatedBy": "FEDERATED_JIT"
  }
}
```
**Consumidores:** KAGE, KIZUNA (cache de roles), TORII (onboarding HRIS), Notification Service (bienvenida).

### Evento 3: `Account.Linked`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Account.Linked",
  "timestamp": "2026-09-07T08:31:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "issuer": "https://khcs.okta.com",
    "subject": "okta-sub-555",
    "method": "email_verified",
    "consentRequired": false
  }
}
```

### Evento 4: `FederatedAuth.Failed` (Evento de Seguridad)
```json
{
  "eventId": "uuid-v4",
  "eventType": "FederatedAuth.Failed",
  "timestamp": "2026-09-07T08:32:00Z",
  "tenantId": "khcs-001",
  "severity": "SECURITY",
  "payload": {
    "connectionId": "okta-corp",
    "stage": "signature_validation",
    "internalReason": "UNKNOWN_KID_AFTER_REFETCH",
    "ipAddress": "203.0.113.44",
    "payloadHash": "sha256:1a2b..."
  }
}
```
**Consumidores:** KAGE, SIEM (alertas de tasa de fallos de firma/replay), métricas de salud de federación.

### Evento 5: `IdpBackchannelLogoutReceived`
```json
{
  "eventId": "uuid-v4",
  "eventType": "IdpBackchannelLogoutReceived",
  "timestamp": "2026-09-07T09:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "issuer": "https://khcs.okta.com",
    "idpSessionId": "idp-sess-77",
    "familiesTerminated": 2,
    "protocol": "OIDC_BACKCHANNEL"
  }
}
```

---

## 6. Consideraciones de Arquitectura y Seguridad

### 6.1. SHUGO como Relying Party: Confianza Delimitada

La federación delega **autenticación**, no **autorización ni sesión**:
- El IdP decide *quién es* el usuario; SHUGO decide *qué puede hacer* (KIZUNA) y *cuánto dura su sesión* (UC-13 con vida absoluta).
- Los tokens de sesión local **nunca** se exponen al IdP; los tokens del IdP **nunca** se aceptan como tokens de acceso a APIs de SHUGO.
- El `acr/amr` del IdP es **evidencia**, no permiso: UC-17 y las políticas de tenant pueden exigir step-up local adicional.

### 6.2. Transaction State Server-Side (state + nonce + verifier)

En lugar de confiar solo en parámetros de URL, KAGI mantiene el estado completo de la transacción en Redis (`federate:tx:{transaction_id}`, TTL 10 min, single-use):
- **CSRF:** `state` vincula callback a transacción.
- **Replay de id_token:** `nonce` vincula el token a la transacción.
- **PKCE:** `code_verifier` nunca viaja al navegador; solo el challenge va en la redirect.
- **Open-redirect:** el `redirect_uri` final se recupera del transaction state, no del callback.

### 6.3. Validación de Artefactos y Key Rollover

- **Allowlist de algoritmos:** `none` rechazado siempre; HS256 rechazado en conexiones asimétricas; RS256/ES256/EdDSA permitidos según metadata.
- **Issuer/EntityID inmutables:** nunca se aceptan issuers descubiertos dinámicamente; la metadata se pinea en la conexión (UC-25).
- **Rollover controlado (RN-28):** un único refetch ante `unknown kid`; el segundo fallo consecutivo rechaza y alerta (posible ataque de inyección de claves o IdP comprometido).

### 6.4. JIT Provisioning como Superficie de Privilegios

El mapeo de grupos→roles es **declarativo y cerrado** (INV-93):
- Solo grupos explícitamente mapeados en la conexión otorgan roles; cualquier otro grupo se ignora.
- Roles administrativos (`super_admin`, `tenant_admin`) **no son asignables vía JIT** salvo flag explícito de la conexión con aprobación de compliance.
- Dominios de email permitidos por conexión: un id_token de `gmail.com` no puede JIT-provisionar en un tenant que solo federada `khcs.com`.

### 6.5. Minimización de Datos y Forense sin PII (RN-29)

Se persisten únicamente los claims mapeados. Para forense, se audita `payloadHash = SHA256(payload_crudo)`: permite verificar integridad de un payload presentado posteriormente sin retener PII no mapeada (GDPR Art. 5(1)(c)).

### 6.6. Propagación Bidireccional de Logout

- **IdP → SHUGO:** Back-Channel Logout (OIDC) y SAML SLO revocan familias por `idp_session_id` (INV-96).
- **SHUGO → IdP:** El logout RP-initiated de UC-12 redirige/adiciona al `end_session_endpoint` (OIDC con `id_token_hint`) o envía `LogoutRequest` (SAML), completando el círculo de federación de sesiones.

---

## 7. Contratos de API (OpenAPI 3.0 Snippets)

### 7.1. Endpoint: POST /auth/federate/start

```http
POST /v1/auth/federate/start HTTP/1.1
Content-Type: application/json

{
  "email": "user@khcs.com",
  "connection_id": "okta-corp",        // opcional si hay home realm discovery
  "redirect_uri": "https://app.khcs.com/auth/callback",
  "client_state": "opaque-client-value"
}
```
**Response 302:** `Location: https://khcs.okta.com/oauth2/v1/authorize?...&state=...&nonce=...&code_challenge=...`

### 7.2. Endpoint: GET /auth/federate/callback (ACS unificado OIDC/SAML)

```http
GET /v1/auth/federate/callback?code=abc123&state=tx-state-xyz
-- o --
POST /v1/auth/federate/callback   (SAMLResponse=base64...)
```
**Response 302:** `Location: {redirect_uri_whitelist}?code={federate_code}&state={client_state}`

### 7.3. Endpoint: POST /auth/federate/token

```json
{ "code": "federate_code_60s_single_use" }
```
**Response 200:**
```json
{
  "token_type": "Bearer",
  "access_token": "eyJhbGciOiJFZERTQSIs...",
  "refresh_token": "opaqueSuccessor...",
  "expires_in": 900,
  "session_meta": { "auth_method": "federated", "idp": "https://khcs.okta.com", "auth_level": "AAL2" }
}
```

### 7.4. Endpoint: POST /auth/federate/backchannel-logout (IdP → SHUGO)

```http
POST /v1/auth/federate/backchannel-logout
Content-Type: application/x-www-form-urlencoded

logout_token=<JWT firmado por el IdP con event=logout>
```
**Response 200 OK** (siempre, tras validación de firma; OIDC Back-Channel lo exige).

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `02_Use_Case_Diagram.md` | Mostrará la interacción Usuario ↔ Cliente ↔ KAGI ↔ IdP ↔ KAGAMI, con JIT y Linking como `<<extend>>`. |
| `04_Initial_Domain_Model.md` | Definirá `FederatedConnection` (config), `FederatedIdentityLink` (Entity), `IdpArtifact` (VO), `FederationSession` y los services `OidcRelyingParty`, `SamlServiceProvider`, `AccountLinker`, `JitProvisioner`. |
| `05_State_Machine_Diagram.md` | Modelará la máquina de la `FederationTransaction` (INITIATED → AWAITING_CALLBACK → TOKEN_EXCHANGED → LINKED/PROVISIONED → SESSION_ISSUED / FAILED). |
| `10_UC_Specific_Data_Model.md` | Definirá `federated_identity_links`, `federated_connections` (metadata pineada), replay caches Redis y extensión de `refresh_tokens` con columnas de proveniencia. |
| `11_UC_Specific_Security_Threats.md` | Analizará token/assertion injection, key injection via JWKS, open-redirect, JIT privilege escalation, replay de assertions y IdP impersonation. |
| `UC-12/01_Requirements_Specification.md` | Consume `IDP_SESSION_TERMINATED` como nuevo motivo de revocación para Back-Channel Logout. |
| `UC-25_Configurar_Tenant` | Proveedor de la configuración de conexiones (metadata, mapeos, políticas de linking/JIT/fallback). |

---

## 9. Glosario de Términos Específicos de UC-14

| Término | Definición |
| :--- | :--- |
| **Relying Party (RP) / Service Provider (SP)** | Rol de SHUGO en la federación: confía en aserciones del IdP para establecer sesión local, sin delegar autorización ni gestión de sesión. |
| **Identity Provider (IdP)** | Sistema externo que autentica al usuario y emite aserciones (Okta, Entra ID, Google, ADFS). |
| **Home Realm Discovery** | Resolución de tenant + conexión de IdP a partir del dominio del email o selección explícita, sin filtrar qué dominios están federados. |
| **Transaction State** | Contexto server-side single-use en Redis (state, nonce, code_verifier, redirect_uri) que vincula start ↔ callback y previene CSRF, replay y open-redirect. |
| **PKCE (RFC 7636)** | Proof Key for Code Exchange: `code_challenge` S256 en la redirect y `code_verifier` server-side en el canje, previniendo interceptación del code. |
| **Account Linking** | Asociación de un `(issuer, sub)` del IdP a una identidad local existente, gobernada por política (email verificado, consentimiento, admin). |
| **JIT Provisioning** | Creación automática de identidad local en el primer login federado, con atributos y roles derivados exclusivamente del mapeo declarativo de la conexión. |
| **acr / amr** | Authentication Context Class Reference / Authentication Methods References: evidencia del nivel y métodos de autenticación en el IdP; base del `auth_level` heredado. |
| **Back-Channel Logout** | Notificación firmada del IdP a SHUGO (logout_token OIDC / LogoutRequest SAML) para terminar sesiones locales vinculadas a `idp_session_id`. |
| **IdP-Initiated (Unsolicited)** | Flujo SAML que arranca en el IdP sin AuthnRequest de SHUGO; default deny, habilitable por conexión con controles reforzados. |
| **Replay Cache** | Claves Redis con TTL que registran nonces, AssertionIDs y codes consumidos para impedir reutilización dentro de su ventana de validez. |
| **Federate Code** | Código local de un solo uso (60s) que el cliente intercambia por tokens, evitando exponer tokens en URLs de redirección. |

---
*Fin del Documento 01_Requirements_Specification para UC-14*