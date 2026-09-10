# UC-02: Actualizar Identidad - Benchmarks, SLAs y Estrategia de Rendimiento

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-02 - Actualizar Identidad |
| **Documento:** | 12_UC_Specific_Performance_Benchmarks.md |
| **Metodología:** | Performance Engineering + Optimistic Concurrency Control (OCC) Profiling + Write Amplification Analysis |
| **Estándares:** | ISO/IEC 25010 (Performance Efficiency), Google SRE (SLI/SLO/SLA) |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 30/08/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 30/08/2026 | Gastón Ruiz | **Creación inicial.** Definición de SLAs, SLOs, estrategia de Load Testing (enfocada en contención de OCC y Merge Patch), y planificación de capacidad para operaciones de lectura-modificación-escritura (Read-Modify-Write). | Documento de referencia para SREs y DBAs. Establece cómo mantener la latencia baja (< 250ms p95) a pesar de la sobrecarga inherente del Control de Concurrencia Optimista, el cálculo de Diffs de auditoría y el re-cifrado condicional de PII. |

---

## 1. Introducción y Propósito

A diferencia del UC-01 (Crear), que es una operación de inserción pura, el **UC-02 (Actualizar)** es una operación compleja de **Lectura-Modificación-Escritura (Read-Modify-Write)**. 

Esta complejidad introduce cuatro desafíos de rendimiento críticos que este documento aborda:
1. **Sobrecarga de OCC:** La necesidad de leer el estado actual y el `RowVersion` antes de escribir.
2. **CPU-Bound JSON Merge Patch:** La fusión profunda de `custom_attributes` (JSONB) consume ciclos de CPU.
3. **Generación de Diffs de Auditoría:** Comparar el estado "Before" y "After" y enmascarar PII añade latencia síncrona.
4. **Contención de Base de Datos:** Múltiples actores actualizando la misma identidad generan conflictos (409) y reintentos.

---

## 2. Service Level Indicators (SLIs) y Objectives (SLOs)

### 2.1. SLIs Específicos para UC-02

| SLI | Descripción | Fórmula / Medición |
| :--- | :--- | :--- |
| **Update Latency** | Tiempo desde que el API Gateway recibe el `PATCH` hasta que responde HTTP 200. | `histogram_quantile(0.95, rate(kagami_identity_update_latency_bucket[5m]))` |
| **OCC Conflict Rate** | Porcentaje de requests que fallan con HTTP 409 debido a conflictos de concurrencia. | `rate(kagami_identity_update_conflicts_total[5m]) / rate(kagami_identity_update_total[5m])` |
| **Audit Diff Generation Time** | Tiempo específico dedicado a calcular el `FieldDiff` y enmascarar PII. | `histogram_quantile(0.95, rate(kagami_audit_diff_latency_bucket[5m]))` |
| **Conditional PII Re-encryption Time** | Tiempo de llamada a KMS *solo* cuando el email/teléfono cambia. | `histogram_quantile(0.95, rate(kagami_pii_re_encryption_latency_bucket[5m]))` |

### 2.2. SLOs y SLAs

| Métrica | SLO (Objetivo Interno - 95%) | SLA (Contrato Enterprise - 99%) | Justificación |
| :--- | :--- | :--- | :--- |
| **Latencia (p95)** | **< 250 ms** | **< 600 ms** | Mayor que UC-01 debido al cálculo de Diffs y validación de OCC. |
| **Latencia (p99)** | **< 800 ms** | **< 1500 ms** | Casos donde ocurre re-cifrado de PII (llamada a KMS) o contención de DB. |
| **Throughput** | **150 req/seg** | **100 req/seg** | Las actualizaciones son más frecuentes que las creaciones en operación diaria. |
| **Tasa de Conflictos (409)** | **< 1%** | **< 5%** | Si supera el 5%, indica un problema de UX (clientes no refrescan el `RowVersion`). |

---

## 3. Cuellos de Botella y Estrategias de Optimización

### 3.1. Contención de OCC (The "Thundering Herd" Problem)

**Problema:** Si 50 administradores intentan actualizar el mismo perfil simultáneamente, 49 recibirán un HTTP 409. Esto genera reintentos automáticos que pueden saturar la API y la DB.

**Estrategia de Optimización:**
1. **Exponential Backoff with Jitter:** Los clientes (SDKs) *deben* implementar reintentos con backoff exponencial (ej. 100ms, 200ms, 400ms) y jitter aleatorio para evitar que todos reintenten al mismo tiempo.
2. **E-Tag Caching en API Gateway:** El API Gateway puede cachear la respuesta `GET /identities/{id}` (incluyendo el `RowVersion`) por 5 segundos. Esto reduce la carga de lectura en PostgreSQL cuando los clientes refrescan el estado antes de reintentar.

### 3.2. Optimización de JSON Merge Patch (CPU-Bound)

**Problema:** Hacer un *Deep Merge* de objetos JSON anidados en C# puede ser lento si se usa reflexión.

**Estrategia de Optimización:**
- **Nativo en PostgreSQL (Opción A - Recomendada para Bulk):** Para actualizaciones masivas, delegar el merge a la DB usando `jsonb_set` y `jsonb_strip_nulls`.
  ```sql
  UPDATE profiles 
  SET custom_attributes = jsonb_strip_nulls(custom_attributes || @patch::jsonb)
  WHERE identity_id = @id;
  ```
- **System.Text.Json con Source Generators (Opción B - Recomendada para API):** Usar `JsonSerializerContext` en .NET 8 para evitar reflexión en tiempo de ejecución durante la deserialización del patch. Reduce la latencia de serialización en un 40%.

### 3.3. Generación de Diffs de Auditoría (Memory-Bound)

**Problema:** Comparar dos diccionarios complejos y enmascarar PII consume memoria y CPU.

**Estrategia de Optimización:**
- **Structural Sharing:** No clonar todo el objeto `Identity` para el estado "Before". Usar un patrón de *Event Sourcing ligero* donde solo se registran los cambios a medida que se aplican en el Aggregate Root.
  ```csharp
  // En lugar de clonar todo el estado:
  public class Profile {
      private readonly List<FieldDiff> _pendingDiffs = new();
      
      public void UpdateDisplayName(string newName) {
          if (this.DisplayName != newName) {
              _pendingDiffs.Add(new FieldDiff("display_name", this.DisplayName, newName));
              this.DisplayName = newName;
          }
      }
  }
  ```
  Esto reduce la huella de memoria en un 90% durante la actualización.

---

## 4. Estrategia de Load Testing (Pruebas de Carga)

Utilizamos **k6** para simular escenarios específicos de actualización que no se cubren en el UC-01.

### 4.1. Escenario 1: "Thundering Herd" en una sola identidad

**Objetivo:** Validar cómo se comporta el sistema cuando 50 usuarios intentan actualizar el *mismo* perfil simultáneamente. Esperamos ver muchos 409, pero **cero 500** y latencia estable.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    single_identity_contention: {
      executor: 'constant-vus',
      vus: 50,
      duration: '2m',
      exec: 'updateSameIdentity',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<300'],
    http_req_failed: ['rate<0.05'], // Aceptamos hasta 5% de fallos (409s), pero no 500s
  },
};

// Setup: Obtener el RowVersion inicial una vez
let currentRowVersion;
export function setup() {
  const res = http.get(`${__ENV.BASE_URL}/v1/identities/${__ENV.TEST_IDENTITY_ID}`);
  return { rowVersion: res.json('row_version') };
}

export function updateSameIdentity(data) {
  // Intentar actualizar con el RowVersion conocido
  const res = http.patch(
    `${__ENV.BASE_URL}/v1/identities/${__ENV.TEST_IDENTITY_ID}`,
    JSON.stringify({ profile: { display_name: `Updated_${Date.now()}` } }),
    {
      headers: {
        'Content-Type': 'application/merge-patch+json',
        'Authorization': `Bearer ${__ENV.JWT_TOKEN}`,
        'If-Match': data.rowVersion, // Todos usan la misma versión inicial -> 49 fallarán
      },
    }
  );

  // Validar que o bien tuvo éxito (200) o fue un conflicto esperado (409)
  check(res, {
    'is success or conflict': (r) => r.status === 200 || r.status === 409,
    'no server errors': (r) => r.status < 500,
  });

  // Si tuvo éxito, actualizar el RowVersion para el siguiente intento (simulación de cliente inteligente)
  if (res.status === 200) {
    data.rowVersion = res.json('row_version');
  }
}
```

### 4.2. Escenario 2: Bulk Update de Custom Attributes

**Objetivo:** Medir el rendimiento al actualizar campos JSONB en 10,000 identidades diferentes.

*   **Métrica clave:** Throughput de escritura en PostgreSQL y uso de CPU por el operador `||` (concatenación JSONB).
*   **Esperado:** < 50ms p95 por actualización individual.

---

## 5. Capacity Planning y Escalabilidad

### 5.1. Write Amplification (Amplificación de Escritura)

Cada `PATCH` exitoso en UC-02 genera **3 escrituras físicas** en PostgreSQL:
1. `UPDATE identities` (Cambio de `row_version`).
2. `UPDATE profiles` (Cambio de datos).
3. `INSERT audit_logs` (Registro forense).

**Impacto en Storage:**
- Para 1 millón de actualizaciones/mes, con un `snapshot_json` promedio de 2KB, se generan **~2GB de datos de auditoría/mes**.
- **Solución:** Particionamiento mensual de `audit_logs` y compresión TOAST nativa de PostgreSQL.

### 5.2. Ajuste de PostgreSQL para HOT Updates

Dado que la tabla `identities` se actualiza frecuentemente, debemos favorecer las **HOT (Heap Only Tuple) Updates** para evitar la fragmentación de índices.

```sql
-- Reducir el fillfactor para dejar espacio libre en las páginas de disco
-- Esto permite que las actualizaciones de 'row_version' ocurran en la misma página
ALTER TABLE identities SET (fillfactor = 85);

-- Asegurar que autovacuum sea agresivo en esta tabla
ALTER TABLE identities SET (
    autovacuum_vacuum_scale_factor = 0.02,
    autovacuum_analyze_scale_factor = 0.01
);
```

---

## 6. Observabilidad y Profiling en Producción

### 6.1. Distributed Tracing (OpenTelemetry)

El Trace de una actualización debe descomponerse en los siguientes Spans para identificar cuellos de botella:

1. `HTTP PATCH /identities/{id}` (Root Span)
2. `DB Query: Load Identity & RowVersion` (Debe ser < 10ms)
3. `Domain Logic: Apply Merge Patch & Validate Scope` (CPU bound, < 5ms)
4. `Domain Logic: Generate Audit Diff & Mask PII` (CPU/Memory bound, < 15ms)
5. `KMS Call: Conditional Re-encryption` (Solo si PII cambia, < 50ms)
6. `DB Transaction: OCC Update & Insert Audit` (Debe ser < 50ms)

### 6.2. Dashboards de Grafana (Específicos para UC-02)

1. **Update Success vs. Conflict Ratio:** Un gráfico de líneas que muestra la proporción de HTTP 200 vs HTTP 409. Si la línea de 409 cruza el 5%, se dispara una alerta para revisar el comportamiento del cliente (SDK).
2. **Audit Diff Latency Heatmap:** Muestra si el enmascaramiento de PII se está volviendo más lento a medida que los perfiles crecen en tamaño (más `custom_attributes`).
3. **KMS Invocation Rate (Conditional):** Muestra cuántas actualizaciones *realmente* requirieron llamar a la KMS. Debería ser < 10% del total de updates.

---

## 7. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los objetivos de rendimiento iniciales (< 250ms p95) que este documento formaliza. |
| `09_Modern_Artifacts.md` | Define las métricas de OpenTelemetry; este documento explica cómo usarlas para validar SLAs de concurrencia. |
| `10_UC_Specific_Data_Model.md` | La configuración de `fillfactor` y HOT Updates definida aquí es crítica para cumplir los SLOs de latencia de escritura. |
| `11_UC_Specific_Security_Threats.md` | La mitigación de "Lost Updates" (OCC) tiene un costo de rendimiento (conflictos 409) que se mide y gestiona en este documento. |
| `SHUGO_Performance_Testing_Strategy.md` (Transversal) | Este documento aplica la estrategia global al contexto específico de operaciones de actualización. |

---

## 8. Anexos: Configuración de PgBouncer para Updates

Las actualizaciones mantienen transacciones abiertas por más tiempo que las lecturas. Es vital configurar PgBouncer correctamente para evitar agotar las conexiones.

```ini
[databases]
shugo_kagami = host=postgres-primary port=5432 dbname=shugo_kagami

[pgbouncer]
# CRÍTICO: Transaction mode es obligatorio para UC-02.
# En Session mode, una conexión se bloquea hasta que el cliente hace COMMIT/ROLLBACK.
# En Transaction mode, PgBouncer puede reutilizar la conexión para otros clientes entre statements.
pool_mode = transaction 

# Aumentar el pool de reserva para manejar picos de reintentos por conflictos OCC
reserve_pool_size = 20 
reserve_pool_timeout = 2
```

---
*Fin del Documento 12_UC_Specific_Performance_Benchmarks para UC-02*