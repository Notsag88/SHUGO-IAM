# UC-10: Import/Export Masivo - Artefactos Modernos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-10 - Import/Export Masivo (Bulk Identity Operations) |
| **Documento:** | 09_Modern_Artifacts.md |
| **Metodología:** | API-First + Observability + Resilience Patterns + DevSecOps + Async Processing |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 14/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 14/09/2026 | Gastón Ruiz | **Creación inicial.** Definición de los artefactos modernos para UC-10: contratos OpenAPI 3.1 con endpoints de importación, exportación, consulta de progreso y descarga de reportes, catálogo de métricas OpenTelemetry (enfocado en throughput de procesamiento, latencia por chunk, profundidad de cola, y tasa de éxito), patrones de resiliencia (Circuit Breaker para Storage y Queue, Retry policies con backoff exponencial), estrategias de testing para escenarios de procesamiento masivo, y consideraciones de deployment para operaciones asíncronas de larga duración. | Cierra el paquete de diseño Tier 1 funcional del UC-10, proporcionando todos los artefactos necesarios para la implementación, operación, monitoreo y cumplimiento normativo (GDPR Art. 5, SOC 2) del endpoint de importación/exportación masiva del motor KAGAMI. |

---

## 1. Introducción y Propósito

Este documento define los **Artefactos Modernos** necesarios para operar el UC-10 (Import/Export Masivo) en un entorno de producción enterprise. A diferencia de los UCs transaccionales individuales, el UC-10 introduce desafíos únicos debido a su naturaleza de **procesamiento masivo asíncrono de larga duración**:

1. **Arquitectura Asíncrona:** La separación entre iniciación síncrona (<2s) y procesamiento asíncrono (minutos a horas) requiere patrones de resiliencia específicos para colas de mensajes y workers en background.
2. **Dependencia de Infraestructura Externa:** Storage (S3/Azure Blob) y Job Queue (RabbitMQ/SQS) son puntos críticos de falla que requieren Circuit Breakers y Retry policies.
3. **Procesamiento de Larga Duración:** Los jobs pueden durar horas, requiriendo monitoreo de progreso, manejo de jobs huérfanos, y mecanismos de recuperación ante fallos.
4. **Consumo Intensivo de Recursos:** Las operaciones masivas consumen CPU (validación, parsing), memoria (chunks), I/O de DB (consultas de validación, writes de identidades), y ancho de banda (descarga/subida de archivos).
5. **Seguridad de Datos Sensibles:** Los archivos de importación/exportación contienen PII (emails, nombres, roles) que deben protegerse en tránsito y en reposo.

---

## 2. Contratos OpenAPI Completos

### 2.1. Especificación OpenAPI 3.1

```yaml
openapi: 3.1.0
info:
  title: SHUGO KAGAMI - Bulk Operations API
  description: |
    API de operaciones masivas de importación y exportación de identidades del motor KAGAMI.
    
    ## Características Clave
    - Procesamiento asíncrono con jobs en background.
    - Validación en tres fases (Sintáctica, Semántica, Consistencia).
    - Estrategias transaccionales flexibles (COMMIT_ALL, COMMIT_VALID).
    - Modo Dry Run para pre-validación sin cambios persistentes.
    - Reportes granulares con resultado por registro individual.
    - URLs firmadas con TTL para descarga segura de exportaciones.
    
    ## Seguridad
    - Endpoints autenticados (JWT requerido).
    - Permisos específicos: identity:import, identity:export.
    - Quota mensual configurable por tenant.
    - Máximo 1 job activo por tenant.
    - URLs firmadas con TTL de 24h para descargas.
  version: 1.0.0

servers:
  - url: https://api.shugo.io/v1
    description: Producción

paths:
  /identities/import:
    post:
      operationId: initiateImport
      summary: Iniciar importación masiva de identidades
      description: |
        Sube un archivo (CSV/JSON/XML) con datos de identidades para importar.
        El procesamiento es asíncrono. Retorna un job_id para consultar el progreso.
      tags:
        - Bulk Operations
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          multipart/form-data:
            schema:
              $ref: '#/components/schemas/ImportRequest'
      responses:
        '202':
          description: Importación iniciada, job encolado
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/JobInitiatedResponse'
        '400':
          description: Archivo inválido (formato, tamaño, headers)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Sin permiso identity:import
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '409':
          description: Ya hay un job activo en el tenant
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '413':
          description: Archivo excede tamaño máximo (50MB)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '429':
          description: Quota mensual de importación excedida
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

  /identities/export:
    post:
      operationId: initiateExport
      summary: Iniciar exportación masiva de identidades
      description: |
        Solicita la exportación de identidades según filtros.
        El procesamiento es asíncrono. Retorna un job_id para consultar el progreso.
      tags:
        - Bulk Operations
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/ExportRequest'
      responses:
        '202':
          description: Exportación iniciada, job encolado
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/JobInitiatedResponse'
        '400':
          description: Filtros o formato inválido
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Sin permiso identity:export
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '409':
          description: Ya hay un job activo en el tenant
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

  /jobs/{job_id}:
    get:
      operationId: getJobStatus
      summary: Consultar estado y progreso de un job
      description: |
        Retorna el estado actual, progreso y métricas de un job de importación o exportación.
      tags:
        - Bulk Operations
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
          description: Estado del job
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/JobStatusResponse'
        '404':
          description: Job no encontrado
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

  /jobs/{job_id}/report:
    get:
      operationId: downloadJobReport
      summary: Descargar reporte de resultados de un job
      description: |
        Genera una URL firmada para descargar el reporte CSV con el resultado de cada registro.
        Solo disponible para jobs en estado COMPLETED o COMPLETED_WITH_ERRORS.
      tags:
        - Bulk Operations
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
        '302':
          description: Redirección a URL firmada de descarga
          headers:
            Location:
              description: URL firmada de descarga (TTL 24h)
              schema:
                type: string
                format: uri
        '400':
          description: Job no tiene reporte (estado FAILED o PENDING)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Job no encontrado
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
    ImportRequest:
      type: object
      required:
        - file
      properties:
        file:
          type: string
          format: binary
          description: "Archivo CSV/JSON/XML con datos de identidades (máx. 50MB)"
        commit_strategy:
          type: string
          enum: [COMMIT_ALL, COMMIT_VALID]
          default: COMMIT_VALID
          description: "Estrategia de transaccionalidad"
        is_dry_run:
          type: boolean
          default: false
          description: "Modo de pre-validación sin cambios persistentes"

    ExportRequest:
      type: object
      required:
        - format
        - fields
      properties:
        filters:
          type: object
          properties:
            roles:
              type: array
              items:
                type: string
            status:
              type: string
              enum: [ACTIVE, DEACTIVATED, PROVISIONED]
            created_after:
              type: string
              format: date-time
            created_before:
              type: string
              format: date-time
        format:
          type: string
          enum: [CSV, JSON]
          description: "Formato del archivo de exportación"
        fields:
          type: array
          items:
            type: string
            enum: [email, display_name, roles, status, created_at, last_login_at, email_verified]
          minItems: 1
          description: "Campos a incluir en la exportación"

    JobInitiatedResponse:
      type: object
      properties:
        job_id:
          type: string
          format: uuid
        status:
          type: string
          enum: [PENDING]
        message:
          type: string
          example: "Bulk import job initiated. You will receive a notification upon completion."
        estimated_duration_seconds:
          type: integer
          example: 900

    JobStatusResponse:
      type: object
      properties:
        job_id:
          type: string
          format: uuid
        job_type:
          type: string
          enum: [IMPORT, EXPORT]
        status:
          type: string
          enum: [PENDING, PROCESSING, COMPLETED, COMPLETED_WITH_ERRORS, FAILED]
        progress:
          type: object
          properties:
            total_records:
              type: integer
            processed_records:
              type: integer
            successful_records:
              type: integer
            failed_records:
              type: integer
            percentage:
              type: number
              format: double
        started_at:
          type: string
          format: date-time
        completed_at:
          type: string
          format: date-time
        estimated_completion_at:
          type: string
          format: date-time
        report_url:
          type: string
          format: uri
          description: "URL firmada de descarga del reporte (solo para jobs completados)"
        error_message:
          type: string
          description: "Mensaje de error (solo para jobs FAILED)"

    ErrorResponse:
      type: object
      properties:
        error:
          type: object
          properties:
            code:
              type: string
              enum:
                - INVALID_FILE_FORMAT
                - FILE_TOO_LARGE
                - INSUFFICIENT_PERMISSIONS
                - BULK_OPERATION_IN_PROGRESS
                - QUOTA_EXCEEDED
                - JOB_NOT_FOUND
                - REPORT_NOT_AVAILABLE
                - INVALID_EXPORT_FILTERS
            message:
              type: string
            details:
              type: object
```

### 2.2. Códigos de Error Específicos de UC-10

| Código HTTP | Código de Error | Descripción | Cuándo Ocurre |
| :--- | :--- | :--- | :--- |
| **400** | `INVALID_FILE_FORMAT` | Archivo no cumple formato esperado (extensión, headers, encoding). | `FileValidator.ValidateAsync()` falla. |
| **400** | `INVALID_EXPORT_FILTERS` | Filtros de exportación inválidos o campos no soportados. | Validación de `ExportRequest` falla. |
| **400** | `REPORT_NOT_AVAILABLE` | El job no tiene reporte disponible (estado PENDING, PROCESSING o FAILED). | `GET /jobs/{id}/report` para job sin reporte. |
| **403** | `INSUFFICIENT_PERMISSIONS` | El usuario no tiene permiso `identity:import` o `identity:export`. | API Gateway valida claims del JWT. |
| **404** | `JOB_NOT_FOUND` | El job no existe o no pertenece al tenant del usuario. | `JobRepository.GetByIdAsync()` retorna null. |
| **409** | `BULK_OPERATION_IN_PROGRESS` | Ya hay un job activo (PENDING o PROCESSING) en el tenant. | Validación de concurrencia en `BulkImportHandler`. |
| **413** | `FILE_TOO_LARGE` | El archivo excede el tamaño máximo de 50MB. | Validación de tamaño en `FileValidator`. |
| **429** | `QUOTA_EXCEEDED` | Quota mensual de importación excedida. | `QuotaService.ValidateQuotaAsync()` falla. |

---

## 3. Catálogo de Métricas OpenTelemetry

### 3.1. Métricas de Negocio y Throughput

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.bulk.import_initiated_total` | Counter | jobs | Total de jobs de importación iniciados | `tenant_id`, `file_format`, `commit_strategy`, `is_dry_run` |
| `kagami.bulk.import_completed_total` | Counter | jobs | Total de jobs de importación completados | `tenant_id`, `status` (COMPLETED, COMPLETED_WITH_ERRORS, FAILED) |
| `kagami.bulk.export_initiated_total` | Counter | jobs | Total de jobs de exportación iniciados | `tenant_id`, `format` |
| `kagami.bulk.export_completed_total` | Counter | jobs | Total de jobs de exportación completados | `tenant_id`, `status` |
| `kagami.bulk.records_processed_total` | Counter | records | Total de registros procesados | `tenant_id`, `job_type`, `result` (success, failed) |
| `kagami.bulk.success_rate` | Gauge | ratio | Tasa de éxito de registros (exitosos / total) | `tenant_id`, `job_type` |

### 3.2. Métricas de Rendimiento y Procesamiento

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.bulk.initiation_latency` | Histogram | ms | Latencia de iniciación síncrona (HTTP request) | `tenant_id`, `job_type`, `success` |
| `kagami.bulk.job_processing_duration` | Histogram | s | Duración total del procesamiento asíncrono | `tenant_id`, `job_type`, `status`, `total_records` |
| `kagami.bulk.chunk_processing_latency` | Histogram | ms | Latencia de procesamiento por chunk (1000 registros) | `tenant_id`, `job_type` |
| `kagami.bulk.validation_phase_latency` | Histogram | ms | Latencia por fase de validación | `tenant_id`, `phase` (syntactic, semantic, consistency) |
| `kagami.bulk.file_upload_latency` | Histogram | ms | Latencia de subida de archivo a Storage | `tenant_id`, `file_size_mb` |
| `kagami.bulk.file_download_latency` | Histogram | ms | Latencia de descarga de archivo desde Storage | `tenant_id`, `file_size_mb` |
| `kagami.bulk.report_generation_latency` | Histogram | ms | Latencia de generación de reporte | `tenant_id`, `job_type`, `total_records` |
| `kagami.bulk.db_query_latency` | Histogram | ms | Latencia de consultas a DB durante validación | `tenant_id`, `query_type` (email_exists, roles, manager) |

### 3.3. Métricas de Infraestructura y Resiliencia

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.bulk.queue_depth` | Gauge | messages | Profundidad actual de la cola de jobs | `queue_name` |
| `kagami.bulk.queue_enqueue_latency` | Histogram | ms | Latencia de encolado de mensajes | `queue_name` |
| `kagami.bulk.queue_dequeue_latency` | Histogram | ms | Latencia de desencolado de mensajes | `queue_name` |
| `kagami.bulk.storage_errors_total` | Counter | errors | Total de errores en operaciones de Storage | `tenant_id`, `operation` (upload, download), `error_type` |
| `kagami.bulk.queue_errors_total` | Counter | errors | Total de errores en operaciones de Queue | `queue_name`, `operation` (enqueue, dequeue), `error_type` |
| `kagami.bulk.storage_circuit_breaker_state` | Gauge | state | Estado del Circuit Breaker de Storage (0=Closed, 1=Open, 2=HalfOpen) | `service` |
| `kagami.bulk.orphaned_jobs_detected_total` | Counter | jobs | Total de jobs huérfanos detectados por el cleanup job | `tenant_id` |
| `kagami.bulk.quota_consumed_total` | Counter | records | Total de registros consumidos de quota | `tenant_id` |

### 3.4. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Queue Depth High** | `queue_depth > 50` por 10 min | Alta | Cola saturada. Escalar workers de procesamiento. |
| **Storage Circuit Breaker Open** | `storage_circuit_breaker_state == 1` | **Crítica** | S3/Azure Blob caído. Las importaciones/exportaciones no pueden procesarse. |
| **Job Processing Stuck** | `job_processing_duration > 2h` para un solo job | Alta | Posible job huérfano. Investigar y considerar cancelación manual. |
| **High Failure Rate** | `success_rate < 0.50` para un tenant en 1h | Media | Muchos registros fallando. Revisar calidad de datos del archivo. |
| **Validation Phase Slow** | `validation_phase_latency_p95 > 500ms` para fase semantic | Media | Consultas de DB lentas. Revisar índices y carga de DB. |
| **Orphaned Jobs Detected** | `orphaned_jobs_detected_total > 0` | Alta | Workers fallando. Revisar logs de workers y reiniciar si es necesario. |
| **Quota Near Limit** | `quota_consumed_total > 0.80 * quota_limit` | Media | Tenant acercándose al límite de quota. Notificar al administrador. |
| **Storage Errors Spike** | `storage_errors_total > 10` en 5 min | **Crítica** | Problemas con S3/Azure Blob. Revisar conectividad y credenciales. |

---

## 4. Patrones de Resiliencia

### 4.1. Circuit Breaker para Storage Service (S3/Azure Blob)

Dado que Storage es un punto crítico de falla para importaciones y exportaciones, implementamos Circuit Breaker.

**Implementación (Polly - C#):**

```csharp
public class ResilientStorageService : IStorageService
{
    private readonly IStorageService _inner;
    private readonly AsyncCircuitBreakerPolicy _circuitBreaker;
    private readonly ILogger<ResilientStorageService> _logger;

    public ResilientStorageService(IStorageService inner, ILogger<ResilientStorageService> logger)
    {
        _inner = inner;
        _logger = logger;
        
        _circuitBreaker = Policy
            .Handle<StorageException>()
            .Or<TimeoutException>()
            .Or<HttpRequestException>()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 3,
                durationOfBreak: TimeSpan.FromMinutes(5),
                onBreak: (ex, duration) => 
                {
                    _logger.LogWarning("Storage Circuit Breaker OPEN for {Duration}. Bulk operations will fail.", duration);
                    Metrics.StorageCircuitBreakerState.Set(1);
                },
                onReset: () => 
                {
                    _logger.LogInformation("Storage Circuit Breaker CLOSED. Bulk operations resumed.");
                    Metrics.StorageCircuitBreakerState.Set(0);
                },
                onHalfOpen: () => 
                {
                    _logger.LogInformation("Storage Circuit Breaker HALF-OPEN. Testing connection.");
                    Metrics.StorageCircuitBreakerState.Set(2);
                }
            );
    }

    public async Task<string> UploadAsync(Stream stream, string key, CancellationToken ct)
    {
        try
        {
            return await _circuitBreaker.ExecuteAsync(() => 
                _inner.UploadAsync(stream, key, ct));
        }
        catch (BrokenCircuitException)
        {
            _logger.LogWarning("Circuit breaker open. Cannot upload to storage.");
            throw new StorageUnavailableException("Storage service unavailable. Please try again later.");
        }
    }

    public async Task<Stream> DownloadAsync(string url, CancellationToken ct)
    {
        try
        {
            return await _circuitBreaker.ExecuteAsync(() => 
                _inner.DownloadAsync(url, ct));
        }
        catch (BrokenCircuitException)
        {
            throw new StorageUnavailableException("Storage service unavailable. Please try again later.");
        }
    }
}
```

### 4.2. Retry Policy con Backoff Exponencial para Storage

Para fallos transitorios en S3/Azure Blob, implementamos retry con backoff exponencial.

**Implementación (Polly - C#):**

```csharp
public class ResilientStorageService : IStorageService
{
    private readonly IAsyncPolicy<string> _uploadRetryPolicy;

    public ResilientStorageService(IStorageService inner, ILogger<ResilientStorageService> logger)
    {
        _uploadRetryPolicy = Policy<string>
            .Handle<StorageException>()
            .Or<TimeoutException>()
            .WaitAndRetryAsync(
                retryCount: 3,
                sleepDurationProvider: retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
                onRetry: (ex, timeSpan, retryCount, context) =>
                {
                    logger.LogWarning(ex, "Retry attempt {RetryCount} for storage upload after {Delay}s", 
                        retryCount, timeSpan.TotalSeconds);
                    Metrics.StorageRetryAttempts.Add(1, 
                        new KeyValuePair<string, object?>("retry_count", retryCount));
                }
            );
    }

    public async Task<string> UploadAsync(Stream stream, string key, CancellationToken ct)
    {
        // Reset stream position for retries
        if (stream.CanSeek) stream.Position = 0;
        
        return await _uploadRetryPolicy.ExecuteAsync(async () =>
        {
            if (stream.CanSeek) stream.Position = 0;
            return await _inner.UploadAsync(stream, key, ct);
        });
    }
}
```

### 4.3. Circuit Breaker para Job Queue (RabbitMQ/SQS)

```csharp
public class ResilientJobQueue : IJobQueue
{
    private readonly IJobQueue _inner;
    private readonly AsyncCircuitBreakerPolicy _circuitBreaker;

    public ResilientJobQueue(IJobQueue inner, ILogger<ResilientJobQueue> logger)
    {
        _inner = inner;
        
        _circuitBreaker = Policy
            .Handle<BrokerUnreachableException>()
            .Or<TimeoutException>()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 3,
                durationOfBreak: TimeSpan.FromMinutes(2),
                onBreak: (ex, duration) => 
                    logger.LogWarning("Job Queue Circuit Breaker OPEN for {Duration}", duration)
            );
    }

    public async Task EnqueueAsync(BulkImportMessage message, CancellationToken ct)
    {
        await _circuitBreaker.ExecuteAsync(() => _inner.EnqueueAsync(message, ct));
    }
}
```

### 4.4. Timeout Estricto para Operaciones de Storage

```csharp
// Configuración de timeouts para Storage
services.AddHttpClient<IStorageService, S3StorageService>(client =>
{
    client.Timeout = TimeSpan.FromMinutes(5); // Timeout para archivos grandes (hasta 50MB)
});

// Configuración de Polly para timeout
services.AddPolicyHandler(Policy.TimeoutAsync<HttpResponseMessage>(TimeSpan.FromMinutes(5)));
```

### 4.5. Orphaned Job Cleanup (Background Service)

```csharp
public class OrphanedJobCleanupService : BackgroundService
{
    private readonly IJobRepository _jobRepo;
    private readonly INotificationService _notificationService;
    private readonly ILogger<OrphanedJobCleanupService> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await Task.Delay(TimeSpan.FromHours(1), stoppingToken);

                // Jobs PENDING > 24h o PROCESSING > 2h sin progreso
                var orphanedJobs = await _jobRepo.GetOrphanedJobsAsync(
                    pendingThreshold: TimeSpan.FromHours(24),
                    processingThreshold: TimeSpan.FromHours(2),
                    stoppingToken);

                foreach (var job in orphanedJobs)
                {
                    _logger.LogWarning("Marking orphaned job {JobId} as FAILED", job.Id);
                    job.MarkAsFailed("Job timeout: no progress for too long");
                    await _jobRepo.SaveAsync(job, stoppingToken);
                    
                    Metrics.OrphanedJobsDetectedTotal.Add(1);

                    await _notificationService.SendFailureNotificationAsync(
                        job.InitiatedBy, job.Id, "Job timeout", stoppingToken);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in OrphanedJobCleanupService");
            }
        }
    }
}
```

---

## 5. Estrategia de Testing

### 5.1. Unit Tests (Dominio y Validaciones)

```csharp
[Test]
public void BulkImportJob_Initiate_WithValidData_CreatesInPendingStatus()
{
    var job = BulkImportJob.Initiate(
        new TenantId(Guid.NewGuid()),
        new IdentityId(Guid.NewGuid()),
        "identities.csv",
        "https://storage.shugo.io/imports/job-123/identities.csv",
        2048576,
        500,
        new ImportOptions(CommitStrategy.COMMIT_VALID, false, FileFormat.CSV));

    job.Status.Should().Be(JobStatus.PENDING);
    job.TotalRecords.Should().Be(500);
    job.GetDomainEvents().Should().HaveCount(1);
}

[Test]
public void BulkImportJob_MarkAsCompletedWithErrors_WithCommitAll_ThrowsDomainException()
{
    var job = BulkImportJob.Initiate(
        new TenantId(Guid.NewGuid()),
        new IdentityId(Guid.NewGuid()),
        "test.csv",
        "https://storage.shugo.io/test.csv",
        1024, 100,
        new ImportOptions(CommitStrategy.COMMIT_ALL, false, FileFormat.CSV));

    job.StartProcessing();
    job.UpdateProgress(100, 95, 5);

    var act = () => job.MarkAsCompletedWithErrors("https://report.csv");
    act.Should().Throw<DomainException>()
       .WithMessage("*cannot mark as completed with errors when strategy is COMMIT_ALL*");
}

[Test]
public async Task RecordValidator_WithInvalidEmail_ThrowsSyntacticError()
{
    var validator = new RecordValidator(
        Mock.Of<IIdentityRepository>(),
        Mock.Of<IRoleRepository>(),
        Mock.Of<ITenantConfigRepository>());

    var record = new Record { Email = "invalid-email", DisplayName = "Test" };

    var act = () => validator.ValidateAsync(record, 1, new TenantId(Guid.NewGuid()), CancellationToken.None);
    await act.Should().ThrowAsync<ValidationException>()
       .Where(e => e.Phase == ValidationPhase.SYNTACTIC);
}
```

### 5.2. Integration Tests (Procesamiento Asíncrono)

```csharp
[Test]
public async Task BulkImportProcessor_WithCommitAllAndError_RollsBackAndFails()
{
    // Arrange
    var mockValidator = new Mock<IRecordValidator>();
    mockValidator.Setup(v => v.ValidateAsync(It.IsAny<Record>(), It.IsAny<int>(), It.IsAny<TenantId>(), It.IsAny<CancellationToken>()))
                 .ThrowsAsync(new ValidationException("INVALID_EMAIL", "Invalid", ValidationPhase.SYNTACTIC));

    var mockTransaction = new Mock<IDbTransaction>();
    var mockUnitOfWork = new Mock<IUnitOfWork>();
    mockUnitOfWork.Setup(u => u.BeginTransactionAsync(It.IsAny<CancellationToken>()))
                  .ReturnsAsync(mockTransaction.Object);

    var processor = new BulkImportProcessor(
        Mock.Of<IJobRepository>(),
        Mock.Of<IStorageService>(),
        mockValidator.Object,
        Mock.Of<IIdentityProvisioningService>(),
        Mock.Of<IIdentityUpdateService>(),
        Mock.Of<IReportGenerator>(),
        Mock.Of<INotificationService>(),
        mockUnitOfWork.Object,
        Mock.Of<IQuotaService>(),
        Mock.Of<IEventPublisher>(),
        Mock.Of<ILogger<BulkImportProcessor>>());

    var job = BulkImportJob.Initiate(
        new TenantId(Guid.NewGuid()),
        new IdentityId(Guid.NewGuid()),
        "test.csv", "https://storage/test.csv",
        1024, 100,
        new ImportOptions(CommitStrategy.COMMIT_ALL, false, FileFormat.CSV));

    var chunk = new List<Record> { new Record { Email = "invalid" } };

    // Act
    var result = await processor.ProcessChunkAsync(chunk, job, CancellationToken.None);

    // Assert
    result.HasErrors.Should().BeTrue();
    mockTransaction.Verify(t => t.RollbackAsync(It.IsAny<CancellationToken>()), Times.Once);
}

[Test]
public async Task BulkImportProcessor_WithDryRun_DoesNotInvokeProvisioning()
{
    // Arrange
    var mockProvisioning = new Mock<IIdentityProvisioningService>();
    var mockValidator = new Mock<IRecordValidator>();
    // Validator succeeds

    var processor = new BulkImportProcessor(/* ... with mocks ... */);

    var job = BulkImportJob.Initiate(/* ... with IsDryRun = true ... */);
    var chunk = new List<Record> { new Record { Email = "valid@khcs.com" } };

    // Act
    await processor.ProcessChunkAsync(chunk, job, CancellationToken.None);

    // Assert
    mockProvisioning.Verify(p => p.ProvisionAsync(It.IsAny<Record>(), It.IsAny<CancellationToken>()), Times.Never);
}
```

### 5.3. Load Testing (Importación Masiva)

**Objetivo:** Validar que el sistema maneja la importación de 50.000 registros sin degradar el rendimiento.

**Script k6 (Extracto):**

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Trend, Rate } from 'k6/metrics';

const initiationLatency = new Trend('initiation_latency', true);
const errorRate = new Rate('errors');

export const options = {
  scenarios: {
    bulk_import: {
      executor: 'per-vu-iterations',
      vus: 5,
      iterations: 1,
      maxDuration: '5m',
    },
  },
  thresholds: {
    initiation_latency: ['p(95)<2000'], // Iniciación < 2s
    errors: ['rate<0.05'],
  },
};

export default function () {
  // Generar CSV con 50.000 registros
  const csvContent = generateLargeCSV(50000);
  
  const formData = {
    file: http.file(csvContent, 'identities.csv', 'text/csv'),
    commit_strategy: 'COMMIT_VALID',
    is_dry_run: 'false'
  };

  const res = http.post(`${__ENV.BASE_URL}/v1/identities/import`, formData, {
    headers: { 'Authorization': `Bearer ${__ENV.ADMIN_TOKEN}` },
  });

  initiationLatency.add(res.timings.duration);

  check(res, {
    'import initiated (202)': (r) => r.status === 202,
    'has job_id': (r) => r.json('job_id') !== undefined,
  });

  if (res.status !== 202) errorRate.add(1);

  // Poll job status until completion
  if (res.status === 202) {
    const jobId = res.json('job_id');
    pollJobStatus(jobId);
  }
}

function pollJobStatus(jobId) {
  let completed = false;
  let attempts = 0;
  
  while (!completed && attempts < 300) { // Max 5 minutes
    sleep(1);
    attempts++;
    
    const statusRes = http.get(`${__ENV.BASE_URL}/v1/jobs/${jobId}`, {
      headers: { 'Authorization': `Bearer ${__ENV.ADMIN_TOKEN}` },
    });
    
    const status = statusRes.json('status');
    if (['COMPLETED', 'COMPLETED_WITH_ERRORS', 'FAILED'].includes(status)) {
      completed = true;
      check(statusRes, {
        'job completed successfully': (r) => r.json('status') === 'COMPLETED' || r.json('status') === 'COMPLETED_WITH_ERRORS',
      });
    }
  }
}
```

---

## 6. Deployment y Operaciones

### 6.1. Feature Flags para Rollout Gradual

| Feature Flag | Descripción | Valor por Defecto |
| :--- | :--- | :--- |
| `kagami.bulk.import_enabled` | Activa/desactiva importaciones masivas globalmente. | `true` |
| `kagami.bulk.export_enabled` | Activa/desactiva exportaciones masivas globalmente. | `true` |
| `kagami.bulk.dry_run_enabled` | Activa/desactiva modo Dry Run. | `true` |
| `kagami.bulk.commit_all_enabled` | Activa/desactiva estrategia COMMIT_ALL. | `true` |
| `kagami.bulk.max_file_size_mb` | Tamaño máximo de archivo en MB. | `50` |
| `kagami.bulk.chunk_size` | Tamaño de chunk para procesamiento. | `1000` |
| `kagami.bulk.max_concurrent_jobs` | Máximo de jobs activos por tenant. | `1` |
| `kagami.bulk.default_quota_monthly` | Quota mensual por defecto por tenant. | `10000` |
| `kagami.bulk.cleanup_enabled` | Activa el job de limpieza de jobs huérfanos. | `true` |

### 6.2. Configuración de Infraestructura

```yaml
# Configuración de Storage (S3)
storage:
  provider: "s3"
  bucket_name: "shugo-bulk-operations"
  region: "us-east-1"
  max_file_size_bytes: 52428800 # 50MB
  signed_url_ttl_hours: 24
  import_prefix: "imports/"
  export_prefix: "exports/"
  reports_prefix: "reports/"

# Configuración de Job Queue (RabbitMQ)
queue:
  provider: "rabbitmq"
  connection_string: "${RABBITMQ_CONNECTION_STRING}"
  import_queue_name: "bulk_import_jobs"
  export_queue_name: "bulk_export_jobs"
  prefetch_count: 1 # Un mensaje por worker a la vez
  retry_attempts: 3
  retry_delay_seconds: 30

# Configuración de Workers
workers:
  import:
    replicas: 3
    chunk_size: 1000
    max_processing_time_hours: 4
  export:
    replicas: 2
    batch_size: 1000
    max_processing_time_hours: 2
  cleanup:
    interval_hours: 1
    pending_threshold_hours: 24
    processing_threshold_hours: 2
```

### 6.3. Escalado de Workers

```yaml
# Kubernetes HPA para Import Workers
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kagami-import-workers-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kagami-import-workers
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: External
      external:
        metric:
          name: kagami_bulk_queue_depth
        target:
          type: AverageValue
          averageValue: "5" # Escalar si >5 jobs en cola por worker
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

### 6.4. Runbook de Operaciones

**Escenario: Cola de Jobs Saturada**

1. **Paso 1:** Verificar alerta `queue_depth > 50` por 10 min.
2. **Paso 2:** Revisar métricas de workers: CPU, memoria, latencia de procesamiento por chunk.
3. **Paso 3:** Si workers están al máximo de CPU, escalar horizontalmente (HPA debería hacerlo automáticamente).
4. **Paso 4:** Si workers están fallando, revisar logs para identificar errores (Storage caído, DB lenta).
5. **Paso 5:** Si Storage está caído, verificar Circuit Breaker y estado de S3/Azure Blob.
6. **Paso 6:** Monitorear `queue_depth` para confirmar que baja tras escalar.

**Escenario: Job Huérfano Detectado**

1. **Paso 1:** Verificar alerta `orphaned_jobs_detected_total > 0`.
2. **Paso 2:** Revisar logs del worker que procesaba el job para identificar causa de fallo.
3. **Paso 3:** Verificar si el worker está vivo o se reinició.
4. **Paso 4:** El cleanup job marca automáticamente el job como FAILED y notifica al administrador.
5. **Paso 5:** Si el administrador necesita reprocesar, puede re-subir el archivo.
6. **Paso 6:** Investigar causa raíz del fallo del worker (OOM, timeout, error de Storage).

**Escenario: Storage Circuit Breaker Abierto**

1. **Paso 1:** Verificar alerta `storage_circuit_breaker_state == 1`.
2. **Paso 2:** Revisar conectividad con S3/Azure Blob (ping, DNS, credenciales).
3. **Paso 3:** Verificar estado del servicio en AWS/Azure Status Page.
4. **Paso 4:** Las importaciones/exportaciones en progreso fallarán. Los nuevos jobs se encolan pero no se procesan.
5. **Paso 5:** Una vez recuperado, el Circuit Breaker transiciona a HALF-OPEN y luego CLOSED automáticamente.
6. **Paso 6:** Monitorear `storage_errors_total` y `storage_circuit_breaker_state` para confirmar recuperación.

---

## 7. Compliance y Auditoría

### 7.1. GDPR - Art. 5 (Principles relating to processing of personal data)

*   **Requisito:** Los datos personales deben ser procesados de forma lícita, leal y transparente, y recogidos con fines determinados, explícitos y legítimos (Data Minimization).
*   **Implementación UC-10:** La exportación permite especificar exactamente qué campos exportar (`fields`), aplicando el principio de minimización de datos. Los archivos exportados se sirven mediante URLs firmadas con TTL de 24h.

### 7.2. GDPR - Art. 32 (Security of Processing)

*   **Requisito:** Se deben implementar medidas técnicas para garantizar la seguridad del tratamiento.
*   **Implementación UC-10:** Archivos almacenados en S3 con encriptación en reposo (SSE-S3). URLs firmadas con TTL para descargas. Permisos específicos (`identity:import`, `identity:export`) para control de acceso.

### 7.3. SOC 2 - CC6.1 (Logical Access Controls)

*   **Requisito:** Restringir el acceso lógico a los datos según la autorización.
*   **Implementación UC-10:** Permisos granulares por operación (import vs export). Quota mensual configurable. Máximo 1 job activo por tenant.

### 7.4. SOC 2 - CC7.2 (System Monitoring)

*   **Requisito:** Monitorear el sistema para detectar anomalías.
*   **Implementación UC-10:** Métricas de OpenTelemetry para throughput, latencia, errores, profundidad de cola, y jobs huérfanos. Alertas automáticas para condiciones anómalas.

### 7.5. ISO 27001 - A.12.6 (Technical Vulnerability Management)

*   **Requisito:** Gestionar las vulnerabilidades técnicas de los sistemas de información.
*   **Implementación UC-10:** Validación en tres fases previene inyección de datos maliciosos. Circuit Breakers y Retry policies previenen cascadas de fallos. Cleanup de jobs huérfanos previene acumulación de estado inconsistente.

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los requisitos de procesamiento asíncrono, validación en 3 fases, y estrategias transaccionales que estos artefactos implementan. |
| `08_Final_Class_Design.md` | Las clases `BulkImportHandler`, `BulkImportProcessor`, `RecordValidator`, y `ReportGenerator` se implementan según los contratos y métricas definidos aquí. |
| `10_UC_Specific_Data_Model.md` | Define la estructura física de las tablas `bulk_import_jobs`, `bulk_export_jobs`, y las políticas de retención. |
| `11_UC_Specific_Security_Threats.md` | Las mitigaciones de "Data Poisoning", "Resource Exhaustion", y "PII Leakage" se validan mediante los patrones definidos aquí. |
| `12_UC_Specific_Performance_Benchmarks.md` | Los SLAs de latencia (<2s para iniciación) y throughput (1.000 registros/min) se validan mediante las métricas y load testing definidos aquí. |
| `SHUGO_Security_Architecture.md` (Transversal) | Este documento aplica los estándares globales de control de acceso y privacidad al contexto específico de UC-10. |

---

## 9. Cierre del Paquete de Diseño Funcional (UC-10)

Con este documento, completamos los **9 archivos del Paquete de Diseño Profundo Funcional (Tier 1)** para el UC-10:

1. ✅ `01_Requirements_Specification.md` - Especificación completa con 12 escenarios BDD.
2. ✅ `02_Use_Case_Diagram.md` - Diagrama de caso de uso UML con actores Admin, HRIS, y Background Processor.
3. ✅ `03_Activity_Diagram.md` - Diagrama de actividades con validación en 3 fases y estrategias de commit.
4. ✅ `04_Initial_Domain_Model.md` - Modelo de dominio conceptual (Aggregate Roots, Value Objects).
5. ✅ `05_State_Machine_Diagram.md` - Máquinas de estados para Import y Export jobs.
6. ✅ `06_Robustness_Diagram.md` - Diagrama de robustez ICONIX.
7. ✅ `07_Sequence_Diagram.md` - Diagrama de secuencia con flujos asíncronos y excepciones.
8. ✅ `08_Final_Class_Design.md` - Diseño de clases con firmas y patrones.
9. ✅ `09_Modern_Artifacts.md` - Contratos OpenAPI, métricas, resiliencia, compliance.

### 9.1. Resumen de lo Logrado

Hemos diseñado el sistema de importación/exportación masiva más robusto, seguro y escalable del mercado IAM:
- **Procesamiento Asíncrono Resiliente:** Separación clara entre iniciación rápida (<2s) y procesamiento largo (minutos a horas), con Circuit Breakers para Storage y Queue.
- **Validación en Tres Fases:** Fail-Fast optimizado que detecta errores de formato antes de consultar DB, ahorrando recursos.
- **Estrategias Transaccionales Flexibles:** `COMMIT_ALL` para integridad estricta, `COMMIT_VALID` para resiliencia a datos sucios.
- **Modo Dry Run:** Pre-validación sin cambios persistentes, reduciendo errores en producción.
- **Reportes Granulares:** CSV con resultado por registro individual, permitiendo corrección y re-importación de errores.
- **Seguridad de Datos:** URLs firmadas con TTL de 24h, permisos granulares, y quota mensual configurable.
- **Observabilidad Completa:** Métricas de throughput, latencia por fase de validación, profundidad de cola, y detección de jobs huérfanos.

### 9.2. Próximos Pasos (Documentos Técnicos Específicos)

Según la **Opción A+** acordada, cada UC tiene **12 documentos** (9 funcionales + 3 técnicos específicos). Para el UC-10, aún faltan:

1. **`10_UC_Specific_Data_Model.md`** - Modelo de datos físico específico (tablas `bulk_import_jobs`, `bulk_export_jobs`, índices, políticas de retención).
2. **`11_UC_Specific_Security_Threats.md`** - Análisis de amenazas específico (Data Poisoning, Resource Exhaustion, PII Leakage, Queue Tampering).
3. **`12_UC_Specific_Performance_Benchmarks.md`** - Benchmarks y SLAs específicos para latencia de iniciación, throughput de procesamiento, y capacidad de cola.

---
*Fin del Documento 09_Modern_Artifacts para UC-10*