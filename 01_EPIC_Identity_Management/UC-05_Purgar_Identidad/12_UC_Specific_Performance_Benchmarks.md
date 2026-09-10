# UC-05: Purgar Identidad - Benchmarks, SLAs y Estrategia de Rendimiento

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-05 - Purgar Identidad (Hard Delete / GDPR Right to Erasure) |
| **Documento:** | 12_UC_Specific_Performance_Benchmarks.md |
| **Metodología:** | Performance Engineering + Cryptographic Optimization + Batch Processing Tuning + Compliance SLAs |
| **Estándares:** | ISO/IEC 25010 (Performance Efficiency), Google SRE (SLI/SLO/SLA), AWS KMS Best Practices, NIST SP 800-88 |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 03/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 03/09/2026 | Gastón Ruiz | **Creación inicial.** Definición de SLAs, SLOs, estrategia de Load Testing (enfocada en latencia de crypto-shredding, throughput de batch processing y resiliencia de Legal Hold), y planificación de capacidad para operaciones destructivas de alto impacto. | Documento de referencia para SREs y Arquitectos de Rendimiento. Establece cómo mantener la latencia predecible (< 500ms p95 para purga individual) y garantizar el throughput masivo (1000 identidades/minuto) sin saturar la base de datos ni el KMS, mientras se respetan los bloqueos de compliance. |

---

## 1. Introducción y Propósito

A diferencia de los UCs de lectura (UC-04) o escritura estándar (UC-02), el **UC-05 (Purgar Identidad)** es una operación de **baja frecuencia pero altísima criticidad y costo computacional**. 

El rendimiento en UC-05 no se mide en "máximo throughput posible", sino en **predictibilidad, aislamiento de recursos y cumplimiento de SLAs de compliance**. Los desafíos únicos incluyen:

1. **Overhead Criptográfico:** La generación de hashes HMAC-SHA256 para PII y la reescritura de `snapshot_json` en audit logs consume CPU y IOPS.
2. **Dependencias Externas Críticas:** La consulta al sistema de Legal Hold (o KMS para claves) puede convertirse en un cuello de botella si no se maneja con timeouts estrictos.
3. **Procesamiento Batch Masivo:** Purgar 1000 identidades no debe bloquear la base de datos ni afectar a otros tenants (Noisy Neighbor).
4. **Transacciones Pesadas:** La atomicidad que incluye `identities`, `profiles`, `audit_logs` y `outbox_events` genera bloqueos (row-level locks) que deben minimizarse.

Este documento define las estrategias de optimización, SLAs realistas y pruebas de carga específicas para garantizar que la purga sea rápida, segura y escalable.

---

## 2. Service Level Indicators (SLIs) y Objectives (SLOs)

### 2.1. SLIs Específicos para UC-05

| SLI | Descripción | Fórmula / Medición |
| :--- | :--- | :--- |
| **Individual Purge Latency** | Tiempo end-to-end desde que el Admin envía la petición hasta que recibe el HTTP 200 OK. | `histogram_quantile(0.95, rate(kagami_identity_purge_latency_bucket[5m]))` |
| **Crypto-Shredding Latency** | Tiempo específico dedicado a la generación de hashes HMAC y anonimización de JSON. | `histogram_quantile(0.95, rate(kagami_identity_crypto_shredding_latency_bucket[5m]))` |
| **Legal Hold Check Latency** | Tiempo de respuesta de la validación de retención legal (incluyendo timeouts). | `histogram_quantile(0.95, rate(kagami_identity_legal_hold_check_latency_bucket[5m]))` |
| **Batch Throughput** | Número de identidades purgadas exitosamente por minuto en un job batch. | `rate(kagami_identity_batch_purge_total[1m])` |
| **Batch Job Completion Time** | Tiempo total para completar un job de 1000 identidades. | `histogram_quantile(0.99, rate(kagami_identity_batch_job_duration_bucket[1h]))` |
| **Audit Log Anonymization Rate** | Número de logs de auditoría anonimizados por segundo durante la purga. | `rate(kagami_identity_audit_logs_anonymized_total[1m])` |

### 2.2. SLOs y SLAs

| Métrica | SLO (Objetivo Interno - 95%) | SLA (Contrato Enterprise - 99%) | Justificación |
| :--- | :--- | :--- | :--- |
| **Individual Purge Latency (p95)** | **< 500 ms** | **< 1000 ms** | Incluye Legal Hold check, HMAC, y transacción atómica. |
| **Legal Hold Check Latency (p95)** | **< 50 ms** | **< 100 ms** | Debe ser rápido; si falla, el Circuit Breaker actúa en < 2s. |
| **Crypto-Shredding Latency (p95)** | **< 5 ms** | **< 10 ms** | Usando HMAC local con clave en memoria (Envelope Encryption). |
| **Batch Throughput** | **> 500 identidades/min** | **> 200 identidades/min** | Suficiente para purgar 1000 identidades en < 2 minutos. |
| **Batch Job Completion (1000 IDs)** | **< 2 min** | **< 5 min** | Incluye reintentos y manejo de fallos parciales. |
| **Disponibilidad del Endpoint** | **99.9%** | **99.5%** | Permite mantenimiento programado para rotación de claves KMS. |

---

## 3. Cuellos de Botella y Estrategias de Optimización

### 3.1. Overhead de KMS y Crypto-Shredding

**Problema:** Llamar a AWS KMS o Azure Key Vault para generar un HMAC por cada campo PII (email, phone) en cada identidad y cada log de auditoría saturará los límites de cuota del KMS (ej. 5500 req/seg en AWS) y añadirá ~20-50ms de latencia de red por llamada.

**Estrategia de Optimización: Envelope Encryption / Local HMAC**

En lugar de llamar al KMS para cada operación, el `PIIAnonymizer` utiliza una **Data Key** simétrica almacenada en memoria (o caché local segura) que fue derivada/desencriptada previamente desde el KMS.

```csharp
public class PIIAnonymizer : IPIIAnonymizer
{
    private readonly IHmacProvider _hmacProvider; // Usa SHA256Managed localmente
    private readonly IKeyManagementService _kms;  // Solo para obtener/rotar la Data Key

    // La Data Key se cachea por Tenant por 24 horas o hasta que se rote
    private readonly IMemoryCache _tenantKeyCache; 

    public AnonymizedData AnonymizeProfile(Profile profile, TenantId tenantId)
    {
        // 1. Obtener clave local (sin llamada a red KMS)
        var key = _tenantKeyCache.GetOrCreate(tenantId, entry => {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(24);
            return _kms.GetTenantDataKey(tenantId); 
        });

        // 2. Generar HMAC localmente (CPU bound, < 1ms)
        var emailHash = _hmacProvider.ComputeHash(key, profile.Email.Value);
        // ...
    }
}
```

**Impacto:**
- **Sin optimización:** ~40ms por campo PII (Llamada KMS).
- **Con optimización:** < 1ms por campo PII (CPU local).
- **Mejora:** 97.5% reducción en latencia criptográfica.

### 3.2. Anonimización Masiva de Audit Logs

**Problema:** Si una identidad tiene 500 logs de auditoría, ejecutar 500 `UPDATE` individuales dentro de la transacción de purga bloqueará la tabla y generará un WAL (Write-Ahead Log) masivo.

**Estrategia de Optimización: Bulk Update con `jsonb_set`**

Utilizar una única consulta SQL optimizada para actualizar todos los logs de una identidad en bloque.

```sql
-- En lugar de 500 UPDATEs individuales:
UPDATE audit_logs 
SET 
    snapshot_json = anonymize_json_function(snapshot_json, :hmac_key),
    anonymized = TRUE
WHERE identity_id = :identity_id 
  AND tenant_id = :tenant_id 
  AND anonymized = FALSE;
```

**Impacto:**
- **Row-by-Row:** ~500ms para 500 logs.
- **Bulk Update:** ~20ms para 500 logs.
- **Mejora:** 96% reducción en tiempo de bloqueo de tabla.

### 3.3. Aislamiento de Batch Processing (Noisy Neighbor Prevention)

**Problema:** Un job de purga de 1000 identidades puede consumir todas las conexiones de la DB y afectar a los UCs de lectura/escritura de otros tenants.

**Estrategia de Optimización: Dedicated Worker & Chunking**

1. **Cola Dedicada:** Los jobs batch no se ejecutan en el thread del API. Se encolan en un worker separado con un pool de conexiones a DB limitado (ej. máximo 5 conexiones concurrentes para purga).
2. **Chunking con Checkpointing:** Procesar en lotes de 50 identidades. Tras cada lote, hacer `COMMIT` y actualizar el `last_processed_index` en la tabla `purge_jobs`.

```csharp
public async Task ProcessBatchChunkAsync(Guid jobId, List<Guid> identityIds, CancellationToken ct)
{
    using var transaction = await _dbContext.Database.BeginTransactionAsync(ct);
    try
    {
        foreach (var id in identityIds)
        {
            await _purgeService.ExecutePurgeAsync(id, ...);
        }
        await transaction.CommitAsync(); // Commit atómico por chunk
        
        // Actualizar checkpoint
        await _jobRepository.UpdateCheckpointAsync(jobId, identityIds.Last());
    }
    catch
    {
        await transaction.RollbackAsync();
        throw; // El worker reintentará este chunk específico
    }
}
```

**Impacto:** Previene transacciones gigantes que puedan causar `OutOfMemory` en el ORM o bloqueos prolongados en la DB.

---

## 4. Estrategia de Load Testing (Pruebas de Carga)

Utilizamos **k6** para simular escenarios específicos de UC-05.

### 4.1. Escenario 1: Purga Individual con Legal Hold Activo (Fail-Safe)

**Objetivo:** Validar que el Circuit Breaker del Legal Hold no degrada la latencia del endpoint y bloquea correctamente.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    legal_hold_fail_safe: {
      executor: 'constant-vus',
      vus: 50,
      duration: '2m',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<2500'], // El timeout del Legal Hold es 2s + overhead
    http_req_failed: ['rate>0.90'], // Esperamos que >90% fallen con 403
  },
};

export default function () {
  const res = http.post(
    `${__ENV.BASE_URL}/v1/identities/${__ENV.IDENTITY_UNDER_HOLD}/purge`,
    JSON.stringify({ reason: "Load test", confirm_irreversible: true }),
    {
      headers: { 
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${__ENV.ADMIN_TOKEN}` 
      },
    }
  );

  check(res, {
    'blocked by legal hold (403)': (r) => r.status === 403,
    'error code is LEGAL_HOLD_ACTIVE': (r) => r.json('error.code') === 'LEGAL_HOLD_ACTIVE',
  });

  sleep(0.1);
}
```

### 4.2. Escenario 2: Batch Purge Throughput (1000 Identidades)

**Objetivo:** Medir el tiempo total y el consumo de recursos al purgar 1000 identidades.

```javascript
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  scenarios: {
    batch_purge_1000: {
      executor: 'shared-iterations',
      iterations: 1,
      vus: 1,
    },
  },
};

export function setup() {
  // Generar payload de 1000 IDs (en producción vendría de un query de elegibilidad)
  const ids = Array.from({length: 1000}, (_, i) => `uuid-identity-${i}`);
  return { payload: JSON.stringify({ identity_ids: ids, reason: "Batch load test", confirm_irreversible: true }) };
}

export default function (data) {
  const res = http.post(`${__ENV.BASE_URL}/v1/identities/purge-batch`, data.payload, {
    headers: { 'Authorization': `Bearer ${__ENV.ADMIN_TOKEN}` }
  });

  check(res, {
    'batch accepted (202)': (r) => r.status === 202,
  });

  const jobId = res.json('job_id');
  
  // Polling simplificado (en k6 real usaríamos un loop con sleep)
  // ... lógica de polling hasta status == COMPLETED ...
}
```

---

## 5. Capacity Planning y Escalabilidad

### 5.1. Recursos de Base de Datos (PostgreSQL)

La purga es intensiva en **IOPS de escritura** y **CPU** (por las funciones de anonimización JSON).

| Escala | Identidades a Purgar (Batch) | IOPS Requeridas (Pico) | Configuración Recomendada |
| :--- | :--- | :--- | :--- |
| **Startup** | < 100 | ~500 IOPS | db.t3.medium (2 vCPU, 4GB RAM, GP3 SSD) |
| **Scale-up** | < 1000 | ~2000 IOPS | db.r5.large (2 vCPU, 16GB RAM, GP3 SSD 3000 IOPS) |
| **Enterprise** | > 10,000 | ~10,000 IOPS | db.r5.xlarge + Read Replicas + Dedicated Purge Worker |

### 5.2. Límites de KMS (AWS Key Management Service)

Si se decide usar KMS directamente (sin Envelope Encryption local):
- **Límite por defecto:** 5500 requests/segundo por clave simétrica en AWS KMS.
- **Cálculo:** 1000 identidades * 2 campos PII (email, phone) * 10 logs promedio = 22,000 llamadas.
- **Tiempo estimado a límite:** 22,000 / 5500 = 4 segundos de saturación.
- **Solución:** Solicitar aumento de cuota a AWS o, preferiblemente, usar **Envelope Encryption** (como se detalla en la sección 3.1).

### 5.3. Escalado de Workers de Batch

Para tenants Enterprise que requieren purgas masivas frecuentes (>10,000 identidades/día):

```yaml
# Kubernetes HPA para Batch Purge Workers
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kagami-batch-purge-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kagami-batch-purge-worker
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Pods
      pods:
        metric:
          name: kagami_purge_queue_depth
        target:
          type: AverageValue
          averageValue: "10" # Escalar si hay >10 jobs en cola por worker
```

---

## 6. Observabilidad y Profiling en Producción

### 6.1. Distributed Tracing (OpenTelemetry)

El Trace de una purga individual debe descomponerse para identificar cuellos de botella de compliance:

```
HTTP POST /identities/{id}/purge (Root Span)
├── Legal Hold Check (External Call / DB)
│   └── [Hit] Cache / [Miss] DB Query
── Grace Period Evaluation (CPU)
├── Crypto-Shredding (CPU)
│   ├── PII Anonymization (HMAC)
│   └── Audit Log Anonymization (Bulk JSON update)
├── Identity.Purge() (Domain Logic)
└── DB Transaction (Commit)
    ├── UPDATE identities
    ├── UPDATE profiles
    ├── UPDATE audit_logs
    └── INSERT outbox_events
```

**Spans Críticos a Monitorear:**
1. `Legal Hold Check`: Debe ser < 50ms. Si es > 2000ms, el Circuit Breaker debe abrirse.
2. `Audit Log Anonymization`: Si una identidad tiene >1000 logs, este span crecerá linealmente. Alertar si > 500ms.

### 6.2. Dashboards de Grafana (Específicos para UC-05)

#### Dashboard: Compliance & Purge Operations

```
Panel 1: Purge Success vs Blocked (Time Series)
- Métricas: 
  - rate(kagami_identity_purge_total[5m])
  - rate(kagami_identity_purge_blocked_legal_hold_total[5m])
- Objetivo: Visualizar la efectividad del Legal Hold.

Panel 2: Crypto-Shredding Latency (Heatmap)
- Métrica: kagami_identity_crypto_shredding_latency
- Objetivo: Detectar degradación en el rendimiento de HMAC o KMS.

Panel 3: Batch Job Queue Depth (Gauge)
- Métrica: kagami_purge_queue_depth
- Umbral: Rojo si > 50 jobs pendientes.

Panel 4: Incomplete Anonymization Alert (Stat)
- Query SQL: "SELECT count(*) FROM identities WHERE status='PURGED' AND (email_hash IS NULL OR phone_hash IS NULL)"
- Umbral: Rojo si > 0 (Indica fallo crítico en el proceso de purga).
```

### 6.3. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Legal Hold Circuit Breaker OPEN** | `legal_hold_check_errors > 3` en 1 min | **Crítica** | Todas las purgas se bloquean. Investigar conectividad con DB de compliance. |
| **Incomplete Crypto-Shredding** | Query de integridad retorna > 0 filas | **Crítica** | Fuga de datos PII. Ejecutar script de remediación manual inmediatamente. |
| **Batch Job Stuck** | Job en `PROCESSING` por > 2 horas | Alta | Revisar logs del worker, posible deadlock en DB. |
| **KMS Rate Limit Exceeded** | `kms_throttling_exceptions > 0` | Alta | Si no se usa Envelope Encryption, activar backoff o cambiar a clave local. |

---

## 7. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los objetivos de rendimiento iniciales (< 500ms p95, 1000 IDs/minuto) que este documento formaliza en SLOs. |
| `09_Modern_Artifacts.md` | Define las métricas de OpenTelemetry; este documento explica cómo usarlas para validar SLAs de crypto-shredding y batch processing. |
| `10_UC_Specific_Data_Model.md` | Los índices compuestos y las funciones `anonymize_json` definidos aquí son críticos para cumplir los SLOs de latencia de anonimización. |
| `11_UC_Specific_Security_Threats.md` | Las mitigaciones de "Batch Processing DoS" y "KMS Compromise" se validan mediante el rate limiting y Envelope Encryption definidos aquí. |
| `SHUGO_Performance_Testing_Strategy.md` (Transversal) | Este documento aplica la estrategia global al contexto específico de operaciones destructivas y de compliance. |

---

## 8. Anexos: Configuración de Rendimiento

### 8.1. PostgreSQL Tuning para Operaciones de Purga (Heavy UPDATEs)

```ini
# postgresql.conf - Ajustes para mejorar el rendimiento de UPDATEs masivos en audit_logs
# Aumentar el espacio de trabajo para operaciones de hash y sort
work_mem = 128MB                   # Para anonimización de JSON complejos

# Optimizar WAL para escrituras intensivas
wal_buffers = 64MB
checkpoint_completion_target = 0.9
synchronous_commit = on            # Crítico para compliance (no perder datos de purga)

# Autovacuum agresivo para limpiar tuplas muertas tras las purgas
autovacuum_vacuum_scale_factor = 0.01  # Vaciar cuando 1% de filas cambian
autovacuum_analyze_scale_factor = 0.005
```

### 8.2. Límites de Timeouts para Resiliencia

```yaml
# Configuración de Timeouts (Application Layer)
timeouts:
  legal_hold_check: 2000ms         # Si tarda más, el Circuit Breaker asume ACTIVE (Fail-Safe)
  kms_data_key_fetch: 500ms        # Para obtener la clave de Envelope Encryption
  db_transaction_purge: 5000ms     # Timeout estricto para la transacción atómica
  batch_job_max_duration: 7200000ms # 2 horas máximo por job
```

---
*Fin del Documento 12_UC_Specific_Performance_Benchmarks para UC-05*