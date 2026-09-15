# UC-10: Import/Export Masivo - Benchmarks, SLAs y Estrategia de Rendimiento

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-10 - Import/Export Masivo (Bulk Identity Operations) |
| **Documento:** | 12_UC_Specific_Performance_Benchmarks.md |
| **Metodología:** | Performance Engineering + SRE (SLI/SLO/SLA) + Load Testing (k6) + Capacity Planning + Async Processing Profiling |
| **Estándares:** | ISO/IEC 25010 (Performance Efficiency), Google SRE Workbook, OWASP Performance Project |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 14/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 14/09/2026 | Gastón Ruiz | **Creación inicial.** Definición de SLAs, SLOs, estrategia de Load Testing (enfocada en throughput de procesamiento masivo, latencia por fase de validación, capacidad de cola, y procesamiento de archivos grandes), y planificación de capacidad para la arquitectura asíncrona de operaciones masivas del motor KAGAMI. | Documento de referencia para SREs y Arquitectos de Rendimiento. Establece cómo mantener la latencia de iniciación sub-2s, el throughput de procesamiento >1.000 registros/min, y la capacidad de cola adecuada para picos de operaciones masivas enterprise. |

---

## 1. Introducción y Propósito

A diferencia de los UCs transaccionales individuales (UC-01 a UC-09), el **UC-10 (Import/Export Masivo)** es una operación de **procesamiento asíncrono de larga duración** con características de rendimiento únicas y desafiantes:

1. **Arquitectura Asíncrona de Dos Fases:** La iniciación es síncrona y rápida (<2s), mientras que el procesamiento es asíncrono y puede durar desde segundos (100 registros) hasta horas (500.000 registros).
2. **Procesamiento Intensivo de CPU y I/O:** Parsing de archivos, validación en 3 fases, y provisioning de identidades consumen recursos significativos.
3. **Dependencia de Infraestructura Externa:** Storage (S3/Azure Blob) y Job Queue (RabbitMQ/SQS) son puntos críticos que afectan el rendimiento end-to-end.
4. **Actualizaciones Frecuentes de Progreso:** Cada chunk procesado (1.000 registros) requiere una actualización de progreso en DB, generando carga de escritura.
5. **Generación de Reportes Pesada:** Los reportes CSV con resultados por registro pueden ser grandes (varios MB) y requieren procesamiento adicional.
6. **Operaciones de Larga Duración:** Los jobs pueden durar horas, requiriendo monitoreo de progreso en tiempo real y manejo de timeouts.

Este documento aborda estos desafíos definiendo SLAs realistas, estrategias de optimización y pruebas de carga específicas para garantizar que las operaciones masivas sean rápidas, escalables y confiables.

---

## 2. Service Level Indicators (SLIs) y Objectives (SLOs)

### 2.1. SLIs Específicos para UC-10

| SLI | Descripción | Fórmula / Medición |
| :--- | :--- | :--- |
| **Initiation Latency (p95)** | Tiempo desde que el usuario sube el archivo hasta que recibe HTTP 202 Accepted. | `histogram_quantile(0.95, rate(kagami_bulk_initiation_latency_bucket[5m]))` |
| **Job Processing Duration** | Tiempo total desde que el job inicia procesamiento hasta que se completa. | `histogram_quantile(0.95, rate(kagami_bulk_job_processing_duration_bucket[5m]))` |
| **Chunk Processing Latency (p95)** | Tiempo de procesamiento de un chunk de 1.000 registros. | `histogram_quantile(0.95, rate(kagami_bulk_chunk_processing_latency_bucket[5m]))` |
| **Validation Phase Latency (p95)** | Latencia de cada fase de validación (Sintáctica, Semántica, Consistencia). | `histogram_quantile(0.95, rate(kagami_bulk_validation_phase_latency_bucket[5m]))` |
| **Progress Update Latency (p95)** | Tiempo de actualización de progreso en DB. | `histogram_quantile(0.95, rate(kagami_bulk_progress_update_latency_bucket[5m]))` |
| **File Upload Latency (p95)** | Tiempo de subida de archivo a Storage. | `histogram_quantile(0.95, rate(kagami_bulk_file_upload_latency_bucket[5m]))` |
| **File Download Latency (p95)** | Tiempo de descarga de archivo desde Storage. | `histogram_quantile(0.95, rate(kagami_bulk_file_download_latency_bucket[5m]))` |
| **Report Generation Latency (p95)** | Tiempo de generación de reporte CSV. | `histogram_quantile(0.95, rate(kagami_bulk_report_generation_latency_bucket[5m]))` |
| **Queue Depth** | Número de jobs en cola esperando procesamiento. | `kagami_bulk_queue_depth` |
| **Processing Throughput** | Registros procesados por minuto. | `rate(kagami_bulk_records_processed_total[5m]) * 60` |
| **Import Success Rate** | Porcentaje de registros importados exitosamente. | `rate(kagami_bulk_records_processed_total{result="success"}[7d]) / rate(kagami_bulk_records_processed_total[7d])` |
| **Export Throughput** | Registros exportados por minuto. | `rate(kagami_bulk_export_records_total[5m]) * 60` |

### 2.2. SLOs y SLAs

| Métrica | SLO (Objetivo Interno - 95%) | SLA (Contrato Enterprise - 99%) | Justificación |
| :--- | :--- | :--- | :--- |
| **Latencia de Iniciación (p95)** | **< 2.000 ms** | **< 3.000 ms** | Incluye validación de archivo (~200ms), subida a Storage (~500ms), creación de job (~100ms), y encolado (~50ms). |
| **Latencia de Chunk (p95)** | **< 10.000 ms** | **< 15.000 ms** | Procesamiento de 1.000 registros: parsing (~100ms), validación 3 fases (~5.000ms), provisioning (~4.000ms), actualización progreso (~500ms). |
| **Latencia Fase 1 - Sintáctica (p95)** | **< 1 ms** | **< 2 ms** | Validación en memoria sin acceso a DB. |
| **Latencia Fase 2 - Semántica (p95)** | **< 3 ms** | **< 5 ms** | Incluye consultas a DB/Cache (unicidad de email, roles válidos). |
| **Latencia Fase 3 - Consistencia (p95)** | **< 2 ms** | **< 3 ms** | Incluye consultas de dependencias (manager_id). |
| **Latencia de Actualización de Progreso (p95)** | **< 50 ms** | **< 100 ms** | UPDATE en tabla `bulk_import_jobs` con OCC. |
| **Latencia de Subida de Archivo (p95)** | **< 1.000 ms** | **< 2.000 ms** | Upload de archivo de 50MB a S3/Azure Blob. |
| **Latencia de Descarga de Archivo (p95)** | **< 500 ms** | **< 1.000 ms** | Download de archivo de 50MB desde Storage. |
| **Latencia de Generación de Reporte (p95)** | **< 5.000 ms** | **< 10.000 ms** | Generación de CSV con 500.000 filas y subida a Storage. |
| **Throughput de Importación** | **> 1.000 registros/min** | **> 500 registros/min** | Limitado por validación en 3 fases y provisioning de identidades. |
| **Throughput de Exportación** | **> 5.000 registros/min** | **> 2.000 registros/min** | Exportación es más rápida (solo lectura, sin validación ni provisioning). |
| **Profundidad de Cola** | **< 10 jobs** | **< 20 jobs** | Más de 10 jobs en cola indica que los workers no dan abasto. |
| **Tasa de Éxito de Importación** | **> 90%** | **> 80%** | Porcentaje de registros importados exitosamente (excluyendo errores de datos del usuario). |
| **Disponibilidad del Servicio** | **99.9%** | **99.5%** | Permite caídas breves de Storage o Queue (usando Circuit Breaker). |

---

## 3. Cuellos de Botella y Estrategias de Optimización

### 3.1. Cuello de Botella #1: Validación en Tres Fases (CPU + I/O Bound)

**Problema:** La validación en 3 fases es el cuello de botella principal. La Fase 1 es rápida (en memoria), pero las Fases 2 y 3 requieren consultas a DB, lo que puede saturar la base de datos bajo carga.

**Estrategia de Optimización:**

#### A. Cache de Validaciones Semánticas (Redis)
```csharp
public class RecordValidator : IRecordValidator
{
    private readonly IDistributedCache _cache;
    private readonly TimeSpan _cacheTtl = TimeSpan.FromMinutes(5);

    public async Task ValidateSemanticPhaseAsync(Record record, int rowNumber, TenantId tenantId, CancellationToken ct)
    {
        // Cache de unicidad de email
        var emailCacheKey = $"email_exists:{tenantId.Value}:{record.Email.ToLower()}";
        var emailExists = await _cache.GetOrCreateAsync(emailCacheKey, async () =>
        {
            return await _identityRepo.EmailExistsAsync(record.Email, tenantId, ct);
        }, _cacheTtl);

        if (emailExists)
            throw new ValidationException("EMAIL_ALREADY_EXISTS", ...);

        // Cache de roles válidos por tenant
        var rolesCacheKey = $"valid_roles:{tenantId.Value}";
        var validRoles = await _cache.GetOrCreateAsync(rolesCacheKey, async () =>
        {
            return await _roleRepo.GetByTenantIdAsync(tenantId, ct);
        }, _cacheTtl);

        // Validar roles contra cache
        if (record.Roles.Any(r => !validRoles.Contains(r)))
            throw new ValidationException("INVALID_ROLES", ...);
    }
}
```

#### B. Batch Queries para Validación de Consistencia
```csharp
public class RecordValidator : IRecordValidator
{
    public async Task ValidateConsistencyPhaseAsync(
        List<Record> records, // Validar en batch, no uno por uno
        TenantId tenantId, 
        CancellationToken ct)
    {
        // Extraer todos los manager_ids del chunk
        var managerIds = records
            .Where(r => !string.IsNullOrWhiteSpace(r.ManagerId))
            .Select(r => r.ManagerId)
            .Distinct()
            .ToList();

        if (!managerIds.Any()) return;

        // Consulta batch (1 query en lugar de N queries)
        var existingManagers = await _identityRepo.GetByIdsAsync(managerIds, tenantId, ct);
        var existingManagerIds = existingManagers.Select(m => m.Id).ToHashSet();

        // Validar cada registro contra el resultado batch
        foreach (var record in records.Where(r => !string.IsNullOrWhiteSpace(r.ManagerId)))
        {
            if (!existingManagerIds.Contains(record.ManagerId))
                throw new ValidationException("MANAGER_NOT_FOUND", ...);
        }
    }
}
```

#### C. Índices Optimizados en DB
```sql
-- Índices para acelerar validaciones semánticas
CREATE INDEX IF NOT EXISTS idx_identities_tenant_email 
    ON kagami_identity.identities(tenant_id, LOWER(email));

CREATE INDEX IF NOT EXISTS idx_roles_tenant_name 
    ON kagami_identity.roles(tenant_id, name);

-- Índices para validaciones de consistencia
CREATE INDEX IF NOT EXISTS idx_identities_tenant_id 
    ON kagami_identity.identities(tenant_id, id);
```

### 3.2. Cuello de Botella #2: Provisioning de Identidades (I/O Bound)

**Problema:** El provisioning de identidades (UC-01) requiere múltiples INSERTs en DB (identities, passwords, consent_records, audit_logs), lo que puede ser lento bajo carga.

**Estrategia de Optimización:**

#### A. Batch Inserts para Provisioning
```csharp
public class IdentityProvisioningService : IIdentityProvisioningService
{
    public async Task ProvisionBatchAsync(List<Record> records, TenantId tenantId, CancellationToken ct)
    {
        // Preparar todas las entidades para insert en batch
        var identities = records.Select(r => new Identity
        {
            Id = Guid.NewGuid(),
            TenantId = tenantId.Value,
            Email = r.Email.ToLower(),
            DisplayName = r.DisplayName,
            Status = "PROVISIONED",
            CreatedAt = DateTime.UtcNow
        }).ToList();

        // Batch insert (1 query en lugar de N queries)
        await _dbContext.Identities.AddRangeAsync(identities, ct);
        await _dbContext.SaveChangesAsync(ct);

        // Batch insert de passwords
        var passwords = records.Zip(identities, (r, i) => new PasswordCredential
        {
            IdentityId = i.Id,
            PasswordHash = HashPassword(r.Password),
            CreatedAt = DateTime.UtcNow
        }).ToList();

        await _dbContext.PasswordCredentials.AddRangeAsync(passwords, ct);
        await _dbContext.SaveChangesAsync(ct);

        // ... más batch inserts ...
    }
}
```

#### B. Conexión Pooling Optimizado
```csharp
// Configuración de Npgsql para bulk operations
services.AddDbContext<AppDbContext>(options =>
{
    options.UseNpgsql(connectionString, npgsqlOptions =>
    {
        npgsqlOptions.EnableRetryOnFailure(
            maxRetryCount: 3,
            maxRetryDelay: TimeSpan.FromSeconds(5),
            errorCodesToAdd: null);
        
        npgsqlOptions.CommandTimeout(60); // 60s para operaciones bulk
    });
});

// Connection string optimizado
// Host=postgres;Database=shugo;Username=app;Password=***;Pooling=true;MinPoolSize=10;MaxPoolSize=100;ConnectionIdleLifetime=300;
```

### 3.3. Cuello de Botella #3: Actualización de Progreso (Write-Heavy)

**Problema:** Cada chunk procesado (1.000 registros) requiere una actualización de progreso en DB. Para un job de 500.000 registros, esto significa 500 UPDATEs, lo que puede saturar la DB.

**Estrategia de Optimización:**

#### A. Actualización Asíncrona de Progreso
```csharp
public class BulkImportProcessor : IBulkImportProcessor
{
    private readonly Channel<JobProgress> _progressChannel;

    public BulkImportProcessor(/* ... */)
    {
        // Canal asíncrono para actualizaciones de progreso
        _progressChannel = Channel.CreateBounded<JobProgress>(new BoundedChannelOptions(100)
        {
            FullMode = BoundedChannelFullMode.DropOldest
        });

        // Worker en background que procesa actualizaciones de progreso
        _ = Task.Run(async () =>
        {
            await foreach (var progress in _progressChannel.Reader.ReadAllAsync())
            {
                await _jobRepo.UpdateProgressAsync(progress.JobId, progress, CancellationToken.None);
            }
        });
    }

    public async Task ProcessJobAsync(JobId jobId, CancellationToken ct)
    {
        // ... procesar chunks ...

        foreach (var chunk in chunks)
        {
            await ProcessChunkAsync(chunk, job, ct);

            // Enviar actualización de progreso al canal (no bloqueante)
            await _progressChannel.Writer.WriteAsync(new JobProgress
            {
                JobId = jobId,
                ProcessedRecords = processedCount,
                SuccessfulRecords = successfulCount,
                FailedRecords = failedCount
            }, ct);
        }
    }
}
```

#### B. Throttling de Actualizaciones de Progreso
```csharp
public class BulkImportProcessor : IBulkImportProcessor
{
    private DateTime _lastProgressUpdate = DateTime.MinValue;
    private const int PROGRESS_UPDATE_INTERVAL_SECONDS = 5;

    public async Task ProcessJobAsync(JobId jobId, CancellationToken ct)
    {
        foreach (var chunk in chunks)
        {
            await ProcessChunkAsync(chunk, job, ct);

            // Solo actualizar progreso cada 5 segundos (no en cada chunk)
            if ((DateTime.UtcNow - _lastProgressUpdate).TotalSeconds >= PROGRESS_UPDATE_INTERVAL_SECONDS)
            {
                await _jobRepo.UpdateProgressAsync(jobId, progress, ct);
                _lastProgressUpdate = DateTime.UtcNow;
            }
        }

        // Actualización final obligatoria
        await _jobRepo.UpdateProgressAsync(jobId, finalProgress, ct);
    }
}
```

### 3.4. Cuello de Botella #4: Generación de Reportes (CPU + I/O Bound)

**Problema:** Generar un reporte CSV con 500.000 filas puede consumir mucha CPU y memoria, y la subida a Storage puede ser lenta.

**Estrategia de Optimización:**

#### A. Streaming de Reporte (No Acumular en Memoria)
```csharp
public class ReportGenerator : IReportGenerator
{
    public async Task<string> GenerateImportReportAsync(
        JobId jobId, 
        IAsyncEnumerable<RecordResult> results, 
        CancellationToken ct)
    {
        var reportKey = $"reports/{jobId.Value}.csv";
        
        // Usar MemoryStream con capacidad inicial para evitar reallocaciones
        using var memoryStream = new MemoryStream(capacity: 10 * 1024 * 1024); // 10MB inicial
        using var writer = new StreamWriter(memoryStream, Encoding.UTF8, leaveOpen: true);
        using var csv = new CsvWriter(writer, CultureInfo.InvariantCulture);

        // Escribir header
        csv.WriteHeader<ReportRow>();
        await csv.NextRecordAsync();

        // Streaming de resultados (no acumular en lista)
        await foreach (var result in results.WithCancellation(ct))
        {
            csv.WriteRecord(new ReportRow
            {
                RowNumber = result.RowNumber,
                Email = result.Email,
                Status = result.IsSuccess ? "SUCCESS" : "FAILED",
                ErrorCode = result.ErrorCode,
                ErrorMessage = result.ErrorMessage
            });
            await csv.NextRecordAsync();
        }

        await writer.FlushAsync();
        memoryStream.Position = 0;

        // Subir a Storage
        await _storageService.UploadAsync(memoryStream, reportKey, ct);

        return reportKey;
    }
}
```

#### B. Compresión de Reportes Grandes
```csharp
public class ReportGenerator : IReportGenerator
{
    public async Task<string> GenerateImportReportAsync(/* ... */)
    {
        // ... generar CSV en memoryStream ...

        // Si el reporte es > 10MB, comprimirlo
        if (memoryStream.Length > 10 * 1024 * 1024)
        {
            using var compressedStream = new MemoryStream();
            using (var gzip = new GZipStream(compressedStream, CompressionLevel.Fastest, leaveOpen: true))
            {
                memoryStream.Position = 0;
                await memoryStream.CopyToAsync(gzip, ct);
            }

            compressedStream.Position = 0;
            await _storageService.UploadAsync(compressedStream, $"{reportKey}.gz", ct);
            
            return $"{reportKey}.gz";
        }

        // Si es pequeño, subir sin comprimir
        memoryStream.Position = 0;
        await _storageService.UploadAsync(memoryStream, reportKey, ct);
        return reportKey;
    }
}
```

### 3.5. Cuello de Botella #5: Capacidad de Cola (Queue Depth)

**Problema:** Si hay más jobs en cola que workers disponibles, los jobs esperan demasiado tiempo antes de ser procesados, degradando la UX.

**Estrategia de Optimización:**

#### A. Auto-Scaling de Workers basado en Queue Depth
```yaml
# Kubernetes HPA basado en profundidad de cola
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
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100 # Duplicar workers rápidamente
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

#### B. Priorización de Jobs por Tenant
```csharp
public class RabbitMqJobQueue : IJobQueue
{
    public async Task EnqueueAsync(BulkImportMessage message, CancellationToken ct)
    {
        // Prioridad basada en plan del tenant (Enterprise > Professional > Basic)
        var priority = await GetTenantPriorityAsync(message.TenantId, ct);
        
        await _channel.BasicPublishAsync(
            exchange: "",
            routingKey: "bulk_import_jobs",
            mandatory: false,
            basicProperties: new BasicProperties
            {
                Priority = priority, // 0 (lowest) to 9 (highest)
                DeliveryMode = DeliveryModes.Persistent
            },
            body: body,
            cancellationToken: ct);
    }

    private async Task<byte> GetTenantPriorityAsync(Guid tenantId, CancellationToken ct)
    {
        var tenant = await _tenantRepo.GetByIdAsync(tenantId, ct);
        return tenant.Plan switch
        {
            "ENTERPRISE" => 9,
            "PROFESSIONAL" => 5,
            "BASIC" => 1,
            _ => 0
        };
    }
}
```

---

## 4. Estrategia de Load Testing (Pruebas de Carga)

Utilizamos **k6** para simular escenarios realistas de operaciones masivas. Es crucial probar no solo el rendimiento, sino también la estabilidad bajo carga sostenida y la capacidad de recuperación ante fallos.

### 4.1. Escenario 1: Importación Masiva de 50.000 Registros

**Objetivo:** Validar que el sistema puede importar 50.000 registros en menos de 50 minutos (1.000 registros/min).

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Trend, Rate, Counter } from 'k6/metrics';

const initiationLatency = new Trend('initiation_latency', true);
const processingDuration = new Trend('processing_duration', true);
const errorRate = new Rate('errors');
const recordsProcessed = new Counter('records_processed');

export const options = {
  scenarios: {
    bulk_import: {
      executor: 'per-vu-iterations',
      vus: 5,
      iterations: 1,
      maxDuration: '60m',
    },
  },
  thresholds: {
    initiation_latency: ['p(95)<2000'], // Iniciación < 2s
    processing_duration: ['p(95)<3000000'], // Procesamiento < 50 min (3M ms)
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

  const startInitiation = Date.now();
  const res = http.post(`${__ENV.BASE_URL}/v1/identities/import`, formData, {
    headers: { 'Authorization': `Bearer ${__ENV.ADMIN_TOKEN}` },
    timeout: '10s',
  });
  const initiationTime = Date.now() - startInitiation;
  initiationLatency.add(initiationTime);

  check(res, {
    'import initiated (202)': (r) => r.status === 202,
    'has job_id': (r) => r.json('job_id') !== undefined,
  });

  if (res.status !== 202) {
    errorRate.add(1);
    return;
  }

  const jobId = res.json('job_id');

  // Poll job status until completion
  const startProcessing = Date.now();
  let completed = false;
  let attempts = 0;
  
  while (!completed && attempts < 3000) { // Max 50 minutes
    sleep(1);
    attempts++;
    
    const statusRes = http.get(`${__ENV.BASE_URL}/v1/jobs/${jobId}`, {
      headers: { 'Authorization': `Bearer ${__ENV.ADMIN_TOKEN}` },
    });
    
    const status = statusRes.json('status');
    const progress = statusRes.json('progress');
    
    if (progress) {
      recordsProcessed.add(progress.processed_records);
    }
    
    if (['COMPLETED', 'COMPLETED_WITH_ERRORS', 'FAILED'].includes(status)) {
      completed = true;
      const processingTime = Date.now() - startProcessing;
      processingDuration.add(processingTime);
      
      check(statusRes, {
        'job completed successfully': (r) => 
          r.json('status') === 'COMPLETED' || r.json('status') === 'COMPLETED_WITH_ERRORS',
        'success rate > 90%': (r) => {
          const successful = r.json('progress.successful_records');
          const total = r.json('progress.total_records');
          return (successful / total) > 0.90;
        },
      });
    }
  }
  
  if (!completed) {
    errorRate.add(1);
    console.error('Job did not complete within 50 minutes');
  }
}

function generateLargeCSV(numRecords) {
  let csv = 'email,display_name,password,roles\n';
  for (let i = 0; i < numRecords; i++) {
    csv += `user${i}@khcs.com,User ${i},SecureP@ssw0rd!${i},user\n`;
  }
  return csv;
}
```

### 4.2. Escenario 2: Exportación Masiva de 100.000 Registros

**Objetivo:** Validar que el sistema puede exportar 100.000 registros en menos de 20 minutos (5.000 registros/min).

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Trend, Rate } from 'k6/metrics';

const initiationLatency = new Trend('initiation_latency', true);
const processingDuration = new Trend('processing_duration', true);
const errorRate = new Rate('errors');

export const options = {
  scenarios: {
    bulk_export: {
      executor: 'per-vu-iterations',
      vus: 3,
      iterations: 1,
      maxDuration: '30m',
    },
  },
  thresholds: {
    initiation_latency: ['p(95)<2000'],
    processing_duration: ['p(95)<1200000'], // < 20 min
    errors: ['rate<0.05'],
  },
};

export default function () {
  const payload = JSON.stringify({
    filters: {
      status: 'ACTIVE'
    },
    format: 'CSV',
    fields: ['email', 'display_name', 'roles', 'status', 'created_at']
  });

  const startInitiation = Date.now();
  const res = http.post(`${__ENV.BASE_URL}/v1/identities/export`, payload, {
    headers: { 
      'Authorization': `Bearer ${__ENV.ADMIN_TOKEN}`,
      'Content-Type': 'application/json'
    },
    timeout: '10s',
  });
  const initiationTime = Date.now() - startInitiation;
  initiationLatency.add(initiationTime);

  check(res, {
    'export initiated (202)': (r) => r.status === 202,
    'has job_id': (r) => r.json('job_id') !== undefined,
  });

  if (res.status !== 202) {
    errorRate.add(1);
    return;
  }

  const jobId = res.json('job_id');

  // Poll job status until completion
  const startProcessing = Date.now();
  let completed = false;
  let attempts = 0;
  
  while (!completed && attempts < 1200) { // Max 20 minutes
    sleep(1);
    attempts++;
    
    const statusRes = http.get(`${__ENV.BASE_URL}/v1/jobs/${jobId}`, {
      headers: { 'Authorization': `Bearer ${__ENV.ADMIN_TOKEN}` },
    });
    
    const status = statusRes.json('status');
    
    if (status === 'COMPLETED') {
      completed = true;
      const processingTime = Date.now() - startProcessing;
      processingDuration.add(processingTime);
      
      // Descargar archivo exportado
      const reportUrl = statusRes.json('download_url');
      const downloadRes = http.get(reportUrl, {
        timeout: '60s',
      });
      
      check(downloadRes, {
        'file downloaded (200)': (r) => r.status === 200,
        'file size > 0': (r) => r.body.length > 0,
      });
    } else if (status === 'FAILED') {
      completed = true;
      errorRate.add(1);
    }
  }
  
  if (!completed) {
    errorRate.add(1);
    console.error('Export job did not complete within 20 minutes');
  }
}
```

### 4.3. Escenario 3: Carga Sostenida de Operaciones Masivas

**Objetivo:** Validar que el sistema puede manejar múltiples operaciones masivas simultáneas sin degradar el rendimiento.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Trend, Rate, Counter } from 'k6/metrics';

const queueDepth = new Trend('queue_depth', true);
const errorRate = new Rate('errors');
const jobsCompleted = new Counter('jobs_completed');

export const options = {
  scenarios: {
    sustained_load: {
      executor: 'constant-arrival-rate',
      rate: 2, // 2 jobs por minuto
      timeUnit: '1m',
      duration: '30m',
      preAllocatedVUs: 10,
      maxVUs: 50,
    },
  },
  thresholds: {
    queue_depth: ['p(95)<10'], // Cola no debe exceder 10 jobs
    errors: ['rate<0.10'],
  },
};

export default function () {
  // Generar CSV con 10.000 registros
  const csvContent = generateCSV(10000);
  
  const formData = {
    file: http.file(csvContent, `identities_${__VU}_${__ITER}.csv`, 'text/csv'),
    commit_strategy: 'COMMIT_VALID',
    is_dry_run: 'false'
  };

  const res = http.post(`${__ENV.BASE_URL}/v1/identities/import`, formData, {
    headers: { 'Authorization': `Bearer ${__ENV.ADMIN_TOKEN}` },
    timeout: '10s',
  });

  check(res, {
    'import initiated (202)': (r) => r.status === 202,
  });

  if (res.status !== 202) {
    errorRate.add(1);
    return;
  }

  const jobId = res.json('job_id');

  // Monitorear profundidad de cola
  const queueRes = http.get(`${__ENV.BASE_URL}/v1/admin/queue-depth`, {
    headers: { 'Authorization': `Bearer ${__ENV.ADMIN_TOKEN}` },
  });
  
  if (queueRes.status === 200) {
    queueDepth.add(queueRes.json('depth'));
  }

  // Esperar a que el job se complete
  let completed = false;
  let attempts = 0;
  
  while (!completed && attempts < 600) { // Max 10 minutes
    sleep(1);
    attempts++;
    
    const statusRes = http.get(`${__ENV.BASE_URL}/v1/jobs/${jobId}`, {
      headers: { 'Authorization': `Bearer ${__ENV.ADMIN_TOKEN}` },
    });
    
    const status = statusRes.json('status');
    
    if (['COMPLETED', 'COMPLETED_WITH_ERRORS', 'FAILED'].includes(status)) {
      completed = true;
      jobsCompleted.add(1);
    }
  }
  
  if (!completed) {
    errorRate.add(1);
  }
}

function generateCSV(numRecords) {
  let csv = 'email,display_name,password,roles\n';
  for (let i = 0; i < numRecords; i++) {
    csv += `user${__VU}_${__ITER}_${i}@khcs.com,User ${i},SecureP@ssw0rd!${i},user\n`;
  }
  return csv;
}
```

### 4.4. Escenario 4: Stress Test de Resource Exhaustion

**Objetivo:** Validar que el sistema maneja correctamente archivos grandes y previene Resource Exhaustion.

```javascript
import http from 'k6/http';
import { check } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  scenarios: {
    stress_test: {
      executor: 'per-vu-iterations',
      vus: 10,
      iterations: 1,
      maxDuration: '10m',
    },
  },
  thresholds: {
    errors: ['rate<0.05'],
  },
};

export default function () {
  // Escenario 1: Archivo de 50MB (límite máximo)
  const largeFile = generateLargeFile(50 * 1024 * 1024); // 50MB
  
  const formData = {
    file: http.file(largeFile, 'large.csv', 'text/csv'),
    commit_strategy: 'COMMIT_VALID',
  };

  const res = http.post(`${__ENV.BASE_URL}/v1/identities/import`, formData, {
    headers: { 'Authorization': `Bearer ${__ENV.ADMIN_TOKEN}` },
    timeout: '30s',
  });

  check(res, {
    'large file accepted (202)': (r) => r.status === 202,
  });

  if (res.status !== 202) {
    errorRate.add(1);
  }

  // Escenario 2: Archivo de 60MB (debe ser rechazado)
  const tooLargeFile = generateLargeFile(60 * 1024 * 1024); // 60MB
  
  const formData2 = {
    file: http.file(tooLargeFile, 'too_large.csv', 'text/csv'),
    commit_strategy: 'COMMIT_VALID',
  };

  const res2 = http.post(`${__ENV.BASE_URL}/v1/identities/import`, formData2, {
    headers: { 'Authorization': `Bearer ${__ENV.ADMIN_TOKEN}` },
    timeout: '30s',
  });

  check(res2, {
    'too large file rejected (413)': (r) => r.status === 413,
  });

  if (res2.status !== 413) {
    errorRate.add(1);
  }
}

function generateLargeFile(sizeBytes) {
  const header = 'email,display_name,password,roles\n';
  const row = 'user@khcs.com,User,SecureP@ssw0rd!123,user\n';
  const rowSize = row.length;
  const numRows = Math.floor((sizeBytes - header.length) / rowSize);
  
  let content = header;
  for (let i = 0; i < numRows; i++) {
    content += row;
  }
  
  return content;
}
```

---

## 5. Capacity Planning y Escalabilidad

### 5.1. Cálculo de Recursos para Workers de Importación

Para procesar **1.000 registros/min** con validación en 3 fases y provisioning:

1. **CPU por Worker:**
   - Parsing de archivo: ~10% CPU
   - Validación Fase 1 (Sintáctica): ~5% CPU
   - Validación Fase 2 (Semántica): ~15% CPU (consultas a DB/Cache)
   - Validación Fase 3 (Consistencia): ~10% CPU (consultas a DB)
   - Provisioning de identidades: ~40% CPU (hashing de passwords, múltiples INSERTs)
   - Generación de reporte: ~20% CPU
   - **Total:** ~100% CPU por worker (1 core dedicado)

2. **Memoria por Worker:**
   - Chunk de 1.000 registros en memoria: ~10MB
   - Cache de validaciones: ~50MB
   - Buffer de streaming: ~5MB
   - **Total:** ~65MB por worker

3. **Conexiones a DB por Worker:**
   - Validaciones semánticas: ~5 conexiones concurrentes
   - Provisioning: ~10 conexiones concurrentes
   - Actualización de progreso: ~1 conexión
   - **Total:** ~16 conexiones por worker

**Recomendación de Infraestructura:**
- **Workers de Importación:** 2-10 pods (auto-scaling basado en queue depth)
  - CPU: 1 core por pod
  - Memoria: 256MB por pod
  - Conexiones a DB: 20 por pod (con margen de seguridad)

### 5.2. Cálculo de Recursos para Workers de Exportación

Para exportar **5.000 registros/min** (solo lectura, sin validación ni provisioning):

1. **CPU por Worker:**
   - Consulta de identidades: ~30% CPU
   - Generación de CSV en streaming: ~50% CPU
   - Subida a Storage: ~20% CPU
   - **Total:** ~100% CPU por worker (1 core dedicado)

2. **Memoria por Worker:**
   - Batch de 1.000 registros en memoria: ~10MB
   - Buffer de streaming: ~5MB
   - **Total:** ~15MB por worker

**Recomendación de Infraestructura:**
- **Workers de Exportación:** 2-5 pods (auto-scaling basado en queue depth)
  - CPU: 1 core por pod
  - Memoria: 128MB por pod
  - Conexiones a DB: 10 por pod

### 5.3. Cálculo de Capacidad de Storage (S3/Azure Blob)

Para soportar **100 operaciones masivas/día** con archivos de hasta 50MB:

1. **Espacio de Almacenamiento:**
   - Archivos de importación: 100 jobs * 50MB = 5GB/día
   - Archivos de exportación: 100 jobs * 50MB = 5GB/día
   - Reportes de importación: 100 jobs * 10MB = 1GB/día
   - **Total:** ~11GB/día

2. **Política de Retención:**
   - Archivos de importación: 7 días (luego se eliminan)
   - Archivos de exportación: 7 días (luego se eliminan)
   - Reportes: 90 días (luego se archivan)
   - **Espacio máximo:** 11GB/día * 7 días + 1GB/día * 90 días = 77GB + 90GB = 167GB

**Recomendación:**
- **Storage:** S3 Standard con lifecycle policies para eliminación automática tras 7 días
- **Costo estimado:** ~$4/mes (167GB * $0.023/GB)

### 5.4. Cálculo de Capacidad de Cola (RabbitMQ/SQS)

Para soportar **100 operaciones masivas/día** con picos de hasta 20 jobs simultáneos:

1. **Throughput de Cola:**
   - 100 jobs/día = ~0.001 jobs/seg (promedio)
   - Pico: 20 jobs en 1 hora = ~0.005 jobs/seg
   - **Total:** Muy bajo, RabbitMQ/SQS pueden manejar miles de msgs/seg

2. **Profundidad de Cola:**
   - SLO: < 10 jobs en cola
   - Si hay 10 jobs en cola y cada job tarda 50 min en procesarse, el último job esperará 500 min (~8 horas)
   - **Recomendación:** Auto-scaling de workers para mantener queue depth < 10

**Recomendación:**
- **Cola:** RabbitMQ cluster con 3 nodos o AWS SQS Standard
- **Workers:** Auto-scaling de 2 a 10 pods basado en queue depth

### 5.5. Cálculo de IOPS de Base de Datos

Para soportar **1.000 registros/min** con validación en 3 fases y provisioning:

1. **Lecturas por registro:**
   - Fase 2 (Semántica): 2 lecturas (unicidad de email, roles válidos)
   - Fase 3 (Consistencia): 1 lectura (manager_id existe)
   - **Total:** 3 lecturas por registro

2. **Escrituras por registro:**
   - Provisioning: 4 escrituras (identities, passwords, consent_records, audit_logs)
   - Actualización de progreso: 0.001 escrituras por registro (1 actualización cada 1.000 registros)
   - **Total:** ~4 escrituras por registro

3. **IOPS totales:**
   - Lecturas: 1.000 registros/min * 3 lecturas = 3.000 lecturas/min = 50 IOPS
   - Escrituras: 1.000 registros/min * 4 escrituras = 4.000 escrituras/min = 67 IOPS
   - **Total:** ~117 IOPS

**Recomendación:**
- **PostgreSQL:** db.r5.large (2 vCPUs, 16GB RAM) con GP3 (3.000 IOPS base)
- **Read Replica:** 1 replica para consultas de exportación y reportes

---

## 6. Observabilidad y Profiling en Producción

### 6.1. Distributed Tracing (OpenTelemetry)

El Trace de una operación masiva debe descomponerse en spans granulares:

**Iniciación (Síncrona):**
```
HTTP POST /identities/import (Root Span) [Total: 1.500ms]
├── File Validation [200ms]
│   ├── Parse Headers [50ms]
│   ├── Validate MIME Type [10ms]
│   └── Count Records [140ms]
├── Storage Upload [500ms]
│   └── S3 PutObject [500ms]
├── Job Creation [100ms]
│   └── INSERT bulk_import_jobs [100ms]
└── Queue Enqueue [50ms]
    └── RabbitMQ Publish [50ms]
```

**Procesamiento (Asíncrono):**
```
Background Job: Process Import Job (Root Span) [Total: 3.000.000ms = 50min]
├── File Download [500ms]
│   └── S3 GetObject [500ms]
├── Chunk Processing [2.900.000ms]
│   ├── Chunk 1 [10.000ms]
│   │   ├── Parse Records [100ms]
│   │   ├── Validation Phase 1 (Syntactic) [50ms]
│   │   ├── Validation Phase 2 (Semantic) [5.000ms]
│   │   │   ├── Email Uniqueness Check [3.000ms]
│   │   │   └── Roles Validation [2.000ms]
│   │   ├── Validation Phase 3 (Consistency) [2.000ms]
│   │   │   └── Manager ID Check [2.000ms]
│   │   ├── Identity Provisioning [2.500ms]
│   │   │   ├── INSERT identities [500ms]
│   │   │   ├── INSERT passwords [500ms]
│   │   │   ├── INSERT consent_records [500ms]
│   │   │   └── INSERT audit_logs [1.000ms]
│   │   └── Progress Update [350ms]
│   │       └── UPDATE bulk_import_jobs [350ms]
│   ├── Chunk 2 [10.000ms]
│   │   └── ... (mismo patrón)
│   └── Chunk 50 [10.000ms]
│       └── ... (mismo patrón)
├── Report Generation [50.000ms]
│   ├── Generate CSV [30.000ms]
│   └── S3 PutObject [20.000ms]
└── Notification [1.000ms]
    └── Send Email [1.000ms]
```

**Spans Críticos a Monitorear:**
1. **Validation Phase 2 (Semantic):** Si >5.000ms por chunk, revisar índices de DB y cache hit ratio.
2. **Identity Provisioning:** Si >2.500ms por chunk, revisar conexión pooling y batch inserts.
3. **Progress Update:** Si >500ms, revisar carga de DB y optimizar queries.
4. **Report Generation:** Si >50.000ms, revisar streaming y compresión.

### 6.2. Dashboards de Grafana (Específicos para UC-10)

#### Panel 1: Throughput de Operaciones Masivas
*   **Métricas:** `records_processed_total` (importación), `export_records_total` (exportación).
*   **Objetivo:** Importación >1.000 registros/min, Exportación >5.000 registros/min.
*   **Alerta:** Si throughput cae por debajo del SLO, investigar cuellos de botella.

#### Panel 2: Latencia por Fase de Validación
*   **Métricas:** `validation_phase_latency` (p50, p95, p99) por fase (Sintáctica, Semántica, Consistencia).
*   **Objetivo:** Fase 1 <1ms, Fase 2 <3ms, Fase 3 <2ms.
*   **Alerta:** Si Fase 2 o 3 superan los SLOs, revisar índices de DB y cache hit ratio.

#### Panel 3: Profundidad de Cola y Auto-Scaling
*   **Métricas:** `queue_depth`, `workers_count`.
*   **Objetivo:** Queue depth <10, workers escalando automáticamente.
*   **Alerta:** Si queue depth >20 por >5 min, escalar workers manualmente o investigar por qué el auto-scaling no funciona.

#### Panel 4: Tasa de Éxito de Importación
*   **Métricas:** `records_processed_total{result="success"}` / `records_processed_total`.
*   **Objetivo:** >90% de éxito.
*   **Alerta:** Si tasa de éxito cae por debajo del 80%, revisar calidad de datos de los archivos subidos.

#### Panel 5: Capacidad de Storage
*   **Métricas:** `storage_bytes_used`, `storage_files_count`.
*   **Objetivo:** <200GB de almacenamiento total.
*   **Alerta:** Si storage supera 200GB, revisar lifecycle policies y eliminar archivos antiguos.

### 6.3. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Queue Depth High** | `queue_depth > 20` por 5 min | **Alta** | Workers no dan abasto. Escalar workers o investigar por qué jobs tardan tanto. |
| **Validation Phase Slow** | `validation_phase_latency_p95 > 5.000ms` para Fase 2 | Media | Consultas de DB lentas. Revisar índices y cache hit ratio. |
| **Provisioning Slow** | `identity_provisioning_latency_p95 > 2.500ms` | Media | Provisioning lento. Revisar conexión pooling y batch inserts. |
| **Progress Update Slow** | `progress_update_latency_p95 > 500ms` | Media | Actualizaciones de progreso lentas. Revisar carga de DB. |
| **Storage Errors** | `storage_errors_total > 5` en 5 min | **Crítica** | S3/Azure Blob caído. Circuit Breaker debe abrirse automáticamente. |
| **Queue Errors** | `queue_errors_total > 5` en 5 min | **Crítica** | RabbitMQ/SQS caído. Circuit Breaker debe abrirse automáticamente. |
| **Low Success Rate** | `import_success_rate < 0.80` en 1h | Media | Muchos registros fallando. Revisar calidad de datos de los archivos. |
| **Orphaned Jobs** | `orphaned_jobs_detected_total > 0` | Alta | Workers fallando. Revisar logs de workers y reiniciar si es necesario. |
| **Storage Near Limit** | `storage_bytes_used > 200GB` | Media | Storage接近límite. Revisar lifecycle policies y eliminar archivos antiguos. |

---

## 7. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los objetivos de rendimiento iniciales (<2s para iniciación, >1.000 registros/min para procesamiento) que este documento formaliza. |
| `08_Final_Class_Design.md` | Las clases `BulkImportHandler`, `BulkImportProcessor`, `RecordValidator`, y `ReportGenerator` deben implementar las optimizaciones definidas aquí. |
| `10_UC_Specific_Data_Model.md` | Los índices y configuraciones de DB definidos allí son críticos para cumplir los SLAs de latencia de validación y actualización de progreso. |
| `11_UC_Specific_Security_Threats.md` | Las mitigaciones de "Resource Exhaustion" (límites de tamaño, timeouts) tienen un costo de rendimiento que se gestiona mediante las optimizaciones definidas aquí. |
| `SHUGO_Performance_Testing_Strategy.md` (Transversal) | Este documento aplica la estrategia global al contexto específico de operaciones masivas asíncronas de larga duración. |

---

## 8. Anexos: Configuración de Rendimiento

### 8.1. Configuración de PostgreSQL para Bulk Operations

```ini
# postgresql.conf - Optimizaciones para UC-10

# Conexiones y pooling
max_connections = 500                    # Ajustar según número de workers
superuser_reserved_connections = 5

# Memoria compartida
shared_buffers = 8GB                     # 25% de RAM total
effective_cache_size = 24GB              # 75% de RAM total
work_mem = 128MB                         # Para operaciones de ordenamiento en reportes
maintenance_work_mem = 2GB               # Para VACUUM y CREATE INDEX

# WAL y Checkpoints (crítico para actualizaciones frecuentes de progreso)
wal_level = replica
max_wal_size = 4GB
min_wal_size = 1GB
checkpoint_completion_target = 0.9
checkpoint_timeout = 10min
wal_buffers = 64MB

# Autovacuum agresivo para tablas de jobs (actualizaciones frecuentes)
autovacuum_max_workers = 5
autovacuum_vacuum_scale_factor = 0.01    # Vaciar cuando 1% de filas cambian
autovacuum_analyze_scale_factor = 0.005  # Analizar cuando 0.5% de filas cambian

# Logging para monitoreo de rendimiento
log_min_duration_statement = 100         # Log queries > 100ms
log_checkpoints = on
log_lock_waits = on
```

### 8.2. Configuración de Redis para Cache de Validaciones

```ini
# redis.conf - Optimizaciones para UC-10

# Memoria
maxmemory 4gb
maxmemory-policy allkeys-lru

# Persistencia (deshabilitada para cache)
save ""
appendonly no

# Performance
tcp-keepalive 300
timeout 0
tcp-backlog 511

# Cluster (para alta disponibilidad)
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
```

### 8.3. Configuración de RabbitMQ para Cola de Jobs

```ini
# rabbitmq.conf - Optimizaciones para UC-10

# Límites de memoria y disco
vm_memory_high_watermark.relative = 0.6
disk_free_limit.absolute = 5GB

# Cola de jobs
queue_master_locator = min-masters
consumer_timeout = 3600000  # 1 hora (timeout para jobs largos)

# Prefetch count (cuántos mensajes puede tener un worker sin ack)
consumer_prefetch = 1  # Solo 1 job por worker a la vez

# Heartbeat (detectar workers caídos)
heartbeat = 60
```

### 8.4. Configuración de Kubernetes HPA para Workers

```yaml
# HPA para Workers de Importación
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
          averageValue: "5"
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

---

# HPA para Workers de Exportación
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kagami-export-workers-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kagami-export-workers
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: External
      external:
        metric:
          name: kagami_bulk_queue_depth
        target:
          type: AverageValue
          averageValue: "3"
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---
*Fin del Documento 12_UC_Specific_Performance_Benchmarks para UC-10*