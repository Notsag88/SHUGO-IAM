# UC-08: Self-Service Profile - Especificación de Requisitos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-08 - Self-Service Profile (User-Initiated Profile Management) |
| **Documento:** | 01_Requirements_Specification.md |
| **Metodología:** | ICONIX (Deep Dive) + DDD + BDD + Event-Driven Architecture |
| **Estándares:** | NIST SP 800-63A (Enrollment), OWASP ASVS V2 (Authentication), GDPR Art. 16 (Right to Rectification), SOC 2 CC6.1 |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 05/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 05/09/2026 | Gastón Ruiz | **Creación inicial.** Definición completa de la especificación analítica para la gestión self-service del perfil de usuario. Establece el flujo donde usuarios autenticados pueden actualizar su propio perfil (display_name, avatar, preferencias, idioma, zona horaria), con validaciones de scope específicas, integración con UC-09 para cambios de email, y cumplimiento del derecho de rectificación GDPR Art. 16. | Documento base para el paquete de diseño Tier 1 del UC-08. Es el caso de uso complementario al UC-02 (Actualizar Identidad por Admin), permitiendo que los usuarios gestionen su propia información personal de forma segura y auditada. |

---

## 1. Introducción y Alcance

Este documento detalla la especificación de requisitos para el Caso de Uso **UC-08**, que representa la operación de **gestión self-service del perfil de usuario** en el motor **KAGAMI** de SHUGO.

A diferencia del UC-02 (Actualizar Identidad por Admin) donde un administrador modifica cualquier campo de la identidad, y del UC-01 (Provisionar) donde se crea la identidad completa, el UC-08 permite que **usuarios autenticados gestionen su propio perfil** con las siguientes características críticas:

1. **Autenticación Obligatoria:** Solo el propio usuario puede actualizar su perfil (validación de JWT con `sub` claim).
2. **Campos Limitados:** El usuario solo puede actualizar campos específicos: `display_name`, `avatar`, `preferences`, `language`, `timezone`. NO puede cambiar `email` directamente (requiere UC-09), ni `status`, ni `roles`.
3. **Validación de Avatar:** El avatar debe cumplir restricciones de tamaño (máx. 5MB), tipo (PNG, JPG, WebP), y dimensiones (máx. 1024x1024px).
4. **GDPR Art. 16 (Right to Rectification):** El usuario tiene derecho a rectificar sus datos personales inexactos.
5. **Auditoría Completa:** Cada actualización genera un log de auditoría con el campo anterior y nuevo valor.
6. **Prevención de Abuso:** Rate limiting específico para prevenir spam de actualizaciones.

### 1.1. Importancia Estratégica

El UC-08 es **crítico para la experiencia de usuario y cumplimiento GDPR** porque:

- **Autonomía del Usuario:** Permite a los usuarios controlar su propia información personal, mejorando la satisfacción y reduciendo la carga del soporte técnico.
- **GDPR Compliance:** Cumple con el Art. 16 (Derecho de Rectificación), permitiendo a los usuarios corregir datos inexactos.
- **Seguridad por Diseño:** Restringe qué campos puede modificar el usuario, previniendo escalación de privilegios (ej. asignarse roles admin).
- **Auditabilidad:** Cada cambio queda registrado para trazabilidad y forense.

### 1.2. Relación con Otros Casos de Uso

| UC | Relación | Descripción |
| :--- | :--- | :--- |
| **UC-02 (Actualizar)** | Alternativa | UC-02 es la actualización por admin (todos los campos); UC-08 es self-service (campos limitados). |
| **UC-04 (Consultar)** | Consumidor | El usuario consulta su perfil antes de actualizarlo. |
| **UC-09 (Verificación de Contactos)** | Integración | Si el usuario cambia su email, se dispara el flujo de verificación (UC-09). |
| **UC-11 (Autenticar)** | Predecesor | El usuario debe estar autenticado para gestionar su perfil. |
| **UC-46 (Consultar Logs de Auditoría)** | Consumidor | Los cambios de perfil se registran en logs de auditoría consultables. |

### 1.3. Objetivos de Rendimiento

| Métrica | Objetivo | Justificación |
| :--- | :--- | :--- |
| **Latencia p95** | < 200ms | La actualización de perfil debe ser rápida (sin integración con servicios externos lentos). |
| **Throughput** | 200 req/seg | Suficiente para picos de usuarios actualizando perfiles simultáneamente. |
| **Disponibilidad** | 99.99% | El perfil es crítico para la experiencia de usuario. |
| **Tasa de Error** | < 0.1% | Fallos en actualización de perfil son críticos (pérdida de datos del usuario). |

---

## 2. Descripción Analítica (ICONIX Clásico)

| Campo | Detalle |
| :--- | :--- |
| **ID Y NOMBRE:** | UC-08 - Self-Service Profile (User-Initiated Profile Management) |
| **ESTADO:** | Preliminar |
| **DESCRIPCIÓN:** | Un usuario autenticado envía una solicitud PATCH a `/profile` con los campos que desea actualizar (display_name, avatar, preferences, language, timezone). El sistema valida el JWT (el usuario solo puede actualizar su propio perfil), valida los campos permitidos, valida el avatar (tamaño, tipo, dimensiones), actualiza el perfil en la base de datos, genera un log de auditoría con los cambios, y retorna el perfil actualizado. Si el usuario intenta cambiar su email, se redirige al flujo de UC-09 (Verificación de Contactos). |
| **ACTOR PRINCIPAL:** | Usuario Autenticado (Self-Service User). |
| **ACTOR SECUNDARIO:** | Sistema de Almacenamiento de Avatares (S3/Azure Blob), UC-09 (Verificación de Email si cambia email). |
| **PRECONDICIONES:** | 1. El usuario debe estar autenticado (JWT válido).<br>2. El `sub` claim del JWT debe coincidir con el `identity_id` del perfil a actualizar.<br>3. Los campos a actualizar deben estar en la lista de campos permitidos para self-service.<br>4. El avatar (si se proporciona) debe cumplir las restricciones de tamaño y tipo. |
| **PUNTOS DE EXTENSIÓN:** | Si el usuario intenta cambiar su email, se extiende a UC-09 (Verificación de Contactos). Si el avatar excede el tamaño máximo, se retorna error. |
| **CONDICIÓN:** | El perfil se actualiza con los nuevos valores, se genera un log de auditoría, y se retorna el perfil actualizado al usuario. |

### 2.1. Escenario Principal (Happy Path) - Actualización Exitosa de Perfil

1. El usuario autenticado envía una solicitud PATCH a `/profile` con: `display_name`, `avatar` (opcional), `preferences`, `language`, `timezone`.
2. El API Gateway valida el JWT y extrae el `identity_id` del claim `sub`.
3. El Profile Handler recibe la solicitud y valida que el `identity_id` del JWT coincida con el perfil a actualizar.
4. El Handler valida que los campos a actualizar estén en la lista de campos permitidos para self-service.
5. Si se proporciona un avatar, el Handler valida: tamaño (<5MB), tipo (PNG/JPG/WebP), dimensiones (<1024x1024px).
6. El Handler consulta el perfil actual en la base de datos.
7. El Handler compara los valores actuales con los nuevos valores para detectar cambios.
8. El Handler actualiza el perfil en la base de datos con los nuevos valores.
9. El Handler genera un log de auditoría con: campo anterior, campo nuevo, timestamp, IP, user agent.
10. El Handler retorna HTTP 200 OK con el perfil actualizado.

### 2.2. Escenario Alternativo - Cambio de Email (Integración con UC-09)

1-4. (Idénticos al escenario principal)
5. El Handler detecta que el usuario intenta cambiar su email.
6. El Handler retorna HTTP 200 OK con mensaje: "Para cambiar tu email, por favor verifica el nuevo email."
7. El Handler dispara el flujo de UC-09 (Verificación de Contactos) para el nuevo email.
8. El email se actualiza solo tras la verificación exitosa.
9. Fin del Caso de Uso.

### 2.3. Escenario Alternativo - Avatar Inválido

1-4. (Idénticos al escenario principal)
5. El Handler detecta que el avatar excede el tamaño máximo (5MB).
6. El Handler retorna HTTP 400 Bad Request con mensaje: "Avatar size exceeds maximum allowed (5MB)."
7. Fin del Caso de Uso.

### 2.4. Flujos de Excepción

**[2.1] JWT Inválido o Expirado**
- 2.1.1 El API Gateway detecta que el JWT es inválido o expirado.
- 2.1.2 Retorna HTTP 401 Unauthorized con mensaje "Invalid or expired token."
- 2.1.3 Fin del Caso de Uso.

**[3.1] Mismatch de identity_id**
- 3.1.1 El `sub` claim del JWT no coincide con el `identity_id` del perfil.
- 3.1.2 Retorna HTTP 403 Forbidden con mensaje "You can only update your own profile."
- 3.1.3 Fin del Caso de Uso.

**[4.1] Campo No Permitido**
- 4.1.1 El usuario intenta actualizar un campo no permitido (ej. `status`, `roles`).
- 4.1.2 Retorna HTTP 400 Bad Request con mensaje "Field 'status' is not allowed for self-service updates."
- 4.1.3 Fin del Caso de Uso.

**[8.1] Error de Base de Datos**
- 8.1.1 La actualización del perfil falla (ej. deadlock, conexión caída).
- 8.1.2 Retorna HTTP 500 Internal Server Error con mensaje "Failed to update profile. Please try again."
- 8.1.3 Fin del Caso de Uso.

### 2.5. Postcondiciones

**De Éxito:**
1. El perfil ha sido actualizado con los nuevos valores.
2. Se ha generado un log de auditoría con los cambios.
3. Se ha retornado HTTP 200 OK con el perfil actualizado.

**De Falla:**
1. El perfil no ha sido modificado.
2. No se ha generado ningún log de auditoría.
3. Se ha retornado un código de error HTTP apropiado al usuario.

---

## 3. Reglas de Negocio e Invariantes de Dominio (DDD)

| ID | Invariante / Regla | Descripción Técnica |
| :--- | :--- | :--- |
| **INV-48** | Autenticación Obligatoria | Solo usuarios autenticados pueden actualizar su perfil. JWT válido requerido. |
| **INV-49** | Ownership Validation | El `sub` claim del JWT debe coincidir con el `identity_id` del perfil. No se puede actualizar el perfil de otro usuario. |
| **INV-50** | Campos Permitidos Limitados | Solo se pueden actualizar: `display_name`, `avatar`, `preferences`, `language`, `timezone`. NO: `email`, `status`, `roles`, `password`. |
| **INV-51** | Validación de Avatar | Avatar debe ser: tamaño <5MB, tipo PNG/JPG/WebP, dimensiones <1024x1024px. |
| **INV-52** | Email Requiere Verificación | Si el usuario cambia su email, se dispara UC-09. El email se actualiza solo tras verificación. |
| **INV-53** | Auditoría Obligatoria | Cada actualización genera un log de auditoría con: campo anterior, campo nuevo, timestamp, IP, user agent. |
| **INV-54** | Prevención de No-Op | Si no hay cambios (valores actuales == nuevos valores), no se actualiza ni se genera log. |
| **RN-01** | Rate Limiting | Máximo 10 actualizaciones de perfil por usuario por hora. |
| **RN-02** | Avatar Storage | Avatares se almacenan en S3/Azure Blob con URL pública. Se genera thumbnail (200x200px). |
| **RN-03** | GDPR Art. 16 | El usuario tiene derecho a rectificar sus datos personales. Este UC cumple con ese derecho. |
| **RN-04** | Preferencias por Defecto | Si el usuario no proporciona preferencias, se usan las del tenant o valores por defecto. |
| **RN-05** | Language/Timezone Validation | Language debe ser código ISO 639-1 (ej. "en", "es"). Timezone debe ser IANA (ej. "America/New_York"). |

---

## 4. Criterios de Aceptación (BDD / Gherkin)

### Escenario 4.1: Actualización exitosa de perfil
```gherkin
DADO que el usuario "user-uuid-123" está autenticado con JWT válido
Y el claim "sub" del JWT es "user-uuid-123"
CUANDO el usuario envía PATCH /profile con:
  | Campo         | Valor                              |
  |---------------|------------------------------------|
  | display_name  | "Nuevo Nombre"                     |
  | language      | "es"                               |
  | timezone      | "America/Argentina/Buenos_Aires"   |
ENTONCES el sistema retorna HTTP 200 OK
Y el perfil se actualiza con los nuevos valores
Y se genera un log de auditoría con los cambios
Y el mensaje es "Profile updated successfully"
```

### Escenario 4.2: Actualización con avatar válido
```gherkin
DADO que el usuario "user-uuid-123" está autenticado
Y el avatar es una imagen PNG de 2MB y 800x800px
CUANDO el usuario envía PATCH /profile con el avatar
ENTONCES el sistema retorna HTTP 200 OK
Y el avatar se sube a S3/Azure Blob
Y se genera un thumbnail de 200x200px
Y la URL del avatar se actualiza en el perfil
```

### Escenario 4.3: Intento de actualizar campo no permitido
```gherkin
DADO que el usuario "user-uuid-123" está autenticado
CUANDO el usuario envía PATCH /profile con:
  | Campo   | Valor      |
  |---------|------------|
  | status  | "ACTIVE"   |
ENTONCES el sistema retorna HTTP 400 Bad Request
Y el mensaje es "Field 'status' is not allowed for self-service updates"
Y el perfil no se modifica
```

### Escenario 4.4: Mismatch de identity_id
```gherkin
DADO que el usuario "user-uuid-123" está autenticado
Y el claim "sub" del JWT es "user-uuid-123"
CUANDO el usuario envía PATCH /profile/other-user-uuid con datos
ENTONCES el sistema retorna HTTP 403 Forbidden
Y el mensaje es "You can only update your own profile"
```

### Escenario 4.5: Avatar excede tamaño máximo
```gherkin
DADO que el usuario "user-uuid-123" está autenticado
Y el avatar es una imagen de 6MB
CUANDO el usuario envía PATCH /profile con el avatar
ENTONCES el sistema retorna HTTP 400 Bad Request
Y el mensaje es "Avatar size exceeds maximum allowed (5MB)"
```

### Escenario 4.6: Cambio de email (integración con UC-09)
```gherkin
DADO que el usuario "user-uuid-123" está autenticado
Y el email actual es "old@khcs.com"
CUANDO el usuario envía PATCH /profile con:
  | Campo  | Valor              |
  |--------|--------------------|
  | email  | "new@khcs.com"     |
ENTONCES el sistema retorna HTTP 200 OK
Y el mensaje es "To change your email, please verify the new email"
Y se dispara el flujo de UC-09 (Verificación de Contactos)
Y el email se actualiza solo tras verificación exitosa
```

### Escenario 4.7: Sin cambios (no-op)
```gherkin
DADO que el usuario "user-uuid-123" está autenticado
Y el display_name actual es "Nombre Actual"
CUANDO el usuario envía PATCH /profile con:
  | Campo         | Valor           |
  |---------------|-----------------|
  | display_name  | "Nombre Actual" |
ENTONCES el sistema retorna HTTP 200 OK
Y el mensaje es "No changes detected"
Y no se genera log de auditoría
```

### Escenario 4.8: Rate limiting excedido
```gherkin
DADO que el usuario "user-uuid-123" ha realizado 10 actualizaciones en la última hora
CUANDO el usuario intenta actualizar su perfil nuevamente
ENTONCES el sistema retorna HTTP 429 Too Many Requests
Y el mensaje es "Profile update limit reached. Please try again in 1 hour"
```

### Escenario 4.9: JWT inválido o expirado
```gherkin
DADO que el JWT del usuario ha expirado
CUANDO el usuario envía PATCH /profile con datos
ENTONCES el sistema retorna HTTP 401 Unauthorized
Y el mensaje es "Invalid or expired token"
```

### Escenario 4.10: Language/Timezone inválido
```gherkin
DADO que el usuario "user-uuid-123" está autenticado
CUANDO el usuario envía PATCH /profile con:
  | Campo    | Valor              |
  |----------|--------------------|
  | language | "invalid_code"     |
ENTONCES el sistema retorna HTTP 400 Bad Request
Y el mensaje es "Invalid language code. Must be ISO 639-1 (e.g., 'en', 'es')"
```

---

## 5. Eventos de Dominio Generados

### Evento 1: `Profile.Updated`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Profile.Updated",
  "timestamp": "2024-01-30T10:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "updatedFields": [
      {
        "field": "display_name",
        "oldValue": "Nombre Anterior",
        "newValue": "Nuevo Nombre"
      },
      {
        "field": "language",
        "oldValue": "en",
        "newValue": "es"
      }
    ],
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
1. **KAGE (Audit):** Registra el evento en logs de auditoría.
2. **TORII (Webhooks):** Notifica a sistemas externos sobre cambios de perfil (ej. CRM para actualizar datos de contacto).
3. **Sistema de Preferencias:** Actualiza las preferencias del usuario en caché.

### Evento 2: `Avatar.Uploaded`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Avatar.Uploaded",
  "timestamp": "2024-01-30T10:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "avatarUrl": "https://s3.amazonaws.com/shugo-avatars/user-uuid-123.png",
    "thumbnailUrl": "https://s3.amazonaws.com/shugo-avatars/user-uuid-123_thumb.png",
    "size": 2097152,
    "mimeType": "image/png"
  },
  "metadata": {
    "traceId": "opentelemetry-trace-id",
    "priority": "NORMAL"
  }
}
```

**Consumidores:**
1. **CDN:** Invalida caché del avatar anterior.
2. **Sistema de Notificaciones:** Envía notificación push si el usuario tiene habilitadas las notificaciones de cambios de perfil.

---

## 6. Consideraciones de Arquitectura y Seguridad

### 6.1. Ownership Validation (Prevención de Acceso No Autorizado)

El sistema debe validar que el usuario solo pueda actualizar su propio perfil:

```csharp
public class ProfileHandler : IProfileHandler
{
    public async Task<ProfileUpdateResult> UpdateProfile(
        UpdateProfileCommand command, 
        CancellationToken ct)
    {
        // Validar que el identity_id del JWT coincida con el perfil a actualizar
        if (command.IdentityId != command.JwtIdentityId)
            throw new ForbiddenException("You can only update your own profile.");

        // ... continuar con actualización ...
    }
}
```

### 6.2. Validación de Avatar

El avatar debe cumplir restricciones estrictas para prevenir ataques (ej. upload de malware, imágenes maliciosas):

```csharp
public class AvatarValidator : IAvatarValidator
{
    private const int MAX_SIZE_BYTES = 5 * 1024 * 1024; // 5MB
    private const int MAX_WIDTH = 1024;
    private const int MAX_HEIGHT = 1024;
    private static readonly HashSet<string> ALLOWED_TYPES = new() { "image/png", "image/jpeg", "image/webp" };

    public async Task ValidateAsync(Stream avatarStream, string mimeType, CancellationToken ct)
    {
        // Validar tamaño
        if (avatarStream.Length > MAX_SIZE_BYTES)
            throw new DomainException($"Avatar size exceeds maximum allowed ({MAX_SIZE_BYTES / 1024 / 1024}MB).");

        // Validar tipo
        if (!ALLOWED_TYPES.Contains(mimeType))
            throw new DomainException($"Invalid avatar type. Allowed types: {string.Join(", ", ALLOWED_TYPES)}");

        // Validar dimensiones (requiere procesamiento de imagen)
        using var image = await Image.LoadAsync(avatarStream, ct);
        if (image.Width > MAX_WIDTH || image.Height > MAX_HEIGHT)
            throw new DomainException($"Avatar dimensions exceed maximum allowed ({MAX_WIDTH}x{MAX_HEIGHT}px).");
    }
}
```

### 6.3. Integración con UC-09 (Verificación de Email)

Si el usuario intenta cambiar su email, se dispara el flujo de verificación:

```csharp
public class ProfileHandler : IProfileHandler
{
    private readonly IEmailVerificationService _emailVerificationService; // UC-09

    public async Task<ProfileUpdateResult> UpdateProfile(
        UpdateProfileCommand command, 
        CancellationToken ct)
    {
        // ... validaciones ...

        // Si el email cambió, disparar UC-09
        if (command.Email != currentProfile.Email)
        {
            await _emailVerificationService.InitiateVerificationAsync(
                command.IdentityId, 
                command.Email, 
                ct);
            
            return new ProfileUpdateResult(
                message: "To change your email, please verify the new email.",
                emailVerificationPending: true);
        }

        // ... continuar con actualización normal ...
    }
}
```

### 6.4. Auditoría Completa

Cada actualización genera un log de auditoría con los cambios específicos:

```csharp
public class AuditLogger : IAuditLogger
{
    public async Task LogProfileUpdateAsync(
        IdentityId identityId,
        List<FieldChange> changes,
        string ipAddress,
        string userAgent,
        CancellationToken ct)
    {
        var auditLog = new AuditLog
        {
            IdentityId = identityId,
            Action = "Profile.Updated",
            Timestamp = DateTime.UtcNow,
            IpAddress = ipAddress,
            UserAgent = userAgent,
            SnapshotJson = JsonSerializer.Serialize(changes)
        };

        await _auditRepository.SaveAsync(auditLog, ct);
    }
}

public class FieldChange
{
    public string Field { get; set; }
    public string OldValue { get; set; }
    public string NewValue { get; set; }
}
```

### 6.5. Rate Limiting Específico

Para prevenir spam de actualizaciones:

```csharp
public class ProfileRateLimiter
{
    private readonly IDistributedCache _cache;
    private const int MAX_UPDATES_PER_HOUR = 10;

    public async Task<bool> IsAllowedAsync(IdentityId identityId, CancellationToken ct)
    {
        var key = $"rate_limit:profile:{identityId.Value}";
        var count = await _cache.GetAsync<int>(key, ct);

        if (count >= MAX_UPDATES_PER_HOUR)
            return false;

        await _cache.IncrementAsync(key, TimeSpan.FromHours(1), ct);
        return true;
    }
}
```

---

## 7. Contratos de API (OpenAPI 3.0 Snippets)

### 7.1. Endpoint: PATCH /profile

**Request:**
```http
PATCH /v1/profile HTTP/1.1
Host: api.shugo.io
Authorization: Bearer <JWT>
Content-Type: application/json

{
  "display_name": "Nuevo Nombre",
  "language": "es",
  "timezone": "America/Argentina/Buenos_Aires",
  "preferences": {
    "notifications": {
      "email": true,
      "push": false
    },
    "theme": "dark"
  }
}
```

**Response (200 OK):**
```json
{
  "identity_id": "user-uuid-123",
  "display_name": "Nuevo Nombre",
  "email": "user@khcs.com",
  "avatar_url": "https://s3.amazonaws.com/shugo-avatars/user-uuid-123.png",
  "language": "es",
  "timezone": "America/Argentina/Buenos_Aires",
  "preferences": {
    "notifications": {
      "email": true,
      "push": false
    },
    "theme": "dark"
  },
  "message": "Profile updated successfully"
}
```

**Response (400 Bad Request - Campo no permitido):**
```json
{
  "error": {
    "code": "FIELD_NOT_ALLOWED",
    "message": "Field 'status' is not allowed for self-service updates.",
    "allowed_fields": ["display_name", "avatar", "preferences", "language", "timezone"]
  }
}
```

**Response (400 Bad Request - Avatar inválido):**
```json
{
  "error": {
    "code": "INVALID_AVATAR",
    "message": "Avatar size exceeds maximum allowed (5MB)."
  }
}
```

**Response (403 Forbidden - Mismatch de identity_id):**
```json
{
  "error": {
    "code": "FORBIDDEN",
    "message": "You can only update your own profile."
  }
}
```

**Response (429 Too Many Requests - Rate limit excedido):**
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Profile update limit reached. Please try again in 1 hour."
  }
}
```

### 7.2. Endpoint: POST /profile/avatar

**Request:**
```http
POST /v1/profile/avatar HTTP/1.1
Host: api.shugo.io
Authorization: Bearer <JWT>
Content-Type: multipart/form-data

--boundary
Content-Disposition: form-data; name="avatar"; filename="avatar.png"
Content-Type: image/png

[binary data]
--boundary--
```

**Response (200 OK):**
```json
{
  "avatar_url": "https://s3.amazonaws.com/shugo-avatars/user-uuid-123.png",
  "thumbnail_url": "https://s3.amazonaws.com/shugo-avatars/user-uuid-123_thumb.png",
  "message": "Avatar uploaded successfully"
}
```

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `02_Use_Case_Diagram.md` | Mostrará el límite del sistema, el actor (Usuario Autenticado), y las relaciones con UC-09 (Verificación de Email) y el sistema de almacenamiento de avatares. |
| `03_Activity_Diagram.md` | Visualizará el flujo de validación, actualización de perfil, validación de avatar, y generación de log de auditoría. |
| `04_Initial_Domain_Model.md` | Definirá el Profile Handler, el Avatar Validator, el Email Verification Service (UC-09), y las reglas de negocio. |
| `05_State_Machine_Diagram.md` | Mostrará la máquina de estados del perfil (aunque es simple, puede incluir estados de verificación de email). |
| `09_Modern_Artifacts.md` | Definirá los contratos OpenAPI completos, métricas de OpenTelemetry, y patrones de resiliencia. |
| `UC-02/01_Requirements_Specification.md` | Documento hermano que define la actualización por admin (alternativa al self-service). |
| `UC-09/01_Requirements_Specification.md` | Documento que define la verificación de contactos (integrado con cambio de email). |
| `UC-11/01_Requirements_Specification.md` | Documento que define la autenticación (predecesor obligatorio). |

---

## 9. Glosario de Términos Específicos de UC-08

| Término | Definición |
| :--- | :--- |
| **Self-Service Profile** | Gestión del propio perfil por parte del usuario autenticado, con campos limitados y validaciones específicas. |
| **Ownership Validation** | Validación de que el usuario solo puede actualizar su propio perfil (comparación de `sub` claim con `identity_id`). |
| **Avatar Storage** | Almacenamiento de avatares en S3/Azure Blob con generación de thumbnails y URLs públicas. |
| **Field Change Audit** | Registro de auditoría que captura el valor anterior y nuevo de cada campo modificado. |
| **GDPR Art. 16** | Derecho de Rectificación: el usuario tiene derecho a corregir datos personales inexactos. |
| **Rate Limiting** | Mecanismo para limitar el número de actualizaciones de perfil por usuario por hora (máx. 10/hora). |
| **No-Op Detection** | Detección de actualizaciones sin cambios reales para evitar logs de auditoría innecesarios. |
| **Email Verification Flow** | Flujo de UC-09 que se dispara cuando el usuario intenta cambiar su email. |
| **Thumbnail Generation** | Generación de imagen reducida (200x200px) del avatar para optimizar carga en listas y perfiles. |
| **ISO 639-1** | Estándar internacional para códigos de idioma de dos letras (ej. "en" para inglés, "es" para español). |
| **IANA Timezone** | Base de datos de zonas horarias mantenida por IANA (ej. "America/New_York", "Europe/London"). |

---
*Fin del Documento 01_Requirements_Specification para UC-08*