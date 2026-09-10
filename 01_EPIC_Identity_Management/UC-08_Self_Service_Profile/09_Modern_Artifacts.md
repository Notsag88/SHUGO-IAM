# UC-08: Self-Service Profile - Artefactos Modernos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-08 - Self-Service Profile (User-Initiated Profile Management) |
| **Documento:** | 09_Modern_Artifacts.md |
| **Metodología:** | API-First + Observability + Resilience Patterns + DevSecOps + GDPR Compliance by Design |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 09/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 09/09/2026 | Gastón Ruiz | **Creación inicial.** Definición de los artefactos modernos para UC-08: contratos OpenAPI 3.1 con endpoint PATCH /profile, catálogo de métricas OpenTelemetry (enfocado en detección de No-Op, latencia de ownership validation, y tasa de cambios de email), patrones de resiliencia (Circuit Breaker para Email Service, Retry policies para Avatar Storage), estrategias de testing para escenarios de seguridad (IDOR prevention, No-Op detection), y consideraciones de deployment para operaciones de autonomía del usuario. | Cierra el paquete de diseño Tier 1 funcional del UC-08, proporcionando todos los artefactos necesarios para la implementación, operación, monitoreo y cumplimiento normativo (GDPR Art. 16 - Derecho de Rectificación) del endpoint de gestión self-service del perfil del motor KAGAMI. |

---

## 1. Introducción y Propósito

Este documento define los **Artefactos Modernos** necesarios para operar el UC-08 (Self-Service Profile) en un entorno de producción enterprise. A diferencia de los UCs administrados (UC-01 a UC-05), el UC-08 introduce desafíos únicos debido a su naturaleza de **autonomía del usuario con restricciones de seguridad**:

1. **Ownership Validation Crítico:** Cada request debe validar que el usuario solo modifique su propio perfil, previniendo IDOR (Insecure Direct Object Reference).
2. **Detección de No-Op:** El sistema debe detectar actualizaciones sin cambios reales para optimizar recursos y evitar ruido en auditoría.
3. **Integración Condicional con UC-09:** Si el usuario cambia su email, se dispara el flujo de verificación de contactos.
4. **Upload Seguro de Avatares:** Validación de tamaño, tipo y dimensiones para prevenir upload de malware.
5. **Auditoría Granular (GDPR Art. 16):** Cada cambio se registra con valor anterior, nuevo valor, timestamp, IP, y user agent.

---

## 2. Contratos OpenAPI Completos

### 2.1. Especificación OpenAPI 3.1

```yaml
openapi: 3.1.0
info:
  title: SHUGO KAGAMI - Self-Service Profile API
  description: |
    API de gestión self-service del perfil de usuario del motor KAGAMI.
    
    ## Características Clave
    - Autonomía del usuario con validación estricta de ownership (JWT sub == identity_id).
    - Campos limitados (display_name, avatar, preferences, language, timezone).
    - Detección de No-Op para optimización de recursos.
    - Integración condicional con UC-09 para cambios de email.
    - Auditoría granular con valor anterior y nuevo (GDPR Art. 16).
    
    ## Seguridad
    - Endpoint autenticado (JWT requerido).
    - Prevención de IDOR mediante ownership validation.
    - Rate limiting: 10 actualizaciones por usuario por hora.
    - Validación de avatar: <5MB, PNG/JPG/WebP, <1024x1024px.
  version: 1.0.0

servers:
  - url: https://api.shugo.io/v1
    description: Producción

paths:
  /profile:
    patch:
      operationId: updateProfile
      summary: Actualizar perfil propio (Self-Service)
      description: |
        Actualiza el perfil del usuario autenticado con campos limitados.
        Requiere JWT válido con claim sub coincidente con identity_id.
      tags:
        - Profile
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UpdateProfileRequest'
            examples:
              standardUpdate:
                summary: Actualización estándar de perfil
                value:
                  display_name: "Nuevo Nombre"
                  language: "es"
                  timezone: "America/Argentina/Buenos_Aires"
                  preferences:
                    notifications:
                      email: true
                      push: false
                    theme: "dark"
              avatarUpdate:
                summary: Upload de avatar (multipart/form-data)
                value:
                  display_name: "Nuevo Nombre"
                  # El avatar se envía como archivo separado
      responses:
        '200':
          description: Perfil actualizado exitosamente (o No-Op detectado)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ProfileUpdateResponse'
        '202':
          description: Email verification sent (cambio de email requiere verificación)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/EmailVerificationResponse'
        '400':
          description: Error de validación (campo no permitido, avatar inválido)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '401':
          description: No autorizado (JWT inválido o expirado)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Prohibido (IDOR attempt - ownership mismatch)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '429':
          description: Demasiadas solicitudes (rate limit excedido)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '500':
          description: Error interno (fallo en persistencia o auditoría)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    UpdateProfileRequest:
      type: object
      properties:
        display_name:
          type: string
          minLength: 2
          maxLength: 100
          description: "Nombre para mostrar (alphanumeric, spaces, hyphens, underscores, dots, @)"
        avatar:
          type: string
          format: binary
          description: "Archivo de avatar (PNG/JPG/WebP, <5MB, <1024x1024px)"
        preferences:
          type: object
          properties:
            notifications:
              type: object
              properties:
                email:
                  type: boolean
                push:
                  type: boolean
            theme:
              type: string
              enum: [light, dark, system]
          description: "Preferencias del usuario en formato JSON"
        language:
          type: string
          pattern: "^[a-z]{2}$"
          example: "es"
          description: "Código de idioma ISO 639-1 (2 caracteres)"
        timezone:
          type: string
          example: "America/New_York"
          description: "Nombre de zona horaria IANA"
        email:
          type: string
          format: email
          description: "Nuevo email (requiere verificación vía UC-09)"

    ProfileUpdateResponse:
      type: object
      properties:
        identity_id:
          type: string
          format: uuid
        display_name:
          type: string
        email:
          type: string
          format: email
        avatar_url:
          type: string
          format: uri
        language:
          type: string
        timezone:
          type: string
        preferences:
          type: object
        message:
          type: string
          example: "Profile updated successfully"
        changes:
          type: array
          items:
            $ref: '#/components/schemas/FieldChange'

    EmailVerificationResponse:
      type: object
      properties:
        message:
          type: string
          example: "Email verification sent. Please verify your new email."
        email_verification_pending:
          type: boolean
          example: true

    FieldChange:
      type: object
      properties:
        field:
          type: string
          example: "display_name"
        old_value:
          type: string
          example: "Old Name"
        new_value:
          type: string
          example: "New Name"

    ErrorResponse:
      type: object
      properties:
        error:
          type: object
          properties:
            code:
              type: string
              enum:
                - INVALID_JWT
                - IDOR_ATTEMPT
                - FIELD_NOT_ALLOWED
                - INVALID_AVATAR
                - RATE_LIMIT_EXCEEDED
                - PROFILE_NOT_FOUND
                - CONCURRENCY_CONFLICT
                - UPDATE_FAILED
            message:
              type: string
            details:
              type: object
```

### 2.2. Códigos de Error Específicos de UC-08

| Código HTTP | Código de Error | Descripción | Cuándo Ocurre |
| :--- | :--- | :--- | :--- |
| **400** | `FIELD_NOT_ALLOWED` | Campo no está en whitelist (ej. `status`, `roles`). | `SecurityValidator.ValidateAllowedFields()` detecta campo prohibido. |
| **400** | `INVALID_AVATAR` | Avatar excede tamaño, tipo o dimensiones permitidos. | `AvatarValidator.ValidateAsync()` falla. |
| **401** | `INVALID_JWT` | JWT inválido o expirado. | API Gateway valida JWT y falla. |
| **403** | `IDOR_ATTEMPT` | Intento de actualizar perfil de otro usuario (ownership mismatch). | `SecurityValidator.ValidateOwnershipAsync()` detecta `JWT sub != identity_id`. |
| **404** | `PROFILE_NOT_FOUND` | El ID de perfil no existe en el tenant. | `ProfileRepository.GetByIdAsync()` retorna null. |
| **409** | `CONCURRENCY_CONFLICT` | El perfil fue modificado por otra sesión simultáneamente (OCC failure). | `DbUpdateConcurrencyException` en EF Core. |
| **429** | `RATE_LIMIT_EXCEEDED` | Usuario excedió 10 actualizaciones por hora. | `SecurityValidator.ValidateRateLimitAsync()` detecta límite. |
| **500** | `UPDATE_FAILED` | Fallo en persistencia o auditoría. | Excepción no manejada en `ProfileRepository.SaveAsync()` o `AuditLogger.LogProfileUpdateAsync()`. |

---

## 3. Catálogo de Métricas OpenTelemetry

### 3.1. Métricas de Negocio y Autonomía del Usuario

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.profile.update_attempts_total` | Counter | attempts | Total de intentos de actualización de perfil | `tenant_id`, `user_id` |
| `kagami.profile.update_success_total` | Counter | successes | Total de actualizaciones exitosas | `tenant_id`, `has_changes` |
| `kagami.profile.no_op_detected_total` | Counter | no_ops | Total de detecciones de No-Op (sin cambios reales) | `tenant_id` |
| `kagami.profile.email_change_requested_total` | Counter | requests | Total de solicitudes de cambio de email | `tenant_id` |
| `kagami.profile.avatar_uploaded_total` | Counter | uploads | Total de avatares subidos exitosamente | `tenant_id`, `mime_type` |
| `kagami.profile.idor_attempts_total` | Counter | blocks | Total de intentos de IDOR bloqueados | `tenant_id`, `source_ip` |

### 3.2. Métricas de Rendimiento y Seguridad

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.profile.update_latency` | Histogram | ms | Latencia end-to-end de actualización de perfil | `tenant_id`, `success`, `has_changes` |
| `kagami.profile.ownership_validation_latency` | Histogram | ms | Latencia de validación de ownership (JWT sub == identity_id) | `tenant_id`, `success` |
| `kagami.profile.avatar_validation_latency` | Histogram | ms | Latencia de validación de avatar (tamaño, tipo, dimensiones) | `tenant_id`, `success` |
| `kagami.profile.avatar_upload_latency` | Histogram | ms | Latencia de upload de avatar a S3/Azure Blob | `tenant_id`, `success` |
| `kagami.profile.audit_log_latency` | Histogram | ms | Latencia de generación de log de auditoría | `tenant_id`, `success` |
| `kagami.profile.rate_limit_cache_latency` | Histogram | ms | Latencia de consulta a Redis para rate limiting | `tenant_id`, `cache_hit` |

### 3.3. Métricas de Compliance y Auditoría

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.profile.audit_logs_generated_total` | Counter | logs | Total de logs de auditoría generados | `tenant_id`, `action` |
| `kagami.profile.fields_changed_total` | Counter | fields | Total de campos modificados (para GDPR Art. 16 tracking) | `tenant_id`, `field_name` |
| `kagami.profile.email_verifications_initiated_total` | Counter | verifications | Total de verificaciones de email iniciadas (UC-09) | `tenant_id` |

### 3.4. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Alta Tasa de IDOR Attempts** | `idor_attempts_total > 10` en 5 min | **Crítica** | Posible ataque de escalación horizontal. Revisar IPs y usuarios sospechosos. |
| **No-Op Rate Alto** | `no_op_detected_total / update_attempts_total > 0.50` en 1h | Media | Los usuarios están enviando los mismos datos. Revisar UX del formulario. |
| **Avatar Upload Failures** | `avatar_uploaded_total` con `success=false > 20%` | Alta | Validaciones de avatar demasiado estrictas o problemas en S3/Azure. |
| **Ownership Validation Slow** | `ownership_validation_latency_p95 > 100ms` | Media | Consulta de perfil lenta. Revisar índices de DB. |
| **Audit Log Failures** | `audit_logs_generated_total` con `success=false > 0` | **Crítica** | Fallo en auditoría viola GDPR Art. 16. Investigar inmediatamente. |
| **Rate Limiting Excessive** | `rate_limit_exceeded_total > 50` en 10 min | Media | Posible abuso o bug en cliente. Revisar usuarios con más requests. |

---

## 4. Patrones de Resiliencia

### 4.1. Circuit Breaker para Email Service (UC-09 Integration)

Dado que el cambio de email depende del Email Service (UC-09), implementamos Circuit Breaker para prevenir cascadas de fallos.

**Implementación (Polly - C#):**

```csharp
public class ResilientEmailVerificationService : IEmailVerificationService
{
    private readonly IEmailVerificationService _inner;
    private readonly AsyncCircuitBreakerPolicy _circuitBreaker;
    private readonly ILogger<ResilientEmailVerificationService> _logger;

    public ResilientEmailVerificationService(
        IEmailVerificationService inner, 
        ILogger<ResilientEmailVerificationService> logger)
    {
        _inner = inner;
        _logger = logger;
        
        _circuitBreaker = Policy
            .Handle<EmailServiceException>()
            .Or<TimeoutException>()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 3,
                durationOfBreak: TimeSpan.FromMinutes(5),
                onBreak: (ex, duration) => 
                {
                    _logger.LogWarning("Email Verification Circuit Breaker OPEN for {Duration}. Email changes will be rejected.", duration);
                    Metrics.EmailCircuitBreakerState.Set(1); // Open
                },
                onReset: () => 
                {
                    _logger.LogInformation("Email Verification Circuit Breaker CLOSED. Email changes resumed.");
                    Metrics.EmailCircuitBreakerState.Set(0); // Closed
                }
            );
    }

    public async Task InitiateVerificationAsync(IdentityId identityId, string newEmail, CancellationToken ct)
    {
        try
        {
            await _circuitBreaker.ExecuteAsync(() => 
                _inner.InitiateVerificationAsync(identityId, newEmail, ct));
        }
        catch (BrokenCircuitException)
        {
            _logger.LogWarning("Circuit breaker open. Email verification cannot be initiated for {IdentityId}.", identityId);
            throw new EmailVerificationUnavailableException(
                "Email verification service unavailable. Please try again later.", ex);
        }
    }
}
```

### 4.2. Retry Policy con Backoff Exponencial para Avatar Storage

Para fallos transitorios en S3/Azure Blob, implementamos retry con backoff exponencial.

**Implementación (Polly - C#):**

```csharp
public class ResilientAvatarStorage : IAvatarStorage
{
    private readonly IAvatarStorage _inner;
    private readonly IAsyncPolicy<AvatarUrls> _retryPolicy;
    private readonly ILogger<ResilientAvatarStorage> _logger;

    public ResilientAvatarStorage(IAvatarStorage inner, ILogger<ResilientAvatarStorage> logger)
    {
        _inner = inner;
        _logger = logger;
        
        _retryPolicy = Policy
            .Handle<StorageException>()
            .Or<TimeoutException>()
            .WaitAndRetryAsync(
                retryCount: 3,
                sleepDurationProvider: retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)), // 2s, 4s, 8s
                onRetry: (ex, timeSpan, retryCount, context) =>
                {
                    _logger.LogWarning(ex, "Retry attempt {RetryCount} for avatar upload after {Delay}s", 
                        retryCount, timeSpan.TotalSeconds);
                    Metrics.AvatarUploadRetryAttempts.Add(1, 
                        new KeyValuePair<string, object?>("retry_count", retryCount));
                }
            );
    }

    public async Task<AvatarUrls> UploadAsync(Stream stream, string mimeType, CancellationToken ct)
    {
        return await _retryPolicy.ExecuteAsync(() => 
            _inner.UploadAsync(stream, mimeType, ct));
    }
}
```

### 4.3. Timeout Estricto para Ownership Validation

Para prevenir que la validación de ownership bloquee threads indefinidamente, configuramos timeouts estrictos.

**Configuración (C#):**

```csharp
// Configuración de timeouts para validación de ownership
services.AddDbContext<AppDbContext>(options =>
{
    options.UseNpgsql(configuration.GetConnectionString("DefaultConnection"));
    options.CommandTimeout(5); // 5 segundos máximo para consultas de perfil
});

// Configuración de Polly para timeout
services.AddPolicyHandler(Policy.TimeoutAsync<Profile>(TimeSpan.FromSeconds(5)));
```

### 4.4. Detección de No-Op (Optimización de Recursos)

El patrón de detección de No-Op previene writes innecesarios en la base de datos y reduce el ruido en logs de auditoría.

**Implementación (C#):**

```csharp
public class ProfileHandler : IProfileHandler
{
    public async Task<ProfileUpdateResult> UpdateProfile(
        UpdateProfileCommand command, 
        IdentityId jwtIdentityId,
        CancellationToken ct)
    {
        // 1. Validaciones de seguridad
        await _securityValidator.ValidateOwnershipAsync(command.ProfileId, jwtIdentityId, ct);
        await _securityValidator.ValidateRateLimitAsync(jwtIdentityId, ct);
        _securityValidator.ValidateAllowedFields(command.GetFieldNames());

        // 2. Cargar perfil actual
        var profile = await _repository.GetByIdAsync(command.ProfileId, ct);
        if (profile == null)
            throw new ProfileNotFoundException($"Profile {command.ProfileId} not found.");

        // 3. Detección de No-Op (CRÍTICO para optimización)
        var changes = profile.CalculateChanges(command);
        if (!changes.Any())
        {
            _logger.LogInformation("No changes detected for profile {ProfileId} - No-Op", command.ProfileId);
            Metrics.NoOpDetectedTotal.Add(1, 
                new KeyValuePair<string, object?>("tenant_id", profile.TenantId.Value.ToString()));
            
            return new ProfileUpdateResult(
                message: "No changes detected", 
                hasChanges: false);
        }

        // 4. Continuar con actualización...
        // ...
    }
}
```

---

## 5. Estrategia de Testing

### 5.1. Unit Tests (Dominio y Validaciones)

```csharp
[Test]
public void Profile_CalculateChanges_WithSameValues_ReturnsEmptyList_NoOp()
{
    // Arrange
    var profile = Profile.Create(
        new ProfileId(Guid.NewGuid()),
        new IdentityId(Guid.NewGuid()),
        new TenantId(Guid.NewGuid()),
        "user@khcs.com",
        new DisplayName("Test User"),
        new Preferences(JsonDocument.Parse("{}")),
        new Language("en"),
        new Timezone("UTC"));

    var command = new UpdateProfileCommand
    {
        ProfileId = profile.Id,
        DisplayName = "Test User", // Mismo valor
        Language = "en"            // Mismo valor
    };

    // Act
    var changes = profile.CalculateChanges(command);

    // Assert
    changes.Should().BeEmpty();
}

[Test]
public void Profile_ValidateOwnership_WithDifferentIdentityId_ThrowsForbidden()
{
    // Arrange
    var profile = Profile.Create(
        new ProfileId(Guid.NewGuid()),
        new IdentityId(Guid.NewGuid()),
        new TenantId(Guid.NewGuid()),
        "user@khcs.com",
        new DisplayName("Test User"),
        new Preferences(JsonDocument.Parse("{}")),
        new Language("en"),
        new Timezone("UTC"));

    var attackerIdentityId = new IdentityId(Guid.NewGuid());

    // Act
    var act = () => profile.ValidateOwnership(attackerIdentityId);

    // Assert
    act.Should().Throw<ForbiddenException>()
       .WithMessage("*cannot modify profile*");
}

[Test]
public void SecurityValidator_ValidateAllowedFields_WithStatus_ThrowsDomainException()
{
    // Arrange
    var validator = new SecurityValidator(
        Mock.Of<IProfileRepository>(),
        Mock.Of<IDistributedCache>());
    
    var fields = new List<string> { "display_name", "status" }; // 'status' no está permitido

    // Act
    var act = () => validator.ValidateAllowedFields(fields);

    // Assert
    act.Should().Throw<DomainException>()
       .WithMessage("*not allowed*");
}

[Test]
public void Avatar_WithExceededSize_ThrowsDomainException()
{
    // Act
    var act = () => new Avatar(
        "https://s3.amazonaws.com/test.png",
        "https://s3.amazonaws.com/test_thumb.png",
        "image/png",
        6 * 1024 * 1024, // 6MB (exceeds 5MB limit)
        800,
        800);

    // Assert
    act.Should().Throw<DomainException>()
       .WithMessage("*exceeds maximum allowed*");
}
```

### 5.2. Integration Tests (IDOR Prevention y No-Op Detection)

```csharp
[Test]
public async Task ProfileHandler_WhenOwnershipMismatch_ThrowsForbidden_IDORPrevention()
{
    // Arrange
    var mockRepo = new Mock<IProfileRepository>();
    var profile = Profile.Create(
        new ProfileId(Guid.NewGuid()),
        new IdentityId(Guid.NewGuid()), // Owner real
        new TenantId(Guid.NewGuid()),
        "user@khcs.com",
        new DisplayName("Test User"),
        new Preferences(JsonDocument.Parse("{}")),
        new Language("en"),
        new Timezone("UTC"));
    
    mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<ProfileId>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(profile);

    var handler = new ProfileHandler(
        new SecurityValidator(mockRepo.Object, Mock.Of<IDistributedCache>()),
        Mock.Of<IAvatarValidator>(),
        Mock.Of<IAuditLogger>(),
        Mock.Of<IEmailVerificationService>(),
        mockRepo.Object,
        Mock.Of<IAvatarStorage>(),
        Mock.Of<ICDNService>(),
        Mock.Of<ILogger<ProfileHandler>>());

    var attackerIdentityId = new IdentityId(Guid.NewGuid()); // Atacante
    var command = new UpdateProfileCommand
    {
        ProfileId = profile.Id,
        DisplayName = "Hacked Name"
    };

    // Act & Assert
    var act = () => handler.UpdateProfile(command, attackerIdentityId, CancellationToken.None);
    await act.Should().ThrowAsync<ForbiddenException>();
}

[Test]
public async Task ProfileHandler_WhenNoChangesDetected_ReturnsNoOpResult()
{
    // Arrange
    var mockRepo = new Mock<IProfileRepository>();
    var profile = Profile.Create(
        new ProfileId(Guid.NewGuid()),
        new IdentityId(Guid.NewGuid()),
        new TenantId(Guid.NewGuid()),
        "user@khcs.com",
        new DisplayName("Test User"),
        new Preferences(JsonDocument.Parse("{}")),
        new Language("en"),
        new Timezone("UTC"));
    
    mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<ProfileId>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(profile);

    var handler = new ProfileHandler(
        Mock.Of<ISecurityValidator>(),
        Mock.Of<IAvatarValidator>(),
        Mock.Of<IAuditLogger>(),
        Mock.Of<IEmailVerificationService>(),
        mockRepo.Object,
        Mock.Of<IAvatarStorage>(),
        Mock.Of<ICDNService>(),
        Mock.Of<ILogger<ProfileHandler>>());

    var command = new UpdateProfileCommand
    {
        ProfileId = profile.Id,
        DisplayName = "Test User" // Mismo valor
    };

    // Act
    var result = await handler.UpdateProfile(command, profile.IdentityId, CancellationToken.None);

    // Assert
    result.HasChanges.Should().BeFalse();
    result.Message.Should().Be("No changes detected");
    
    // Verificar que NO se llamó SaveAsync (No-Op)
    mockRepo.Verify(r => r.SaveAsync(It.IsAny<Profile>(), It.IsAny<CancellationToken>()), Times.Never);
}
```

### 5.3. Load Testing (Picos de Actualizaciones de Perfil)

**Objetivo:** Validar que el sistema maneja 200 actualizaciones/minuto sin degradar el rendimiento.

**Script k6 (Extracto):**

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');
const noOpRate = new Rate('no_ops');

export const options = {
  scenarios: {
    profile_updates: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 50 },   // Calentamiento
        { duration: '3m', target: 200 },  // Pico de actualizaciones
        { duration: '1m', target: 200 },  // Mantener pico
        { duration: '1m', target: 0 },    // Enfriamiento
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<200'], // SLO de 200ms
    errors: ['rate<0.01'],             // Menos del 1% de errores
    no_ops: ['rate<0.30'],             // Menos del 30% de No-Ops (usuarios enviando mismos datos)
  },
};

export default function () {
  const payload = JSON.stringify({
    display_name: `User_${__VU}_${__ITER}`,
    language: 'es',
    timezone: 'America/Argentina/Buenos_Aires',
    preferences: {
      notifications: {
        email: true,
        push: false
      },
      theme: 'dark'
    }
  });

  const res = http.patch(`${__ENV.BASE_URL}/v1/profile`, payload, {
    headers: { 
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${__ENV.USER_TOKEN}`
    },
  });

  const success = check(res, {
    'profile updated (200 or 202)': (r) => r.status === 200 || r.status === 202,
    'has message': (r) => r.json('message') !== undefined,
  });

  if (!success) errorRate.add(1);
  
  // Detectar No-Op
  if (res.json('message') === 'No changes detected') {
    noOpRate.add(1);
  }

  sleep(0.3); // ~200 actualizaciones/min por VU
}
```

---

## 6. Deployment y Operaciones

### 6.1. Feature Flags para Rollout Gradual

| Feature Flag | Descripción | Valor por Defecto |
| :--- | :--- | :--- |
| `kagami.profile.self_service_enabled` | Activa/desactiva la gestión self-service del perfil globalmente. | `true` |
| `kagami.profile.no_op_detection_enabled` | Activa la detección de No-Op para optimización de recursos. | `true` |
| `kagami.profile.avatar_upload_enabled` | Activa el upload de avatares. | `true` |
| `kagami.profile.email_change_enabled` | Activa el cambio de email (requiere UC-09). | `true` |
| `kagami.profile.granular_audit_enabled` | Activa auditoría granular con valor anterior y nuevo (GDPR Art. 16). | `true` |
| `kagami.profile.rate_limiting_enabled` | Activa rate limiting de 10 actualizaciones/hora. | `true` |

### 6.2. Configuración de Avatar Storage

```yaml
# Configuración de S3/Azure Blob para avatares
avatar_storage:
  provider: "s3" # o "azure_blob"
  bucket_name: "shugo-avatars"
  region: "us-east-1"
  max_size_bytes: 5242880 # 5MB
  allowed_mime_types:
    - "image/png"
    - "image/jpeg"
    - "image/webp"
  max_dimensions:
    width: 1024
    height: 1024
  thumbnail_size:
    width: 200
    height: 200
  cdn_enabled: true
  cdn_domain: "avatars.shugo.io"
```

### 6.3. Configuración de Rate Limiting

```yaml
# Configuración de rate limiting para actualizaciones de perfil
rate_limiting:
  profile_updates:
    max_requests: 10
    window_minutes: 60
    key_prefix: "rate_limit:profile:"
    cache_ttl_minutes: 60
```

### 6.4. Escalado Horizontal de Profile Handlers

```yaml
# Kubernetes HPA para Profile Handlers
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kagami-profile-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kagami-profile
  minReplicas: 3
  maxReplicas: 15
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Pods
      pods:
        metric:
          name: kagami_profile_update_latency
        target:
          type: AverageValue
          averageValue: "150" # Escalar si latencia p95 > 150ms
    - type: External
      external:
        metric:
          name: kagami_profile_update_attempts_total
        target:
          type: AverageValue
          averageValue: "200" # Escalar si >200 actualizaciones/min por pod
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

### 6.5. Runbook de Operaciones

**Escenario: Alta Tasa de IDOR Attempts**

1. **Paso 1:** Verificar alerta `idor_attempts_total > 10` en 5 min.
2. **Paso 2:** Revisar logs de `SecurityValidator.ValidateOwnershipAsync()` para identificar IPs y usuarios sospechosos.
3. **Paso 3:** Bloquear IPs ofensivas en WAF.
4. **Paso 4:** Notificar al equipo de seguridad para investigación forense.
5. **Paso 5:** Monitorear métricas `idor_attempts_total` para confirmar que el ataque cesó.

**Escenario: No-Op Rate Alto**

1. **Paso 1:** Verificar métrica `no_op_detected_total / update_attempts_total > 0.50` en 1h.
2. **Paso 2:** Revisar UX del formulario de perfil. ¿Los usuarios están enviando los mismos datos repetidamente?
3. **Paso 3:** Considerar mejorar la UX para prevenir envíos innecesarios (ej. deshabilitar botón "Guardar" si no hay cambios).
4. **Paso 4:** Monitorear métricas `no_op_detected_total` para confirmar mejora.

**Escenario: Audit Log Failures**

1. **Paso 1:** Verificar alerta `audit_logs_generated_total` con `success=false > 0`.
2. **Paso 2:** Revisar logs de `AuditLogger.LogProfileUpdateAsync()` para identificar causa raíz.
3. **Paso 3:** Si la tabla `audit_logs` está llena, archivar logs antiguos.
4. **Paso 4:** **CRÍTICO:** Fallo en auditoría viola GDPR Art. 16. Investigar inmediatamente y notificar al DPO.
5. **Paso 5:** Monitorear métricas `audit_logs_generated_total` para confirmar recuperación.

---

## 7. Compliance y Auditoría

### 7.1. GDPR - Art. 16 (Right to Rectification)

*   **Requisito:** El usuario tiene derecho a obtener la rectificación de datos personales inexactos.
*   **Implementación UC-08:** El endpoint `PATCH /profile` permite a los usuarios actualizar su propia información personal (display_name, preferences, language, timezone). Cada cambio se registra con valor anterior y nuevo, proporcionando trazabilidad completa.

### 7.2. GDPR - Art. 5 (Principles relating to processing of personal data)

*   **Requisito:** Los datos personales deben ser exactos y, si es necesario, actualizados.
*   **Implementación UC-08:** La detección de No-Op previene actualizaciones innecesarias, manteniendo la integridad de los datos. La validación de campos (DisplayName, Language, Timezone) garantiza que los datos sean exactos.

### 7.3. SOC 2 - CC6.1 (Logical Access Controls)

*   **Requisito:** La organización debe restringir el acceso lógico a los datos según el rol del usuario.
*   **Implementación UC-08:** Solo el propio usuario puede actualizar su perfil (ownership validation). La whitelist de campos previene escalación de privilegios (ej. usuario no puede cambiar `status` o `roles`).

### 7.4. SOC 2 - CC7.2 (System Monitoring)

*   **Requisito:** La organización debe monitorear el sistema para detectar anomalías.
*   **Implementación UC-08:** Métricas de OpenTelemetry (`idor_attempts_total`, `no_op_detected_total`, `audit_logs_generated_total`) permiten detectar ataques de IDOR, abuso de No-Op, y fallos en auditoría.

### 7.5. ISO 27001 - A.9.4 (User and Entity Authentication)

*   **Requisito:** Se debe verificar la identidad de los usuarios antes de permitir el acceso a los datos.
*   **Implementación UC-08:** Validación de JWT con claim `sub` coincidente con `identity_id` del perfil. Prevención de IDOR mediante ownership validation en múltiples capas.

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los requisitos de ownership validation, whitelist de campos, detección de No-Op, y auditoría granular que estos artefactos implementan. |
| `08_Final_Class_Design.md` | Las clases `ProfileHandler`, `SecurityValidator`, `AvatarValidator`, y `AuditLogger` se implementan según los contratos y métricas definidos aquí. |
| `10_UC_Specific_Data_Model.md` | Define la estructura física de las tablas `profiles`, `audit_logs`, y índices para ownership validation. |
| `11_UC_Specific_Security_Threats.md` | Las mitigaciones de "IDOR Prevention", "No-Op Abuse", y "Avatar Upload Attacks" se validan mediante los patrones definidos aquí. |
| `12_UC_Specific_Performance_Benchmarks.md` | Los SLAs de latencia (<200ms p95) y throughput (200 actualizaciones/min) se validan mediante las métricas y load testing definidos aquí. |
| `SHUGO_Security_Architecture.md` (Transversal) | Este documento aplica los estándares globales de control de acceso y privacidad al contexto específico de UC-08. |

---

## 9. Cierre del Paquete de Diseño Funcional (UC-08)

Con este documento, completamos los **9 archivos del Paquete de Diseño Profundo Funcional (Tier 1)** para el UC-08:

1. ✅ `01_Requirements_Specification.md` - Especificación completa con 10 escenarios BDD.
2. ✅ `02_Use_Case_Diagram.md` - Diagrama de caso de uso UML con actor Usuario Autenticado.
3. ✅ `03_Activity_Diagram.md` - Diagrama de actividades con validaciones de seguridad y detección de No-Op.
4. ✅ `04_Initial_Domain_Model.md` - Modelo de dominio conceptual (Aggregate Root Profile, Value Objects).
5. ✅ `05_State_Machine_Diagram.md` - 3 máquinas de estados (Perfil, Verificación de Email, Upload de Avatar).
6. ✅ `06_Robustness_Diagram.md` - Diagrama de robustez ICONIX.
7. ✅ `07_Sequence_Diagram.md` - Diagrama de secuencia con flujos alternativos y detección de No-Op.
8. ✅ `08_Final_Class_Design.md` - Diseño de clases con firmas y patrones.
9. ✅ `09_Modern_Artifacts.md` - Contratos OpenAPI, métricas, resiliencia, compliance.

### 9.1. Resumen de lo Logrado

Hemos diseñado el endpoint de gestión self-service del perfil más robusto, seguro y compliant del mercado IAM:
- **Autonomía del Usuario:** Los usuarios pueden gestionar su propia información personal (GDPR Art. 16 - Derecho de Rectificación).
- **Seguridad por Diseño:** Ownership validation en múltiples capas previene IDOR. Whitelist de campos previene escalación de privilegios.
- **Optimización de Recursos:** Detección de No-Op evita writes innecesarios y ruido en auditoría.
- **Auditoría Granular:** Cada cambio se registra con valor anterior y nuevo, proporcionando trazabilidad forense completa.
- **Resiliencia Transaccional:** Circuit Breaker para Email Service y Retry policies para Avatar Storage garantizan disponibilidad.
- **Observabilidad Completa:** Métricas de IDOR attempts, No-Op detection, y audit log failures permiten monitoreo proactivo.

### 9.2. Próximos Pasos (Documentos Técnicos Específicos)

Según la **Opción A+** acordada, cada UC tiene **12 documentos** (9 funcionales + 3 técnicos específicos). Para el UC-08, aún faltan:

1. **`10_UC_Specific_Data_Model.md`** - Modelo de datos físico específico (tablas `profiles`, `audit_logs`, índices para ownership validation).
2. **`11_UC_Specific_Security_Threats.md`** - Análisis de amenazas específico (IDOR Prevention, No-Op Abuse, Avatar Upload Attacks).
3. **`12_UC_Specific_Performance_Benchmarks.md`** - Benchmarks y SLAs específicos para latencia de actualización, throughput, y tasa de No-Op.

---
*Fin del Documento 09_Modern_Artifacts para UC-08*