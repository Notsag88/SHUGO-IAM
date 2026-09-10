# UC-04: Consultar Identidad - Benchmarks, SLAs y Estrategia de Rendimiento

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-04 - Consultar Identidad |
| **Documento:** | 12_UC_Specific_Performance_Benchmarks.md |
| **Metodología:** | Performance Engineering + CQRS Query Optimization + Cache-Aside Tuning + Read Replica Scaling |
| **Estándares:** | ISO/IEC 25010 (Performance Efficiency), Google SRE (SLI/SLO/SLA), Redis Best Practices, PostgreSQL Tuning Guide |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 02/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 02/09/2026 | Gastón Ruiz | **Creación inicial.** Definición de SLAs, SLOs, estrategia de Load Testing (enfocada en alto throughput, cache hit rate, y fallback scenarios), y planificación de capacidad para operaciones de lectura de alto volumen con patrón Cache-Aside y Read Replicas. | Documento de referencia para SREs y Arquitectos de Rendimiento. Establece cómo mantener la latencia baja (< 5ms cache hit, < 50ms DB query) a pesar de la sobrecarga inherente del enmascaramiento de PII, la validación de tenant isolation, y los fallbacks de resiliencia, mientras se soporta 500+ req/seg con >80% cache hit rate. |

---

## 1. Introducción y Propósito

A diferencia de los UCs de escritura (UC-01, UC-02, UC-03) que son operaciones de bajo volumen y alta criticidad, el **UC-04 (Consultar Identidad)** es una operación de **alto volumen y baja latencia** con características de rendimiento únicas:

1. **Alto Throughput:** Las lecturas son ~10-50x más frecuentes que las escrituras. El sistema debe soportar 500+ req/seg de forma sostenida.
2. **Cache-Aside Pattern:** El rendimiento depende críticamente del cache hit rate (>80% objetivo). Un cache hit debe ser < 5ms, mientras que un cache miss puede ser < 50ms.
3. **Múltiples Fuentes de Datos:** El sistema puede servir datos desde Redis (caché), PostgreSQL (primary/replica), o caché stale, cada uno con diferentes perfiles de latencia.
4. **Enmascaramiento de PII:** El procesamiento adicional para Self-Service (enmascaramiento de email/teléfono) añade latencia computacional.
5. **Fallbacks de Resiliencia:** Los escenarios de degradación (Redis down, PostgreSQL down) deben mantener el servicio operativo, aunque con latencia incrementada.
6. **Tenant Isolation:** La validación de tenant isolation en cada consulta añade overhead que debe ser minimizado.

Este documento aborda estos desafíos y define estrategias de optimización, SLAs realistas, y pruebas de carga específicas para garantizar que el UC-04 cumpla con los requisitos de rendimiento enterprise.

---

## 2. Service Level Indicators (SLIs) y Objectives (SLOs)

### 2.1. SLIs Específicos para UC-04

| SLI | Descripción | Fórmula / Medición |
| :--- | :--- | :--- |
| **Query Latency (Cache Hit)** | Tiempo de respuesta cuando los datos se sirven desde caché Redis. | `histogram_quantile(0.95, rate(kagami_identity_query_latency_bucket{data_source="cache"}[5m]))` |
| **Query Latency (DB Query)** | Tiempo de respuesta cuando los datos se sirven desde PostgreSQL (cache miss). | `histogram_quantile(0.95, rate(kagami_identity_query_latency_bucket{data_source="database"}[5m]))` |
| **Query Latency (Stale Cache)** | Tiempo de respuesta cuando se sirven datos stale (PostgreSQL down). | `histogram_quantile(0.95, rate(kagami_identity_query_latency_bucket{data_source="cache_stale"}[5m]))` |
| **Cache Hit Rate** | Porcentaje de consultas servidas desde caché Redis. | `rate(kagami_identity_cache_hit_total[5m]) / (rate(kagami_identity_cache_hit_total[5m]) + rate(kagami_identity_cache_miss_total[5m]))` |
| **Redis Query Latency** | Tiempo específico de consulta a Redis (GET/SET). | `histogram_quantile(0.95, rate(kagami_identity_redis_query_latency_bucket[5m]))` |
| **PostgreSQL Query Latency** | Tiempo específico de consulta a PostgreSQL. | `histogram_quantile(0.95, rate(kagami_identity_postgres_query_latency_bucket[5m]))` |
| **PII Masking Latency** | Tiempo dedicado al enmascaramiento de PII para Self-Service. | `histogram_quantile(0.95, rate(kagami_identity_pii_masking_latency_bucket[5m]))` |
| **Throughput** | Número de consultas de identidad procesadas por segundo. | `rate(kagami_identity_query_total[1m])` |
| **Fallback Rate** | Porcentaje de consultas que activan fallbacks (Redis→DB o DB→Stale). | `rate(kagami_identity_fallback_triggered_total[5m]) / rate(kagami_identity_query_total[5m])` |
| **Error Rate** | Porcentaje de consultas que resultan en error (5xx). | `rate(kagami_identity_query_errors_total[5m]) / rate(kagami_identity_query_total[5m])` |

### 2.2. SLOs y SLAs

| Métrica | SLO (Objetivo Interno - 95%) | SLA (Contrato Enterprise - 99%) | Justificación |
| :--- | :--- | :--- | :--- |
| **Latencia Cache Hit (p95)** | **< 5 ms** | **< 10 ms** | Redis es extremadamente rápido. 5ms incluye red + deserialización JSON. |
| **Latencia DB Query (p95)** | **< 50 ms** | **< 100 ms** | PostgreSQL con índices optimizados y Read Replicas. Incluye JOIN con profiles. |
| **Latencia Stale Cache (p95)** | **< 15 ms** | **< 30 ms** | Similar a cache hit, pero con datos potencialmente desactualizados. |
| **Latencia End-to-End (p95)** | **< 50 ms** | **< 100 ms** | Combinación de cache hit/miss + enmascaramiento PII + validaciones. |
| **Cache Hit Rate** | **> 80%** | **> 70%** | Objetivo realista considerando invalidaciones por escrituras y TTL de 60s. |
| **Throughput** | **500 req/seg** | **300 req/seg** | Suficiente para soportar 10x el volumen de escrituras (~50 req/seg). |
| **Disponibilidad** | **99.99%** | **99.95%** | 4.38 min vs 21.9 min de downtime permitido al mes. |
| **Tasa de Error (5xx)** | **< 0.1%** | **< 0.5%** | Fallos críticos (Redis + PostgreSQL caídos simultáneamente). |
| **PII Masking Latency (p95)** | **< 2 ms** | **< 5 ms** | Enmascaramiento es computacionalmente ligero (string manipulation). |
| **Fallback Rate** | **< 5%** | **< 10%** | Fallbacks deben ser raros en operación normal. |

### 2.3. Error Budgets

| SLO | Error Budget Mensual | Acción si se Agota |
| :--- | :--- | :--- |
| **Disponibilidad 99.99%** | 4.38 minutos | Detener despliegues no críticos, enfocarse en estabilidad. |
| **Latencia p95 < 50ms** | 219 minutos (3.65 horas) | Investigar cuellos de botella, optimizar queries, escalar recursos. |
| **Cache Hit Rate > 80%** | N/A (métrica de calidad) | Revisar TTL, patrones de acceso, invalidación excesiva. |
| **Error Rate < 0.1%** | 4.38 minutos | Investigar errores, revisar logs, corregir bugs. |

---

## 3. Cuellos de Botella y Estrategias de Optimización

### 3.1. Cache Miss y Thundering Herd Problem

**Problema:** Cuando múltiples Query Handlers llegan simultáneamente a un cache miss, todos consultan PostgreSQL, causando una carga innecesaria y latencia incrementada.

**Estrategia de Optimización:**

#### Distributed Locks con Redis SETNX

```csharp
public class QueryHandler : IQueryHandler
{
    private readonly IDistributedLockManager _lockManager;
    private readonly ILogger<QueryHandler> _logger;

    public async Task<IdentityQueryResult> Execute(IdentityQuery query, Actor actor, CancellationToken ct)
    {
        var cachedIdentity = await _cacheManager.GetAsync(query.TenantId, query.IdentityId.Value, ct);
        
        if (cachedIdentity != null)
        {
            return new IdentityQueryResult(cachedIdentity, "cache", actor.IsSelfService);
        }

        // Cache miss: intentar adquirir lock para consultar PostgreSQL
        var lockKey = $"lock:identity:{query.TenantId.Value}:{query.IdentityId.Value}";
        var lockAcquired = await _lockManager.TryAcquireAsync(lockKey, TimeSpan.FromSeconds(5), ct);
        
        if (lockAcquired)
        {
            try
            {
                // Double-check: verificar si alguien más ya populó el caché
                cachedIdentity = await _cacheManager.GetAsync(query.TenantId, query.IdentityId.Value, ct);
                
                if (cachedIdentity != null)
                {
                    return new IdentityQueryResult(cachedIdentity, "cache", actor.IsSelfService);
                }

                // Consultar PostgreSQL (solo un handler lo hace)
                var identity = await _repository.GetByIdAsync(query.TenantId, query.IdentityId.Value, ct);
                
                if (identity == null)
                {
                    throw new IdentityNotFoundException(query.IdentityId.Value);
                }

                // Popular caché
                await _cacheManager.SetAsync(query.TenantId, query.IdentityId.Value, identity, 
                    TimeSpan.FromSeconds(60), ct);

                return new IdentityQueryResult(identity, "database", actor.IsSelfService);
            }
            finally
            {
                await _lockManager.ReleaseAsync(lockKey, ct);
            }
        }
        else
        {
            // No se pudo adquirir el lock: esperar y reintentar
            await Task.Delay(50, ct);
            
            cachedIdentity = await _cacheManager.GetAsync(query.TenantId, query.IdentityId.Value, ct);
            
            if (cachedIdentity != null)
            {
                return new IdentityQueryResult(cachedIdentity, "cache", actor.IsSelfService);
            }

            // Fallback: consultar PostgreSQL directamente (evita timeout infinito)
            var identity = await _repository.GetByIdAsync(query.TenantId, query.IdentityId.Value, ct);
            
            if (identity == null)
            {
                throw new IdentityNotFoundException(query.IdentityId.Value);
            }

            return new IdentityQueryResult(identity, "database", actor.IsSelfService);
        }
    }
}
```

**Impacto:**
- **Sin locks:** 100 Query Handlers consultan PostgreSQL simultáneamente → 100 queries a DB.
- **Con locks:** 1 Query Handler consulta PostgreSQL, los demás esperan → 1 query a DB.
- **Mejora:** 99% reducción en carga de PostgreSQL durante cache stampede.

### 3.2. Covering Indexes para Evitar Table Lookups

**Problema:** Las queries que necesitan columnas no incluidas en el índice requieren un table lookup adicional, incrementando la latencia.

**Estrategia de Optimización:**

```sql
-- Índice sin covering (requiere table lookup)
CREATE INDEX idx_identities_tenant_status ON identities(tenant_id, status);

-- Consulta: necesita id, created_at (no están en el índice)
SELECT id, tenant_id, status, created_at 
FROM identities 
WHERE tenant_id = $1 AND status = 'ACTIVE';
-- Resultado: Index Scan + Table Lookup (2 I/O operations)

-- Índice con covering (INCLUDE)
CREATE INDEX idx_identities_tenant_status_covering 
ON identities(tenant_id, status)
INCLUDE (id, created_at, type, row_version);

-- Misma consulta
SELECT id, tenant_id, status, created_at 
FROM identities 
WHERE tenant_id = $1 AND status = 'ACTIVE';
-- Resultado: Index-Only Scan (1 I/O operation)
```

**Impacto:**
- **Sin covering:** ~15ms (Index Scan + Table Lookup).
- **Con covering:** ~5ms (Index-Only Scan).
- **Mejora:** 66% reducción en latencia de queries.

### 3.3. Connection Pooling Optimizado para Lecturas

**Problema:** Abrir/cerrar conexiones a PostgreSQL para cada query añade overhead significativo.

**Estrategia de Optimización:**

#### PgBouncer Configuration

```ini
# pgbouncer.ini optimizado para lecturas de alto volumen
[databases]
shugo_kagami_replica = host=postgres-replica port=5432 dbname=shugo_kagami

[pgbouncer]
listen_port = 6432
listen_addr = 0.0.0.0
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt

# Transaction mode: reutiliza conexiones entre transacciones
pool_mode = transaction

# Conexiones máximas
max_client_conn = 2000          # Conexiones desde aplicación
default_pool_size = 100         # Conexiones por usuario/DB
min_pool_size = 20              # Mantener conexiones calientes
reserve_pool_size = 50          # Para picos de carga
reserve_pool_timeout = 3        # Segundos antes de usar reserve pool

# Timeouts
server_connect_timeout = 5      # Segundos
server_login_retry = 1          # Segundos
query_timeout = 10              # Segundos (cancela queries lentas)
client_idle_timeout = 300       # Segundos

# Logging
log_connections = 0
log_disconnections = 0
log_pooler_errors = 1

# Statistics
stats_period = 60               # Segundos
```

**Impacto:**
- **Sin pooling:** ~5ms overhead por conexión (TCP handshake + auth).
- **Con pooling:** ~0.1ms overhead (conexión reutilizada).
- **Mejora:** 98% reducción en overhead de conexión.

### 3.4. Query Optimization para Listados Paginados

**Problema:** Las queries con `OFFSET` grande son lentas porque PostgreSQL debe escanear y descartar filas.

**Estrategia de Optimización:**

#### Cursor-Based Pagination (Keyset Pagination)

```sql
-- Offset-based pagination (lento para páginas profundas)
SELECT id, tenant_id, status, created_at
FROM identities
WHERE tenant_id = $1
ORDER BY created_at DESC
LIMIT 20 OFFSET 10000;
-- Resultado: ~200ms (escanea 10,020 filas, descarta 10,000)

-- Cursor-based pagination (rápido para páginas profundas)
SELECT id, tenant_id, status, created_at
FROM identities
WHERE tenant_id = $1
  AND created_at < $2  -- Cursor: último created_at de la página anterior
ORDER BY created_at DESC
LIMIT 20;
-- Resultado: ~5ms (usa índice, no escanea filas innecesarias)
```

**Implementación en C#:**

```csharp
public class IdentityRepositoryImpl : IIdentityRepository
{
    public async Task<PagedResult<Identity>> GetFilteredAsync(IdentityQuery query, CancellationToken ct)
    {
        var dbQuery = _replicaDbContext.Identities
            .Include(i => i.Profile)
            .AsNoTracking()
            .Where(i => i.TenantId == query.TenantId.Value);

        // Aplicar filtros
        if (query.Status.HasValue)
        {
            dbQuery = dbQuery.Where(i => i.Status == query.Status.Value);
        }

        // Cursor-based pagination si se proporciona cursor
        if (!string.IsNullOrEmpty(query.Cursor))
        {
            var cursorDate = DateTime.Parse(query.Cursor);
            dbQuery = dbQuery.Where(i => i.CreatedAt < cursorDate);
        }

        var totalCount = await dbQuery.CountAsync(ct);

        var items = await dbQuery
            .OrderByDescending(i => i.CreatedAt)
            .Take(query.Limit)
            .ToListAsync(ct);

        var metadata = new PaginationMetadata(
            totalCount: totalCount,
            page: query.Page,
            limit: query.Limit,
            nextCursor: items.LastOrDefault()?.CreatedAt.ToString("O")
        );

        return new PagedResult<Identity>(items, metadata);
    }
}
```

**Impacto:**
- **Offset-based (página 500):** ~200ms.
- **Cursor-based (página 500):** ~5ms.
- **Mejora:** 97.5% reducción en latencia para páginas profundas.

### 3.5. Redis Optimization para Cache-Aside

**Problema:** Configuración subóptima de Redis puede limitar el throughput y aumentar la latencia.

**Estrategia de Optimización:**

#### Redis Configuration

```ini
# redis.conf optimizado para cache de identidades
# Memoria
maxmemory 4gb                    # Ajustar según RAM disponible
maxmemory-policy allkeys-lru     # Evicción LRU cuando se llena

# Persistencia (deshabilitar para caché)
save ""                          # No guardar snapshots
appendonly no                    # No usar AOF

# Red
timeout 300                      # Cerrar conexiones inactivas después de 5 min
tcp-keepalive 60                 # Detectar conexiones muertas
tcp-backlog 511                  # Cola de conexiones pendientes

# Rendimiento
hz 100                           # Frecuencia de background tasks (default: 10)
dynamic-hz yes                   # Ajustar hz dinámicamente
aof-rewrite-incremental-fsync yes
rdb-save-incremental-fsync yes

# Clientes
maxclients 10000                 # Conexiones máximas simultáneas

# Logging
loglevel notice                  # Menos logging = más rendimiento
```

**Impacto:**
- **Con persistencia habilitada:** ~10ms latencia (I/O en disco).
- **Sin persistencia:** ~1ms latencia (solo memoria).
- **Mejora:** 90% reducción en latencia de Redis.

### 3.6. Read Replica Load Balancing

**Problema:** Múltiples Query Handlers consultando la misma Read Replica puede saturarla.

**Estrategia de Optimización:**

#### PgBouncer con Múltiples Replicas

```ini
# pgbouncer.ini con load balancing entre replicas
[databases]
shugo_kagami_replica_1 = host=postgres-replica-1 port=5432 dbname=shugo_kagami
shugo_kagami_replica_2 = host=postgres-replica-2 port=5432 dbname=shugo_kagami
shugo_kagami_replica_3 = host=postgres-replica-3 port=5432 dbname=shugo_kagami

[pgbouncer]
pool_mode = transaction

# Distribuir conexiones entre replicas
[databases_replicas]
shugo_kagami_replica = host=postgres-replica-lb port=5432 dbname=shugo_kagami
max_client_conn = 3000
default_pool_size = 300
```

**Load Balancer Configuration (HAProxy):**

```haproxy
frontend postgres_replicas
    bind *:5432
    mode tcp
    default_backend postgres_replicas_backend

backend postgres_replicas_backend
    mode tcp
    balance leastconn            # Distribuir por menor carga
    option pgsql-check user pg_haproxy  # Health checks específicos de PostgreSQL
    
    server replica-1 postgres-replica-1:5432 check inter 5s fall 3 rise 2
    server replica-2 postgres-replica-2:5432 check inter 5s fall 3 rise 2
    server replica-3 postgres-replica-3:5432 check inter 5s fall 3 rise 2
```

**Impacto:**
- **Sin load balancing:** 1 replica saturada, otras ociosas.
- **Con load balancing:** Carga distribuida uniformemente.
- **Mejora:** 3x incremento en throughput total de lecturas.

---

## 4. Estrategia de Load Testing (Pruebas de Carga)

Utilizamos **k6** para simular escenarios específicos de UC-04 que no se cubren en los UCs de escritura.

### 4.1. Escenario 1: Alto Throughput con Cache Hit Rate Alto

**Objetivo:** Validar que el sistema soporta 500 req/seg con >80% cache hit rate.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const cacheHitRate = new Rate('cache_hit_rate');
const queryLatency = new Trend('query_latency');

export const options = {
  scenarios: {
    high_throughput_reads: {
      executor: 'constant-vus',
      vus: 100,
      duration: '10m',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<50', 'p(99)<100'],
    http_req_failed: ['rate<0.001'],
    cache_hit_rate: ['rate>0.80'],  // >80% cache hit rate
    query_latency: ['p(95)<50'],
  },
};

// Setup: Precargar caché con identidades de prueba
export function setup() {
  const identities = [];
  for (let i = 0; i < 1000; i++) {
    identities.push(`identity-${i}`);
  }
  return { identityPool: identities };
}

export default function (data) {
  const identityId = data.identityPool[Math.floor(Math.random() * data.identityPool.length)];
  
  const res = http.get(
    `${__ENV.BASE_URL}/v1/identities/${identityId}`,
    {
      headers: {
        'Authorization': `Bearer ${__ENV.JWT_TOKEN}`,
      },
      tags: { name: 'GetIdentity' },
    }
  );

  const isSuccess = check(res, {
    'status is 200': (r) => r.status === 200,
    'has X-Data-Source header': (r) => r.headers['X-Data-Source'] !== undefined,
  });

  if (isSuccess) {
    const dataSource = res.headers['X-Data-Source'];
    const isCacheHit = dataSource === 'cache';
    cacheHitRate.add(isCacheHit ? 1 : 0);
    queryLatency.add(res.timings.duration);
  }

  sleep(0.2); // 5 requests per second per VU (100 VUs * 5 = 500 req/seg)
}
```

**Resultados Esperados:**
- **Throughput:** 500 req/seg.
- **Cache Hit Rate:** >80%.
- **Latencia p95:** <50ms.
- **Error Rate:** <0.1%.

### 4.2. Escenario 2: Cache Stampede (Thundering Herd)

**Objetivo:** Validar que los distributed locks previenen la sobrecarga de PostgreSQL durante cache misses masivos.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Counter } from 'k6/metrics';

const dbQueries = new Counter('db_queries');

export const options = {
  scenarios: {
    cache_stampede: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '30s', target: 200 },  // Ramp up rápido
        { duration: '1m', target: 200 },   // Mantener 200 VUs
        { duration: '30s', target: 0 },    // Ramp down
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<100'],
    http_req_failed: ['rate<0.01'],
    db_queries: ['count<50'],  // Menos de 50 queries a DB (gracias a locks)
  },
};

export default function () {
  // Todos los VUs consultan la misma identidad (cache miss simultáneo)
  const identityId = 'identity-stampede-test';
  
  const res = http.get(
    `${__ENV.BASE_URL}/v1/identities/${identityId}`,
    {
      headers: {
        'Authorization': `Bearer ${__ENV.JWT_TOKEN}`,
      },
    }
  );

  check(res, {
    'status is 200 or 404': (r) => r.status === 200 || r.status === 404,
    'latency < 100ms': (r) => r.timings.duration < 100,
  });

  // Contar queries a DB (basado en header X-Data-Source)
  if (res.headers['X-Data-Source'] === 'database') {
    dbQueries.add(1);
  }

  sleep(0.05); // 20 requests per second per VU
}
```

**Resultados Esperados:**
- **Sin locks:** 200 queries a DB (todos los VUs consultan PostgreSQL).
- **Con locks:** <50 queries a DB (solo 1 VU consulta PostgreSQL, los demás esperan).
- **Mejora:** 75% reducción en carga de PostgreSQL.

### 4.3. Escenario 3: Fallback Scenarios (Degradación Graceful)

**Objetivo:** Validar que el sistema mantiene operatividad durante fallos de Redis o PostgreSQL.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const staleCacheRate = new Rate('stale_cache_rate');

export const options = {
  scenarios: {
    fallback_test: {
      executor: 'constant-vus',
      vus: 50,
      duration: '5m',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<100'],
    http_req_failed: ['rate<0.05'],  // Aceptar hasta 5% de fallos durante degradación
    stale_cache_rate: ['rate>0'],   // Debe servir datos stale cuando DB está caído
  },
};

export default function () {
  const identityId = `identity-${Math.floor(Math.random() * 1000)}`;
  
  const res = http.get(
    `${__ENV.BASE_URL}/v1/identities/${identityId}`,
    {
      headers: {
        'Authorization': `Bearer ${__ENV.JWT_TOKEN}`,
      },
    }
  );

  const isSuccess = check(res, {
    'status is 200, 404, or 503': (r) => [200, 404, 503].includes(r.status),
  });

  if (isSuccess) {
    const dataSource = res.headers['X-Data-Source'];
    
    if (dataSource === 'cache_stale') {
      staleCacheRate.add(1);
    }
  }

  sleep(0.2); // 5 requests per second per VU
}
```

**Sub-escenarios:**

1. **Redis Down:** PostgreSQL debe servir todas las consultas directamente.
2. **PostgreSQL Down:** Redis debe servir datos stale (si están disponibles).
3. **Ambos Down:** Retornar HTTP 503 Service Unavailable.

**Resultados Esperados:**
- **Redis Down:** Latencia incrementa a ~50ms (DB query), pero servicio operativo.
- **PostgreSQL Down:** Latencia ~10ms (stale cache), servicio operativo con datos desactualizados.
- **Ambos Down:** HTTP 503, servicio no disponible (esperado).

### 4.4. Escenario 4: Paginación Profunda con Cursor

**Objetivo:** Validar que cursor-based pagination mantiene latencia baja incluso en páginas profundas.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Trend } from 'k6/metrics';

const paginationLatency = new Trend('pagination_latency');

export const options = {
  scenarios: {
    deep_pagination: {
      executor: 'constant-vus',
      vus: 20,
      duration: '5m',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<50'],
    pagination_latency: ['p(95)<50'],  // Latencia constante independientemente de página
  },
};

export default function () {
  let cursor = null;
  const pagesToTest = 100;  // Probar hasta página 100
  
  for (let page = 0; page < pagesToTest; page++) {
    let url = `${__ENV.BASE_URL}/v1/identities?limit=20&status=ACTIVE`;
    
    if (cursor) {
      url += `&cursor=${cursor}`;
    }
    
    const res = http.get(url, {
      headers: {
        'Authorization': `Bearer ${__ENV.JWT_TOKEN}`,
      },
    });

    check(res, {
      'status is 200': (r) => r.status === 200,
      'latency < 50ms': (r) => r.timings.duration < 50,
    });

    paginationLatency.add(res.timings.duration);

    // Extraer cursor de la respuesta
    const body = JSON.parse(res.body);
    cursor = body.pagination?.next_cursor;
    
    if (!cursor) {
      break;  // No hay más páginas
    }

    sleep(0.1);
  }
}
```

**Resultados Esperados:**
- **Offset-based (página 100):** ~200ms.
- **Cursor-based (página 100):** ~5ms.
- **Mejora:** 97.5% reducción en latencia para páginas profundas.

---

## 5. Capacity Planning y Escalabilidad

### 5.1. Escenarios de Escala

| Escala | Identidades Totales | Throughput Pico | Recursos KAGAMI (Query API) | Recursos Redis | Recursos PostgreSQL (Read Replicas) | Estrategia de Escalado |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Startup** | 10,000 | 100 req/seg | 2 Pods (1 vCPU, 2GB RAM) | 1 nodo (2GB RAM) | 1 Primary + 1 Replica (2 vCPU, 4GB RAM) | Manual / HPA básico |
| **Scale-up** | 100,000 | 300 req/seg | 5 Pods (2 vCPU, 4GB RAM) | 1 nodo (8GB RAM) | 1 Primary + 2 Replicas (4 vCPU, 8GB RAM) | HPA basado en CPU |
| **Enterprise** | 1,000,000+ | 500+ req/seg | 15+ Pods (4 vCPU, 8GB RAM) | Redis Cluster (3 nodos, 16GB RAM) | 1 Primary + 3 Replicas (8 vCPU, 16GB RAM) | **Particionamiento DB + Redis Cluster + HPA avanzado** |

### 5.2. Cuellos de Botella Predictivos

#### Redis Memory Saturation

**Problema:** Con 1M de identidades y TTL de 60s, Redis puede llenarse rápidamente.

**Cálculo:**
- Tamaño promedio de identidad en caché: ~2KB (JSON serializado).
- Identidades en caché simultáneamente: ~50,000 (asumiendo 500 req/seg * 60s TTL * 10% únicas).
- Memoria requerida: 50,000 * 2KB = ~100MB.

**Solución:**
```ini
# redis.conf
maxmemory 4gb                    # 4GB para caché (suficiente para 2M identidades)
maxmemory-policy allkeys-lru     # Evicción LRU cuando se llena
```

#### PostgreSQL Connection Exhaustion

**Problema:** Múltiples Query Handlers pueden agotar las conexiones de PostgreSQL.

**Cálculo:**
- Query Handlers: 15 pods.
- Conexiones por pod: 50 (pool size).
- Total conexiones: 15 * 50 = 750.

**Solución:**
```ini
# pgbouncer.ini
max_client_conn = 2000           # Suficiente para 15 pods * 50 conexiones + overhead
default_pool_size = 100          # Conexiones por usuario/DB
```

#### Read Replica Lag

**Problema:** Las Read Replicas pueden tener lag respecto al Primary, sirviendo datos desactualizados.

**Monitoreo:**
```sql
-- Verificar lag de replicación
SELECT client_addr, state, sync_state, 
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

**Solución:**
- Configurar `synchronous_commit = on` en Primary para replicación síncrona (si la consistencia es crítica).
- O aceptar lag de ~1-2 segundos y usar cache hit para datos frescos.

### 5.3. Auto-Scaling Policies

#### Kubernetes HPA para Query Handlers

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kagami-query-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kagami-query
  minReplicas: 3
  maxReplicas: 20
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
          name: kagami_identity_query_latency
        target:
          type: AverageValue
          averageValue: "50"  # Escalar si latencia p95 > 50ms
    - type: External
      external:
        metric:
          name: kagami_identity_throughput
        target:
          type: AverageValue
          averageValue: "100"  # Escalar si throughput > 100 req/seg por pod
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60  # Esperar 1 min antes de escalar up
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Esperar 5 min antes de escalar down
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

---

## 6. Observabilidad y Profiling en Producción

### 6.1. Distributed Tracing (OpenTelemetry)

El Trace de una consulta debe descomponerse en los siguientes Spans para identificar cuellos de botella:

```
HTTP GET /identities/{id} (Root Span)
├── Cache Lookup (Redis GET)
│   ├── [Hit] Return cached data
│   └── [Miss] Continue to DB
├── DB Query (PostgreSQL SELECT)
│   ├── Acquire lock (if cache miss)
│   ├── Execute query
│   └── Release lock
├── PII Masking (if Self-Service)
└── Cache Update (Redis SET, if cache miss)
```

**Spans Críticos:**
1. `Cache Lookup`: Debe ser < 2ms.
2. `DB Query`: Debe ser < 20ms (con índices optimizados).
3. `PII Masking`: Debe ser < 1ms.
4. `Cache Update`: Debe ser < 2ms.

### 6.2. Dashboards de Grafana (Específicos para UC-04)

#### Dashboard 1: Cache Performance

```
Panel 1: Cache Hit Rate (Gauge)
- Métrica: kagami_identity_cache_hit_rate
- Umbral: Verde >80%, Amarillo 60-80%, Rojo <60%

Panel 2: Query Latency by Data Source (Time Series)
- Métricas: 
  - kagami_identity_query_latency{data_source="cache"}
  - kagami_identity_query_latency{data_source="database"}
  - kagami_identity_query_latency{data_source="cache_stale"}
- Comparar latencias entre fuentes

Panel 3: Cache Operations (Time Series)
- Métricas:
  - rate(kagami_identity_cache_hit_total[5m])
  - rate(kagami_identity_cache_miss_total[5m])
  - rate(kagami_identity_cache_stale_served_total[5m])
```

#### Dashboard 2: Database Performance

```
Panel 1: PostgreSQL Query Latency (Heatmap)
- Métrica: kagami_identity_postgres_query_latency
- Identificar queries lentas

Panel 2: Read Replica Lag (Time Series)
- Métrica: pg_replication_lag_bytes
- Alertar si lag > 10MB

Panel 3: Connection Pool Usage (Gauge)
- Métrica: pgbouncer_active_connections / pgbouncer_max_connections
- Alertar si >80%
```

#### Dashboard 3: Throughput and Errors

```
Panel 1: Query Throughput (Time Series)
- Métrica: rate(kagami_identity_query_total[1m])
- Comparar con SLO (500 req/seg)

Panel 2: Error Rate (Time Series)
- Métrica: rate(kagami_identity_query_errors_total[5m]) / rate(kagami_identity_query_total[5m])
- Alertar si >0.1%

Panel 3: Fallback Rate (Time Series)
- Métrica: rate(kagami_identity_fallback_triggered_total[5m])
- Identificar fallos de Redis/PostgreSQL
```

### 6.3. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Cache Hit Rate Bajo** | `cache_hit_rate < 0.60` por 10 min | Media | Revisar TTL, patrones de acceso, invalidación excesiva. |
| **Latencia DB Alta** | `postgres_query_latency_p95 > 100ms` por 5 min | Alta | Investigar queries lentas, verificar índices, considerar Read Replicas adicionales. |
| **Redis Degradado** | `redis_query_latency_p95 > 10ms` por 5 min | Alta | Verificar memoria, conexiones, y configuración de Redis. |
| **Fallback Activo** | `fallback_triggered_total > 0` | **Crítica** | Redis o PostgreSQL están caídos. Activar runbook de incidentes. |
| **Throughput Bajo** | `query_total < 300 req/seg` (esperado 500) | Media | Investigar cuellos de botella, verificar auto-scaling. |
| **Error Rate Alto** | `error_rate > 0.005` por 5 min | **Crítica** | Investigar errores, revisar logs, corregir bugs. |
| **Read Replica Lag** | `replication_lag > 10MB` por 2 min | Alta | Verificar red, carga en Primary, configuración de replicación. |
| **Connection Pool Exhausted** | `pgbouncer_active_connections > 80%` | Alta | Aumentar pool size, optimizar queries, o agregar más PgBouncers. |

---

## 7. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los objetivos de rendimiento iniciales (< 5ms cache hit, < 50ms DB query, 500 req/seg) que este documento formaliza en SLOs. |
| `09_Modern_Artifacts.md` | Define las métricas de OpenTelemetry; este documento explica cómo usarlas para validar SLAs de rendimiento y cache hit rate. |
| `10_UC_Specific_Data_Model.md` | Los índices covering, Read Replicas, y configuraciones de PostgreSQL/Redis definidos aquí son críticos para cumplir los SLOs de latencia. |
| `11_UC_Specific_Security_Threats.md` | Las mitigaciones de seguridad (distributed locks, rate limiting) tienen un costo de rendimiento que se mide y gestiona en este documento. |
| `SHUGO_Performance_Testing_Strategy.md` (Transversal) | Este documento aplica la estrategia global al contexto específico de operaciones de lectura de alto volumen. |

---

## 8. Anexos: Configuración de Rendimiento

### 8.1. PostgreSQL Tuning para Lecturas

```ini
# postgresql.conf optimizado para cargas de lectura intensivas
# Memoria
shared_buffers = 8GB              # 25% de RAM total (32GB)
effective_cache_size = 24GB       # 75% de RAM total
work_mem = 64MB                   # Para operaciones de ordenamiento
maintenance_work_mem = 2GB        # Para VACUUM y CREATE INDEX

# WAL y Checkpoints
wal_buffers = 64MB
checkpoint_completion_target = 0.9
checkpoint_timeout = 15min

# Query Planner
random_page_cost = 1.1            # Ajustar para SSD
effective_io_concurrency = 200    # Para SSD
default_statistics_target = 500   # Estadísticas más precisas

# Replicación
max_wal_senders = 10
wal_level = replica
hot_standby = on

# Logging (reducir para rendimiento)
log_min_duration_statement = 100  # Solo log queries > 100ms
log_checkpoints = on
log_connections = off
log_disconnections = off
```

### 8.2. Redis Cluster Configuration

```bash
# Crear cluster de Redis (3 nodos)
redis-cli --cluster create \
  redis-node-1:6379 \
  redis-node-2:6379 \
  redis-node-3:6379 \
  --cluster-replicas 0

# Verificar cluster
redis-cli cluster info
redis-cli cluster nodes
```

### 8.3. PgBouncer Monitoring

```sql
-- Conectar a PgBouncer admin console
psql -h localhost -p 6432 -U pgbouncer pgbouncer

-- Ver estadísticas
SHOW STATS;
SHOW POOLS;
SHOW SERVERS;
SHOW CLIENTS;
```

---
*Fin del Documento 12_UC_Specific_Performance_Benchmarks para UC-04*