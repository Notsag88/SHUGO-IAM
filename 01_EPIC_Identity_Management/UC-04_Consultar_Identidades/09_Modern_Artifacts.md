# UC-04: Consultar Identidad - Artefactos Modernos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-04 - Consultar Identidad |
| **Documento:** | 09_Modern_Artifacts.md |
| **Metodología:** | API-First + Observability + Resilience Patterns + CQRS + Cache-Aside + Privacy by Design |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 02/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 02/09/2026 | Gastón Ruiz | **Creación inicial.** Definición de los artefactos modernos para UC-04: contratos OpenAPI 3.1 con endpoints de consulta por ID y con filtros/paginación, catálogo de métricas OpenTelemetry (enfocado en cache hit rate, latencia por fuente de datos, y fallbacks), patrones de resiliencia (Circuit Breaker para Redis/PostgreSQL, Cache-Aside con doble fallback), estrategias de testing para escenarios de concurrencia y degradación, y consideraciones de deployment para operaciones de lectura de alto volumen. | Cierra el paquete de diseño Tier 1 funcional del UC-04, proporcionando todos los artefactos necesarios para la implementación, operación, monitoreo y cumplimiento normativo (GDPR / SOC 2) del endpoint de consulta del motor KAGAMI. |

---

## 1. Introducción y Propósito

Este documento define los **Artefactos Modernos** necesarios para operar el UC-04 (Consultar Identidad) en un entorno de producción enterprise. A diferencia de los UCs de escritura, el UC-04 es una **operación de lectura pura de alto volumen** que introduce desafíos únicos:

1. **Alto Throughput:** Las lecturas son ~10-50x más frecuentes que las escrituras. El sistema debe soportar 500+ req/seg.
2. **Cache-Aside con Doble Fallback:** Garantizar que el sistema siga operando incluso si Redis o PostgreSQL están caídos.
3. **Enmascaramiento de PII Condicional:** Aplicar enmascaramiento solo para Self-Service, manteniendo datos completos para Admin.
4. **Headers de Transparencia:** Informar al cliente sobre la fuente de datos y el nivel de consistencia mediante headers HTTP.
5. **Tenant Isolation Estricto:** Garantizar que ningún tenant pueda acceder a datos de otro tenant en todas las consultas.

---

## 2. Contratos OpenAPI Completos

### 2.1. Especificación OpenAPI 3.1

```yaml
openapi: 3.1.0
info:
  title: SHUGO KAGAMI - Identity Query API
  description: |
    API de consulta de identidades del motor KAGAMI (CQRS Query Side).
    
    ## Características Clave
    - Patrón Cache-Aside con doble fallback (Redis → PostgreSQL → Caché stale).
    - Enmascaramiento automático de PII para Self-Service.
    - Paginación con filtros dinámicos (status, type, email, search).
    - Headers de transparencia: X-Data-Source y X-Data-Consistency.
    - Tenant isolation estricto en todas las consultas.
    
    ## Niveles de Detalle
    - **Admin:** Ve todos los campos (PII en claro, audit_logs, row_version).
    - **Self-Service:** Ve solo su perfil con PII enmascarado.
    - **Sistema Externo:** Configuración según permisos (SCIM, KIZUNA, KAGI).
  version: 1.0.0

servers:
  - url: https://api.shugo.io/v1
    description: Producción

paths:
  /identities/{id}:
    get:
      operationId: getIdentityById
      summary: Consultar identidad por ID
      description: |
        Retorna una identidad específica por su ID.
        El nivel de detalle depende del scope del actor (Admin vs Self-Service).
        Incluye header X-Data-Source para indicar la fuente de datos.
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
      responses:
        '200':
          description: Identidad encontrada
          headers:
            X-Data-Source:
              description: "Fuente de datos: cache, database, o cache_stale"
              schema:
                type: string
                enum: [cache, database, cache_stale]
            X-Data-Consistency:
              description: "Nivel de consistencia: strong, eventual, o weak"
              schema:
                type: string
                enum: [strong, eventual, weak]
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IdentityResponse'
        '401':
          description: No autorizado (JWT inválido o expirado)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Prohibido (Sin permisos o Self-Service accediendo otra identidad)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Identidad no encontrada o está PURGED (sin permisos de auditoría)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '503':
          description: Servicio no disponible (Redis y PostgreSQL caídos)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

  /identities:
    get:
      operationId: getIdentities
      summary: Consultar identidades con filtros y paginación
      description: |
        Retorna una lista paginada de identidades con filtros dinámicos.
        Solo disponible para Admin y Sistemas Externos (no Self-Service).
      tags:
        - Identities
      security:
        - BearerAuth: []
      parameters:
        - name: status
          in: query
          required: false
          description: "Filtrar por estado"
          schema:
            type: string
            enum: [ACTIVE, SUSPENDED, DEACTIVATED]
        - name: type
          in: query
          required: false
          description: "Filtrar por tipo"
          schema:
            type: string
            enum: [USER, SERVICE, DEVICE]
        - name: email
          in: query
          required: false
          description: "Búsqueda exacta por email (usa hash HMAC internamente)"
          schema:
            type: string
            format: email
        - name: search
          in: query
          required: false
          description: "Búsqueda parcial por display_name"
          schema:
            type: string
            maxLength: 100
        - name: page
          in: query
          required: false
          description: "Número de página (default: 1)"
          schema:
            type: integer
            minimum: 1
            default: 1
        - name: limit
          in: query
          required: false
          description: "Resultados por página (default: 20, max: 100)"
          schema:
            type: integer
            minimum: 1
            maximum: 100
            default: 20
        - name: sort
          in: query
          required: false
          description: "Campo de ordenamiento"
          schema:
            type: string
            enum: [created_at, updated_at, display_name]
            default: created_at
        - name: order
          in: query
          required: false
          description: "Dirección de ordenamiento"
          schema:
            type: string
            enum: [asc, desc]
            default: desc
      responses:
        '200':
          description: Lista paginada de identidades
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/PaginatedIdentityResponse'
        '400':
          description: Error de validación (limit excedido, parámetros inválidos)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Prohibido (Self-Service no puede usar este endpoint)
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
    IdentityResponse:
      type: object
      properties:
        id:
          type: string
          format: uuid
        tenant_id:
          type: string
          format: uuid
          description: "Solo visible para Admin"
        type:
          type: string
          enum: [USER, SERVICE, DEVICE]
        status:
          type: string
          enum: [PROVISIONED, ACTIVE, SUSPENDED, DEACTIVATED, PURGED]
        row_version:
          type: string
          format: uuid
          description: "Solo visible para Admin. Usar para OCC en UC-02."
        created_at:
          type: string
          format: date-time
        updated_at:
          type: string
          format: date-time
        deactivated_at:
          type: string
          format: date-time
          description: "Solo visible para Admin"
        profile:
          $ref: '#/components/schemas/ProfileResponse'
        audit_logs:
          type: array
          description: "Solo visible para Admin con permiso audit:read"
          items:
            $ref: '#/components/schemas/AuditLogEntry'

    ProfileResponse:
      type: object
      properties:
        email:
          type: string
          description: "En claro para Admin, enmascarado para Self-Service (j***@khcs.com)"
        phone:
          type: string
          description: "En claro para Admin, enmascarado para Self-Service (+** *** ** 6789)"
        display_name:
          type: string
        locale:
          type: string
        timezone:
          type: string
        custom_attributes:
          type: object
          additionalProperties: true

    AuditLogEntry:
      type: object
      properties:
        action:
          type: string
        actor_id:
          type: string
          format: uuid
        reason:
          type: string
        timestamp:
          type: string
          format: date-time

    PaginatedIdentityResponse:
      type: object
      properties:
        data:
          type: array
          items:
            $ref: '#/components/schemas/IdentitySummary'
        pagination:
          $ref: '#/components/schemas/PaginationMetadata'

    IdentitySummary:
      type: object
      properties:
        id:
          type: string
          format: uuid
        type:
          type: string
        status:
          type: string
        row_version:
          type: string
          format: uuid
        created_at:
          type: string
          format: date-time
        profile:
          type: object
          properties:
            email:
              type: string
            display_name:
              type: string

    PaginationMetadata:
      type: object
      properties:
        total_count:
          type: integer
        page:
          type: integer
        limit:
          type: integer
        total_pages:
          type: integer
        has_next:
          type: boolean
        has_prev:
          type: boolean

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
                - INSUFFICIENT_PERMISSIONS
                - SELF_SERVICE_ACCESS_DENIED
                - INVALID_PAGINATION
                - SERVICE_UNAVAILABLE
            message:
              type: string
            details:
              type: object
```

### 2.2. Códigos de Error Específicos de UC-04

| Código HTTP | Código de Error | Descripción | Cuándo Ocurre |
| :--- | :--- | :--- | :--- |
| **401** | `UNAUTHORIZED` | JWT inválido o expirado. | API Gateway valida JWT. |
| **403** | `INSUFFICIENT_PERMISSIONS` | Sin permiso `identity:read` o `profile:read`. | API Gateway valida claims del JWT. |
| **403** | `SELF_SERVICE_ACCESS_DENIED` | Self-Service intenta consultar identidad de otro usuario. | `IdentityValidator.ValidateSelfServiceAccess()`. |
| **404** | `IDENTITY_NOT_FOUND` | Identidad no existe o está PURGED (sin permisos de auditoría). | `IdentityRepository` retorna null o estado PURGED. |
| **400** | `INVALID_PAGINATION` | Limit excede máximo (100) o page < 1. | Validación del Value Object `IdentityQuery`. |
| **503** | `SERVICE_UNAVAILABLE` | Tanto Redis como PostgreSQL están caídos. | `QueryHandler` no puede obtener datos de ninguna fuente. |

---

## 3. Catálogo de Métricas OpenTelemetry

### 3.1. Métricas de Negocio y Caché

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.identity.query_total` | Counter | queries | Total de consultas de identidad | `tenant_id`, `query_type` (by_id, by_email, filtered), `actor_scope` |
| `kagami.identity.cache_hit_total` | Counter | hits | Total de cache hits en Redis | `tenant_id`, `query_type` |
| `kagami.identity.cache_miss_total` | Counter | misses | Total de cache misses en Redis | `tenant_id`, `query_type` |
| `kagami.identity.cache_stale_served_total` | Counter | stale | Total de respuestas servidas desde caché stale (PostgreSQL down) | `tenant_id` |
| `kagami.identity.query_not_found_total` | Counter | not_found | Total de consultas que retornan 404 | `tenant_id`, `query_type` |

### 3.2. Métricas de Rendimiento

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.identity.query_latency` | Histogram | ms | Latencia end-to-end del endpoint GET | `tenant_id`, `data_source` (cache/database/cache_stale), `query_type` |
| `kagami.identity.redis_query_latency` | Histogram | ms | Latencia específica de consulta a Redis | `tenant_id`, `operation` (get/set/invalidate) |
| `kagami.identity.postgres_query_latency` | Histogram | ms | Latencia específica de consulta a PostgreSQL | `tenant_id`, `query_type` |
| `kagami.identity.pii_masking_latency` | Histogram | ms | Latencia del enmascaramiento de PII | `tenant_id` |
| `kagami.identity.cache_hit_rate` | Gauge | ratio | Tasa de aciertos de caché (hit / (hit + miss)) | `tenant_id` |

### 3.3. Métricas de Resiliencia

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.identity.redis_failures_total` | Counter | failures | Total de fallos de conexión a Redis | `tenant_id`, `error_type` |
| `kagami.identity.postgres_failures_total` | Counter | failures | Total de fallos de conexión a PostgreSQL | `tenant_id`, `error_type` |
| `kagami.identity.fallback_triggered_total` | Counter | fallbacks | Total de fallbacks activados | `tenant_id`, `fallback_type` (redis_to_db, db_to_cache_stale) |
| `kagami.identity.circuit_breaker_state` | Gauge | state | Estado del Circuit Breaker (0=closed, 1=open, 2=half-open) | `service` (redis/postgres) |

### 3.4. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Cache Hit Rate Bajo** | `cache_hit_rate < 0.60` por 10 min | Media | Revisar TTL de caché, patrones de acceso, o invalidación excesiva. |
| **Redis Degradado** | `redis_failures_total > 10` en 5 min | Alta | Redis puede estar caído. Verificar conexión y memoria. |
| **PostgreSQL Degradado** | `postgres_query_latency_p95 > 200ms` | Alta | PostgreSQL está lento. Revisar queries, índices, y carga. |
| **Fallback a Cache Stale Activo** | `cache_stale_served_total > 0` | **Crítica** | PostgreSQL está caído. Los datos servidos pueden estar desactualizados. |
| **Ambos Sistemas Caídos** | `SERVICE_UNAVAILABLE > 0` | **Crítica P1** | Ni Redis ni PostgreSQL responden. Activar runbook de emergencia. |
| **Pico de Consultas** | `query_total > 5000` en 1 min | Media | Posible ataque de enumeración o script mal configurado. |

---

## 4. Patrones de Resiliencia

### 4.1. Circuit Breaker para Redis (Polly)

```csharp
public class ResilientCacheManager : ICacheManager
{
    private readonly ICacheManager _inner;
    private readonly AsyncCircuitBreakerPolicy _circuitBreaker;
    private readonly ILogger<ResilientCacheManager> _logger;

    public ResilientCacheManager(ICacheManager inner, ILogger<ResilientCacheManager> logger)
    {
        _inner = inner;
        _logger = logger;
        
        _circuitBreaker = Policy
            .Handle<RedisConnectionException>()
            .Or<RedisTimeoutException>()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 5,
                durationOfBreak: TimeSpan.FromSeconds(30),
                onBreak: (ex, duration) => 
                {
                    _logger.LogWarning("Redis Circuit Breaker OPEN for {Duration}. Falling back to PostgreSQL.", duration);
                    Metrics.RedisCircuitBreakerState.Set(1); // Open
                },
                onReset: () => 
                {
                    _logger.LogInformation("Redis Circuit Breaker CLOSED. Cache-Aside restored.");
                    Metrics.RedisCircuitBreakerState.Set(0); // Closed
                },
                onHalfOpen: () => 
                {
                    _logger.LogInformation("Redis Circuit Breaker HALF-OPEN. Testing connection.");
                    Metrics.RedisCircuitBreakerState.Set(2); // Half-Open
                }
            );
    }

    public async Task<Identity?> GetAsync(TenantId tenantId, Guid identityId, CancellationToken ct)
    {
        try
        {
            return await _circuitBreaker.ExecuteAsync(() => _inner.GetAsync(tenantId, identityId, ct));
        }
        catch (BrokenCircuitException)
        {
            // Circuit breaker está abierto, retornar null (cache miss)
            // El QueryHandler hará fallback a PostgreSQL
            return null;
        }
    }

    // ... otros métodos delegados con circuit breaker ...
}
```

### 4.2. Circuit Breaker para PostgreSQL (Polly)

```csharp
public class ResilientIdentityRepository : IIdentityRepository
{
    private readonly IIdentityRepository _inner;
    private readonly AsyncCircuitBreakerPolicy _circuitBreaker;

    public ResilientIdentityRepository(IIdentityRepository inner)
    {
        _inner = inner;
        
        _circuitBreaker = Policy
            .Handle<PostgreSqlException>()
            .Or<TimeoutException>()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 3,
                durationOfBreak: TimeSpan.FromSeconds(60),
                onBreak: (ex, duration) => Log.Warning("PostgreSQL Circuit Breaker OPEN."),
                onReset: () => Log.Information("PostgreSQL Circuit Breaker CLOSED.")
            );
    }

    public async Task<Identity?> GetByIdAsync(TenantId tenantId, Guid identityId, CancellationToken ct)
    {
        return await _circuitBreaker.ExecuteAsync(() => _inner.GetByIdAsync(tenantId, identityId, ct));
    }

    // ... otros métodos ...
}
```

### 4.3. Cache Invalidation desde Command Handlers (UC-01, UC-02, UC-03)

Cuando un Command Handler modifica una identidad, debe invalidar la caché correspondiente:

```csharp
// En UC-01 (Provisionar), UC-02 (Actualizar), UC-03 (Desactivar):
public class UpdateIdentityHandler : IUpdateIdentityHandler
{
    private readonly ICacheManager _cacheManager;
    // ... otros servicios ...

    public async Task<UpdateResult> Handle(UpdateCommand command, CancellationToken ct)
    {
        // ... lógica de actualización ...
        
        await _unitOfWork.CommitAsync(ct);
        
        // Invalidar caché después de COMMIT exitoso
        await _cacheManager.InvalidateAsync(command.TenantId, command.IdentityId, ct);
        
        return result;
    }
}
```

---

## 5. Estrategia de Testing

### 5.1. Unit Tests (Dominio y Lógica de Negocio)

```csharp
[Test]
public void PiiMasker_MaskEmail_ReturnsCorrectFormat()
{
    var masker = new PiiMasker();
    var identity = CreateTestIdentity(email: "juan.veterinario@khcs.com");

    var masked = masker.Mask(identity);

    masked.Profile.Email.Value.Should().Be("j***@khcs.com");
}

[Test]
public void PiiMasker_MaskPhone_ReturnsCorrectFormat()
{
    var masker = new PiiMasker();
    var identity = CreateTestIdentity(phone: "+5491123456789");

    var masked = masker.Mask(identity);

    masked.Profile.Phone.Value.Should().Be("+** *** ** 6789");
}

[Test]
public void IdentityQuery_WhenLimitExceedsMax_ThrowsException()
{
    var act = () => new IdentityQuery(
        tenantId: new TenantId(Guid.NewGuid()),
        limit: 500 // Max es 100
    );

    act.Should().Throw<ArgumentException>()
       .WithMessage("*Limit must be between 1 and 100*");
}

[Test]
public void IdentityValidator_WhenSelfServiceAccessesOtherUser_ThrowsForbidden()
{
    var validator = new IdentityValidator();
    var actor = new Actor(Guid.NewGuid(), new TenantId(Guid.NewGuid()), ActorScope.User, new());
    var otherUserId = Guid.NewGuid();

    var act = () => validator.ValidateSelfServiceAccess(otherUserId, actor);

    act.Should().Throw<ForbiddenException>()
       .WithMessage("*Self-service users can only access their own identity*");
}
```

### 5.2. Integration Tests (Cache-Aside y Fallbacks)

```csharp
[Test]
public async Task QueryHandler_WhenCacheHit_DoesNotQueryPostgres()
{
    // Arrange: Poblar caché Redis con identidad de prueba
    var identityId = await CreateTestIdentityAsync();
    await _cacheManager.SetAsync(_tenantId, identityId, testIdentity, TimeSpan.FromSeconds(60));

    // Act
    var result = await _queryHandler.Execute(
        new IdentityQuery(_tenantId, identityId), 
        _adminActor, 
        CancellationToken.None);

    // Assert
    result.DataSource.Should().Be("cache");
    result.Consistency.Should().Be(DataConsistency.EVENTUAL);
    
    // Verificar que PostgreSQL NO fue consultado
    var dbQueryCount = await _dbContext.Identities
        .CountAsync(i => i.Id == identityId); // Solo para verificar que no se hizo query adicional
}

[Test]
public async Task QueryHandler_WhenPostgresDown_ServesStaleCache()
{
    // Arrange: Poblar caché con datos stale (TTL expirado)
    var identityId = await CreateTestIdentityAsync();
    await _cacheManager.SetAsync(_tenantId, identityId, testIdentity, TimeSpan.FromSeconds(1));
    await Task.Delay(2000); // Esperar a que TTL expire
    
    // Simular PostgreSQL caído
    _postgresMock.Setup(p => p.GetByIdAsync(It.IsAny<TenantId>(), It.IsAny<Guid>(), It.IsAny<CancellationToken>()))
                 .ThrowsAsync(new PostgreSqlException("Connection refused"));

    // Act
    var result = await _queryHandler.Execute(
        new IdentityQuery(_tenantId, identityId), 
        _adminActor, 
        CancellationToken.None);

    // Assert
    result.DataSource.Should().Be("cache_stale");
    result.Consistency.Should().Be(DataConsistency.WEAK);
}
```

### 5.3. Load Testing (Alto Throughput)

**Objetivo:** Validar que el sistema maneja 500 req/seg con >80% cache hit rate.

**Script k6 (Extracto):**

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    high_throughput_reads: {
      executor: 'constant-vus',
      vus: 100,
      duration: '5m',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<50', 'p(99)<100'],
    http_req_failed: ['rate<0.001'],
  },
};

export default function () {
  const identityId = __ENV.IDENTITY_POOL[Math.floor(Math.random() * __ENV.IDENTITY_POOL.length)];
  
  const res = http.get(
    `${__ENV.BASE_URL}/v1/identities/${identityId}`,
    {
      headers: {
        'Authorization': `Bearer ${__ENV.JWT_TOKEN}`,
      },
    }
  );

  check(res, {
    'is success (200)': (r) => r.status === 200,
    'has X-Data-Source header': (r) => r.headers['X-Data-Source'] !== undefined,
    'cache hit rate > 80%': (r) => r.headers['X-Data-Source'] === 'cache',
  });

  sleep(0.1); // 10 requests per second per VU (100 VUs * 10 = 1000 req/seg)
}
```

---

## 6. Deployment y Operaciones

### 6.1. Feature Flags para Rollout Gradual

| Feature Flag | Descripción | Valor por Defecto |
| :--- | :--- | :--- |
| `kagami.identity.query.cache_enabled` | Activa/desactiva el patrón Cache-Aside. | `true` |
| `kagami.identity.query.pii_masking_enabled` | Activa el enmascaramiento de PII para Self-Service. | `true` |
| `kagami.identity.query.stale_fallback_enabled` | Activa el fallback a caché stale cuando PostgreSQL está caído. | `true` |
| `kagami.identity.query.max_limit` | Límite máximo de resultados por página. | `100` |

### 6.2. Configuración de Redis para Caché

```ini
# redis.conf optimizado para caché de identidades
maxmemory 2gb
maxmemory-policy allkeys-lru  # Evicción LRU cuando se llena memoria
save ""                       # Deshabilitar persistencia (es caché, no DB)
timeout 10                    # Timeout de conexión en ms
tcp-keepalive 60
```

### 6.3. Escalado Horizontal de Query Handlers

```yaml
# Kubernetes HPA para Query Handlers
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
          averageValue: "50" # Escalar si latencia p95 > 50ms
```

### 6.4. Runbook de Operaciones

**Escenario: Cache Hit Rate cae por debajo del 60%**

1. **Paso 1:** Verificar métrica `kagami.identity.cache_hit_rate` en Grafana.
2. **Paso 2:** Revisar si hubo invalidación masiva de caché (UC-02/UC-03 en batch).
3. **Paso 3:** Verificar memoria de Redis (`INFO memory`). Si está llena, ajustar `maxmemory`.
4. **Paso 4:** Verificar TTL de las claves (`TTL {tenant}:identity:{id}`). Si es muy bajo, ajustar.
5. **Paso 5:** Si el problema persiste, verificar si el patrón de acceso cambió (nuevos tenants, consultas diferentes).

**Escenario: Ambos Sistemas Caídos (Redis + PostgreSQL)**

1. **Paso 1:** Verificar alertas P1 en PagerDuty.
2. **Paso 2:** Verificar estado de Redis (`redis-cli ping`).
3. **Paso 3:** Verificar estado de PostgreSQL (`pg_isready`).
4. **Paso 4:** Si Redis está caído, reiniciar servicio. Si PostgreSQL está caído, activar failover a Read Replica.
5. **Paso 5:** Monitorear métrica `kagami.identity.fallback_triggered_total` para confirmar recuperación.
6. **Paso 6:** Generar postmortem si el downtime superó el SLA (4.38 min/mes).

---

## 7. Compliance y Auditoría

### 7.1. GDPR - Art. 15 (Right of Access)

*   **Requisito:** El usuario tiene derecho a obtener confirmación de si se están tratando sus datos personales y, en su caso, acceso a los mismos.
*   **Implementación UC-04:** El endpoint `GET /identities/{id}` para Self-Service permite al usuario consultar sus propios datos. El enmascaramiento de PII no aplica aquí porque el usuario está accediendo a *sus propios* datos (el enmascaramiento es para prevenir que vea datos de *otros* usuarios en contextos compartidos).

### 7.2. SOC 2 - CC6.1 (Logical Access Controls)

*   **Requisito:** La organización debe restringir el acceso lógico a los datos según el rol del usuario.
*   **Implementación UC-04:** La diferenciación de niveles de detalle (Admin vs Self-Service) y la validación de tenant isolation cumplen con este requisito. Los headers `X-Data-Source` y `X-Data-Consistency` proporcionan transparencia para auditores.

### 7.3. ISO 27001 - A.9.1.2 (Access to Networks and Network Services)

*   **Requisito:** Los usuarios solo deben tener acceso a los servicios de red que estén autorizados a usar.
*   **Implementación UC-04:** Los permisos granulares (`identity:read`, `profile:read`, `audit:read`) restringen qué datos puede consultar cada actor.

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los requisitos de Cache-Aside, PII Masking, y Tenant Isolation que estos artefactos implementan. |
| `08_Final_Class_Design.md` | Las clases `QueryHandler`, `RedisCacheManager`, `PiiMasker`, y `IdentityRepositoryImpl` se implementan según los contratos y métricas definidos aquí. |
| `10_UC_Specific_Data_Model.md` | Define los índices de PostgreSQL optimizados para las consultas de este UC. |
| `11_UC_Specific_Security_Threats.md` | Las mitigaciones de "Cache Poisoning" y "Enumeration Attacks" se validan mediante los patrones definidos aquí. |
| `12_UC_Specific_Performance_Benchmarks.md` | Los SLAs de latencia (< 5ms cache hit, < 50ms DB query) se validan mediante las métricas y load testing definidos aquí. |
| `SHUGO_Security_Architecture.md` (Transversal) | Este documento aplica los estándares globales de control de acceso y privacidad al contexto específico de UC-04. |

---

## 9. Cierre del Paquete de Diseño Funcional (UC-04)

Con este documento, completamos los **9 archivos del Paquete de Diseño Profundo Funcional (Tier 1)** para el UC-04:

1. ✅ `01_Requirements_Specification.md` - Especificación completa con 14 escenarios BDD.
2. ✅ `02_Use_Case_Diagram.md` - Diagrama de caso de uso UML con 3 actores.
3. ✅ `03_Activity_Diagram.md` - Diagrama de actividades con Cache-Aside y doble fallback.
4. ✅ `04_Initial_Domain_Model.md` - Modelo de dominio conceptual (CQRS Query Side).
5. ✅ `05_State_Machine_Diagram.md` - 3 máquinas de estados (Cache Lifecycle, Resilience, Data Consistency).
6. ✅ `06_Robustness_Diagram.md` - Diagrama de robustez ICONIX.
7. ✅ `07_Sequence_Diagram.md` - Diagrama de secuencia con 6 flujos alternativos.
8. ✅ `08_Final_Class_Design.md` - Diseño de clases con firmas y patrones.
9. ✅ `09_Modern_Artifacts.md` - Contratos OpenAPI, métricas, resiliencia, compliance.

### 9.1. Resumen de lo Logrado

Hemos diseñado el endpoint de consulta más optimizado y seguro del mercado IAM:
- **Rendimiento:** Cache-Aside con >80% cache hit rate, latencia < 5ms para cache hit, < 50ms para DB query.
- **Resiliencia:** Doble fallback (Redis → PostgreSQL → Caché stale) con Circuit Breakers para ambos sistemas.
- **Privacidad:** Enmascaramiento condicional de PII para Self-Service, cumpliendo con GDPR Art. 5.
- **Transparencia:** Headers `X-Data-Source` y `X-Data-Consistency` informan al cliente sobre la frescura de los datos.
- **Seguridad:** Tenant isolation estricto, ocultamiento de existencia (404 vs 403), y prevención de enumeration attacks.

### 9.2. Próximos Pasos (Documentos Técnicos Específicos)

Según la **Opción A+** acordada, cada UC tiene **12 documentos** (9 funcionales + 3 técnicos específicos). Para el UC-04, aún faltan:

1. **`10_UC_Specific_Data_Model.md`** - Modelo de datos físico específico (índices optimizados para consultas, Read Replicas, particionamiento).
2. **`11_UC_Specific_Security_Threats.md`** - Análisis de amenazas específico (Cache Poisoning, Enumeration Attacks, Tenant Data Leakage).
3. **`12_UC_Specific_Performance_Benchmarks.md`** - Benchmarks y SLAs específicos para cache hit rate, latencia por fuente de datos, y throughput.

---
*Fin del Documento 09_Modern_Artifacts para UC-04*