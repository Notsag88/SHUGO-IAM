# UC-16: Passwordless Authentication - Diagrama de Actividades

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-16 - Passwordless Authentication (Passkeys + Magic Links + OTP Restringido) |
| **Documento:** | 03_Activity_Diagram.md |
| **Metodología:** | UML 2.5 + ICONIX + DDD + Security by Design (Two-Step Consumption & Silent Issuance) |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 09/11/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 09/11/2026 | Gastón Ruiz | **Creación inicial.** Diagramas de Actividades UML con carriles (swimlanes) para UC-16 en cuatro partes: (1) la ceremonia passkey con resolución de identidad por credentialId, UV obligatorio pre-criptografía, contención de clonación y emisión directa AAL2+; (2) el ciclo completo de magic link con emisión silenciosa, despacho async, validación GET sin consumo, confirmación POST con GETDEL post-verificación de binding/handoff y pausa AAL2+; (3) el ciclo OTP con supersede de emisión, dummy hash anti-timing y agotamiento de intentos; y (4) el enrollment de passkey desde sesión autenticada con attestation por política. | Documento base del paquete de diseño Tier 1 del UC-16. Sirve como el "guion de ejecución" del Passwordless Core, fijando el orden consumo-solo-tras-validación, la inmunidad a prefetch por separación GET/POST, la emisión silenciosa idéntica exista o no la identidad, y el mapa fail-secure por dependencia — de modo que ningún canal emita assurance que no pueda demostrar. |

---

## 1. Introducción y Propósito

El **Diagrama de Actividades** para el UC-16 materializa la lógica de negocio y las reglas de seguridad definidas en la Especificación de Requisitos (01) y el Diagrama de Casos de Uso (02). Su propósito es:

1. **Separar los tres canales en flujos explícitos:** Passkey, magic link y OTP tienen naturalezas distintas (ceremonia criptográfica síncrona, enlace mediado por email con dos pasos, código con intentos acotados) y cada una exige su propio diagrama para que el orden de los controles sea auditable.
2. **Fijar el contrato de dos pasos del magic link:** El GET valida sin consumir y el POST consume solo tras verificar binding/handoff — el orden exacto que hace inofensivo el prefetch y única la confirmación.
3. **Modelar la emisión silenciosa:** Los canales OOB responden 202 genérico **antes** de decidir internamente si despachan; el despacho es async y su fallo es métrica, nunca respuesta.
4. **Documentar la pausa por assurance:** Los canales AAL1 divergen hacia UC-15 cuando la política exige más; los passkeys nunca pausan por assurance pero sí pueden ser contenidos por riesgo.
5. **Dejar explícito el mapa fail-secure:** Qué ocurre cuando Redis, KAGAMI o el proveedor de email fallan en cada punto de cada canal.

---

## 2. Diagrama de Actividades #1: Ceremonia Passkey (Primer Factor WebAuthn)

```plantuml
@startuml UC-16_Actividades_Passkey
<style>
diagram {
  BackgroundColor #FEFEFE
}
activity {
  BackgroundColor #E3F2FD
  BorderColor #1565C0
  ArrowColor #2C3E50
}
partition {
  BackgroundColor #F5F5F5
  BorderColor #9E9E9E
}
</style>

title UC-16: Autenticación con Passkey (WebAuthn Primer Factor, UV Obligatorio)

|Cliente (SPA/Mobile)|
start
:Solicitar login passwordless;
:POST /auth/passwordless/webauthn/options\n{ username_hint? };

|API Gateway|
:WAF + Rate Limit;
:Forward;

|Motor KAGI (Passwordless Core)|
:Generar challenge aleatorio ≥16 bytes;
:Construir PublicKeyRequestOptions\n(rpId del tenant,\nuserVerification = "required",\nallowCredentials según hint);
note right
  **Sin fuga de existencia:**
  La respuesta es idéntica con o sin
  credenciales sugeridas para el hint.
end note

|Redis (Tokens/Challenges)|
:SET challenge single-use\nTTL 5 min;

|Motor KAGI|
:Retornar opciones al cliente;

|Cliente (SPA/Mobile)|
:Iniciar ceremonia (conditional UI /\nautofill de passkeys);

|Authenticator de Plataforma|
:Usuario verifica localmente\n(biometría / PIN);
:Firmar assertion\n(UV=1, counter incremental);

|Cliente (SPA/Mobile)|
:POST /auth/passwordless/webauthn/verify\n{ assertion };

|API Gateway|
:Forward;

|Motor KAGI|
:GETDEL challenge (single-use);
if (¿Challenge ausente o expirado?) then (Sí)
  :passwordless_failed genérico;
  stop
endif

:Validar origin / rpId exactos;
if (¿Flag UV presente?) then (No)
  :Rechazo stage=uv_missing\n(INV-105: nunca sesión AAL2+ sin UV);
  :Auditar Passwordless.Failed;
  stop
endif

|KAGAMI (Identidades/Credenciales)|
:Resolver credencial por\n(tenant, credentialId)\n→ identidad + clave pública + counter;

|Motor KAGI|
if (¿Credencial existe y activa?) then (No)
  :Rechazo genérico\n(sin revelar si el credentialId es desconocido);
  stop
endif

:Verificar firma de la assertion;
if (¿Firma válida?) then (No)
  :Rechazo stage=signature_invalid;
  stop
endif

if (¿Counter no-cero Y\npresentado ≤ almacenado?) then (Sí: clonación)
  |KAGAMI|
  :Revocar credencial\n(motivo CLONE_DETECTED);
  |KAGE (Outbox)|
  :PasskeyCloneDetected\n(alerta SIEM crítica);
  |Motor KAGI|
  :Rechazo genérico;
  stop
else (No)
  |KAGAMI|
  :Actualizar counter;
endif

|UC-17 Risk|
:Evaluar delta de contexto\n(device fingerprint, IP, geo);

|Motor KAGI|
if (¿Score alto según política?) then (Sí)
  if (¿Política = denegar?) then (Sí)
    :Rechazo stage=risk_denied;
    stop
  else (No: step-up)
    :Pausar hacia UC-15\n(mfa_token, sin familia aún);
    stop
  endif
endif

:UC-V4: Assurance del passkey\n(AAL2; AAL3 si hardware verificado\n+ attestation directa por política);
note right
  **Sin pausa por assurance:**
  La ceremonia UV ES la evidencia.
  Solo el riesgo puede pausar/denegar.
end note

|KAGAMI|
:CreateRoot familia:\nauth_method = passwordless_webauthn\namr = ["hwk","user"]\nauth_level = AAL2/3\nauth_time = now;

|KAGE (Outbox)|
:Passwordless.WebAuthnAuthenticated;

|Motor KAGI|
:Emitir session_code\n(single-use, TTL 60s);

|Redis|
:SET passwordless:code:{value}\nTTL 60s;

|Motor KAGI|
:Retornar session_code;

|Cliente (SPA/Mobile)|
:POST /auth/passwordless/token\n{ session_code };

|Motor KAGI|
:GETDEL session_code\n→ emitir Access + Refresh tokens;

|Cliente (SPA/Mobile)|
:Persistir tokens;\nsesión passwordless establecida;
stop

@enduml
```

---

## 3. Diagrama de Actividades #2: Ciclo Completo de Magic Link

```plantuml
@startuml UC-16_Actividades_MagicLink
<style>
diagram {
  BackgroundColor #FEFEFE
}
activity {
  BackgroundColor #FFF3E0
  BorderColor #E65100
  ArrowColor #2C3E50
}
partition {
  BackgroundColor #F5F5F5
  BorderColor #9E9E9E
}
</style>

title UC-16: Magic Link - Emisión Silenciosa, Dos Pasos, Handoff Cross-Device

|Cliente (SPA/Mobile)|
start
:POST /auth/passwordless/magic-link\n{ email, redirect_uri };

|API Gateway|
:WAF + Rate Limit grueso (anti-DDoS);
:Forward;

|Motor KAGI (Passwordless Core)|
:UC-V1: Control de abuso\n(solicitudes por email/IP; suspensión\nsilenciosa al superar umbral);
:Normalizar email y calcular email_hash;

|KAGAMI (Identidades)|
:Lookup identidad por email\n→ activa? email verificado?\ncanal magic_link habilitado?;

|Motor KAGI|
if (¿Identidad válida Y canal habilitado\nY dentro de límites?) then (Sí)
  :Generar token 256 bits CSPRNG\n+ device binding (fingerprint, IP-class);
  if (¿Política cross_device_handoff?) then (Sí)
    :Generar handoff code 6 dígitos\n(se muestra SOLO en dispositivo origen);
  endif
  |Redis (Tokens/Challenges)|
  :SET magiclink:{token_hash}\n{identity, binding, handoff_hash?,\nttl 10 min};
  |Motor KAGI|
  :Despacho ASYNC del email\n(fire-and-forget, 1 retry);
  note right
    Si el despacho falla:
    métrica dispatch_failed.
    El 202 ya fue retornado;
    el usuario re-solicita.
  end note
  |KAGE (Outbox)|
  :MagicLinkIssued;
else (No)
  :Sin despacho;\nmétrica interna unknown_email\n(respuesta idéntica);
endif

:Retornar 202 genérico\n(exista o no la identidad);

|Cliente (SPA/Mobile)|
:202: "Si la cuenta existe, se envió un enlace";

|Proveedor de Email|
:Entrega del email\n(fuera de nuestro SLO);

|Usuario|
:Clic en el enlace\n(desde el mismo u otro dispositivo);

|Cliente (SPA/Mobile)|
:GET /auth/passwordless/link?token=...;

|API Gateway|
:Forward;

|Motor KAGI|
:GET magiclink:{token_hash}\n(SIN consumir);
if (¿Token ausente o expirado?) then (Sí)
  :Página de enlace inválido/expirado\ncon opción de re-solicitud;
  stop
endif

:UC-V2: Verificar device binding;
if (¿Binding coincide con dispositivo actual?) then (Sí)
  :Página de confirmación\n(modo mismo dispositivo);
else (No)
  if (¿Política cross_device_handoff?) then (Sí)
    :Página de confirmación\ncon entrada de handoff code;
  else (No)
    :Página de device mismatch:\nreintentar desde dispositivo origen;
    note right
      El enlace NO se consume:
      el GET es inofensivo.
    end note
    stop
  endif
endif

|Usuario|
:Confirmar (botón) e ingresar\nhandoff code si aplica;

|Cliente (SPA/Mobile)|
:POST /auth/passwordless/link/confirm\n{ token, handoff_code? };

|Motor KAGI|
:Validar binding/handoff\n(comparación tiempo-constante;\nhandoff con intentos acotados);
if (¿Binding/handoff válido?) then (No)
  :MagicLinkFailed{stage=device_mismatch\n| handoff_invalid};
  :Enlace NO consumido\n(reintento posible);
  stop
endif

|Redis (Tokens/Challenges)|
:GETDEL magiclink:{token_hash}\n(consumo atómico);
note right
  **Carrera de confirmaciones:**
  Dos POST simultáneos → exactamente
  uno gana el GETDEL; el otro cae
  en "consumido" (EXT_REPLAY_SIGNAL).
end note

|Motor KAGI|
if (¿Token ya consumido?) then (Sí)
  :MagicLinkFailed{stage=consumed};
  if (¿Replay repetido?) then (Sí)
    :Alerta SIEM: posible intercepción;
  endif
  stop
endif

|UC-17 Risk|
:Evaluar riesgo\n(contexto confirm vs solicitud);

|Motor KAGI|
if (¿Score alto?) then (Sí)
  :MagicLinkFailed{stage=risk_denied}\no step-up según política;
  stop
endif

:UC-V4: Assurance AAL1 (canal OOB);
if (¿required_level ≥ AAL2?) then (Sí)
  |UC-15 MFA|
  :Pausa con mfa_token\n(completitud tras segundo factor);
  |Cliente (SPA/Mobile)|
  :200 { mfa_required, mfa_token };
  stop
endif

|KAGAMI|
:CreateRoot familia:\nauth_method = passwordless_magic_link\namr = ["otk"], auth_level = AAL1\ncontexto cross_device;

|KAGE (Outbox)|
:MagicLinkConsumed\n{crossDevice, ttlElapsed};

|Motor KAGI|
:Emitir session_code (60s);
:302 → redirect_uri?session_code=...;

|Cliente (SPA/Mobile)|
:POST /auth/passwordless/token\n{ session_code } → tokens;
:Sesión establecida (AAL1);
stop

@enduml
```

---

## 4. Diagrama de Actividades #3: Ciclo OTP de Email (Restringido)

```plantuml
@startuml UC-16_Actividades_OTP
<style>
diagram {
  BackgroundColor #FEFEFE
}
activity {
  BackgroundColor #E8F5E9
  BorderColor #2E7D32
  ArrowColor #2C3E50
}
</style>

title UC-16: OTP de Email - Emisión con Supersede, Verificación con Intentos Acotados

|Cliente (SPA/Mobile)|
start
:POST /auth/passwordless/otp/request\n{ email };

|Motor KAGI (Passwordless Core)|
:UC-V1: Control de abuso\n(202 genérico siempre;\nsuspensión silenciosa por umbral);
:email_hash + lookup KAGAMI\n(identidad activa, email verificado,\ncanal otp habilitado por tenant);

if (¿Válido Y ≤ 3 emisiones/hora?) then (Sí)
  :Generar código 8 dígitos CSPRNG;
  |Redis (Tokens/Challenges)|
  :Supersede: invalidar OTP previo activo;\nSET otp:{email_hash}\n{code_hash, intentos=0, TTL 5 min};
  |Motor KAGI|
  :Despacho ASYNC del email\n(código, sin enlace);
  |KAGE (Outbox)|
  :OtpIssued;
else (No)
  :Sin despacho;\nmétrica interna (respuesta idéntica);
endif

:Retornar 202 genérico;

|Cliente (SPA/Mobile)|
:POST /auth/passwordless/otp/verify\n{ email, code };

|Motor KAGI|
:Lookup emisión activa por email_hash;
if (¿Emisión ausente o expirada?) then (Sí)
  :Ejecutar hash compare DUMMY\n(igualar timing del camino real);
  :passwordless_failed genérico;
  stop
endif

if (¿intentos ≥ 5?) then (Sí)
  :Invalidar emisión;\nOtpFailed{attempts_exhausted};
  :Sujeto a límite de re-emisión (3/h);
  stop
endif

:Hash compare tiempo-constante;
if (¿Código correcto?) then (Sí)
  |Redis|
  :GETDEL emisión (single-use);
  |UC-17 Risk|
  :Evaluar riesgo del contexto;
  |Motor KAGI|
  if (¿required_level ≥ AAL2?) then (Sí)
    |UC-15 MFA|
    :Pausa con mfa_token;
    stop
  endif
  |KAGAMI|
  :CreateRoot familia:\nauth_method = passwordless_email_otp\namr = ["otp"], auth_level = AAL1;
  |KAGE (Outbox)|
  :OtpVerified;
  |Motor KAGI|
  :session_code → tokens;
  stop
else (No)
  |Redis|
  :INCR intentos;
  |Motor KAGI|
  :OtpFailed{invalid_code};
  :passwordless_failed genérico;
  stop
endif

@enduml
```

---

## 5. Diagrama de Actividades #4: Enrollment de Passkey (desde Sesión Autenticada)

```plantuml
@startuml UC-16_Actividades_Enroll
<style>
diagram {
  BackgroundColor #FEFEFE
}
activity {
  BackgroundColor #F3E5F5
  BorderColor #6A1B9A
  ArrowColor #2C3E50
}
</style>

title UC-16: Enrollment de Passkey (Expansión de Superficie Protegida)

|Cliente (SPA/Mobile)|
start
:Sesión autenticada activa;
:POST /auth/passwordless/webauthn/enroll\n(Authorization: Bearer);

|Motor KAGI (Passwordless Core)|
:Validar assurance de la sesión\n(step-up si política lo exige);
if (¿Assurance insuficiente?) then (Sí)
  :403 step_up_required;
  stop
endif
:Validar política de tenant\n(passkeys permitidos);

|Redis (Tokens/Challenges)|
:SET enrollment challenge\nsingle-use, TTL 5 min;

|Motor KAGI|
:PublicKeyCreationOptions:\nresidentKey = "required" (discoverable)\nuserVerification = "required"\nattestation por política (none/indirect);

|Cliente (SPA/Mobile)|
:Iniciar ceremonia de registro;

|Authenticator de Plataforma|
:Usuario verifica localmente;\nCrear credencial discoverable;

|Cliente (SPA/Mobile)|
:POST /auth/passwordless/webauthn/enroll/verify\n{ attestation };

|Motor KAGI|
:GETDEL challenge;
:Validar origin/rpId;
:Verificar attestation según política;
:Extraer clave pública + credentialId;

|KAGAMI (Identidades/Credenciales)|
if (¿credentialId ya registrado?) then (Sí)
  :Rechazo: credencial duplicada;
  stop
else (No)
  :INSERT credencial:\nidentity, usage = first_factor,\ncounter = 0, enrolled_at;
endif

|KAGE (Outbox)|
:Passkey.Enrolled;

|Motor KAGI|
:200 enrolled\n(visible/revocable en UC-22);

|Cliente (SPA/Mobile)|
:Confirmación al usuario;
stop

@enduml
```

---

## 6. Análisis de los Carriles (Swimlanes) y Responsabilidades

### 6.1. Diagrama #1 (Passkey)

1. **Cliente:** Orquesta opciones → ceremonia → verify → canje de session_code. Con conditional UI/autofill para credenciales discoverable.
2. **Authenticator de Plataforma:** Produce la evidencia fuera del perímetro: biometría local + firma con UV=1 y counter incremental.
3. **Motor KAGI:** Genera challenges, valida ceremonia en orden UV→credencial→firma→counter, evalúa riesgo, emite familia y session_code.
4. **KAGAMI:** Resolución identidad por `(tenant, credentialId)`, actualización de counter, y `CreateRoot` con proveniencia.
5. **UC-17 / KAGE:** Riesgo antes de emisión; auditoría de autenticación y contención.

### 6.2. Diagrama #2 (Magic Link)

1. **Motor KAGI:** Emisión silenciosa (la decisión interna de despachar nunca altera la respuesta), validación GET sin consumo, confirmación POST con GETDEL post-validación.
2. **Proveedor de Email:** Transporte async fuera de SLO; su fallo es métrica, no error.
3. **Redis:** Estado del token (binding, handoff hash, TTL) con consumo atómico.
4. **UC-15:** Receptor de la pausa cuando AAL1 no satisface la política.

### 6.3. Diagrama #3 (OTP)

Ciclo más corto con tres propiedades clave: **supersede** (una emisión activa por identidad), **dummy hash** en verificación sin emisión (timing igualado), y **agotamiento** (5 intentos invalidan la emisión; 3 emisiones/hora limitan el ciclo completo).

### 6.4. Diagrama #4 (Enrollment)

Flujo autenticado con step-up condicional: el enrollment es expansión de superficie de acceso y exige assurance suficiente antes de crear un nuevo camino de login.

---

## 7. Puntos Críticos de Seguridad en el Flujo

### 7.1. Consumo Solo Tras Validación Exitosa (Magic Link)

En el Diagrama #2, el `GETDEL` ocurre **después** de verificar binding/handoff, no antes:

*   **Razón:** Si el consumo precediera a la validación, un device mismatch o handoff incorrecto destruiría el enlace del usuario legítimo (DoS autoinfligido por error).
*   **Carrera controlada:** Dos POST de confirmación simultáneos validan ambos, pero exactamente uno gana el `GETDEL`; el perdedor cae en `stage=consumed` (EXT_REPLAY_SIGNAL). El orden "validar → consumir" preserva el enlace ante fallos de validación sin sacrificar el single-use.

### 7.2. Inmunidad a Prefetch por Separación GET/POST

El GET (Diagrama #2, segundo bloque) ejecuta **solo lecturas**: valida TTL, comprueba binding, y renderiza la confirmación. Scanners de seguridad corporativos, pre-render de Outlook/Gmail, y re-aperturas del usuario ejecutan GETs sin efecto. El único acto con consecuencias es el POST de confirmación.

### 7.3. UV Antes de Criptografía (Passkey)

En el Diagrama #1, el flag UV se comprueba **antes** de resolver la credencial y verificar la firma:

*   **Razón doble:** (a) una assertion sin UV es estructuralmente inaceptable (INV-105) — no merece gasto criptográfico; (b) el motivo interno `uv_missing` queda aislado como fallo de ceremonia, no de credencial, para auditoría.

### 7.4. Identidad desde el Registro, Nunca Desde la Assertion

La resolución de identidad usa `(tenant, credentialId)` contra KAGAMI; el `userHandle` de la assertion se verifica **contra** el registro, no se confía como fuente. Una assertion forjada con userHandle ajeno pero credentialId propio no produce sesión de otra identidad.

### 7.5. Emisión Silenciosa y Despacho Async (Canales OOB)

El 202 genérico se retorna **antes** de la decisión interna de despacho:

*   La respuesta no depende de la existencia de la identidad (anti-enumeración, INV-109).
*   El despacho es fire-and-forget con 1 retry: si el proveedor de email falla, el token existe en Redis pero el email no llega; el usuario re-solicita y la emisión anterior queda superseded. El fallo de despacho **nunca** convierte el 202 en error.

### 7.6. OTP: Supersede + Dummy Hash + Agotamiento

*   **Supersede:** Cada nueva emisión invalida la anterior — nunca hay dos códigos activos por identidad (un atacante no puede multiplicar superficies solicitando emisiones).
*   **Dummy hash:** La verificación contra email sin emisión ejecuta un hash compare fantasma, igualando el timing del camino real (anti-oráculo de existencia en el verify, no solo en el request).
*   **Agotamiento en dos niveles:** 5 intentos invalidan la emisión; 3 emisiones/hora limitan el ciclo completo. Ambos con eventos de rate para el SIEM.

### 7.7. Riesgo Antes de Emisión en Todos los Canales

UC-17 evalúa **después** de la verificación de la prueba y **antes** de la emisión de familia en los tres canales:

*   Passkey: riesgo alto → step-up o denegación (única causa de pausa, nunca el assurance).
*   Magic link: riesgo compara contexto de confirmación vs solicitud (device, geo, tiempo transcurrido).
*   OTP: riesgo sobre el contexto del verify.

### 7.8. Session Code: Tokens Fuera de URLs en Todos los Canales

Los tres caminos de emisión terminan en el mismo puente: `session_code` de 60s single-use canjeado por back-channel. En magic link, el redirect lleva solo el session_code — el token del enlace ya fue consumido y jamás aparece en la URL final ni en el historial del destino.

### 7.9. Mapa Fail-Secure por Dependencia

| Fallo | Canal afectado | Comportamiento | Justificación |
| :--- | :--- | :--- | :--- |
| **Redis caído** | Todos | Emisiones OOB → 503 (sin store no hay token); passkey options/verify → 503; confirm/verify en curso → rechazo sin efectos secundarios | Sin store single-use no hay garantía de consumo único |
| **KAGAMI caído** | Todos | 503 fail-secure; ninguna resolución de identidad ni emisión | Sin fuente de verdad no hay sesión |
| **Proveedor de email caído** | Magic link / OTP | Emisión almacenada en Redis; despacho fallido = métrica; usuario re-solicita (supersede) | El canal de entrega es conveniencia, no seguridad |
| **UC-17 caído** | Todos | Fail-mode por política de tenant (`open` continúa con warning / `stepup` degrada a MFA) | El riesgo aumenta la seguridad; su ausencia no bloquea la autenticación |
| **Notification/KAGE caídos** | Eventos | Completitud continúa; eventos encolados con retry vía Outbox | La evidencia ya está verificada |

---

## 8. Trazabilidad con Otros Documentos del Paquete

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Implementa visualmente los escenarios 2.1-2.2.4 y 4.1-4.10, las invariantes INV-105 a INV-111 y las reglas RN-36 a RN-39. |
| `02_Use_Case_Diagram.md` | Los cuatro diagramas materializan UC-16a-g, UC-V1 a UC-V4 y las extensiones EXT_HANDOFF/MFA_PAUSE/RISK_STEPUP/CLONE_CONTAIN/REPLAY_SIGNAL/OTP_EXHAUST. |
| `04_Initial_Domain_Model.md` | Las acciones en los carriles de KAGI se mapearán a `PasskeyCeremony`, `MagicLinkFlow`, `OtpFlow` y `PasskeyEnrollment`. |
| `05_State_Machine_Diagram.md` | Las decisiones "¿token consumido?", "¿UV presente?", "¿counter regression?" son guardas de las máquinas de `MagicLinkToken`, `EmailOtpIssuance` y `PasskeyCredential`. |
| `07_Sequence_Diagram.md` | Definirá el orden temporal exacto de los round-trips: despacho async de email, GET/POST del enlace, ceremonias WebAuthn y canje de session_code. |
| `10_UC_Specific_Data_Model.md` | Los keyspaces Redis (`passwordless:magiclink:*`, `passwordless:otp:*`, `passwordless:code:*`, rate limits) y la tabla de credenciales passkey soportan estos flujos. |
| `11_UC_Specific_Security_Threats.md` | El orden consumo-post-validación (§7.1), el dummy hash (§7.6) y la emisión silenciosa (§7.5) mitigan intercepción, enumeración y stuffing. |
| `UC-15/03_Activity_Diagram.md` | La pausa AAL1→UC-15 (rama `required_level ≥ AAL2`) entrega el control al flujo de challenge/verify de ese diagrama. |

---

## 9. Notas de Implementación

### 9.1. Sintaxis PlantUML Moderna
- Sintaxis **Activity Diagram Beta** con carriles (`|Lane|`), `if/then/else/endif` para guardas y `note right` para decisiones de seguridad.
- Cuatro diagramas separados porque cada canal tiene **actores mediadores distintos**: authenticator (passkey), email + browser (magic link), transcripción (OTP), y sesión autenticada (enrollment).
- Bloques `<style>` con colores semánticos por canal: azul (passkey), naranja (magic link), verde (OTP), púrpura (enrollment).

### 9.2. Esqueleto del Enrutamiento por Canal

```csharp
public class PasswordlessHandler : IPasswordlessHandler
{
    public async Task<PasswordlessOutcome> ConfirmMagicLinkAsync(MagicLinkConfirmCommand cmd, CancellationToken ct)
    {
        // 1. VALIDAR antes de consumir: binding y handoff no destruyen el enlace si fallan
        var stored = await _links.GetAsync(cmd.Token, ct);
        if (stored is null) return PasswordlessOutcome.LinkInvalid();

        var binding = await _bindingValidator.ValidateAsync(stored, cmd, ct);   // incluye handoff
        if (!binding.IsValid)
        {
            await _outbox.PublishAsync(MagicLinkFailed(binding.Stage), ct);
            return PasswordlessOutcome.LinkInvalid();   // enlace intacto
        }

        // 2. CONSUMIR atómico: exactamente un ganador en carreras de confirmación
        var consumed = await _links.ConsumeAsync(cmd.Token, ct);
        if (consumed is null)
        {
            await _signals.RecordReplayAsync(cmd.Token, ct);   // EXT_REPLAY_SIGNAL
            return PasswordlessOutcome.LinkInvalid();
        }

        // 3. RIESGO antes de emitir
        var risk = await _risk.EvaluateAsync(consumed.Context, cmd.Context, ct);
        if (risk.IsDenied) return PasswordlessOutcome.RiskDenied();
        if (risk.RequiresStepUp) return PasswordlessOutcome.StepUp(risk.MfaToken);

        // 4. ASSURANCE: AAL1 → pausa si la política exige más
        var policy = await _policyResolver.ResolveAsync(consumed.TenantId, ct);
        if (policy.RequiredLevel > AssuranceLevel.AAL1)
            return await _mfaPauser.PauseAsync(consumed, ct);

        // 5. EMISIÓN con proveniencia + session_code
        return await _issuer.IssuePasswordlessSessionAsync(consumed, PasswordlessChannel.MagicLink, ct);
    }
}
```

### 9.3. Convenciones de Nomenclatura
- **Etapas de fallo:** `MagicLinkStage.{Expired, Consumed, DeviceMismatch, HandoffInvalid, RiskDenied}` y `OtpReason.{InvalidCode, AttemptsExhausted, Expired, RateLimited}` alineadas con los payloads de eventos del doc 01 (§5).
- **Flags de contexto:** `cross_device`, `ttlElapsed` y `handoff_used` viajan en los eventos de consumo para forense y scoring de riesgo.
- **Notas de seguridad:** Texto en negrita (**SIN consumir**, **consumo atómico**, **UV=1**) para resaltar las decisiones de orden que no son evidentes en el flujo.

---
*Fin del Documento 03_Activity_Diagram para UC-16*