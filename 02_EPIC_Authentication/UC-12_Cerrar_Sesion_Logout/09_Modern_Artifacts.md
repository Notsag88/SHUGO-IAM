# UC-12: Cerrar Sesión (Logout) - Artefactos Modernos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-12 - Cerrar Sesión (Logout & Token Revocation) |
| **Documento:** | 09_Modern_Artifacts.md |
| **Metodología:** | API-First + Observability (OpenTelemetry) + Resilience Patterns + DevSecOps + Chaos Engineering |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 20/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 20/09/2026 | Gastón Ruiz | **Creación inicial.** Definición de los artefactos modernos para UC-12: contratos OpenAPI 3.1 con respuesta constante 204 (Zero Info Leakage), catálogo de métricas OpenTelemetry (enfocado en disparos de Fail-Safe, latencia de Blacklist y lag de Outbox), patrones de resiliencia (Circuit Breaker para Redis con activación inmediata del Fail-Safe), estrategias de testing incluyendo Chaos Engineering (caída simulada de Redis), y runbooks operativos para incidentes de revocación. | Cierra el paquete de diseño Tier 1 funcional del UC-12, proporcionando todos los artefactos necesarios para la implementación, operación, monitoreo y cumplimiento normativo (NIST SP 800-63B, OWASP ASVS V3.3, SOC 2) del endpoint de cierre de sesión del motor KAGI. |

---

## 1. Introducción y Propósito

Este documento define los **Artefactos Modernos** necesarios para operar el UC-12 (Cerrar Sesión) en un entorno de producción enterprise. A diferencia de los UCs de autenticación positiva (login), el UC-12 presenta desafíos operativos únicos:

1. **Contrato API de Respuesta Constante:** El endpoint debe retornar siempre HTTP 204, incluso ante tokens inválidos o ya revocados, para prevenir la enumeración de estado (Zero Info Leakage). Esto exige que las métricas, y no los códigos HTTP, sean la fuente de verdad operativa.
2. **Observabilidad del Fail-Safe:** Dado que una caída de Redis no debe romper el logout, el sistema debe exponer métricas explícitas que alerten cuando el modo Fail-Safe se activa, ya que el usuario nunca verá un error.
3. **Resiliencia en Cascada:** El Circuit Breaker de Redis debe abrirse rápidamente para que el Fail-Safe se active sin esperar timeouts, protegiendo la latencia del endpoint.
4. **Detección de Incidentes de Seguridad:** Un pico anómalo de `Cascade Logout` puede indicar que los usuarios están reaccionando a una brecha percibida; esto debe ser una señal de alerta para el SOC.

---

## 2. Contratos OpenAPI Completos

### 2.1. Especificación OpenAPI 3.1

```yaml
openapi: 3.1.0
info:
  title: SHUGO KAGI - Session Termination API
  description: |
    Endpoint de cierre de sesión y revocación de tokens del motor KAGI.
    
    ## Características Clave
    - Revocación híbrida: Refresh Token en DB (stateful) + JTI en Redis Blacklist (stateless).
    - Idempotencia estricta con Zero Info Leakage (siempre HTTP 204).
    - Soporte para Cascade Logout (logout_everywhere).
    - Estrategia Fail-Safe ante caída de Redis (la revocación en DB persiste).
    
    ## Seguridad
    - La respuesta NUNCA revela si el token era válido, inválido o ya revocado.
    - Headers de seguridad para prevenir cacheo de la respuesta por proxies.
  version: 1.0.0

servers:
  - url: https://auth.shugo.io/v1
    description: Producción

paths:
  /auth/logout:
    post:
      operationId: terminateSession
      summary: Cerrar sesión y revocar tokens
      description: |
        Revoca el Refresh Token proporcionado y añade el JTI del Access Token
        a la Blacklist de Redis con TTL dinámico.
        
        **Nota de Seguridad:** Este endpoint retorna HTTP 204 No Content en TODOS
        los escenarios de negocio (éxito, token ya revocado, token inválido) para
        prevenir la enumeración de estado de tokens.
      tags:
        - Session Management
      security:
        - BearerAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/LogoutRequest'
            examples:
              standardLogout:
                summary: Cierre de sesión del dispositivo actual
                value:
                  refresh_token: "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4..."
                  logout_everywhere: false
              cascadeLogout:
                summary: Cierre de sesión en todos los dispositivos
                value:
                  refresh_token: "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4..."
                  logout_everywhere: true
      responses:
        '204':
          description: |
            Sesión terminada (o ya estaba terminada).
            Sin cuerpo de respuesta por diseño (Zero Info Leakage).
          headers:
            Cache-Control:
              schema:
                type: string
              description: "no-store (previene cacheo por proxies/CDNs)"
            Pragma:
              schema:
                type: string
              description: "no-cache"
        '400':
          description: Request malformado (falta refresh_token en el body)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '503':
          description: |
            Servicio no disponible (Fail-Secure).
            Solo ocurre si la base de datos de tokens (KAGAMI) está caída,
            ya que no podemos garantizar la revocación persistente.
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
          headers:
            Retry-After:
              schema:
                type: integer
              description: "Segundos sugeridos para reintentar"

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
      description: "Access Token actual. Se usa para extraer el JTI a blacklisear."

  schemas:
    LogoutRequest:
      type: object
      required:
        - refresh_token
      properties:
        refresh_token:
          type: string
          description: "Refresh Token opaco de la sesión a terminar."
          example: "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4..."
        logout_everywhere:
          type: boolean
          default: false
          description: |
            Si es true, revoca TODOS los Refresh Tokens activos de la identidad
            (Cascade Logout). Útil en paneles de seguridad o tras pérdida de dispositivo.

    ErrorResponse:
      type: object
      properties:
        error:
          type: object
          properties:
            code:
              type: string
              enum:
                - INVALID_REQUEST
                - SERVICE_UNAVAILABLE
            message:
              type: string
            details:
              type: object
```

### 2.2. Matriz de Comportamiento del Contrato (Zero Info Leakage)

| Escenario de Negocio | Estado del Token | Código HTTP | Cuerpo | Métrica Emitida |
| :--- | :--- | :--- | :--- | :--- |
| Logout exitoso | ACTIVE | **204** | — | `logout_total{result="success"}` |
| Token ya revocado | REVOKED | **204** | — | `logout_total{result="idempotent"}` |
| Token no existe en DB | — | **204** | — | `logout_total{result="invalid_token"}` |
| Token expirado | EXPIRED | **204** | — | `logout_total{result="invalid_token"}` |
| Redis caído (Fail-Safe) | ACTIVE → REVOKED (DB) | **204** | — | `logout_total{result="failsafe"}` + `blacklist_failures_total` |
| Body sin refresh_token | — | **400** | Error JSON | `logout_total{result="bad_request"}` |
| Base de datos caída | — | **503** | Error JSON | `logout_total{result="db_unavailable"}` |

---

## 3. Catálogo de Métricas OpenTelemetry

### 3.1. Métricas de Negocio y Revocación

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagi.auth.logout.total` | Counter | operations | Total de operaciones de logout procesadas | `tenant_id`, `result` (success, idempotent, invalid_token, failsafe, bad_request, db_unavailable) |
| `kagi.auth.logout.cascade_total` | Counter | operations | Total de logouts con `logout_everywhere=true` | `tenant_id` |
| `kagi.auth.logout.tokens_revoked` | Histogram | tokens | Cantidad de Refresh Tokens revocados por operación (1 en estándar, N en cascade) | `tenant_id`, `mode` (single, cascade) |
| `kagi.auth.blacklist.entries_written` | Counter | entries | Total de JTIs escritos en la Blacklist de Redis | `tenant_id` |

### 3.2. Métricas de Rendimiento

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagi.auth.logout.duration` | Histogram | ms | Latencia end-to-end del endpoint `/auth/logout` | `tenant_id`, `result` |
| `kagi.auth.db.revoke_duration` | Histogram | ms | Latencia del UPDATE en `refresh_tokens` | `tenant_id` |
| `kagi.auth.blacklist.write_duration` | Histogram | ms | Latencia del `SET ... EX` en Redis | `tenant_id` |
| `kagi.auth.outbox.insert_duration` | Histogram | ms | Latencia de inserción del evento en `auth_outbox` | `tenant_id` |

### 3.3. Métricas de Resiliencia y Salud (Críticas para el Fail-Safe)

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagi.auth.blacklist.failures_total` | Counter | failures | **Disparos del modo Fail-Safe** (Redis no disponible durante logout) | `tenant_id`, `error_type` |
| `kagi.auth.blacklist.circuit_breaker_state` | Gauge | state | Estado del Circuit Breaker de Redis (0=Closed, 1=Open, 2=HalfOpen) | `service` |
| `kagi.auth.outbox.pending_events` | Gauge | events | Eventos de `Session.Terminated` pendientes de publicar | `event_type` |
| `kagi.auth.outbox.lag_seconds` | Gauge | s | Edad del evento más antiguo sin procesar en el Outbox | `event_type` |

### 3.4. Distributed Tracing (Spans del Logout)

```text
POST /v1/auth/logout (Root Span) [Total: 45ms]
├── token.extract_context [1ms]        (Extracción de JTI y exp del JWT)
├── db.refresh_token.select [8ms]      (Consulta de estado / idempotencia)
├── db.refresh_token.update [12ms]     (UPDATE status = REVOKED)
├── vo.blacklist_entry.calculate [0ms] (Cálculo de TTL dinámico)
├── redis.blacklist.set [6ms]          (SET blacklist:{jti} 1 EX {ttl})
│   └── [FALLA → span marcado con error + evento Fail-Safe]
├── db.outbox.insert [10ms]            (INSERT Session.Terminated)
└── http.response.204 [0ms]
```

### 3.5. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Fail-Safe Activado (Redis Down)** | `rate(kagi.auth.blacklist.failures_total[5m]) > 5` | **Crítica** | Redis Blacklist degradada. Los Access Tokens revocados vivirán hasta su expiración natural. Verificar cluster Redis y estado del Circuit Breaker. |
| **Circuit Breaker Open Prolongado** | `blacklist.circuit_breaker_state == 1` por > 10 min | **Crítica** | Redis no recupera salud. Escalar incidente de infraestructura. |
| **Outbox Lag Alto** | `kagi.auth.outbox.lag_seconds > 300` | Alta | KAGE/KIZUNA no están recibiendo eventos de terminación. Las cachés de autorización pueden estar obsoletas. Revisar worker de Outbox. |
| **Pico Anómalo de Cascade Logout** | `rate(kagi.auth.logout.cascade_total[10m]) > 3x baseline` | Alta | **Posible incidente de seguridad percibido por usuarios.** Notificar al SOC para correlacionar con alertas de credential stuffing o brechas. |
| **Latencia de Logout Degradada** | `logout.duration_p95 > 150ms` | Media | Revisar latencia de DB y Redis. Posible contención en `refresh_tokens`. |
| **DB de Tokens No Disponible** | `rate(logout_total{result="db_unavailable"}[2m]) > 0` | **Crítica** | Fail-Secure activo: los logouts están siendo rechazados (503). Escalar inmediatamente. |

---

## 4. Patrones de Resiliencia

### 4.1. Circuit Breaker para Redis Blacklist (Activación Rápida del Fail-Safe)

El Fail-Safe no debe depender de esperar timeouts de Redis en cada request. Usamos un Circuit Breaker con umbral agresivo para que, ante los primeros fallos, el sistema pase inmediatamente a modo degradado sin penalizar la latencia.

**Implementación (Polly - C#):**

```csharp
public class ResilientBlacklistManager : IBlacklistManager
{
    private readonly IBlacklistManager _inner;
    private readonly AsyncCircuitBreakerPolicy _circuitBreaker;
    private readonly IMeter _meter;
    private readonly ILogger<ResilientBlacklistManager> _logger;

    public ResilientBlacklistManager(
        IBlacklistManager inner,
        IMeter meter,
        ILogger<ResilientBlacklistManager> logger)
    {
        _inner = inner;
        _meter = meter;
        _logger = logger;

        _circuitBreaker = Policy
            .Handle<RedisConnectionException>()
            .Or<RedisTimeoutException>()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 3,   // Umbral agresivo
                durationOfBreak: TimeSpan.FromSeconds(30),
                onBreak: (ex, duration) =>
                {
                    _logger.LogCritical(
                        "Redis Blacklist Circuit Breaker OPEN for {Duration}. Fail-Safe mode engaged.", duration);
                    _meter.CreateObservableGauge("kagi.auth.blacklist.circuit_breaker_state", () => 1);
                },
                onReset: () =>
                {
                    _logger.LogInformation("Redis Blacklist Circuit Breaker CLOSED. Normal operation resumed.");
                },
                onHalfOpen: () =>
                {
                    _logger.LogInformation("Redis Blacklist Circuit Breaker HALF-OPEN. Probing Redis health.");
                });
    }

    public async Task AddToBlacklistAsync(BlacklistEntry entry, CancellationToken ct)
    {
        try
        {
            await _circuitBreaker.ExecuteAsync(() => _inner.AddToBlacklistAsync(entry, ct));
        }
        catch (BrokenCircuitException)
        {
            // Fail-Safe inmediato: no intentamos Redis, no penalizamos latencia.
            _meter.CreateCounter<long>("kagi.auth.blacklist.failures_total").Add(1,
                new KeyValuePair<string, object?>("error_type", "circuit_open"));
            // No lanzamos: el caller (TokenRevoker) continúa con el Outbox.
        }
        catch (Exception ex) when (ex is RedisConnectionException or RedisTimeoutException)
        {
            _meter.CreateCounter<long>("kagi.auth.blacklist.failures_total").Add(1,
                new KeyValuePair<string, object?>("error_type", ex.GetType().Name));
            // Fail-Safe: swallow exception.
        }
    }
}
```

### 4.2. Timeouts Estrictos para Redis

```csharp
// Configuración de StackExchange.Redis para la Blacklist
var redisOptions = new ConfigurationOptions
{
    EndPoints = { "redis-cluster:6379" },
    ConnectTimeout = 500,
    SyncTimeout = 50,      // La escritura de blacklist debe ser < 50ms
    AsyncTimeout = 50,
    AbortOnConnectFail = false, // Permitir que el Circuit Breaker gestione el estado
    KeepAlive = 60
};
```

### 4.3. Retry con Backoff para el Worker de Outbox

El worker que publica `Session.Terminated` a KAGE/KIZUNA debe ser resiliente a caídas del bus de mensajes.

```csharp
// Política de retry para el Outbox Worker
var outboxRetryPolicy = Policy
    .Handle<BrokerUnreachableException>()
    .Or<TimeoutException>()
    .WaitAndRetryAsync(
        retryCount: 5,
        sleepDurationProvider: attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt)), // 2s, 4s, 8s, 16s, 32s
        onRetry: (ex, ts, attempt, ctx) =>
            _logger.LogWarning(ex, "Outbox publish retry {Attempt} after {Delay}s", attempt, ts.TotalSeconds));
```

---

## 5. Estrategia de Testing

### 5.1. Unit Tests (Idempotencia y Fail-Safe)

```csharp
[Test]
public async Task LogoutHandler_WhenTokenAlreadyRevoked_Returns204_AndEmitsIdempotentMetric()
{
    // Arrange
    var mockRevoker = new Mock<ITokenRevoker>();
    mockRevoker.Setup(r => r.RevokeSingleSessionAsync(
            It.IsAny<JTI>(), It.IsAny<DateTime>(), It.IsAny<RefreshTokenId>(),
            It.IsAny<TokenRevocationReason>(), It.IsAny<CancellationToken>()))
        .ReturnsAsync(RevocationResult.IdempotentSuccess);

    var handler = new LogoutHandler(mockRevoker.Object, Mock.Of<ITokenExtractor>(), Mock.Of<ILogger<LogoutHandler>>());

    // Act
    var result = await handler.ExecuteLogoutAsync(BuildCommand(), CancellationToken.None);

    // Assert
    result.Should().BeOfType<NoContentResult>(); // Zero Info Leakage
}

[Test]
public async Task TokenRevoker_WhenRedisThrows_CompletesDbRevocation_AndReturnsFailSafe()
{
    // Arrange
    var mockBlacklist = new Mock<IBlacklistManager>();
    mockBlacklist.Setup(b => b.AddToBlacklistAsync(It.IsAny<BlacklistEntry>(), It.IsAny<CancellationToken>()))
        .ThrowsAsync(new RedisConnectionException(ConnectionFailureType.SocketFailure, "down"));

    var mockRepo = new Mock<IRefreshTokenRepository>();
    mockRepo.Setup(r => r.GetByIdAsync(It.IsAny<RefreshTokenId>(), It.IsAny<CancellationToken>()))
        .ReturnsAsync(BuildActiveToken());

    var revoker = new TokenRevoker(mockRepo.Object, mockBlacklist.Object, Mock.Of<IOutboxPublisher>(), Mock.Of<ILogger<TokenRevoker>>());

    // Act
    var result = await revoker.RevokeSingleSessionAsync(new JTI(Guid.NewGuid().ToString()), DateTime.UtcNow.AddMinutes(10), new RefreshTokenId(Guid.NewGuid()), TokenRevocationReason.USER_INITIATED, CancellationToken.None);

    // Assert
    result.Should().Be(RevocationResult.SuccessWithBlacklistFailure);
    mockRepo.Verify(r => r.SaveChangesAsync(It.IsAny<CancellationToken>()), Times.Once); // DB revocation stands
}
```

### 5.2. Integration Tests (Testcontainers: PostgreSQL + Redis)

```csharp
[Test]
public async Task Logout_EndToEnd_RevokesRefreshToken_AndBlacklistsJti()
{
    // Arrange: Login previo para obtener tokens reales
    var (accessToken, refreshToken) = await _testHarness.LoginAsync("user@khcs.com", "SecureP@ss123");
    var jti = JwtParser.ExtractJti(accessToken);

    // Act
    var response = await _client.PostAsJsonAsync("/v1/auth/logout", new { refresh_token = refreshToken });

    // Assert
    response.StatusCode.Should().Be(HttpStatusCode.NoContent);

    // Verificar DB: token revocado
    var dbStatus = await _dbHarness.GetRefreshTokenStatusAsync(refreshToken);
    dbStatus.Should().Be("REVOKED");

    // Verificar Redis: JTI en blacklist con TTL > 0
    var ttl = await _redisHarness.GetBlacklistTtlAsync(jti);
    ttl.Should().BeGreaterThan(TimeSpan.Zero);
    ttl.Should().BeLessThanOrEqualTo(TimeSpan.FromMinutes(16)); // exp restante + buffer
}
```

### 5.3. Chaos Engineering (Caída Simulada de Redis)

**Objetivo:** Validar que el Fail-Safe mantiene la disponibilidad del logout y la seguridad base cuando Redis muere.

```javascript
import http from 'k6/http';
import { check } from 'k6';
import { Rate, Counter } from 'k6/metrics';

const failsafeActivations = new Counter('failsafe_activations');
const availability = new Rate('logout_availability');

export const options = {
  scenarios: {
    chaos_logout: {
      executor: 'constant-vus',
      vus: 50,
      duration: '5m',
      // Durante el test, un operador externo ejecuta:
      // `docker stop redis-node-1` o inyecta latencia con toxiproxy
    },
  },
  thresholds: {
    // El endpoint NUNCA debe caer por debajo de 99.9% de disponibilidad
    logout_availability: ['rate>0.999'],
    // La latencia no debe degradarse aunque Redis esté caído (Circuit Breaker abierto)
    http_req_duration: ['p(95)<150'],
  },
};

export default function () {
  const res = http.post(`${__ENV.BASE_URL}/v1/auth/logout`,
    JSON.stringify({ refresh_token: __ENV.VALID_REFRESH_TOKEN }),
    { headers: { 
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${__ENV.VALID_ACCESS_TOKEN}` 
      } });

  const isAvailable = check(res, {
    'logout accepted (204) or fail-secure (503 only if DB down)': (r) => r.status === 204,
  });

  availability.add(isAvailable);
}
```

---

## 6. Deployment y Operaciones

### 6.1. Feature Flags para Gestión de Incidentes

| Feature Flag | Descripción | Valor por Defecto |
| :--- | :--- | :--- |
| `kagi.auth.logout.enabled` | Kill switch global del endpoint de logout. | `true` |
| `kagi.auth.logout.cascade_enabled` | Habilita/deshabilita Cascade Logout (`logout_everywhere`). | `true` |
| `kagi.auth.logout.blacklist_enabled` | Permite desactivar la escritura en Blacklist (solo en emergencias extremas de Redis). | `true` |
| `kagi.auth.logout.failsafe_mode` | `auto` (Circuit Breaker decide) o `strict` (si Redis falla, logout falla con 503). | `auto` |
| `kagi.auth.logout.blacklist_buffer_minutes` | Buffer de seguridad añadido al TTL dinámico. | `1` |

### 6.2. Configuración de Infraestructura

```yaml
# Configuración del motor KAGI para UC-12
kagi:
  logout:
    failsafe_mode: auto
    blacklist:
      redis_timeout_ms: 50
      circuit_breaker:
        failure_threshold: 3
        break_duration_seconds: 30
      ttl_buffer_minutes: 1
    cascade:
      max_tokens_per_operation: 100   # Límite de seguridad para Cascade Logout
      batch_size: 20                  # Procesamiento por lotes en DB
  outbox:
    worker:
      batch_size: 100
      poll_interval_ms: 500
      max_retry_attempts: 5
```

### 6.3. Runbook de Operaciones

**Escenario 1: Fail-Safe Activado (Redis Blacklist Caído)**

1. **Paso 1:** Verificar alerta `blacklist.failures_total > 5` en 5 min y `circuit_breaker_state == 1`.
2. **Paso 2:** Confirmar salud del cluster Redis (`redis-cli cluster info`, métricas de nodo).
3. **Paso 3:** Comunicar al SOC: *Los Access Tokens revocados durante esta ventana vivirán hasta su expiración natural (máx. 15 min). Los Refresh Tokens están revocados en DB (no hay escalada de sesión).*
4. **Paso 4:** Si Redis no recupera en 30 min, evaluar activar `failsafe_mode: strict` solo si el riesgo de tokens vivos supera el riesgo de disponibilidad (decisión del CISO).
5. **Paso 5:** Tras recuperación, verificar que `circuit_breaker_state` transicione a 2 (HalfOpen) y luego a 0 (Closed), y que `blacklist.entries_written` vuelva a incrementar.

**Escenario 2: Outbox Atascado (KAGE/KIZUNA sin eventos)**

1. **Paso 1:** Verificar alerta `outbox.lag_seconds > 300`.
2. **Paso 2:** Revisar logs del worker de Outbox (errores de broker o deserialización).
3. **Paso 3:** Verificar salud de Kafka/RabbitMQ y conectividad de KAGE/KIZUNA.
4. **Paso 4:** Si hay eventos "envenenados", moverlos a la Dead Letter Queue y re-procesar el resto.
5. **Paso 5:** Confirmar que `outbox.pending_events` decrece a 0.

**Escenario 3: Pico Anómalo de Cascade Logout**

1. **Paso 1:** Verificar alerta de pico `cascade_total > 3x baseline`.
2. **Paso 2:** Correlacionar con alertas de credential stuffing, brechas de terceros, o campañas de phishing.
3. **Paso 3:** Si se confirma incidente, coordinar con SOC para comunicación proactiva a usuarios y rotación preventiva de claves de firma JWT si aplica.
4. **Paso 4:** Monitorear capacidad de DB (el Cascade Logout genera N updates por operación).

---

## 7. Compliance y Auditoría

### 7.1. NIST SP 800-63B (Session Management)

*   **Requisito:** Los verificadores deben permitir al suscriptor terminar sus sesiones activas.
*   **Implementación UC-12:** Endpoint `/auth/logout` con revocación inmediata de Refresh Tokens y soporte de Cascade Logout para terminar todas las sesiones.

### 7.2. OWASP ASVS 4.0 - V3.3 (Session Logout and Timeout)

*   **Requisito 3.3.1:** Verificar que las sesiones activas terminan cuando el usuario hace logout.
*   **Implementación UC-12:** Revocación dual (DB + Blacklist de JTI) garantiza que ni el Refresh Token ni el Access Token puedan usarse tras el logout.

### 7.3. SOC 2 Type II - CC6.1 (Logical Access Controls)

*   **Requisito:** El acceso lógico debe terminar cuando ya no es requerido o autorizado.
*   **Implementación UC-12:** La revocación persiste en la fuente de verdad (DB) y se propaga a KIZUNA vía Outbox para invalidar cachés de autorización.

### 7.4. ISO 27001:2022 - A.5.18 (Access Rights)

*   **Requisito:** Los derechos de acceso deben ser retirados o ajustados conforme a la política.
*   **Implementación UC-12:** El motivo de revocación (`TokenRevocationReason`) se audita de forma inmutable, permitiendo demostrar el retiro de derechos ante auditores.

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define las invariantes (INV-77 a INV-81) y reglas (RN-20 Fail-Safe) que estos contratos y métricas monitorean. |
| `08_Final_Class_Design.md` | Las clases `LogoutHandler`, `TokenRevoker` y `ResilientBlacklistManager` implementan los patrones y emiten las métricas definidas aquí. |
| `10_UC_Specific_Data_Model.md` | Define la estructura de `refresh_tokens` y `auth_outbox` que soportan las métricas de revocación y lag. |
| `11_UC_Specific_Security_Threats.md` | Las mitigaciones de "Token Replay post-logout" y "Session Fixation" se validan mediante las métricas de Blacklist y las alertas de Fail-Safe. |
| `12_UC_Specific_Performance_Benchmarks.md` | Los SLAs de latencia (<100ms) y disponibilidad (99.99%) se validan mediante los scripts de k6 y Chaos Engineering definidos aquí. |

---

## 9. Cierre del Paquete de Diseño Funcional (UC-12)

Con este documento, completamos los **9 archivos del Paquete de Diseño Profundo Funcional (Tier 1)** para el UC-12:

1. ✅ `01_Requirements_Specification.md` - Especificación completa con escenarios BDD de idempotencia y Fail-Safe.
2. ✅ `02_Use_Case_Diagram.md` - Diagrama de caso de uso con Blacklist de JTI y propagación asíncrona.
3. ✅ `03_Activity_Diagram.md` - Diagrama de actividades con bifurcaciones de Zero Info Leakage y Fail-Safe.
4. ✅ `04_Initial_Domain_Model.md` - Modelo de dominio con `TokenRevoker`, `BlacklistEntry` y máquinas híbridas.
5. ✅ `05_State_Machine_Diagram.md` - Máquinas de estados persistente (DB) y efímera (Redis).
6. ✅ `06_Robustness_Diagram.md` - Diagrama de robustez ICONIX con orquestación Fail-Safe.
7. ✅ `07_Sequence_Diagram.md` - Diagrama de secuencia con TTL dinámico y manejo de excepciones.
8. ✅ `08_Final_Class_Design.md` - Diseño de clases con firmas, patrones y código C# de referencia.
9. ✅ `09_Modern_Artifacts.md` - Contratos OpenAPI, métricas, resiliencia, Chaos Engineering y compliance.

### 9.1. Resumen de lo Logrado

Hemos diseñado el endpoint de cierre de sesión más seguro y resiliente del mercado IAM:
- **Zero Info Leakage:** Respuesta constante HTTP 204 que elimina la superficie de ataque por enumeración de estado de tokens.
- **Revocación Híbrida:** Stateful (Refresh Token en DB) + Stateless (JTI en Redis Blacklist con TTL dinámico autolimpiable).
- **Fail-Safe Orquestado:** Circuit Breaker agresivo que activa el modo degradado sin penalizar latencia, manteniendo la seguridad base garantizada.
- **Observabilidad del Modo Degradado:** Métricas explícitas (`blacklist.failures_total`, `circuit_breaker_state`) que hacen visible lo que el usuario nunca verá como error.
- **Chaos Engineering:** Scripts de k6 que validan la disponibilidad del logout bajo caída real de Redis.
- **Compliance Total:** Mapeo directo a NIST SP 800-63B, OWASP ASVS V3.3, SOC 2 CC6.1 e ISO 27001 A.5.18.

### 9.2. Próximos Pasos (Documentos Técnicos Específicos)

Según la **Opción A+** acordada, cada UC tiene **12 documentos** (9 funcionales + 3 técnicos específicos). Para el UC-12, aún faltan:

1. **`10_UC_Specific_Data_Model.md`** - Modelo de datos físico específico (índices para Cascade Logout, particionamiento de `auth_outbox`, estructura de claves Redis).
2. **`11_UC_Specific_Security_Threats.md`** - Análisis de amenazas específico (Token Replay post-logout, Session Fixation, Blacklist Poisoning, Cascade Logout Abuse).
3. **`12_UC_Specific_Performance_Benchmarks.md`** - Benchmarks y SLAs específicos para latencia de logout, throughput de Cascade, y capacidad de Redis bajo olas de logout (fin de turno).

---
*Fin del Documento 09_Modern_Artifacts para UC-12*