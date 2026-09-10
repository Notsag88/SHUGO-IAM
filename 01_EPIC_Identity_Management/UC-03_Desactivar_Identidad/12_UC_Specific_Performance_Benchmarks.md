# UC-03: Desactivar Identidad - Benchmarks, SLAs y Estrategia de Rendimiento

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-03 - Desactivar Identidad (Soft Delete) |
| **Documento:** | 12_UC_Specific_Performance_Benchmarks.md |
| **Metodología:** | Performance Engineering + Domain Guard Profiling + Mass Offboarding Analysis + Fail Secure Latency Budget |
| **Estándares:** | ISO/IEC 25010 (Performance Efficiency), Google SRE (SLI/SLO/SLA), NIST SP 800-57 (Key Management) |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 01/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 01/09/2026 | Gastón Ruiz | **Creación inicial.** Definición de SLAs, SLOs, estrategia de Load Testing (enfocada en guardas de dominio y offboarding masivo), y planificación de capacidad para operaciones de desactivación crítica. | Documento de referencia para SREs y Arquitectos de Rendimiento. Establece cómo mantener la latencia baja (< 100ms p95) a pesar de la sobrecarga inherente de las guardas de seguridad (Last Admin Check), la generación de auditoría forense y la propagación asíncrona de eventos a KAGI/KIZUNA. |

---

## 1. Introducción y Propósito

A diferencia del UC-01 (Crear) y UC-02 (Actualizar), el **UC-03 (Desactivar)** es una operación de **alta criticidad y bajo volumen** con características de rendimiento únicas:

1. **Guardas de Dominio Costosas:** La validación de "Último Administrador Activo" requiere una consulta adicional a la base de datos que puede convertirse en cuello de botella.
2. **Auditoría Forense Obligatoria:** Cada desactivación debe generar un log inmutable con `reason`, `actor`, y `timestamp`, añadiendo latencia de escritura.
3. **Propagación Asíncrona Crítica:** El evento `IdentityDeactivatedEvent` debe llegar a KAGI (revocación de sesiones) y KIZUNA (invalidación de caché) para aplicar Fail Secure.
4. **Escenarios de Pico (Mass Offboarding):** Despidos colectivos o incidentes de seguridad pueden generar ráfagas de 1000+ desactivaciones en minutos.
5. **Idempotencia y Reintentos:** Sistemas HRIS (SCIM) pueden reintentar desactivaciones, requiriendo manejo eficiente de operaciones no-op.

Este documento aborda estos desafíos y define estrategias de optimización, SLAs realistas, y pruebas de carga específicas.

---

## 2. Service Level Indicators (SLIs) y Objectives (SLOs)

### 2.1. SLIs Específicos para UC-03

| SLI | Descripción | Fórmula / Medición |
| :--- | :--- | :--- |
| **Deactivation Latency** | Tiempo desde que el API Gateway recibe `POST /deactivate` hasta que responde HTTP 200. | `histogram_quantile(0.95, rate(kagami_identity_deactivation_latency_bucket[5m]))` |
| **Last Admin Check Latency** | Tiempo específico de la consulta `GetActiveAdminCountAsync` en la guarda de dominio. | `histogram_quantile(0.95, rate(kagami_identity_last_admin_check_latency_bucket[5m]))` |
| **Audit Log Write Latency** | Tiempo de inserción del log forense en `audit_logs`. | `histogram_quantile(0.95, rate(kagami_audit_log_write_latency_bucket[5m]))` |
| **Outbox Event Publish Latency** | Tiempo de inserción del evento en `outbox_events` (no incluye publicación a Kafka). | `histogram_quantile(0.95, rate(kagami_outbox_event_insert_latency_bucket[5m]))` |
| **Idempotent Call Rate** | Porcentaje de llamadas de desactivación que son no-op (identidad ya desactivada). | `rate(kagami_identity_deactivation_idempotent_total[5m]) / rate(kagami_identity_deactivation_total[5m])` |
| **Guard Failure Rate** | Porcentaje de desactivaciones bloqueadas por guardas (Self-Deactivation, Last Admin). | `rate(kagami_identity_deactivation_guard_failures_total[5m]) / rate(kagami_identity_deactivation_total[5m])` |

### 2.2. SLOs y SLAs

| Métrica | SLO (Objetivo Interno - 95%) | SLA (Contrato Enterprise - 99%) | Justificación |
| :--- | :--- | :--- | :--- |
| **Latencia (p95)** | **< 100 ms** | **< 250 ms** | Menor que UC-02 porque no hay re-cifrado de PII ni Merge Patch complejo. |
| **Latencia (p99)** | **< 300 ms** | **< 500 ms** | Casos donde la guarda de "Último Admin" requiere consulta a DB con índice no cacheado. |
| **Throughput** | **50 req/seg** | **30 req/seg** | Suficiente para offboarding masivo (180k desactivaciones/hora). |
| **Disponibilidad** | **99.99%** | **99.95%** | 4.38 min vs 21.9 min de downtime permitido al mes. |
| **Tasa de Error (5xx)** | **< 0.1%** | **< 0.5%** | Fallos críticos (DB down, Outbox full). |
| **Last Admin Check Latency (p95)** | **< 20 ms** | **< 50 ms** | Consulta optimizada con índice parcial en `identities(status, tenant_id)`. |

---

## 3. Cuellos de Botella y Estrategias de Optimización

### 3.1. Guarda de "Último Administrador" (CPU-Bound + I/O-Bound)

**Problema:** La validación `IsLastActiveAdminAsync` requiere contar administradores activos en el tenant. Si no hay índice adecuado, esta consulta puede escanear miles de filas.

**Estrategia de Optimización:**

#### Opción A: Índice Parcial Optimizado (Recomendado)

```sql
-- Índice parcial que solo incluye administradores activos
-- Reduce el tamaño del índice en ~95% (solo ~5% de identidades son admins activos)
CREATE INDEX idx_identities_active_admins 
    ON identities (tenant_id) 
    WHERE status = 'ACTIVE' AND has_role = 'admin';

-- Consulta optimizada que usa el índice parcial
SELECT COUNT(*) 
FROM identities 
WHERE tenant_id = @tenantId 
  AND status = 'ACTIVE' 
  AND has_role = 'admin';
```

**Rendimiento Esperado:**
- Sin índice: Full table scan en partición del tenant (~50ms para 1M identidades).
- Con índice parcial: Index-only scan (~2ms para 1M identidades, 50k admins).

#### Opción B: Caché de Conteo de Admins (Para Tenants Grandes)

Para tenants con >100k identidades y >1k admins, podemos cachear el conteo:

```csharp
public class AdminCountCache
{
    private readonly IMemoryCache _cache;
    private readonly IIdentityRepository _repository;
    private const string CACHE_KEY_PREFIX = "admin_count:";
    private static readonly TimeSpan CACHE_TTL = TimeSpan.FromSeconds(30);

    public async Task<int> GetActiveAdminCountAsync(TenantId tenantId)
    {
        var cacheKey = $"{CACHE_KEY_PREFIX}{tenantId.Value}";
        
        if (_cache.TryGetValue(cacheKey, out int cachedCount))
        {
            return cachedCount;
        }

        // Cache miss: consultar DB
        int count = await _repository.GetActiveAdminCountAsync(tenantId);
        
        // Cache con TTL corto para balancear consistencia y rendimiento
        _cache.Set(cacheKey, count, CACHE_TTL);
        
        return count;
    }

    // Invalidar caché cuando se desactiva/activa un admin
    public void InvalidateCache(TenantId tenantId)
    {
        var cacheKey = $"{CACHE_KEY_PREFIX}{tenantId.Value}";
        _cache.Remove(cacheKey);
    }
}
```

**Trade-off:** 
- **Ventaja:** Reduce latencia de 2ms a <1ms para llamadas repetidas.
- **Desventaja:** Ventana de inconsistencia de 30 segundos donde el conteo puede estar desactualizado.
- **Mitigación:** Invalidar caché explícitamente en cada desactivación/activación de admin.

### 3.2. Generación de Auditoría Forense (CPU-Bound)

**Problema:** Serializar el `snapshot_json` con `reason`, `actor`, `timestamp`, y datos enmascarados consume CPU.

**Estrategia de Optimización:**

#### System.Text.Json con Source Generators (.NET 8)

```csharp
// Definir el contexto de serialización con Source Generators
[JsonSerializable(typeof(AuditSnapshotDto))]
public partial class AuditSnapshotContext : JsonSerializerContext
{
}

public class AuditLogger : IAuditLogger
{
    public AuditSnapshot CreateSnapshot(Identity identity, Actor actor, DeactivationReason reason)
    {
        var snapshotDto = new AuditSnapshotDto
        {
            IdentityId = identity.Id.Value,
            TenantId = identity.TenantId.Value,
            Action = "Identity.Deactivated",
            ActorId = actor.Id,
            Reason = reason.Value,
            Timestamp = DateTime.UtcNow,
            PreviousStatus = "ACTIVE", // O el estado anterior real
            NewStatus = "DEACTIVATED"
        };

        // Serialización optimizada con Source Generators (sin reflexión en runtime)
        var snapshotJson = JsonSerializer.Serialize(snapshotDto, AuditSnapshotContext.Default.AuditSnapshotDto);

        return new AuditSnapshot(
            identityId: identity.Id,
            action: snapshotDto.Action,
            actorId: snapshotDto.ActorId,
            reason: snapshotDto.Reason,
            timestamp: snapshotDto.Timestamp,
            snapshotJson: snapshotJson
        );
    }
}
```

**Rendimiento:**
- `JsonSerializer.Serialize` con reflexión: ~5ms para objetos complejos.
- `JsonSerializer.Serialize` con Source Generators: ~0.5ms (10x más rápido).

### 3.3. Propagación Asíncrona de Eventos (I/O-Bound)

**Problema:** Insertar el evento en `outbox_events` y luego publicarlo en Kafka añade latencia.

**Estrategia de Optimización:**

#### Outbox Pattern con Inserción Atómica

```csharp
public async Task<DeactivateIdentityResult> Handle(DeactivateIdentityCommand command, CancellationToken ct)
{
    // ... (Carga, validación de guardas, acción de dominio) ...

    // Preparar evento Outbox
    var outboxEvent = new OutboxEvent(
        eventType: "Identity.Deactivated",
        aggregateId: identity.Id,
        aggregateType: "Identity",
        payload: JsonSerializer.Serialize(new IdentityDeactivatedEvent(
            identity.Id, identity.TenantId, reason, actor, DateTime.UtcNow)),
        status: "PENDING",
        createdAt: DateTime.UtcNow
    );

    // Persistencia atómica (Identity + AuditLog + OutboxEvent en una transacción)
    using var transaction = await _dbContext.Database.BeginTransactionAsync(ct);
    try
    {
        _dbContext.Identities.Update(identity);
        _dbContext.AuditLogs.Add(auditSnapshot);
        _dbContext.OutboxEvents.Add(outboxEvent);
        
        await _dbContext.SaveChangesAsync(ct);
        await transaction.CommitAsync(ct);
    }
    catch (Exception ex)
    {
        await transaction.RollbackAsync(ct);
        _logger.LogError(ex, "Failed to deactivate identity {IdentityId}", command.IdentityId);
        throw;
    }

    // Publicación asíncrona a Kafka (no bloquea la respuesta HTTP)
    _ = Task.Run(async () => 
    {
        try
        {
            await _kafkaProducer.ProduceAsync("identity-events", outboxEvent);
            await _dbContext.OutboxEvents
                .Where(e => e.Id == outboxEvent.Id)
                .ExecuteUpdateAsync(e => e.SetProperty(x => x.Status, "PUBLISHED"));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to publish event to Kafka. Outbox Processor will retry.");
            // El evento permanece en estado PENDING y será procesado por el Outbox Processor
        }
    }, ct);

    return new DeactivateIdentityResult(identity.Id, identity.Status, identity.DeactivatedAt);
}
```

**Rendimiento:**
- Inserción en `outbox_events`: ~5ms (incluida en la transacción atómica).
- Publicación a Kafka: ~10-20ms (asíncrona, no bloquea la respuesta HTTP).
- **Latencia total percibida por el cliente:** ~100ms (sin incluir publicación a Kafka).

---

## 4. Estrategia de Load Testing (Pruebas de Carga)

Utilizamos **k6** para simular escenarios específicos de desactivación que no se cubren en UC-01 o UC-02.

### 4.1. Escenario 1: Offboarding Masivo (Despidos Colectivos)

**Objetivo:** Validar que el sistema maneja 1000 desactivaciones en 10 minutos sin degradar el rendimiento ni corromper datos.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const guardFailures = new Rate('guard_failures');

export const options = {
  scenarios: {
    mass_offboarding: {
      executor: 'constant-vus',
      vus: 50,
      duration: '10m',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<100', 'p(99)<300'],
    http_req_failed: ['rate<0.01'],
    guard_failures: ['rate<0.05'], // < 5% de fallos por guardas (esperado)
  },
};

// Setup: Cargar pool de identidades a desactivar
export function setup() {
  const res = http.get(`${__ENV.BASE_URL}/v1/identities?status=ACTIVE&limit=1000`);
  const identities = res.json('data');
  return { identityPool: identities.map(i => i.id) };
}

export default function (data) {
  // Seleccionar identidad aleatoria del pool
  const identityId = data.identityPool[Math.floor(Math.random() * data.identityPool.length)];
  
  const payload = JSON.stringify({
    reason: `Mass offboarding test - ${Date.now()}`
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
    'is guard failure (400)': (r) => r.status === 400,
    'is not server error': (r) => r.status < 500,
  });

  if (res.status === 400) {
    guardFailures.add(1);
  }

  sleep(0.6); // ~1.6 requests per second per VU (50 VUs * 1.6 = 80 req/seg)
}
```

### 4.2. Escenario 2: Incidente de Seguridad (Desactivación de Emergencia)

**Objetivo:** Simular la desactivación de 100 identidades comprometidas en < 1 minuto.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    security_incident: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '10s', target: 100 }, // Ramp up rápido
        { duration: '50s', target: 100 }, // Mantener 100 VUs
        { duration: '10s', target: 0 },   // Ramp down
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<150'], // Latencia ligeramente mayor aceptable en emergencia
    http_req_failed: ['rate<0.02'],
  },
};

export default function () {
  // Simular desactivación de identidad específica (comprometida)
  const identityId = __ENV.COMPROMISED_IDENTITY_ID;
  
  const payload = JSON.stringify({
    reason: `Security incident - immediate deactivation - ${Date.now()}`
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
    'is not server error': (r) => r.status < 500,
  });

  sleep(0.01); // 100 requests per second per VU
}
```

### 4.3. Escenario 3: Reintentos de SCIM/HRIS (Idempotencia)

**Objetivo:** Validar que reintentos de desactivación no generan eventos duplicados ni degradan el rendimiento.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    scim_retries: {
      executor: 'constant-vus',
      vus: 20,
      duration: '2m',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<50'], // Debe ser muy rápido (no-op)
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const identityId = __ENV.ALREADY_DEACTIVATED_IDENTITY_ID;
  
  const payload = JSON.stringify({
    reason: `SCIM retry - ${Date.now()}`
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
    'is success (200 - idempotent)': (r) => r.status === 200,
    'is not server error': (r) => r.status < 500,
  });

  sleep(0.1); // 10 requests per second per VU
}
```

---

## 5. Capacity Planning y Escalabilidad

### 5.1. Escenarios de Escala

| Escala | Identidades Totales | Desactivaciones/Día | Recursos KAGAMI (API) | Recursos PostgreSQL (DB) | Estrategia de Escalado |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Startup** | 10,000 | 10-50 | 2 Pods (1 vCPU, 2GB RAM) | db.t3.medium (2 vCPU, 4GB RAM, 50GB SSD) | Manual / HPA básico |
| **Scale-up** | 100,000 | 100-500 | 5 Pods (2 vCPU, 4GB RAM) | db.r5.large (2 vCPU, 16GB RAM, 200GB SSD) | HPA basado en CPU |
| **Enterprise** | 1,000,000+ | 1000-5000 | 15+ Pods (4 vCPU, 8GB RAM) | db.r5.xlarge (4 vCPU, 32GB RAM, 1TB SSD) + Read Replicas | **Particionamiento DB + HPA avanzado** |

### 5.2. Cuellos de Botella Predictivos

1. **Índice de `audit_logs`:** Con 5000 desactivaciones/día y un `snapshot_json` promedio de 1KB, se generan ~5MB de logs/día. En 1 año: ~1.8GB.
   - **Solución:** Particionamiento mensual de `audit_logs` y compresión TOAST nativa de PostgreSQL.

2. **Outbox Events Accumulation:** Si Kafka está caído por >1 hora, la tabla `outbox_events` puede crecer significativamente.
   - **Solución:** Monitorear `kagami_outbox_events_pending_count` y alertar si >1000 eventos pendientes.

3. **Last Admin Check en Tenants Grandes:** Para tenants con >10k admins activos, la consulta puede degradarse.
   - **Solución:** Implementar caché de conteo con invalidación explícita (Opción B de la sección 3.1).

---

## 6. Observabilidad y Profiling en Producción

### 6.1. Distributed Tracing (OpenTelemetry)

El Trace de una desactivación debe descomponerse en los siguientes Spans para identificar cuellos de botella:

1. `HTTP POST /identities/{id}/deactivate` (Root Span)
2. `DB Query: Load Identity` (Debe ser < 10ms)
3. `Domain Logic: Validate Guards` (CPU bound, < 5ms)
   - `DB Query: GetActiveAdminCount` (Solo si es admin, < 20ms con índice parcial)
4. `Domain Logic: Execute Deactivate Action` (CPU bound, < 2ms)
5. `Domain Logic: Generate Audit Snapshot` (CPU bound, < 5ms con Source Generators)
6. `DB Transaction: Update Identity + Insert Audit + Insert Outbox` (Debe ser < 50ms)
7. `Kafka: Publish Event` (Asíncrono, no bloquea el Root Span, < 20ms)

### 6.2. Dashboards de Grafana (Específicos para UC-03)

1. **Deactivation Success vs. Guard Failure Ratio:** Un gráfico de líneas que muestra la proporción de HTTP 200 vs HTTP 400 (guard failures). Si la línea de 400 cruza el 10%, se dispara una alerta para revisar si hay un problema de configuración de roles.

2. **Last Admin Check Latency Heatmap:** Muestra si la consulta de conteo de admins se está volviendo más lenta a medida que crece el número de administradores en el tenant.

3. **Outbox Event Accumulation:** Monitorea el conteo de eventos pendientes en `outbox_events`. Si >1000 eventos por >10 minutos, alerta crítica (Kafka puede estar caído).

4. **Mass Deactivation Alert:** Dashboard específico que muestra el rate de desactivaciones por tenant. Si >20 desactivaciones/minuto, alerta crítica al SIEM.

---

## 7. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los objetivos de rendimiento iniciales (< 100ms p95) que este documento formaliza. |
| `09_Modern_Artifacts.md` | Define las métricas de OpenTelemetry; este documento explica cómo usarlas para validar SLAs de guardas y auditoría. |
| `10_UC_Specific_Data_Model.md` | Los índices parciales (`idx_identities_active_admins`, `idx_identities_deactivated_grace_period`) definidos aquí son críticos para cumplir los SLOs de latencia. |
| `11_UC_Specific_Security_Threats.md` | Las mitigaciones de "Last Admin Deactivation" y "Audit Log Tampering" tienen un costo de rendimiento que se mide y gestiona en este documento. |
| `SHUGO_Performance_Testing_Strategy.md` (Transversal) | Este documento aplica la estrategia global al contexto específico de operaciones de desactivación. |

---

## 8. Anexos: Configuración de PostgreSQL para UC-03

### 8.1. Índices Parciales Optimizados

```sql
-- Índice para guarda de "Último Administrador"
CREATE INDEX idx_identities_active_admins 
    ON identities (tenant_id) 
    WHERE status = 'ACTIVE' AND has_role = 'admin';

-- Índice para consultas de Grace Period (UC-05)
CREATE INDEX idx_identities_deactivated_grace_period 
    ON identities (tenant_id, deactivated_at ASC) 
    WHERE status = 'DEACTIVATED';

-- Índice para auditoría forense
CREATE INDEX idx_audit_logs_deactivation_forensic 
    ON audit_logs (tenant_id, actor_id, created_at DESC) 
    WHERE action = 'Identity.Deactivated';
```

### 8.2. Configuración de Autovacuum para `audit_logs`

```sql
-- audit_logs es append-only, no necesita vacuum frecuente
ALTER TABLE audit_logs SET (
    autovacuum_vacuum_scale_factor = 0.1, -- Vaciar cuando 10% de filas cambian (raro)
    autovacuum_analyze_scale_factor = 0.05,
    autovacuum_vacuum_threshold = 1000,
    autovacuum_analyze_threshold = 500
);
```

### 8.3. Configuración de PgBouncer para Desactivaciones

```ini
[databases]
shugo_kagami = host=postgres-primary port=5432 dbname=shugo_kagami

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 50
reserve_pool_size = 20 -- Mayor reserva para picos de offboarding masivo
reserve_pool_timeout = 2
```

---
*Fin del Documento 12_UC_Specific_Performance_Benchmarks para UC-03*