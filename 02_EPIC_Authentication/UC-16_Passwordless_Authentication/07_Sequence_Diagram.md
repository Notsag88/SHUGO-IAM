# UC-16: Passwordless Authentication - Diagrama de Secuencia

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-16 - Passwordless Authentication (Passkeys + Magic Links + OTP Restringido) |
| **Documento:** | 07_Sequence_Diagram.md |
| **Metodología:** | ICONIX (Diagrama de Secuencia) + DDD + Security by Design (Two-Step Consumption & Silent Issuance) + Temporal Boundary Analysis |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 10/11/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 10/11/2026 | Gastón Ruiz | **Creación inicial.** Diagrama de Secuencia completo para UC-16 en cuatro partes: (1) la ceremonia passkey con la escalera de guardas (challenge → origin → UV → registro → firma → counter) y contención de clonación transaccional; (2) el ciclo magic link con emisión silenciosa, despacho async, validación GET sin consumo, confirmación POST con gate pre-consumo y carrera de confirmaciones resuelta por GETDEL; (3) el ciclo OTP con supersede atómico, dummy hash en ausencia de emisión, agotamiento de intentos y consumo single-use; y (4) el enrollment de passkey desde sesión autenticada con step-up condicional. Incluye presupuesto de timeouts por dependencia, análisis de carreras y trazas por canal. | Documento base del paquete de diseño Tier 1 del UC-16. Sirve como guion temporal exacto para la codificación de los flujos por canal, fijando que el consumo ocurre siempre después de la validación de guardas, que la emisión silenciosa responde antes de decidir el despacho, y que ninguna completitud ocurre sin pasar por la puerta única de assurance. |

---

## 1. Introducción y Propósito

El **Diagrama de Secuencia** para el UC-16 materializa temporalmente las colaboraciones definidas en el Diagrama de Robustez. Su propósito es:

1. **Ordenar las guardas por costo:** La ceremonia passkey ejecuta sus validaciones de la más barata a la más cara; el orden es la optimización y la defensa simultáneamente.
2. **Separar los tiempos de cada canal:** Passkey es síncrono (criptografía local), magic link es asíncrono mediado por email y humano, OTP es asíncrono con intentos acotados — cada uno tiene su presupuesto de timeout y su SLO correcto.
3. **Fijar el orden validación→consumo:** En magic link, el gate de binding/handoff se evalúa **antes** del GETDEL; en OTP, el compare ocurre **antes** del consumo. Un fallo de guarda nunca destruye el artefacto.
4. **Modelar las carreras reales:** Doble confirmación de enlace, doble solicitud de OTP (supersede), doble verify con el mismo challenge — cada una con su mecanismo de resolución observable.
5. **Documentar la convergencia de emisión:** Los tres canales terminan en el mismo emisor con la misma puerta de assurance y el mismo puente de session_code.

---

## 2. Diagrama de Secuencia #1: Ceremonia Passkey (Primer Factor)

```plantuml
@startuml UC-16_Secuencia_Passkey
<style>
diagram {
  BackgroundColor #FEFEFE
}
sequence {
  ArrowThickness 2
  ArrowColor #2C3E50
  ParticipantPadding 10
  BoxPadding 8
}
</style>

title UC-16: Ceremonia Passkey - Escalera de Guardas y Emisión Directa

actor "Usuario" as User
participant "Authenticator\nde Plataforma" as Auth
participant "Cliente\n(SPA/Mobile)" as Client
participant "API Gateway /\nPasswordless API" as GW

box "Motor KAGI - Passwordless Core" #E3F2FD
participant "Passwordless\nHandler" as Handler
participant "Passkey\nCeremony" as Ceremony
participant "Session\nIssuer" as Issuer
end box

box "Infraestructura & Contextos" #FFF3E0
participant "Redis\n(challenges/codes)" as Redis
participant "KAGAMI\n(credenciales/familias)" as Kagami
participant "UC-17\nRisk" as Risk
participant "UC-15\nPause Broker" as Pause
participant "Auth Outbox\n(KAGE)" as Outbox
end box

== FASE 1: OPTIONS (Front-Channel Initiation) ==
User -> Client : 1. "Entrar con passkey"\n(conditional UI / autofill)
Client -> GW : 2. POST /passwordless/webauthn/options\n{ username_hint? }
GW -> Handler : 3. OptionsCommand
Handler -> Ceremony : 4. BuildOptions(hint, policy)
Ceremony -> Redis : 5. SET challenge single-use\n(≥16 bytes, TTL 5 min)
Ceremony -> Kagami : 6. (opcional) credenciales sugeridas\npor hint — respuesta idéntica\ncon o sin resultados
Ceremony --> GW : 7. PublicKeyRequestOptions\n(userVerification = "required")
GW --> Client : 8. Options

== FASE 2: CEREMONIA (Fuera de nuestro SLO) ==
Client -> Auth : 9. navigator.credentials.get()
User -> Auth : 10. Biometría / PIN local (UV)
Auth -> Client : 11. Assertion firmada\n(UV=1, counter incremental)

== FASE 3: VERIFY (Escalera de Guardas) ==
Client -> GW : 12. POST /passwordless/webauthn/verify\n{ assertion }
GW -> Handler : 13. VerifyCommand
Handler -> Ceremony : 14. VerifyAsync(assertion, policy)

Ceremony -> Redis : 15. GETDEL challenge (single-use)
alt Challenge ausente / expirado
  Ceremony --> Handler : 16a. Failed(stage=challenge)
end

Ceremony -> Ceremony : 17. Validar origin / rpId exactos
alt Origin inválido
  Ceremony --> Handler : 18a. Failed(stage=origin)
end

Ceremony -> Ceremony : 19. **Verificar flag UV**\n(antes de criptografía y lookups)
alt UV = 0
  Ceremony --> Handler : 20a. Failed(stage=uv_missing)\n[INV-105: jamás AAL2+ sin UV]
end

Ceremony -> Kagami : 21. FindActive(tenant, credentialId)\n→ identidad + clave pública + counter
alt Credencial desconocida / revocada
  Ceremony --> Handler : 22a. Failed genérico\n(sin revelar existencia del credentialId)
end

Ceremony -> Ceremony : 23. Verificar firma de la assertion
alt Firma inválida
  Ceremony --> Handler : 24a. Failed(stage=signature)
end

Ceremony -> Kagami : 25. UPDATE counter (fail-secure)
alt Counter regression (clonación)
  Ceremony -> Kagami : 26b. REVOKE credencial\n(CLONE_DETECTED) [misma TX]
  Ceremony -> Outbox : 27b. PasskeyCloneDetected\n(SIEM crítica)
  Ceremony --> Handler : 28b. Failed(stage=clone)
end

Ceremony -> Risk : 29. Evaluar delta de contexto
alt Score alto: política revoke
  Ceremony --> Handler : 30a. Failed(stage=risk_denied)
else Score alto: política step-up
  Ceremony -> Pause : 30b. PauseForMfa → mfa_token
  Ceremony --> Handler : 31b. StepUp(mfa_token)
end

Ceremony -> Issuer : 32. Proveniencia passkey\n(AAL2/3, amr=["hwk","user"])

== FASE 4: EMISIÓN Y CANJE ==
Issuer -> Kagami : 33. CreateRoot familia\n(auth_method=passwordless_webauthn)
Issuer -> Redis : 34. SET passwordless:code\n(single-use, TTL 60s)
Issuer -> Outbox : 35. Passwordless.SessionIssued\n+ WebAuthnAuthenticated
Issuer --> Handler : 36. session_code
Handler --> GW : 37. { session_code, expires_in: 60 }
GW --> Client : 38. session_code

Client -> GW : 39. POST /passwordless/token\n{ session_code }
GW -> Handler : 40. RedeemCommand
Handler -> Redis : 41. GETDEL passwordless:code
Handler -> Kagami : 42. Emitir Access + Refresh\n(sobre la familia creada)
Handler --> GW : 43. Tokens
GW --> Client : 44. Sesión passwordless establecida

@enduml
```

---

## 3. Diagrama de Secuencia #2: Ciclo Completo de Magic Link

```plantuml
@startuml UC-16_Secuencia_MagicLink
<style>
diagram { BackgroundColor #FEFEFE }
sequence { ArrowThickness 2; ArrowColor #2C3E50; ParticipantPadding 10 }
</style>

title UC-16: Magic Link - Emisión Silenciosa, Dos Pasos, Carrera de Confirmaciones

actor "Usuario" as User
participant "Cliente\n(SPA/Mobile)" as Client
participant "API Gateway /\nPasswordless API" as GW
participant "Proveedor\nde Email" as EmailProv

box "Motor KAGI - Passwordless Core" #FFF3E0
participant "Passwordless\nHandler" as Handler
participant "Magic Link\nFlow" as Flow
participant "Abuse\nControl" as Abuse
participant "Session\nIssuer" as Issuer
end box

box "Infraestructura & Contextos" #E8F5E9
participant "Redis\n(tokens/codes)" as Redis
participant "KAGAMI\n(identidades/familias)" as Kagami
participant "Email\nDispatcher" as EmailDisp
participant "UC-17 Risk" as Risk
participant "UC-15\nPause Broker" as Pause
participant "Auth Outbox" as Outbox
end box

== FASE 1: EMISIÓN SILENCIOSA ==
User -> Client : 1. Ingresa email
Client -> GW : 2. POST /passwordless/magic-link\n{ email, redirect_uri }
GW -> Handler : 3. RequestCommand
Handler -> Flow : 4. RequestAsync(email, context)
Flow -> Abuse : 5. EvaluateIssuance\n(rate por email/IP, silencioso)
alt Decisión = Suppress (rate suspendido)
  Abuse --> Flow : 6a. SuppressSilently
  Flow --> Handler : 7a. PublicIssuanceResult.Accepted\n(IDÉNTICO al camino con despacho)
end
Flow -> Kagami : 8. Lookup identidad por email\n→ activa? email verificado?\ncanal habilitado?
alt Identidad inexistente / canal deshabilitado
  Flow --> Handler : 9a. PublicIssuanceResult.Accepted\n(IDÉNTICO; métrica unknown_email)
end
Flow -> Flow : 10. MagicLinkToken.Issue\n(256 bits + binding + handoff?)
Flow -> Redis : 11. SET magiclink:{token_hash}\n(TTL ≤10 min)
Flow ->> EmailDisp : 12. DispatchMagicLinkAsync\n(fire-and-forget, 1 retry)
EmailDisp ->> EmailProv : 13. Transporte del email
note right of Flow
  Si el despacho falla:
  métrica dispatch_failed.
  El token ya está persistido;
  el usuario re-solicita (supersede
  natural por nueva emisión).
end note
Flow -> Outbox : 14. MagicLinkIssued
Flow --> Handler : 15. PublicIssuanceResult.Accepted\n(+ handoff code si cross-device)
Handler --> GW : 16. 202 genérico
GW --> Client : 17. "Si la cuenta existe,\nse envió un enlace"

== FASE 2: ENTREGA Y CLIC (Fuera de nuestro SLO) ==
EmailProv -> User : 18. Email recibido\n(latencia del proveedor)
User -> Client : 19. Clic en el enlace\n(mismo u otro dispositivo)

== FASE 3: VALIDACIÓN GET (Pura, sin consumo) ==
Client -> GW : 20. GET /passwordless/link?token=...
GW -> Handler : 21. ValidateCommand
Handler -> Flow : 22. ValidateAsync(token, context)
Flow -> Redis : 23. **Peek** magiclink:{hash}\n(GET, sin mutación)
alt Token ausente / expirado
  Flow --> Handler : 24a. LinkValidationView.Invalid
end
Flow -> Flow : 25. ValidateBinding(context)\n(pura: SameDevice / HandoffRequired / Mismatch)
Flow --> Handler : 26. View: confirmación same-device,\nconfirmación con handoff,\no device mismatch
Handler --> GW : 27. Página de confirmación
note right of Flow
  **Inmunidad a prefetch:**
  Este GET puede ejecutarse
  infinitas veces (scanners,
  pre-render, re-aperturas)
  sin efecto sobre el token.
end note

== FASE 4: CONFIRMACIÓN POST (Gate → Consumo) ==
User -> Client : 28. Confirmar (+ handoff code si aplica)
Client -> GW : 29. POST /passwordless/link/confirm\n{ token, handoff_code? }
GW -> Handler : 30. ConfirmCommand
Handler -> Flow : 31. ConfirmAsync(token, handoff, context)
Flow -> Redis : 32. Peek magiclink:{hash}
Flow -> Flow : 33. CanBeConfirmedBy:\nTTL ∧ (binding ∨ handoff válido)
alt Gate denegado (mismatch / handoff inválido)
  Flow -> Outbox : 34a. MagicLinkFailed{stage}
  Flow --> Handler : 35a. LinkInvalid\n(el enlace NO se consume:\nreintento legítimo posible)
end
Flow -> Redis : 36. **GETDEL** magiclink:{hash}\n(consumo single-writer)
alt Carrera: otra confirmación ganó
  Flow -> Flow : 37b. Señal de replay\n(2ª ocurrencia → alerta SIEM)
  Flow --> Handler : 38b. LinkInvalid(stage=consumed)
end
Flow -> Risk : 39. Evaluar riesgo\n(contexto confirm vs solicitud)
alt Riesgo alto
  Flow --> Handler : 40a. RiskDenied / StepUp
end
Flow -> Issuer : 41. Proveniencia magic_link (AAL1)

== FASE 5: PUERTA DE ASSURANCE Y EMISIÓN ==
alt required_level ≥ AAL2
  Issuer -> Pause : 42a. PauseForMfa\n(mfa_token → UC-15)
  Issuer --> Handler : 43a. PausedForMfa
  Handler --> GW : 44a. { mfa_required, mfa_token }
else AAL1 suficiente
  Issuer -> Kagami : 42b. CreateRoot familia\n(amr=["otk"], cross_device flag)
  Issuer -> Redis : 43b. SET passwordless:code (60s)
  Issuer -> Outbox : 44b. MagicLinkConsumed\n+ SessionIssued
  Issuer --> Handler : 45b. session_code
  Handler --> GW : 46b. 302 redirect_uri?session_code=...
  GW --> Client : 47b. Canje back-channel\n→ tokens (sesión AAL1)
end

@enduml
```

---

## 4. Diagrama de Secuencia #3: Ciclo OTP de Email (Restringido)

```plantuml
@startuml UC-16_Secuencia_OTP
<style>
diagram { BackgroundColor #FEFEFE }
sequence { ArrowThickness 2; ArrowColor #2C3E50; ParticipantPadding 12 }
</style>

title UC-16: OTP de Email - Supersede, Dummy Hash y Agotamiento

actor "Usuario" as User
participant "Cliente" as Client
participant "API Gateway /\nPasswordless API" as GW

box "Motor KAGI - Passwordless Core" #E8F5E9
participant "Passwordless\nHandler" as Handler
participant "OTP Flow" as Flow
participant "Abuse\nControl" as Abuse
participant "Session\nIssuer" as Issuer
end box

box "Infraestructura & Contextos" #FFF3E0
participant "Redis" as Redis
participant "KAGAMI" as Kagami
participant "Email\nDispatcher" as EmailDisp
participant "UC-17 Risk" as Risk
participant "Auth Outbox" as Outbox
end box

== EMISIÓN CON SUPERSEDE ==
Client -> GW : 1. POST /passwordless/otp/request { email }
GW -> Handler : 2. RequestCommand
Handler -> Flow : 3. RequestAsync
Flow -> Abuse : 4. EvaluateIssuance\n(≤3 emisiones/hora, silencioso)
Flow -> Kagami : 5. Lookup identidad\n(solo si Dispatch)
alt Identidad válida Y canal habilitado
  Flow -> Flow : 6. OtpIssuance.Issue\n(8 dígitos CSPRNG)
  Flow -> Redis : 7. **SUPERSEDE + SET** otp:{email_hash}\n(reemplaza emisión previa;\nTTL 5m, intentos=0)
  Flow ->> EmailDisp : 8. DispatchOtpAsync (código, sin enlace)
  Flow -> Outbox : 9. OtpIssued
end
Flow --> Handler : 10. PublicIssuanceResult.Accepted\n(idéntico en todos los caminos)
Handler --> GW : 11. 202 genérico

== VERIFICACIÓN CON INTENTOS ACOTADOS ==
User -> Client : 12. Transcribe el código
Client -> GW : 13. POST /passwordless/otp/verify\n{ email, code }
GW -> Handler : 14. VerifyCommand
Handler -> Flow : 15. VerifyAsync
Flow -> Abuse : 16. EvaluateVerify\n(rate de verificación)
Flow -> Redis : 17. GetActive(email_hash)
alt Sin emisión activa
  Flow -> Flow : 18a. **Dummy hash compare**\n(igualar timing del camino real)
  Flow --> Handler : 19a. Failed(invalid_code)
end
Flow -> Flow : 20. TryVerify:\n1º expirada? 2º agotada? 3º compare
alt Código incorrecto (intentos < 5)
  Flow -> Redis : 21b. INCR intentos
  Flow -> Outbox : 22b. OtpFailed{invalid_code}
  Flow --> Handler : 23b. Failed (genérico)
else Quinto intento fallido
  Flow -> Redis : 21c. Invalidar emisión
  Flow -> Outbox : 22c. OtpFailed{attempts_exhausted}
  Flow --> Handler : 23c. Failed (genérico)
else Código correcto
  Flow -> Redis : 21d. **GETDEL** emisión (single-use)
  Flow -> Risk : 22d. Evaluar riesgo
  Flow -> Issuer : 23d. Proveniencia email_otp (AAL1)
  Issuer --> Handler : 24d. Puerta de assurance:\nemisión directa o pausa UC-15\n(idéntica al Diagrama #2, Fase 5)
end

@enduml
```

---

## 5. Diagrama de Secuencia #4: Enrollment de Passkey (Sesión Autenticada)

```plantuml
@startuml UC-16_Secuencia_Enroll
<style>
diagram { BackgroundColor #FEFEFE }
sequence { ArrowThickness 2; ArrowColor #2C3E50; ParticipantPadding 12 }
</style>

title UC-16: Enrollment de Passkey - Expansión de Superficie Protegida

actor "Usuario" as User
participant "Authenticator" as Auth
participant "Cliente" as Client
participant "API Gateway /\nPasswordless API" as GW
participant "Passwordless\nHandler" as Handler
participant "Passkey\nEnrollment" as Enroll
participant "Redis" as Redis
participant "KAGAMI" as Kagami
participant "Auth Outbox" as Outbox

User -> Client : 1. "Añadir passkey" (configuración de cuenta)
Client -> GW : 2. POST /passwordless/webauthn/enroll\n(Authorization: Bearer sesión activa)
GW -> Handler : 3. EnrollCommand
Handler -> Handler : 4. Validar assurance de la sesión\ncontra política de enrollment
alt Assurance insuficiente
  Handler --> GW : 5a. 403 step_up_required\n(requiere step-up vía UC-15)
end
Handler -> Enroll : 6. BuildRegistrationOptions(session, policy)
Enroll -> Redis : 7. SET enrollment challenge\n(single-use, TTL 5 min)
Enroll --> GW : 8. PublicKeyCreationOptions\n(residentKey="required",\nUV="required", attestation por política)
GW --> Client : 9. Options

Client -> Auth : 10. navigator.credentials.create()
User -> Auth : 11. Biometría / PIN (UV)
Auth -> Client : 12. Attestation + credencial discoverable

Client -> GW : 13. POST /passwordless/webauthn/enroll/verify\n{ attestation }
GW -> Handler : 14. CompleteCommand
Handler -> Enroll : 15. CompleteAsync(attestation, session)
Enroll -> Redis : 16. GETDEL enrollment challenge
Enroll -> Enroll : 17. Validar origin/rpId + UV +\nattestation según política
alt Attestación inválida / UV ausente
  Enroll --> Handler : 18a. Failed (genérico)
end
Enroll -> Kagami : 19. ¿credentialId ya registrado en el tenant?
alt Duplicado
  Enroll --> Handler : 20a. Failed(credential_exists)\n(sin revelar a qué identidad pertenece)
end
Enroll -> Kagami : 21. INSERT credencial\n(usage=first_factor, counter=0)
Enroll -> Outbox : 22. Passkey.Enrolled
Enroll --> Handler : 23. EnrollmentResult.Active
Handler --> GW : 24. 200 enrolled\n(visible/revocable en UC-22)

@enduml
```

---

## 6. Análisis Detallado del Flujo

### 6.1. Diagrama #1: La Escalera de Guardas Passkey

El verify passkey (pasos 15-25) ejecuta las guardas en orden estricto de costo creciente:

1. **Challenge GETDEL (paso 15):** Mata ceremonias replayadas sin tocar nada más.
2. **Origin/rpId (17):** Comparación de strings; mata assertions de otros orígenes.
3. **UV (19):** Lectura de un flag; mata assertions sin verificación **antes** de gastar criptografía o lookups.
4. **Registro (21):** Lookup indexado; resuelve identidad desde KAGAMI, nunca desde la assertion.
5. **Firma (23):** El único paso criptográfico del camino.
6. **Counter (25):** Detección de clonación con contención transaccional.

**Justificación:** Un flood de assertions basura muere en los peldaños 1-3 (~2-3 ms); solo las assertions estructuralmente plausibles alcanzan la criptografía y la base de datos.

### 6.2. Diagrama #2: Los Cinco Momentos del Magic Link

1. **Emisión silenciosa (pasos 1-17):** El 202 se construye antes de que termine la decisión interna; el despacho (12-13) es fire-and-forget fuera de la respuesta. La identidad inexistente y la rate-suspendida producen exactamente la misma secuencia observable.
2. **Entrega (18-19):** Fuera de todo SLO; el TTL de 10 min es el horizonte duro.
3. **Validación pura (20-27):** Peek + ValidateBinding sin mutación; cualquier número de GETs es inofensivo.
4. **Gate antes del consumo (31-35):** Binding/handoff se evalúan con el token aún vivo; el fallo preserva el enlace para reintento legítimo.
5. **Consumo y emisión (36-47):** GETDEL como único escritor terminal; riesgo; puerta de assurance; emisión o pausa.

### 6.3. Diagrama #3: Las Tres Defensas del OTP

1. **Supersede (paso 7):** La nueva emisión reemplaza atómicamente la previa — nunca coexisten dos códigos activos.
2. **Dummy hash (pasos 18a-19a):** La verificación contra email sin emisión iguala el timing del camino real, cerrando el oráculo en el verify (no solo en el request).
3. **Agotamiento ordenado (paso 20):** Expiración → agotamiento → compare; el código expirado no gasta intentos y el agotado no revela por timing si el código presentado era correcto.

### 6.4. Diagrama #4: Enrollment como Expansión Protegida

El enrollment exige sesión autenticada con assurance suficiente (step-up condicional), challenge single-use, UV en la ceremonia de registro, y verificación de duplicado de credentialId **sin revelar** a qué identidad pertenece el duplicado. La credencial nace con `usage=first_factor` y counter inicializado.

---

## 7. Flujos Alternativos y Excepciones

| # | Escenario | Secuencia abreviada | Resultado |
| :--- | :--- | :--- | :--- |
| 7.1 | **Assertion sin UV** | Paso 19 → 20a | `uv_missing`; sin lookup de credencial ni criptografía gastada |
| 7.2 | **CredentialId desconocido** | Paso 21 → 22a | Rechazo genérico; no revela si el credentialId existe en otro tenant |
| 7.3 | **Clonación de passkey** | Paso 25 → 26b-28b | Revocación + `PasskeyCloneDetected` en la misma TX; sesión denegada |
| 7.4 | **Challenge reutilizado** | Paso 15 falla | `stage=challenge`; una ceremonia por challenge |
| 7.5 | **Enlace expirado** | Paso 23 → 24a | Página de expirado con re-solicitud; sin efectos |
| 7.6 | **Doble confirmación (carrera)** | Paso 36: un GETDEL gana | Ganador emite; perdedor → `stage=consumed` + señal de replay |
| 7.7 | **Device mismatch sin handoff** | Paso 33 → 34a-35a | Enlace intacto; usuario reintenta desde dispositivo origen |
| 7.8 | **Handoff incorrecto** | Paso 33 → 34a | Rechazo sin consumo; intentos de handoff acotados |
| 7.9 | **Riesgo denegado en confirmación** | Paso 39 → 40a | `MagicLinkFailed{risk_denied}`; token ya consumido (la prueba era válida; el contexto no) |
| 7.10 | **OTP agotado** | Paso 20 → 21c-23c | Emisión invalidada; re-emisión sujeta a 3/hora |
| 7.11 | **Email inexistente (OOB)** | Paso 8 → 9a | 202 idéntico al caso con despacho; métrica interna `unknown_email` |
| 7.12 | **Redis caído** | Cualquier paso con store | 503 fail-secure; ningún artefacto consumido ni emitido; pausas y solicitudes preservadas donde sea posible |
| 7.13 | **Despacho de email fallido** | Paso 12/8 falla | Token persistido; métrica `dispatch_failed`; usuario re-solicita |
| 7.14 | **Pause Broker caído con required ≥ AAL2** | Paso 42a falla | 503: nunca emitir AAL1 donde la política exige AAL2 |

---

## 8. Consideraciones de Arquitectura (Security & Performance)

### 8.1. Presupuesto de Timeouts por Dependencia

| Dependencia | Operación | Timeout / Latencia esperada | Comportamiento de fallo |
| :--- | :--- | :--- | :--- |
| **Redis** | GETDEL/Peek/SET de tokens, challenges, codes | 50 ms | Fail-secure: 503, sin efectos laterales |
| **KAGAMI** | Lookup identidad/credencial, UPDATE counter, CreateRoot | 200 ms c/u | Fail-secure: 503; counter update fallido bloquea la emisión passkey |
| **Firma WebAuthn** | Verificación ECDSA/EdDSA | ~1 ms | Fallo = prueba inválida |
| **Email Dispatcher** | Transporte SES/SMTP | 2 s, 1 retry, fire-and-forget | Métrica; el 202 ya fue retornado |
| **UC-17 Risk** | Evaluación de contexto | 100 ms | Fail-mode por política (`open` / `stepup`) |
| **TTLs de artefactos** | Challenge 5m / Magic link ≤10m / OTP 5m / Session code 60s | — | Expiración natural; purge sin jobs |

**SLOs locales (excluyen email y decisión humana):** options p95 < 50 ms; passkey verify p95 < 80 ms; magic confirm p95 < 150 ms; OTP verify p95 < 50 ms; redeem p95 < 50 ms.

### 8.2. Carreras y sus Mecanismos de Resolución

| Carrera | Mecanismo | Comportamiento observable |
| :--- | :--- | :--- |
| Doble confirmación de magic link | GETDEL atómico | Un ganador emite; el otro recibe `consumed` + señal de replay |
| Doble solicitud de magic link | Segunda emisión supersede la primera | El enlace anterior queda inválido al confirmarse el nuevo; ambos 202 idénticos |
| Doble solicitud de OTP | SUPERSEDE + SET atómico | Exactamente un código activo por identidad |
| Doble verify del mismo challenge passkey | GETDEL del challenge | Una ceremonia gana; la otra falla en `stage=challenge` |
| Doble canje de session_code | GETDEL del code | Un canje emite tokens; el segundo falla genérico |
| Prefetch del enlace (scanners) | GET sin mutación | Sin carrera: la validación es idempotente por construcción |

### 8.3. Identidad Desde el Registro, Nunca Desde la Assertion

El paso 21 del Diagrama #1 resuelve la identidad por `(tenant, credentialId)` contra KAGAMI; el `userHandle` de la assertion se verifica **contra** el registro (paso implícito en `FindActive`). Una assertion forjada con userHandle ajeno y credentialId propio no produce sesión de otra identidad — la fuente de verdad es el registro, no el authenticator.

### 8.4. Riesgo en la Posición Correcta por Canal

*   **Passkey (paso 29):** Tras la verificación completa de la prueba; el riesgo es la única causa de pausa/denegación para un canal que ya demostró AAL2+.
*   **Magic link (paso 39):** Compara el contexto de la confirmación contra el de la solicitud (device, geo, tiempo transcurrido) — un enlace solicitado en una red y confirmado en otra es señal de intercepción.
*   **OTP (paso 22d):** Riesgo sobre el contexto del verify.

### 8.5. Trazas por Canal (OpenTelemetry)

```text
POST /passwordless/webauthn/verify [~55ms]
├── redis.challenge.getdel [2ms]
├── guard.origin_uv [0.2ms]
├── kagami.credential.fetch [3ms]
├── webauthn.verify_signature [1ms]
├── kagami.counter.update [4ms]
├── risk.evaluate [30ms]
└── issuer.complete [15ms]

POST /passwordless/magic-link [~18ms, respuesta antes del despacho]
├── abuse.evaluate [2ms]
├── kagami.identity.lookup [5ms]
├── redis.token.set [2ms]
├── outbox.insert [4ms]
└── email.dispatch [async, fuera de la respuesta]

POST /passwordless/link/confirm [~70ms]
├── redis.token.peek [2ms]
├── gate.binding_handoff [0.5ms]
├── redis.token.getdel [2ms]
├── risk.evaluate [30ms]
└── issuer.or_pause [20ms]

POST /passwordless/otp/verify [~40ms]
├── redis.issuance.get [2ms]
├── otp.compare [0.1ms]
├── redis.issuance.getdel [2ms]
├── risk.evaluate [30ms]
└── issuer.or_pause [5ms]
```

---

## 9. Trazabilidad con Otros Documentos del Paquete

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Los escenarios 2.1-2.2.4 y 4.1-4.10 se secuencian aquí paso a paso; las excepciones 4.1-4.7 corresponden a las filas de la tabla §7. |
| `03_Activity_Diagram.md` | Los cuatro diagramas de actividad se proyectan en los cuatro diagramas de secuencia con los mismos carriles y decisiones. |
| `05_State_Machine_Diagram.md` | Las transiciones ISSUED→CONSUMED (GETDEL paso 36/21d), ACTIVE→EXHAUSTED (paso 21c) y ACTIVE→REVOKED/CLONE (pasos 26b-28b) ocurren exactamente en estos mensajes. |
| `06_Robustness_Diagram.md` | Este diagrama define el orden temporal de las 80 colaboraciones que el de robustez define estructuralmente. |
| `08_Final_Class_Design.md` | Definirá las firmas exactas con `CancellationToken`, el record `PasswordlessTimeouts` y el colapso `PasswordlessOutcome.Match` en el boundary. |
| `10_UC_Specific_Data_Model.md` | Los GETDEL/Peek/SUPERSEDE de estos diagramas consumen los keyspaces `passwordless:magiclink:*`, `passwordless:otp:*`, `passwordless:challenge:*` y `passwordless:code:*`. |
| `11_UC_Specific_Security_Threats.md` | El orden gate→consumo (§6.2), el dummy hash (§6.3) y la escalera de guardas (§6.1) son las mitigaciones ejecutables de intercepción, enumeración y flood. |
| `12_UC_Specific_Performance_Benchmarks.md` | El presupuesto de timeouts (§8.1) y las trazas (§8.5) son la base de los SLOs y escenarios k6 con despacho simulado. |

---

## 10. Notas de Implementación

### 10.1. Configuración Tipada de Timeouts (C#)

```csharp
public sealed record PasswordlessTimeouts
{
    public TimeSpan RedisOperation     { get; init; } = TimeSpan.FromMilliseconds(50);
    public TimeSpan KagamiOperation    { get; init; } = TimeSpan.FromMilliseconds(200);
    public TimeSpan EmailDispatch      { get; init; } = TimeSpan.FromSeconds(2);
    public TimeSpan RiskEvaluation     { get; init; } = TimeSpan.FromMilliseconds(100);
    public TimeSpan ChallengeTtl       { get; init; } = TimeSpan.FromMinutes(5);
    public TimeSpan MagicLinkTtl       { get; init; } = TimeSpan.FromMinutes(10);  // tope INV-106
    public TimeSpan OtpTtl             { get; init; } = TimeSpan.FromMinutes(5);   // tope RN-38
    public TimeSpan SessionCodeTtl     { get; init; } = TimeSpan.FromSeconds(60);
    public int EmailDispatchRetries    { get; init; } = 1;   // único retry del UC: transporte no crítico
    public int Retries                 { get; init; } = 0;   // GETDEL/SUPERSEDE no son reintentables
}
```

### 10.2. Verificación de Orden en Tests de Integración

```csharp
[Test]
public async Task PasskeyVerify_ChecksUv_BeforeRegistryLookup()
{
    var order = new List<string>();
    CeremonyHooks.OnUvCheck(() => order.Add("uv"));
    CeremonyHooks.OnRegistryLookup(() => order.Add("registry"));

    await Ceremony.VerifyAsync(AssertionWithoutUv, Policy, CT);

    order.Should().StartWith("uv");
    order.Should().NotContain("registry");   // UV falla antes del lookup
}

[Test]
public async Task MagicLinkConfirm_ValidatesGate_BeforeConsumption()
{
    var order = new List<string>();
    Store.Setup(s => s.ConsumeAsync(It.IsAny<string>(), It.IsAny<CancellationToken>()))
         .Callback(() => order.Add("consume")).ReturnsAsync(Token);
    // Gate denegado por device mismatch:
    var outcome = await Flow.ConfirmAsync(CmdFromOtherDevice, handoff: null, CT);

    outcome.Stage.Should().Be(MagicLinkStage.DeviceMismatch);
    order.Should().BeEmpty();                // el consumo nunca ocurrió
}

[Test]
public async Task OtpVerify_WithoutIssuance_ExecutesDummyCompare()
{
    Store.Setup(s => s.GetActiveAsync(It.IsAny<string>(), It.IsAny<CancellationToken>()))
         .ReturnsAsync((EmailOtpIssuance?)null);

    await Flow.VerifyAsync(cmd, context, Policy, CT);

    Dummy.Verify(d => d.Compare(It.IsAny<OtpCode>()), Times.Once);
}

[Test]
public async Task ConcurrentMagicLinkConfirms_ExactlyOneIssues()
{
    var results = await Task.WhenAll(
        Flow.ConfirmAsync(cmd, deviceCtx, Policy, CT),
        Flow.ConfirmAsync(cmd, deviceCtx, Policy, CT));

    results.Count(r => r.IsSessionIssued || r.IsPausedForMfa).Should().Be(1);
    results.Count(r => r.Stage == MagicLinkStage.Consumed).Should().Be(1);
}
```

### 10.3. Convenciones de Numeración

- Cada diagrama tiene numeración independiente (1-n) con sufijos `a/b/c/d` para ramas mutuamente excluyentes dentro de un `alt`.
- Los pasos que corresponden a operaciones puras (validación GET, gate de binding) se anotan explícitamente como **sin mutación** para que la revisión de seguridad verifique el contrato de dos pasos directamente sobre el diagrama.
- Las flechas `->>` marcan despachos async fire-and-forget (email), diferenciándolos de los round-trips síncronos del verify.

---
*Fin del Documento 07_Sequence_Diagram para UC-16*