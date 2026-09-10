# UC-05: Purgar Identidad - Artefactos Modernos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-05 - Purgar Identidad (Hard Delete / GDPR Right to Erasure) |
| **Documento:** | 09_Modern_Artifacts.md |
| **Metodología:** | API-First + Observability + Resilience Patterns + DevSecOps + Compliance by Design |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 03/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 03/09/2026 | Gastón Ruiz | **Creación inicial.** Definición de los artefactos modernos para UC-05: contratos OpenAPI 3.1 con endpoints de purga individual y batch, catálogo de métricas OpenTelemetry (enfocado en bloqueos de compliance, latencia de crypto-shredding y propagación downstream), patrones de resiliencia (Circuit Breaker Fail-Safe para Legal Hold), estrategias de testing para verificar la irreversibilidad criptográfica, y consideraciones de deployment para operaciones destructivas. | Cierra el paquete de diseño Tier 1 funcional del UC-05, proporcionando todos los artefactos necesarios para la implementación, operación, monitoreo y cumplimiento normativo (GDPR Art. 17 / NIST SP 800-88) del endpoint de purga del motor KAGAMI. |

---

## 1. Introducción y Propósito

Este documento define los **Artefactos Modernos** necesarios para operar el UC-05 (Purgar Identidad) en un entorno de producción enterprise. A diferencia de los UCs previos, el UC-05 introduce desafíos únicos debido a su naturaleza destructiva e irreversible:

1. **Seguridad UX para Acciones Destructivas:** Requiere confirmaciones explícitas (`confirm_irreversible`) y justificaciones auditables.
2. **Fail-Safe en Compliance:** Si el sistema de Legal Hold falla, el sistema debe bloquear todas las purgas por defecto para evitar violaciones legales catastróficas.
3. **Verificación Criptográfica:** Las pruebas y métricas deben garantizar que el *crypto-shredding* (HMAC-SHA256) se ejecuta correctamente y que los datos originales son irrecuperables.
4. **Procesamiento Batch Resiliente:** Las purgas masivas deben soportar fallos parciales, reintentos y checkpointing sin corromper el estado de las identidades ya purgadas.

---

## 2. Contratos OpenAPI Completos

### 2.1. Especificación OpenAPI 3.1

```yaml
openapi: 3.1.0
info:
  title: SHUGO KAGAMI - Identity Purge API
  description: |
    API de purga irreversible (Hard Delete) del ciclo de vida de identidades del motor KAGAMI.
    
    ## Características Clave
    - Acción destructiva e irreversible con confirmación explícita.
    - Validación estricta de Legal Hold (bloqueador de prioridad) y Grace Period.
    - Crypto-shredding automático (HMAC-SHA256) de PII y Audit Logs.
    - Soporte para procesamiento batch (hasta 1000 identidades) con reporte de progreso.
    
    ## Advertencia
    Esta API ejecuta el Derecho al Olvido (GDPR Art. 17). Una vez ejecutada, 
    la operación NO puede ser revertida.
  version: 1.0.0

servers:
  - url: https://api.shugo.io/v1
    description: Producción

paths:
  /identities/{id}/purge:
    post:
      operationId: purgeIdentity
      summary: Purgar identidad de forma irreversible (Hard Delete)
      description: |
        Ejecuta la purga irreversible de una identidad desactivada.
        Requiere el permiso granular `identity:purge`.
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
              $ref: '#/components/schemas/PurgeRequest'
            examples:
              gdprRequest:
                summary: Solicitud GDPR Art. 17
                value:
                  reason: "GDPR Art. 17 - Solicitud directa del usuario"
                  gdpr_request_id: "gdpr-req-2024-0042"
                  confirm_irreversible: true
      responses:
        '200':
          description: Identidad purgada exitosamente
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PurgeResponse'
        '400':
          description: Error de validación (Grace Period, Estado inválido)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Prohibido (Legal Hold activo o sin permisos)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Identidad no encontrada (ocultamiento de existencia)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '500':
          description: Error interno (Fallo en anonimización o DB)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

  /identities/purge-batch:
    post:
      operationId: purgeIdentitiesBatch
      summary: Iniciar job de purga masiva
      tags:
        - Identities
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/BatchPurgeRequest'
      responses:
        '202':
          description: Job de purga aceptado y en procesamiento
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/BatchPurgeAcceptedResponse'

  /purge-jobs/{job_id}:
    get:
      operationId: getPurgeJobStatus
      summary: Consultar progreso de job de purga masiva
      tags:
        - Identities
      security:
        - BearerAuth: []
      parameters:
        - name: job_id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Estado actual del job
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/BatchPurgeJobResponse'

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    PurgeRequest:
      type: object
      required:
        - reason
        - confirm_irreversible
      properties:
        reason:
          type: string
          minLength: 10
          maxLength: 500
          description: "Justificación legal o de negocio obligatoria."
        gdpr_request_id:
          type: string
          description: "ID del ticket de GDPR para trazabilidad (Opcional)."
        confirm_irreversible:
          type: boolean
          description: "Debe ser true para confirmar que el actor entiende que la acción es irreversible."
          enum: [true] # Fuerza a que sea true en la validación

    PurgeResponse:
      type: object
      properties:
        id:
          type: string
          format: uuid
        status:
          type: string
          enum: [PURGED]
        purged_at:
          type: string
          format: date-time
        anonymized_fields:
          type: array
          items:
            type: string
        audit_logs_anonymized_count:
          type: integer

    BatchPurgeRequest:
      type: object
      required:
        - identity_ids
        - reason
        - confirm_irreversible
      properties:
        identity_ids:
          type: array
          items:
            type: string
            format: uuid
          maxItems: 1000
        reason:
          type: string
        confirm_irreversible:
          type: boolean
          enum: [true]

    BatchPurgeAcceptedResponse:
      type: object
      properties:
        job_id:
          type: string
          format: uuid
        status:
          type: string
          enum: [PENDING, PROCESSING]
        total_identities:
          type: integer
        status_url:
          type: string
          format: uri

    BatchPurgeJobResponse:
      type: object
      properties:
        job_id:
          type: string
          format: uuid
        status:
          type: string
          enum: [PENDING, PROCESSING, COMPLETED, PARTIALLY_COMPLETED, FAILED]
        total:
          type: integer
        purged:
          type: integer
        skipped:
          type: integer
        failed:
          type: integer
        skipped_details:
          type: object
          properties:
            grace_period_not_expired:
              type: integer
            legal_hold_active:
              type: integer
        started_at:
          type: string
          format: date-time
        completed_at:
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
              enum: 
                - IDENTITY_NOT_FOUND
                - LEGAL_HOLD_ACTIVE
                - GRACE_PERIOD_NOT_EXPIRED
                - INVALID_TRANSITION
                - PURGE_PERSISTENCE_FAILED
            message:
              type: string
            details:
              type: object
```

### 2.2. Códigos de Error Específicos de UC-05

| Código HTTP | Código de Error | Descripción | Cuándo Ocurre |
| :--- | :--- | :--- | :--- |
| **400** | `GRACE_PERIOD_NOT_EXPIRED` | El período de gracia no ha transcurrido. | `GracePeriodEvaluator` retorna `false`. Incluye `purge_available_at` en `details`. |
| **400** | `INVALID_TRANSITION` | La identidad no está en estado `DEACTIVATED`. | `Identity.Purge()` lanza `InvalidTransitionException`. |
| **403** | `LEGAL_HOLD_ACTIVE` | Hay retención legal activa sobre la identidad. | `LegalHoldChecker` retorna `true`. Incluye `case_reference` en `details`. |
| **404** | `IDENTITY_NOT_FOUND` | Identidad no existe o ya está `PURGED`. | Ocultamiento de existencia para prevenir enumeration attacks. |
| **500** | `PURGE_PERSISTENCE_FAILED` | Fallo en la transacción atómica (DB down, error de anonimización). | Se hace Rollback completo. Se genera alerta crítica. |

---

## 3. Catálogo de Métricas OpenTelemetry

### 3.1. Métricas de Negocio y Compliance

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.identity.purge_total` | Counter | purges | Total de purgas exitosas | `tenant_id`, `actor_type` (Admin/GDPR_System) |
| `kagami.identity.purge_blocked_legal_hold_total` | Counter | blocks | Total de purgas bloqueadas por Legal Hold | `tenant_id`, `case_reference` |
| `kagami.identity.purge_blocked_grace_period_total` | Counter | blocks | Total de purgas bloqueadas por Grace Period | `tenant_id` |
| `kagami.identity.batch_purge_jobs_total` | Counter | jobs | Total de jobs de purga masiva iniciados | `tenant_id`, `final_status` |

### 3.2. Métricas de Rendimiento y Crypto-Shredding

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.identity.purge_latency` | Histogram | ms | Latencia end-to-end del endpoint de purga individual | `tenant_id`, `success` |
| `kagami.identity.crypto_shredding_latency` | Histogram | ms | Latencia específica del HMAC-SHA256 y eliminación de atributos | `tenant_id` |
| `kagami.identity.legal_hold_check_latency` | Histogram | ms | Latencia de la consulta al sistema de Legal Hold | `tenant_id`, `circuit_breaker_state` |
| `kagami.identity.audit_log_anonymization_latency` | Histogram | ms | Latencia de la anonimización de logs históricos | `tenant_id`, `logs_count` |

### 3.3. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Legal Hold System Down** | `legal_hold_check_latency` timeout > 3 o Circuit Breaker OPEN | **Crítica** | **Todas las purgas se bloquean automáticamente (Fail-Safe).** Investigar infraestructura de Legal Hold. |
| **Alta Tasa de Bloqueos por Grace Period** | `purge_blocked_grace_period_total > 50` en 1h | Media | Posible error en el proceso de desactivación (UC-03) o UI confusa para el Admin. |
| **Fallo en Purga Individual** | `PURGE_PERSISTENCE_FAILED > 0` | **Crítica** | Error en la transacción atómica. Los datos PII no fueron anonimizados. Revisar logs de DB. |
| **Job de Purga Masiva Atascado** | Job en estado `PROCESSING` por > 2 horas | Alta | Posible deadlock en DB o fallo en el worker. Revisar métricas de throughput. |

---

## 4. Patrones de Resiliencia

### 4.1. Circuit Breaker Fail-Safe para Legal Hold

Dado que el Legal Hold tiene supremacía sobre el GDPR Art. 17, si el sistema de Legal Hold no responde, **debemos asumir que SÍ hay un Legal Hold activo** y bloquear la purga.

**Implementación (Polly - C#):**

```csharp
public class ResilientLegalHoldChecker : ILegalHoldChecker
{
    private readonly ILegalHoldChecker _inner;
    private readonly AsyncCircuitBreakerPolicy _circuitBreaker;
    private readonly ILogger<ResilientLegalHoldChecker> _logger;

    public ResilientLegalHoldChecker(ILegalHoldChecker inner, ILogger<ResilientLegalHoldChecker> logger)
    {
        _inner = inner;
        _logger = logger;
        
        _circuitBreaker = Policy
            .Handle<Exception>() // Cualquier excepción (Timeout, DB down, Network)
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 2, // Muy sensible, es compliance
                durationOfBreak: TimeSpan.FromMinutes(5),
                onBreak: (ex, duration) => 
                {
                    _logger.LogCritical("Legal Hold Circuit Breaker OPEN. BLOCKING ALL PURGES for safety.");
                    Metrics.LegalHoldCircuitBreakerState.Set(1);
                },
                onReset: () => 
                {
                    _logger.LogInformation("Legal Hold Circuit Breaker CLOSED. Purges resumed.");
                    Metrics.LegalHoldCircuitBreakerState.Set(0);
                }
            );
    }

    public async Task<bool> HasActiveHoldAsync(IdentityId identityId)
    {
        try
        {
            return await _circuitBreaker.ExecuteAsync(() => _inner.HasActiveHoldAsync(identityId));
        }
        catch (BrokenCircuitException)
        {
            // FAIL-SAFE: Si el circuito está abierto, asumimos que SÍ hay Legal Hold
            _logger.LogWarning("Circuit breaker open. Assuming Legal Hold ACTIVE for {IdentityId} to prevent compliance violation.", identityId);
            return true; 
        }
    }
}
```

### 4.2. Resiliencia en Batch Purge Jobs

Para evitar que un job de 1000 identidades falle completamente por un error transitorio en la identidad #500, implementamos **Checkpointing y Atomicidad por Lote**:

```csharp
public class BatchPurgeWorker
{
    public async Task ProcessBatchAsync(BatchPurgeJob job, CancellationToken ct)
    {
        var batchSize = 100;
        var chunks = job.IdentityIds.Chunk(batchSize);

        foreach (var chunk in chunks)
        {
            if (ct.IsCancellationRequested) break;

            // Actualizar estado del job en DB (Checkpoint)
            job.Status = BatchPurgeJobStatus.Processing;
            job.LastProcessedIndex = chunk.First();
            await _jobRepository.UpdateProgressAsync(job);

            foreach (var identityId in chunk)
            {
                try
                {
                    await _purgeHandler.Handle(new PurgeCommand { IdentityId = identityId, ... }, ct);
                    job.RecordSuccess();
                }
                catch (LegalHoldActiveException)
                {
                    job.RecordSkip("legal_hold");
                }
                catch (GracePeriodNotExpiredException)
                {
                    job.RecordSkip("grace_period");
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Failed to purge {IdentityId}", identityId);
                    job.RecordFailure();
                    // No rompemos el loop, continuamos con la siguiente identidad
                }
            }
        }

        job.Complete();
        await _jobRepository.SaveFinalStateAsync(job);
    }
}
```

---

## 5. Estrategia de Testing

### 5.1. Unit Tests (Dominio y Compliance)

```csharp
[Test]
public void Identity_Purge_WhenActive_ThrowsInvalidTransition()
{
    var identity = new Identity(/* Status = ACTIVE */);
    var act = () => identity.Purge(new PurgeReason("Test"), new Actor(Guid.NewGuid(), ActorType.Admin));
    act.Should().Throw<InvalidTransitionException>();
}

[Test]
public async Task PurgeService_WhenLegalHoldActive_ThrowsException()
{
    var mockLegalHold = new Mock<ILegalHoldChecker>();
    mockLegalHold.Setup(l => l.HasActiveHoldAsync(It.IsAny<IdentityId>())).ReturnsAsync(true);
    
    var service = new IdentityPurgeService(mockLegalHold.Object, ...);
    
    var act = () => service.ExecutePurgeAsync(identity, reason, actor, config);
    await act.Should().ThrowAsync<LegalHoldActiveException>();
}
```

### 5.2. Integration Tests (Verificación de Crypto-Shredding)

Es crucial verificar que los datos PII originales **no existen** en la base de datos después de la purga.

```csharp
[Test]
public async Task PurgeHandler_WhenExecuted_PII_IsCryptographicallyShredded()
{
    // Arrange: Crear identidad con PII conocido
    var identityId = await CreateTestIdentityAsync(email: "juan.veterinario@khcs.com");
    var command = new PurgeCommand { IdentityId = identityId, Reason = "Test", ConfirmIrreversible = true };

    // Act: Ejecutar purga
    await _handler.Handle(command, CancellationToken.None);

    // Assert 1: Verificar que la DB tiene hashes, no texto plano
    var dbIdentity = await _dbContext.Identities.FindAsync(identityId);
    dbIdentity.Status.Should().Be(IdentityStatus.PURGED);
    dbIdentity.Profile.Email.Should().NotBe("juan.veterinario@khcs.com");
    dbIdentity.Profile.Email.Should().Be(_hmacService.ComputeHash("juan.veterinario@khcs.com"));

    // Assert 2: Verificar que no hay texto plano en Audit Logs
    var auditLogs = await _dbContext.AuditLogs.Where(l => l.IdentityId == identityId).ToListAsync();
    auditLogs.Should().NotContain(l => l.SnapshotJson.Contains("juan.veterinario@khcs.com"));
}
```

### 5.3. Load Testing (Batch Throughput)

**Objetivo:** Validar que el sistema puede procesar 1000 identidades en menos de 2 minutos.

```javascript
// Script k6 simplificado para Batch Purge
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  scenarios: {
    batch_purge: {
      executor: 'per-vu-iterations',
      vus: 1,
      iterations: 1,
    },
  },
};

export default function () {
  const payload = JSON.stringify({
    identity_ids: __ENV.IDENTITY_POOL_IDS, // Array de 1000 IDs
    reason: "Load test batch purge",
    confirm_irreversible: true
  });

  const res = http.post(`${__ENV.BASE_URL}/v1/identities/purge-batch`, payload, {
    headers: { 'Authorization': `Bearer ${__ENV.JWT_TOKEN}` }
  });

  check(res, {
    'batch accepted (202)': (r) => r.status === 202,
  });

  const jobId = res.json('job_id');
  
  // Polling del estado del job (simplificado)
  // En un test real, usaríamos un loop con sleep hasta que status == COMPLETED
}
```

---

## 6. Deployment y Operaciones

### 6.1. Feature Flags para Rollout y Emergencias

| Feature Flag | Descripción | Valor por Defecto |
| :--- | :--- | :--- |
| `kagami.identity.purge.crypto_shredding_enabled` | Activa el reemplazo de PII por hashes HMAC. Si es `false`, solo cambia el estado (útil para debugging en Dev). | `true` |
| `kagami.identity.purge.grace_period_bypass` | Permite a Super-Admins purgar identidades sin esperar el Grace Period (para emergencias legales). | `false` |
| `kagami.identity.purge.batch_processing_enabled` | Activa el endpoint `/purge-batch`. | `true` |
| `kagami.identity.purge.legal_hold_fail_safe` | Si es `true`, bloquea purgas si el Legal Hold Checker falla. | `true` (Nunca desactivar en Prod) |

### 6.2. Runbook de Operaciones

**Escenario: Job de Purga Masiva Atascado en `PROCESSING`**

1. **Paso 1:** Verificar métricas `kagami.identity.batch_purge_jobs_total` y logs del worker.
2. **Paso 2:** Revisar si hay deadlocks en PostgreSQL (`pg_stat_activity`).
3. **Paso 3:** Si el worker murió, reiniciar el pod. El job retomará desde el último `LastProcessedIndex` (Checkpoint).
4. **Paso 4:** Si el job está corrupto, marcarlo como `FAILED` manualmente en DB y notificar al Admin para que reenvíe el batch.

**Escenario: Falso Positivo en Legal Hold (Bloqueo incorrecto)**

1. **Paso 1:** El oficial de cumplimiento verifica el caso en el sistema de Legal Hold.
2. **Paso 2:** Levantar el Legal Hold (`LIFTED` state).
3. **Paso 3:** El Admin reintentará la purga. El sistema permitirá la operación si el Grace Period ya expiró.

---

## 7. Compliance y Auditoría

### 7.1. GDPR - Art. 17 (Right to Erasure)

*   **Requisito:** El usuario tiene derecho a obtener la eliminación de sus datos personales sin dilación indebida.
*   **Implementación UC-05:** El endpoint `POST /purge` ejecuta el crypto-shredding. El `PurgeReason` permite vincular la operación con el `gdpr_request_id` del ticket legal, proporcionando evidencia auditable de cumplimiento.

### 7.2. NIST SP 800-88 (Media Sanitization)

*   **Requisito:** Define niveles de sanitización (Clear, Purge, Destroy).
*   **Implementación UC-05:** El uso de HMAC-SHA256 con claves gestionadas por KMS equivale al nivel "Purge" (los datos no son recuperables mediante técnicas de laboratorio estándar). La rotación de la clave maestra del tenant equivaldría a "Destroy".

### 7.3. SOC 2 - CC7.4 (System Operations - Data Disposal)

*   **Requisito:** La organización debe eliminar los datos de acuerdo con las políticas de retención.
*   **Implementación UC-05:** El `GracePeriodEvaluator` asegura que los datos no se eliminen antes del período mínimo de retención contractual/legal. Los logs de auditoría anonimizados preservan la evidencia de la operación sin violar la privacidad.

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los requisitos de irreversibilidad, legal hold y batch processing que estos artefactos implementan. |
| `08_Final_Class_Design.md` | Las clases `PurgeHandler`, `IdentityPurgeService`, y `ResilientLegalHoldChecker` se implementan según los contratos y métricas definidos aquí. |
| `10_UC_Specific_Data_Model.md` | Define la estructura física de las tablas `identities` (hashes), `legal_holds`, y `purge_jobs`. |
| `11_UC_Specific_Security_Threats.md` | Las mitigaciones de "Purga no autorizada" y "Evasión de Legal Hold" se validan mediante los patrones definidos aquí. |
| `12_UC_Specific_Performance_Benchmarks.md` | Los SLAs de latencia y throughput batch se validan mediante las métricas y load testing definidos aquí. |

---

## 9. Cierre del Paquete de Diseño Funcional (UC-05)

Con este documento, completamos los **9 archivos del Paquete de Diseño Profundo Funcional (Tier 1)** para el UC-05:

1. ✅ `01_Requirements_Specification.md` - Especificación completa con escenarios BDD.
2. ✅ `02_Use_Case_Diagram.md` - Diagrama de caso de uso UML.
3. ✅ `03_Activity_Diagram.md` - Diagrama de actividades con validaciones de compliance y anonimización.
4. ✅ `04_Initial_Domain_Model.md` - Modelo de dominio conceptual.
5. ✅ `05_State_Machine_Diagram.md` - Máquinas de estados (Transición Terminal, Legal Hold, Batch).
6. ✅ `06_Robustness_Diagram.md` - Diagrama de robustez ICONIX.
7. ✅ `07_Sequence_Diagram.md` - Diagrama de secuencia con flujos alternativos.
8. ✅ `08_Final_Class_Design.md` - Diseño de clases con firmas y patrones.
9. ✅ `09_Modern_Artifacts.md` - Contratos OpenAPI, métricas, resiliencia, compliance.

### 9.1. Resumen de lo Logrado

Hemos diseñado el endpoint de purga más robusto, seguro y compliant del mercado IAM:
- **Cumplimiento Normativo:** GDPR Art. 17, NIST SP 800-88, SOC 2 CC7.4 integrados nativamente en el flujo.
- **Supremacía Legal:** Sistema de Legal Hold con patrón Fail-Safe (Circuit Breaker) que previene violaciones legales catastróficas.
- **Crypto-Shredding:** Anonimización criptográfica irreversible (HMAC-SHA256) de PII y Audit Logs.
- **Resiliencia Operativa:** Procesamiento batch con checkpointing, atomicidad por lote y manejo de fallos parciales.
- **Seguridad UX:** Confirmaciones explícitas (`confirm_irreversible`) y justificaciones auditables para acciones destructivas.

### 9.2. Próximos Pasos (Documentos Técnicos Específicos)

Según la **Opción A+** acordada, cada UC tiene **12 documentos** (9 funcionales + 3 técnicos específicos). Para el UC-05, aún faltan:

1. **`10_UC_Specific_Data_Model.md`** - Modelo de datos físico específico (estrategia de hashes, tabla `legal_holds`, tabla `purge_jobs`).
2. **`11_UC_Specific_Security_Threats.md`** - Análisis de amenazas específico (Purga prematura, Evasión de Legal Hold, Fuga de datos post-purga).
3. **`12_UC_Specific_Performance_Benchmarks.md`** - Benchmarks y SLAs específicos para latencia de crypto-shredding y throughput batch.

---
*Fin del Documento 09_Modern_Artifacts para UC-05*