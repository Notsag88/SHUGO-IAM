# UC-01: Provisionar Nueva Identidad - Artefactos Modernos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-01 - Provisionar Nueva Identidad |
| **Documento:** | 09_Modern_Artifacts.md |
| **Metodología:** | API-First + Observability + Resilience Patterns + DevSecOps + Privacy by Design |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 30/08/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 30/08/2026 | Gastón Ruiz | **Creación inicial.** Definición de los artefactos modernos para UC-01: contratos OpenAPI completos, catálogo de métricas OpenTelemetry (incluyendo métricas de cifrado PII), patrones de resiliencia (Circuit Breaker para KMS, Outbox Pattern), rate limiting, webhooks, SDKs, seguridad adicional (Privacy by Design), observabilidad avanzada, estrategia de testing, deployment, y compliance (GDPR/SOC 2). | Cierra el paquete de diseño Tier 1 del UC-01, proporcionando todos los artefactos necesarios para la implementación, operación y monitoreo del endpoint fundamental del motor KAGAMI. |

---

## 1. Introducción y Propósito

Este documento define los **Artefactos Modernos** necesarios para operar el UC-01 (Provisionar Nueva Identidad) en un entorno de producción enterprise. Mientras que los documentos anteriores (01-08) se enfocaron en el diseño arquitectónico y de dominio, este documento se enfoca en:

1. **Contratos de API Completos**: Especificaciones OpenAPI 3.0 con schemas detallados, validación de formatos (RFC 5322, E.164), y códigos de error estandarizados.
2. **Observabilidad**: Catálogo completo de métricas, traces, y logs. **Énfasis crítico en el masking de PII** en logs y trazas.
3. **Resiliencia**: Patrones de resiliencia específicos para dependencias críticas como el **KMS (Key Management Service)** y la base de datos.
4. **Seguridad Operativa**: Rate limiting para prevenir flooding de identidades, y validación estricta de schemas.
5. **Integración**: Webhooks para notificar a sistemas downstream (ej. bienvenida por email), SDKs para clientes.
6. **Operaciones**: Deployment, health checks, y gestión del **Outbox Processor**.
7. **Compliance**: Auditoría, GDPR (creación de datos sujetos a borrado), SOC 2.

---

## 2. Contratos OpenAPI Completos

### 2.1. Especificación OpenAPI 3.0

```yaml
openapi: 3.0.3
info:
  title: SHUGO KAGAMI - Identity Provisioning API
  description: |
    API de gestión del ciclo de vida de identidades del motor KAGAMI.
    
    ## Características
    - Provisionamiento de identidades (Usuarios, Servicios, Dispositivos)
    - Cifrado automático de PII (Email, Teléfono) antes de persistencia (Privacy by Design)
    - Soporte multi-tenant con aislamiento estricto
    - Atributos custom validados contra JSON Schema del tenant
    - Patrón Outbox para garantía de eventos de dominio
    
    ## Autenticación
    Todas las requests requieren un JWT válido en el header `Authorization: Bearer <token>`.
    El JWT debe contener el claim `tenant_id` y el rol `admin` o el permiso `identity:create`.
  version: 1.0.0
  contact:
    name: SHUGO Support
    email: support@shugo.io

servers:
  - url: https://api.shugo.io/v1
    description: Producción

tags:
  - name: Identities
    description: Gestión de identidades

paths:
  /identities:
    post:
      operationId: provisionIdentity
      summary: Provisionar nueva identidad
      description: |
        Crea una nueva identidad en el sistema.
        
        ## Flujo de Seguridad
        1. Validación de JWT y Permisos.
        2. Validación de formato (Email RFC 5322, Phone E.164).
        3. Validación de unicidad y Schema.
        4. **Cifrado de PII (AES-256-GCM) vía KMS.**
        5. Persistencia atómica (Identity + Profile + OutboxEvent).
      tags:
        - Identities
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateIdentityRequest'
            examples:
              user:
                summary: Usuario Humano
                value:
                  type: "USER"
                  profile:
                    email: "juan.veterinario@khcs.com"
                    phone: "+5491123456789"
                    display_name: "Juan Veterinario"
                    custom_attributes:
                      department: "Veterinaria"
                      clearance_level: 3
              service:
                summary: Cuenta de Servicio
                value:
                  type: "SERVICE"
                  profile:
                    name: "payment-gateway-api"
                    description: "API para procesamiento de pagos"
      responses:
        '201':
          description: Identidad creada exitosamente
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IdentityResponse'
        '400':
          description: Error de validación (Formato, Schema)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '401':
          description: No autorizado
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Prohibido (Sin permisos identity:create)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '409':
          description: Conflicto (Email duplicado)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '500':
          description: Error interno (Ej. Fallo de cifrado KMS)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

  /identities/bulk:
    post:
      operationId: provisionIdentitiesBulk
      summary: Provisionamiento masivo de identidades
      description: |
        Crea múltiples identidades en una sola request.
        Procesamiento asíncrono para lotes grandes (>100).
      tags:
        - Identities
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                identities:
                  type: array
                  maxItems: 1000
                  items:
                    $ref: '#/components/schemas/CreateIdentityRequest'
      responses:
        '202':
          description: Aceptado para procesamiento asíncrono
          content:
            application/json:
              schema:
                type: object
                properties:
                  batch_id:
                    type: string
                  status_url:
                    type: string

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    CreateIdentityRequest:
      type: object
      required:
        - type
        - profile
      properties:
        type:
          type: string
          enum: [USER, SERVICE, DEVICE]
        profile:
          type: object
          properties:
            email:
              type: string
              format: email
              description: "Será cifrado antes de persistir."
            phone:
              type: string
              pattern: '^\+[1-9]\d{1,14}$' # E.164
              description: "Será cifrado antes de persistir."
            display_name:
              type: string
            name:
              type: string
              description: "Requerido para tipo SERVICE/DEVICE."
            custom_attributes:
              type: object
              additionalProperties: true

    IdentityResponse:
      type: object
      properties:
        id:
          type: string
          format: uuid
        tenant_id:
          type: string
          format: uuid
        type:
          type: string
        status:
          type: string
          enum: [PROVISIONED, ACTIVE, SUSPENDED, DEACTIVATED, PURGED]
        created_at:
          type: string
          format: date-time

    ErrorResponse:
      type: object
      properties:
        error:
          type: object
          properties:
            code:
              type: string
              example: "DUPLICATE_EMAIL"
            message:
              type: string
            details:
              type: object
```

### 2.2. Códigos de Error Estandarizados

| Código HTTP | Código de Error | Descripción | Cuándo Ocurre |
| :--- | :--- | :--- | :--- |
| **400** | `INVALID_EMAIL_FORMAT` | Email no cumple RFC 5322 | Validación de Value Object `EmailAddress` |
| **400** | `INVALID_PHONE_FORMAT` | Teléfono no cumple E.164 | Validación de Value Object `PhoneNumber` |
| **400** | `INVALID_CUSTOM_ATTRIBUTES` | Atributos no cumplen JSON Schema | Validación en `IdentityValidator` |
| **401** | `TOKEN_EXPIRED` | JWT expirado | API Gateway |
| **403** | `INSUFFICIENT_PERMISSIONS` | Falta rol admin o permiso `identity:create` | API Gateway |
| **409** | `DUPLICATE_EMAIL` | Email ya existe en el tenant | `IdentityValidator` (Query DB) |
| **500** | `PII_ENCRYPTION_FAILED` | Fallo al cifrar datos (KMS down) | `PIIEncryptor` (Fail Secure) |
| **500** | `PERSISTENCE_FAILED` | Error de base de datos | `IdentityRepository` |

---

## 3. Catálogo de Métricas OpenTelemetry

### 3.1. Métricas de Negocio y Privacidad

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.identity.provisioned_total` | Counter | identities | Total de identidades creadas exitosamente | `tenant_id`, `type` (USER/SERVICE/DEVICE) |
| `kagami.identity.provisioning_failed_total` | Counter | errors | Total de fallos en el provisionamiento | `tenant_id`, `error_code` (ej. DUPLICATE_EMAIL) |
| `kagami.pii.encryption_total` | Counter | operations | Total de operaciones de cifrado de PII | `tenant_id`, `field_type` (email/phone) |
| `kagami.pii.encryption_failed_total` | Counter | errors | Fallos de cifrado (KMS errors) | `tenant_id`, `error_type` |

### 3.2. Métricas de Rendimiento

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.identity.provisioning_latency` | Histogram | ms | Latencia end-to-end del endpoint | `tenant_id`, `success` |
| `kagami.pii.encryption_latency` | Histogram | ms | Latencia específica del cifrado (KMS call) | `tenant_id`, `algorithm` (AES-256-GCM) |
| `kagami.db.transaction_latency` | Histogram | ms | Latencia de la transacción atómica (Identity + Outbox) | `tenant_id` |

### 3.3. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **KMS Degradado** | `pii_encryption_failed_total > 5` en 1 min | **Crítica** | El sistema está en modo Fail Secure. Nadie puede crear identidades. Revisar AWS KMS/Vault. |
| **Alta Latencia de Cifrado** | `pii_encryption_latency_p95 > 100ms` | Alta | Posible throttling de KMS o problemas de red. |
| **Flood de Creación** | `provisioned_total > 1000` en 1 min por tenant | Media | Posible ataque de fuerza bruta o script mal configurado. Activar Rate Limit. |
| **Outbox Lag** | `outbox_events_pending_count > 1000` | Alta | Los eventos de dominio no se están publicando en Kafka. Revisar Outbox Processor. |

---

## 4. Patrones de Resiliencia

### 4.1. Circuit Breaker para KMS (Key Management Service)

Dado que el cifrado de PII es obligatorio (Privacy by Design), si el KMS falla, debemos fallar rápido y proteger la base de datos de intentos constantes.

**Implementación (Polly - C#):**

```csharp
public class ResilientKmsClient : IKmsClient
{
    private readonly IKmsClient _inner;
    private readonly AsyncCircuitBreakerPolicy _circuitBreaker;

    public ResilientKmsClient(IKmsClient inner)
    {
        _inner = inner;
        _circuitBreaker = Policy
            .Handle<KmsServiceUnavailableException>()
            .Or<TimeoutException>()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 3, // Falla rápido
                durationOfBreak: TimeSpan.FromSeconds(30),
                onBreak: (ex, duration) => Log.Warning("KMS Circuit Breaker OPEN. PII Encryption disabled."),
                onReset: () => Log.Information("KMS Circuit Breaker CLOSED.")
            );
    }

    public async Task<EncryptionResult> EncryptAsync(string plainText)
    {
        // Si el circuito está abierto, lanzamos excepción inmediatamente (Fail Secure)
        return await _circuitBreaker.ExecuteAsync(() => _inner.EncryptAsync(plainText));
    }
}
```

### 4.2. Outbox Pattern (Consistencia Eventual Robusta)

Para garantizar que el evento `Identity.Provisioned` se publique incluso si Kafka está caído en el momento de la creación.

**Estrategia:**
1.  **Atomicidad:** El evento se inserta en la tabla `outbox_events` dentro de la misma transacción SQL que la `Identity`.
2.  **Procesamiento Asíncrono:** Un servicio en background (`OutboxProcessor`) lee eventos pendientes y los publica en Kafka.
3.  **Retry con Backoff:** Si Kafka falla, el evento permanece en la tabla con `status = PENDING` y se reintenta con backoff exponencial.

---

## 5. Rate Limiting y Throttling

### 5.1. Estrategia de Rate Limiting

La creación de identidades es una operación sensible. Un atacante podría intentar crear miles de cuentas para spam o agotar recursos.

**Límites por Tenant:**
- **Plan Free:** 50 identidades/hora.
- **Plan Professional:** 1,000 identidades/hora.
- **Plan Enterprise:** 10,000 identidades/hora (o custom).

**Implementación:**
Se utiliza un **Sliding Window Log** en Redis para precisión.

```csharp
public class IdentityRateLimitMiddleware
{
    public async Task InvokeAsync(HttpContext context)
    {
        var tenantId = context.User.FindFirst("tenant_id")?.Value;
        var limit = await _rateLimiter.GetLimitAsync(tenantId, "identity:create");
        
        if (!await _rateLimiter.CheckAsync(tenantId, limit))
        {
            context.Response.StatusCode = 429;
            await context.Response.WriteAsJsonAsync(new { error = "Rate limit exceeded" });
            return;
        }
        await _next(context);
    }
}
```

---

## 6. Webhooks y Notificaciones

### 6.1. Eventos Disponibles para Webhooks

| Evento | Descripción | Payload (Sensible Data Masked) |
| :--- | :--- | :--- |
| `identity.provisioned` | Nueva identidad creada. | `{ identity_id, type, status, tenant_id }` (Sin email/phone en claro) |
| `identity.deactivated` | Identidad desactivada. | `{ identity_id, reason }` |
| `identity.purged` | Identidad eliminada (GDPR). | `{ identity_id, purged_at }` |

**Nota de Seguridad:** Los webhooks **NUNCA** deben enviar PII (email, teléfono) en el payload, incluso si están cifrados en la DB. El sistema downstream debe consultar la API si necesita esos datos (y tener permisos para verlos desencriptados).

---

## 7. SDKs para Clientes

### 7.1. SDK de C# (.NET 8)

```csharp
var client = new ShugoKagamiClient(new ShugoClientOptions { 
    ApiBaseUrl = "https://api.shugo.io/v1", 
    ApiKey = "shugo_sk_live_..." 
});

try 
{
    var identity = await client.Identities.ProvisionAsync(new CreateIdentityRequest 
    {
        Type = IdentityType.USER,
        Profile = new ProfileRequest 
        {
            Email = "juan@khcs.com",
            DisplayName = "Juan Veterinario",
            CustomAttributes = new Dictionary<string, object> { { "department", "Vet" } }
        }
    });
    
    Console.WriteLine($"Identity created: {identity.Id} with status {identity.Status}");
}
catch (ShugoApiException ex) when (ex.ErrorCode == "DUPLICATE_EMAIL")
{
    Console.WriteLine("Email already exists in tenant.");
}
catch (ShugoApiException ex) when (ex.ErrorCode == "PII_ENCRYPTION_FAILED")
{
    Console.WriteLine("Security system unavailable. Please try again later.");
}
```

---

## 8. Seguridad Adicional y Privacy by Design

### 8.1. Masking de PII en Logs y Traces

**Regla de Oro:** Ningún log, traza de OpenTelemetry, o métrica debe contener PII en texto plano.

**Implementación (Serilog - C#):**

```csharp
// ❌ INCORRECTO:
Log.Information("Creating identity for email {Email}", request.Email); 
// Riesgo: El email queda en texto plano en los logs de Datadog/ELK.

// ✅ CORRECTO:
Log.Information("Creating identity for tenant {TenantId}. Email validation passed.", tenantId);
// O si es necesario loguear el email para debugging, usar un hash o máscara:
Log.Information("Creating identity for email hash {EmailHash}", Hash(request.Email));
```

### 8.2. Headers de Seguridad

```http
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
Cache-Control: no-store, no-transform # Prevenir caché de respuestas con datos sensibles
```

---

## 9. Testing Strategy

### 9.1. Unit Tests (Dominio y Cifrado)

```csharp
[Test]
public void Identity_Constructor_Should_Set_Status_To_Provisioned()
{
    var profile = new Profile(EncryptedData.Empty(), EncryptedData.Empty(), "Test", CustomAttributes.Empty());
    var identity = new Identity(new TenantId(Guid.NewGuid()), IdentityType.USER, profile);
    
    Assert.AreEqual(IdentityStatus.PROVISIONED, identity.Status);
}

[Test]
public async Task PIIEncryptor_Should_Return_EncryptedData_Not_PlainText()
{
    var encryptor = new PIIEncryptor(new MockKmsClient());
    var result = await encryptor.EncryptAsync("secret@domain.com");
    
    Assert.AreNotEqual("secret@domain.com", result.CipherText);
    Assert.IsFalse(string.IsNullOrEmpty(result.IV));
}
```

### 9.2. Integration Tests (Base de Datos y Transacciones)

```csharp
[Test]
public async Task ProvisionIdentity_WhenKmsFails_Should_Rollback_Transaction()
{
    // Arrange: Mock KMS para lanzar excepción
    var kmsMock = new Mock<IKmsClient>();
    kmsMock.Setup(k => k.EncryptAsync(It.IsAny<string>())).ThrowsAsync(new KmsException());
    
    var handler = new ProvisionIdentityHandler(..., kmsMock.Object, ...);

    // Act & Assert
    await Assert.ThrowsAsync<PIIEncryptionException>(() => handler.Handle(validCommand, CancellationToken.None));
    
    // Verificar que NO se creó la identidad en la DB
    var count = await _dbContext.Identities.CountAsync();
    Assert.AreEqual(0, count);
}
```

---

## 10. Deployment y Operations

### 10.1. Kubernetes Manifest (Extracto)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shugo-kagami
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: kagami-api
        image: shugo/kagami:1.0.0
        env:
        - name: KMS_KEY_ID
          valueFrom:
            secretKeyRef:
              name: shugo-secrets
              key: kms-master-key-id
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
          limits:
            memory: "1Gi"
            cpu: "1000m"
        # Liveness probe debe verificar conectividad con KMS y DB
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
```

### 10.2. Background Services

El despliegue de KAGAMI debe incluir el **Outbox Processor** como un servicio en background (o un deployment separado de "worker") para garantizar la publicación de eventos.

```csharp
// Program.cs
services.AddHostedService<OutboxEventProcessor>();
```

---

## 11. Compliance y Auditoría

### 11.1. GDPR y Ciclo de Vida

El UC-01 es el punto de entrada de datos personales.
- **Consentimiento:** Antes de llamar a este endpoint, el sistema cliente debe haber recabado el consentimiento (registrado en el UC-36).
- **Derecho al Olvido (UC-05):** Las identidades creadas aquí son sujetas a purga. El diseño con `Profile.Anonymize()` asegura que cuando se ejecute el borrado, los datos cifrados se eliminen o anonimicen correctamente.

### 11.2. SOC 2 (CC6.1 - Logical Access)

- **Segregación de Funciones:** Solo usuarios con rol `admin` o permiso explícito `identity:create` pueden usar este endpoint.
- **Audit Trail:** Cada creación genera un evento `Identity.Provisioned` que se registra en el log inmutable de KAGE.

---

## 12. Cierre del Paquete de Diseño (UC-01)

Con este documento, completamos los **9 archivos del Paquete de Diseño Profundo (Tier 1)** para el UC-01:

1. ✅ `01_Requirements_Specification.md` - Especificación completa con escenarios BDD.
2. ✅ `02_Use_Case_Diagram.md` - Diagrama de caso de uso UML.
3. ✅ `03_Activity_Diagram.md` - Diagrama de actividades con 7 capas de procesamiento.
4. ✅ `04_Initial_Domain_Model.md` - Modelo de dominio conceptual (Aggregate Root, VOs).
5. ✅ `05_State_Machine_Diagram.md` - Máquinas de estados (Identity Lifecycle y Outbox).
6. ✅ `06_Robustness_Diagram.md` - Diagrama de robustez ICONIX.
7. ✅ `07_Sequence_Diagram.md` - Diagrama de secuencia temporal.
8. ✅ `08_Final_Class_Design.md` - Diseño de clases con firmas y patrones.
9. ✅ `09_Modern_Artifacts.md` - Contratos OpenAPI, métricas, resiliencia, SDKs.

### 12.1. Resumen de lo Logrado

Hemos diseñado el endpoint más fundamental de SHUGO con:
- **Privacy by Design:** Cifrado AES-256-GCM de PII *antes* de la persistencia, con integración segura con KMS.
- **Consistencia Robusta:** Patrón Outbox para garantizar la publicación de eventos sin bloquear la respuesta.
- **Seguridad Enterprise:** Validación estricta, rate limiting, y fail secure ante fallos de cifrado.
- **Observabilidad:** Métricas específicas para latencia de cifrado y alertas de degradación de KMS.
- **Agnosticismo:** Soporte para Usuarios, Servicios y Dispositivos con schemas custom.

### 12.2. Próximos Pasos

Con el UC-01 completamente diseñado, el siguiente paso lógico en el EPIC-01 es:

1. **Diseñar UC-02 (Actualizar Identidad):** Modificar atributos y gestionar el ciclo de vida.
2. **Diseñar UC-04 (Consultar Identidad):** Endpoint de lectura optimizado (CQRS).
3. **Diseñar UC-06 (Autenticar):** El consumo natural de la identidad creada (KAGI Engine).

---
*Fin del Documento 09_Modern_Artifacts para UC-01*