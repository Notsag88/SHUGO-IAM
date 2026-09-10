# UC-07: Invitación de Usuarios - Artefactos Modernos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-07 - Invitación de Usuarios (Admin-Initiated Invitation) |
| **Documento:** | 09_Modern_Artifacts.md |
| **Metodología:** | API-First + Observability + Resilience Patterns + Dual-Actor Async Flow + Compliance by Design |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 05/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 05/09/2026 | Gastón Ruiz | **Creación inicial.** Definición de los artefactos modernos para UC-07: contratos OpenAPI 3.1 con endpoints de creación, aceptación, reenvío y cancelación de invitaciones, catálogo de métricas OpenTelemetry (enfocado en conversión de invitaciones, latencia de envío de email, y tasa de compensación), patrones de resiliencia (Circuit Breaker para Email Service, retry policies con backoff exponencial), estrategias de testing para escenarios de flujo dual asíncrono, y consideraciones de deployment para operaciones de onboarding corporativo. | Cierra el paquete de diseño Tier 1 funcional del UC-07, proporcionando todos los artefactos necesarios para la implementación, operación, monitoreo y cumplimiento normativo (GDPR Art. 6 & 7) del endpoint de invitación del motor KAGAMI. |

---

## 1. Introducción y Propósito

Este documento define los **Artefactos Modernos** necesarios para operar el UC-07 (Invitación de Usuarios) en un entorno de producción enterprise. A diferencia de los UCs síncronos, el UC-07 introduce desafíos únicos debido a su naturaleza **dual-actor asíncrona**:

1. **Flujo Temporal Extendido:** Las invitaciones permanecen en estado `PENDING` durante días (TTL de 7 días), requiriendo monitoreo de conversión y expiración.
2. **Dependencia Crítica de Email:** El envío de email es un punto de falla que requiere patrón de compensación y Circuit Breaker.
3. **Integración con UC-06 y MURA:** La aceptación delega al flujo de registro (UC-06) y asignación de roles (MURA), requiriendo orquestación de múltiples servicios.
4. **Seguridad Criptográfica de Tokens:** Los tokens deben ser validados con comparación en tiempo constante para prevenir Timing Attacks.
5. **Job de Limpieza Automática:** Un proceso en background actualiza invitaciones expiradas, requiriendo monitoreo de salud del job.

---

## 2. Contratos OpenAPI Completos

### 2.1. Especificación OpenAPI 3.1

```yaml
openapi: 3.1.0
info:
  title: SHUGO KAGAMI - Invitation API
  description: |
    API de invitación de usuarios del motor KAGAMI (Admin-Initiated Onboarding).
    
    ## Características Clave
    - Flujo dual asíncrono: Admin crea invitación, Usuario Invitado acepta.
    - Tokens criptográficos seguros (UUID v4 + SHA256 hash) con TTL de 7 días.
    - Patrón de compensación transaccional si el email falla.
    - Integración con UC-06 (Registro Self-Service) y MURA (Asignación de Roles).
    - Prevención de enumeración de emails con mensajes genéricos.
    
    ## Seguridad
    - Endpoint de creación requiere permiso `user:invite`.
    - Endpoint de aceptación es público (token-based).
    - Tokens almacenados como hashes SHA256, nunca en texto plano.
  version: 1.0.0

servers:
  - url: https://api.shugo.io/v1
    description: Producción

paths:
  /invitations:
    post:
      operationId: createInvitation
      summary: Crear invitación de usuario (Admin)
      description: |
        Crea una nueva invitación para un usuario específico.
        Requiere permiso `user:invite`.
      tags:
        - Invitations
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateInvitationRequest'
            examples:
              standardInvitation:
                summary: Invitación estándar con roles
                value:
                  email: "nuevo.usuario@khcs.com"
                  roles: ["user", "developer"]
                  message: "Bienvenido al equipo!"
                  ttl_days: 7
      responses:
        '201':
          description: Invitación creada exitosamente, email enviado
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/InvitationCreatedResponse'
        '400':
          description: Error de validación (email inválido, roles inválidos)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Prohibido (sin permiso user:invite o tenant no permite invitaciones)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '409':
          description: Conflicto (email ya registrado en el tenant)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '429':
          description: Demasiadas solicitudes (límite de invitaciones excedido)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '500':
          description: Error interno (fallo en envío de email, compensación ejecutada)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

  /invitations/accept:
    post:
      operationId: acceptInvitation
      summary: Aceptar invitación (Usuario Invitado)
      description: |
        Valida el token de invitación y redirige al flujo de registro (UC-06).
        Endpoint público, no requiere autenticación.
      tags:
        - Invitations
      security: []  # Endpoint público
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/AcceptInvitationRequest'
      responses:
        '200':
          description: Invitación aceptada, registro completado
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/InvitationAcceptedResponse'
        '400':
          description: Token inválido, expirado o invitación ya usada
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

  /invitations/{invitation_id}/resend:
    post:
      operationId: resendInvitation
      summary: Reenviar invitación (Admin)
      description: |
        Genera un nuevo token y reinicia el TTL para una invitación expirada o pendiente.
        Requiere permiso `user:invite`.
      tags:
        - Invitations
      security:
        - BearerAuth: []
      parameters:
        - name: invitation_id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Invitación reenviada exitosamente
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/InvitationResentResponse'
        '404':
          description: Invitación no encontrada
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Prohibido (sin permiso user:invite)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

  /invitations/{invitation_id}:
    delete:
      operationId: cancelInvitation
      summary: Cancelar invitación (Admin)
      description: |
        Cancela una invitación pendiente, invalidando el token.
        Requiere permiso `user:invite`.
      tags:
        - Invitations
      security:
        - BearerAuth: []
      parameters:
        - name: invitation_id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Invitación cancelada exitosamente
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/InvitationCancelledResponse'
        '404':
          description: Invitación no encontrada
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Prohibido (sin permiso user:invite)
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
    CreateInvitationRequest:
      type: object
      required:
        - email
      properties:
        email:
          type: string
          format: email
          maxLength: 255
          description: "Email del usuario a invitar"
        roles:
          type: array
          items:
            type: string
            enum: [user, admin, developer, manager, viewer]
          maxItems: 10
          description: "Roles a asignar tras la aceptación (opcional)"
        message:
          type: string
          maxLength: 500
          description: "Mensaje personalizado para la invitación (opcional)"
        ttl_days:
          type: integer
          minimum: 1
          maximum: 30
          default: 7
          description: "Días de validez de la invitación (opcional)"

    InvitationCreatedResponse:
      type: object
      properties:
        invitation_id:
          type: string
          format: uuid
        email:
          type: string
          format: email
        status:
          type: string
          enum: [PENDING]
        expires_at:
          type: string
          format: date-time
        message:
          type: string
          example: "Invitation sent successfully"

    AcceptInvitationRequest:
      type: object
      required:
        - token
      properties:
        token:
          type: string
          format: uuid
          description: "Token de invitación original (no el hash)"

    InvitationAcceptedResponse:
      type: object
      properties:
        invitation_id:
          type: string
          format: uuid
        identity_id:
          type: string
          format: uuid
        status:
          type: string
          enum: [ACCEPTED]
        message:
          type: string
          example: "Bienvenido a la plataforma"

    InvitationResentResponse:
      type: object
      properties:
        invitation_id:
          type: string
          format: uuid
        status:
          type: string
          enum: [PENDING]
        new_expires_at:
          type: string
          format: date-time
        message:
          type: string
          example: "Invitation resent successfully"

    InvitationCancelledResponse:
      type: object
      properties:
        invitation_id:
          type: string
          format: uuid
        status:
          type: string
          enum: [CANCELLED]
        message:
          type: string
          example: "Invitation cancelled successfully"

    ErrorResponse:
      type: object
      properties:
        error:
          type: object
          properties:
            code:
              type: string
              enum:
                - INVALID_EMAIL
                - INVALID_ROLES
                - INSUFFICIENT_PERMISSIONS
                - INVITATIONS_DISABLED
                - EMAIL_ALREADY_REGISTERED
                - INVITATION_LIMIT_EXCEEDED
                - INVALID_TOKEN
                - INVITATION_EXPIRED
                - INVITATION_NOT_FOUND
                - EMAIL_SEND_FAILED
                - REGISTRATION_FAILED
            message:
              type: string
            details:
              type: object
```

### 2.2. Códigos de Error Específicos de UC-07

| Código HTTP | Código de Error | Descripción | Cuándo Ocurre |
| :--- | :--- | :--- | :--- |
| **400** | `INVALID_EMAIL` | Email no cumple RFC 5322 o es dominio desechable. | `Email` Value Object lanza excepción en constructor. |
| **400** | `INVALID_ROLES` | Roles especificados no existen o exceden el límite (10). | Validación en `InvitationRoles` Value Object. |
| **400** | `INVALID_TOKEN` | Token malformado o no coincide con ningún hash en DB. | `InvitationToken.Validate()` retorna false. |
| **400** | `INVITATION_EXPIRED` | El TTL de la invitación ha transcurrido. | `InvitationToken.IsExpired()` retorna true. |
| **403** | `INSUFFICIENT_PERMISSIONS` | El usuario no tiene el permiso `user:invite`. | API Gateway valida claims del JWT. |
| **403** | `INVITATIONS_DISABLED` | El tenant no permite invitaciones (`allow_invitations = false`). | `InvitationValidator.ValidateTenantConfig()`. |
| **404** | `INVITATION_NOT_FOUND` | La invitación no existe (para reenvío o cancelación). | `InvitationRepository.GetByIdAsync()` retorna null. |
| **409** | `EMAIL_ALREADY_REGISTERED` | El email ya está registrado en el tenant. | `InvitationValidator.ValidateEmailUniqueness()`. |
| **429** | `INVITATION_LIMIT_EXCEEDED` | El admin tiene 100 invitaciones pendientes. | `InvitationValidator.ValidateAdminLimit()`. |
| **500** | `EMAIL_SEND_FAILED` | Fallo en el envío de email. La invitación fue compensada (cancelada). | `EmailService` lanza excepción, `InvitationHandler` ejecuta compensación. |
| **500** | `REGISTRATION_FAILED` | Fallo en la integración con UC-06. La invitación permanece `PENDING`. | `RegistrationService` lanza excepción durante el registro. |

---

## 3. Catálogo de Métricas OpenTelemetry

### 3.1. Métricas de Negocio y Conversión

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.invitation.created_total` | Counter | invitations | Total de invitaciones creadas | `tenant_id`, `admin_id`, `has_roles` |
| `kagami.invitation.accepted_total` | Counter | acceptances | Total de invitaciones aceptadas | `tenant_id`, `time_to_accept_hours` |
| `kagami.invitation.expired_total` | Counter | expirations | Total de invitaciones expiradas | `tenant_id`, `days_pending` |
| `kagami.invitation.cancelled_total` | Counter | cancellations | Total de invitaciones canceladas | `tenant_id`, `cancelled_by` |
| `kagami.invitation.conversion_rate` | Gauge | ratio | Tasa de conversión (aceptadas / creadas) | `tenant_id`, `period_days` |
| `kagami.invitation.compensation_executed_total` | Counter | compensations | Total de compensaciones ejecutadas (email falló) | `tenant_id`, `reason` |

### 3.2. Métricas de Rendimiento y Servicios Externos

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.invitation.creation_latency` | Histogram | ms | Latencia end-to-end de creación de invitación | `tenant_id`, `success`, `email_sent` |
| `kagami.invitation.acceptance_latency` | Histogram | ms | Latencia end-to-end de aceptación de invitación | `tenant_id`, `success`, `uc06_integration` |
| `kagami.invitation.email_send_latency` | Histogram | ms | Latencia de envío de email de invitación | `tenant_id`, `success` |
| `kagami.invitation.email_send_errors_total` | Counter | errors | Total de fallos en envío de email | `tenant_id`, `error_type` (smtp_down, timeout, invalid_email) |
| `kagami.invitation.token_validation_latency` | Histogram | ms | Latencia de validación criptográfica de token | `tenant_id`, `success` |
| `kagami.invitation.uc06_integration_latency` | Histogram | ms | Latencia de integración con UC-06 (registro) | `tenant_id`, `success` |
| `kagami.invitation.rbac_assignment_latency` | Histogram | ms | Latencia de asignación de roles (MURA) | `tenant_id`, `roles_count` |

### 3.3. Métricas de Job de Limpieza

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.invitation.cleanup_job_duration` | Histogram | ms | Duración del job de limpieza de invitaciones expiradas | `tenant_id`, `invitations_expired` |
| `kagami.invitation.cleanup_job_errors_total` | Counter | errors | Total de fallos del job de limpieza | `tenant_id`, `error_type` |
| `kagami.invitation.cleanup_job_last_run` | Gauge | timestamp | Timestamp de la última ejecución exitosa del job | `tenant_id` |

### 3.4. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Email Service Down** | `email_send_errors_total > 5` en 5 min | **Crítica** | SMTP caído. Las invitaciones no se están enviando. Revisar logs de SMTP. Compensaciones se ejecutan automáticamente. |
| **Baja Tasa de Conversión** | `conversion_rate < 0.50` en 7 días | Media | Los usuarios no están aceptando invitaciones. Revisar entregabilidad de emails y UX del flujo de aceptación. |
| **Alta Tasa de Compensación** | `compensation_executed_total > 10` en 1h | Alta | Email service inestable. Investigar causa raíz. |
| **Job de Limpieza Fallido** | `cleanup_job_errors_total > 3` en 1h | Alta | Job de limpieza no está actualizando invitaciones expiradas. Revisar logs del job. |
| **Pico de Invitaciones** | `created_total > 100` en 10 min | Media | Posible onboarding masivo o abuso. Revisar admins con más invitaciones. |
| **Token Validation Slow** | `token_validation_latency_p95 > 100ms` | Media | Validación criptográfica lenta. Revisar CPU del servidor. |

---

## 4. Patrones de Resiliencia

### 4.1. Circuit Breaker para Email Service (Polly)

Dado que el Email Service es un punto crítico de falla, implementamos Circuit Breaker para prevenir cascadas de fallos y ejecutar compensación automáticamente.

**Implementación (Polly - C#):**

```csharp
public class ResilientEmailService : IEmailService
{
    private readonly IEmailService _inner;
    private readonly AsyncCircuitBreakerPolicy _circuitBreaker;
    private readonly ILogger<ResilientEmailService> _logger;

    public ResilientEmailService(IEmailService inner, ILogger<ResilientEmailService> logger)
    {
        _inner = inner;
        _logger = logger;
        
        _circuitBreaker = Policy
            .Handle<SmtpException>()
            .Or<TimeoutException>()
            .Or<HttpRequestException>()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 3,
                durationOfBreak: TimeSpan.FromMinutes(5),
                onBreak: (ex, duration) => 
                {
                    _logger.LogWarning("Email Service Circuit Breaker OPEN for {Duration}. Invitations will be compensated.", duration);
                    Metrics.EmailCircuitBreakerState.Set(1); // Open
                },
                onReset: () => 
                {
                    _logger.LogInformation("Email Service Circuit Breaker CLOSED. Invitations resumed.");
                    Metrics.EmailCircuitBreakerState.Set(0); // Closed
                },
                onHalfOpen: () => 
                {
                    _logger.LogInformation("Email Service Circuit Breaker HALF-OPEN. Testing connection.");
                    Metrics.EmailCircuitBreakerState.Set(2); // Half-Open
                }
            );
    }

    public async Task SendInvitationEmailAsync(Email email, InvitationToken token, string message, CancellationToken ct)
    {
        try
        {
            await _circuitBreaker.ExecuteAsync(() => 
                _inner.SendInvitationEmailAsync(email, token, message, ct));
        }
        catch (BrokenCircuitException)
        {
            // Circuit breaker está abierto, el email no se enviará
            // El InvitationHandler ejecutará la compensación automáticamente
            _logger.LogWarning("Circuit breaker open. Email will not be sent for {Email}. Compensation will be executed.", email.Value);
            throw new EmailServiceUnavailableException("Email service unavailable. Invitation will be compensated.", ex);
        }
    }

    public async Task SendAcceptanceConfirmationAsync(Email adminEmail, Email invitedEmail, CancellationToken ct)
    {
        try
        {
            await _circuitBreaker.ExecuteAsync(() => 
                _inner.SendAcceptanceConfirmationAsync(adminEmail, invitedEmail, ct));
        }
        catch (BrokenCircuitException)
        {
            // La confirmación no es crítica, solo logueamos el error
            _logger.LogWarning("Circuit breaker open. Acceptance confirmation not sent to {AdminEmail}", adminEmail.Value);
        }
    }
}
```

### 4.2. Retry Policy con Backoff Exponencial para Email Service

Para fallos transitorios (timeout, conexión intermitente), implementamos retry con backoff exponencial antes de ejecutar la compensación.

**Implementación (Polly - C#):**

```csharp
public class ResilientEmailService : IEmailService
{
    private readonly IEmailService _inner;
    private readonly IAsyncPolicy<HttpResponseMessage> _retryPolicy;
    private readonly ILogger<ResilientEmailService> _logger;

    public ResilientEmailService(IEmailService inner, ILogger<ResilientEmailService> logger)
    {
        _inner = inner;
        _logger = logger;
        
        _retryPolicy = Policy
            .Handle<SmtpException>()
            .Or<TimeoutException>()
            .WaitAndRetryAsync(
                retryCount: 3,
                sleepDurationProvider: retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)), // 2s, 4s, 8s
                onRetry: (ex, timeSpan, retryCount, context) =>
                {
                    _logger.LogWarning(ex, "Retry attempt {RetryCount} for email send after {Delay}s", 
                        retryCount, timeSpan.TotalSeconds);
                    Metrics.EmailRetryAttempts.Add(1, 
                        new KeyValuePair<string, object?>("retry_count", retryCount));
                }
            );
    }

    public async Task SendInvitationEmailAsync(Email email, InvitationToken token, string message, CancellationToken ct)
    {
        await _retryPolicy.ExecuteAsync(() => 
            _inner.SendInvitationEmailAsync(email, token, message, ct));
    }
}
```

### 4.3. Patrón de Compensación Transaccional

Si el Email Service falla (incluso después de retries), el `InvitationHandler` ejecuta la compensación automáticamente.

**Implementación (C#):**

```csharp
public class InvitationHandler : IInvitationHandler
{
    private readonly IEmailService _emailService;
    private readonly IInvitationRepository _repository;
    private readonly ILogger<InvitationHandler> _logger;

    public async Task<InvitationId> CreateInvitation(CreateInvitationCommand command, CancellationToken ct)
    {
        // 1. Validaciones de negocio
        await _validator.ValidateTenantConfig(command.TenantId, ct);
        await _validator.ValidateEmailUniquenessAsync(new Email(command.Email), command.TenantId, ct);
        await _validator.ValidateAdminLimitAsync(new AdminId(command.AdminId), command.TenantId, ct);

        // 2. Crear Aggregate Root
        var invitation = Invitation.Create(
            command.TenantId,
            new Email(command.Email),
            new AdminId(command.AdminId),
            new InvitationRoles(command.Roles),
            command.Message,
            command.TtlDays ?? 7
        );

        // 3. Persistir
        await _repository.SaveAsync(invitation, ct);

        // 4. Enviar email con patrón de compensación
        try
        {
            await _emailService.SendInvitationEmailAsync(
                invitation.Email,
                invitation.Token,
                invitation.Message,
                ct);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Fallo al enviar email de invitación. Ejecutando compensación para {InvitationId}", invitation.Id);
            
            // COMPENSACIÓN: Cancelar la invitación creada
            invitation.Cancel();
            await _repository.SaveAsync(invitation, ct);
            
            // Publicar evento de compensación
            await _eventPublisher.PublishAsync(new InvitationCompensatedEvent(
                invitation.Id, 
                invitation.TenantId, 
                "email_send_failed"), ct);
            
            Metrics.CompensationExecutedTotal.Add(1, 
                new KeyValuePair<string, object?>("tenant_id", command.TenantId.Value.ToString()),
                new KeyValuePair<string, object?>("reason", "email_send_failed"));
            
            throw new InvitationEmailFailedException("No se pudo enviar el email de invitación.", ex);
        }

        return invitation.Id;
    }
}
```

### 4.4. Timeout Estricto para Email Service

Para prevenir que el Email Service bloquee threads indefinidamente, configuramos timeouts estrictos.

**Configuración (C#):**

```csharp
// Configuración de timeouts para Email Service
services.AddHttpClient<IEmailService, SmtpEmailService>(client =>
{
    client.Timeout = TimeSpan.FromSeconds(10); // Timeout estricto para SMTP
});

// Configuración de Polly para timeout
services.AddPolicyHandler(Policy.TimeoutAsync<HttpResponseMessage>(TimeSpan.FromSeconds(10)));
```

---

## 5. Estrategia de Testing

### 5.1. Unit Tests (Dominio y Validaciones)

```csharp
[Test]
public void Invitation_Create_WithValidData_CreatesInPendingStatus()
{
    // Arrange
    var tenantId = new TenantId(Guid.NewGuid());
    var email = new Email("test@khcs.com");
    var adminId = new AdminId(Guid.NewGuid());
    var roles = new InvitationRoles(new List<RoleId> { new RoleId(Guid.NewGuid()) });

    // Act
    var invitation = Invitation.Create(tenantId, email, adminId, roles, "Welcome!", 7);

    // Assert
    invitation.Status.Should().Be(InvitationStatus.PENDING);
    invitation.Token.Should().NotBeNull();
    invitation.ExpiresAt.Should().BeCloseTo(DateTime.UtcNow.AddDays(7), TimeSpan.FromMinutes(1));
    invitation.GetDomainEvents().Should().HaveCount(1);
    invitation.GetDomainEvents().First().Should().BeOfType<InvitationCreatedEvent>();
}

[Test]
public void Invitation_Accept_WhenExpired_ThrowsInvitationExpiredException()
{
    // Arrange
    var invitation = Invitation.Create(
        new TenantId(Guid.NewGuid()),
        new Email("test@khcs.com"),
        new AdminId(Guid.NewGuid()),
        new InvitationRoles(new List<RoleId>()),
        "Welcome!",
        ttlDays: -1); // Expirado inmediatamente

    // Act
    var act = () => invitation.Accept(new IdentityId(Guid.NewGuid()));

    // Assert
    act.Should().Throw<InvitationExpiredException>();
}

[Test]
public void InvitationToken_Validate_WithCorrectToken_ReturnsTrue()
{
    // Arrange
    var token = InvitationToken.Create(7);
    // Nota: En testing real, necesitaríamos acceder al token original o mockear
    // Este test es conceptual para mostrar la intención
}

[Test]
public void Invitation_Resend_WhenExpired_TransitionsToPending()
{
    // Arrange
    var invitation = Invitation.Create(
        new TenantId(Guid.NewGuid()),
        new Email("test@khcs.com"),
        new AdminId(Guid.NewGuid()),
        new InvitationRoles(new List<RoleId>()),
        "Welcome!",
        ttlDays: -1); // Expirado

    invitation.Expire(); // Transición a EXPIRED

    // Act
    invitation.Resend(7);

    // Assert
    invitation.Status.Should().Be(InvitationStatus.PENDING);
    invitation.ExpiresAt.Should().BeCloseTo(DateTime.UtcNow.AddDays(7), TimeSpan.FromMinutes(1));
}
```

### 5.2. Integration Tests (Compensación y Servicios Externos)

```csharp
[Test]
public async Task InvitationHandler_WhenEmailServiceFails_CompensatesByCancellingInvitation()
{
    // Arrange
    var mockRepo = new Mock<IInvitationRepository>();
    var mockEmailService = new Mock<IEmailService>();
    mockEmailService.Setup(e => e.SendInvitationEmailAsync(It.IsAny<Email>(), It.IsAny<InvitationToken>(), It.IsAny<string>(), It.IsAny<CancellationToken>()))
                    .ThrowsAsync(new SmtpException("SMTP Down"));
    
    var handler = new InvitationHandler(
        Mock.Of<IInvitationValidator>(),
        mockRepo.Object,
        mockEmailService.Object,
        Mock.Of<IRegistrationService>(),
        Mock.Of<IRBACService>(),
        Mock.Of<ILogger<InvitationHandler>>());

    var command = new CreateInvitationCommand 
    { 
        TenantId = new TenantId(Guid.NewGuid()),
        Email = "test@khcs.com",
        AdminId = new AdminId(Guid.NewGuid()),
        Roles = new List<Guid>(),
        Message = "Welcome!"
    };

    // Act & Assert
    var act = () => handler.CreateInvitation(command, CancellationToken.None);
    await act.Should().ThrowAsync<InvitationEmailFailedException>();

    // Verificar que se llamó al método SaveAsync para persistir la cancelación (compensación)
    mockRepo.Verify(r => r.SaveAsync(It.Is<Invitation>(i => i.Status == InvitationStatus.CANCELLED), It.IsAny<CancellationToken>()), Times.Once);
}

[Test]
public async Task InvitationHandler_WhenRegistrationServiceFails_InvitationRemainsPending()
{
    // Arrange
    var mockRepo = new Mock<IInvitationRepository>();
    var mockRegService = new Mock<IRegistrationService>();
    mockRegService.Setup(r => r.CompleteRegistrationFromInvitationAsync(It.IsAny<Email>(), It.IsAny<InvitationRoles>(), It.IsAny<CancellationToken>()))
                  .ThrowsAsync(new RegistrationException("UC-06 failed"));
    
    var handler = new InvitationHandler(
        Mock.Of<IInvitationValidator>(),
        mockRepo.Object,
        Mock.Of<IEmailService>(),
        mockRegService.Object,
        Mock.Of<IRBACService>(),
        Mock.Of<ILogger<InvitationHandler>>());

    // Simular invitación existente
    var invitation = Invitation.Create(
        new TenantId(Guid.NewGuid()),
        new Email("test@khcs.com"),
        new AdminId(Guid.NewGuid()),
        new InvitationRoles(new List<RoleId>()),
        "Welcome!",
        7);
    
    mockRepo.Setup(r => r.GetByTokenHashAsync(It.IsAny<string>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(invitation);

    // Act & Assert
    var act = () => handler.AcceptInvitation("valid-token", CancellationToken.None);
    await act.Should().ThrowAsync<InvitationAcceptanceFailedException>();

    // Verificar que la invitación permanece en estado PENDING
    invitation.Status.Should().Be(InvitationStatus.PENDING);
}
```

### 5.3. Load Testing (Picos de Invitaciones)

**Objetivo:** Validar que el sistema maneja 50 invitaciones/minuto sin degradar el rendimiento.

**Script k6 (Extracto):**

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    invitation_spike: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 10 },  // Ramp up
        { duration: '2m', target: 10 },  // Mantener carga
        { duration: '1m', target: 0 },   // Ramp down
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.05'],
  },
};

export default function () {
  const payload = JSON.stringify({
    email: `user_${__VU}_${__ITER}@khcs.com`,
    roles: ['user', 'developer'],
    message: 'Welcome to the team!',
    ttl_days: 7
  });

  const res = http.post(`${__ENV.BASE_URL}/v1/invitations`, payload, {
    headers: { 
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${__ENV.ADMIN_TOKEN}`
    },
  });

  check(res, {
    'invitation created (201)': (r) => r.status === 201,
    'has invitation_id': (r) => r.json('invitation_id') !== undefined,
    'has expires_at': (r) => r.json('expires_at') !== undefined,
  });

  sleep(1); // 1 invitación por segundo por VU
}
```

---

## 6. Deployment y Operaciones

### 6.1. Feature Flags para Rollout Gradual

| Feature Flag | Descripción | Valor por Defecto |
| :--- | :--- | :--- |
| `kagami.invitation.enabled` | Activa/desactiva la funcionalidad de invitaciones globalmente. | `true` |
| `kagami.invitation.email_compensation_enabled` | Activa el patrón de compensación si el email falla. | `true` |
| `kagami.invitation.token_security_enabled` | Activa la validación criptográfica de tokens (SHA256 + SecureCompare). | `true` |
| `kagami.invitation.max_ttl_days` | Límite máximo de días de validez de invitaciones. | `30` |
| `kagami.invitation.max_roles_per_invitation` | Límite máximo de roles por invitación. | `10` |
| `kagami.invitation.cleanup_job_enabled` | Activa el job de limpieza de invitaciones expiradas. | `true` |

### 6.2. Configuración de Email Service

```yaml
# Configuración de SMTP para invitaciones
email:
  smtp:
    host: "smtp.sendgrid.net"
    port: 587
    username: "${SMTP_USERNAME}"
    password: "${SMTP_PASSWORD}"
    from_email: "noreply@shugo.io"
    from_name: "SHUGO IAM"
    timeout_seconds: 10
    retry_attempts: 3
    retry_delay_seconds: 2  # Backoff exponencial: 2s, 4s, 8s

  templates:
    invitation:
      subject: "You've been invited to join {{tenant_name}}"
      template_id: "d-1234567890abcdef"  # SendGrid Template ID
    acceptance_confirmation:
      subject: "User {{invited_email}} accepted your invitation"
      template_id: "d-0987654321fedcba"
```

### 6.3. Configuración del Job de Limpieza

```yaml
# Configuración del job de limpieza de invitaciones expiradas
cleanup_job:
  enabled: true
  interval_hours: 1
  batch_size: 1000
  max_consecutive_failures: 3
  backoff_exponential_base: 2  # 1min, 2min, 4min, 8min
```

### 6.4. Escalado Horizontal de Invitation Handlers

```yaml
# Kubernetes HPA para Invitation Handlers
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kagami-invitation-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kagami-invitation
  minReplicas: 2
  maxReplicas: 10
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
          name: kagami_invitation_creation_latency
        target:
          type: AverageValue
          averageValue: "300" # Escalar si latencia p95 > 300ms
    - type: External
      external:
        metric:
          name: kagami_invitation_created_total
        target:
          type: AverageValue
          averageValue: "50" # Escalar si >50 invitaciones/min por pod
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

**Escenario: Email Service Caído**

1. **Paso 1:** Verificar alerta `email_send_errors_total > 5` en 5 min.
2. **Paso 2:** Revisar logs de `ResilientEmailService` para confirmar Circuit Breaker OPEN.
3. **Paso 3:** Las invitaciones se están compensando automáticamente (status = CANCELLED). Los admins reciben HTTP 500.
4. **Paso 4:** Revisar conectividad con SMTP (SendGrid, Exchange, etc.).
5. **Paso 5:** Una vez recuperado, el Circuit Breaker transiciona a HALF-OPEN y luego CLOSED automáticamente.
6. **Paso 6:** Monitorear métricas `email_send_latency` y `compensation_executed_total` para confirmar recuperación.

**Escenario: Job de Limpieza Fallido**

1. **Paso 1:** Verificar alerta `cleanup_job_errors_total > 3` en 1h.
2. **Paso 2:** Revisar logs del job de limpieza para identificar causa raíz.
3. **Paso 3:** Si el job está atascado, reiniciar el pod del job.
4. **Paso 4:** Verificar que las invitaciones expiradas se están actualizando a estado `EXPIRED`.
5. **Paso 5:** Monitorear métrica `cleanup_job_last_run` para confirmar ejecución exitosa.

**Escenario: Baja Tasa de Conversión**

1. **Paso 1:** Verificar métrica `conversion_rate < 0.50` en 7 días.
2. **Paso 2:** Revisar métricas de entregabilidad de email (bounce rate, spam complaints).
3. **Paso 3:** Revisar UX del flujo de aceptación (¿el link funciona? ¿el registro es claro?).
4. **Paso 4:** Considerar reenviar invitaciones pendientes con mensaje personalizado.
5. **Paso 5:** Si la tasa persiste baja, investigar con el equipo de producto.

---

## 7. Compliance y Auditoría

### 7.1. GDPR - Art. 6 (Lawfulness of Processing)

*   **Requisito:** El procesamiento de datos personales debe tener una base legal.
*   **Implementación UC-07:** La invitación se basa en el consentimiento explícito del administrador (quien conoce al usuario) y el consentimiento posterior del usuario invitado (al aceptar la invitación y completar el registro).

### 7.2. GDPR - Art. 7 (Conditions for Consent)

*   **Requisito:** El consentimiento debe ser libre, específico, informado e inequívoco.
*   **Implementación UC-07:** El usuario invitado acepta explícitamente los términos de servicio y política de privacidad durante el flujo de registro (UC-06). El registro de consentimiento es inmutable y auditable.

### 7.3. GDPR - Art. 17 (Right to Erasure)

*   **Requisito:** El usuario tiene derecho a obtener la eliminación de sus datos personales.
*   **Implementación UC-07:** Si el usuario invitado no acepta la invitación, puede solicitar la eliminación de sus datos (email) del sistema. Las invitaciones no aceptadas expiran automáticamente tras 7 días.

### 7.4. SOC 2 - CC6.1 (Logical Access Controls)

*   **Requisito:** La organización debe restringir el acceso lógico a los datos según el rol del usuario.
*   **Implementación UC-07:** Solo administradores con permiso `user:invite` pueden crear invitaciones. Los tokens de invitación son de un solo uso y tienen TTL estricto.

### 7.5. SOC 2 - CC7.4 (System Operations - Data Disposal)

*   **Requisito:** Los datos deben eliminarse de forma segura al final de su ciclo de vida.
*   **Implementación UC-07:** Las invitaciones no aceptadas expiran automáticamente y se marcan como `EXPIRED`. Los tokens se invalidan inmediatamente tras la aceptación o cancelación.

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los requisitos de compensación transaccional, seguridad de tokens, y integración con UC-06 que estos artefactos implementan. |
| `08_Final_Class_Design.md` | Las clases `InvitationHandler`, `ResilientEmailService`, y `InvitationCleanupJob` se implementan según los contratos y métricas definidos aquí. |
| `10_UC_Specific_Data_Model.md` | Define la estructura física de las tablas `invitations`, `consent_records`, y índices para tokens. |
| `11_UC_Specific_Security_Threats.md` | Las mitigaciones de "Token Hijacking", "Enumeration Attacks", y "Invitation Spam" se validan mediante los patrones definidos aquí. |
| `12_UC_Specific_Performance_Benchmarks.md` | Los SLAs de latencia (<500ms p95 para creación) y throughput (50 invitaciones/min) se validan mediante las métricas y load testing definidos aquí. |
| `SHUGO_Security_Architecture.md` (Transversal) | Este documento aplica los estándares globales de control de acceso y privacidad al contexto específico de UC-07. |

---

## 9. Cierre del Paquete de Diseño Funcional (UC-07)

Con este documento, completamos los **9 archivos del Paquete de Diseño Profundo Funcional (Tier 1)** para el UC-07:

1. ✅ `01_Requirements_Specification.md` - Especificación completa con 10 escenarios BDD.
2. ✅ `02_Use_Case_Diagram.md` - Diagrama de caso de uso UML con actores Admin y Usuario Invitado.
3. ✅ `03_Activity_Diagram.md` - Diagrama de actividades con flujo dual asíncrono y compensación.
4. ✅ `04_Initial_Domain_Model.md` - Modelo de dominio conceptual (Aggregate Root, Value Objects).
5. ✅ `05_State_Machine_Diagram.md` - 3 máquinas de estados (Invitación, Token, Job de Limpieza).
6. ✅ `06_Robustness_Diagram.md` - Diagrama de robustez ICONIX.
7. ✅ `07_Sequence_Diagram.md` - Diagrama de secuencia con flujos alternativos y compensación.
8. ✅ `08_Final_Class_Design.md` - Diseño de clases con firmas y patrones.
9. ✅ `09_Modern_Artifacts.md` - Contratos OpenAPI, métricas, resiliencia, compliance.

### 9.1. Resumen de lo Logrado

Hemos diseñado el endpoint de invitación más robusto, seguro y compliant del mercado IAM:
- **Flujo Dual Asíncrono:** Separación clara entre creación (Admin) y aceptación (Usuario Invitado) con validaciones independientes.
- **Seguridad Criptográfica:** Tokens UUID v4 almacenados como hashes SHA256, con comparación en tiempo constante para prevenir Timing Attacks.
- **Resiliencia Transaccional:** Patrón de compensación garantiza que no queden invitaciones huérfanas si el email falla.
- **GDPR Compliance Nativo:** Consentimiento explícito con registro inmutable (timestamp, IP, User Agent) cumple con Art. 7(1).
- **Observabilidad Completa:** Métricas de conversión, compensación, y job de limpieza permiten monitoreo proactivo.

### 9.2. Próximos Pasos (Documentos Técnicos Específicos)

Según la **Opción A+** acordada, cada UC tiene **12 documentos** (9 funcionales + 3 técnicos específicos). Para el UC-07, aún faltan:

1. **`10_UC_Specific_Data_Model.md`** - Modelo de datos físico específico (tablas `invitations`, índices para tokens, políticas de retención).
2. **`11_UC_Specific_Security_Threats.md`** - Análisis de amenazas específico (Token Hijacking, Enumeration Attacks, Invitation Spam).
3. **`12_UC_Specific_Performance_Benchmarks.md`** - Benchmarks y SLAs específicos para latencia de creación, throughput de invitaciones, y tasa de conversión.

---
*Fin del Documento 09_Modern_Artifacts para UC-07*