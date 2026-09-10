# UC-07: Invitación de Usuarios - Especificación de Requisitos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-07 - Invitación de Usuarios (Admin-Initiated Invitation) |
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
| **1.0.0** | 04/09/2026 | Gastón Ruiz | **Creación inicial.** Definición completa de la especificación analítica para la invitación de usuarios por administradores. Establece el flujo de invitación con tokens seguros, expiración configurable, reenvío de invitaciones, aceptación/rechazo por parte del usuario invitado, y la integración con el flujo de registro (UC-06) para completar el onboarding. | Documento base para el paquete de diseño Tier 1 del UC-07. Es el caso de uso complementario al UC-06 (Registro Self-Service), permitiendo que administradores inviten usuarios específicos sin requerir que el usuario inicie el registro por sí mismo. |

---

## 1. Introducción y Alcance

Este documento detalla la especificación de requisitos para el Caso de Uso **UC-07**, que representa la operación de **invitación de usuarios por administradores** en el motor **KAGAMI** de SHUGO.

A diferencia del UC-06 (Registro Self-Service) donde el usuario inicia el registro por sí mismo, y del UC-01 (Provisionar) donde el admin crea la identidad con datos completos, el UC-07 permite que **un administrador invite a usuarios específicos** mediante un flujo controlado:

1. **Invitación Dirigida:** El administrador especifica el email del usuario a invitar, sin necesidad de conocer todos sus datos personales.
2. **Token de Invitación Seguro:** Se genera un token criptográfico con TTL configurable (ej. 7 días) para aceptar la invitación.
3. **Control de Expiración:** Las invitaciones no aceptadas expiran automáticamente y pueden ser reenviadas o canceladas.
4. **Flujo de Aceptación:** El usuario invitado recibe un email con un link seguro para completar su registro (integración con UC-06).
5. **Asignación de Roles Iniciales:** El administrador puede asignar roles básicos al usuario invitado desde el momento de la invitación.

### 1.1. Importancia Estratégica

El UC-07 es **crítico para el onboarding corporativo** porque:

- **Control Administrativo:** Permite a las organizaciones controlar quién se une a su tenant, a diferencia del registro público (UC-06).
- **Experiencia de Usuario Simplificada:** El usuario invitado solo necesita crear una contraseña y aceptar términos, sin completar formularios extensos.
- **Seguridad Mejorada:** Reduce el riesgo de cuentas falsas o no autorizadas, ya que cada invitación es trazable a un administrador específico.
- **Integración con HRIS:** Puede integrarse con sistemas de Recursos Humanos para automatizar invitaciones basadas en eventos de contratación.

### 1.2. Relación con Otros Casos de Uso

| UC | Relación | Descripción |
| :--- | :--- | :--- |
| **UC-01 (Provisionar)** | Alternativa | UC-01 crea la identidad con datos completos; UC-07 invita al usuario a completar su propio registro. |
| **UC-06 (Registro Self-Service)** | Integración | El usuario invitado completa su registro vía UC-06 al aceptar la invitación. |
| **UC-09 (Verificación de Contactos)** | Consumidor | Tras aceptar la invitación, el email debe verificarse (UC-09). |
| **UC-19 (Definir Rol)** | Consumidor | El administrador puede asignar roles iniciales durante la invitación. |
| **UC-20 (Asignar Rol a Identidad)** | Consumidor | Los roles asignados en la invitación se aplican tras la aceptación. |

### 1.3. Objetivos de Rendimiento

| Métrica | Objetivo | Justificación |
| :--- | :--- | :--- |
| **Latencia p95 (crear invitación)** | < 200ms | La creación de invitación debe ser rápida (generación de token + envío de email asíncrono). |
| **Throughput** | 50 invitaciones/minuto | Suficiente para onboarding masivo de nuevos empleados. |
| **Disponibilidad** | 99.9% | Las invitaciones son críticas para el onboarding de usuarios. |
| **Tasa de Entrega de Email** | > 95% | Las invitaciones deben llegar a la bandeja de entrada (no spam). |
| **Tasa de Aceptación** | > 70% | Los usuarios invitados deben aceptar dentro del período de expiración. |

---

## 2. Descripción Analítica (ICONIX Clásico)

| Campo | Detalle |
| :--- | :--- |
| **ID Y NOMBRE:** | UC-07 - Invitación de Usuarios (Admin-Initiated Invitation) |
| **ESTADO:** | Preliminar |
| **DESCRIPCIÓN:** | Un administrador envía una invitación a un usuario especificando su email y opcionalmente roles iniciales. El sistema valida que el email no esté ya registrado, genera un token de invitación seguro con TTL configurable, crea un registro de invitación en estado PENDING, envía un email al usuario invitado con un link seguro, y retorna confirmación al administrador. El usuario invitado puede aceptar la invitación (completando su registro vía UC-06) o rechazarla. Las invitaciones no aceptadas expiran automáticamente tras el TTL configurado. |
| **ACTOR PRINCIPAL:** | Administrador de Tenant (Humano), Sistema de RRHH (Automatizado, vía integración). |
| **ACTOR SECUNDARIO:** | Usuario Invitado (Humano), Sistema de Email (Envío de invitación), UC-06 (Registro Self-Service para completar onboarding). |
| **PRECONDICIONES:** | 1. El administrador debe tener el permiso `user:invite`.<br>2. El email del usuario invitado no debe estar ya registrado en el tenant.<br>3. El tenant debe tener habilitada la funcionalidad de invitaciones.<br>4. El administrador no debe exceder el límite de invitaciones pendientes (ej. 100 por admin). |
| **PUNTOS DE EXTENSIÓN:** | Si el email ya está registrado, se extiende a UC-20 (Asignar Rol) para agregar roles al usuario existente. Si la invitación expira, el administrador puede reenviarla o cancelarla. |
| **CONDICIÓN:** | La invitación se crea en estado PENDING con un token seguro. El usuario invitado tiene un tiempo limitado (TTL) para aceptarla. Tras la aceptación, se inicia el flujo de registro (UC-06) para completar el onboarding. |

### 2.1. Escenario Principal (Happy Path) - Invitación Exitosa con Aceptación

1. El administrador envía una solicitud POST a `/invitations` con: `email`, `roles[]` (opcional), `message` (opcional).
2. El API Gateway valida el JWT del administrador y el permiso `user:invite`.
3. El Invitation Handler recibe la solicitud y valida que el tenant permita invitaciones.
4. El Handler consulta la base de datos para verificar que el email no esté ya registrado en el tenant.
5. El Handler valida que el administrador no haya excedido el límite de invitaciones pendientes.
6. El Handler genera un token de invitación seguro (UUID v4 criptográfico) con TTL de 7 días.
7. El Handler crea un registro de invitación en estado `PENDING` con: `token`, `email`, `invited_by`, `roles`, `expires_at`.
8. El Handler envía un email al usuario invitado con el link de aceptación (incluye el token).
9. El Handler retorna HTTP 201 Created al administrador con el ID de la invitación.
10. El usuario invitado recibe el email y hace clic en el link de aceptación.
11. El sistema valida el token de invitación (no expirado, no usado).
12. El sistema redirige al usuario al flujo de registro (UC-06) pre-poblando el email.
13. El usuario completa su registro (crea contraseña, acepta términos).
14. El sistema transiciona la invitación a estado `ACCEPTED` y crea la identidad en estado `ACTIVE`.
15. El sistema asigna los roles especificados en la invitación al nuevo usuario.
16. El sistema envía un email de confirmación al administrador indicando que la invitación fue aceptada.

### 2.2. Escenario Alternativo - Invitación Rechazada por el Usuario

1-9. (Idénticos al escenario principal)
10. El usuario invitado recibe el email y hace clic en el link de rechazo (opcional).
11. El sistema transiciona la invitación a estado `DECLINED`.
12. El sistema envía un email al administrador notificando el rechazo.
13. Fin del Caso de Uso.

### 2.3. Escenario Alternativo - Invitación Expirada

1-9. (Idénticos al escenario principal)
10. El usuario invitado intenta aceptar la invitación después de 7 días.
11. El sistema detecta que el token ha expirado.
12. El sistema transiciona la invitación a estado `EXPIRED`.
13. El sistema retorna un mensaje al usuario: "Esta invitación ha expirado. Por favor contacta al administrador."
14. El administrador puede reenviar la invitación (generando un nuevo token) o cancelarla.
15. Fin del Caso de Uso.

### 2.4. Escenario Alternativo - Email Ya Registrado

1-3. (Idénticos al escenario principal)
4. El Handler detecta que el email ya está registrado en el tenant.
5. El Handler retorna HTTP 409 Conflict con mensaje: "Este email ya está registrado en el tenant."
6. El Handler sugiere al administrador usar UC-20 (Asignar Rol) para agregar roles al usuario existente.
7. Fin del Caso de Uso.

### 2.5. Flujos de Excepción

**[2.1] Administrador sin Permisos**
- 2.1.1 El API Gateway detecta que el administrador no tiene el permiso `user:invite`.
- 2.1.2 Retorna HTTP 403 Forbidden con mensaje "Insufficient permissions".
- 2.1.3 Fin del Caso de Uso.

**[3.1] Tenant No Permite Invitaciones**
- 3.1.1 El Handler detecta que `tenant_config.allow_invitations = false`.
- 3.1.2 Retorna HTTP 403 Forbidden con mensaje "Invitations are disabled for this tenant".
- 3.1.3 Fin del Caso de Uso.

**[5.1] Límite de Invitaciones Excedido**
- 5.1.1 El Handler detecta que el administrador tiene 100 invitaciones pendientes.
- 5.1.2 Retorna HTTP 429 Too Many Requests con mensaje "Invitation limit reached. Please wait for pending invitations to be accepted or cancel them".
- 5.1.3 Fin del Caso de Uso.

**[8.1] Error de Envío de Email**
- 8.1.1 El sistema de email falla al enviar la invitación.
- 8.1.2 El Handler hace rollback de la transacción (elimina la invitación creada).
- 8.1.3 Retorna HTTP 500 Internal Server Error con mensaje "Failed to send invitation email".
- 8.1.4 Fin del Caso de Uso.

**[11.1] Token de Invitación Inválido**
- 11.1.1 El usuario invitado intenta aceptar con un token inválido o malformado.
- 11.1.2 El sistema retorna HTTP 400 Bad Request con mensaje "Invalid invitation token".
- 11.1.3 Fin del Caso de Uso.

### 2.6. Postcondiciones

**De Éxito (Invitación Creada):**
1. Se ha creado un registro de invitación en estado `PENDING`.
2. Se ha generado un token seguro con TTL de 7 días.
3. Se ha enviado el email de invitación al usuario.
4. Se ha retornado HTTP 201 Created al administrador.

**De Éxito (Invitación Aceptada):**
1. La invitación ha transicionado a estado `ACCEPTED`.
2. Se ha creado la identidad del usuario en estado `ACTIVE`.
3. Se han asignado los roles especificados en la invitación.
4. Se ha enviado email de confirmación al administrador.

**De Falla:**
1. No se ha creado la invitación (rollback completo).
2. No se ha enviado ningún email.
3. Se ha retornado un código de error HTTP apropiado al administrador.

---

## 3. Reglas de Negocio e Invariantes de Dominio (DDD)

| ID | Invariante / Regla | Descripción Técnica |
| :--- | :--- | :--- |
| **INV-40** | Permiso de Invitación | Solo administradores con permiso `user:invite` pueden crear invitaciones. |
| **INV-41** | Unicidad de Email por Tenant | El email del usuario invitado no debe estar ya registrado en el tenant. |
| **INV-42** | Límite de Invitaciones Pendientes | Cada administrador puede tener máximo 100 invitaciones pendientes simultáneamente. |
| **INV-43** | Token de Invitación Seguro | El token debe ser UUID v4 criptográfico con TTL configurable (default 7 días). |
| **INV-44** | Estado Inicial PENDING | Las invitaciones se crean en estado `PENDING` y solo pueden transicionar a `ACCEPTED`, `DECLINED`, o `EXPIRED`. |
| **INV-45** | Expiración Automática | Las invitaciones no aceptadas expiran automáticamente tras el TTL. Un job en background actualiza el estado a `EXPIRED`. |
| **INV-46** | Roles Iniciales Opcionales | El administrador puede asignar roles iniciales durante la invitación. Estos roles se aplican tras la aceptación. |
| **INV-47** | Trazabilidad de Invitación | Cada invitación registra quién la creó (`invited_by`), cuándo (`created_at`), y cuándo fue aceptada/rechazada/expirada. |
| **RN-01** | Reenvío de Invitación | El administrador puede reenviar una invitación expirada o pendiente, generando un nuevo token y reiniciando el TTL. |
| **RN-02** | Cancelación de Invitación | El administrador puede cancelar una invitación pendiente, transicionándola a estado `CANCELLED`. |
| **RN-03** | Notificación de Aceptación | Tras la aceptación, se envía un email al administrador notificando que el usuario aceptó la invitación. |
| **RN-04** | Prevención de Enumeración | Si el email ya está registrado, no se revela si existe o no (mensaje genérico). |
| **RN-05** | Integración con UC-06 | Al aceptar la invitación, el usuario es redirigido al flujo de registro (UC-06) con el email pre-poblado. |
| **RN-06** | Auditoría de Invitaciones | Cada acción sobre una invitación (crear, reenviar, cancelar, aceptar, rechazar) genera un log de auditoría. |

---

## 4. Criterios de Aceptación (BDD / Gherkin)

### Escenario 4.1: Invitación exitosa con aceptación del usuario
```gherkin
DADO que el administrador "admin-001" tiene el permiso "user:invite"
Y el email "nuevo.usuario@khcs.com" no está registrado en el tenant "khcs-001"
Y el administrador tiene menos de 100 invitaciones pendientes
CUANDO el administrador envía POST /invitations con:
  | Campo    | Valor                              |
  |----------|------------------------------------|
  | email    | "nuevo.usuario@khcs.com"           |
  | roles    | ["user", "developer"]              |
  | message  | "Bienvenido al equipo!"            |
ENTONCES el sistema retorna HTTP 201 Created
Y se crea una invitación en estado "PENDING"
Y se genera un token seguro con TTL de 7 días
Y se envía un email de invitación al usuario
Y el mensaje es "Invitation sent successfully"
```

### Escenario 4.2: Aceptación de invitación por el usuario
```gherkin
DADO que existe una invitación "inv-uuid-123" en estado "PENDING"
Y el token de invitación es válido y no ha expirado
CUANDO el usuario invitado hace clic en el link de aceptación
ENTONCES el sistema valida el token de invitación
Y redirige al usuario al flujo de registro (UC-06) con el email pre-poblado
Y el usuario completa su registro (crea contraseña, acepta términos)
Y la invitación transiciona a estado "ACCEPTED"
Y se crea la identidad del usuario en estado "ACTIVE"
Y se asignan los roles "user" y "developer" al nuevo usuario
Y se envía un email de confirmación al administrador
```

### Escenario 4.3: Invitación rechazada por el usuario
```gherkin
DADO que existe una invitación "inv-uuid-456" en estado "PENDING"
CUANDO el usuario invitado hace clic en el link de rechazo
ENTONCES la invitación transiciona a estado "DECLINED"
Y se envía un email al administrador notificando el rechazo
Y el usuario no puede completar el registro con esa invitación
```

### Escenario 4.4: Invitación expirada
```gherkin
DADO que existe una invitación "inv-uuid-789" en estado "PENDING"
Y el token de invitación expiró hace 1 día (TTL de 7 días superado)
CUANDO el usuario invitado intenta aceptar la invitación
ENTONCES el sistema detecta que el token ha expirado
Y la invitación transiciona a estado "EXPIRED"
Y el sistema retorna el mensaje "Esta invitación ha expirado. Por favor contacta al administrador."
Y el administrador puede reenviar la invitación (generando nuevo token)
```

### Escenario 4.5: Email ya registrado en el tenant
```gherkin
DADO que el email "existente@khcs.com" ya está registrado en el tenant "khcs-001"
Y el administrador tiene el permiso "user:invite"
CUANDO el administrador envía POST /invitations con ese email
ENTONCES el sistema retorna HTTP 409 Conflict
Y el mensaje es "Este email ya está registrado en el tenant."
Y se sugiere al administrador usar UC-20 (Asignar Rol) para agregar roles
```

### Escenario 4.6: Administrador sin permisos
```gherkin
DADO que el usuario "user-001" NO tiene el permiso "user:invite"
CUANDO el usuario intenta enviar POST /invitations
ENTONCES el sistema retorna HTTP 403 Forbidden
Y el mensaje es "Insufficient permissions"
```

### Escenario 4.7: Límite de invitaciones excedido
```gherkin
DADO que el administrador "admin-001" tiene 100 invitaciones pendientes
CUANDO el administrador intenta crear una nueva invitación
ENTONCES el sistema retorna HTTP 429 Too Many Requests
Y el mensaje es "Invitation limit reached. Please wait for pending invitations to be accepted or cancel them"
```

### Escenario 4.8: Reenvío de invitación expirada
```gherkin
DADO que existe una invitación "inv-uuid-012" en estado "EXPIRED"
Y el administrador "admin-001" tiene el permiso "user:invite"
CUANDO el administrador envía POST /invitations/inv-uuid-012/resend
ENTONCES el sistema genera un nuevo token seguro con TTL de 7 días
Y la invitación transiciona de "EXPIRED" a "PENDING"
Y se envía un nuevo email de invitación al usuario
Y el mensaje es "Invitation resent successfully"
```

### Escenario 4.9: Cancelación de invitación pendiente
```gherkin
DADO que existe una invitación "inv-uuid-345" en estado "PENDING"
Y el administrador "admin-001" es quien creó la invitación
CUANDO el administrador envía DELETE /invitations/inv-uuid-345
ENTONCES la invitación transiciona a estado "CANCELLED"
Y se envía un email al usuario invitado notificando la cancelación (opcional)
Y el usuario no puede aceptar la invitación cancelada
```

### Escenario 4.10: Error de envío de email
```gherkin
DADO que el sistema de email está caído
Y el administrador completa la invitación exitosamente
CUANDO el sistema intenta enviar el email de invitación
ENTONCES el sistema hace rollback de la transacción
Y la invitación no se crea
Y el sistema retorna HTTP 500 Internal Server Error
Y el mensaje es "Failed to send invitation email. Please try again."
```

---

## 5. Eventos de Dominio Generados

### Evento 1: `Invitation.Created`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Invitation.Created",
  "timestamp": "2024-01-30T10:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "invitationId": "inv-uuid-123",
    "email": "nuevo.usuario@khcs.com",
    "invitedBy": "admin-001",
    "roles": ["user", "developer"],
    "expiresAt": "2024-02-06T10:00:00Z",
    "status": "PENDING"
  },
  "metadata": {
    "traceId": "opentelemetry-trace-id",
    "priority": "NORMAL"
  }
}
```

**Consumidores:**
1. **KAGE (Audit):** Registra el evento en logs de auditoría.
2. **TORII (Webhooks):** Notifica a sistemas externos (ej. HRIS) sobre la nueva invitación.
3. **Sistema de Email:** Envía el email de invitación al usuario.

### Evento 2: `Invitation.Accepted`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Invitation.Accepted",
  "timestamp": "2024-01-31T10:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "invitationId": "inv-uuid-123",
    "email": "nuevo.usuario@khcs.com",
    "identityId": "user-uuid-456",
    "acceptedAt": "2024-01-31T10:00:00Z",
    "roles": ["user", "developer"]
  },
  "metadata": {
    "traceId": "opentelemetry-trace-id",
    "priority": "NORMAL"
  }
}
```

**Consumidores:**
1. **KAGE (Audit):** Registra la aceptación en logs de auditoría.
2. **MURA (RBAC):** Asigna los roles especificados al nuevo usuario.
3. **TORII (Webhooks):** Notifica a sistemas externos que el usuario aceptó la invitación.
4. **Sistema de Email:** Envía email de confirmación al administrador.

### Evento 3: `Invitation.Expired`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Invitation.Expired",
  "timestamp": "2024-02-06T10:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "invitationId": "inv-uuid-789",
    "email": "otro.usuario@khcs.com",
    "invitedBy": "admin-001",
    "expiredAt": "2024-02-06T10:00:00Z"
  },
  "metadata": {
    "traceId": "opentelemetry-trace-id",
    "priority": "LOW"
  }
}
```

**Consumidores:**
1. **KAGE (Audit):** Registra la expiración en logs de auditoría.
2. **Job de Limpieza:** Actualiza el estado de la invitación a `EXPIRED`.

---

## 6. Consideraciones de Arquitectura y Seguridad

### 6.1. Seguridad de Tokens de Invitación

Los tokens de invitación deben ser:
- **Criptográficamente Seguros:** UUID v4 generado con `RandomNumberGenerator`.
- **De Un Solo Uso:** Una vez aceptada la invitación, el token se invalida.
- **Con TTL Estricto:** Expiran automáticamente tras 7 días (configurable por tenant).
- **Almacenados Hasheados:** El token se almacena como SHA256 hash en la base de datos, nunca en texto plano.

### 6.2. Prevención de Enumeración de Emails

Para prevenir que atacantes descubran qué emails están registrados en el tenant:
- **Mensaje Genérico:** Si el email ya está registrado, retornamos "Si el email está registrado, el usuario recibirá una invitación" (no revelamos si existe o no).
- **Tiempo Constante:** El tiempo de respuesta es constante independientemente de si el email existe o no.

### 6.3. Integración con UC-06 (Registro Self-Service)

Cuando el usuario invitado acepta la invitación:
1. El sistema valida el token de invitación.
2. El sistema redirige al usuario al flujo de registro (UC-06) con el email pre-poblado.
3. El usuario completa su registro (crea contraseña, acepta términos).
4. El sistema transiciona la invitación a `ACCEPTED` y crea la identidad en estado `ACTIVE`.
5. El sistema asigna los roles especificados en la invitación.

### 6.4. Job de Limpieza de Invitaciones Expiradas

Un job en background (cada hora) actualiza el estado de las invitaciones expiradas:
```sql
UPDATE invitations
SET status = 'EXPIRED'
WHERE status = 'PENDING'
  AND expires_at < NOW();
```

### 6.5. Auditoría Completa de Invitaciones

Cada acción sobre una invitación genera un log de auditoría:
- **Creación:** Quién invitó, a quién, qué roles, cuándo.
- **Reenvío:** Quién reenvió, cuándo, nuevo token generado.
- **Cancelación:** Quién canceló, cuándo, razón (opcional).
- **Aceptación:** Cuándo aceptó el usuario, identidad creada, roles asignados.
- **Rechazo:** Cuándo rechazó el usuario.
- **Expiración:** Cuándo expiró automáticamente.

---

## 7. Contratos de API (OpenAPI 3.0 Snippets)

### 7.1. Endpoint: POST /invitations

**Request:**
```http
POST /v1/invitations HTTP/1.1
Host: api.shugo.io
Authorization: Bearer <JWT>
Content-Type: application/json

{
  "email": "nuevo.usuario@khcs.com",
  "roles": ["user", "developer"],
  "message": "Bienvenido al equipo!"
}
```

**Response (201 Created):**
```json
{
  "invitation_id": "inv-uuid-123",
  "email": "nuevo.usuario@khcs.com",
  "status": "PENDING",
  "expires_at": "2024-02-06T10:00:00Z",
  "message": "Invitation sent successfully"
}
```

**Response (409 Conflict - Email ya registrado):**
```json
{
  "error": {
    "code": "EMAIL_ALREADY_REGISTERED",
    "message": "Este email ya está registrado en el tenant.",
    "suggested_action": "POST /v1/identities/{id}/roles"
  }
}
```

**Response (429 Too Many Requests - Límite excedido):**
```json
{
  "error": {
    "code": "INVITATION_LIMIT_REACHED",
    "message": "Invitation limit reached. Please wait for pending invitations to be accepted or cancel them."
  }
}
```

### 7.2. Endpoint: POST /invitations/{id}/accept

**Request:**
```http
POST /v1/invitations/inv-uuid-123/accept HTTP/1.1
Host: api.shugo.io
Content-Type: application/json

{
  "token": "uuid-v4-token-from-email"
}
```

**Response (200 OK):**
```json
{
  "invitation_id": "inv-uuid-123",
  "status": "ACCEPTED",
  "identity_id": "user-uuid-456",
  "message": "Invitation accepted. Please complete your registration."
}
```

### 7.3. Endpoint: POST /invitations/{id}/resend

**Request:**
```http
POST /v1/invitations/inv-uuid-789/resend HTTP/1.1
Host: api.shugo.io
Authorization: Bearer <JWT>
```

**Response (200 OK):**
```json
{
  "invitation_id": "inv-uuid-789",
  "status": "PENDING",
  "new_expires_at": "2024-02-13T10:00:00Z",
  "message": "Invitation resent successfully"
}
```

### 7.4. Endpoint: DELETE /invitations/{id}

**Request:**
```http
DELETE /v1/invitations/inv-uuid-345 HTTP/1.1
Host: api.shugo.io
Authorization: Bearer <JWT>
```

**Response (200 OK):**
```json
{
  "invitation_id": "inv-uuid-345",
  "status": "CANCELLED",
  "message": "Invitation cancelled successfully"
}
```

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `02_Use_Case_Diagram.md` | Mostrará el límite del sistema, los actores (Administrador, Usuario Invitado), y las relaciones con UC-06 (Registro) y UC-20 (Asignar Rol). |
| `03_Activity_Diagram.md` | Visualizará el flujo de creación de invitación, envío de email, aceptación/rechazo, y expiración. |
| `04_Initial_Domain_Model.md` | Definirá el Aggregate Root `Invitation`, los Value Objects `InvitationToken`, `InvitationStatus`, y el Domain Service `InvitationHandler`. |
| `05_State_Machine_Diagram.md` | Mostrará la máquina de estados de la invitación: `PENDING` → `ACCEPTED`/`DECLINED`/`EXPIRED`/`CANCELLED`. |
| `09_Modern_Artifacts.md` | Definirá los contratos OpenAPI completos, métricas de OpenTelemetry, y patrones de resiliencia. |
| `UC-01/01_Requirements_Specification.md` | Documento hermano que define el provisioning administrado (alternativa a la invitación). |
| `UC-06/01_Requirements_Specification.md` | Documento que define el registro self-service (integrado con la aceptación de invitación). |
| `UC-19/01_Requirements_Specification.md` | Documento que define la asignación de roles (aplicada tras la aceptación de invitación). |

---

## 9. Glosario de Términos Específicos de UC-07

| Término | Definición |
| :--- | :--- |
| **Invitación (Invitation)** | Registro que representa una invitación enviada por un administrador a un usuario específico para unirse al tenant. |
| **Token de Invitación** | UUID v4 criptográfico seguro con TTL configurable, usado para validar la aceptación de la invitación. |
| **Estado PENDING** | Estado inicial de una invitación. El usuario invitado aún no ha aceptado, rechazado, ni ha expirado. |
| **Estado ACCEPTED** | Estado final de una invitación aceptada. Se ha creado la identidad del usuario y asignado los roles. |
| **Estado DECLINED** | Estado final de una invitación rechazada por el usuario invitado. |
| **Estado EXPIRED** | Estado final de una invitación no aceptada dentro del TTL. Actualizado automáticamente por job de limpieza. |
| **Estado CANCELLED** | Estado final de una invitación cancelada por el administrador. |
| **TTL (Time To Live)** | Tiempo de vida de una invitación antes de expirar automáticamente (default: 7 días). |
| **Reenvío de Invitación** | Acción de generar un nuevo token y reiniciar el TTL para una invitación expirada o pendiente. |
| **Onboarding** | Proceso de incorporación de nuevos usuarios al tenant, que puede iniciarse vía invitación (UC-07) o registro self-service (UC-06). |

---
*Fin del Documento 01_Requirements_Specification para UC-07*