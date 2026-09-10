# UC-06: Registro Self-Service - Especificación de Requisitos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-06 - Registro Self-Service (Self-Service Sign-Up) |
| **Documento:** | 01_Requirements_Specification.md |
| **Metodología:** | ICONIX (Deep Dive) + DDD + BDD + Event-Driven Architecture |
| **Estándares:** | NIST SP 800-63A (Enrollment), OWASP ASVS V2 (Authentication), GDPR Art. 6 & 7 (Consent), SOC 2 CC6.1 |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 04/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 04/09/2026 | Gastón Ruiz | **Creación inicial.** Definición completa de la especificación analítica para el registro self-service de identidades. Establece el flujo de registro público con verificación de email, aplicación de políticas de password, consentimiento GDPR, prevención de account takeover, y generación de estado inicial con verificación pendiente. | Documento base para el paquete de diseño Tier 1 del UC-06. Es el caso de uso de entrada para usuarios no administrados, complementando el UC-01 (Provisionar por Admin) y estableciendo los fundamentos de seguridad para el onboarding masivo. |

---

## 1. Introducción y Alcance

Este documento detalla la especificación de requisitos para el Caso de Uso **UC-06**, que representa la operación de **registro self-service de identidades** en el motor **KAGAMI** de SHUGO.

A diferencia del UC-01 (Provisionar por Admin) donde un administrador crea la identidad con control total, el UC-06 permite que **usuarios externos se registren por sí mismos** mediante un flujo público, con las siguientes características críticas:

1. **Verificación de Email Obligatoria:** El usuario debe confirmar su email mediante un token OTP o Magic Link antes de que la identidad esté completamente activa.
2. **Aplicación de Políticas de Password:** El sistema valida la contraseña contra las políticas configuradas por el tenant (longitud mínima, complejidad, breach detection).
3. **Consentimiento GDPR:** El usuario debe aceptar explícitamente los términos de servicio y la política de privacidad.
4. **Prevención de Account Takeover:** Mecanismos anti-abuso como rate limiting, captcha, y detección de emails desechables.
5. **Estado Inicial Controlado:** La identidad se crea en estado `PROVISIONED` o `ACTIVE_PENDING_VERIFICATION`, no puede tener roles administrativos por defecto.

### 1.1. Importancia Estratégica

El UC-06 es **crítico para la escalabilidad y experiencia de usuario** porque:

- **Onboarding Masivo:** Permite que miles de usuarios se registren sin intervención administrativa, reduciendo costos operativos.
- **Primera Impresión de Seguridad:** El flujo de registro establece el tono de seguridad de la plataforma. Un registro débil compromete toda la seguridad posterior.
- **Compliance desde el Inicio:** El consentimiento GDPR y la verificación de email deben ocurrir desde el primer momento, no como parche posterior.
- **Prevención de Abuso:** Sin controles adecuados, el registro self-service puede ser explotado para crear cuentas falsas, spam, o ataques de enumeración.

### 1.2. Relación con Otros Casos de Uso

| UC | Relación | Descripción |
| :--- | :--- | :--- |
| **UC-01 (Provisionar)** | Alternativa | UC-01 es el registro administrado; UC-06 es el registro self-service. Ambos crean identidades pero con flujos y permisos distintos. |
| **UC-02 (Actualizar)** | Consumidor | Tras el registro, el usuario puede actualizar su perfil (UC-02) una vez verificado. |
| **UC-03 (Desactivar)** | Consumidor | Si el usuario viola términos, puede ser desactivado vía UC-03. |
| **UC-04 (Consultar)** | Consumidor | El usuario puede consultar su propio perfil (UC-04) tras el registro. |
| **UC-09 (Verificación de Contactos)** | Predecesor | UC-06 inicia el flujo de verificación de email que UC-09 completa. |
| **UC-11 (Autenticar Credenciales Locales)** | Sucesor | Tras verificar el email, el usuario puede autenticarse vía UC-11. |
| **UC-41 (Configurar Password Policies)** | Configuración | Las políticas de password aplicadas en UC-06 son configuradas vía UC-41. |

### 1.3. Objetivos de Rendimiento

| Métrica | Objetivo | Justificación |
| :--- | :--- | :--- |
| **Latencia p95** | < 300ms | El registro debe ser rápido pero incluye validaciones de seguridad (breach detection, email verification). |
| **Throughput** | 100 req/seg | Suficiente para picos de registro masivo (ej. lanzamiento de producto). |
| **Disponibilidad** | 99.99% | El registro es crítico para adquisición de usuarios. |
| **Tasa de Verificación de Email** | > 85% | Los usuarios deben completar la verificación en < 24 horas. |
| **Tasa de Abuso Detectado** | < 1% | Mecanismos anti-abuso deben detectar y bloquear registros maliciosos. |

---

## 2. Descripción Analítica (ICONIX Clásico)

| Campo | Detalle |
| :--- | :--- |
| **ID Y NOMBRE:** | UC-06 - Registro Self-Service (Self-Service Sign-Up) |
| **ESTADO:** | Preliminar |
| **DESCRIPCIÓN:** | Un usuario externo envía una solicitud de registro proporcionando email, password, display_name, y aceptando los términos de servicio. El sistema valida el formato de los datos, verifica que el email no esté registrado, aplica las políticas de password del tenant, detecta si el email está en listas de breach (Have I Been Pwned), genera un token de verificación de email, crea la identidad en estado `PROVISIONED`, envía el email de verificación, y retorna confirmación al usuario. El usuario debe verificar su email dentro de un período configurable (ej. 24 horas) para que la identidad transicione a `ACTIVE`. |
| **ACTOR PRINCIPAL:** | Usuario Externo (Self-Service Registrant). |
| **ACTOR SECUNDARIO:** | Sistema de Email (Envío de verificación), Have I Been Pwned API (Breach detection), Tenant Config (Políticas de password). |
| **PRECONDICIONES:** | 1. El tenant debe permitir registro self-service (configuración `allow_self_registration = true`).<br>2. El email no debe estar ya registrado en el tenant.<br>3. La contraseña debe cumplir las políticas del tenant.<br>4. El usuario debe aceptar los términos de servicio y política de privacidad. |
| **PUNTOS DE EXTENSIÓN:** | Si el email ya está registrado, se extiende a UC-18 (Recuperación de Credenciales) para guiar al usuario. Si el tenant no permite self-registration, se retorna error. |
| **CONDICIÓN:** | La identidad se crea en estado `PROVISIONED` con verificación de email pendiente. El usuario tiene un tiempo limitado para verificar su email antes de que la identidad sea eliminada automáticamente. |

### 2.1. Escenario Principal (Happy Path) - Registro Exitoso con Verificación Pendiente

1. El usuario externo envía una solicitud POST a `/auth/register` con: `email`, `password`, `display_name`, `accept_terms: true`.
2. El API Gateway valida el formato de la request y aplica rate limiting (máximo 5 registros por IP por hora).
3. El Register Handler recibe la solicitud y extrae el `tenant_id` del contexto (subdominio o header).
4. El Handler valida que el tenant permita self-registration (`tenant_config.allow_self_registration == true`).
5. El Handler valida el formato del email (RFC 5322) y verifica que no sea un email desechable (lista de dominios bloqueados).
6. El Handler consulta PostgreSQL para verificar que el email no esté ya registrado en el tenant.
7. El Handler valida la contraseña contra las políticas del tenant (longitud mínima, complejidad, no común).
8. El Handler consulta la API de Have I Been Pwned para verificar si el email ha sido comprometido en brechas conocidas.
9. El Handler genera un token de verificación de email (UUID v4 con TTL de 24 horas).
10. El Handler crea la identidad en estado `PROVISIONED` con `email_verified = false`.
11. El Handler envía el email de verificación al usuario con el token y el link de verificación.
12. El Handler retorna HTTP 201 Created con mensaje: "Registro exitoso. Por favor verifica tu email."
13. El usuario recibe el email y hace clic en el link de verificación.
14. El sistema valida el token y transiciona la identidad a `ACTIVE` con `email_verified = true`.
15. El usuario puede ahora autenticarse (UC-11) y acceder al sistema.

### 2.2. Escenario Alternativo - Email Ya Registrado

1-5. (Idénticos al escenario principal)
6. El Handler detecta que el email ya está registrado en el tenant.
7. Si la identidad existente está en estado `PROVISIONED` (no verificada), el sistema reenvía el email de verificación.
8. Si la identidad existente está en estado `ACTIVE` (ya verificada), el sistema retorna HTTP 409 Conflict con mensaje: "Este email ya está registrado. ¿Olvidaste tu contraseña?"
9. El usuario es guiado hacia UC-18 (Recuperación de Credenciales).
10. Fin del Caso de Uso.

### 2.3. Escenario Alternativo - Tenant No Permite Self-Registration

1-3. (Idénticos al escenario principal)
4. El Handler detecta que `tenant_config.allow_self_registration == false`.
5. El Handler retorna HTTP 403 Forbidden con mensaje: "El registro self-service no está habilitado para este tenant. Contacta al administrador."
6. Fin del Caso de Uso.

### 2.4. Escenario Alternativo - Contraseña en Lista de Breach

1-7. (Idénticos al escenario principal)
8. El Handler consulta Have I Been Pwned y detecta que el email ha sido comprometido en una brecha conocida.
9. El Handler retorna HTTP 400 Bad Request con mensaje: "Este email ha sido comprometido en una brecha de seguridad. Por seguridad, no podemos permitir el registro con este email."
10. Fin del Caso de Uso.

### 2.5. Flujos de Excepción

**[2.1] Rate Limiting Excedido**
- 2.1.1 El API Gateway detecta que la IP ha excedido el límite de 5 registros por hora.
- 2.1.2 El Gateway retorna HTTP 429 Too Many Requests con header `Retry-After: 3600`.
- 2.1.3 El usuario recibe el error y debe esperar 1 hora antes de reintentar.
- 2.1.4 Fin del Caso de Uso.

**[5.1] Email Desechable Detectado**
- 5.1.1 El Handler detecta que el dominio del email está en la lista de emails desechables (ej. mailinator.com, guerrillamail.com).
- 5.1.2 El Handler retorna HTTP 400 Bad Request con mensaje: "No se permiten emails desechables. Por favor usa un email permanente."
- 5.1.3 Fin del Caso de Uso.

**[7.1] Contraseña No Cumple Políticas**
- 7.1.1 El Handler detecta que la contraseña no cumple las políticas del tenant (ej. muy corta, sin números).
- 7.1.2 El Handler retorna HTTP 400 Bad Request con mensaje detallado: "La contraseña debe tener al menos 12 caracteres, incluir mayúsculas, minúsculas, números y símbolos."
- 7.1.3 Fin del Caso de Uso.

**[10.1] Error de Envío de Email**
- 10.1.1 El sistema de email falla al enviar el email de verificación.
- 10.1.2 El Handler hace rollback de la transacción (elimina la identidad creada).
- 10.1.3 El Handler retorna HTTP 500 Internal Server Error con mensaje: "Error al enviar el email de verificación. Por favor intenta nuevamente."
- 10.1.4 Fin del Caso de Uso.

**[14.1] Token de Verificación Expirado**
- 14.1.1 El usuario intenta verificar su email después de 24 horas.
- 14.1.2 El sistema detecta que el token ha expirado.
- 14.1.3 El sistema retorna HTTP 400 Bad Request con mensaje: "El token de verificación ha expirado. Por favor solicita un nuevo registro."
- 14.1.4 El sistema elimina la identidad en estado `PROVISIONED`.
- 14.1.5 Fin del Caso de Uso.

### 2.6. Postcondiciones

**De Éxito (Registro Inicial):**
1. La identidad ha sido creada en estado `PROVISIONED` con `email_verified = false`.
2. Se ha generado un token de verificación con TTL de 24 horas.
3. Se ha enviado el email de verificación al usuario.
4. Se ha retornado HTTP 201 Created al usuario.

**De Éxito (Verificación de Email):**
1. La identidad ha transicionado de `PROVISIONED` a `ACTIVE`.
2. El campo `email_verified` ha sido actualizado a `true`.
3. El token de verificación ha sido invalidado.
4. El usuario puede ahora autenticarse (UC-11).

**De Falla:**
1. No se ha creado la identidad (rollback completo).
2. No se ha enviado ningún email.
3. Se ha retornado un código de error HTTP apropiado al usuario.

---

## 3. Reglas de Negocio e Invariantes de Dominio (DDD)

| ID | Invariante / Regla | Descripción Técnica |
| :--- | :--- | :--- |
| **INV-31** | Tenant Self-Registration Config | El tenant debe tener `allow_self_registration = true` para permitir registros self-service. Si es `false`, se retorna 403. |
| **INV-32** | Email Uniqueness por Tenant | El email debe ser único dentro del tenant. No se permiten duplicados. |
| **INV-33** | Email Format Validation | El email debe cumplir RFC 5322 y no ser un dominio desechable (lista configurable por tenant). |
| **INV-34** | Password Policy Enforcement | La contraseña debe cumplir las políticas del tenant (longitud, complejidad, no común). |
| **INV-35** | Breach Detection | El email no debe estar en listas de brechas conocidas (Have I Been Pwned API). |
| **INV-36** | Terms Acceptance Required | El usuario debe aceptar explícitamente los términos de servicio y política de privacidad (`accept_terms = true`). |
| **INV-37** | Initial State is PROVISIONED | La identidad se crea en estado `PROVISIONED` con `email_verified = false`. No puede tener roles admin. |
| **INV-38** | Email Verification TTL | El token de verificación tiene un TTL configurable (default 24 horas). Tras expirar, la identidad se elimina. |
| **INV-39** | No Admin Roles by Default | Las identidades creadas vía self-service no pueden tener roles administrativos por defecto. |
| **RN-01** | Rate Limiting | Máximo 5 registros por IP por hora para prevenir abuso. |
| **RN-02** | Email Re-send | Si el email ya está registrado en estado `PROVISIONED`, se reenvía el email de verificación. |
| **RN-03** | Automatic Cleanup | Las identidades en estado `PROVISIONED` con token expirado se eliminan automáticamente tras 24 horas. |
| **RN-04** | Audit Log Creation | Cada registro self-service genera un log de auditoría con IP, user agent, y timestamp. |
| **RN-05** | GDPR Consent Logging | El consentimiento de términos y política de privacidad se registra con timestamp y versión de los documentos. |

---

## 4. Criterios de Aceptación (BDD / Gherkin)

### Escenario 4.1: Registro exitoso con verificación pendiente
```gherkin
DADO que el tenant "khcs-001" tiene allow_self_registration = true
Y el email "nuevo.usuario@khcs.com" no está registrado en el tenant
Y la contraseña "SecureP@ssw0rd!" cumple las políticas del tenant
Y el email no está en listas de breach
Y el usuario acepta los términos de servicio
CUANDO el usuario envía POST /auth/register con:
  | Campo         | Valor                              |
  |---------------|------------------------------------|
  | email         | "nuevo.usuario@khcs.com"           |
  | password      | "SecureP@ssw0rd!"                  |
  | display_name  | "Nuevo Usuario"                    |
  | accept_terms  | true                               |
ENTONCES el sistema retorna HTTP 201 Created
Y la identidad se crea en estado "PROVISIONED" con email_verified = false
Y se genera un token de verificación con TTL de 24 horas
Y se envía un email de verificación al usuario
Y el mensaje es "Registro exitoso. Por favor verifica tu email."
```

### Escenario 4.2: Verificación de email exitosa
```gherkin
DADO que la identidad "user-uuid-123" está en estado "PROVISIONED" con email_verified = false
Y se generó un token de verificación válido con TTL de 24 horas
CUANDO el usuario hace clic en el link de verificación con el token válido
ENTONCES el sistema transiciona la identidad a estado "ACTIVE"
Y el campo email_verified se actualiza a true
Y el token de verificación se invalida
Y el usuario puede autenticarse (UC-11)
```

### Escenario 4.3: Email ya registrado (activo)
```gherkin
DADO que el email "existente@khcs.com" ya está registrado en el tenant "khcs-001"
Y la identidad está en estado "ACTIVE" con email_verified = true
CUANDO un usuario intenta registrarse con el mismo email
ENTONCES el sistema retorna HTTP 409 Conflict
Y el mensaje es "Este email ya está registrado. ¿Olvidaste tu contraseña?"
Y se sugiere al usuario ir a UC-18 (Recuperación de Credenciales)
```

### Escenario 4.4: Email ya registrado (no verificado)
```gherkin
DADO que el email "pendiente@khcs.com" está registrado en estado "PROVISIONED"
Y el token de verificación aún no ha expirado
CUANDO un usuario intenta registrarse con el mismo email
ENTONCES el sistema reenvía el email de verificación
Y retorna HTTP 200 OK
Y el mensaje es "Email de verificación reenviado. Por favor revisa tu bandeja de entrada."
```

### Escenario 4.5: Tenant no permite self-registration
```gherkin
DADO que el tenant "khcs-001" tiene allow_self_registration = false
CUANDO un usuario intenta registrarse vía POST /auth/register
ENTONCES el sistema retorna HTTP 403 Forbidden
Y el mensaje es "El registro self-service no está habilitado para este tenant. Contacta al administrador."
```

### Escenario 4.6: Email desechable detectado
```gherkin
DADO que el dominio "mailinator.com" está en la lista de emails desechables del tenant
CUANDO un usuario intenta registrarse con email "test@mailinator.com"
ENTONCES el sistema retorna HTTP 400 Bad Request
Y el mensaje es "No se permiten emails desechables. Por favor usa un email permanente."
```

### Escenario 4.7: Contraseña no cumple políticas
```gherkin
DADO que las políticas del tenant requieren: mínimo 12 caracteres, mayúsculas, minúsculas, números, símbolos
CUANDO un usuario intenta registrarse con password "123456"
ENTONCES el sistema retorna HTTP 400 Bad Request
Y el mensaje es "La contraseña debe tener al menos 12 caracteres, incluir mayúsculas, minúsculas, números y símbolos."
```

### Escenario 4.8: Email en lista de breach
```gherkin
DADO que el email "comprometido@khcs.com" está en la lista de Have I Been Pwned
CUANDO un usuario intenta registrarse con ese email
ENTONCES el sistema retorna HTTP 400 Bad Request
Y el mensaje es "Este email ha sido comprometido en una brecha de seguridad. Por seguridad, no podemos permitir el registro con este email."
```

### Escenario 4.9: Rate limiting excedido
```gherkin
DADO que la IP "192.168.1.100" ya ha realizado 5 registros en la última hora
CUANDO la misma IP intenta registrarse nuevamente
ENTONCES el sistema retorna HTTP 429 Too Many Requests
Y el header Retry-After es "3600"
Y el mensaje es "Demasiados intentos de registro. Por favor intenta nuevamente en 1 hora."
```

### Escenario 4.10: Token de verificación expirado
```gherkin
DADO que la identidad "user-uuid-456" está en estado "PROVISIONED"
Y el token de verificación expiró hace 1 hora (TTL de 24 horas superado)
CUANDO el usuario intenta verificar su email con el token expirado
ENTONCES el sistema retorna HTTP 400 Bad Request
Y el mensaje es "El token de verificación ha expirado. Por favor solicita un nuevo registro."
Y la identidad en estado "PROVISIONED" se elimina automáticamente
```

### Escenario 4.11: Términos no aceptados
```gherkin
DADO que el usuario no acepta los términos de servicio (accept_terms = false)
CUANDO el usuario intenta registrarse
ENTONCES el sistema retorna HTTP 400 Bad Request
Y el mensaje es "Debes aceptar los términos de servicio y política de privacidad para registrarte."
```

### Escenario 4.12: Error de envío de email
```gherkin
DADO que el sistema de email está caído
Y el usuario completa el registro exitosamente
CUANDO el sistema intenta enviar el email de verificación
ENTONCES el sistema hace rollback de la transacción
Y la identidad no se crea
Y el sistema retorna HTTP 500 Internal Server Error
Y el mensaje es "Error al enviar el email de verificación. Por favor intenta nuevamente."
```

---

## 5. Eventos de Dominio Generados

### Evento 1: `Identity.Registered`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Identity.Registered",
  "timestamp": "2024-01-30T10:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "email": "nuevo.usuario@khcs.com",
    "displayName": "Nuevo Usuario",
    "status": "PROVISIONED",
    "emailVerified": false,
    "registrationMethod": "SELF_SERVICE",
    "ipAddress": "192.168.1.100",
    "userAgent": "Mozilla/5.0..."
  },
  "metadata": {
    "traceId": "opentelemetry-trace-id",
    "priority": "NORMAL"
  }
}
```

**Consumidores:**
1. **KAGE (Audit):** Registra el evento en logs de auditoría con IP y user agent.
2. **TORII (Webhooks):** Notifica a sistemas externos sobre el nuevo registro (ej. CRM para onboarding).
3. **Sistema de Email:** Envía el email de verificación al usuario.

### Evento 2: `Identity.EmailVerified`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Identity.EmailVerified",
  "timestamp": "2024-01-30T11:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "email": "nuevo.usuario@khcs.com",
    "previousStatus": "PROVISIONED",
    "newStatus": "ACTIVE",
    "emailVerified": true,
    "verifiedAt": "2024-01-30T11:00:00Z"
  },
  "metadata": {
    "traceId": "opentelemetry-trace-id",
    "priority": "NORMAL"
  }
}
```

**Consumidores:**
1. **KAGI (Auth):** Permite que la identidad se autentique (UC-11).
2. **KIZUNA (AuthZ):** Asigna permisos básicos de usuario (no admin).
3. **TORII (Webhooks):** Notifica a sistemas externos que el usuario está activo.

### Evento 3: `Identity.RegistrationFailed`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Identity.RegistrationFailed",
  "timestamp": "2024-01-30T10:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "email": "test@mailinator.com",
    "reason": "DISPOSABLE_EMAIL_DETECTED",
    "ipAddress": "192.168.1.100"
  },
  "metadata": {
    "traceId": "opentelemetry-trace-id",
    "priority": "HIGH"
  }
}
```

**Consumidores:**
1. **KAGE (Audit):** Registra el intento fallido para análisis de seguridad.
2. **Sistema de Rate Limiting:** Incrementa contador de intentos fallidos por IP.

---

## 6. Consideraciones de Arquitectura y Seguridad

### 6.1. Prevención de Account Takeover durante Registro

El registro self-service es un vector de ataque común para account takeover. Implementamos múltiples capas de defensa:

1. **Email Verification Obligatoria:** Sin verificar el email, la identidad no puede autenticarse.
2. **Breach Detection:** Consultamos Have I Been Pwned para detectar emails comprometidos.
3. **Disposable Email Detection:** Bloqueamos dominios de emails desechables.
4. **Rate Limiting:** Limitamos registros por IP para prevenir abuso.
5. **Password Breach Detection:** Validamos que la contraseña no esté en listas de contraseñas comunes (ej. "123456", "password").

### 6.2. GDPR Compliance desde el Inicio

El registro self-service debe cumplir GDPR desde el primer momento:

1. **Consentimiento Explícito:** El usuario debe aceptar explícitamente los términos y política de privacidad.
2. **Registro de Consentimiento:** Guardamos timestamp, versión de documentos, e IP del consentimiento.
3. **Derecho al Olvido:** El usuario puede solicitar la eliminación de su cuenta en cualquier momento (UC-05).
4. **Minimización de Datos:** Solo recopilamos datos esenciales (email, password, display_name).

### 6.3. Estado Inicial y Transiciones

La identidad se crea en estado `PROVISIONED` con las siguientes restricciones:

- **No puede autenticarse** (UC-11 bloquea identidades no verificadas).
- **No tiene roles asignados** (ni siquiera roles básicos de usuario).
- **No puede ser actualizada** (UC-02 bloquea updates en estado PROVISIONED).
- **Solo puede ser verificada** (vía token de email) o **eliminada** (tras expirar TTL).

Tras la verificación de email, la identidad transiciona a `ACTIVE` y puede:
- Autenticarse (UC-11).
- Actualizar su perfil (UC-02).
- Consultar su perfil (UC-04).

### 6.4. Seguridad de Contraseñas

Implementamos las mejores prácticas de seguridad de contraseñas:

1. **Hashing con Argon2id:** Las contraseñas se hashean con Argon2id (resistente a GPU attacks).
2. **Salt Único:** Cada contraseña tiene un salt único generado criptográficamente.
3. **Políticas Configurables:** El tenant configura longitud mínima, complejidad, y blacklist de contraseñas comunes.
4. **Breach Detection:** Validamos contra listas de contraseñas filtradas (Have I Been Pwned Passwords API).

### 6.5. Prevención de Enumeración de Emails

Para prevenir que atacantes descubran qué emails están registrados:

- **Mensaje Genérico:** Si el email ya está registrado, retornamos "Si el email está registrado, recibirás un email de verificación" (no revelamos si existe o no).
- **Tiempo Constante:** El tiempo de respuesta es constante independientemente de si el email existe o no.
- **Rate Limiting:** Limitamos consultas por IP para prevenir brute force.

---

## 7. Contratos de API (OpenAPI 3.0 Snippets)

### 7.1. Endpoint: POST /auth/register

**Request:**
```http
POST /v1/auth/register HTTP/1.1
Host: khcs.shugo.io
Content-Type: application/json

{
  "email": "nuevo.usuario@khcs.com",
  "password": "SecureP@ssw0rd!",
  "display_name": "Nuevo Usuario",
  "accept_terms": true,
  "terms_version": "2.1",
  "privacy_policy_version": "1.3"
}
```

**Response (201 Created):**
```json
{
  "message": "Registro exitoso. Por favor verifica tu email.",
  "email": "nuevo.usuario@khcs.com",
  "verification_email_sent": true,
  "verification_ttl_hours": 24
}
```

**Response (400 Bad Request - Contraseña débil):**
```json
{
  "error": {
    "code": "WEAK_PASSWORD",
    "message": "La contraseña debe tener al menos 12 caracteres, incluir mayúsculas, minúsculas, números y símbolos."
  }
}
```

**Response (400 Bad Request - Email desechable):**
```json
{
  "error": {
    "code": "DISPOSABLE_EMAIL",
    "message": "No se permiten emails desechables. Por favor usa un email permanente."
  }
}
```

**Response (403 Forbidden - Tenant no permite self-registration):**
```json
{
  "error": {
    "code": "SELF_REGISTRATION_DISABLED",
    "message": "El registro self-service no está habilitado para este tenant. Contacta al administrador."
  }
}
```

**Response (409 Conflict - Email ya registrado):**
```json
{
  "error": {
    "code": "EMAIL_ALREADY_REGISTERED",
    "message": "Este email ya está registrado. ¿Olvidaste tu contraseña?",
    "suggested_action": "POST /v1/auth/forgot-password"
  }
}
```

**Response (429 Too Many Requests - Rate limit excedido):**
```http
HTTP/1.1 429 Too Many Requests
Retry-After: 3600

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Demasiados intentos de registro. Por favor intenta nuevamente en 1 hora."
  }
}
```

### 7.2. Endpoint: POST /auth/verify-email

**Request:**
```http
POST /v1/auth/verify-email HTTP/1.1
Host: khcs.shugo.io
Content-Type: application/json

{
  "token": "uuid-v4-verification-token"
}
```

**Response (200 OK):**
```json
{
  "message": "Email verificado exitosamente.",
  "identity_id": "user-uuid-123",
  "status": "ACTIVE",
  "email_verified": true
}
```

**Response (400 Bad Request - Token expirado):**
```json
{
  "error": {
    "code": "VERIFICATION_TOKEN_EXPIRED",
    "message": "El token de verificación ha expirado. Por favor solicita un nuevo registro."
  }
}
```

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `02_Use_Case_Diagram.md` | Mostrará el límite del sistema, el actor (Usuario Externo), y las relaciones con sistemas de email y breach detection. |
| `03_Activity_Diagram.md` | Visualizará el flujo de validación, creación de identidad, envío de email, y verificación. |
| `04_Initial_Domain_Model.md` | Definirá el Register Handler, el Email Verification Token, y las reglas de negocio. |
| `05_State_Machine_Diagram.md` | Mostrará la transición de `PROVISIONED` a `ACTIVE` tras la verificación de email. |
| `09_Modern_Artifacts.md` | Definirá los contratos OpenAPI completos, métricas de OpenTelemetry, y rate limiting. |
| `UC-01/01_Requirements_Specification.md` | Documento hermano que define el provisioning administrado (alternativa al self-service). |
| `UC-09/01_Requirements_Specification.md` | Documento que define la verificación de contactos (complementa el flujo de verificación de email). |
| `UC-11/01_Requirements_Specification.md` | Documento que define la autenticación (sucesor del registro self-service). |
| `UC-18/01_Requirements_Specification.md` | Documento que define la recuperación de credenciales (alternativa cuando el email ya está registrado). |

---

## 9. Glosario de Términos Específicos de UC-06

| Término | Definición |
| :--- | :--- |
| **Self-Service Registration** | Proceso mediante el cual un usuario externo se registra por sí mismo sin intervención administrativa. |
| **Email Verification** | Proceso de confirmación de que el usuario tiene acceso al email proporcionado, mediante un token OTP o Magic Link. |
| **PROVISIONED State** | Estado inicial de una identidad self-service antes de verificar el email. No permite autenticación ni updates. |
| **Disposable Email** | Email de servicios temporales (ej. mailinator.com) que no son permanentes. Bloqueados para prevenir abuso. |
| **Breach Detection** | Consulta a bases de datos de brechas conocidas (Have I Been Pwned) para detectar emails o contraseñas comprometidas. |
| **Rate Limiting** | Mecanismo para limitar el número de requests por IP en un período de tiempo, previniendo abuso. |
| **GDPR Consent** | Aceptación explícita del usuario a los términos de servicio y política de privacidad, registrada con timestamp. |
| **Argon2id** | Algoritmo de hashing de contraseñas resistente a ataques de GPU y side-channel. Recomendado por OWASP. |
| **Account Takeover (ATO)** | Ataque mediante el cual un atacante toma control de una cuenta legítima. Prevenido mediante verificación de email y breach detection. |
| **Magic Link** | Link de un solo uso enviado por email para verificar la identidad del usuario sin requerir contraseña. |

---
*Fin del Documento 01_Requirements_Specification para UC-06*