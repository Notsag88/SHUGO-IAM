# UC-15: Multi-Factor Authentication (MFA) - Especificación de Requisitos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-15 - Multi-Factor Authentication (MFA - TOTP, WebAuthn/FIDO2, Push, Recovery) |
| **Documento:** | 01_Requirements_Specification.md |
| **Metodología:** | ICONIX (Deep Dive) + DDD + BDD + Zero Trust Architecture + Security by Design |
| **Estándares:** | NIST SP 800-63B (AAL2/AAL3, Verifier Impersonation Resistance, Restricted Authenticators), RFC 6238 (TOTP), W3C WebAuthn Level 2 / FIDO2, OWASP ASVS 4.0 (V2) + OWASP MFA Cheat Sheet, PSD2 SCA RTS (tenants fintech), GDPR Art. 32, SOC 2 CC6.1 |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 23/10/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 23/10/2026 | Gastón Ruiz | **Creación inicial.** Definición completa de la especificación analítica para el segundo factor en SHUGO. Cubre el catálogo de métodos con clasificación de assurance y resistencia a phishing (TOTP RFC 6238, WebAuthn/FIDO2 con detección de clonación por signature counter, Push con number matching anti-fatiga, Recovery Codes como break-glass, y Email OTP como método RESTRINGIDO por política), el ciclo de vida del `mfa_token` (MFA_PENDING) single-use que completa las pausas de UC-11/UC-13/UC-14, los flujos de enrollment con activación por prueba, el step-up en sesión para operaciones sensibles, los códigos de recuperación con revisión forzada, y el bypass de trusted device restringido exclusivamente a login (nunca a step-up). | Documento base para el paquete de diseño Tier 1 del UC-15. Es el caso de uso que **cierra todas las pausas de autenticación** de la plataforma: ningún flujo pausado por MFA (login local, refresh con frescura expirada, federación con assurance insuficiente, riesgo alto) puede completarse sin pasar por este UC, y ningún token elevado existe sin evidencia de método en `amr`. |

---

## 1. Introducción y Alcance

Este documento detalla la especificación de requisitos para el Caso de Uso **UC-15**, que representa el **segundo factor de autenticación** en el motor **KAGI** de SHUGO.

UC-15 no es un flujo aislado: es el **punto de completitud** de toda la arquitectura de autenticación de la plataforma. Tres UCs pausan sus flujos esperando al segundo factor (UC-11 tras password, UC-13 tras frescura expirada o riesgo, UC-14 tras assurance insuficiente del IdP), y un cuarto contexto lo invoca en sesión viva (step-up para operaciones sensibles exigido por KIZUNA). El diseño debe garantizar que:

1. **La pausa se complete exactamente una vez:** El `mfa_token` (claim `auth_level: MFA_PENDING`) es single-use, de vida corta, y vinculado al contexto pausado original y a su contexto de dispositivo.
2. **Cada método aporte evidencia clasificada:** No todos los segundos factores son iguales. WebAuthn es resistente a verifier impersonation (phishing); TOTP y Push no; Email OTP está restringido por NIST. La política del tenant debe poder exigir la clase de método, no solo "un método".
3. **El segundo factor sea atacable solo a costo detectable:** Replay de códigos TOTP, clonación de authenticators WebAuthn (signature counter), push fatigue, y brute-force del espacio de 6 dígitos tienen cada uno su control y su evento de detección.
4. **La elevación quede grabada en la sesión:** Un token elevado porta `auth_level` AAL2+, `amr` con el método usado y `step_up_at`; KIZUNA decide sobre evidencia, no sobre declaraciones del cliente.
5. **La pérdida del dispositivo no sea un callejón:** Recovery codes single-use con revisión forzada, y enrollment revocable por el usuario o por señales de riesgo.

### 1.1. Importancia Estratégica

El UC-15 es **el multiplicador de seguridad de toda la plataforma**:

- **Convierte AAL1 en AAL2/AAL3:** Sin MFA, un password robado o un token federado de IdP débil es una sesión completa. Con MFA, es solo media credencial.
- **Hace realizable el Zero Trust interno:** KIZUNA puede exigir step-up por scope sensible (ej. `payroll:write`, `iam:admin`) porque UC-15 provee la elevación verificable y auditable.
- **Sostiene el cumplimiento regulatorio:** PSD2 SCA, normas de supervisores financieros y marcos zero-trust gubernamentales exigen autenticación multifactor con características específicas (independencia de canales, resistencia a phishing) que el catálogo de métodos debe expresar.
- **Equilibra seguridad y fricción:** Trusted devices, push con number matching y políticas por tenant permiten reducir fricción legítima sin abrir superficies para el atacante.

### 1.2. Relación con Otros Casos de Uso

| UC | Relación | Descripción |
| :--- | :--- | :--- |
| **UC-11 (Login Local)** | Proveedor de pausa | Tras password válido con MFA requerido, emite `mfa_token` (MFA_PENDING). UC-15 completa y emite los tokens finales con `amr` elevado. |
| **UC-13 (Refresh)** | Proveedor de pausa | Frescura de MFA expirada (`mfa_freshness`) o riesgo en renovación pausan en `StepUpPending`; UC-15 reanuda la emisión. |
| **UC-14 (Federada)** | Proveedor de pausa | `acr` insuficiente del IdP o riesgo alto con política step-up pausan la emisión federada; UC-15 eleva el assurance local. |
| **UC-17 (Risk-Based)** | Consumidor bidireccional | UC-17 dispara step-ups; los eventos de UC-15 (método, dispositivo, fallos) alimentan el baseline de comportamiento. |
| **UC-20 / UC-22 / UC-23 (Sessions & Devices)** | Consumidor | Trusted devices y enrollments son visibles y revocables desde la gestión de sesiones/dispositivos. |
| **UC-18 (Recuperación Credenciales)** | Colateral | Un flujo de recuperación de credenciales debe invalidar enrollments MFA según política (attacker-with-email no debe heredar el segundo factor de la víctima). |
| **UC-42 (Account Lockout)** | Colateral | El brute-force de MFA comparte infraestructura de lockout con umbrales propios por método y por `mfa_token`. |
| **KIZUNA (Authorization)** | Consumidor de elevación | Exige step-up por scope sensible y valida `auth_level`/`step_up_at` del access token elevado. |

### 1.3. Catálogo de Métodos y Clasificación de Assurance

| Método | Estándar | AAL Máximo | Phishing-Resistant | Clasificación NIST 800-63B | Restricciones SHUGO |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **webauthn** | W3C WebAuthn L2 / FIDO2 | AAL3 (hardware + UV) / AAL2 | **Sí** (verifier impersonation resistant) | Multifactor con authenticator hardware/software protegido | Attestation policy por tenant (none/indirect/direct) |
| **totp** | RFC 6238 | AAL2 | No | Look-up secret + time-based | Skew ±1 step; replay por step; secreto ≥160 bits |
| **push** | OOB alineado a NIST §5.1.3.3 con number matching | AAL2 | No (mitiga phishing en tiempo real vía number matching) | Out-of-band push | Number matching obligatorio; rate limit anti push-fatigue por dispositivo |
| **recovery_code** | Look-up secret one-time | AAL2 (uso restringido) | No | One-time look-up secret | Break-glass: uso dispara revisión forzada y notificación; lote de 10, hash SHA256 en reposo |
| **email_otp** | OOB **restringido** | Step-up AAL1→AAL2 solo si política lo habilita | No | **RESTRICTED** (NIST desaconseja email como verifier OOB) | Default OFF; nunca satisface `require_phishing_resistant` ni AAL3; solo tenants con flag explícito |

### 1.4. Objetivos de Rendimiento

| Métrica | Objetivo | Justificación |
| :--- | :--- | :--- |
| **Challenge issue (p95)** | < 50 ms | Generación de challenge WebAuthn / dispatch de push (async) / OTP email (async). |
| **Verify TOTP (p95)** | < 30 ms | HMAC-SHA1 local + lookup de replay en Redis. |
| **Verify WebAuthn (p95)** | < 80 ms | Verificación de firma ES256/EdDSA + counter check + lookup de credential. |
| **Push status poll (p95)** | < 20 ms | Lectura de estado de challenge en Redis. |
| **Enrollment activation (p95)** | < 100 ms | Verificación de prueba + persistencia de enrollment. |
| **Disponibilidad** | 99.99% | UC-15 es el cuello de completitud de login, refresh y federación pausados. |
| **Push challenge TTL** | 60 s | Ventana de respuesta humana sin acumular challenges colgados. |

---

## 2. Descripción Analítica (ICONIX Clásico)

| Campo | Detalle |
| :--- | :--- |
| **ID Y NOMBRE:** | UC-15 - Multi-Factor Authentication (MFA) |
| **ESTADO:** | Preliminar |
| **DESCRIPCIÓN:** | El usuario completa un desafío de segundo factor presentado por un flujo pausado (login, refresh, federación) o por una exigencia de step-up en sesión viva. El sistema valida el `mfa_token` (single-use, TTL ≤5 min, vinculado al contexto pausado y al dispositivo de origen), aplica throttling de intentos, despacha el desafío al método seleccionado (TOTP, WebAuthn, Push con number matching, Recovery Code, o Email OTP restringido), verifica la prueba con los controles anti-replay/anti-clonación propios del método, y —solo ante éxito— **completa el flujo pausado original**: emite los tokens finales con `auth_level` elevado, `amr` con el método y `auth_time`, o reanuda la transacción de refresh/federación con assurance elevada. En step-up en sesión, emite un access token elevado vinculado al mismo `sid`/familia con `step_up_at`. |
| **ACTOR PRINCIPAL:** | Usuario Autenticado (parcialmente, en pausa MFA_PENDING) o Usuario en Sesión Viva (step-up). |
| **ACTOR SECUNDARIO:** | Dispositivo Authenticator (app TOTP, plataforma/roaming WebAuthn, app móvil Push), API Gateway, Redis (replay, challenges push, throttling), KAGAMI (enrollments, trusted devices), KAGE (Audit), Notification Service, UC-17 (Risk), KIZUNA (consumidor de elevación). |
| **PRECONDICIONES:** | 1. Existe un `mfa_token` vigente (pausa de UC-11/13/14) o una sesión viva elegible para step-up.<br>2. El usuario tiene al menos un enrollment ACTIVE compatible con la política del tenant, o se le ofrece enrollment inline (first-time).<br>3. El contexto de dispositivo coincide con el de la pausa (binding de `mfa_token`). |
| **PUNTOS DE EXTENSIÓN:** | Enrollment inline de primer uso; trusted-device bypass (solo login, nunca step-up); recovery code como break-glass; step-up reutilizable dentro de ventana `step_up_freshness` para operaciones sensibles sucesivas. |
| **CONDICIÓN:** | El flujo pausado queda completado con evidencia de método en `amr`/`acr` y `auth_time`, o el step-up queda grabado en un token elevado; todo intento (éxito o fallo) queda auditado. |

### 2.1. Escenario Principal (Happy Path) - Completitud de Pausa con TOTP

1. El cliente posee un `mfa_token` emitido por UC-11 (password verificado, MFA requerido).
2. El cliente envía `POST /auth/mfa/challenge` con `{ mfa_token, method: "totp" }`.
3. KAGI valida que el `mfa_token` esté vigente y no consumido (no se consume en challenge), y retorna metadatos del desafío (ventana de skew permitida, longitud de código).
4. El usuario ingresa el código de 6 dígitos de su app authenticator.
5. El cliente envía `POST /auth/mfa/verify` con `{ mfa_token, method: "totp", proof: { code } }`.
6. KAGI **consume** el `mfa_token` (single-use) y verifica throttling (intentos fallidos previos < umbral por token y por usuario/hora).
7. El verificador TOTP computa HMAC-SHA1 sobre el secreto del enrollment con ventana ±1 step, compara en tiempo constante, y chequea replay: el step usado no puede repetirse (Redis `mfa:totp:laststep:{enrollment_id}`).
8. **Éxito:** KAGI reanuda el contexto pausado: `AuthenticationAttempt.CompleteAuthentication(AAL2)` (UC-11) y emite Access Token + Refresh Token con `amr: ["pwd", "otp"]`, `acr` elevado y `auth_time = now`.
9. Se publican `MfaVerification.Succeeded` y los eventos de sesión del flujo original.
10. Se retorna HTTP 200 con los tokens finales. El cliente nunca re-intenta el flujo original: la pausa quedó completada aquí.

### 2.2. Escenarios Alternativos

**2.2.1 WebAuthn/FIDO2 (Ceremonia de Autenticación)**
1. `POST /auth/mfa/challenge { method: "webauthn" }` retorna `PublicKeyCredentialRequestOptions` con challenge aleatorio de ≥16 bytes, `rpId`, `allowCredentials` del usuario, `userVerification` según política.
2. El authenticator del dispositivo produce una assertion firmada (counter + clientDataJSON con origin).
3. `POST /auth/mfa/verify { proof: { assertion } }`:
   a. Validar origin contra `rpId` esperado y challenge contra el emitido (single-use en Redis).
   b. Verificar firma con la credential pública almacenada en el enrollment.
   c. **Signature counter:** si el counter reportado es no-cero y **no es estrictamente mayor** al almacenado → `MfaCloneDetected`: rechazo, revocación del enrollment según política, alerta SIEM crítica.
   d. Persistir nuevo counter.
4. Completitud del flujo pausado con `amr: ["pwd", "hwk"]` (o `["hwk"]` en step-up sin password) y AAL2/AAL3 según attachment + userVerification.

**2.2.2 Push con Number Matching (Anti Push-Fatigue)**
1. `POST /auth/mfa/challenge { method: "push" }` crea un challenge `ISSUED` en Redis (TTL 60s) con un número de 2 dígitos aleatorio, y despacha notificación push al dispositivo registrado (async, FCM/APNs).
2. La app móvil muestra el número esperado; el usuario selecciona el número correcto y autentica localmente (biometría/PIN del dispositivo).
3. El dispositivo responde al callback interno de KAGI con `{ challenge_id, selected_number, device_attestation }`; KAGI marca `APPROVED` solo si el número coincide y la attestation del dispositivo es válida.
4. El cliente poll-ea `GET /auth/mfa/push/status/{challenge_id}` hasta `APPROVED`/`DENIED`/`EXPIRED`, o presenta el approval en `POST /auth/mfa/verify`.
5. **Rate limit anti-fatiga:** máx. 3 challenges push por dispositivo por 10 min; al exceder, cooldown de 30 min + evento `MfaPushFatigueSuspected` (posible ataque de fatiga para que el usuario apruebe por cansancio).

**2.2.3 Step-Up en Sesión Viva (Operaciones Sensibles)**
1. KIZUNA rechaza una operación con scope sensible exigiendo step-up; el cliente invoca `POST /auth/mfa/stepup` con su access token actual.
2. KAGI valida la sesión (familia/sid) y emite un desafío vinculado al **mismo sid** (no a un mfa_token de login).
3. Tras verify exitoso, KAGI emite un **access token elevado** (misma familia, `auth_level` AAL2+, `amr` con método, `step_up_at = now`, TTL reducido ej. 15 min) y publica `MfaStepUp.Completed`.
4. La elevación es reutilizable dentro de `step_up_freshness` (policy, ej. 10 min) para operaciones sensibles sucesivas; después, nuevo step-up.

**2.2.4 Recovery Codes (Break-Glass)**
1. Generados en lote de 10 al activar cualquier enrollment primario; cada código ≥128 bits de entropía, mostrado una sola vez al usuario, almacenado como hash SHA256.
2. Uso: `method: "recovery_code"`; verificación por hash compare single-use (el código usado se marca consumido).
3. **Consecuencias obligatorias del uso:** evento `MfaRecovery.Used`, notificación al titular y a admins del tenant según política, y flag `mfa_review_required` en la identidad (el próximo login exige re-enrollment o confirmación de enrollments).

**2.2.5 Trusted Device Bypass (Solo Login)**
1. Tras un verify exitoso, si la política lo permite y el usuario opta in, KAGI emite un `trusted_device_token` (cookie httpOnly same-site strict o token en secure storage) con TTL policy (default 14 días) vinculado a device fingerprint + identidad.
2. En logins futuros desde ese dispositivo, UC-11 consulta el trusted device y **omite la pausa MFA** (registrando `amr` sin el método pero con `trusted_device: true`).
3. **Restricciones absolutas:** el bypass **nunca** aplica a step-up (INV-104), se revoca automáticamente ante señales de riesgo (UC-17), cambio de contexto anómalo, o revocación manual (UC-22); y no existe si el tenant exige `require_phishing_resistant` o AAL3.

### 2.3. Flujos de Excepción

**[4.1] mfa_token Reutilizado o Expirado**
- 4.1.1 El `mfa_token` ya fue consumido por un verify previo o superó su TTL (≤5 min).
- 4.1.2 Rechazar con error genérico `mfa_failed`; **invalidar la pausa original** (el attempt/tx pausado transiciona a FAILED) para impedir sondeo iterativo con tokens reciclados.
- 4.1.3 Auditar `MfaVerification.Failed{reason=token_reuse}`.

**[4.2] Throttling de Intentos por mfa_token y por Usuario**
- 4.2.1 ≥5 verify fallidos con el mismo `mfa_token`, o ≥10 fallos MFA por usuario/hora.
- 4.2.2 Invalidar la pausa / aplicar cooldown de step-up; evento `MfaVerification.Failed{reason=throttled}`; opcionalmente interlock con UC-42 (lockout de cuenta) según política del tenant.

**[4.3] Replay de Código TOTP (mismo step)**
- 4.3.1 El código corresponde al step actual pero ya fue aceptado para ese enrollment.
- 4.3.2 Rechazar (`mfa_failed` genérico); auditar con motivo interno `totp_replay`; no cuenta como fallo de brute-force (es replay, no guess), pero ≥2 replays seguidos elevan alerta de posible sesión compartida.

**[4.4] Clonación WebAuthn (Counter Regression)**
- 4.4.1 Signature counter reportado ≤ counter almacenado (con counter no-cero).
- 4.4.2 Rechazar, revocar el enrollment clonado, emitir `MfaCloneDetected` (SIEM crítica) y notificar al titular; los demás enrollments del usuario permanecen (el usuario puede re-autenticar con otro método).

**[4.5] Push Denegado o Expirado**
- 4.5.1 El usuario rechaza el push, o el challenge expira (60s), o el number matching falla.
- 4.5.2 El estado del challenge pasa a `DENIED`/`EXPIRED`; el verify correspondiente falla genérico; ≥3 denegaciones seguidas disparan cooldown anti-fatiga y evento.

**[4.6] Sin Enrollments Compatibles con la Política**
- 4.6.1 El tenant exige `require_phishing_resistant` y el usuario solo tiene TOTP.
- 4.6.2 El challenge retorna únicamente la opción de **enrollment inline de WebAuthn** (no permite verificar con TOTP); si el usuario abandona, la pausa expira naturalmente.

**[4.7] Enrollment con Prueba Fallida Repetida**
- 4.7.1 La activación de un enrollment pending falla 3 veces (código TOTP incorrecto, ceremony inválida).
- 4.7.2 El enrollment pending se cancela (TTL natural 10 min + cancelación explícita); el secreto/challenge descartados se auditan.

### 2.4. Postcondiciones

**De Éxito (completitud de pausa):**
1. Tokens finales emitidos con `auth_level` elevado, `amr` incluyendo el método, `acr` mapeado y `auth_time`.
2. La pausa original (attempt/tx/federación) queda completada **una sola vez**; el `mfa_token` queda consumido.
3. Eventos `MfaVerification.Succeeded` + eventos de sesión del flujo original publicados.
4. Opcionalmente, trusted device emitido si política y opt-in.

**De Éxito (step-up):**
1. Access token elevado con `step_up_at` vinculado al mismo `sid`/familia.
2. `MfaStepUp.Completed` publicado; KIZUNA acepta el scope sensible dentro de `step_up_freshness`.

**De Falla:**
1. Sin tokens emitidos ni elevación; la pausa permanece (hasta umbral de throttling que la invalida).
2. Intento auditado con motivo interno; respuesta al cliente siempre genérica `mfa_failed`.
3. Contadores de throttling y señales de detección (replay, clone, fatigue) actualizados.

---

## 3. Reglas de Negocio e Invariantes de Dominio (DDD)

| ID | Invariante / Regla | Descripción Técnica |
| :--- | :--- | :--- |
| **INV-97** | mfa_token Single-Use y Context-Bound | TTL ≤5 min; consumo atómico en verify; vinculado al contexto pausado (attempt/tx/federation) y al device fingerprint de origen. Segundo uso → rechazo + invalidación de la pausa. |
| **INV-98** | TOTP Replay Prevention por Step | Un código de un time-step dado se acepta una sola vez por enrollment (`mfa:totp:laststep:{enrollment_id}` en Redis). Skew máximo ±1 step, configurable por política. |
| **INV-99** | WebAuthn Clone Detection por Counter | Signature counter no-cero debe ser estrictamente creciente. Regresión → `MfaCloneDetected` + revocación del enrollment según política + alerta SIEM crítica. |
| **INV-100** | Clasificación de Resistencia a Phishing | Cada método porta clase (`phishing_resistant` solo webauthn). Tenants con `require_phishing_resistant` rechazan verify de métodos no-PR y ofrecen solo enrollment PR inline. |
| **INV-101** | Throttling de Intentos MFA | Máx. 5 fallos por `mfa_token` y 10 por usuario/hora (default). Al exceder: pausa invalidada / cooldown de step-up + evento; interlock opcional con UC-42. |
| **INV-102** | Recovery Codes Single-Use y Hasheados | Lote de 10, entropía ≥128 bits, mostrados una vez, almacenados SHA256. Uso → consumido + `MfaRecovery.Used` + notificación + flag `mfa_review_required`. |
| **INV-103** | Elevación Grabada y Vinculada al sid | Tokens elevados portan `auth_level`, `amr` con método, `step_up_at` y el mismo `sid`/familia. La elevación no es transferible entre familias ni entre usuarios. |
| **INV-104** | Trusted Device Bypass Restringido | Solo aplica a MFA de login; **nunca** a step-up; inexistente bajo `require_phishing_resistant`/AAL3; revocable por usuario (UC-22), por riesgo (UC-17) y por cambio anómalo de contexto. |
| **RN-32** | Catálogo y Política de Métodos | Métodos permitidos = intersección(enrollments ACTIVE del usuario, política del tenant). `email_otp` default OFF, nunca PR ni AAL3. `recovery_code` solo como break-glass con consecuencias obligatorias. |
| **RN-33** | Enrollment con Activación por Prueba | Todo enrollment nace `PENDING` y solo pasa a `ACTIVE` tras prueba exitosa (primer código TOTP, ceremony WebAuthn válida, ack de push). Pendings expiran en 10 min. |
| **RN-34** | Binding de Challenges | Challenges WebAuthn single-use y vinculados a `rpId`/origin; challenges Push vinculados a dispositivo registrado con number matching y TTL 60s; rate limit push por dispositivo (3/10 min). |
| **RN-35** | Evidencia en amr/acr y auth_time | Todo verify exitoso actualiza `amr` (pwd/otp/hwk/push), mapea `acr` según método y registra `auth_time`; los eventos alimentan auditoría (KAGE) y baseline de riesgo (UC-17). |

---

## 4. Criterios de Aceptación (BDD / Gherkin)

### Escenario 4.1: Completitud de pausa de login con TOTP
```gherkin
DADO que UC-11 emitió un mfa_token con auth_level MFA_PENDING para el usuario "user@khcs.com"
Y el usuario tiene un enrollment TOTP ACTIVE
CUANDO envía POST /auth/mfa/verify con mfa_token, method=totp y un código válido del step actual
ENTONCES el sistema retorna HTTP 200 con access_token y refresh_token
Y el access_token contiene auth_level=AAL2, amr=["pwd","otp"] y auth_time reciente
Y la pausa de UC-11 queda completada (el attempt transiciona a AUTHENTICATED)
Y se publica "MfaVerification.Succeeded" con method=totp
```

### Escenario 4.2: Replay de código TOTP dentro del mismo step
```gherkin
DADO que un código del step actual ya fue aceptado para el enrollment "enr-1"
CUANDO se envía el MISMO código nuevamente dentro de la ventana
ENTONCES el sistema rechaza con mfa_failed genérico
Y se audita motivo interno "totp_replay"
Y el intento NO consume el umbral de brute-force pero sí queda registrado
```

### Escenario 4.3: Clonación de authenticator WebAuthn detectada
```gherkin
DADO un enrollment WebAuthn con signature counter almacenado = 42
CUANDO una assertion presenta counter = 42 (o menor) con firma válida
ENTONCES el sistema rechaza la verificación
Y revoca el enrollment "enr-w-1" con motivo CLONE_DETECTED
Y publica "MfaCloneDetected" con alerta SIEM crítica
Y notifica al titular del posible authenticator clonado
```

### Escenario 4.4: Push con number matching y anti-fatiga
```gherkin
DADO un dispositivo push registrado "dev-7"
CUANDO se emiten 3 challenges push en 10 minutos y todos son ignorados o denegados
ENTONCES el 4to challenge es rechazado con cooldown de 30 minutos
Y se publica "MfaPushFatigueSuspected" con device_id y conteo
```

### Escenario 4.5: mfa_token reutilizado invalida la pausa
```gherkin
DADO que un mfa_token ya fue consumido en un verify exitoso
CUANDO un segundo verify presenta el mismo mfa_token
ENTONCES el sistema rechaza con mfa_failed
Y la pausa original queda invalidada (attempt/tx en FAILED)
Y se audita "MfaVerification.Failed" con reason=token_reuse
```

### Escenario 4.6: Step-up en sesión para scope sensible
```gherkin
DADO una sesión viva AAL1 con sid "fam-200"
CUANDO KIZUNA exige step-up para scope "payroll:write" y el usuario completa verify webauthn
ENTONCES el sistema emite un access_token elevado con auth_level=AAL2, sid="fam-200", step_up_at=now
Y KIZUNA acepta el scope dentro de la ventana step_up_freshness
Y se publica "MfaStepUp.Completed"
```

### Escenario 4.7: Recovery code single-use con revisión forzada
```gherkin
DADO un lote de recovery codes del usuario con 10 códigos activos
CUANDO se usa el código "RC-7" exitosamente
ENTONCES el sistema completa la verificación como break-glass
Y marca "RC-7" como consumido (segundo uso rechaza)
Y publica "MfaRecovery.Used" y notifica al titular
Y setea flag mfa_review_required en la identidad
```

### Escenario 4.8: Enrollment TOTP con activación por prueba
```gherkin
CUANDO el usuario autenticado crea un enrollment TOTP
ENTONCES el enrollment nace PENDING con secreto ≥160 bits y otpauth URI
Y al verificar el primer código correcto el enrollment pasa a ACTIVE
Y al fallar 3 activaciones el enrollment pending se cancela y el secreto se descarta
```

### Escenario 4.9: Trusted device bypass limitado a login
```gherkin
DADO un trusted_device_token vigente para "dev-3" y política de bypass habilitada
CUANDO el usuario loguea desde dev-3 con password correcto
ENTONCES UC-11 no pausa en MFA y emite tokens con trusted_device=true en amr-context
PERO cuando KIZUNA exige step-up en esa misma sesión
ENTONCES el bypass NO aplica y se requiere verify de un método ACTIVE
```

### Escenario 4.10: Política phishing-resistant sin método PR enrollado
```gherkin
DADO un tenant con require_phishing_resistant=true
Y un usuario con solo enrollment TOTP
CUANDO el flujo pausado llega a UC-15
ENTONCES el challenge NO ofrece totp como método verificable
Y ofrece enrollment inline de WebAuthn como única vía
Y si el usuario abandona, la pausa expira sin tokens emitidos
```

---

## 5. Eventos de Dominio Generados

### Evento 1: `MfaVerification.Succeeded`
```json
{
  "eventId": "uuid-v4",
  "eventType": "MfaVerification.Succeeded",
  "timestamp": "2026-09-08T10:15:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "method": "totp",
    "enrollmentId": "enr-1",
    "context": "login_pause",          // login_pause | refresh_pause | federation_pause | stepup
    "originFlow": "UC-11",
    "deviceFingerprint": "dev-hash-abc",
    "authLevelResulting": "AAL2",
    "phishingResistant": false
  }
}
```
**Consumidores:** KAGE (audit), UC-17 (baseline de comportamiento), métricas de adopción MFA, KIZUNA (cache de elevación por sid).

### Evento 2: `MfaVerification.Failed`
```json
{
  "eventId": "uuid-v4",
  "eventType": "MfaVerification.Failed",
  "timestamp": "2026-09-08T10:16:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "method": "totp",
    "reason": "invalid_code",           // invalid_code | totp_replay | token_reuse | throttled | clone | denied | expired
    "attemptCount": 3,
    "deviceFingerprint": "dev-hash-abc"
  }
}
```

### Evento 3: `MfaCloneDetected` (Evento de Seguridad)
```json
{
  "eventId": "uuid-v4",
  "eventType": "MfaCloneDetected",
  "timestamp": "2026-09-08T10:20:00Z",
  "tenantId": "khcs-001",
  "severity": "SECURITY",
  "payload": {
    "identityId": "user-uuid-123",
    "enrollmentId": "enr-w-1",
    "storedCounter": 42,
    "presentedCounter": 42,
    "actionTaken": "ENROLLMENT_REVOKED",
    "ipAddress": "203.0.113.77"
  }
}
```
**Consumidores:** SIEM (alerta crítica), KAGE, Notification Service, UC-17 (score de compromiso).

### Evento 4: `MfaEnrollment.Created` / `MfaEnrollment.Activated` / `MfaEnrollment.Revoked`
```json
{
  "eventId": "uuid-v4",
  "eventType": "MfaEnrollment.Activated",
  "timestamp": "2026-09-08T09:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "enrollmentId": "enr-1",
    "method": "totp",
    "activatedVia": "first_code_proof",
    "deviceName": "iPhone de Gastón"
  }
}
```

### Evento 5: `MfaStepUp.Completed`
```json
{
  "eventId": "uuid-v4",
  "eventType": "MfaStepUp.Completed",
  "timestamp": "2026-09-08T11:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "sid": "fam-200",
    "method": "webauthn",
    "requestedScope": "payroll:write",
    "stepUpAt": "2026-09-08T11:00:00Z",
    "freshnessWindowSeconds": 600
  }
}
```

### Evento 6: `MfaRecovery.Used` y `MfaPushFatigueSuspected`
```json
{
  "eventId": "uuid-v4",
  "eventType": "MfaRecovery.Used",
  "timestamp": "2026-09-08T12:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "codeIndex": 7,
    "remainingCodes": 2,
    "reviewRequired": true,
    "ipAddress": "198.51.100.23"
  }
}
```

---

## 6. Consideraciones de Arquitectura y Seguridad

### 6.1. UC-15 como Punto Único de Completitud (ADR de Diseño)

**Decisión:** El verify exitoso de UC-15 **emite los tokens finales** del flujo pausado (login, refresh, federación), en lugar de retornar un "receipt" que el cliente presentaría de vuelta al endpoint original.

**Justificación:**
- **Menos superficie de replay:** Un receipt re-presentable sería un artefacto de medio-credencial circulando por el cliente; el `mfa_token` single-use que ya existe basta.
- **Atomicidad de evidencia:** La emisión del token elevado ocurre en el mismo acto que la verificación del método: no puede existir un token AAL2 sin un `MfaVerification.Succeeded` correspondiente.
- **Simplicidad de cliente:** El cliente no orquesta re-intentos del flujo original; la pausa se completa en un solo endpoint con un solo contrato.

**Consecuencia:** UC-15 depende de los resolvers de pausa de UC-11/13/14 (puertos `IPausedFlowResolver`), manteniendo la dirección de dependencia hacia adentro: los flujos pausados exponen cómo completarse, UC-15 solo orquesta la evidencia.

### 6.2. Anti-Brute-Force del Espacio de 6 Dígitos

Un código TOTP tiene 10^6 combinaciones; con ventana ±1 son 3 códigos válidos por período de 30s. Los controles combinados:
1. **Throttling por mfa_token** (5 fallos) limita el sondeo por pausa.
2. **Throttling por usuario/hora** (10 fallos) limita el sondeo multi-pausa.
3. **Replay guard por step** impide que un código observado sirva dos veces.
4. **Rotación natural del step** (30s) limita la ventana de validez de cualquier código interceptado.
El costo esperado de un brute-force exitoso bajo estos controles excede cualquier ventana práctica; y cada intento fallido es un evento auditable con fingerprint de dispositivo.

### 6.3. Push Fatigue como Ataque de Primera Clase

El ataque de fatiga (bombardeo de pushes hasta que el usuario apruebe por cansancio) se mitiga con:
1. **Number matching obligatorio:** el usuario debe transcribir un número de 2 dígitos visible en el flujo origen; aprobar sin ver el número correcto es imposible.
2. **Rate limit por dispositivo** (3 challenges/10 min) + cooldown de 30 min al exceder.
3. **Evento `MfaPushFatigueSuspected`** que alimenta a UC-17 y al SIEM: una campaña de fatiga es visible antes de tener éxito.
4. **Denegaciones consecutivas** como señal: 3 denegaciones explícitas del usuario también disparan cooldown (el usuario está diciendo "no fui yo").

### 6.4. Recovery Codes: Break-Glass con Consecuencias

El uso de un recovery code es **siempre** un evento anómalo potencial (el usuario perdió su dispositivo o alguien más lo encontró). Por eso el uso no es silencioso: notificación al titular, evento de seguridad, y flag `mfa_review_required` que fuerza revisión de enrollments en el próximo login. El atacante que use un code robado no obtiene una sesión limpia: obtiene una sesión marcada y una víctima alertada.

### 6.5. Trusted Devices: Fricción Reducida sin Debilitar Step-Up

El bypass de trusted device existe para la fricción del login recurrente, pero **nunca** para step-up: una operación sensible exige evidencia fresca del segundo factor, incluso desde un dispositivo confiable. Esta asimetría (INV-104) es la que permite ofrecer UX de "recordar este dispositivo" sin degradar el control que protege las operaciones de alto impacto.

### 6.6. Email OTP como Método Restringido

Coherente con NIST SP 800-63B (verifiers OOB restringidos) y con la realidad del phishing de buzones: `email_otp` está **default OFF**, no cuenta como phishing-resistant, no satisface AAL3, y solo puede habilitarse por flag explícito de tenant para step-ups de bajo riesgo. Su existencia responde a realidades operativas (usuarios sin smartphone corporativo), no a una postura de seguridad.

---

## 7. Contratos de API (OpenAPI 3.0 Snippets)

### 7.1. Endpoint: POST /auth/mfa/challenge

```http
POST /v1/auth/mfa/challenge HTTP/1.1
Content-Type: application/json

{ "mfa_token": "mfa_pending_jwt...", "method": "webauthn" }
```
**Response 200 (webauthn):**
```json
{
  "challenge_id": "chg-uuid",
  "method": "webauthn",
  "publicKey": {
    "challenge": "base64url-16-bytes-min",
    "rpId": "shugo.io",
    "allowCredentials": [ { "type": "public-key", "id": "cred-1" } ],
    "userVerification": "required"
  }
}
```
**Response 200 (push):** `{ "challenge_id": "chg-uuid", "expires_in": 60, "number_matching_required": true }`

### 7.2. Endpoint: POST /auth/mfa/verify

```http
POST /v1/auth/mfa/verify HTTP/1.1
Content-Type: application/json

{
  "mfa_token": "mfa_pending_jwt...",
  "method": "totp",
  "challenge_id": "chg-uuid",          // requerido para webauthn/push
  "proof": { "code": "493021" }         // o { "assertion": {...} } o { "selected_number": 42 }
}
```
**Response 200 (completitud de pausa):**
```json
{
  "token_type": "Bearer",
  "access_token": "eyJ...amr:[pwd,otp]...auth_level:AAL2...",
  "refresh_token": "opaque...",
  "expires_in": 900,
  "session_meta": { "auth_level": "AAL2", "amr": ["pwd","otp"], "auth_time": 1757326500 }
}
```
**Response 400:** `{ "error": "mfa_failed", "error_description": "Multi-factor verification could not be completed." }` (genérico para todos los motivos internos).

### 7.3. Endpoints de Enrollment y Step-Up

```http
POST   /v1/auth/mfa/enrollments                 { method, device_name }      → pending payload
POST   /v1/auth/mfa/enrollments/{id}/activate   { proof }                    → enrollment ACTIVE
GET    /v1/auth/mfa/enrollments                                              → lista de métodos
DELETE /v1/auth/mfa/enrollments/{id}                                          → revocación (step-up per policy)
POST   /v1/auth/mfa/stepup                      { requested_scope? }         → challenge de step-up
GET    /v1/auth/mfa/push/status/{challenge_id}                               → ISSUED|APPROVED|DENIED|EXPIRED
POST   /v1/auth/mfa/recovery-codes/regenerate                                → nuevo lote (step-up requerido)
```

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `02_Use_Case_Diagram.md` | Mostrará Usuario/Dispositivo Authenticator como actores, los flujos challenge/verify/enrollment/step-up, y las relaciones con las pausas de UC-11/13/14 y con KIZUNA. |
| `04_Initial_Domain_Model.md` | Definirá `MfaEnrollment` (Entity con estado PENDING/ACTIVE/REVOKED), `MfaChallenge` (Entity efímera), `PausedFlowContext` (VO), y los services `TotpVerifier`, `WebAuthnVerifier`, `PushVerifier`, `StepUpIssuer`. |
| `05_State_Machine_Diagram.md` | Modelará las máquinas de `MfaEnrollment`, `MfaChallenge` (ISSUED→APPROVED/DENIED/EXPIRED) y del `mfa_token` (PENDING→CONSUMED/EXPIRED/INVALIDATED). |
| `10_UC_Specific_Data_Model.md` | Definirá `mfa_enrollments` (secretos TOTP cifrados, credentials WebAuthn, counters), `recovery_codes` (hashes), trusted devices y keyspaces Redis de replay/throttle/challenges. |
| `11_UC_Specific_Security_Threats.md` | Analizará TOTP replay/brute-force, clonación WebAuthn, push fatigue, phishing de pushes, robo de recovery codes, y abuso de trusted devices. |
| `UC-11/01 y UC-13/01 y UC-14/01` | Documentos hermanos que emiten los `mfa_token`/`StepUpRequirement` que este UC consume y completa. |

---

## 9. Glosario de Términos Específicos de UC-15

| Término | Definición |
| :--- | :--- |
| **mfa_token (MFA_PENDING)** | JWT de corta vida (≤5 min) emitido por un flujo pausado; single-use; vincula la pausa original, el dispositivo de origen y el assurance pendiente. |
| **Number Matching** | Mecanismo anti push-fatigue donde el flujo origen muestra un número de 2 dígitos que el usuario debe seleccionar en la app del dispositivo para aprobar. |
| **Signature Counter** | Contador monotónico del authenticator WebAuthn; su regresión con firma válida es evidencia de clonación (INV-99). |
| **Phishing-Resistant (PR)** | Clase de método que resiste verifier impersonation (WebAuthn/FIDO2); exigible por política de tenant (INV-100). |
| **Step-Up** | Elevación de assurance dentro de una sesión viva, grabada como `step_up_at` en un token elevado del mismo `sid` (INV-103). |
| **Trusted Device** | Credencial de dispositivo que omite MFA de login por política; nunca aplica a step-up (INV-104). |
| **Recovery Code** | Look-up secret one-time de break-glass con consecuencias obligatorias de revisión y notificación (INV-102). |
| **Replay Window (TOTP)** | Ventana de skew ±1 step con guard de un solo uso por step (INV-98). |
| **Punto de Completitud** | Decisión arquitectónica por la cual UC-15 emite los tokens finales del flujo pausado, en lugar de retornar receipts re-presentables. |
| **Restricted Method** | Método que NIST desaconseja como verifier (email OTP); habilitable solo por flag explícito y con assurance capado. |

---
*Fin del Documento 01_Requirements_Specification para UC-15*