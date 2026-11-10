# UC-16: Passwordless Authentication - Especificación de Requisitos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-16 - Passwordless Authentication (Passkeys + Magic Links + OTP Restringido) |
| **Documento:** | 01_Requirements_Specification.md |
| **Metodología:** | ICONIX (Deep Dive) + DDD + BDD + Security by Design (Phishing-Resistant First) |
| **Estándares:** | W3C WebAuthn Level 2 / FIDO2, NIST SP 800-63B (Verifier Impersonation Resistance, Restricted Authenticators), RFC 8176 (amr), OWASP ASVS 4.0 (V2) + OWASP Passwordless Cheat Sheet, GDPR Art. 5/32, SOC 2 CC6.1, ISO 27001:2022 A.5.18 |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 09/11/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 09/11/2026 | Gastón Ruiz | **Creación inicial.** Definición completa de la especificación analítica para autenticación sin contraseña. Cubre tres canales con clasificación de assurance explícita: Passkeys (WebAuthn como primer factor con userVerification obligatorio, credenciales discoverable y conditional UI), Magic Links con consumo en dos pasos anti-prefetch (validación GET que no consume + confirmación POST que consume), binding de dispositivo y handoff cross-device opcional, y OTP de email como método RESTRINGIDO con intentos acotados. Establece la interacción con UC-15 (métodos AAL1 pausan hacia segundo factor), UC-17 (riesgo en login passwordless) y la maquinaria de sesiones de UC-11 (familias con proveniencia `auth_method=passwordless_*`). | Documento base para el paquete de diseño Tier 1 del UC-16. Es el caso de uso que elimina el secreto compartido (password) del camino primario de autenticación, sustituyéndolo por posesión criptográfica (passkeys) o canales out-of-band con vida útil corta (magic links/OTP), sin sacrificar la detección de abuso ni la postura anti-enumeración de la plataforma. |

---

## 1. Introducción y Alcance

Este documento detalla la especificación de requisitos para el Caso de Uso **UC-16**, que representa la operación de **Autenticación sin Contraseña** en el motor **KAGI** de SHUGO.

La contraseña es el eslabón más atacado de la autenticación enterprise: phishable, reutilizable entre servicios, y costosa de operar (reset, rotación, lockout). UC-16 ofrece tres canales passwordless con posturas de seguridad radicalmente distintas que el diseño debe expresar sin ambigüedad:

1. **Passkeys (WebAuthn como primer factor):** Credenciales discoverable con `userVerification` obligatorio. Es el único canal **phishing-resistant** del catálogo: no hay secreto que robar, la ceremonia está vinculada al origen, y la posesión se demuestra criptográficamente.
2. **Magic Links:** Enlace de un solo uso enviado al email verificado del usuario. Conveniente y sin secreto memorizado, pero depende de la seguridad del buzón: clasificación **AAL1 restringida**.
3. **OTP de Email (restringido):** Código de 8 dígitos con TTL corto e intentos acotados, para contextos donde el enlace no es viable (clientes sin browser embebido). Default OFF por tenant.

### 1.1. Importancia Estratégica

El UC-16 es **la ruta de eliminación del password** para tenants maduros, y su diseño debe sostener cuatro propiedades:

- **Clasificación honesta de assurance:** No todo passwordless es igual. Un passkey con UV es evidencia de posesión criptográfica (AAL2/AAL3); un magic link es posesión de buzón (AAL1). Los tenants deben poder exigir el nivel, y KIZUNA debe ver la diferencia en `amr`/`auth_level`.
- **Interoperabilidad con el stack de sesión:** El login passwordless emite la misma familia de tokens (UC-11/13) con proveniencia `passwordless_*`, rota igual, y termina igual (UC-12). No es un sistema paralelo.
- **Anti-abuso sin oráculos:** Los canales por email son vectores naturales de enumeración y stuffing. Las respuestas deben ser genéricas, el timing normalizado, y el rate limiting aplicado sin revelar existencia de cuentas.
- **Consumo a prueba de intermediarios:** Los email clients modernos hacen prefetch/pre-render de enlaces. Un magic link consumido por un prefetch es un DoS de login para el usuario legítimo; el diseño de dos pasos (validar sin consumir → confirmar para consumir) elimina esta clase de fallo.

### 1.2. Relación con Otros Casos de Uso

| UC | Relación | Descripción |
| :--- | :--- | :--- |
| **UC-11 (Login Local)** | Reutiliza maquinaria | La emisión de familia (`CreateRoot`), el intento de autenticación y el lockout se reutilizan; cambia el factor de verificación (posesión en lugar de conocimiento). |
| **UC-15 (MFA)** | Consumidor de pausas | Métodos AAL1 (magic link, OTP) **pausan** hacia UC-15 si la política del tenant exige AAL2+. Passkeys con UV satisfacen AAL2 directamente sin pausa. El enroll de passkeys reutiliza la superficie de enrollments de UC-15 con `usage=first_factor`. |
| **UC-13 (Refresh)** | Consumidor | Las familias passwordless rotan idénticamente; la vida absoluta fuerza re-autenticación passwordless periódica. |
| **UC-12 (Logout)** | Consumidor | Terminación y back-channel operan igual sobre familias passwordless (proveniencia en la fila). |
| **UC-14 (Federada)** | Alternativa complementaria | Un tenant puede ofrecer federación corporativa Y passkeys; el patrón de one-time code de canje (`session_code` 60s) se reutiliza de UC-14. |
| **UC-17 (Risk-Based)** | Consumidor | Cada login passwordless evalúa delta de contexto; magic links desde geo/IP anómalo pueden requerir step-up o denegación. |
| **UC-18 (Recuperación)** | Distinción de alcance | Magic links de recuperación (UC-18) tienen política propia (reset de credenciales); UC-16 es autenticación primaria. Comparten infraestructura de despacho de email pero NO tokens. |
| **UC-22 (Device Management)** | Consumidor | Los passkeys enrolados son visibles/revocables desde la gestión de dispositivos. |

### 1.3. Catálogo de Canales y Clasificación de Assurance

| Canal | Estándar | AAL | Phishing-Resistant | amr (RFC 8176) | Política Default |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **passkey** | WebAuthn L2 (discoverable, UV) | AAL2 (AAL3 con hardware verificado + attestation directa) | **Sí** | `["hwk", "user"]` | Habilitado |
| **magic_link** | OOB por email, single-use | AAL1 restringido | No | `["otk"]` | Opt-in por tenant |
| **email_otp** | OOB por email, intentos acotados | AAL1 restringido | No | `["otp"]` | Default OFF |

**Regla de interacción:** Si `required_level ≥ AAL2` y el usuario autentica por canal AAL1, el flujo **pausa hacia UC-15** para segundo factor; el login no se completa con assurance insuficiente. Los passkeys nunca pausan por assurance (su evidencia es el propio factor).

### 1.4. Objetivos de Rendimiento

| Métrica | Objetivo | Justificación |
| :--- | :--- | :--- |
| **Passkey options (p95)** | < 50 ms | Generación de challenge + resolución de credenciales sugeridas. |
| **Passkey verify (p95)** | < 80 ms | Ceremonia WebAuthn: firma + counter + resolución de identidad. |
| **Magic link issue (p95)** | < 100 ms | Store de token + despacho de email **async**. |
| **Magic link confirm (p95)** | < 150 ms | Consumo + riesgo + emisión de familia + session_code. |
| **OTP verify (p95)** | < 50 ms | Hash compare en tiempo constante + intentos. |
| **Disponibilidad local** | 99.99% | El despacho de email es asíncrono y no bloqueante. |

---

## 2. Descripción Analítica (ICONIX Clásico)

| Campo | Detalle |
| :--- | :--- |
| **ID Y NOMBRE:** | UC-16 - Passwordless Authentication (Passkeys + Magic Links + OTP Restringido) |
| **ESTADO:** | Preliminar |
| **DESCRIPCIÓN:** | El usuario autentica sin contraseña mediante uno de tres canales. **Passkeys:** ceremonia WebAuthn de primer factor con userVerification obligatorio que resuelve la identidad por credencial y emite sesión AAL2+ directamente. **Magic Links:** el sistema genera un token de 256 bits vinculado a identidad y dispositivo, lo envía al email verificado, y el usuario completa en dos pasos (GET que valida sin consumir, POST que consume y emite). **OTP:** código de 8 dígitos con TTL 5 min y máximo 5 intentos. Los canales AAL1 pausan hacia UC-15 si la política exige más assurance; todos emiten familias con proveniencia `passwordless_*` y son evaluados por UC-17. |
| **ACTOR PRINCIPAL:** | Usuario con identidad activa en el tenant (sin password requerido). |
| **ACTOR SECUNDARIO:** | Authenticator de plataforma (passkey), Proveedor de Email (despacho), API Gateway, Redis (tokens, intentos, rate), KAGAMI (identidades, credenciales, familias), UC-17 (Risk), UC-15 (MFA), KAGE (Audit), KIZUNA (Authorization). |
| **PRECONDICIONES:** | 1. La identidad existe, está activa y tiene email verificado (canales OOB) o passkey enrolado (canal WebAuthn).<br>2. El canal está habilitado por política del tenant.<br>3. El redirect_uri del cliente está en whitelist (canal magic link). |
| **PUNTOS DE EXTENSIÓN:** | Handoff cross-device de magic link (código de 6 dígitos en el dispositivo origen); pausa hacia UC-15 cuando assurance AAL1 no satisface política; step-up preventivo por riesgo (UC-17); enrollment de passkey desde sesión autenticada. |
| **CONDICIÓN:** | El usuario posee una familia de tokens con `auth_method=passwordless_{canal}`, `amr` correspondiente y `auth_time`; ningún password fue transmitido ni verificado. |

### 2.1. Escenario Principal (Happy Path) - Login con Passkey

1. El cliente invoca `POST /auth/passwordless/webauthn/options` con `{ username_hint? }`.
2. KAGI genera un challenge aleatorio (≥16 bytes), resuelve credenciales sugeridas si hay hint (sin revelar existencia a terceros: la respuesta es idéntica con o sin credenciales), y retorna `PublicKeyCredentialRequestOptions` con `userVerification: "required"` y `rpId` del tenant.
3. El authenticator de plataforma realiza la ceremonia (biometría/PIN local) y produce una assertion con flag UV=1.
4. El cliente envía `POST /auth/passwordless/webauthn/verify` con la assertion.
5. KAGI valida: challenge single-use, origin/rpId exactos, **flag UV presente (INV-105)**, firma contra la credencial registrada, y signature counter (mecanismo de clonación de UC-15).
6. KAGI resuelve la identidad por `(rpId, credentialId)` y verifica que está activa en el tenant.
7. UC-17 evalúa el delta de contexto (device fingerprint, IP, geo). Score alto → step-up o denegación según política.
8. KAGI emite la familia de tokens (`CreateRoot`) con proveniencia: `auth_method = passwordless_webauthn`, `amr = ["hwk","user"]`, `auth_level = AAL2` (o AAL3 con hardware verificado + attestation directa según política).
9. Se publica `Passwordless.WebAuthnAuthenticated` y se retorna el par de tokens (vía `session_code` one-time de 60s si el cliente es público).

### 2.2. Escenarios Alternativos

**2.2.1 Magic Link - Mismo Dispositivo (Consumo en Dos Pasos)**
1. El usuario solicita `POST /auth/passwordless/magic-link { email, redirect_uri }`.
2. KAGI responde **siempre** con el mismo mensaje genérico (`202 accepted`), exista o no la identidad (INV-109).
3. Internamente: si la identidad existe, está activa, el canal está habilitado y el email es verificado → genera token de 256 bits + `session binding` (device fingerprint, IP-class) y almacena `magiclink:{token_hash}` con TTL 10 min; despacha el email **async** con el enlace y publica `Passwordless.MagicLinkIssued`.
4. El usuario hace clic: `GET /auth/passwordless/link?token=...`. KAGI valida existencia, TTL y no-consumo, compara el binding de dispositivo, y retorna una **página de confirmación** — **sin consumir el token** (RN-37). Un prefetch/pre-render del email client muere aquí sin efecto.
5. El usuario confirma: `POST /auth/passwordless/link/confirm { token }`. KAGI consume (GETDEL), evalúa riesgo (UC-17), y emite la familia con `auth_method = passwordless_magic_link`, `amr = ["otk"]`, `auth_level = AAL1`.
6. Si la política del tenant exige AAL2+: la emisión **pausa** hacia UC-15 con `mfa_token`; si no, emite tokens finales vía `session_code` (60s) para canje back-channel.

**2.2.2 Magic Link - Cross-Device con Handoff**
1. En la solicitud, si la política `cross_device_handoff` está habilitada, KAGI genera además un **código de handoff de 6 dígitos** y lo muestra en la respuesta al dispositivo origen (no viaja en el email).
2. El usuario abre el enlace en otro dispositivo; el GET de validación detecta device mismatch y presenta la opción de handoff.
3. El usuario ingresa el código de 6 dígitos en el dispositivo nuevo; el confirm lo incluye y KAGI lo verifica contra el binding original antes de consumir.
4. El evento `MagicLinkConsumed` registra `cross_device = true` para auditoría y scoring de riesgo.

**2.2.3 OTP de Email (Restringido)**
1. El usuario solicita `POST /auth/passwordless/otp/request { email }` → respuesta genérica idéntica al magic link.
2. Internamente: si procede, genera código de 8 dígitos (CSPRNG), almacena hash + contador de intentos con TTL 5 min, limita a 3 emisiones por identidad/hora (RN-38), y despacha el email con el código (sin enlace).
3. El usuario envía `POST /auth/passwordless/otp/verify { email, code }`: comparación en tiempo constante contra el hash, máximo 5 intentos por emisión, single-use al primer acierto.
4. Éxito → emisión AAL1 o pausa hacia UC-15 según política; fallo agotado → OTP invalidado + evento de seguridad.

**2.2.4 Enrollment de Passkey (desde Sesión Autenticada)**
1. El usuario autenticado (con step-up si la política lo exige) inicia `POST /auth/passwordless/webauthn/enroll`.
2. KAGI retorna `PublicKeyCredentialCreationOptions` con credencial **discoverable** (`residentKey: "required"`), `authenticatorSelection.userVerification: "required"`, y challenge single-use.
3. El cliente completa la ceremonia; KAGI valida attestation según política (`none`/`indirect`), registra la credencial vinculada a la identidad con `usage = first_factor`, y publica `Passkey.Enrolled`.
4. La credencial queda visible/revocable desde UC-22 (Device Management).

### 2.3. Flujos de Excepción

**[4.1] Assertion sin UserVerification (Passkey)**
- 4.1.1 La assertion llega con flag UV=0 o userVerification no realizado.
- 4.1.2 Rechazo inmediato (`passwordless_failed` genérico); auditoría con motivo interno `uv_missing`. Nunca se emite sesión AAL2+ sin UV (INV-105).

**[4.2] Magic Link Expirado o Ya Consumido**
- 4.2.1 El GET de validación o el POST de confirmación encuentran el token ausente/expirado/consumido.
- 4.2.2 Respuesta genérica de enlace inválido/expirado con opción de re-solicitud; evento `MagicLinkFailed{stage}`. Un replay del enlace consumido es señal de intercepción → alerta si se repite.

**[4.3] Device Mismatch sin Handoff**
- 4.3.1 El GET de validación detecta device fingerprint divergente y la política no habilita handoff (o el código es incorrecto).
- 4.3.2 La confirmación se rechaza con motivo `device_mismatch`; el enlace **no se consume** en el GET (el usuario puede reintentar desde el dispositivo original); evento de seguridad con ambos fingerprints.

**[4.4] OTP Agotado por Intentos**
- 4.4.1 Cinco intentos fallidos sobre la misma emisión.
- 4.4.2 El OTP queda invalidado; el usuario debe solicitar una nueva emisión (sujeta al límite de 3/hora); evento `Passwordless.RateLimited{channel=otp}` al agotar también emisiones.

**[4.5] Identidad Inexistente o Inactiva (Canales OOB)**
- 4.5.1 La solicitud de magic link/OTP refiere un email sin identidad activa en el tenant.
- 4.5.2 **Respuesta idéntica al caso exitoso** (202 genérico), sin despacho de email; métrica interna `passwordless_request_unknown_email` para detección de stuffing. Ningún comportamiento observable diferencia los casos (INV-109).

**[4.6] Rate Limit por Email/IP (Stuffing)**
- 4.6.1 El número de solicitudes por email o por IP supera umbrales (ej. 5/hora por email, 20/hora por IP).
- 4.6.2 La respuesta sigue siendo el 202 genérico (para no revelar el límite como oráculo); internamente se suspende el despacho y se emite `Passwordless.RateLimited` al SIEM. El usuario legítimo afectado puede solicitar tras la ventana, o usar passkey.

**[4.7] Clonación de Passkey en Login**
- 4.7.1 Assertion con firma válida pero counter no creciente.
- 4.7.2 Contención idéntica a UC-15: revocación de la credencial, `MfaCloneDetected` adaptado a `PasskeyCloneDetected`, notificación al titular.

### 2.4. Postcondiciones

**De Éxito (Passkey):**
1. Familia de tokens emitida con `auth_method = passwordless_webauthn`, `amr = ["hwk","user"]`, `auth_level ≥ AAL2`, sin pausa de MFA.
2. Counter de la credencial actualizado (o contención si regresó).
3. Evento `Passwordless.WebAuthnAuthenticated` con contexto completo.

**De Éxito (Magic Link / OTP):**
1. Token/código consumido single-use; enlace/código inutilizable de por vida.
2. Familia AAL1 emitida directamente **o** pausa hacia UC-15 si la política exige más.
3. Eventos `MagicLinkConsumed` / `OtpVerified` con flags `cross_device` cuando aplique.

**De Falla:**
1. Ningún token emitido ni pausa creada.
2. Respuesta genérica al cliente; motivo interno solo en auditoría/métricas.
3. Contadores de rate/ intentos actualizados según canal.

---

## 3. Reglas de Negocio e Invariantes de Dominio (DDD)

| ID | Invariante / Regla | Descripción Técnica |
| :--- | :--- | :--- |
| **INV-105** | Passkeys exigen UserVerification | Toda assertion passwordless sin flag UV se rechaza. `auth_level ≥ AAL2` requiere evidencia UV; AAL3 requiere además hardware verificado + attestation directa según política. |
| **INV-106** | Magic Link single-use, TTL ≤10min, device-bound | Token de 256 bits almacenado por hash; consumo atómico GETDEL en la confirmación; binding a device fingerprint + IP-class del solicitante; cross-device solo con handoff habilitado por política. |
| **INV-107** | Higiene de tokens en tránsito | Tokens/códigos solo viajan en cuerpos POST y enlaces HTTPS; jamás en logs, métricas ni URLs de terceros (el redirect final usa `session_code` one-time, no el token del enlace). El email contiene el mínimo indispensable. |
| **INV-108** | Canales OOB son AAL1 restringido | Magic link y email OTP emiten assurance AAL1. Si `required_level ≥ AAL2`, el flujo pausa hacia UC-15; nunca se completa con assurance insuficiente. |
| **INV-109** | Anti-enumeración absoluta | Las solicitudes OOB responden idéntico exista o no la identidad, con timing normalizado y rate limiting interno no observable. La única diferencia vive en métricas internas. |
| **INV-110** | Binding de credencial passkey | Una credencial se vincula a `(tenant, identity, credentialId)` con unicidad; enrollamiento solo desde sesión autenticada con step-up según política; revocación vía UC-22. |
| **INV-111** | Identidad preexistente requerida | UC-16 no crea identidades: magic link/OTP requieren identidad activa con email verificado; el provisioning sigue siendo dominio de UC-01/UC-14 (JIT federado). |
| **RN-36** | Catálogo y política por tenant | Canales habilitados por tenant: passkeys default ON, magic_link opt-in, email_otp default OFF. La política declara `required_level` y `cross_device_handoff`. |
| **RN-37** | Consumo en dos pasos del Magic Link | GET valida sin consumir (inmune a prefetch/pre-render); POST confirm consume. Un GET repetido no degrada el enlace; solo el confirm lo hace. |
| **RN-38** | Límites de emisión y verificación OTP | Máx. 3 emisiones por identidad/hora; TTL 5 min por código; máx. 5 intentos por emisión; comparación en tiempo constante; single-use al primer acierto. |
| **RN-39** | Proveniencia passwordless en la sesión | Toda familia registra `auth_method = passwordless_{canal}`, `amr`, `auth_time` y `passwordless_channel_context` (device binding, cross_device). KIZUNA puede exigir step-up para operaciones sensibles realizadas con AAL1. |

---

## 4. Criterios de Aceptación (BDD / Gherkin)

### Escenario 4.1: Login con passkey (UV satisface AAL2)
```gherkin
DADO que el tenant "khcs-001" tiene passkeys habilitados y required_level = AAL2
Y la identidad "user-123" tiene un passkey enrolado con usage=first_factor
CUANDO el usuario completa la ceremonia WebAuthn con userVerification
ENTONCES el sistema emite una familia con auth_method=passwordless_webauthn y amr=["hwk","user"]
Y auth_level = AAL2 sin pausa de MFA
Y el counter de la credencial queda actualizado
Y se publica "Passwordless.WebAuthnAuthenticated"
```

### Escenario 4.2: Assertion sin userVerification rechazada
```gherkin
DADO un passkey enrolado con política UV obligatoria
CUANDO se presenta una assertion con flag UV=0
ENTONCES el sistema rechaza con passwordless_failed genérico
Y audita motivo interno "uv_missing"
Y no se emite sesión ni pausa alguna
```

### Escenario 4.3: Magic link mismo dispositivo con confirmación en dos pasos
```gherkin
DADO que la identidad tiene email verificado y el canal magic_link habilitado
CUANDO el usuario solicita el enlace y hace clic en él desde el mismo dispositivo
ENTONCES el GET muestra la página de confirmación sin consumir el token
Y el POST de confirmación consume el token (single-use) y emite familia AAL1
Y si required_level = AAL2, la emisión pausa hacia UC-15 con mfa_token
```

### Escenario 4.4: Replay de magic link consumido
```gherkin
DADO un magic link ya confirmado y consumido
CUANDO un segundo POST presenta el mismo token
ENTONCES el sistema responde enlace inválido (genérico)
Y publica "MagicLinkFailed{stage=consumed}"
Y si el replay se repite, alerta de posible intercepción al SIEM
```

### Escenario 4.5: Prefetch del email client no consume el enlace
```gherkin
DADO un magic link recién emitido
CUANDO el email client hace prefetch del enlace (GET automático)
ENTONCES el GET valida y muestra confirmación pero NO consume
Y el usuario puede confirmar después exitosamente
Y el prefetch queda registrado como acceso de validación (no como consumo)
```

### Escenario 4.6: Magic link cross-device con handoff
```gherkin
DADO política cross_device_handoff habilitada
CUANDO el usuario solicita el enlace y recibe el código de handoff en pantalla
Y abre el enlace en otro dispositivo e ingresa el código correcto
ENTONCES la confirmación se completa con cross_device=true en el evento
Y el código incorrecto rechaza la confirmación sin consumir el enlace
```

### Escenario 4.7: OTP con intentos acotados
```gherkin
DADO un OTP de 8 dígitos emitido con TTL de 5 minutos
CUANDO se fallan 5 intentos de verificación
ENTONCES el OTP queda invalidado
Y una nueva emisión está sujeta al límite de 3 por identidad/hora
Y se publica "Passwordless.RateLimited{channel=otp}" al agotar emisiones
```

### Escenario 4.8: Enumeración por email sin oráculo
```gherkin
DADO un atacante que solicita magic links para emails aleatorios
CUANDO el email no corresponde a identidad activa del tenant
ENTONCES la respuesta es idéntica (202 genérico) al caso de email existente
Y no se despacha email
Y la métrica interna registra la solicitud sin exponerla en la respuesta
```

### Escenario 4.9: Stuffing contenido por rate limit silencioso
```gherkin
DADO un volumen de solicitudes que supera 5/hora por email o 20/hora por IP
CUANDO el límite se alcanza
ENTONCES la respuesta sigue siendo el 202 genérico
Y el despacho interno se suspende para los excedentes
Y se emite "Passwordless.RateLimited" al SIEM con el patrón detectado
```

### Escenario 4.10: Clonación de passkey en login passwordless
```gherkin
DADO un passkey con signature counter almacenado = 57
CUANDO se presenta una assertion con firma válida y counter ≤ 57
ENTONCES el sistema revoca la credencial con motivo CLONE_DETECTED
Y publica "PasskeyCloneDetected" con alerta crítica
Y la ceremonia no produce sesión
```

---

## 5. Eventos de Dominio Generados

### Evento 1: `Passwordless.WebAuthnAuthenticated`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Passwordless.WebAuthnAuthenticated",
  "timestamp": "2026-09-09T08:30:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "credentialId": "cred-hash-abc",
    "authLevel": "AAL2",
    "userVerification": true,
    "attachment": "platform",
    "deviceFingerprint": "dev-hash-xyz",
    "ipAddress": "192.168.1.100",
    "familyRootId": "fam-300"
  }
}
```
**Consumidores:** KAGE (audit), KIZUNA (contexto de autorización), UC-17 (baseline de comportamiento passwordless), métricas de adopción.

### Evento 2: `Passwordless.MagicLinkIssued` / `Passwordless.MagicLinkConsumed` / `Passwordless.MagicLinkFailed`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Passwordless.MagicLinkConsumed",
  "timestamp": "2026-09-09T08:32:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "tokenHash": "sha256:ab12...",
    "crossDevice": true,
    "requestDeviceFingerprint": "dev-hash-xyz",
    "confirmDeviceFingerprint": "dev-hash-other",
    "ttlElapsedSeconds": 94,
    "resultingAuthLevel": "AAL1",
    "pausedForMfa": true
  }
}
```
```json
{
  "eventType": "Passwordless.MagicLinkFailed",
  "payload": {
    "stage": "device_mismatch | expired | consumed | handoff_invalid | risk_denied",
    "tokenHash": "sha256:ab12...",
    "ipAddress": "203.0.113.44"
  }
}
```
**Consumidores:** KAGE, SIEM (replays y mismatches), UC-17 (señales de intercepción).

### Evento 3: `Passwordless.OtpIssued` / `Passwordless.OtpVerified` / `Passwordless.OtpFailed`
```json
{
  "eventType": "Passwordless.OtpFailed",
  "payload": {
    "reason": "invalid_code | attempts_exhausted | expired | rate_limited",
    "emailHash": "sha256:cd34...",
    "attemptCount": 5,
    "ipAddress": "203.0.113.55"
  }
}
```

### Evento 4: `Passkey.Enrolled` / `Passkey.Revoked` / `PasskeyCloneDetected`
```json
{
  "eventId": "uuid-v4",
  "eventType": "PasskeyCloneDetected",
  "timestamp": "2026-09-09T09:00:00Z",
  "tenantId": "khcs-001",
  "severity": "SECURITY",
  "payload": {
    "identityId": "user-uuid-123",
    "credentialId": "cred-hash-abc",
    "storedCounter": 57,
    "presentedCounter": 55,
    "actionTaken": "CREDENTIAL_REVOKED",
    "ipAddress": "203.0.113.77"
  }
}
```

### Evento 5: `Passwordless.RateLimited` (Señal de Abuso)
```json
{
  "eventType": "Passwordless.RateLimited",
  "payload": {
    "channel": "magic_link | email_otp",
    "dimension": "email | ip",
    "emailHash": "sha256:cd34...",
    "count": 7,
    "windowMinutes": 60
  }
}
```
**Consumidores:** SIEM (campañas de stuffing), métricas de abuso, UC-17 (correlación con riesgo de sesión).

---

## 6. Consideraciones de Arquitectura y Seguridad

### 6.1. Clasificación de Assurance como Contrato, no como Sugerencia

La tabla del §1.3 es el corazón del UC: cada canal declara su assurance y el sistema la aplica mecánicamente:

*   **Passkeys con UV:** La evidencia criptográfica de posesión + verificación de usuario satisface AAL2 sin segundo factor adicional. Exigir MFA tras un passkey con UV sería fricción sin seguridad.
*   **Canales OOB (AAL1):** La posesión del buzón es un factor débil y recuperable (account recovery de email). Por eso **pausan hacia UC-15** cuando la política exige AAL2+, y KIZUNA puede exigir step-up para scopes sensibles aunque la sesión AAL1 sea válida.
*   **Justificación normativa:** Coherente con NIST SP 800-63B, que restringe los verificadores out-of-band basados en email y reserva la resistencia a verifier impersonation a mecanismos criptográficos vinculados al origen.

### 6.2. Consumo en Dos Pasos: Inmunidad a Prefetch

Los clientes de email (Outlook, Gmail, scanners de seguridad corporativos) hacen **pre-fetch y pre-render** de enlaces. Un magic link consumido por un prefetch es un login fallido para el usuario legítimo — y un vector que un atacante puede inducir (enviar el enlace a un scanner que lo consuma).

*   **GET `/link?token=...`:** Valida existencia, TTL, no-consumo y device binding. Retorna la página de confirmación. **Nunca consume.**
*   **POST `/link/confirm`:** Consume (GETDEL), evalúa riesgo y emite. Solo la acción deliberada del usuario produce efecto.
*   **Consecuencia:** El enlace sobrevive a cualquier número de GETs (scanners, previews, re-aperturas) y muere en exactamente un POST.

### 6.3. Anti-Enumeración con Rate Limiting No Observable

Los canales OOB son el vector clásico de enumeración ("si existe, llega un email"). El diseño lo cierra en tres capas:

1. **Respuesta idéntica:** 202 genérico exista o no la identidad; sin differences en headers o cuerpo.
2. **Timing normalizado:** El camino "no existe" ejecuta las mismas operaciones baratas (hash del email, consulta indexada, constante de tiempo) que el camino existe-hasta-el-despacho, y el despacho es async (fuera de la respuesta).
3. **Rate limit silencioso:** Al superar umbrales, la respuesta sigue siendo 202 pero el despacho se suspende internamente. El atacante no puede usar el límite como oráculo de existencia, y el SIEM recibe `Passwordless.RateLimited` con el patrón.

### 6.4. Passkeys como Primer Factor vs WebAuthn como Segundo Factor

UC-16 introduce una distinción que UC-15 no tenía: el **uso** de la credencial WebAuthn.

*   **UC-15 (segundo factor):** Ceremonia de verificación tras un primer factor; el enrollment existe porque el usuario lo añadió como MFA; `usage = second_factor`.
*   **UC-16 (primer factor):** Ceremonia de autenticación completa; credencial discoverable con UV obligatorio; `usage = first_factor`; resolución de identidad por credentialId (no hay identity previa en el request).
*   **Mecánica compartida:** Validación de firma, counter/clone detection y challenges single-use reutilizan los mismos componentes (verificador WebAuthn, stores de challenge), pero las políticas de assurance y enrollment son distintas. Una credencial puede existir en ambos usos si el usuario la enrola para ambos.

### 6.5. Handoff Cross-Device con Código Visible

El flujo cross-device legítimo (usuario solicita en desktop, confirma en mobile) se soporta con un **código de handoff de 6 dígitos mostrado en el dispositivo origen**, no enviado por email:

*   El email lleva el enlace; el código vive solo en la pantalla del dispositivo que solicitó.
*   Un atacante que intercepte el email necesita además el código del dispositivo origen.
*   El handoff es opt-in por política (`cross_device_handoff`); deshabilitado, el device mismatch rechaza la confirmación.

### 6.6. Higiene de Tokens y Session Bridge

*   **Tokens OOB:** 256 bits CSPRNG, almacenados por hash (SHA256) en Redis con TTL, nunca logueados; el email contiene el enlace y nada más identificable.
*   **Session bridge:** La emisión final usa el patrón de one-time code de UC-14: `session_code` de 60s canjeable por back-channel, de modo que los tokens nunca aparecen en URLs de redirect ni en el historial del browser.
*   **OTP:** 8 dígitos amplían el espacio frente a los 6 del TOTP (sin ventana de reintentos de 30s que los justifique menor); hash + intentos en Redis; invalidación al agotar.

---

## 7. Contratos de API (OpenAPI 3.0 Snippets)

### 7.1. Endpoint: POST /auth/passwordless/webauthn/options

```http
POST /v1/auth/passwordless/webauthn/options HTTP/1.1
Content-Type: application/json

{ "username_hint": "user@khcs.com" }
```
**Response 200:**
```json
{
  "publicKey": {
    "challenge": "base64url-16-bytes-min",
    "rpId": "khcs.com",
    "userVerification": "required",
    "allowCredentials": [],
    "extensions": { "appid": null }
  }
}
```
*Nota: `allowCredentials` vacío habilita credenciales discoverable; con hint puede poblarse sin revelar existencia a terceros (la respuesta es idéntica con o sin credenciales).*

### 7.2. Endpoint: POST /auth/passwordless/webauthn/verify

```json
{ "assertion": { "id": "credentialId", "rawId": "...", "response": { "clientDataJSON": "...", "authenticatorData": "...", "signature": "...", "userHandle": "..." } } }
```
**Response 200:** `{ "session_code": "one_time_60s", "expires_in": 60 }` → canje en `POST /auth/passwordless/token`.

### 7.3. Endpoints Magic Link

```http
POST /v1/auth/passwordless/magic-link
{ "email": "user@khcs.com", "redirect_uri": "https://app.khcs.com/auth/pwless" }
→ 202 { "message": "If the account exists and the channel is enabled, a sign-in link has been sent." }

GET /v1/auth/passwordless/link?token=...
→ 200 (página de confirmación; NO consume) | 400/410 genérico si inválido/expirado

POST /v1/auth/passwordless/link/confirm
{ "token": "...", "handoff_code": "123456" }   // handoff solo si aplica
→ 302 redirect_uri?session_code=... | 200 { "mfa_required": true, "mfa_token": "..." } si pausa UC-15
```

### 7.4. Endpoints OTP

```http
POST /v1/auth/passwordless/otp/request
{ "email": "user@khcs.com" }
→ 202 { "message": "If the account exists and the channel is enabled, a code has been sent." }

POST /v1/auth/passwordless/otp/verify
{ "email": "user@khcs.com", "code": "48291057" }
→ 200 { "session_code": "..." } | 200 { "mfa_required": true, "mfa_token": "..." } | 400 passwordless_failed
```

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `02_Use_Case_Diagram.md` | Mostrará los tres canales como casos de uso con validaciones comunes (anti-enumeración, rate, binding) y las extensiones (handoff, pausa UC-15, enrollment). |
| `04_Initial_Domain_Model.md` | Definirá `PasskeyCredential`, `MagicLinkToken`, `EmailOtpIssuance`, `PasswordlessSession` y los services `PasskeyCeremony`, `MagicLinkFlow`, `OtpFlow`. |
| `05_State_Machine_Diagram.md` | Modelará las máquinas de `MagicLinkToken` (ISSUED → VALIDATED → CONSUMED/EXPIRED), `EmailOtpIssuance` y `PasskeyCredential`. |
| `10_UC_Specific_Data_Model.md` | Definirá `passkey_credentials` (o extensión de enrollments con `usage`), keyspaces Redis (`passwordless:magiclink:*`, `passwordless:otp:*`, rate limits) y columnas de proveniencia passwordless. |
| `11_UC_Specific_Security_Threats.md` | Analizará magic link interception/prefetch abuse, OTP stuffing, passkey cloning, enumeración y cross-device hijacking. |
| `UC-15/01_Requirements_Specification.md` | Los métodos AAL1 pausan hacia UC-15; el mecanismo de clonación WebAuthn se reutiliza para passkeys. |
| `UC-14/01_Requirements_Specification.md` | El patrón de `session_code` one-time de 60s se reutiliza para el canje de tokens. |
| `UC-11/01_Requirements_Specification.md` | La maquinaria de familias (`CreateRoot`) y intentos de autenticación se reutiliza con proveniencia passwordless. |

---

## 9. Glosario de Términos Específicos de UC-16

| Término | Definición |
| :--- | :--- |
| **Passkey** | Credencial WebAuthn discoverable con userVerification obligatorio usada como primer factor; phishing-resistant por vinculación de origen y posesión criptográfica. |
| **UserVerification (UV)** | Verificación biométrica o PIN local del authenticator; requisito absoluto del login passkey (INV-105). |
| **Magic Link** | Enlace de un solo uso (token 256 bits, TTL ≤10 min) enviado al email verificado; consumo en dos pasos inmune a prefetch. |
| **Handoff Code** | Código de 6 dígitos mostrado en el dispositivo origen que autoriza la confirmación de un magic link desde otro dispositivo. |
| **Consumo en Dos Pasos** | GET que valida sin consumir + POST que consume; inmuniza el enlace contra prefetch, pre-render y scanners. |
| **Session Code** | Código one-time de 60s que puentea la emisión final de tokens por back-channel, evitando tokens en URLs (patrón UC-14). |
| **Canal Restringido** | Método OOB por email clasificado AAL1 que nunca satisface políticas AAL2+ sin segundo factor adicional. |
| **Stuffing Silencioso** | Patrón de solicitudes OOB masivas contenido por rate limit interno sin respuesta observable que revele el límite. |
| **Proveniencia Passwordless** | `auth_method = passwordless_{canal}` + `amr` específico + contexto de binding registrado en la familia de tokens. |
| **Discoverable Credential** | Credencial WebAuthn almacenada en el authenticator que permite autenticación sin identificador previo (username-less flow). |

---
*Fin del Documento 01_Requirements_Specification para UC-16*