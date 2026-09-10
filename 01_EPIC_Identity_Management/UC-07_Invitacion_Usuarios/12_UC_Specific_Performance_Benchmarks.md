# UC-07: Invitación de Usuarios - Benchmarks, SLAs y Estrategia de Rendimiento

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-07 - Invitación de Usuarios (Admin-Initiated Invitation) |
| **Documento:** | 12_UC_Specific_Performance_Benchmarks.md |
| **Metodología:** | Performance Engineering + SRE (SLI/SLO/SLA) + Load Testing (k6) + Capacity Planning + Dual-Actor Async Profiling |
| **Estándares:** | ISO/IEC 25010 (Performance Efficiency), Google SRE Workbook, OWASP Performance Project |
| **Autor:** | Gastón Ruiz |
| **Revisado por:** | [Security Architect] |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 05/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 05/09/2026 | Gastón Ruiz | **Creación inicial.** Definición de SLAs, SLOs, estrategia de Load Testing (enfocada en picos de onboarding, degradación de Email Service, race conditions en aceptación, y coste computacional de SecureCompare), y planificación de capacidad para el endpoint de invitación del motor KAGAMI. | Documento de referencia para SREs y Arquitectos de Rendimiento. Establece cómo mantener la latencia controlada y la disponibilidad alta a pesar de la naturaleza dual-actor asíncrona, la dependencia crítica del Email Service, y la integración con UC-06. |

---

## 1. Introducción y Propósito

A diferencia de los UCs síncronos (UC-01 a UC-05) y del UC-06 (Registro Self-Service), el **UC-07 (Invitación de Usuarios)** es una operación **dual-actor asíncrona** con características de rendimiento únicas y desafiantes:

1. **Dos Fases Separadas Temporalmente:** La Fase 1 (Creación por Admin) y la Fase 2 (Aceptación por Usuario) ocurren en momentos distintos, con separación de hasta 7 días (TTL).
2. **Dependencia Crítica de Email Service:** La Fase 1 depende de un servicio externo (SMTP) que puede fallar, requiriendo patrón de compensación transaccional.
3. **Coste Computacional de Seguridad:** La validación de tokens usa `SecureCompare` (comparación en tiempo constante) que tiene coste CPU adicional.
4. **Integración con UC-06:** La Fase 2 delega al flujo de registro (UC-06), añadiendo latencia adicional.
5. **Job de Limpieza en Background:** Un proceso cada hora actualiza invitaciones expiradas, consumiendo recursos de DB.
6. **Rate Limiting Multi-Capa:** Tres capas de rate limiting (admin, tenant, IP) requieren consultas a Redis en cada request.
7. **Métrica de Negocio Crítica:** La tasa de conversión (aceptadas/creadas) es el KPI principal, no solo la latencia.

Este documento aborda estos desafíos definiendo SLAs realistas, estrategias de optimización y pruebas de carga específicas para garantizar que el onboarding corporativo sea rápido, seguro y escalable.

---

## 2. Service Level Indicators (SLIs) y Objectives (SLOs)

### 2.1. SLIs Específicos para UC-07

| SLI | Descripción | Fórmula / Medición |
| :--- | :--- | :--- |
| **Invitation Creation Latency (p95)** | Tiempo total de la Fase 1 (creación + envío de email). | `histogram_quantile(0.95, rate(kagami_invitation_creation_latency_bucket[5m]))` |
| **Invitation Acceptance Latency (p95)** | Tiempo total de la Fase 2 (validación token + UC-06 + asignación roles). | `histogram_quantile(0.95, rate(kagami_invitation_acceptance_latency_bucket[5m]))` |
| **Token Validation Latency (p95)** | Tiempo específico de validación criptográfica (SecureCompare). | `histogram_quantile(0.95, rate(kagami_invitation_token_validation_latency_bucket[5m]))` |
| **Email Send Latency (p95)** | Tiempo de aceptación del email por el servidor SMTP. | `histogram_quantile(0.95, rate(kagami_invitation_email_send_latency_bucket[5m]))` |
| **UC-06 Integration Latency (p95)** | Tiempo de integración con el flujo de registro. | `histogram_quantile(0.95, rate(kagami_invitation_uc06_integration_latency_bucket[5m]))` |
| **Cleanup Job Duration (p95)** | Duración del job de limpieza de invitaciones expiradas. | `histogram_quantile(0.95, rate(kagami_invitation_cleanup_job_duration_bucket[5m]))` |
| **Invitation Conversion Rate** | Porcentaje de invitaciones aceptadas vs. creadas. | `rate(kagami_invitation_accepted_total[7d]) / rate(kagami_invitation_created_total[7d])` |
| **Compensation Rate** | Porcentaje de invitaciones que requieren compensación (email falló). | `rate(kagami_invitation_compensation_total[5m]) / rate(kagami_invitation_created_total[5m])` |
| **Rate Limiting Cache Latency (p95)** | Tiempo de consulta a Redis para rate limiting multi-capa. | `histogram_quantile(0.95, rate(kagami_invitation_rate_limit_cache_latency_bucket[5m]))` |
| **Invitation Success Rate** | Porcentaje de invitaciones que completan el flujo sin errores 5xx. | `rate(kagami_invitation_success_total[5m]) / rate(kagami_invitation_attempts_total[5m])` |

### 2.2. SLOs y SLAs

| Métrica | SLO (Objetivo Interno - 95%) | SLA (Contrato Enterprise - 99%) | Justificación |
| :--- | :--- | :--- | :--- |
| **Latencia Creación (p95)** | **< 500 ms** | **< 1.000 ms** | Incluye validaciones (~50ms), DB insert (~30ms), Email send (~300ms), y rate limiting (~20ms). |
| **Latencia Aceptación (p95)** | **< 800 ms** | **< 1.500 ms** | Incluye validación token (~30ms), UC-06 integration (~400ms), DB update (~50ms), RBAC assignment (~100ms). |
| **Latencia Validación Token (p95)** | **< 30 ms** | **< 50 ms** | SecureCompare en tiempo constante + lookup en DB con índice único. |
| **Latencia Email Send (p95)** | **< 300 ms** | **< 500 ms** | Asumiendo SMTP saludable. Si falla, se ejecuta compensación. |
| **Latencia UC-06 Integration (p95)** | **< 400 ms** | **< 800 ms** | Delegación al flujo de registro (creación de password, aceptación términos). |
| **Duración Cleanup Job (p95)** | **< 5.000 ms** | **< 10.000 ms** | Procesamiento en lotes de 1000 invitaciones con FOR UPDATE SKIP LOCKED. |
| **Throughput Creación** | **50 invitaciones/min** | **30 invitaciones/min** | Limitado por Email Service y rate limiting distribuido. |
| **Throughput Aceptación** | **100 aceptaciones/min** | **60 aceptaciones/min** | Limitado por integración con UC-06. |
| **Disponibilidad** | **99.9%** | **99.5%** | Permite caídas breves de Email Service (usando Circuit Breaker + compensación). |
| **Tasa de Error (5xx)** | **< 0.5%** | **< 1.0%** | Errores de DB, SMTP, o UC-06 no manejados. |
| **Tasa de Conversión** | **> 70%** | **> 60%** | Usuarios que aceptan la invitación dentro del TTL de 7 días. |
| **Tasa de Compensación** | **< 5%** | **< 10%** | Invitaciones que fallan en envío de email y requieren compensación. |

---

## 3. Cuellos de Botella y Estrategias de Optimización

### 3.1. Cuello de Botella #1: Email Service (I/O Bound)

**Problema:** El envío de email es el paso más lento de la Fase 1 (~300ms) y un punto crítico de falla. Si SMTP está caído, se ejecuta compensación (DELETE), duplicando la carga de escritura.

**Estrategia de Optimización:**

#### A. Circuit Breaker con Polly
```csharp
public class ResilientEmailService : IEmailService
{
    private readonly AsyncCircuitBreakerPolicy _circuitBreaker;

    public ResilientEmailService(IEmailService inner)
    {
        _circuitBreaker = Policy
            .Handle<SmtpException>()
            .Or<TimeoutException>()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 3,
                durationOfBreak: TimeSpan.FromMinutes(5),
                onBreak: (ex, duration) => 
                    Metrics.EmailCircuitBreakerState.Set(1), // Open
                onReset: () => 
                    Metrics.EmailCircuitBreakerState.Set(0)  // Closed
            );
    }

    public async Task SendInvitationEmailAsync(Email email, InvitationToken token, string message, CancellationToken ct)
    {
        await _circuitBreaker.ExecuteAsync(() => 
            _inner.SendInvitationEmailAsync(email, token, message, ct));
    }
}
```

#### B. Timeouts Estrictos y Conexiones Persistentes
```csharp
services.AddFluentEmail("noreply@shugo.io")
    .AddSmtpSender(new SmtpClient("smtp.sendgrid.net") {
        Timeout = 2000, // 2 segundos máximo. Si falla, compensamos.
        EnableSsl = true,
        ServicePoint.MaxIdleTime = 1000 // Mantener conexión viva
    });
```

#### C. Retry Policy con Backoff Exponencial
```csharp
var retryPolicy = Policy
    .Handle<SmtpException>()
    .Or<TimeoutException>()
    .WaitAndRetryAsync(
        retryCount: 3,
        sleepDurationProvider: retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)), // 2s, 4s, 8s
        onRetry: (ex, timeSpan, retryCount, context) =>
            Metrics.EmailRetryAttempts.Add(1)
    );
```

### 3.2. Cuello de Botella #2: Validación Criptográfica de Tokens (CPU Bound)

**Problema:** `SecureCompare` en tiempo constante tiene coste CPU adicional (~5ms por validación) comparado con comparación estándar (~0.1ms). Para 100 aceptaciones/min, necesitamos ~500ms de CPU dedicados.

**Estrategia de Optimización:**

#### A. Implementación Optimizada de SecureCompare
```csharp
public static bool SecureCompare(string a, string b)
{
    if (a == null || b == null) return false;
    if (a.Length != b.Length) return false;

    // Usar Span<T> para evitar allocations
    var spanA = a.AsSpan();
    var spanB = b.AsSpan();
    
    var result = 0;
    for (int i = 0; i < spanA.Length; i++)
    {
        result |= spanA[i] ^ spanB[i];
    }

    return result == 0;
}
```

#### B. Cache de Tokens Recientes (Redis)
```csharp
// Cache de tokens validados recientemente para prevenir re-validación
var cacheKey = $"token_validated:{tokenHash}";
var isValidated = await _cache.GetAsync<bool>(cacheKey, ct);

if (isValidated)
{
    // Token ya validado, saltar SecureCompare
    return true;
}

// Validar con SecureCompare
var isValid = token.Validate(providedToken);

if (isValid)
{
    // Cache por 5 minutos (TTL corto para prevenir replay)
    await _cache.SetAsync(cacheKey, true, TimeSpan.FromMinutes(5), ct);
}
```

#### C. Búsqueda por Índice Único (O(1))
```sql
-- Índice único para búsqueda O(1) por token_hash
CREATE UNIQUE INDEX idx_invitations_token_hash 
    ON invitations(token_hash) 
    WHERE status = 'PENDING';
```

### 3.3. Cuello de Botella #3: Rate Limiting Multi-Capa (Redis Bound)

**Problema:** Cada request de creación requiere 3 consultas a Redis (admin, tenant, IP). Para 50 invitaciones/min, son 150 consultas/min a Redis.

**Estrategia de Optimización:**

#### A. Pipeline de Redis para Consultas Batch
```csharp
public async Task<bool> IsAllowedAsync(TenantId tenantId, AdminId adminId, string ipAddress, CancellationToken ct)
{
    var adminKey = $"rate_limit:invitation:admin:{adminId.Value}";
    var tenantKey = $"rate_limit:invitation:tenant:{tenantId.Value}";
    var ipKey = $"rate_limit:invitation:ip:{ipAddress}";

    // Pipeline: ejecutar las 3 consultas en un solo round-trip
    var db = _redis.GetDatabase();
    var tasks = new[]
    {
        db.StringGetAsync(adminKey),
        db.StringGetAsync(tenantKey),
        db.StringGetAsync(ipKey)
    };

    await Task.WhenAll(tasks);

    var adminCount = (int)(tasks[0].Result ?? 0);
    var tenantCount = (int)(tasks[1].Result ?? 0);
    var ipCount = (int)(tasks[2].Result ?? 0);

    // Validar límites
    if (adminCount >= MAX_INVITATIONS_PER_ADMIN) return false;
    if (tenantCount >= MAX_INVITATIONS_PER_TENANT_PER_HOUR) return false;
    if (ipCount >= MAX_INVITATIONS_PER_IP_PER_HOUR) return false;

    // Incrementar contadores en pipeline
    var incrementTasks = new[]
    {
        db.StringIncrementAsync(adminKey, TimeSpan.FromDays(1)),
        db.StringIncrementAsync(tenantKey, TimeSpan.FromHours(1)),
        db.StringIncrementAsync(ipKey, TimeSpan.FromHours(1))
    };

    await Task.WhenAll(incrementTasks);

    return true;
}
```

#### B. Cache Local para Rate Limiting (Caffeine)
```csharp
// Cache local para reducir consultas a Redis
private readonly ICache _localCache = new MemoryCache(new MemoryCacheOptions
{
    SizeLimit = 10000,
    ExpirationScanFrequency = TimeSpan.FromMinutes(1)
});

public async Task<bool> IsAllowedAsync(TenantId tenantId, AdminId adminId, string ipAddress, CancellationToken ct)
{
    var cacheKey = $"rate_limit:local:{adminId.Value}:{tenantId.Value}";
    
    // Consultar cache local primero
    if (_localCache.TryGetValue(cacheKey, out int localCount))
    {
        if (localCount >= MAX_INVITATIONS_PER_ADMIN)
            return false;
    }

    // Si no está en cache local, consultar Redis
    var isAllowed = await CheckRedisAsync(tenantId, adminId, ipAddress, ct);
    
    if (isAllowed)
    {
        // Actualizar cache local
        _localCache.Set(cacheKey, localCount + 1, TimeSpan.FromMinutes(5));
    }

    return isAllowed;
}
```

### 3.4. Cuello de Botella #4: Job de Limpieza (Batch Processing)

**Problema:** El job de limpieza cada hora procesa invitaciones expiradas. Si hay miles de invitaciones, puede tomar minutos y bloquear la DB.

**Estrategia de Optimización:**

#### A. Procesamiento en Lotes con FOR UPDATE SKIP LOCKED
```sql
-- Función optimizada para el job de limpieza
CREATE OR REPLACE FUNCTION cleanup_expired_invitations(
    p_batch_size INT DEFAULT 1000
)
RETURNS INT AS $$
DECLARE
    v_expired_count INT;
BEGIN
    WITH expired_invitations AS (
        SELECT id 
        FROM invitations 
        WHERE status = 'PENDING' 
          AND expires_at < NOW()
        LIMIT p_batch_size
        FOR UPDATE SKIP LOCKED -- Prevenir bloqueos con otras transacciones
    )
    UPDATE invitations 
    SET status = 'EXPIRED'
    WHERE id IN (SELECT id FROM expired_invitations)
    RETURNING id;
    
    GET DIAGNOSTICS v_expired_count = ROW_COUNT;
    
    RETURN v_expired_count;
END;
$$ LANGUAGE plpgsql;
```

#### B. Ejecución en Background con Prioridad Baja
```csharp
public class InvitationCleanupJob : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                // Esperar 1 hora
                await Task.Delay(TimeSpan.FromHours(1), stoppingToken);

                // Ejecutar con prioridad baja (no afectar operaciones críticas)
                using var scope = _serviceProvider.CreateScope();
                var repository = scope.ServiceProvider.GetRequiredService<IInvitationRepository>();
                
                int totalExpired = 0;
                bool hasMore = true;

                while (hasMore)
                {
                    var expiredCount = await repository.ExpirePendingInvitationsAsync(
                        batchSize: 1000, 
                        stoppingToken);
                    
                    totalExpired += expiredCount;
                    hasMore = expiredCount == 1000; // Si procesó 1000, puede haber más
                    
                    // Pausa entre lotes para no saturar DB
                    if (hasMore)
                        await Task.Delay(TimeSpan.FromSeconds(1), stoppingToken);
                }

                _logger.LogInformation(
                    "Invitation cleanup job completed. Total expired: {Total}", 
                    totalExpired);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error in invitation cleanup job");
            }
        }
    }
}
```

### 3.5. Cuello de Botella #5: Integración con UC-06 (Registro)

**Problema:** La Fase 2 delega al flujo de registro (UC-06) que tiene su propia latencia (~400ms). Si UC-06 falla, la invitación permanece `PENDING` pero el usuario ya recibió error.

**Estrategia de Optimización:**

#### A. Timeout Estricto para UC-06
```csharp
public async Task<AcceptInvitationResult> AcceptInvitation(string providedToken, CancellationToken ct)
{
    // ... validación de token ...

    // Timeout estricto para UC-06
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
    cts.CancelAfter(TimeSpan.FromSeconds(5)); // 5 segundos máximo

    try
    {
        identityId = await _registrationService.CompleteRegistrationFromInvitationAsync(
            invitation.Email,
            invitation.Roles,
            cts.Token);
    }
    catch (OperationCanceledException)
    {
        _logger.LogWarning("UC-06 integration timeout for invitation {InvitationId}", invitation.Id);
        throw new InvitationAcceptanceTimeoutException("Registration took too long. Please try again.");
    }

    // ... continuar con aceptación ...
}
```

#### B. Circuit Breaker para UC-06
```csharp
var uc06CircuitBreaker = Policy
    .Handle<RegistrationException>()
    .CircuitBreakerAsync(
        exceptionsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromMinutes(2),
        onBreak: (ex, duration) => 
            _logger.LogWarning("UC-06 Circuit Breaker OPEN for {Duration}", duration)
    );
```

---

## 4. Estrategia de Load Testing (Pruebas de Carga)

Utilizamos **k6** para simular escenarios realistas del flujo dual asíncrono.

### 4.1. Escenario 1: Spike de Invitaciones (Onboarding Masivo)

**Objetivo:** Validar que el sistema soporta un pico de 100 invitaciones/min durante 5 minutos sin caer.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');
const conversionRate = new Rate('conversions');

export const options = {
  scenarios: {
    invitation_spike: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 20 },   // Calentamiento
        { duration: '3m', target: 100 },  // Pico de onboarding masivo
        { duration: '1m', target: 100 },  // Mantener pico
        { duration: '1m', target: 0 },    // Enfriamiento
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<500'], // SLO de 500ms
    errors: ['rate<0.05'],             // Menos del 5% de errores
  },
};

export default function () {
  const payload = JSON.stringify({
    email: `user_${__VU}_${__ITER}@khcs.com`,
    roles: ['user', 'developer'],
    message: 'Welcome to the team!',
    ttl_days: 7
  });

  const res = http.post(`${__ENV.BASE_URL}/v1/invitations`, payload, {
    headers: { 
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${__ENV.ADMIN_TOKEN}`
    },
  });

  const success = check(res, {
    'invitation created (201)': (r) => r.status === 201,
    'has invitation_id': (r) => r.json('invitation_id') !== undefined,
  });

  if (!success) errorRate.add(1);

  sleep(0.6); // ~100 invitaciones/min por VU
}
```

### 4.2. Escenario 2: Degradación de Email Service

**Objetivo:** Validar que si SMTP se vuelve lento o cae, el sistema ejecuta compensación sin colapsar.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    slow_smtp: {
      executor: 'constant-vus',
      vus: 30,
      duration: '5m',
    },
  },
  thresholds: {
    // Esperamos que el timeout de 2s corte la conexión y ejecute compensación
    http_req_duration: ['p(95)<2500'], 
    http_req_failed: ['rate<0.30'], // Hasta 30% de fallos aceptables
  },
};

export default function () {
  const payload = JSON.stringify({
    email: `user_${__VU}_${__ITER}@khcs.com`,
    roles: ['user'],
    message: 'Welcome!',
    ttl_days: 7
  });

  const res = http.post(`${__ENV.BASE_URL}/v1/invitations`, payload, {
    headers: { 
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${__ENV.ADMIN_TOKEN}`
    },
    timeout: '3s', // Timeout estricto
  });

  // Aceptar tanto 201 (éxito) como 500 (compensación ejecutada)
  check(res, {
    'status is 201 or 500': (r) => r.status === 201 || r.status === 500,
  });

  sleep(1);
}
```

**Resultado Esperado:**
- La latencia se dispara a ~2.100ms (2s timeout + overhead).
- La tasa de compensación sube al 100%.
- El sistema **no se cae** gracias al Circuit Breaker.
- Las invitaciones se marcan como `CANCELLED` en DB.

### 4.3. Escenario 3: Aceptación Concurrente (Race Conditions)

**Objetivo:** Validar que si múltiples usuarios intentan aceptar la misma invitación simultáneamente, solo uno tiene éxito (OCC).

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    concurrent_acceptance: {
      executor: 'per-vu-iterations',
      vus: 10,
      iterations: 1,
      maxDuration: '30s',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<800'],
  },
};

// Todos los VUs intentan aceptar la misma invitación
const SHARED_INVITATION_TOKEN = __ENV.SHARED_TOKEN;

export default function () {
  const payload = JSON.stringify({
    token: SHARED_INVITATION_TOKEN
  });

  const res = http.post(`${__ENV.BASE_URL}/v1/invitations/accept`, payload, {
    headers: { 'Content-Type': 'application/json' },
  });

  // Solo uno debe tener éxito (200), los demás deben fallar (400)
  check(res, {
    'status is 200 or 400': (r) => r.status === 200 || r.status === 400,
  });

  sleep(0.1); // Intentos casi simultáneos
}
```

**Resultado Esperado:**
- 1 VU recibe 200 OK (aceptación exitosa).
- 9 VUs reciben 400 Bad Request (token ya usado o invitación ya aceptada).
- La invitación queda en estado `ACCEPTED` con un solo `identity_id`.

### 4.4. Escenario 4: Job de Limpieza Bajo Carga

**Objetivo:** Validar que el job de limpieza no afecta el rendimiento de operaciones críticas.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    cleanup_under_load: {
      executor: 'constant-vus',
      vus: 50,
      duration: '10m',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<500'], // Latencia no debe degradarse
  },
};

export default function () {
  // Simular operaciones normales mientras el job corre
  const payload = JSON.stringify({
    email: `user_${__VU}_${__ITER}@khcs.com`,
    roles: ['user'],
    message: 'Welcome!',
    ttl_days: 7
  });

  const res = http.post(`${__ENV.BASE_URL}/v1/invitations`, payload, {
    headers: { 
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${__ENV.ADMIN_TOKEN}`
    },
  });

  check(res, {
    'invitation created (201)': (r) => r.status === 201,
  });

  sleep(1);
}
```

**Resultado Esperado:**
- El job de limpieza procesa lotes de 1000 invitaciones cada hora.
- La latencia de creación se mantiene <500ms p95.
- No hay bloqueos en DB gracias a `FOR UPDATE SKIP LOCKED`.

---

## 5. Capacity Planning y Escalabilidad

### 5.1. Cálculo de Recursos para Validación de Tokens

Para soportar **100 aceptaciones/min** con SecureCompare (~5ms por validación):

1.  **CPU:** 100 validaciones/min × 5ms = 500ms de CPU por minuto = ~0.5 cores dedicados.
2.  **Memoria:** Cada validación requiere ~1KB para hashing SHA256. 100 validaciones/min = 100KB/min.

**Recomendación:**
- No se requieren recursos adicionales significativos. SecureCompare es ligero.

### 5.2. Cálculo de IOPS de Base de Datos

Por cada invitación creada (Fase 1):
1.  `SELECT` (unicidad email)
2.  `SELECT` (límite admin)
3.  `INSERT` (invitations)
4.  `INSERT` (audit_logs)
5.  `UPDATE` (solo si falla email - ~5% de los casos)

Total: ~4 escrituras + 2 lecturas por invitación.
Para 50 invitaciones/min = **200 IOPS de escritura + 100 IOPS de lectura**.

Por cada invitación aceptada (Fase 2):
1.  `SELECT` (buscar por token_hash)
2.  `UPDATE` (transición a ACCEPTED)
3.  `INSERT` (audit_logs)

Total: ~2 escrituras + 1 lectura por aceptación.
Para 100 aceptaciones/min = **200 IOPS de escritura + 100 IOPS de lectura**.

**Recomendación:**
- PostgreSQL con almacenamiento GP3 (AWS) configurado para **3.000 IOPS** y **250 MB/s throughput** para tener margen de seguridad.

### 5.3. Cálculo de Redis para Rate Limiting

Para 50 invitaciones/min con 3 consultas Redis por request:
- **Consultas:** 50 × 3 = 150 consultas/min = 2.5 ops/seg.
- **Memoria:** Cada clave de rate limiting ocupa ~100 bytes. Con 10.000 admins activos = 1MB.

**Recomendación:**
- Redis con 256MB de RAM es más que suficiente.
- Usar Redis Cluster si se espera crecimiento masivo.

### 5.4. Cálculo de Email Service

Para 50 invitaciones/min:
- **Emails:** 50 emails/min = 3.000 emails/hora = 72.000 emails/día.
- **SMTP Provider:** SendGrid permite 100.000 emails/día en plan básico.

**Recomendación:**
- Usar SendGrid o Amazon SES con límite de 100.000 emails/día.
- Implementar cola de emails (RabbitMQ/SQS) si se espera crecimiento.

---

## 6. Observabilidad y Profiling en Producción

### 6.1. Distributed Tracing (OpenTelemetry)

El Trace de una invitación debe descomponerse en dos fases:

**Fase 1 (Creación):**
```
HTTP POST /invitations (Root Span)
├── Rate Limit Check (Redis - 3 consultas)
├── Validate Tenant Config (DB)
├── Validate Email Uniqueness (DB)
├── Validate Admin Limit (DB)
├── Generate Token (CPU - SHA256)
├── DB Transaction
│   ├── Insert Invitation
│   └── Insert Audit Log
└── SMTP Send (External I/O)
    └── [Success] or [Failure -> DB Update CANCELLED]
```

**Fase 2 (Aceptación):**
```
HTTP POST /invitations/accept (Root Span)
├── Token Validation (CPU - SecureCompare)
├── DB Lookup (token_hash)
├── UC-06 Integration (External Service)
│   ├── Create Password
│   ├── Accept Terms
│   └── Create Identity
├── DB Update (ACCEPTED)
├── RBAC Assignment (MURA)
└── SMTP Send (Confirmation to Admin)
```

**Spans Críticos a Monitorear:**
1.  **Token Validation:** Si >30ms, revisar CPU del servidor.
2.  **Email Send:** Si >300ms, el Circuit Breaker debe actuar.
3.  **UC-06 Integration:** Si >400ms, revisar rendimiento del servicio de registro.
4.  **Rate Limit Check:** Si >20ms, revisar latencia de Redis.

### 6.2. Dashboards de Grafana (Específicos para UC-07)

#### Panel 1: Funnel de Conversión de Invitaciones
*   **Métricas:** `Created` → `Email Sent` → `Token Validated` → `UC-06 Completed` → `Accepted`.
*   **Objetivo:** Identificar en qué paso se pierden las invitaciones (ej. si muchas fallan en "Email Sent", el SMTP está caído).

#### Panel 2: Tasa de Conversión por Tenant
*   **Eje X:** Tenant ID.
*   **Eje Y:** Tasa de conversión (%).
*   **Alerta:** Si la tasa cae por debajo del 60%, investigar UX del flujo de aceptación o entregabilidad de emails.

#### Panel 3: Salud del Job de Limpieza
*   **Métricas:** `cleanup_job_duration`, `cleanup_job_errors_total`, `cleanup_job_last_run`.
*   **Alerta:** Si el job falla 3 veces consecutivas, las invitaciones expiradas se acumulan.

#### Panel 4: Coste de Compensación
*   **Métrica:** `rate(kagami_invitation_compensation_total[5m])`.
*   **Alerta:** Si >10%, el servicio SMTP está fallando intermitentemente. Revisar logs de SendGrid/SMTP.

#### Panel 5: Rate Limiting Health
*   **Métricas:** `rate_limit_cache_latency`, `rate_limit_hits_total`, `rate_limit_blocks_total`.
*   **Alerta:** Si la latencia de cache >20ms, revisar Redis.

### 6.3. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Email Service Down** | `email_send_errors_total > 5` en 5 min | **Crítica** | SMTP caído. Las invitaciones se están compensando. Revisar logs de SMTP. |
| **Baja Tasa de Conversión** | `conversion_rate < 0.60` en 7 días | Media | Los usuarios no están aceptando invitaciones. Revisar UX y entregabilidad. |
| **Alta Tasa de Compensación** | `compensation_rate > 0.10` en 1h | Alta | Email service inestable. Investigar causa raíz. |
| **Job de Limpieza Fallido** | `cleanup_job_errors_total > 3` en 1h | Alta | Job no está actualizando invitaciones expiradas. Revisar logs. |
| **Pico de Invitaciones** | `created_total > 100` en 10 min | Media | Posible onboarding masivo o abuso. Revisar admins con más invitaciones. |
| **Token Validation Slow** | `token_validation_latency_p95 > 30ms` | Media | Validación criptográfica lenta. Revisar CPU del servidor. |
| **Rate Limiting Slow** | `rate_limit_cache_latency_p95 > 20ms` | Media | Redis lento. Revisar conectividad y carga. |

---

## 7. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los objetivos de rendimiento iniciales (<500ms p95 para creación) que este documento formaliza. |
| `09_Modern_Artifacts.md` | Define las métricas de OpenTelemetry; este documento explica cómo usarlas para validar SLAs. |
| `10_UC_Specific_Data_Model.md` | Los índices únicos y la estructura de tablas definidos allí son críticos para cumplir los SLOs de latencia. |
| `11_UC_Specific_Security_Threats.md` | Las mitigaciones de "SecureCompare" y "Rate Limiting Multi-Capa" tienen un costo de rendimiento que se gestiona mediante optimizaciones. |
| `SHUGO_Performance_Testing_Strategy.md` (Transversal) | Este documento aplica la estrategia global al contexto específico de operaciones dual-actor asíncronas. |

---

## 8. Anexos: Configuración de Rendimiento

### 8.1. Configuración de Redis para Rate Limiting

```ini
# redis.conf
maxmemory 256mb
maxmemory-policy allkeys-lru
save "" # Deshabilitar persistencia (es caché)
timeout 300
tcp-keepalive 60
```

### 8.2. Configuración de PgBouncer para Invitaciones

```ini
[databases]
shugo_kagami = host=postgres-primary port=5432 dbname=shugo_kagami

[pgbouncer]
pool_mode = transaction
max_client_conn = 2000
default_pool_size = 100
reserve_pool_size = 50 # Reserva alta para picos de invitaciones
reserve_pool_timeout = 2
```

### 8.3. Configuración de Kubernetes HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kagami-invitation-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kagami-invitation
  minReplicas: 2
  maxReplicas: 10
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
          name: kagami_invitation_creation_latency
        target:
          type: AverageValue
          averageValue: "300" # Escalar si latencia p95 > 300ms
    - type: External
      external:
        metric:
          name: kagami_invitation_created_total
        target:
          type: AverageValue
          averageValue: "50" # Escalar si >50 invitaciones/min por pod
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

---
*Fin del Documento 12_UC_Specific_Performance_Benchmarks para UC-07*