# UC-09: Verificación de Contactos - Artefactos Modernos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-09 - Verificación de Contactos (Out-of-Band Verification) |
| **Documento:** | 09_Modern_Artifacts.md |
| **Metodología:** | API-First + Observability + Resilience Patterns + DevSecOps + Security by Design |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 10/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 10/09/2026 | Gastón Ruiz | **Creación inicial.** Definición de los artefactos modernos para UC-09: contratos OpenAPI 3.1 con endpoints de iniciación y completación de verificación, catálogo de métricas OpenTelemetry (enfocado en tasa de éxito de verificación, latencia de gateways, intentos de fuerza bruta), patrones de resiliencia (Circuit Breaker para SMS/Email Gateways, Retry policies con backoff exponencial), estrategias de testing para escenarios de seguridad (Anti-Brute Force, Race Conditions), y consideraciones de deployment para operaciones de verificación transversales. | Cierra el paquete de diseño Tier 1 funcional del UC-09, proporcionando todos los artefactos necesarios para la implementación, operación, monitoreo y cumplimiento normativo (NIST SP 800-63A) del motor de verificación Out-of-Band del motor KAGAMI. |

---

## 1. Introducción y Propósito

Este documento define los **Artefactos Modernos** necesarios para operar el UC-09 (Verificación de Contactos) en un entorno de producción enterprise. A diferencia de los UCs transaccionales estándar, el UC-09 introduce desafíos únicos debido a su naturaleza **transversal y de alta seguridad**:

1. **Servicio de Dominio Transversal:** Consumido por UC-06, UC-08, UC-15, y UC-18, requiriendo APIs claras y contratos estables.
2. **Dependencia Crítica de Gateways Externos:** SMS (Twilio) y Email (SendGrid) son puntos de falla que requieren Circuit Breakers y patrones de resiliencia.
3. **Seguridad Criptográfica:** Generación de OTPs, almacenamiento seguro, y validación con SecureCompare requieren monitoreo específico.
4. **Prevención de Abuso Financiero:** Rate Limiting estricto para prevenir SMS/Email Bombing que podría generar facturas masivas.
5. **Atomicidad en Redis:** Las operaciones de incremento de intentos deben ser atómicas para prevenir Race Conditions.
6. **TTL Automático:** Redis maneja la expiración de tokens, eliminando la necesidad de jobs de limpieza.

---

## 2. Contratos OpenAPI Completos

### 2.1. Especificación OpenAPI 3.1

```yaml
openapi: 3.1.0
info:
  title: SHUGO KAGAMI - Contact Verification API
  description: |
    API de verificación de contactos Out-of-Band (OOB) del motor KAGAMI.
    
    ## Características Clave
    - Servicio transversal consumido por UC-06, UC-08, UC-15, UC-18.
    - Generación de OTPs criptográficos (6 dígitos) y Magic Links (UUIDv4).
    - Rate Limiting estricto para prevenir SMS/Email Bombing.
    - Anti-Brute Force con contador atómico de intentos en Redis.
    - Validación con SecureCompare para prevenir Timing Attacks.
    - TTL automático gestionado por Redis (10 min OTP, 24h Magic Link).
    
    ## Seguridad
    - Endpoints autenticados (JWT) para flujos de usuario logueado.
    - Prevención de enumeración en flujos públicos (mensajes genéricos).
    - Hashing SHA256 de contactos para protección de PII.
    - Single-Use Token Invalidation tras validación exitosa.
  version: 1.0.0

servers:
  - url: https://api.shugo.io/v1
    description: Producción

paths:
  /verification/initiate:
    post:
      operationId: initiateVerification
      summary: Iniciar verificación de contacto
      description: |
        Genera un OTP o Magic Link y lo envía al contacto especificado.
        Protegido por Rate Limiting para prevenir abuso.
      tags:
        - Verification
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/InitiateVerificationRequest'
            examples:
              smsVerification:
                summary: Verificación por SMS (OTP)
                value:
                  contact_type: "PHONE"
                  contact_value: "+5491155556666"
                  channel: "SMS"
              emailVerification:
                summary: Verificación por Email (Magic Link)
                value:
                  contact_type: "EMAIL"
                  contact_value: "user@khcs.com"
                  channel: "EMAIL"
      responses:
        '202':
          description: Verificación iniciada, código/link enviado
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/VerificationInitiatedResponse'
        '400':
          description: Error de validación (contacto inválido, formato incorrecto)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '429':
          description: Rate limit excedido (Anti-Bombing)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '500':
          description: Error interno (fallo en gateway externo)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

  /verification/complete:
    post:
      operationId: completeVerification
      summary: Completar verificación de contacto
      description: |
        Valida el OTP o Magic Link proporcionado.
        Protegido por Anti-Brute Force (máx. 5 intentos).
      tags:
        - Verification
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CompleteVerificationRequest'
            examples:
              otpValidation:
                summary: Validación de OTP
                value:
                  contact_hash: "sha256_hash_of_contact"
                  provided_challenge: "123456"
              magicLinkValidation:
                summary: Validación de Magic Link
                value:
                  contact_hash: "sha256_hash_of_contact"
                  provided_challenge: "uuid-v4-token"
      responses:
        '200':
          description: Verificación completada exitosamente
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/VerificationCompletedResponse'
        '400':
          description: Código/link inválido o expirado
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '429':
          description: Máximo de intentos excedido (Anti-Brute Force)
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
    InitiateVerificationRequest:
      type: object
      required:
        - contact_type
        - contact_value
        - channel
      properties:
        contact_type:
          type: string
          enum: [EMAIL, PHONE]
          description: "Tipo de contacto a verificar"
        contact_value:
          type: string
          description: "Valor del contacto (email o teléfono en formato E.164)"
        channel:
          type: string
          enum: [EMAIL, SMS]
          description: "Canal de envío del desafío"

    VerificationInitiatedResponse:
      type: object
      properties:
        message:
          type: string
          example: "Verification code sent."
        expires_in_seconds:
          type: integer
          example: 600
          description: "TTL del OTP/Link en segundos"
        contact_hash:
          type: string
          description: "Hash SHA256 del contacto (para validación posterior)"

    CompleteVerificationRequest:
      type: object
      required:
        - contact_hash
        - provided_challenge
      properties:
        contact_hash:
          type: string
          description: "Hash SHA256 del contacto"
        provided_challenge:
          type: string
          description: "OTP (6 dígitos) o Magic Link token proporcionado"

    VerificationCompletedResponse:
      type: object
      properties:
        contact_type:
          type: string
          enum: [EMAIL, PHONE]
        contact_hash:
          type: string
        is_verified:
          type: boolean
          example: true
        verified_at:
          type: string
          format: date-time
        message:
          type: string
          example: "Contact verified successfully"

    ErrorResponse:
      type: object
      properties:
        error:
          type: object
          properties:
            code:
              type: string
              enum:
                - INVALID_CONTACT
                - RATE_LIMIT_EXCEEDED
                - BRUTE_FORCE_DETECTED
                - VERIFICATION_EXPIRED
                - VERIFICATION_NOT_FOUND
                - INVALID_CHALLENGE
                - GATEWAY_FAILED
            message:
              type: string
            details:
              type: object
```

### 2.2. Códigos de Error Específicos de UC-09

| Código HTTP | Código de Error | Descripción | Cuándo Ocurre |
| :--- | :--- | :--- | :--- |
| **400** | `INVALID_CONTACT` | Formato de email/teléfono inválido. | `ContactInfo.CreateEmail()` o `ContactInfo.CreatePhone()` falla. |
| **400** | `VERIFICATION_EXPIRED` | El OTP/Link ha expirado (TTL superado). | Redis TTL eliminó la clave, o `DateTime.UtcNow > ExpiresAt`. |
| **400** | `VERIFICATION_NOT_FOUND` | El OTP/Link no existe o ya fue invalidado. | `Redis.StringGetAsync()` retorna null. |
| **400** | `INVALID_CHALLENGE` | El código/link proporcionado es incorrecto. | `SecureCompare.FixedTimeEquals()` retorna false. |
| **429** | `RATE_LIMIT_EXCEEDED` | Rate limit de iniciación excedido (Anti-Bombing). | `RateLimiter.IsAllowedAsync()` retorna false. |
| **429** | `BRUTE_FORCE_DETECTED` | Máximo de intentos de validación excedido (Anti-Brute Force). | `RateLimiter.IncrementAttemptsAsync()` retorna count > 5. |
| **500** | `GATEWAY_FAILED` | Fallo en envío a SMS/Email Gateway. | Twilio/SendGrid retorna error o timeout. |

---

## 3. Catálogo de Métricas OpenTelemetry

### 3.1. Métricas de Negocio y Conversión

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.verification.initiated_total` | Counter | initiations | Total de verificaciones iniciadas | `tenant_id`, `channel`, `contact_type` |
| `kagami.verification.completed_total` | Counter | completions | Total de verificaciones completadas exitosamente | `tenant_id`, `channel`, `contact_type` |
| `kagami.verification.failed_total` | Counter | failures | Total de verificaciones fallidas | `tenant_id`, `channel`, `failure_reason` |
| `kagami.verification.success_rate` | Gauge | ratio | Tasa de éxito de verificación (completadas / iniciadas) | `tenant_id`, `channel` |
| `kagami.verification.expired_total` | Counter | expirations | Total de OTPs/Links que expiraron sin validación | `tenant_id`, `channel` |

### 3.2. Métricas de Seguridad y Prevención de Abuso

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.verification.rate_limit_exceeded_total` | Counter | blocks | Total de requests bloqueadas por Rate Limiting | `tenant_id`, `channel`, `contact_hash` |
| `kagami.verification.brute_force_detected_total` | Counter | detections | Total de intentos de fuerza bruta detectados | `tenant_id`, `channel`, `contact_hash` |
| `kagami.verification.invalid_attempts_total` | Counter | attempts | Total de intentos de validación inválidos | `tenant_id`, `channel`, `contact_hash` |
| `kagami.verification.anti_enumeration_triggered_total` | Counter | triggers | Total de veces que se activó la prevención de enumeración | `tenant_id` |

### 3.3. Métricas de Rendimiento y Servicios Externos

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.verification.initiation_latency` | Histogram | ms | Latencia end-to-end de iniciación de verificación | `tenant_id`, `channel`, `success` |
| `kagami.verification.completion_latency` | Histogram | ms | Latencia end-to-end de completación de verificación | `tenant_id`, `channel`, `success` |
| `kagami.verification.otp_generation_latency` | Histogram | ms | Latencia de generación criptográfica de OTP | `tenant_id` |
| `kagami.verification.secure_compare_latency` | Histogram | ms | Latencia de validación con SecureCompare | `tenant_id`, `success` |
| `kagami.verification.redis_operation_latency` | Histogram | ms | Latencia de operaciones atómicas en Redis | `tenant_id`, `operation_type` |
| `kagami.verification.sms_gateway_latency` | Histogram | ms | Latencia de envío a SMS Gateway (Twilio) | `tenant_id`, `success` |
| `kagami.verification.email_gateway_latency` | Histogram | ms | Latencia de envío a Email Gateway (SendGrid) | `tenant_id`, `success` |
| `kagami.verification.gateway_errors_total` | Counter | errors | Total de errores en gateways externos | `tenant_id`, `gateway`, `error_type` |

### 3.4. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **SMS Gateway Down** | `sms_gateway_errors_total > 10` en 5 min | **Crítica** | Twilio caído. Las verificaciones por SMS están fallando. Revisar estado de Twilio. |
| **Email Gateway Down** | `email_gateway_errors_total > 10` en 5 min | **Crítica** | SendGrid caído. Las verificaciones por Email están fallando. Revisar estado de SendGrid. |
| **Alta Tasa de Brute Force** | `brute_force_detected_total > 50` en 10 min | **Crítica** | Posible ataque de fuerza bruta masivo. Revisar IPs y contactos atacados. |
| **Rate Limiting Excesivo** | `rate_limit_exceeded_total > 100` en 10 min | Alta | Posible ataque de SMS/Email Bombing. Revisar IPs ofensivas. |
| **Baja Tasa de Éxito** | `success_rate < 0.50` en 1h | Media | Los usuarios no están completando verificaciones. Revisar UX o entregabilidad. |
| **Redis Latency High** | `redis_operation_latency_p95 > 50ms` | Media | Redis lento. Revisar conectividad y carga. |
| **OTP Generation Slow** | `otp_generation_latency_p95 > 10ms` | Media | Generación criptográfica lenta. Revisar CPU del servidor. |

---

## 4. Patrones de Resiliencia

### 4.1. Circuit Breaker para SMS Gateway (Twilio)

Dado que Twilio es un servicio externo crítico para verificaciones por SMS, implementamos Circuit Breaker para prevenir cascadas de fallos.

**Implementación (Polly - C#):**

```csharp
public class ResilientSmsGateway : ISmsGateway
{
    private readonly ISmsGateway _inner;
    private readonly AsyncCircuitBreakerPolicy _circuitBreaker;
    private readonly ILogger<ResilientSmsGateway> _logger;

    public ResilientSmsGateway(ISmsGateway inner, ILogger<ResilientSmsGateway> logger)
    {
        _inner = inner;
        _logger = logger;
        
        _circuitBreaker = Policy
            .Handle<TwilioException>()
            .Or<TimeoutException>()
            .Or<HttpRequestException>()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 3,
                durationOfBreak: TimeSpan.FromMinutes(5),
                onBreak: (ex, duration) => 
                {
                    _logger.LogWarning("SMS Gateway Circuit Breaker OPEN for {Duration}. SMS verifications will fail.", duration);
                    Metrics.SmsGatewayCircuitBreakerState.Set(1); // Open
                },
                onReset: () => 
                {
                    _logger.LogInformation("SMS Gateway Circuit Breaker CLOSED. SMS verifications resumed.");
                    Metrics.SmsGatewayCircuitBreakerState.Set(0); // Closed
                },
                onHalfOpen: () => 
                {
                    _logger.LogInformation("SMS Gateway Circuit Breaker HALF-OPEN. Testing connection.");
                    Metrics.SmsGatewayCircuitBreakerState.Set(2); // Half-Open
                }
            );
    }

    public async Task SendAsync(string phoneNumber, string message, CancellationToken ct)
    {
        try
        {
            await _circuitBreaker.ExecuteAsync(() => 
                _inner.SendAsync(phoneNumber, message, ct));
        }
        catch (BrokenCircuitException)
        {
            _logger.LogWarning("Circuit breaker open. SMS cannot be sent to {PhoneNumber}.", phoneNumber);
            throw new GatewayUnavailableException("SMS Gateway unavailable. Please try again later.", ex);
        }
    }
}
```

### 4.2. Circuit Breaker para Email Gateway (SendGrid)

Similar al SMS Gateway, implementamos Circuit Breaker para SendGrid.

**Implementación (Polly - C#):**

```csharp
public class ResilientEmailGateway : IEmailGateway
{
    private readonly IEmailGateway _inner;
    private readonly AsyncCircuitBreakerPolicy _circuitBreaker;
    private readonly ILogger<ResilientEmailGateway> _logger;

    public ResilientEmailGateway(IEmailGateway inner, ILogger<ResilientEmailGateway> logger)
    {
        _inner = inner;
        _logger = logger;
        
        _circuitBreaker = Policy
            .Handle<SendGridException>()
            .Or<TimeoutException>()
            .Or<HttpRequestException>()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 3,
                durationOfBreak: TimeSpan.FromMinutes(5),
                onBreak: (ex, duration) => 
                {
                    _logger.LogWarning("Email Gateway Circuit Breaker OPEN for {Duration}. Email verifications will fail.", duration);
                    Metrics.EmailGatewayCircuitBreakerState.Set(1); // Open
                },
                onReset: () => 
                {
                    _logger.LogInformation("Email Gateway Circuit Breaker CLOSED. Email verifications resumed.");
                    Metrics.EmailGatewayCircuitBreakerState.Set(0); // Closed
                }
            );
    }

    public async Task SendAsync(string emailAddress, string subject, string body, CancellationToken ct)
    {
        try
        {
            await _circuitBreaker.ExecuteAsync(() => 
                _inner.SendAsync(emailAddress, subject, body, ct));
        }
        catch (BrokenCircuitException)
        {
            _logger.LogWarning("Circuit breaker open. Email cannot be sent to {EmailAddress}.", emailAddress);
            throw new GatewayUnavailableException("Email Gateway unavailable. Please try again later.", ex);
        }
    }
}
```

### 4.3. Retry Policy con Backoff Exponencial para Gateways

Para fallos transitorios en Twilio/SendGrid, implementamos retry con backoff exponencial.

**Implementación (Polly - C#):**

```csharp
public class ResilientSmsGateway : ISmsGateway
{
    private readonly ISmsGateway _inner;
    private readonly IAsyncPolicy _retryPolicy;
    private readonly ILogger<ResilientSmsGateway> _logger;

    public ResilientSmsGateway(ISmsGateway inner, ILogger<ResilientSmsGateway> logger)
    {
        _inner = inner;
        _logger = logger;
        
        _retryPolicy = Policy
            .Handle<TwilioException>()
            .Or<TimeoutException>()
            .WaitAndRetryAsync(
                retryCount: 3,
                sleepDurationProvider: retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)), // 2s, 4s, 8s
                onRetry: (ex, timeSpan, retryCount, context) =>
                {
                    _logger.LogWarning(ex, "Retry attempt {RetryCount} for SMS send after {Delay}s", 
                        retryCount, timeSpan.TotalSeconds);
                    Metrics.SmsGatewayRetryAttempts.Add(1, 
                        new KeyValuePair<string, object?>("retry_count", retryCount));
                }
            );
    }

    public async Task SendAsync(string phoneNumber, string message, CancellationToken ct)
    {
        await _retryPolicy.ExecuteAsync(() => 
            _inner.SendAsync(phoneNumber, message, ct));
    }
}
```

### 4.4. Timeout Estricto para Gateways

Para prevenir que los gateways bloqueen threads indefinidamente, configuramos timeouts estrictos.

**Configuración (C#):**

```csharp
// Configuración de timeouts para gateways externos
services.AddHttpClient<ISmsGateway, TwilioSmsGateway>(client =>
{
    client.Timeout = TimeSpan.FromSeconds(5); // Timeout estricto para SMS
});

services.AddHttpClient<IEmailGateway, SendGridEmailGateway>(client =>
{
    client.Timeout = TimeSpan.FromSeconds(10); // Timeout estricto para Email
});

// Configuración de Polly para timeout
services.AddPolicyHandler(Policy.TimeoutAsync<HttpResponseMessage>(TimeSpan.FromSeconds(5)));
```

### 4.5. Fire-and-Forget para Envío Asíncrono

El envío de OTPs/Links a gateways externos se ejecuta de forma asíncrona para no bloquear la respuesta HTTP.

**Implementación (C#):**

```csharp
public async Task<VerificationRequestId> InitiateVerification(
    InitiateVerificationCommand command,
    CancellationToken ct)
{
    // ... validaciones, generación, persistencia ...

    // Envío asíncrono (Fire-and-Forget)
    var challengeValue = challenge.GetValueForDelivery();
    _ = Task.Run(async () =>
    {
        try
        {
            await _gatewayAdapter.SendAsync(command.ContactValue, challengeValue, CancellationToken.None);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to send verification to {ContactValue}", command.ContactValue);
            // Outbox Pattern o Background Worker puede reintentar
        }
    }, ct);

    return verificationRequest.Id;
}
```

---

## 5. Estrategia de Testing

### 5.1. Unit Tests (Dominio y Validaciones)

```csharp
[Test]
public void OtpCode_Generate_ReturnsSixDigitCode()
{
    // Act
    var otp = OtpCode.Generate();

    // Assert
    otp.Code.Should().MatchRegex(@"^\d{6}$");
}

[Test]
public void OtpCode_Validate_WithCorrectCode_ReturnsTrue()
{
    // Arrange
    var otp = OtpCode.Generate();

    // Act
    var isValid = otp.Validate(otp.Code);

    // Assert
    isValid.Should().BeTrue();
}

[Test]
public void OtpCode_Validate_WithIncorrectCode_ReturnsFalse()
{
    // Arrange
    var otp = OtpCode.Generate();

    // Act
    var isValid = otp.Validate("000000");

    // Assert
    isValid.Should().BeFalse();
}

[Test]
public void MagicLinkToken_Generate_ReturnsUUIDv4Format()
{
    // Act
    var token = MagicLinkToken.Generate();

    // Assert
    token.Token.Should().HaveLength(32); // UUID v4 sin guiones
    token.TokenHash.Should().NotBeNullOrEmpty();
}

[Test]
public void SecureCompare_FixedTimeEquals_WithSameValues_ReturnsTrue()
{
    // Arrange
    var secureCompare = new SecureCompare();
    var a = Encoding.UTF8.GetBytes("123456");
    var b = Encoding.UTF8.GetBytes("123456");

    // Act
    var result = secureCompare.FixedTimeEquals(a, b);

    // Assert
    result.Should().BeTrue();
}

[Test]
public void VerificationRequest_Initiate_WithValidData_CreatesInPendingStatus()
{
    // Arrange
    var identityId = new IdentityId(Guid.NewGuid());
    var tenantId = new TenantId(Guid.NewGuid());
    var contactInfo = ContactInfo.CreateEmail("test@khcs.com");
    var channel = VerificationChannel.EMAIL;
    var ttl = TimeSpan.FromHours(24);

    // Act
    var request = VerificationRequest.Initiate(identityId, tenantId, contactInfo, channel, ttl);

    // Assert
    request.Status.Should().Be(VerificationStatus.PENDING);
    request.Challenge.Should().NotBeNull();
    request.Challenge.Type.Should().Be(ChallengeType.MAGIC_LINK);
    request.Attempts.Should().Be(0);
    request.ExpiresAt.Should().BeCloseTo(DateTime.UtcNow.Add(ttl), TimeSpan.FromMinutes(1));
}

[Test]
public void VerificationRequest_Validate_WithCorrectOtp_TransitionsToValidated()
{
    // Arrange
    var request = VerificationRequest.Initiate(
        new IdentityId(Guid.NewGuid()),
        new TenantId(Guid.NewGuid()),
        ContactInfo.CreatePhone("+1234567890"),
        VerificationChannel.SMS,
        TimeSpan.FromMinutes(10));

    var otpCode = request.Challenge.OtpCode!.Code;

    // Act
    var isValid = request.Validate(otpCode);

    // Assert
    isValid.Should().BeTrue();
    request.Status.Should().Be(VerificationStatus.VALIDATED);
    request.ValidatedAt.Should().NotBeNull();
}

[Test]
public void VerificationRequest_Validate_WhenExpired_ThrowsVerificationExpiredException()
{
    // Arrange
    var request = VerificationRequest.Initiate(
        new IdentityId(Guid.NewGuid()),
        new TenantId(Guid.NewGuid()),
        ContactInfo.CreateEmail("test@khcs.com"),
        VerificationChannel.EMAIL,
        TimeSpan.FromMinutes(-1)); // Expirado inmediatamente

    // Act
    var act = () => request.Validate("123456");

    // Assert
    act.Should().Throw<VerificationExpiredException>();
}
```

### 5.2. Integration Tests (Atomicidad en Redis y Anti-Brute Force)

```csharp
[Test]
public async Task VerificationHandler_WhenRateLimitExceeded_ThrowsRateLimitExceededException()
{
    // Arrange
    var mockRateLimiter = new Mock<IRateLimiter>();
    mockRateLimiter.Setup(r => r.IsAllowedAsync(It.IsAny<string>(), It.IsAny<int>(), It.IsAny<TimeSpan>(), It.IsAny<CancellationToken>()))
                   .ReturnsAsync(false);

    var handler = new VerificationHandler(
        mockRateLimiter.Object,
        Mock.Of<IOtpGenerator>(),
        Mock.Of<ISecureCompare>(),
        Mock.Of<IGatewayAdapter>(),
        Mock.Of<IVerificationRepository>(),
        Mock.Of<IContactRepository>(),
        Mock.Of<ILogger<VerificationHandler>>());

    var command = new InitiateVerificationCommand
    {
        ContactHash = "test_hash",
        ContactValue = "test@khcs.com",
        Channel = "EMAIL"
    };

    // Act & Assert
    var act = () => handler.InitiateVerification(command, CancellationToken.None);
    await act.Should().ThrowAsync<RateLimitExceededException>();
}

[Test]
public async Task VerificationHandler_WhenBruteForceDetected_ThrowsBruteForceDetectedException()
{
    // Arrange
    var mockRateLimiter = new Mock<IRateLimiter>();
    mockRateLimiter.Setup(r => r.IncrementAttemptsAsync(It.IsAny<string>(), It.IsAny<CancellationToken>()))
                   .ReturnsAsync(6); // Excede el máximo de 5

    var mockRepo = new Mock<IVerificationRepository>();
    var request = VerificationRequest.Initiate(
        new IdentityId(Guid.NewGuid()),
        new TenantId(Guid.NewGuid()),
        ContactInfo.CreateEmail("test@khcs.com"),
        VerificationChannel.EMAIL,
        TimeSpan.FromHours(24));
    
    mockRepo.Setup(r => r.GetChallengeFromRedisAsync(It.IsAny<string>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(request);

    var handler = new VerificationHandler(
        Mock.Of<IRateLimiter>(),
        Mock.Of<IOtpGenerator>(),
        Mock.Of<ISecureCompare>(),
        Mock.Of<IGatewayAdapter>(),
        mockRepo.Object,
        Mock.Of<IContactRepository>(),
        Mock.Of<ILogger<VerificationHandler>>());

    var command = new CompleteVerificationCommand
    {
        ContactHash = "test_hash",
        ProvidedChallenge = "123456"
    };

    // Act & Assert
    var act = () => handler.CompleteVerification(command, CancellationToken.None);
    await act.Should().ThrowAsync<BruteForceDetectedException>();
}

[Test]
public async Task RateLimiter_IncrementAttemptsAsync_IsAtomic_UnderConcurrency()
{
    // Arrange
    var redis = ConnectionMultiplexer.Connect("localhost");
    var rateLimiter = new RateLimiter(redis);
    var key = $"test_attempts:{Guid.NewGuid()}";

    // Act: Simular 10 requests concurrentes
    var tasks = Enumerable.Range(0, 10)
        .Select(_ => rateLimiter.IncrementAttemptsAsync(key, CancellationToken.None))
        .ToArray();

    var results = await Task.WhenAll(tasks);

    // Assert: Todos los resultados deben ser únicos y secuenciales
    results.Should().BeInAscendingOrder();
    results.Should().HaveCount(10);
    results.Last().Should().Be(10);

    // Cleanup
    await redis.GetDatabase().KeyDeleteAsync(key);
}
```

### 5.3. Load Testing (Picos de Verificación y Anti-Brute Force)

**Objetivo:** Validar que el sistema maneja 100 verificaciones/minuto sin degradar el rendimiento, y que el Anti-Brute Force funciona bajo carga concurrente.

**Script k6 (Extracto):**

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');
const bruteForceRate = new Rate('brute_force_detected');

export const options = {
  scenarios: {
    verification_spike: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 20 },   // Calentamiento
        { duration: '3m', target: 100 },  // Pico de verificaciones
        { duration: '1m', target: 100 },  // Mantener pico
        { duration: '1m', target: 0 },    // Enfriamiento
      ],
    },
    brute_force_attack: {
      executor: 'constant-vus',
      vus: 10,
      duration: '2m',
      exec: 'bruteForceAttack',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<100'], // SLO de 100ms
    errors: ['rate<0.01'],             // Menos del 1% de errores
    brute_force_detected: ['rate>0.80'], // 80% de los ataques deben ser detectados
  },
};

export default function () {
  const payload = JSON.stringify({
    contact_type: 'EMAIL',
    contact_value: `user_${__VU}_${__ITER}@khcs.com`,
    channel: 'EMAIL'
  });

  const res = http.post(`${__ENV.BASE_URL}/v1/verification/initiate`, payload, {
    headers: { 
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${__ENV.USER_TOKEN}`
    },
  });

  const success = check(res, {
    'verification initiated (202)': (r) => r.status === 202,
    'has expires_in': (r) => r.json('expires_in_seconds') !== undefined,
  });

  if (!success) errorRate.add(1);

  sleep(0.6); // ~100 verificaciones/min por VU
}

export function bruteForceAttack() {
  const payload = JSON.stringify({
    contact_hash: 'test_contact_hash',
    provided_challenge: '000000' // Código incorrecto
  });

  const res = http.post(`${__ENV.BASE_URL}/v1/verification/complete`, payload, {
    headers: { 
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${__ENV.USER_TOKEN}`
    },
  });

  // Esperamos 400 (invalid code) o 429 (brute force detected)
  check(res, {
    'invalid code or brute force': (r) => r.status === 400 || r.status === 429,
  });

  if (res.status === 429) {
    bruteForceRate.add(1);
  }

  sleep(0.1); // 10 intentos/segundo por VU
}
```

---

## 6. Deployment y Operaciones

### 6.1. Feature Flags para Rollout Gradual

| Feature Flag | Descripción | Valor por Defecto |
| :--- | :--- | :--- |
| `kagami.verification.enabled` | Activa/desactiva la verificación de contactos globalmente. | `true` |
| `kagami.verification.sms_enabled` | Activa/desactiva verificaciones por SMS. | `true` |
| `kagami.verification.email_enabled` | Activa/desactiva verificaciones por Email. | `true` |
| `kagami.verification.rate_limiting_enabled` | Activa Rate Limiting para prevenir abuso. | `true` |
| `kagami.verification.anti_brute_force_enabled` | Activa Anti-Brute Force con contador de intentos. | `true` |
| `kagami.verification.anti_enumeration_enabled` | Activa prevención de enumeración en flujos públicos. | `true` |

### 6.2. Configuración de Gateways Externos

```yaml
# Configuración de Twilio para SMS
twilio:
  account_sid: "${TWILIO_ACCOUNT_SID}"
  auth_token: "${TWILIO_AUTH_TOKEN}"
  from_number: "+1234567890"
  timeout_seconds: 5
  retry_attempts: 3
  retry_delay_seconds: 2  # Backoff exponencial: 2s, 4s, 8s

# Configuración de SendGrid para Email
sendgrid:
  api_key: "${SENDGRID_API_KEY}"
  from_email: "noreply@shugo.io"
  from_name: "SHUGO IAM"
  timeout_seconds: 10
  retry_attempts: 3
  retry_delay_seconds: 2
```

### 6.3. Configuración de Redis

```yaml
# Configuración de Redis para OTPs y Rate Limiting
redis:
  connection_string: "${REDIS_CONNECTION_STRING}"
  otp_ttl_minutes: 10
  magic_link_ttl_hours: 24
  rate_limit_window_minutes: 60
  max_attempts_per_contact: 5
  max_requests_per_hour: 5
```

### 6.4. Escalado Horizontal de Verification Handlers

```yaml
# Kubernetes HPA para Verification Handlers
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kagami-verification-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kagami-verification
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
          name: kagami_verification_initiation_latency
        target:
          type: AverageValue
          averageValue: "80" # Escalar si latencia p95 > 80ms
    - type: External
      external:
        metric:
          name: kagami_verification_initiated_total
        target:
          type: AverageValue
          averageValue: "100" # Escalar si >100 verificaciones/min por pod
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

**Escenario: SMS Gateway Caído**

1. **Paso 1:** Verificar alerta `sms_gateway_errors_total > 10` en 5 min.
2. **Paso 2:** Revisar logs de `ResilientSmsGateway` para confirmar Circuit Breaker OPEN.
3. **Paso 3:** Las verificaciones por SMS están fallando. Ofrecer fallback a Email si está disponible.
4. **Paso 4:** Revisar estado de Twilio en https://status.twilio.com/.
5. **Paso 5:** Una vez recuperado, el Circuit Breaker transiciona a HALF-OPEN y luego CLOSED automáticamente.
6. **Paso 6:** Monitorear métricas `sms_gateway_latency` y `gateway_errors_total` para confirmar recuperación.

**Escenario: Ataque de Fuerza Bruta Detectado**

1. **Paso 1:** Verificar alerta `brute_force_detected_total > 50` en 10 min.
2. **Paso 2:** Revisar logs de `VerificationHandler` para identificar IPs y contactos atacados.
3. **Paso 3:** Bloquear IPs ofensivas en WAF.
4. **Paso 4:** Notificar al equipo de seguridad para investigación forense.
5. **Paso 5:** Monitorear métricas `brute_force_detected_total` para confirmar que el ataque cesó.

**Escenario: Redis Latency Alta**

1. **Paso 1:** Verificar alerta `redis_operation_latency_p95 > 50ms`.
2. **Paso 2:** Revisar conectividad entre pods y Redis.
3. **Paso 3:** Verificar carga de CPU y memoria en Redis.
4. **Paso 4:** Si Redis está saturado, considerar escalar verticalmente o agregar nodos al cluster.
5. **Paso 5:** Monitorear métricas `redis_operation_latency` para confirmar mejora.

---

## 7. Compliance y Auditoría

### 7.1. NIST SP 800-63A (Enrollment & Identity Proofing)

*   **Requisito:** Los OTPs deben ser generados criptográficamente seguros y tener entropía suficiente.
*   **Implementación UC-09:** Los OTPs se generan con `RandomNumberGenerator.Create()` (6 dígitos = 20 bits de entropía). Los Magic Links usan UUID v4 (128 bits de entropía).

### 7.2. NIST SP 800-63B (Authentication)

*   **Requisito:** Los OTPs deben tener TTL estricto y límite de intentos.
*   **Implementación UC-09:** TTL de 10 minutos para OTPs, 24 horas para Magic Links. Máximo 5 intentos fallidos antes de invalidación.

### 7.3. SOC 2 - CC6.1 (Logical Access Controls)

*   **Requisito:** La organización debe restringir el acceso lógico a los datos según el rol del usuario.
*   **Implementación UC-09:** Los contactos se almacenan como hashes SHA256, protegiendo PII en caso de compromiso de la base de datos.

### 7.4. GDPR - Art. 5 (Principles relating to processing of personal data)

*   **Requisito:** Los datos personales deben ser procesados de forma segura.
*   **Implementación UC-09:** Hashing SHA256 de contactos, Rate Limiting para prevenir abuso, y auditoría completa de todas las operaciones.

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los requisitos de Rate Limiting, SecureCompare, y atomicidad que estos artefactos implementan. |
| `08_Final_Class_Design.md` | Las clases `VerificationHandler`, `RateLimiter`, `SecureCompare`, y `OtpGenerator` se implementan según los contratos y métricas definidos aquí. |
| `10_UC_Specific_Data_Model.md` | Define la estructura física de las claves en Redis y las políticas de TTL. |
| `11_UC_Specific_Security_Threats.md` | Las mitigaciones de "OTP Brute-Forcing", "SMS Bombing", y "Timing Attacks" se validan mediante los patrones definidos aquí. |
| `12_UC_Specific_Performance_Benchmarks.md` | Los SLAs de latencia (<100ms p95) y throughput (100 verificaciones/min) se validan mediante las métricas y load testing definidos aquí. |
| `SHUGO_Security_Architecture.md` (Transversal) | Este documento aplica los estándares globales de criptografía y control de acceso al contexto específico de UC-09. |

---

## 9. Cierre del Paquete de Diseño Funcional (UC-09)

Con este documento, completamos los **9 archivos del Paquete de Diseño Profundo Funcional (Tier 1)** para el UC-09:

1. ✅ `01_Requirements_Specification.md` - Especificación completa con 6 escenarios BDD.
2. ✅ `02_Use_Case_Diagram.md` - Diagrama de caso de uso UML con actor Usuario y Gateways.
3. ✅ `03_Activity_Diagram.md` - Diagrama de actividades con validaciones de seguridad y atomicidad.
4. ✅ `04_Initial_Domain_Model.md` - Modelo de dominio conceptual (Aggregate Root, Value Objects).
5. ✅ `05_State_Machine_Diagram.md` - Máquina de estados (PENDING → VALIDATED/EXPIRED/BRUTE_FORCED).
6. ✅ `06_Robustness_Diagram.md` - Diagrama de robustez ICONIX.
7. ✅ `07_Sequence_Diagram.md` - Diagrama de secuencia con flujos alternativos y atomicidad.
8. ✅ `08_Final_Class_Design.md` - Diseño de clases con firmas y patrones.
9. ✅ `09_Modern_Artifacts.md` - Contratos OpenAPI, métricas, resiliencia, compliance.

### 9.1. Resumen de lo Logrado

Hemos diseñado el motor de verificación Out-of-Band más robusto, seguro y transversal del mercado IAM:
- **Servicio Transversal:** Consumido por UC-06, UC-08, UC-15, y UC-18, proporcionando confianza criptográfica a toda la plataforma.
- **Seguridad por Diseño:** Generación criptográfica de OTPs, almacenamiento seguro con hashing SHA256, validación con SecureCompare, y prevención de Timing Attacks.
- **Prevención de Abuso:** Rate Limiting estricto para prevenir SMS/Email Bombing, Anti-Brute Force con contador atómico de intentos, y prevención de enumeración.
- **Atomicidad en Redis:** Operaciones atómicas (`INCR`) para prevenir Race Conditions bajo carga concurrente.
- **Resiliencia Transaccional:** Circuit Breakers para gateways externos, Retry policies con backoff exponencial, y envío asíncrono (Fire-and-Forget).
- **TTL Automático:** Redis maneja la expiración de tokens automáticamente, eliminando la necesidad de jobs de limpieza.
- **Observabilidad Completa:** Métricas de éxito de verificación, intentos de fuerza bruta, latencia de gateways, y tasa de conversión permiten monitoreo proactivo.

### 9.2. Próximos Pasos (Documentos Técnicos Específicos)

Según la **Opción A+** acordada, cada UC tiene **12 documentos** (9 funcionales + 3 técnicos específicos). Para el UC-09, aún faltan:

1. **`10_UC_Specific_Data_Model.md`** - Modelo de datos físico específico (estructura de claves en Redis, políticas de TTL, hashing de contactos).
2. **`11_UC_Specific_Security_Threats.md`** - Análisis de amenazas específico (OTP Brute-Forcing, SMS Bombing, Timing Attacks, Token Replay).
3. **`12_UC_Specific_Performance_Benchmarks.md`** - Benchmarks y SLAs específicos para latencia de iniciación, throughput de verificaciones, y detección de fuerza bruta.

---
*Fin del Documento 09_Modern_Artifacts para UC-09*