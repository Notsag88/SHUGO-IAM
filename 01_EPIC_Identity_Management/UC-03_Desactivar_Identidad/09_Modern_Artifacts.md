# UC-03: Desactivar Identidad - Artefactos Modernos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-03 - Desactivar Identidad (Soft Delete) |
| **Documento:** | 09_Modern_Artifacts.md |
| **Metodología:** | API-First + Observability + Resilience Patterns + DevSecOps + Compliance by Design |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 01/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 01/09/2026 | Gastón Ruiz | **Creación inicial.** Definición de los artefactos modernos para UC-03: contratos OpenAPI 3.1 con endpoint de acción explícito (`POST /deactivate`), catálogo de métricas OpenTelemetry (enfocado en guardas de seguridad y latencia de revocación), patrones de resiliencia (Circuit Breaker para KAGI/KIZUNA), estrategias de testing para escenarios de concurrencia y compliance, y consideraciones de deployment para operaciones críticas de offboarding. | Cierra el paquete de diseño Tier 1 funcional del UC-03, proporcionando todos los artefactos necesarios para la implementación, operación, monitoreo y cumplimiento normativo (SOC 2 / ISO 27001 / GDPR) del endpoint de desactivación del motor KAGAMI. |

---

## 1. Introducción y Propósito

Este documento define los **Artefactos Modernos** necesarios para operar el UC-03 (Desactivar Identidad) en un entorno de producción enterprise. A diferencia de los UCs anteriores, el UC-03 introduce desafíos únicos que requieren artefactos específicos:

1. **Acción de Negocio Crítica:** La desactivación es una operación de alto impacto con efectos secundarios inmediatos (revocación de sesiones, invalidación de caché).
2. **Guardas de Seguridad Complejas:** Prevención de auto-desactivación, validación de último administrador, y máquina de estados.
3. **Auditoría Forense Obligatoria:** Cada desactivación debe registrar actor, razón y timestamp inmutable para cumplimiento SOC 2 e ISO 27001.
4. **Consistencia Eventual con Fail Secure:** Garantizar que el acceso se deniegue incluso si los sistemas downstream (KAGI, KIZUNA) están caídos.

---

## 2. Contratos OpenAPI Completos

### 2.1. Especificación OpenAPI 3.1

```yaml
openapi: 3.1.0
info:
  title: SHUGO KAGAMI - Identity Deactivation API
  description: |
    API de desactivación lógica (Soft Delete) del ciclo de vida de identidades del motor KAGAMI.
    
    ## Características Clave
    - Acción de negocio explícita (POST /deactivate) vs. mutación genérica (PATCH).
    - Guardas de seguridad: Prevención de auto-desactivación, validación de último admin.
    - Auditoría forense automática con justificación (Reason) obligatoria.
    - Idempotencia nativa para integraciones SCIM/HRIS.
    - Consistencia eventual con Fail Secure para revocación de sesiones.
    
    ## Autenticación y Autorización
    Requiere JWT válido con permiso granular `identity:deactivate`.
    El actor no puede ser la misma identidad que se está desactivando.
  version: 1.0.0

servers:
  - url: https://api.shugo.io/v1
    description: Producción

paths:
  /identities/{id}/deactivate:
    post:
      operationId: deactivateIdentity
      summary: Desactivar identidad (Soft Delete)
      description: |
        Ejecuta la desactivación lógica de una identidad existente.
        Esta acción es irreversible excepto dentro del período de gracia (Grace Period).
        Genera auditoría forense y dispara revocación de sesiones en KAGI.
      tags:
        - Identities
      security:
        - BearerAuth: []
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/DeactivationRequest'
            examples:
              voluntaryTermination:
                summary: Terminación voluntaria
                value:
                  reason: "Empleado renunció voluntariamente"
                  notify_user: false
              securityIncident:
                summary: Incidente de seguridad
                value:
                  reason: "Credenciales comprometidas - bloqueo inmediato"
                  notify_user: false
              hrOffboarding:
                summary: Offboarding de RRHH
                value:
                  reason: "Contrato finalizado - 2024-01-30"
                  notify_user: true
      responses:
        '200':
          description: Identidad desactivada exitosamente (o ya estaba desactivada - idempotente)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/DeactivationResponse'
        '400':
          description: Error de validación (Auto-desactivación, Último admin, Razón inválida)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Prohibido (Sin permiso identity:deactivate)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Identidad no encontrada o está PURGED
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '500':
          description: Error interno (Ej. Fallo de persistencia, DB down)
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
    DeactivationRequest:
      type: object
      required:
        - reason
      properties:
        reason:
          type: string
          minLength: 10
          maxLength: 500
          description: "Justificación obligatoria para la desactivación. Se registra en auditoría forense."
          example: "Empleado renunció voluntariamente"
        notify_user:
          type: boolean
          default: false
          description: "Si true, envía notificación al usuario sobre la desactivación."

    DeactivationResponse:
      type: object
      properties:
        id:
          type: string
          format: uuid
        status:
          type: string
          enum: [DEACTIVATED]
        deactivated_at:
          type: string
          format: date-time
          description: "Timestamp inmutable de desactivación. Usado para calcular Grace Period."
        message:
          type: string
          example: "Identity deactivated successfully"

    ErrorResponse:
      type: object
      properties:
        error:
          type: object
          properties:
            code:
              type: string
              enum: 
                - SELF_DEACTIVATION_NOT_ALLOWED
                - LAST_ADMIN_DEACTIVATION_NOT_ALLOWED
                - INVALID_REASON
                - IDENTITY_NOT_FOUND
                - IDENTITY_ALREADY_PURGED
            message:
              type: string
            details:
              type: object
```

### 2.2. Códigos de Error Específicos de UC-03

| Código HTTP | Código de Error | Descripción | Cuándo Ocurre |
| :--- | :--- | :--- | :--- |
| **400** | `SELF_DEACTIVATION_NOT_ALLOWED` | El actor intentó desactivar su propia identidad. | `IdentityLifecycleService` detecta `actor_id == identity_id`. |
| **400** | `LAST_ADMIN_DEACTIVATION_NOT_ALLOWED` | Se intentó desactivar al último administrador activo del tenant. | `IdentityLifecycleService` cuenta admins activos <= 1. |
| **400** | `INVALID_REASON` | La razón proporcionada está vacía o excede 500 caracteres. | Validación del Value Object `DeactivationReason`. |
| **404** | `IDENTITY_NOT_FOUND` | La identidad no existe. | `IdentityRepository.GetByIdAsync` retorna null. |
| **404** | `IDENTITY_ALREADY_PURGED` | La identidad ya fue purgada (Hard Delete). | `Identity.Status == PURGED`. |
| **403** | `INSUFFICIENT_PERMISSIONS` | El actor no tiene el permiso `identity:deactivate`. | API Gateway valida JWT claims. |
| **500** | `PERSISTENCE_FAILED` | Error de base de datos durante la transacción. | `DbUpdateException` o fallo de COMMIT. |

---

## 3. Catálogo de Métricas OpenTelemetry

### 3.1. Métricas de Negocio y Seguridad

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.identity.deactivated_total` | Counter | deactivations | Total de identidades desactivadas exitosamente | `tenant_id`, `actor_type` (Admin/HRIS), `reason_category` |
| `kagami.identity.deactivation_guard_failures_total` | Counter | failures | Total de desactivaciones bloqueadas por guardas de seguridad | `tenant_id`, `guard_type` (SelfDeactivation/LastAdmin) |
| `kagami.identity.deactivation_idempotent_calls_total` | Counter | calls | Total de llamadas idempotentes (identidad ya desactivada) | `tenant_id` |
| `kagami.identity.session_revocation_latency` | Histogram | ms | Latencia de revocación de sesiones en KAGI (async) | `tenant_id`, `session_count` |

### 3.2. Métricas de Rendimiento

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.identity.deactivation_latency` | Histogram | ms | Latencia end-to-end del endpoint POST /deactivate | `tenant_id`, `success`, `has_audit_log` |
| `kagami.identity.audit_snapshot_generation_latency` | Histogram | ms | Latencia de generación del snapshot forense | `tenant_id` |
| `kagami.identity.outbox_event_publish_latency` | Histogram | ms | Latencia de publicación del evento en Kafka | `tenant_id`, `event_type` |
| `kagami.identity.last_admin_check_latency` | Histogram | ms | Latencia de la consulta "IsLastActiveAdmin" | `tenant_id` |

### 3.3. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Pico de Desactivaciones** | `deactivated_total > 100` en 10 min | **Crítica** | Posible ataque o script mal configurado. Revisar logs de auditoría inmediatamente. |
| **Auto-Desactivaciones Bloqueadas** | `deactivation_guard_failures_total{guard_type="SelfDeactivation"} > 10` en 1h | Alta | Posible intento de evasión de auditoría. Investigar actores involucrados. |
| **Último Admin Bloqueado** | `deactivation_guard_failures_total{guard_type="LastAdmin"} > 5` en 1h | Media | Administradores intentando desactivarse sin backup. Revisar política de roles. |
| **Revocación de Sesiones Lenta** | `session_revocation_latency_p95 > 5000ms` | Alta | KAGI está degradado. Ventana de vulnerabilidad aumentada. |
| **Fallo de Persistencia** | `PERSISTENCE_FAILED > 0` | **Crítica** | La identidad no se desactivó. Reintentar manualmente. |

---

## 4. Patrones de Resiliencia

### 4.1. Circuit Breaker para Sistemas Downstream (KAGI/KIZUNA)

Dado que la revocación de sesiones y la invalidación de caché son asíncronas, el Circuit Breaker se aplica a nivel de consumidor de eventos, no en el endpoint de desactivación.

**Implementación (Polly - C#):**

```csharp
public class ResilientEventPublisher : IEventPublisher
{
    private readonly IEventPublisher _inner;
    private readonly AsyncCircuitBreakerPolicy _circuitBreaker;

    public ResilientEventPublisher(IEventPublisher inner)
    {
        _inner = inner;
        _circuitBreaker = Policy
            .Handle<KafkaException>()
            .Or<TimeoutException>()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 5,
                durationOfBreak: TimeSpan.FromSeconds(60),
                onBreak: (ex, duration) => Log.Warning("Kafka Circuit Breaker OPEN. Events queued in Outbox."),
                onReset: () => Log.Information("Kafka Circuit Breaker CLOSED.")
            );
    }

    public async Task PublishAsync(DomainEvent @event)
    {
        // Si el circuito está abierto, el evento permanece en el Outbox
        // y será procesado cuando el circuito se cierre.
        await _circuitBreaker.ExecuteAsync(() => _inner.PublishAsync(@event));
    }
}
```

### 4.2. Fail Secure en la Revocación de Sesiones

Si KAGI está caído durante la desactivación:

1. **Source of Truth:** KAGAMI marca la identidad como `DEACTIVATED` en PostgreSQL.
2. **Outbox Pattern:** El evento `IdentityDeactivatedEvent` se guarda en la tabla `outbox_events`.
3. **Fail Secure:** Si un usuario intenta usar un Access Token viejo:
   - KIZUNA (Check API) consulta el estado en KAGAMI (o caché) y **deniega el acceso**.
   - KAGI (al validar el token) rechaza la autenticación porque la identidad está `DEACTIVATED`.
4. **Recuperación:** En cuanto KAGI se recupere, el Outbox Processor publicará el evento y se limpiarán las sesiones en Redis.

**Ventana de Vulnerabilidad Máxima:** TTL del Access Token (15 minutos).

### 4.3. Idempotencia para Integraciones SCIM/HRIS

Los sistemas de RRHH (Workday, BambooHR) pueden enviar múltiples solicitudes de baja para la misma identidad debido a reintentos de red o sincronizaciones programadas.

**Implementación:**
```csharp
public void Deactivate(DeactivationReason reason, Actor actor)
{
    // Idempotencia: Si ya está desactivada, retornar silenciosamente
    if (this.Status == IdentityStatus.DEACTIVATED)
        return; // No genera eventos duplicados ni logs de auditoría

    // ... resto de la lógica
}
```

**Beneficio:** Permite reintentos seguros sin duplicar eventos ni corromper logs de auditoría.

---

## 5. Estrategia de Testing

### 5.1. Unit Tests (Dominio y Guardas)

```csharp
[Test]
public void Identity_Deactivate_WhenSelfDeactivation_ThrowsException()
{
    // Arrange
    var identityId = new IdentityId(Guid.NewGuid());
    var actor = new Actor(identityId.Value, ActorType.Admin); // Mismo ID
    var identity = new Identity(identityId, /* ... */);
    var reason = new DeactivationReason("Test reason");

    // Act & Assert
    var act = () => identity.Deactivate(reason, actor);
    act.Should().Throw<SelfDeactivationNotAllowedException>()
       .WithMessage("Users cannot deactivate their own identity.");
}

[Test]
public void Identity_Deactivate_WhenAlreadyDeactivated_IsIdempotent()
{
    // Arrange
    var identity = new Identity(/* Status = DEACTIVATED */);
    var reason = new DeactivationReason("Test");
    var actor = new Actor(Guid.NewGuid(), ActorType.Admin);

    // Act
    identity.Deactivate(reason, actor);

    // Assert
    identity.GetDomainEvents().Should().BeEmpty(); // No eventos duplicados
}

[Test]
public void IdentityLifecycleService_WhenLastAdmin_ThrowsException()
{
    // Arrange
    var mockRepo = new Mock<IIdentityRepository>();
    mockRepo.Setup(r => r.GetActiveAdminCountAsync(It.IsAny<TenantId>()))
            .ReturnsAsync(1); // Solo 1 admin activo
    
    var service = new IdentityLifecycleService(mockRepo.Object);
    var identity = new Identity(/* with admin role */);
    var actor = new Actor(Guid.NewGuid(), ActorType.Admin);

    // Act & Assert
    var act = () => service.CanDeactivateAsync(identity, actor);
    act.Should().ThrowAsync<LastAdminDeactivationException>();
}
```

### 5.2. Integration Tests (Transacción Atómica y Outbox)

```csharp
[Test]
public async Task DeactivateIdentity_WhenSuccessful_CreatesAuditLogAndOutboxEvent()
{
    // Arrange: Crear identidad activa
    var identityId = await CreateTestIdentityAsync(status: IdentityStatus.ACTIVE);
    var actorId = Guid.NewGuid(); // Diferente de identityId

    // Act: Desactivar identidad
    var command = new DeactivateIdentityCommand 
    { 
        IdentityId = identityId, 
        ActorId = actorId, 
        Reason = "Integration test" 
    };
    
    await _handler.Handle(command, CancellationToken.None);

    // Assert: Verificar estado en DB
    var identity = await _dbContext.Identities.FindAsync(identityId);
    identity.Status.Should().Be(IdentityStatus.DEACTIVATED);
    identity.DeactivatedAt.Should().NotBeNull();

    // Assert: Verificar auditoría
    var auditLog = await _dbContext.AuditLogs
        .FirstOrDefaultAsync(a => a.IdentityId == identityId);
    auditLog.Should().NotBeNull();
    auditLog.Reason.Should().Be("Integration test");
    auditLog.ActorId.Should().Be(actorId);

    // Assert: Verificar Outbox
    var outboxEvent = await _dbContext.OutboxEvents
        .FirstOrDefaultAsync(e => e.AggregateId == identityId);
    outboxEvent.Should().NotBeNull();
    outboxEvent.EventType.Should().Be("Identity.Deactivated");
}

[Test]
public async Task DeactivateIdentity_WhenDbFails_RollbackEntireTransaction()
{
    // Arrange: Mock para fallar en SaveChanges
    var mockDbContext = new Mock<AppDbContext>();
    mockDbContext.Setup(db => db.SaveChangesAsync(It.IsAny<CancellationToken>()))
                 .ThrowsAsync(new DbUpdateException("DB failure"));

    var handler = new DeactivateIdentityHandler(/* ... with mock DbContext ... */);
    var command = new DeactivateIdentityCommand { /* ... */ };

    // Act & Assert
    await Assert.ThrowsAsync<DbUpdateException>(
        () => handler.Handle(command, CancellationToken.None));

    // Verificar que NO se creó auditoría ni outbox
    var auditLog = await _dbContext.AuditLogs.FirstOrDefaultAsync();
    auditLog.Should().BeNull(); // Rollback exitoso
}
```

### 5.3. Load Testing (Escenario de Offboarding Masivo)

**Objetivo:** Validar que el sistema maneja 1000 desactivaciones simultáneas sin degradar el rendimiento ni corromper datos.

**Script k6 (Extracto):**

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    mass_offboarding: {
      executor: 'constant-vus',
      vus: 100,
      duration: '1m',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<100'], // < 100ms p95
    http_req_failed: ['rate<0.01'], // < 1% fallos
  },
};

export default function () {
  // Seleccionar identidad aleatoria del pool de test
  const identityId = __ENV.IDENTITY_POOL[Math.floor(Math.random() * __ENV.IDENTITY_POOL.length)];
  
  const payload = JSON.stringify({
    reason: `Load test deactivation - ${Date.now()}`
  });

  const res = http.post(
    `${__ENV.BASE_URL}/v1/identities/${identityId}/deactivate`,
    payload,
    {
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${__ENV.JWT_TOKEN}`,
      },
    }
  );

  check(res, {
    'is success (200)': (r) => r.status === 200,
    'is not found (404 - already deactivated)': (r) => r.status === 404,
    'is not server error': (r) => r.status < 500,
  });

  sleep(0.01); // 100 requests per second per VU
}
```

---

## 6. Deployment y Operaciones

### 6.1. Feature Flags para Rollout Gradual

| Feature Flag | Descripción | Valor por Defecto |
| :--- | :--- | :--- |
| `kagami.identity.deactivation.guard_self_deactivation_enabled` | Activa la guarda de prevención de auto-desactivación. | `true` (Siempre activo en Prod) |
| `kagami.identity.deactivation.guard_last_admin_enabled` | Activa la guarda de último administrador. | `true` |
| `kagami.identity.deactivation.audit_forensic_enabled` | Activa la generación de snapshot forense con Reason. | `true` |
| `kagami.identity.deactivation.async_session_revocation_enabled` | Activa la revocación asíncrona de sesiones vía Outbox. | `true` |

### 6.2. Migraciones de Base de Datos

Para soportar UC-03, se requieren las siguientes migraciones en PostgreSQL:

```sql
-- 1. Agregar columna deactivated_at a la tabla identities (si no existe)
ALTER TABLE identities 
ADD COLUMN IF NOT EXISTS deactivated_at TIMESTAMPTZ;

-- 2. Crear índice para consultas de Grace Period (UC-05)
CREATE INDEX idx_identities_deactivated_at 
    ON identities (deactivated_at) 
    WHERE status = 'DEACTIVATED';

-- 3. Agregar columna reason a la tabla audit_logs (si no existe)
ALTER TABLE audit_logs 
ADD COLUMN IF NOT EXISTS reason VARCHAR(500);

-- 4. Crear índice para búsquedas de auditoría por razón
CREATE INDEX idx_audit_logs_reason 
    ON audit_logs (reason) 
    WHERE reason IS NOT NULL;
```

### 6.3. Runbook de Operaciones

**Escenario: Desactivación Masiva de Emergencia**

Si se detecta una brecha de seguridad y se deben desactivar 1000+ identidades inmediatamente:

1. **Paso 1:** Identificar las identidades comprometidas (query a `audit_logs` o sistema SIEM).
2. **Paso 2:** Generar script de desactivación masiva usando el endpoint `POST /identities/{id}/deactivate` con reason "Security incident - emergency deactivation".
3. **Paso 3:** Monitorear métricas `kagami.identity.deactivated_total` y `kagami.identity.session_revocation_latency`.
4. **Paso 4:** Verificar en KAGI que todas las sesiones fueron revocadas (query a Redis).
5. **Paso 5:** Generar reporte de auditoría para compliance (query a `audit_logs` con reason "Security incident").

**Tiempo Estimado:** 1000 desactivaciones en ~10 segundos (100 req/seg).

---

## 7. Compliance y Auditoría

### 7.1. SOC 2 - CC7.2 (System Monitoring)

*   **Requisito:** La organización debe monitorear el sistema para detectar comportamientos anómalos.
*   **Implementación UC-03:** Cada desactivación genera un evento `IdentityDeactivatedEvent` que alimenta el sistema de SIEM. Los picos inusuales de desactivaciones (ej. un script desactivando usuarios masivamente) disparan alertas en tiempo real.

### 7.2. ISO 27001 - A.9.2.6 (Removal of Access Rights)

*   **Requisito:** Los derechos de acceso de todos los empleados y usuarios externos deben ser removidos tras la terminación del empleo o contrato.
*   **Implementación UC-03:** El endpoint `POST /deactivate` con reason obligatorio cumple con este requisito. El log de auditoría forense proporciona evidencia para auditores.

### 7.3. GDPR - Art. 17 (Right to Erasure)

*   **Requisito:** El usuario tiene derecho a obtener el borrado de sus datos personales.
*   **Implementación UC-03:** La desactivación (Soft Delete) es el primer paso. El campo `deactivated_at` inicia el Grace Period (ej. 30 días). Tras expirar, UC-05 (Purgar) ejecuta el Hard Delete.

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los requisitos de guardas de seguridad, auditoría forense y consistencia eventual que estos artefactos implementan. |
| `08_Final_Class_Design.md` | Las clases `DeactivateIdentityHandler`, `IdentityLifecycleService`, y `AuditLogger` se implementan según los contratos y métricas definidos aquí. |
| `10_UC_Specific_Data_Model.md` | Define la estructura física de las tablas `identities` (con `deactivated_at`) y `audit_logs` (con `reason`). |
| `11_UC_Specific_Security_Threats.md` | Las mitigaciones de "Self-Deactivation" y "Last Admin Deactivation" se validan mediante las guardas definidas en este documento. |
| `SHUGO_Security_Architecture.md` (Transversal) | Este documento aplica los estándares globales de control de acceso y auditoría al contexto específico de UC-03. |

---

## 9. Cierre del Paquete de Diseño Funcional (UC-03)

Con este documento, completamos los **9 archivos del Paquete de Diseño Profundo Funcional (Tier 1)** para el UC-03:

1. ✅ `01_Requirements_Specification.md` - Especificación completa con escenarios BDD.
2. ✅ `02_Use_Case_Diagram.md` - Diagrama de caso de uso UML.
3. ✅ `03_Activity_Diagram.md` - Diagrama de actividades con guardas y auditoría.
4. ✅ `04_Initial_Domain_Model.md` - Modelo de dominio conceptual.
5. ✅ `05_State_Machine_Diagram.md` - Máquinas de estados (Transiciones, Grace Period, Sesiones).
6. ✅ `06_Robustness_Diagram.md` - Diagrama de robustez ICONIX.
7. ✅ `07_Sequence_Diagram.md` - Diagrama de secuencia con flujos alternativos.
8. ✅ `08_Final_Class_Design.md` - Diseño de clases con firmas y patrones.
9. ✅ `09_Modern_Artifacts.md` - Contratos OpenAPI, métricas, resiliencia, compliance.

### 9.1. Resumen de lo Logrado

Hemos diseñado el endpoint de desactivación más robusto y seguro del mercado IAM:
- **Acción de Negocio Explícita:** Endpoint dedicado (`POST /deactivate`) con semántica clara y permisos granulares.
- **Guardas de Seguridad:** Prevención de auto-desactivación y validación de último administrador previenen errores operativos críticos.
- **Auditoría Forense:** Snapshot inmutable con actor, razón y timestamp cumple con SOC 2 e ISO 27001.
- **Consistencia Eventual con Fail Secure:** La revocación de sesiones ocurre eventualmente, pero el acceso se deniega inmediatamente gracias al Source of Truth en KAGAMI.
- **Idempotencia Nativa:** Soporta reintentos de sistemas SCIM/HRIS sin duplicar eventos ni logs.

### 9.2. Próximos Pasos (Documentos Técnicos Específicos)

Con el paquete funcional completado, los siguientes 3 documentos técnicos específicos cerrarán el diseño completo del UC-03:

1. **`10_UC_Specific_Data_Model.md`** - Modelo de datos físico específico para UC-03 (índices para Grace Period, políticas de retención).
2. **`11_UC_Specific_Security_Threats.md`** - Análisis de amenazas específico para UC-03 (evasion de auditoría, lockout malicioso).
3. **`12_UC_Specific_Performance_Benchmarks.md`** - Benchmarks y SLAs específicos para desactivación masiva y revocación de sesiones.

---
*Fin del Documento 09_Modern_Artifacts para UC-03*