# UC-10: Import/Export Masivo - Especificación de Requisitos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-10 - Import/Export Masivo (Bulk Identity Operations) |
| **Documento:** | 01_Requirements_Specification.md |
| **Metodología:** | ICONIX (Deep Dive) + DDD + Event-Driven Architecture + Async Processing |
| **Estándares:** | NIST SP 800-63A (Enrollment), GDPR Art. 5 (Data Minimization), SOC 2 CC6.1, ISO 27001 A.12.6 |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 10/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 10/09/2026 | Gastón Ruiz | **Creación inicial.** Definición completa de la especificación analítica para operaciones masivas de importación y exportación de identidades. Establece el flujo asíncrono de procesamiento por lotes (batch processing), validación de datos en múltiples fases, manejo de errores con reportes detallados, integración con sistemas externos (HRIS, Active Directory), y auditoría completa de operaciones masivas. | Documento base para el paquete de diseño Tier 1 del UC-10. Es el caso de uso que cierra el EPIC-01 (Identity Management), permitiendo la migración masiva de identidades desde sistemas legacy y la exportación para integración con sistemas de reporting y compliance. |

---

## 1. Introducción y Alcance

Este documento detalla la especificación de requisitos para el Caso de Uso **UC-10**, que representa la operación de **importación y exportación masiva de identidades** en el motor **KAGAMI** de SHUGO.

A diferencia de los UCs transaccionales individuales (UC-01 a UC-09) que procesan una identidad a la vez, el UC-10 maneja **operaciones por lotes (batch processing)** con las siguientes características críticas:

1. **Procesamiento Asíncrono:** Las operaciones masivas (100+ identidades) se ejecutan en background jobs para no bloquear la UI del administrador.
2. **Validación Multi-Fase:** Los datos se validan en tres fases: (1) validación sintáctica del archivo, (2) validación semántica de cada registro, (3) validación de consistencia con el estado actual del sistema.
3. **Manejo de Errores Granular:** Cada registro del lote puede fallar independientemente. El sistema genera un reporte detallado con errores específicos por fila.
4. **Transaccionalidad por Lote:** El administrador puede elegir entre "commit total" (todo o nada) o "commit parcial" (procesar los válidos, rechazar los inválidos).
5. **Rate Limiting y Throttling:** Las operaciones masivas consumen recursos significativos. El sistema limita el tamaño de lotes y la frecuencia de operaciones para proteger la infraestructura.
6. **Integración con Sistemas Externos:** Soporta formatos estándar (CSV, JSON, XML) para integración con HRIS (Workday, BambooHR), Active Directory, y sistemas legacy.
7. **Auditoría Completa:** Cada operación masiva se registra con metadata completa: quién la inició, cuándo, cuántos registros procesó, cuántos fallaron, y por qué.

### 1.1. Importancia Estratégica

El UC-10 es **crítico para la adopción enterprise y la migración de sistemas legacy** porque:

- **Onboarding Masivo:** Permite importar miles de empleados de una sola vez durante la implementación inicial o fusiones empresariales.
- **Sincronización con HRIS:** Facilita la integración bidireccional con sistemas de Recursos Humanos para mantener identidades actualizadas.
- **Compliance y Reporting:** La exportación masiva permite generar reportes para auditorías SOC 2, GDPR, y compliance normativo.
- **Migración de Sistemas Legacy:** Facilita la transición desde sistemas de identidad antiguos (LDAP, AD, bases de datos SQL) sin intervención manual.
- **Reducción de Costos Operativos:** Elimina la necesidad de crear identidades una por una, reduciendo el tiempo de onboarding de semanas a horas.

### 1.2. Relación con Otros Casos de Uso

| UC | Relación | Descripción |
| :--- | :--- | :--- |
| **UC-01 (Provisionar)** | Consumidor | UC-10 invoca UC-01 para cada identidad válida del lote. |
| **UC-02 (Actualizar)** | Consumidor | UC-10 invoca UC-02 para actualizar identidades existentes durante la importación. |
| **UC-04 (Consultar)** | Complementario | UC-04 permite consultar el estado de una operación masiva en progreso. |
| **UC-06 (Registro Self-Service)** | Alternativa | UC-06 es para registro individual; UC-10 es para importación masiva por admin. |
| **UC-07 (Invitación)** | Complementario | Tras importar identidades, UC-07 puede enviar invitaciones masivas. |
| **UC-46 (Auditoría)** | Consumidor | UC-46 permite consultar los logs de operaciones masivas. |

### 1.3. Objetivos de Rendimiento

| Métrica | Objetivo | Justificación |
| :--- | :--- | :--- |
| **Throughput de Importación** | 1.000 identidades/minuto | Suficiente para migraciones enterprise de hasta 50.000 identidades en 8 horas. |
| **Throughput de Exportación** | 5.000 identidades/minuto | Exportación es más rápida (solo lectura). Suficiente para reportes compliance. |
| **Latencia de Validación** | < 50ms por registro | Validación debe ser rápida para no convertirse en cuello de botella. |
| **Tasa de Éxito** | > 95% | En un lote bien formado, la mayoría de registros deben procesarse exitosamente. |
| **Disponibilidad** | 99.9% | Las operaciones masivas son críticas para implementación y migración. |
| **Tiempo de Respuesta API** | < 2s (aceptación del job) | El endpoint debe responder rápidamente, el procesamiento es asíncrono. |

---

## 2. Descripción Analítica (ICONIX Clásico)

| Campo | Detalle |
| :--- | :--- |
| **ID Y NOMBRE:** | UC-10 - Import/Export Masivo (Bulk Identity Operations) |
| **ESTADO:** | Preliminar |
| **DESCRIPCIÓN:** | Un administrador sube un archivo (CSV/JSON/XML) con datos de identidades para importar, o solicita la exportación de un conjunto de identidades. El sistema valida el formato del archivo, procesa cada registro en un background job, valida los datos contra las reglas de negocio (unicidad de email, formato de password, roles válidos), y ejecuta las operaciones (crear/actualizar) según la estrategia seleccionada (commit total o parcial). Genera un reporte detallado con el resultado de cada registro (éxito, error, advertencia) y notifica al administrador upon completion. Para exportación, genera un archivo descargable con las identidades solicitadas en el formato especificado. |
| **ACTOR PRINCIPAL:** | Administrador de Tenant (Humano), Sistema de RRHH (HRIS Automatizado vía API). |
| **ACTOR SECUNDARIO:** | Background Job Processor (Sistema), Storage Service (S3/Azure Blob para archivos), Notification Service (Email/Slack para notificaciones). |
| **PRECONDICIONES:** | 1. El administrador debe tener el permiso `identity:import` o `identity:export`.<br>2. El archivo debe cumplir el formato esperado (CSV/JSON/XML con schema válido).<br>3. El tenant debe tener quota disponible para la operación masiva.<br>4. Para importación: los datos deben cumplir las políticas del tenant (password complexity, email domain whitelist). |
| **PUNTOS DE EXTENSIÓN:** | Si el archivo es muy grande (>10.000 registros), se divide en chunks para procesamiento paralelo. Si hay errores críticos de formato, se rechaza el archivo completo antes de procesar. |
| **CONDICIÓN:** | La operación masiva se ejecuta asíncronamente. El administrador recibe un job_id para consultar el progreso. Al finalizar, recibe una notificación con el reporte de resultados. |

### 2.1. Escenario Principal (Happy Path) - Importación Exitosa

1. El administrador sube un archivo CSV con 500 identidades mediante POST `/identities/import`.
2. El API Gateway valida el JWT y el permiso `identity:import`.
3. El Import Handler recibe el archivo y valida el formato (tamaño < 50MB, extensión .csv/.json/.xml).
4. El Handler parsea el archivo y valida la estructura (headers correctos, campos requeridos presentes).
5. El Handler crea un job de importación en estado `PENDING` y retorna HTTP 202 Accepted con `job_id`.
6. El Background Job Processor toma el job y cambia su estado a `PROCESSING`.
7. El Processor valida cada registro en tres fases:
   - **Fase 1 (Sintáctica):** Valida formato de email, longitud de password, formato de campos.
   - **Fase 2 (Semántica):** Valida que el email no exista, que los roles sean válidos, que el dominio esté permitido.
   - **Fase 3 (Consistencia):** Valida que no haya duplicados dentro del lote, que las dependencias existan (ej. manager_id).
8. Para cada registro válido, el Processor invoca UC-01 (Provisionar) o UC-02 (Actualizar) según corresponda.
9. El Processor actualiza el progreso del job (ej. "Procesados 250 de 500").
10. Al finalizar, el Processor cambia el estado del job a `COMPLETED` y genera un reporte detallado.
11. El Notification Service envía un email al administrador con el resumen: "495 identidades importadas exitosamente, 5 fallaron".
12. El administrador descarga el reporte completo (CSV con detalles de cada registro).

### 2.2. Escenario Alternativo - Exportación Masiva

1. El administrador solicita exportar todas las identidades del tenant con rol "employee" mediante POST `/identities/export`.
2. El API Gateway valida el JWT y el permiso `identity:export`.
3. El Export Handler crea un job de exportación en estado `PENDING` y retorna HTTP 202 Accepted con `job_id`.
4. El Background Job Processor toma el job y cambia su estado a `PROCESSING`.
5. El Processor consulta la base de datos en lotes de 1.000 registros para no saturar memoria.
6. El Processor genera el archivo de exportación (CSV/JSON) en el storage temporal (S3/Azure Blob).
7. Al finalizar, el Processor cambia el estado del job a `COMPLETED` y genera una URL de descarga firmada (válida por 24h).
8. El Notification Service envía un email al administrador con el link de descarga.
9. El administrador descarga el archivo y lo usa para reportes o migración.

### 2.3. Escenario Alternativo - Importación con Errores Parciales

1-6. (Idénticos al escenario principal)
7. Durante la validación, el Processor detecta errores:
   - Fila 15: Email duplicado (ya existe en el sistema).
   - Fila 42: Password no cumple política de complejidad.
   - Fila 103: Rol "super_admin" no existe en el tenant.
   - Fila 250: Email inválido (formato incorrecto).
   - Fila 378: Manager ID no existe (dependencia rota).
8. El Processor marca estos registros como `FAILED` con el motivo específico.
9. El Processor continúa procesando los registros válidos (commit parcial).
10. Al finalizar, el job se marca como `COMPLETED_WITH_ERRORS`.
11. El reporte incluye 495 registros exitosos y 5 fallidos con detalles de cada error.
12. El administrador puede corregir los errores y re-importar solo los registros fallidos.

### 2.4. Flujos de Excepción

**[3.1] Archivo Inválido (Formato Incorrecto)**
- 3.1.1 El Handler detecta que el archivo no cumple el formato esperado (ej. CSV sin headers, JSON malformado).
- 3.1.2 Retorna HTTP 400 Bad Request con mensaje: "Invalid file format. Expected CSV with headers: email, display_name, password, roles."
- 3.1.3 No se crea ningún job.
- 3.1.4 Fin del Caso de Uso.

**[3.2] Archivo Demasiado Grande**
- 3.2.1 El Handler detecta que el archivo excede el tamaño máximo (50MB).
- 3.2.2 Retorna HTTP 413 Payload Too Large con mensaje: "File size exceeds maximum allowed (50MB)."
- 3.2.3 Fin del Caso de Uso.

**[4.1] Sin Permisos**
- 4.1.1 El API Gateway detecta que el administrador no tiene el permiso `identity:import`.
- 4.1.2 Retorna HTTP 403 Forbidden con mensaje "Insufficient permissions."
- 4.1.3 Fin del Caso de Uso.

**[7.1] Quota Excedida**
- 7.1.1 El Handler detecta que el tenant ha excedido su quota de importaciones (ej. máximo 10.000 identidades/mes).
- 7.1.2 Retorna HTTP 429 Too Many Requests con mensaje: "Monthly import quota exceeded. Contact support to increase limit."
- 7.1.3 Fin del Caso de Uso.

**[10.1] Error Crítico del Sistema**
- 10.1.1 El Background Job Processor encuentra un error crítico (ej. base de datos caída, storage lleno).
- 10.1.2 El job cambia a estado `FAILED` con el mensaje de error.
- 10.1.3 El Notification Service envía una alerta al administrador y al equipo de operaciones.
- 10.1.4 Fin del Caso de Uso.

### 2.5. Postcondiciones

**De Éxito (Importación):**
1. Las identidades válidas han sido creadas o actualizadas en el sistema.
2. Se ha generado un reporte detallado con el resultado de cada registro.
3. Se ha notificado al administrador con el resumen.
4. Se ha registrado la operación en los logs de auditoría.

**De Éxito (Exportación):**
1. Se ha generado un archivo con las identidades solicitadas.
2. El archivo está disponible para descarga mediante URL firmada (válida 24h).
3. Se ha notificado al administrador con el link de descarga.
4. Se ha registrado la operación en los logs de auditoría.

**De Falla:**
1. No se han procesado identidades (o solo las válidas en commit parcial).
2. Se ha generado un reporte de errores detallado.
3. Se ha notificado al administrador con los motivos de fallo.
4. Se ha registrado el fallo en los logs de auditoría.

---

## 3. Reglas de Negocio e Invariantes de Dominio (DDD)

| ID | Invariante / Regla | Descripción Técnica |
| :--- | :--- | :--- |
| **INV-60** | Permiso de Importación/Exportación | Solo administradores con permiso `identity:import` o `identity:export` pueden ejecutar operaciones masivas. |
| **INV-61** | Tamaño Máximo de Archivo | Los archivos de importación no pueden exceder 50MB. Los de exportación no tienen límite (se generan en streaming). |
| **INV-62** | Formatos Soportados | Importación: CSV, JSON, XML. Exportación: CSV, JSON. |
| **INV-63** | Validación en Tres Fases | Cada registro se valida en: (1) sintáctica, (2) semántica, (3) consistencia. |
| **INV-64** | Estrategia de Commit | El administrador elige entre `COMMIT_ALL` (todo o nada) o `COMMIT_VALID` (procesar válidos, rechazar inválidos). |
| **INV-65** | Procesamiento Asíncrono | Operaciones con >100 registros se ejecutan en background jobs. Operaciones pequeñas pueden ser síncronas. |
| **INV-66** | Rate Limiting de Operaciones | Máximo 10 operaciones masivas por tenant por hora. Máximo 1 operación masiva en progreso por tenant. |
| **INV-67** | Quota Mensual | Cada tenant tiene una quota mensual de identidades importables (ej. 10.000/mes). Configurable por plan. |
| **INV-68** | Reporte Detallado | Cada operación masiva genera un reporte con: total procesados, exitosos, fallidos, advertencias, y detalles por registro. |
| **INV-69** | URL de Descarga Firmada | Los archivos de exportación se sirven mediante URLs firmadas con TTL de 24 horas para seguridad. |
| **RN-10** | Chunking para Lotes Grandes | Archivos con >10.000 registros se dividen en chunks de 1.000 para procesamiento paralelo. |
| **RN-11** | Re-importación de Errores | El administrador puede descargar solo los registros fallidos, corregirlos, y re-importarlos. |
| **RN-12** | Notificaciones Multi-Canal | Las notificaciones de completación se envían por Email y opcionalmente por Slack/Teams. |
| **RN-13** | Auditoría Completa | Cada operación masiva registra: quién, cuándo, cuántos registros, cuántos fallaron, motivos de fallo. |
| **RN-14** | Integración con HRIS | Soporte para formatos estándar de HRIS (Workday, BambooHR, SAP SuccessFactors). |
| **RN-15** | Dry Run Mode | El administrador puede ejecutar una importación en modo "dry run" para validar sin procesar. |

---

## 4. Criterios de Aceptación (BDD / Gherkin)

### Escenario 4.1: Importación exitosa de lote válido
```gherkin
DADO que el administrador "admin-001" tiene el permiso "identity:import"
Y sube un archivo CSV con 500 registros válidos
Y selecciona estrategia "COMMIT_VALID"
CUANDO el sistema procesa el archivo
ENTONCES retorna HTTP 202 Accepted con job_id
Y el job cambia a estado "PROCESSING"
Y procesa las 500 identidades exitosamente
Y el job cambia a estado "COMPLETED"
Y envía notificación al administrador: "500 identidades importadas exitosamente"
Y genera reporte descargable con detalles de cada registro
```

### Escenario 4.2: Importación con errores parciales
```gherkin
DADO que el administrador sube un archivo CSV con 500 registros
Y 5 registros tienen errores (email duplicado, password débil, rol inválido, email inválido, manager inexistente)
Y selecciona estrategia "COMMIT_VALID"
CUANDO el sistema procesa el archivo
ENTONCES retorna HTTP 202 Accepted con job_id
Y procesa 495 identidades exitosamente
Y marca 5 registros como "FAILED" con motivos específicos
Y el job cambia a estado "COMPLETED_WITH_ERRORS"
Y envía notificación: "495 importadas, 5 fallaron"
Y genera reporte con detalles de los 5 errores
```

### Escenario 4.3: Importación con estrategia COMMIT_ALL (todo o nada)
```gherkin
DADO que el administrador sube un archivo CSV con 500 registros
Y 5 registros tienen errores
Y selecciona estrategia "COMMIT_ALL"
CUANDO el sistema procesa el archivo
ENTONCES detecta los 5 errores durante validación
Y NO procesa ninguna identidad (rollback total)
Y el job cambia a estado "FAILED"
Y envía notificación: "Importación fallida: 5 errores encontrados"
Y genera reporte con detalles de los 5 errores
```

### Escenario 4.4: Exportación masiva exitosa
```gherkin
DADO que el administrador tiene el permiso "identity:export"
Y solicita exportar 10.000 identidades con rol "employee" en formato CSV
CUANDO el sistema procesa la solicitud
ENTONCES retorna HTTP 202 Accepted con job_id
Y el job cambia a estado "PROCESSING"
Y genera archivo CSV con 10.000 registros en streaming
Y el job cambia a estado "COMPLETED"
Y genera URL de descarga firmada (válida 24h)
Y envía notificación con link de descarga
```

### Escenario 4.5: Archivo inválido (formato incorrecto)
```gherkin
DADO que el administrador sube un archivo CSV sin headers
CUANDO el sistema valida el formato
ENTONCES retorna HTTP 400 Bad Request
Y el mensaje es "Invalid file format. Expected CSV with headers: email, display_name, password, roles"
Y NO se crea ningún job
```

### Escenario 4.6: Archivo demasiado grande
```gherkin
DADO que el administrador sube un archivo de 60MB
CUANDO el sistema valida el tamaño
ENTONCES retorna HTTP 413 Payload Too Large
Y el mensaje es "File size exceeds maximum allowed (50MB)"
```

### Escenario 4.7: Sin permisos
```gherkin
DADO que el usuario "user-001" NO tiene el permiso "identity:import"
CUANDO intenta subir un archivo para importación
ENTONCES el sistema retorna HTTP 403 Forbidden
Y el mensaje es "Insufficient permissions"
```

### Escenario 4.8: Quota excedida
```gherkin
DADO que el tenant ha importado 10.000 identidades este mes (quota máxima)
Y el administrador intenta importar 100 identidades más
CUANDO el sistema valida la quota
ENTONCES retorna HTTP 429 Too Many Requests
Y el mensaje es "Monthly import quota exceeded. Contact support to increase limit"
```

### Escenario 4.9: Dry run mode (validación sin procesar)
```gherkin
DADO que el administrador sube un archivo CSV con 500 registros
Y selecciona modo "DRY_RUN"
CUANDO el sistema procesa el archivo
ENTONCES valida todos los registros
Y NO procesa ninguna identidad
Y retorna HTTP 202 Accepted con job_id
Y el job cambia a estado "COMPLETED"
Y genera reporte: "495 registros válidos, 5 inválidos"
Y NO crea ni actualiza ninguna identidad
```

### Escenario 4.10: Re-importación de registros fallidos
```gherkin
DADO que una importación anterior tuvo 5 registros fallidos
Y el administrador descarga el reporte de errores
Y corrige los 5 errores en el archivo
Y re-sube el archivo corregido
CUANDO el sistema procesa el archivo
ENTONCES procesa los 5 registros exitosamente
Y el job cambia a estado "COMPLETED"
Y envía notificación: "5 identidades importadas exitosamente"
```

### Escenario 4.11: Job en progreso bloquea nueva operación
```gherkin
DADO que ya existe un job de importación en estado "PROCESSING" para el tenant
Y el administrador intenta iniciar una nueva importación
CUANDO el sistema valida si hay jobs en progreso
ENTONCES retorna HTTP 409 Conflict
Y el mensaje es "Another bulk operation is already in progress. Please wait for it to complete"
```

### Escenario 4.12: Error crítico del sistema durante procesamiento
```gherkin
DADO que un job de importación está en estado "PROCESSING"
Y la base de datos se cae inesperadamente
CUANDO el Background Job Processor intenta continuar
ENTONCES el job cambia a estado "FAILED"
Y registra el error: "Database connection failed"
Y envía alerta al administrador y al equipo de operaciones
Y NO procesa más registros
```

---

## 5. Eventos de Dominio Generados

### Evento 1: `BulkImport.Initiated`
```json
{
  "eventId": "uuid-v4",
  "eventType": "BulkImport.Initiated",
  "timestamp": "2026-09-05T10:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "jobId": "job-uuid-123",
    "initiatedBy": "admin-001",
    "totalRecords": 500,
    "fileFormat": "CSV",
    "fileSizeBytes": 2048576,
    "commitStrategy": "COMMIT_VALID",
    "isDryRun": false
  },
  "metadata": {
    "traceId": "opentelemetry-trace-id",
    "ipAddress": "192.168.1.100",
    "userAgent": "Mozilla/5.0..."
  }
}
```

**Consumidores:**
1. **KAGE (Audit):** Registra el inicio de la operación masiva.
2. **TORII (Webhooks):** Notifica a sistemas externos (HRIS) que se inició una importación.
3. **Notification Service:** Envía email al administrador confirmando que el job fue aceptado.

### Evento 2: `BulkImport.Completed`
```json
{
  "eventId": "uuid-v4",
  "eventType": "BulkImport.Completed",
  "timestamp": "2026-09-05T10:15:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "jobId": "job-uuid-123",
    "status": "COMPLETED_WITH_ERRORS",
    "totalRecords": 500,
    "successfulRecords": 495,
    "failedRecords": 5,
    "processingTimeSeconds": 900,
    "reportUrl": "https://storage.shugo.io/reports/job-uuid-123.csv?signature=..."
  }
}
```

**Consumidores:**
1. **KAGE (Audit):** Registra la completación con métricas detalladas.
2. **TORII (Webhooks):** Notifica a sistemas externos que la importación finalizó.
3. **Notification Service:** Envía email al administrador con resumen y link al reporte.
4. **Metrics Service:** Actualiza métricas de throughput y tasa de éxito.

### Evento 3: `BulkExport.Completed`
```json
{
  "eventId": "uuid-v4",
  "eventType": "BulkExport.Completed",
  "timestamp": "2026-09-05T10:20:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "jobId": "job-uuid-456",
    "status": "COMPLETED",
    "totalRecords": 10000,
    "fileFormat": "CSV",
    "fileSizeBytes": 52428800,
    "downloadUrl": "https://storage.shugo.io/exports/job-uuid-456.csv?signature=...",
    "urlExpiresAt": "2026-09-06T10:20:00Z"
  }
}
```

**Consumidores:**
1. **KAGE (Audit):** Registra la exportación con metadata de seguridad.
2. **Notification Service:** Envía email al administrador con link de descarga.
3. **Security Monitor:** Monitorea descargas de archivos masivos para detectar accesos no autorizados.

---

## 6. Consideraciones de Arquitectura y Seguridad

### 6.1. Procesamiento Asíncrono con Background Jobs

Las operaciones masivas se ejecutan en background jobs para no bloquear la UI del administrador:

```csharp
public class BulkImportHandler : IBulkImportHandler
{
    private readonly IJobQueue _jobQueue;
    private readonly IFileValidator _fileValidator;
    private readonly ILogger<BulkImportHandler> _logger;

    public async Task<JobId> InitiateImportAsync(
        Stream fileStream, 
        string fileName, 
        ImportOptions options, 
        IdentityId adminId, 
        TenantId tenantId,
        CancellationToken ct)
    {
        // 1. Validar archivo (síncrono, rápido)
        var validationResult = await _fileValidator.ValidateAsync(fileStream, fileName, ct);
        if (!validationResult.IsValid)
            throw new InvalidFileException(validationResult.ErrorMessage);

        // 2. Crear job en estado PENDING
        var jobId = new JobId(Guid.NewGuid());
        var job = new BulkImportJob
        {
            Id = jobId,
            TenantId = tenantId,
            InitiatedBy = adminId,
            Status = JobStatus.PENDING,
            FileName = fileName,
            FileSizeBytes = fileStream.Length,
            Options = options,
            CreatedAt = DateTime.UtcNow
        };

        await _jobRepository.SaveAsync(job, ct);

        // 3. Subir archivo a storage temporal (S3/Azure Blob)
        var fileUrl = await _storageService.UploadAsync(fileStream, $"imports/{jobId}.csv", ct);
        job.FileUrl = fileUrl;
        await _jobRepository.UpdateAsync(job, ct);

        // 4. Encolar job para procesamiento asíncrono
        await _jobQueue.EnqueueAsync(new BulkImportMessage
        {
            JobId = jobId,
            FileUrl = fileUrl,
            Options = options
        }, ct);

        _logger.LogInformation(
            "Bulk import job {JobId} initiated by {AdminId} for tenant {TenantId}",
            jobId, adminId, tenantId);

        // 5. Publicar evento de dominio
        await _eventPublisher.PublishAsync(new BulkImportInitiated(
            jobId, adminId, tenantId, validationResult.TotalRecords, options), ct);

        return jobId;
    }
}
```

### 6.2. Validación en Tres Fases

Cada registro se valida en tres fases para detectar errores tempranamente:

```csharp
public class BulkImportProcessor : IBulkImportProcessor
{
    public async Task<ImportResult> ProcessAsync(BulkImportJob job, CancellationToken ct)
    {
        var result = new ImportResult { JobId = job.Id };

        // Descargar archivo desde storage
        using var fileStream = await _storageService.DownloadAsync(job.FileUrl, ct);
        var records = await _parser.ParseAsync(fileStream, job.Options.FileFormat, ct);

        result.TotalRecords = records.Count;

        // Procesar en chunks para no saturar memoria
        foreach (var chunk in records.Chunk(1000))
        {
            foreach (var (record, index) in chunk.Select((r, i) => (r, i)))
            {
                var rowNumber = index + 1;

                try
                {
                    // Fase 1: Validación Sintáctica
                    await ValidateSyntaxAsync(record, rowNumber, ct);

                    // Fase 2: Validación Semántica
                    await ValidateSemanticsAsync(record, rowNumber, job.TenantId, ct);

                    // Fase 3: Validación de Consistencia
                    await ValidateConsistencyAsync(record, rowNumber, job.TenantId, ct);

                    // Si es DRY_RUN, solo validar, no procesar
                    if (!job.Options.IsDryRun)
                    {
                        // Procesar registro (crear o actualizar identidad)
                        await ProcessRecordAsync(record, job, ct);
                        result.SuccessfulRecords++;
                    }
                }
                catch (ValidationException ex)
                {
                    result.FailedRecords++;
                    result.Errors.Add(new RecordError
                    {
                        RowNumber = rowNumber,
                        ErrorMessage = ex.Message,
                        ErrorCode = ex.ErrorCode,
                        RecordData = record
                    });

                    // Si estrategia es COMMIT_ALL, abortar todo el lote
                    if (job.Options.CommitStrategy == CommitStrategy.COMMIT_ALL)
                    {
                        throw new BulkImportFailedException(
                            $"Validation failed at row {rowNumber}: {ex.Message}", 
                            result);
                    }
                }
            }

            // Actualizar progreso del job
            result.ProcessedRecords += chunk.Length;
            await UpdateJobProgressAsync(job.Id, result, ct);
        }

        return result;
    }
}
```

### 6.3. Rate Limiting y Throttling

Para proteger la infraestructura, las operaciones masivas están sujetas a rate limiting:

```csharp
public class BulkOperationRateLimiter : IBulkOperationRateLimiter
{
    private const int MAX_CONCURRENT_JOBS_PER_TENANT = 1;
    private const int MAX_JOBS_PER_TENANT_PER_HOUR = 10;
    private const int MAX_RECORDS_PER_MONTH = 10000;

    public async Task ValidateAsync(TenantId tenantId, int recordCount, CancellationToken ct)
    {
        // 1. Validar que no haya otro job en progreso
        var activeJobs = await _jobRepository.GetActiveJobsByTenantAsync(tenantId, ct);
        if (activeJobs.Any())
            throw new BulkOperationInProgressException(
                "Another bulk operation is already in progress. Please wait for it to complete.");

        // 2. Validar rate limit por hora
        var recentJobs = await _jobRepository.GetJobsByTenantAsync(
            tenantId, 
            DateTime.UtcNow.AddHours(-1), 
            ct);
        
        if (recentJobs.Count >= MAX_JOBS_PER_TENANT_PER_HOUR)
            throw new RateLimitExceededException(
                $"Maximum {MAX_JOBS_PER_TENANT_PER_HOUR} bulk operations per hour exceeded.");

        // 3. Validar quota mensual
        var monthlyImports = await _jobRepository.GetMonthlyImportCountAsync(tenantId, ct);
        if (monthlyImports + recordCount > MAX_RECORDS_PER_MONTH)
            throw new QuotaExceededException(
                $"Monthly import quota exceeded. Current: {monthlyImports}, Requested: {recordCount}, Limit: {MAX_RECORDS_PER_MONTH}");
    }
}
```

### 6.4. Generación de Reportes Detallados

Cada operación masiva genera un reporte detallado con el resultado de cada registro:

```csharp
public class ReportGenerator : IReportGenerator
{
    public async Task<string> GenerateImportReportAsync(ImportResult result, CancellationToken ct)
    {
        using var memoryStream = new MemoryStream();
        using var writer = new StreamWriter(memoryStream);
        using var csv = new CsvWriter(writer, CultureInfo.InvariantCulture);

        // Escribir header
        csv.WriteHeader<ReportRow>();
        await csv.NextRecordAsync();

        // Escribir filas exitosas
        foreach (var success in result.SuccessfulRecords)
        {
            csv.WriteRecord(new ReportRow
            {
                RowNumber = success.RowNumber,
                Status = "SUCCESS",
                Email = success.Email,
                Action = success.IsNew ? "CREATED" : "UPDATED",
                IdentityId = success.IdentityId.ToString(),
                ErrorMessage = ""
            });
            await csv.NextRecordAsync();
        }

        // Escribir filas fallidas
        foreach (var error in result.Errors)
        {
            csv.WriteRecord(new ReportRow
            {
                RowNumber = error.RowNumber,
                Status = "FAILED",
                Email = error.RecordData.Email,
                Action = "",
                IdentityId = "",
                ErrorMessage = error.ErrorMessage
            });
            await csv.NextRecordAsync();
        }

        await writer.FlushAsync();
        memoryStream.Position = 0;

        // Subir reporte a storage
        var reportUrl = await _storageService.UploadAsync(
            memoryStream, 
            $"reports/{result.JobId}.csv", 
            ct);

        return reportUrl;
    }
}
```

### 6.5. URLs Firmadas para Descarga Segura

Los archivos de exportación se sirven mediante URLs firmadas con TTL:

```csharp
public class SecureFileService : ISecureFileService
{
    public async Task<string> GenerateSignedDownloadUrlAsync(
        string fileKey, 
        TimeSpan expiration, 
        CancellationToken ct)
    {
        // Generar URL firmada con AWS S3 Presigned URL
        var request = new GetPreSignedUrlRequest
        {
            BucketName = _config.BucketName,
            Key = fileKey,
            Expires = DateTime.UtcNow.Add(expiration),
            Verb = HttpVerb.GET
        };

        var url = _s3Client.GetPreSignedURL(request);
        return url;
    }
}
```

---

## 7. Contratos de API (OpenAPI 3.0 Snippets)

### 7.1. Endpoint: POST /identities/import

**Request:**
```http
POST /v1/identities/import HTTP/1.1
Host: api.shugo.io
Authorization: Bearer <JWT>
Content-Type: multipart/form-data

--boundary
Content-Disposition: form-data; name="file"; filename="identities.csv"
Content-Type: text/csv

email,display_name,password,roles,manager_id
user1@khcs.com,User One,SecureP@ss1,user,manager1@khcs.com
user2@khcs.com,User Two,SecureP@ss2,user,manager1@khcs.com
...
--boundary
Content-Disposition: form-data; name="commit_strategy"

COMMIT_VALID
--boundary
Content-Disposition: form-data; name="is_dry_run"

false
--boundary--
```

**Response (202 Accepted):**
```json
{
  "job_id": "job-uuid-123",
  "status": "PENDING",
  "message": "Bulk import job initiated. You will receive a notification upon completion.",
  "estimated_duration_seconds": 900
}
```

**Response (400 Bad Request - Archivo inválido):**
```json
{
  "error": {
    "code": "INVALID_FILE_FORMAT",
    "message": "Invalid file format. Expected CSV with headers: email, display_name, password, roles"
  }
}
```

**Response (409 Conflict - Job en progreso):**
```json
{
  "error": {
    "code": "BULK_OPERATION_IN_PROGRESS",
    "message": "Another bulk operation is already in progress. Please wait for it to complete.",
    "active_job_id": "job-uuid-456"
  }
}
```

### 7.2. Endpoint: GET /identities/import/jobs/{job_id}

**Request:**
```http
GET /v1/identities/import/jobs/job-uuid-123 HTTP/1.1
Host: api.shugo.io
Authorization: Bearer <JWT>
```

**Response (200 OK):**
```json
{
  "job_id": "job-uuid-123",
  "status": "PROCESSING",
  "progress": {
    "total_records": 500,
    "processed_records": 250,
    "successful_records": 248,
    "failed_records": 2,
    "percentage": 50.0
  },
  "started_at": "2026-09-05T10:00:00Z",
  "estimated_completion_at": "2026-09-05T10:15:00Z"
}
```

### 7.3. Endpoint: POST /identities/export

**Request:**
```http
POST /v1/identities/export HTTP/1.1
Host: api.shugo.io
Authorization: Bearer <JWT>
Content-Type: application/json

{
  "filters": {
    "roles": ["employee"],
    "status": "ACTIVE",
    "created_after": "2026-01-01T00:00:00Z"
  },
  "format": "CSV",
  "fields": ["email", "display_name", "roles", "created_at", "last_login_at"]
}
```

**Response (202 Accepted):**
```json
{
  "job_id": "job-uuid-789",
  "status": "PENDING",
  "message": "Bulk export job initiated. You will receive a notification with the download link.",
  "estimated_duration_seconds": 120
}
```

### 7.4. Endpoint: GET /identities/export/jobs/{job_id}/download

**Request:**
```http
GET /v1/identities/export/jobs/job-uuid-789/download HTTP/1.1
Host: api.shugo.io
Authorization: Bearer <JWT>
```

**Response (302 Found - Redirect to signed URL):**
```http
HTTP/1.1 302 Found
Location: https://storage.shugo.io/exports/job-uuid-789.csv?signature=abc123&expires=2026-09-06T10:20:00Z
```

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `02_Use_Case_Diagram.md` | Mostrará el límite del sistema, los actores (Administrador, HRIS, Background Job Processor), y las relaciones con UC-01, UC-02, y sistemas de storage. |
| `03_Activity_Diagram.md` | Visualizará el flujo de validación en tres fases, procesamiento asíncrono, generación de reportes, y notificaciones. |
| `04_Initial_Domain_Model.md` | Definirá el Aggregate Root `BulkImportJob`, los Value Objects `ImportOptions`, `RecordError`, y los Domain Services `BulkImportHandler`, `BulkImportProcessor`. |
| `05_State_Machine_Diagram.md` | Mostrará la máquina de estados del job (PENDING → PROCESSING → COMPLETED/FAILED). |
| `09_Modern_Artifacts.md` | Definirá los contratos OpenAPI completos, métricas de OpenTelemetry, y patrones de resiliencia para background jobs. |
| `UC-01/01_Requirements_Specification.md` | Documento hermano que define el provisioning individual (UC-10 invoca UC-01 para cada registro). |
| `UC-02/01_Requirements_Specification.md` | Documento hermano que define la actualización individual (UC-10 invoca UC-02 para registros existentes). |

---

## 9. Glosario de Términos Específicos de UC-10

| Término | Definición |
| :--- | :--- |
| **Bulk Operation** | Operación masiva que procesa múltiples registros de una sola vez (importación o exportación). |
| **Background Job** | Tarea asíncrona que se ejecuta en segundo plano sin bloquear la UI del usuario. |
| **Commit Strategy** | Estrategia de transaccionalidad: `COMMIT_ALL` (todo o nada) o `COMMIT_VALID` (procesar válidos, rechazar inválidos). |
| **Dry Run** | Modo de validación que procesa el archivo sin ejecutar cambios reales, útil para pre-validar datos. |
| **Chunking** | División de un lote grande en chunks más pequeños (ej. 1.000 registros) para procesamiento paralelo y eficiencia de memoria. |
| **Three-Phase Validation** | Validación en tres fases: (1) sintáctica, (2) semántica, (3) consistencia. |
| **Signed URL** | URL con firma criptográfica y TTL que permite descarga segura de archivos desde storage. |
| **Quota** | Límite mensual de identidades importables por tenant, configurable por plan. |
| **Record Error** | Error específico de un registro individual dentro de un lote, con número de fila y motivo. |
| **Job Progress** | Métrica de progreso de un job en ejecución: total procesados, exitosos, fallidos, porcentaje. |
| **HRIS Integration** | Integración con sistemas de Recursos Humanos (Workday, BambooHR, SAP) mediante formatos estándar. |
| **Re-import** | Proceso de corregir registros fallidos y re-importarlos sin procesar los exitosos nuevamente. |

---
*Fin del Documento 01_Requirements_Specification para UC-10*