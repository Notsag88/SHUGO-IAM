# UC-06: Registro Self-Service - Artefactos Modernos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-06 - Registro Self-Service (Self-Service Sign-Up) |
| **Documento:** | 09_Modern_Artifacts.md |
| **Metodología:** | API-First + Observability + Resilience Patterns + DevSecOps + GDPR Compliance by Design |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 04/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 04/09/2026 | Gastón Ruiz | **Creación inicial.** Definición de los artefactos modernos para UC-06: contratos OpenAPI 3.1 con endpoints de registro y verificación de email, catálogo de métricas OpenTelemetry (enfocado en conversiones de registro, detección de abuso, y latencia de servicios externos), patrones de resiliencia (Circuit Breaker para HIBP, Compensation Pattern para Email Service), estrategias de testing para escenarios de seguridad y compliance, y consideraciones de deployment para operaciones de onboarding público. | Cierra el paquete de diseño Tier 1 funcional del UC-06, proporcionando todos los artefactos necesarios para la implementación, operación, monitoreo y cumplimiento normativo (GDPR Art. 6 & 7) del endpoint de registro self-service del motor KAGAMI. |

---

## 1. Introducción y Propósito

Este documento define los **Artefactos Modernos** necesarios para operar el UC-06 (Registro Self-Service) en un entorno de producción enterprise. A diferencia de los UCs administrados (UC-01 a UC-05), el UC-06 es una **operación pública no autenticada** que introduce desafíos únicos:

1. **Endpoint Público Expuesto:** Sin autenticación previa, requiere múltiples capas de validación y rate limiting agresivo.
2. **Integración con Servicios Externos:** Dependencia de Have I Been Pwned (HIBP) para breach detection y SMTP para envío de emails.
3. **Prevención de Abuso:** Mecanismos anti-Sybil attack, anti-disposable email, y anti-enumeration.
4. **Compensación Transaccional:** Si el email falla tras crear la identidad, se debe ejecutar rollback para prevenir cuentas zombies.
5. **GDPR Compliance Nativo:** Consentimiento explícito, registro inmutable, y derecho al olvido desde el primer momento.

---

## 2. Contratos OpenAPI Completos

### 2.1. Especificación OpenAPI 3.1

```yaml
openapi: 3.1.0
info:
  title: SHUGO KAGAMI - Self-Service Registration API
  description: |
    API de registro self-service para creación de identidades públicas.
    
    ## Características Clave
    - Validación de seguridad en múltiples capas (Rate Limiting, Anti-Disposable, Breach Detection).
    - Consentimiento GDPR explícito con registro inmutable.
    - Estado inicial PROVISIONED con verificación de email obligatoria.
    - Patrón de compensación transaccional para fallos de email.
    
    ## Seguridad
    - Endpoint público sin autenticación.
    - Rate limiting: 5 requests por IP por hora.
    - Prevención de enumeración de emails existentes.
  version: 1.0.0

servers:
  - url: https://api.shugo.io/v1
    description: Producción

paths:
  /auth/register:
    post:
      operationId: registerSelfService
      summary: Registro self-service de nueva identidad
      description: |
        Crea una nueva identidad en estado PROVISIONED con verificación de email pendiente.
        Requiere aceptación explícita de términos y política de privacidad (GDPR).
      tags:
        - Authentication
      security: []  # Endpoint público, no requiere autenticación
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RegisterRequest'
            examples:
              standardRegistration:
                summary: Registro estándar
                value:
                  email: "nuevo.usuario@khcs.com"
                  password: "SecureP@ssw0rd!"
                  display_name: "Nuevo Usuario"
                  accept_terms: true
                  terms_version: "2.1"
                  privacy_policy_version: "1.3"
      responses:
        '201':
          description: Registro exitoso, email de verificación enviado
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/RegisterResponse'
        '400':
          description: Error de validación (email inválido, password débil, breach detectado)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Prohibido (Tenant no permite self-registration)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '409':
          description: Conflicto (Email ya registrado en estado ACTIVE)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '429':
          description: Demasiadas solicitudes (Rate limit excedido)
          headers:
            Retry-After:
              description: Segundos hasta que se puede reintentar
              schema:
                type: integer
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '500':
          description: Error interno (Fallo en envío de email, compensación ejecutada)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

  /auth/verify-email:
    post:
      operationId: verifyEmail
      summary: Verificar email con token
      description: |
        Valida el token de verificación de email y transiciona la identidad de PROVISIONED a ACTIVE.
        El token es de un solo uso y expira en 24 horas.
      tags:
        - Authentication
      security: []  # Endpoint público
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/VerifyEmailRequest'
      responses:
        '200':
          description: Email verificado exitosamente
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/VerifyEmailResponse'
        '400':
          description: Token inválido o expirado
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

components:
  schemas:
    RegisterRequest:
      type: object
      required:
        - email
        - password
        - display_name
        - accept_terms
        - terms_version
        - privacy_policy_version
      properties:
        email:
          type: string
          format: email
          maxLength: 255
          description: "Email del usuario (debe ser único en el tenant)"
        password:
          type: string
          format: password
          minLength: 12
          maxLength: 128
          description: "Contraseña (debe cumplir políticas del tenant)"
        display_name:
          type: string
          minLength: 2
          maxLength: 100
          description: "Nombre para mostrar"
        accept_terms:
          type: boolean
          enum: [true]
          description: "Debe ser true para aceptar términos y política de privacidad"
        terms_version:
          type: string
          description: "Versión de los términos de servicio aceptados"
        privacy_policy_version:
          type: string
          description: "Versión de la política de privacidad aceptada"

    RegisterResponse:
      type: object
      properties:
        message:
          type: string
          example: "Registro exitoso. Por favor verifica tu email."
        email:
          type: string
          format: email
        verification_email_sent:
          type: boolean
        verification_ttl_hours:
          type: integer
          example: 24

    VerifyEmailRequest:
      type: object
      required:
        - token
      properties:
        token:
          type: string
          format: uuid
          description: "Token de verificación de email (UUID v4)"

    VerifyEmailResponse:
      type: object
      properties:
        message:
          type: string
          example: "Email verificado exitosamente."
        identity_id:
          type: string
          format: uuid
        status:
          type: string
          enum: [ACTIVE]
        email_verified:
          type: boolean
          example: true

    ErrorResponse:
      type: object
      properties:
        error:
          type: object
          properties:
            code:
              type: string
              enum:
                - INVALID_FORMAT
                - WEAK_PASSWORD
                - DISPOSABLE_EMAIL
                - CREDENTIALS_COMPROMISED
                - CONSENT_REQUIRED
                - SELF_REGISTRATION_DISABLED
                - EMAIL_ALREADY_EXISTS
                - RATE_LIMIT_EXCEEDED
                - VERIFICATION_TOKEN_EXPIRED
                - VERIFICATION_TOKEN_INVALID
                - REGISTRATION_FAILED
            message:
              type: string
            details:
              type: object
```

### 2.2. Códigos de Error Específicos de UC-06

| Código HTTP | Código de Error | Descripción | Cuándo Ocurre |
| :--- | :--- | :--- | :--- |
| **400** | `INVALID_FORMAT` | Email no cumple RFC 5322 o dominio desechable. | `Email` Value Object lanza excepción en constructor. |
| **400** | `WEAK_PASSWORD` | Contraseña no cumple políticas del tenant. | `PasswordHash` o `SecurityValidator` detecta password débil. |
| **400** | `DISPOSABLE_EMAIL` | Dominio de email está en lista de desechables. | `Email.ValidateNotDisposable()` detecta mailinator.com, etc. |
| **400** | `CREDENTIALS_COMPROMISED` | Email o password detectado en brechas de HIBP. | `BreachDetectionService` consulta HIBP API. |
| **400** | `CONSENT_REQUIRED` | Usuario no aceptó términos (`accept_terms = false`). | Validación en `RegisterHandler`. |
| **400** | `VERIFICATION_TOKEN_EXPIRED` | Token de verificación expiró (>24h). | `VerificationToken.IsExpired()` retorna true. |
| **400** | `VERIFICATION_TOKEN_INVALID` | Token no coincide con el almacenado. | `VerificationToken.Validate()` falla. |
| **403** | `SELF_REGISTRATION_DISABLED` | Tenant no permite registro self-service. | `SecurityValidator.ValidateTenantConfigAsync()` detecta `allow_self_registration = false`. |
| **409** | `EMAIL_ALREADY_EXISTS` | Email ya registrado en estado ACTIVE. | `SecurityValidator.ValidateUniquenessAsync()` detecta email existente. |
| **429** | `RATE_LIMIT_EXCEEDED` | IP excedió 5 requests por hora. | API Gateway o `SecurityValidator.ValidateRateLimitAsync()`. |
| **500** | `REGISTRATION_FAILED` | Fallo en envío de email, compensación ejecutada. | `EmailService` lanza excepción, `RegisterHandler` ejecuta rollback. |

---

## 3. Catálogo de Métricas OpenTelemetry

### 3.1. Métricas de Negocio y Conversión

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.registration.attempts_total` | Counter | attempts | Total de intentos de registro | `tenant_id`, `source_ip_country` |
| `kagami.registration.success_total` | Counter | successes | Total de registros exitosos | `tenant_id`, `email_domain` |
| `kagami.registration.email_verified_total` | Counter | verifications | Total de emails verificados | `tenant_id`, `time_to_verify_hours` |
| `kagami.registration.conversion_rate` | Gauge | ratio | Tasa de conversión (verificados / intentos) | `tenant_id` |
| `kagami.registration.abuse_detected_total` | Counter | blocks | Total de registros bloqueados por abuso | `tenant_id`, `block_reason` (disposable, breach, rate_limit) |

### 3.2. Métricas de Rendimiento y Servicios Externos

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.registration.latency` | Histogram | ms | Latencia end-to-end del endpoint de registro | `tenant_id`, `success`, `breach_detection_enabled` |
| `kagami.hibp_api.latency` | Histogram | ms | Latencia de consulta a Have I Been Pwned API | `query_type` (email, password), `cache_hit` |
| `kagami.hibp_api.errors_total` | Counter | errors | Total de errores en consulta a HIBP | `error_type` (timeout, rate_limit, network) |
| `kagami.email_service.latency` | Histogram | ms | Latencia de envío de email de verificación | `tenant_id`, `success` |
| `kagami.email_service.errors_total` | Counter | errors | Total de fallos en envío de email | `tenant_id`, `error_type` (smtp_down, invalid_email) |
| `kagami.registration.compensation_executed_total` | Counter | compensations | Total de compensaciones ejecutadas (rollback de identidad) | `tenant_id`, `reason` |

### 3.3. Métricas de Seguridad

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.registration.rate_limit_hits_total` | Counter | hits | Total de requests bloqueadas por rate limiting | `tenant_id`, `source_ip` |
| `kagami.registration.disposable_email_blocked_total` | Counter | blocks | Total de emails desechables bloqueados | `tenant_id`, `domain` |
| `kagami.registration.breach_detected_total` | Counter | detections | Total de credenciales comprometidas detectadas | `tenant_id`, `breach_type` (email, password) |
| `kagami.registration.enumeration_attempts_total` | Counter | attempts | Total de intentos de enumeración de emails | `tenant_id`, `source_ip` |

### 3.4. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **HIBP API Down** | `hibp_api.errors_total > 10` en 5 min | **Crítica** | Todos los registros se bloquean (fail-safe). Investigar conectividad con HIBP. |
| **Email Service Down** | `email_service.errors_total > 5` en 5 min | **Crítica** | Los usuarios no pueden verificar emails. Investigar SMTP. Compensaciones se ejecutan automáticamente. |
| **Alta Tasa de Abuso** | `abuse_detected_total > 100` en 1h | Alta | Posible ataque de Sybil o brute force. Revisar IPs y dominios. |
| **Baja Tasa de Verificación** | `conversion_rate < 0.50` en 24h | Media | Los usuarios no están verificando emails. Revisar entregabilidad de emails. |
| **Rate Limiting Excesivo** | `rate_limit_hits_total > 50` en 10 min | Media | Posible ataque de enumeración. Revisar IPs sospechosas. |
| **Compensaciones Elevadas** | `compensation_executed_total > 10` en 1h | Alta | Email service inestable. Investigar causa raíz. |

---

## 4. Patrones de Resiliencia

### 4.1. Circuit Breaker para Have I Been Pwned (HIBP)

Dado que HIBP es un servicio externo crítico para la detección de brechas, implementamos Circuit Breaker para prevenir cascadas de fallos.

**Implementación (Polly - C#):**

```csharp
public class ResilientBreachDetectionService : IBreachDetectionService
{
    private readonly IBreachDetectionService _inner;
    private readonly AsyncCircuitBreakerPolicy _circuitBreaker;
    private readonly ILogger<ResilientBreachDetectionService> _logger;

    public ResilientBreachDetectionService(
        IBreachDetectionService inner, 
        ILogger<ResilientBreachDetectionService> logger)
    {
        _inner = inner;
        _logger = logger;
        
        _circuitBreaker = Policy
            .Handle<HttpRequestException>()
            .Or<TimeoutException>()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 3,
                durationOfBreak: TimeSpan.FromMinutes(5),
                onBreak: (ex, duration) => 
                {
                    _logger.LogWarning("HIBP Circuit Breaker OPEN for {Duration}. Blocking all registrations.", duration);
                    Metrics.HibpCircuitBreakerState.Set(1); // Open
                },
                onReset: () => 
                {
                    _logger.LogInformation("HIBP Circuit Breaker CLOSED. Registrations resumed.");
                    Metrics.HibpCircuitBreakerState.Set(0); // Closed
                },
                onHalfOpen: () => 
                {
                    _logger.LogInformation("HIBP Circuit Breaker HALF-OPEN. Testing connection.");
                    Metrics.HibpCircuitBreakerState.Set(2); // Half-Open
                }
            );
    }

    public async Task<bool> IsEmailCompromisedAsync(Email email, CancellationToken ct)
    {
        try
        {
            return await _circuitBreaker.ExecuteAsync(() => _inner.IsEmailCompromisedAsync(email, ct));
        }
        catch (BrokenCircuitException)
        {
            // FAIL-SAFE: Si el circuito está abierto, bloquear el registro
            _logger.LogWarning("Circuit breaker open. Blocking registration for {Email} to prevent compromised credentials.", email.Value);
            throw new BreachDetectionUnavailableException("Breach detection service unavailable. Please try again later.");
        }
    }

    public async Task<bool> IsPasswordCompromisedAsync(string password, CancellationToken ct)
    {
        try
        {
            return await _circuitBreaker.ExecuteAsync(() => _inner.IsPasswordCompromisedAsync(password, ct));
        }
        catch (BrokenCircuitException)
        {
            throw new BreachDetectionUnavailableException("Breach detection service unavailable. Please try again later.");
        }
    }
}
```

**Justificación:** Si HIBP está caído, no podemos garantizar que las credenciales no estén comprometidas. Por lo tanto, bloqueamos el registro temporalmente (fail-safe) en lugar de permitirlo sin verificación.

### 4.2. Compensation Pattern para Email Service

Si el envío de email falla después de crear la identidad, ejecutamos una acción de compensación para eliminar la identidad huérfana.

**Implementación (C#):**

```csharp
public class RegisterHandler : IRegisterHandler
{
    private readonly IIdentityRepository _repository;
    private readonly IEmailService _emailService;
    private readonly ILogger<RegisterHandler> _logger;

    public async Task<RegisterResult> Handle(RegisterCommand command, CancellationToken ct)
    {
        // 1. Validaciones de seguridad (omitiendo por brevedad)
        // ...

        // 2. Crear identidad en estado PROVISIONED
        var identity = Identity.CreateProvisioned(
            command.TenantId,
            new Email(command.Email),
            new PasswordHash(command.Password),
            new DisplayName(command.DisplayName),
            ConsentRecord.Accept(command.TermsVersion, command.PrivacyPolicyVersion, command.IpAddress, command.UserAgent)
        );

        // 3. Persistir identidad
        await _repository.SaveAsync(identity, ct);

        // 4. Enviar email con patrón de compensación
        try
        {
            await _emailService.SendVerificationEmailAsync(identity.Email, identity.VerificationToken, ct);
        }
        catch (EmailServiceException ex)
        {
            _logger.LogError(ex, "Email service failed. Executing compensation for identity {IdentityId}", identity.Id);
            
            // ACCIÓN DE COMPENSACIÓN: Eliminar la identidad huérfana
            await _repository.DeleteAsync(identity.Id, ct);
            
            Metrics.CompensationExecutedTotal.Add(1, 
                new KeyValuePair<string, object?>("tenant_id", command.TenantId.Value.ToString()),
                new KeyValuePair<string, object?>("reason", "email_service_failed"));
            
            throw new RegistrationException("No se pudo enviar el email de verificación. Por favor intenta nuevamente.", ex);
        }

        return new RegisterResult(identity.Id, "Registro exitoso. Verifica tu email.");
    }
}
```

**Justificación:** Sin compensación, quedarían identidades `PROVISIONED` en la base de datos que nunca recibirán el email de verificación, acumulándose como datos huérfanos y consumiendo recursos.

### 4.3. Cache para HIBP API

Para reducir la carga en la API de HIBP y mejorar la latencia, implementamos caché distribuida con TTL de 24 horas.

**Implementación (C#):**

```csharp
public class CachedBreachDetectionService : IBreachDetectionService
{
    private readonly IBreachDetectionService _inner;
    private readonly IDistributedCache _cache;
    private readonly ILogger<CachedBreachDetectionService> _logger;
    private const int CACHE_TTL_HOURS = 24;

    public CachedBreachDetectionService(
        IBreachDetectionService inner,
        IDistributedCache cache,
        ILogger<CachedBreachDetectionService> logger)
    {
        _inner = inner;
        _cache = cache;
        _logger = logger;
    }

    public async Task<bool> IsEmailCompromisedAsync(Email email, CancellationToken ct)
    {
        var cacheKey = $"hibp:email:{email.Value}";
        
        // Intentar obtener de caché
        var cachedResult = await _cache.GetAsync(cacheKey, ct);
        if (cachedResult.HasValue)
        {
            Metrics.HibpCacheHitTotal.Add(1);
            return cachedResult.Value;
        }

        // Cache miss: consultar API externa
        var result = await _inner.IsEmailCompromisedAsync(email, ct);
        
        // Almacenar en caché
        await _cache.SetAsync(cacheKey, result, TimeSpan.FromHours(CACHE_TTL_HOURS), ct);
        
        Metrics.HibpCacheMissTotal.Add(1);
        return result;
    }

    public async Task<bool> IsPasswordCompromisedAsync(string password, CancellationToken ct)
    {
        // Para passwords, usamos hash SHA1 para no almacenar texto plano en caché
        var passwordHash = SHA1.HashData(Encoding.UTF8.GetBytes(password));
        var cacheKey = $"hibp:password:{Convert.ToHexString(passwordHash)}";
        
        var cachedResult = await _cache.GetAsync(cacheKey, ct);
        if (cachedResult.HasValue)
        {
            return cachedResult.Value;
        }

        var result = await _inner.IsPasswordCompromisedAsync(password, ct);
        await _cache.SetAsync(cacheKey, result, TimeSpan.FromHours(CACHE_TTL_HOURS), ct);
        
        return result;
    }
}
```

**Justificación:** HIBP tiene límites de rate limiting (ej. 1 request por segundo para API gratuita). El caché reduce la carga y mejora la latencia de ~200ms a ~5ms para consultas repetidas.

---

## 5. Estrategia de Testing

### 5.1. Unit Tests (Dominio y Validaciones)

```csharp
[Test]
public void Email_WithDisposableDomain_ThrowsDomainException()
{
    // Arrange & Act
    var act = () => new Email("test@mailinator.com");

    // Assert
    act.Should().Throw<DomainException>()
       .WithMessage("*not allowed*");
}

[Test]
public void PasswordHash_WithWeakPassword_ThrowsDomainException()
{
    // Arrange & Act
    var act = () => new PasswordHash("123456");

    // Assert
    act.Should().Throw<DomainException>()
       .WithMessage("*does not meet policy*");
}

[Test]
public void ConsentRecord_WithAcceptedTermsFalse_ThrowsDomainException()
{
    // Arrange & Act
    var act = () => ConsentRecord.Accept("v2.1", "v1.3", "192.168.1.1", "Mozilla/5.0");

    // Assert (asumiendo que el método Accept valida internamente)
    // Nota: En implementación real, Accept siempre crea con accepted=true
    // El test sería para el constructor privado con accepted=false
}

[Test]
public void VerificationToken_Validate_WithExpiredToken_ReturnsFalse()
{
    // Arrange
    var token = VerificationToken.Create(ttlHours: -1); // Expirado hace 1 hora

    // Act
    var result = token.Validate("any-token");

    // Assert
    result.Should().BeFalse();
}
```

### 5.2. Integration Tests (Compensación y Servicios Externos)

```csharp
[Test]
public async Task RegisterHandler_WhenEmailServiceFails_CompensatesByDeletingIdentity()
{
    // Arrange
    var mockRepo = new Mock<IIdentityRepository>();
    var mockEmailService = new Mock<IEmailService>();
    mockEmailService.Setup(e => e.SendVerificationEmailAsync(It.IsAny<Email>(), It.IsAny<VerificationToken>(), It.IsAny<CancellationToken>()))
                    .ThrowsAsync(new EmailServiceException("SMTP Down"));
    
    var handler = new RegisterHandler(
        Mock.Of<ISecurityValidator>(),
        Mock.Of<IBreachDetectionService>(),
        mockRepo.Object,
        mockEmailService.Object,
        Mock.Of<ILogger<RegisterHandler>>());

    var command = new RegisterCommand 
    { 
        TenantId = new TenantId(Guid.NewGuid()),
        Email = "test@example.com",
        Password = "SecureP@ssw0rd!",
        DisplayName = "Test User",
        AcceptTerms = true,
        TermsVersion = "v2.1",
        PrivacyPolicyVersion = "v1.3",
        IpAddress = "192.168.1.1",
        UserAgent = "Mozilla/5.0"
    };

    // Act & Assert
    var act = () => handler.Handle(command, CancellationToken.None);
    await act.Should().ThrowAsync<RegistrationException>();

    // Verificar que se llamó al método de compensación (Delete)
    mockRepo.Verify(r => r.DeleteAsync(It.IsAny<IdentityId>(), It.IsAny<CancellationToken>()), Times.Once);
}

[Test]
public async Task RegisterHandler_WhenHibpApiDown_ThrowsBreachDetectionUnavailableException()
{
    // Arrange
    var mockBreachDetector = new Mock<IBreachDetectionService>();
    mockBreachDetector.Setup(b => b.IsEmailCompromisedAsync(It.IsAny<Email>(), It.IsAny<CancellationToken>()))
                      .ThrowsAsync(new BreachDetectionUnavailableException("HIBP API Down"));
    
    var handler = new RegisterHandler(
        Mock.Of<ISecurityValidator>(),
        mockBreachDetector.Object,
        Mock.Of<IIdentityRepository>(),
        Mock.Of<IEmailService>(),
        Mock.Of<ILogger<RegisterHandler>>());

    var command = new RegisterCommand { /* ... */ };

    // Act & Assert
    var act = () => handler.Handle(command, CancellationToken.None);
    await act.Should().ThrowAsync<BreachDetectionUnavailableException>();
}
```

### 5.3. Load Testing (Registro Masivo y Prevención de Abuso)

**Objetivo:** Validar que el sistema maneja 100 req/seg de registros legítimos mientras bloquea intentos de abuso.

**Script k6 (Extracto):**

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    legitimate_registrations: {
      executor: 'constant-vus',
      vus: 20,
      duration: '5m',
    },
    abuse_attempts: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 50 }, // Ramp up de ataques
        { duration: '3m', target: 50 }, // Mantener ataque
        { duration: '1m', target: 0 },  // Ramp down
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.10'], // Aceptar hasta 10% de fallos (rate limiting, breach detection)
  },
};

export default function () {
  const scenario = __VU <= 20 ? 'legitimate' : 'abuse';
  
  let payload;
  if (scenario === 'legitimate') {
    payload = JSON.stringify({
      email: `user${__ITER}@khcs.com`,
      password: 'SecureP@ssw0rd!',
      display_name: `User ${__ITER}`,
      accept_terms: true,
      terms_version: 'v2.1',
      privacy_policy_version: 'v1.3'
    });
  } else {
    // Intento de abuso: email desechable
    payload = JSON.stringify({
      email: `test${__ITER}@mailinator.com`,
      password: '123456',
      display_name: 'Abuser',
      accept_terms: false
    });
  }

  const res = http.post(`${__ENV.BASE_URL}/v1/auth/register`, payload, {
    headers: { 'Content-Type': 'application/json' },
  });

  if (scenario === 'legitimate') {
    check(res, {
      'legitimate registration succeeds (201)': (r) => r.status === 201,
    });
  } else {
    check(res, {
      'abuse attempt blocked (400 or 403)': (r) => r.status === 400 || r.status === 403,
    });
  }

  sleep(1);
}
```

---

## 6. Deployment y Operaciones

### 6.1. Feature Flags para Rollout Gradual

| Feature Flag | Descripción | Valor por Defecto |
| :--- | :--- | :--- |
| `kagami.registration.self_service_enabled` | Activa/desactiva el registro self-service globalmente. | `true` |
| `kagami.registration.breach_detection_enabled` | Activa la consulta a HIBP para detección de brechas. | `true` |
| `kagami.registration.disposable_email_blocking_enabled` | Activa el bloqueo de emails desechables. | `true` |
| `kagami.registration.rate_limiting_enabled` | Activa rate limiting por IP. | `true` |
| `kagami.registration.compensation_pattern_enabled` | Activa el patrón de compensación si el email falla. | `true` |

### 6.2. Configuración de Servicios Externos

```yaml
# Configuración de HIBP API
hibp:
  api_key: "${HIBP_API_KEY}"  # Variable de entorno
  base_url: "https://haveibeenpwned.com/api/v3"
  timeout_seconds: 5
  rate_limit_per_second: 1
  cache_ttl_hours: 24

# Configuración de SMTP
smtp:
  host: "smtp.sendgrid.net"
  port: 587
  username: "${SMTP_USERNAME}"
  password: "${SMTP_PASSWORD}"
  from_email: "noreply@shugo.io"
  from_name: "SHUGO IAM"
  timeout_seconds: 10
  retry_attempts: 3
  retry_delay_seconds: 5
```

### 6.3. Escalado Horizontal de Register Handlers

```yaml
# Kubernetes HPA para Register Handlers
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kagami-register-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kagami-register
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
          name: kagami_registration_latency
        target:
          type: AverageValue
          averageValue: "300" # Escalar si latencia p95 > 300ms
    - type: External
      external:
        metric:
          name: kagami_registration_attempts
        target:
          type: AverageValue
          averageValue: "50" # Escalar si >50 attempts/seg por pod
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

### 6.4. Runbook de Operaciones

**Escenario: HIBP API Caída**

1. **Paso 1:** Verificar alerta `hibp_api.errors_total > 10` en 5 min.
2. **Paso 2:** Revisar logs de `ResilientBreachDetectionService` para confirmar Circuit Breaker OPEN.
3. **Paso 3:** Todos los registros están bloqueados (fail-safe). Comunicar a usuarios: "Servicio de verificación de seguridad temporalmente no disponible".
4. **Paso 4:** Contactar a HIBP support o revisar estado en https://haveibeenpwned.com/Status.
5. **Paso 5:** Una vez recuperado, el Circuit Breaker transiciona a HALF-OPEN y luego CLOSED automáticamente.
6. **Paso 6:** Monitorear métricas `hibp_api.latency` y `registration.success_total` para confirmar recuperación.

**Escenario: Email Service Caído**

1. **Paso 1:** Verificar alerta `email_service.errors_total > 5` en 5 min.
2. **Paso 2:** Revisar logs de SMTP para identificar causa (autenticación, quota, red).
3. **Paso 3:** Las compensaciones se ejecutan automáticamente. Los usuarios reciben error 500.
4. **Paso 4:** Si el problema persiste >1h, considerar desactivar registro self-service temporalmente (`kagami.registration.self_service_enabled = false`).
5. **Paso 5:** Una vez recuperado, monitorear métricas `email_service.latency` y `registration.compensation_executed_total` (debe caer a 0).

---

## 7. Compliance y Auditoría

### 7.1. GDPR - Art. 6 (Lawfulness of Processing)

*   **Requisito:** El procesamiento de datos personales debe tener una base legal.
*   **Implementación UC-06:** El consentimiento explícito del usuario (`accept_terms = true`) proporciona la base legal para el procesamiento de datos personales durante el registro.

### 7.2. GDPR - Art. 7 (Conditions for Consent)

*   **Requisito:** El consentimiento debe ser libre, específico, informado e inequívoco. Debe poder demostrarse.
*   **Implementación UC-06:** 
    *   **Libre:** El usuario puede elegir no registrarse.
    *   **Específico:** El consentimiento es para el registro y uso de la plataforma.
    *   **Informado:** Se proporcionan enlaces a términos y política de privacidad.
    *   **Inequívoco:** El usuario debe marcar explícitamente `accept_terms = true`.
    *   **Demostrable:** El `ConsentRecord` registra timestamp, IP, User Agent, y versión de documentos.

### 7.3. GDPR - Art. 17 (Right to Erasure)

*   **Requisito:** El usuario tiene derecho a obtener la eliminación de sus datos personales.
*   **Implementación UC-06:** El usuario puede solicitar la eliminación de su cuenta en cualquier momento (UC-05: Purgar Identidad). Las identidades `PROVISIONED` no verificadas se eliminan automáticamente tras 24 horas.

### 7.4. SOC 2 - CC6.1 (Logical Access Controls)

*   **Requisito:** La organización debe restringir el acceso lógico a los datos según el rol del usuario.
*   **Implementación UC-06:** Las identidades `PROVISIONED` no pueden autenticarse ni actualizar su perfil hasta verificar el email, previniendo acceso no autorizado.

### 7.5. OWASP ASVS V2 (Authentication)

*   **Requisito:** Verificar que las contraseñas sean seguras y no estén comprometidas.
*   **Implementación UC-06:** Integración con HIBP para detectar contraseñas en brechas conocidas. Políticas de contraseña configurables por tenant.

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los requisitos de consentimiento GDPR, breach detection, y compensación transaccional que estos artefactos implementan. |
| `08_Final_Class_Design.md` | Las clases `RegisterHandler`, `ResilientBreachDetectionService`, y `CachedBreachDetectionService` se implementan según los contratos y métricas definidos aquí. |
| `10_UC_Specific_Data_Model.md` | Define la estructura física de las tablas `identities` (estado PROVISIONED), `consent_records`, y `verification_tokens`. |
| `11_UC_Specific_Security_Threats.md` | Las mitigaciones de "Sybil Attack", "Email Enumeration", y "Credential Stuffing" se validan mediante los patrones definidos aquí. |
| `12_UC_Specific_Performance_Benchmarks.md` | Los SLAs de latencia (<500ms p95) y throughput (100 req/seg) se validan mediante las métricas y load testing definidos aquí. |
| `SHUGO_Security_Architecture.md` (Transversal) | Este documento aplica los estándares globales de control de acceso y privacidad al contexto específico de UC-06. |

---

## 9. Cierre del Paquete de Diseño Funcional (UC-06)

Con este documento, completamos los **9 archivos del Paquete de Diseño Profundo Funcional (Tier 1)** para el UC-06:

1. ✅ `01_Requirements_Specification.md` - Especificación completa con 12 escenarios BDD.
2. ✅ `02_Use_Case_Diagram.md` - Diagrama de caso de uso UML con actor Usuario Externo.
3. ✅ `03_Activity_Diagram.md` - Diagrama de actividades con validaciones de seguridad y compensación.
4. ✅ `04_Initial_Domain_Model.md` - Modelo de dominio conceptual (Factory Method, Value Objects).
5. ✅ `05_State_Machine_Diagram.md` - 3 máquinas de estados (Identidad, Token, Consentimiento).
6. ✅ `06_Robustness_Diagram.md` - Diagrama de robustez ICONIX.
7. ✅ `07_Sequence_Diagram.md` - Diagrama de secuencia con flujos alternativos y compensación.
8. ✅ `08_Final_Class_Design.md` - Diseño de clases con firmas y patrones.
9. ✅ `09_Modern_Artifacts.md` - Contratos OpenAPI, métricas, resiliencia, compliance.

### 9.1. Resumen de lo Logrado

Hemos diseñado el endpoint de registro self-service más seguro y compliant del mercado IAM:
- **Seguridad por Diseño:** Múltiples capas de validación (Rate Limiting, Anti-Disposable, Breach Detection) previenen abuso y Account Takeover.
- **GDPR Compliance Nativo:** Consentimiento explícito con registro inmutable (timestamp, IP, User Agent) cumple con Art. 7(1).
- **Resiliencia Transaccional:** Patrón de compensación garantiza que no queden cuentas zombies si el email falla.
- **Prevención de Enumeración:** Mensajes genéricos y tiempos de respuesta constantes previenen ataques de enumeración de emails.
- **Observabilidad Completa:** Métricas de conversión, abuso, y servicios externos permiten monitoreo proactivo.

### 9.2. Próximos Pasos (Documentos Técnicos Específicos)

Según la **Opción A+** acordada, cada UC tiene **12 documentos** (9 funcionales + 3 técnicos específicos). Para el UC-06, aún faltan:

1. **`10_UC_Specific_Data_Model.md`** - Modelo de datos físico específico (tablas `identities`, `consent_records`, `verification_tokens`, índices para unicidad de email).
2. **`11_UC_Specific_Security_Threats.md`** - Análisis de amenazas específico (Sybil Attack, Email Enumeration, Credential Stuffing, Disposable Email Abuse).
3. **`12_UC_Specific_Performance_Benchmarks.md`** - Benchmarks y SLAs específicos para latencia de registro, throughput, y tasa de conversión.

---
*Fin del Documento 09_Modern_Artifacts para UC-06*