# UC-09: Verificación de Contactos - Benchmarks, SLAs y Estrategia de Rendimiento

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-09 - Verificación de Contactos (Out-of-Band Verification) |
| **Documento:** | 12_UC_Specific_Performance_Benchmarks.md |
| **Metodología:** | Performance Engineering + SRE (SLI/SLO/SLA) + Load Testing (k6) + Capacity Planning + Concurrency Benchmarking |
| **Estándares:** | ISO/IEC 25010 (Performance Efficiency), Google SRE Workbook, NIST SP 800-63B (Authentication Performance) |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 10/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 10/09/2026 | Gastón Ruiz | **Creación inicial.** Definición de SLAs, SLOs, estrategia de Load Testing (enfocada en picos de verificación, concurrencia atómica en Redis, latencia de gateways externos, y detección de fuerza bruta bajo carga), y planificación de capacidad para la arquitectura híbrida Redis + PostgreSQL del motor de verificación OOB. | Documento de referencia para SREs y Arquitectos de Rendimiento. Establece cómo mantener la latencia sub-100ms para iniciación, sub-50ms para validación, y garantizar la atomicidad de operaciones Redis bajo carga concurrente extrema. |

---

## 1. Introducción y Propósito

A diferencia de los UCs transaccionales estándar, el **UC-09 (Verificación de Contactos)** es un **servicio de dominio transversal** con características de rendimiento únicas y desafiantes:

1. **Arquitectura Híbrida (Redis + PostgreSQL):** Las operaciones críticas de validación ocurren en Redis (sub-milisegundo), mientras la auditoría se persiste en PostgreSQL (milisegundos). El rendimiento end-to-end depende de ambos sistemas.
2. **Operaciones Atómicas bajo Concurrencia:** Las operaciones `INCR` en Redis deben mantenerse atómicas incluso bajo miles de requests simultáneos. Cualquier degradación en la atomicidad compromete la seguridad (Anti-Brute Force).
3. **Dependencia de Gateways Externos:** Twilio (SMS) y SendGrid (Email) tienen latencias variables (100ms - 3000ms) y disponibilidad no garantizada. El sistema debe responder al usuario sin esperar confirmación del gateway.
4. **Generación Criptográfica:** La generación de OTPs con `RandomNumberGenerator` y el hashing SHA256 tienen costo CPU que debe ser monitoreado bajo carga.
5. **Servicio Transversal de Alta Demanda:** Consumido por UC-06, UC-08, UC-15, y UC-18, el UC-09 puede recibir picos de tráfico masivos durante eventos de onboarding corporativo o campañas de marketing.

Este documento aborda estos desafíos definiendo SLAs realistas, estrategias de optimización y pruebas de carga específicas para garantizar que el motor de verificación OOB sea rápido, seguro y escalable.

---

## 2. Service Level Indicators (SLIs) y Objectives (SLOs)

### 2.1. SLIs Específicos para UC-09

| SLI | Descripción | Fórmula / Medición |
| :--- | :--- | :--- |
| **Initiation Latency (p95)** | Tiempo total desde que el usuario solicita verificación hasta que recibe HTTP 202 (sin incluir envío a gateway). | `histogram_quantile(0.95, rate(kagami_verification_initiation_latency_bucket[5m]))` |
| **Completion Latency (p95)** | Tiempo total desde que el usuario envía el OTP hasta que recibe HTTP 200/400. | `histogram_quantile(0.95, rate(kagami_verification_completion_latency_bucket[5m]))` |
| **Redis Operation Latency (p95)** | Latencia de operaciones atómicas en Redis (INCR, GET, SET, DEL). | `histogram_quantile(0.95, rate(kagami_verification_redis_latency_bucket[5m]))` |
| **OTP Generation Latency (p95)** | Tiempo de generación criptográfica de OTP (RandomNumberGenerator + SHA256). | `histogram_quantile(0.95, rate(kagami_verification_otp_gen_latency_bucket[5m]))` |
| **SecureCompare Latency (p95)** | Tiempo de comparación en tiempo constante. | `histogram_quantile(0.95, rate(kagami_verification_secure_compare_latency_bucket[5m]))` |
| **SMS Gateway Latency (p95)** | Tiempo de aceptación del SMS por Twilio (asíncrono, no bloquea respuesta). | `histogram_quantile(0.95, rate(kagami_verification_sms_gateway_latency_bucket[5m]))` |
| **Email Gateway Latency (p95)** | Tiempo de aceptación del Email por SendGrid (asíncrono, no bloquea respuesta). | `histogram_quantile(0.95, rate(kagami_verification_email_gateway_latency_bucket[5m]))` |
| **Audit Log Insert Latency (p95)** | Tiempo de inserción en tabla de auditoría PostgreSQL. | `histogram_quantile(0.95, rate(kagami_verification_audit_insert_latency_bucket[5m]))` |
| **Verification Success Rate** | Porcentaje de verificaciones completadas exitosamente vs. iniciadas. | `rate(kagami_verification_completed_total[7d]) / rate(kagami_verification_initiated_total[7d])` |
| **Brute Force Detection Rate** | Porcentaje de ataques de fuerza bruta detectados y bloqueados. | `rate(kagami_verification_brute_force_detected_total[5m]) / rate(kagami_verification_invalid_attempts_total[5m])` |

### 2.2. SLOs y SLAs

| Métrica | SLO (Objetivo Interno - 95%) | SLA (Contrato Enterprise - 99%) | Justificación |
| :--- | :--- | :--- | :--- |
| **Latencia Iniciación (p95)** | **< 50 ms** | **< 100 ms** | Incluye Rate Limit check (~5ms), generación OTP (~2ms), persistencia Redis (~5ms), e incremento rate limit (~5ms). El envío a gateway es asíncrono. |
| **Latencia Completación (p95)** | **< 30 ms** | **< 50 ms** | Incluye GET Redis (~3ms), INCR atómico (~3ms), SecureCompare (~1ms), y DEL (~3ms). Operaciones puramente en Redis. |
| **Latencia Redis (p95)** | **< 5 ms** | **< 10 ms** | Redis en la misma VPC/AZ debe responder en sub-milisegundos. P95 < 5ms es estándar. |
| **Latencia OTP Generation (p95)** | **< 2 ms** | **< 5 ms** | RandomNumberGenerator + SHA256 son operaciones CPU rápidas. |
| **Latencia SecureCompare (p95)** | **< 1 ms** | **< 2 ms** | FixedTimeEquals compara 44 bytes (SHA256 Base64) en tiempo constante. |
| **Latencia SMS Gateway (p99)** | **< 2.000 ms** | **< 5.000 ms** | Twilio SLA. No bloquea respuesta al usuario (Fire-and-Forget). |
| **Latencia Email Gateway (p99)** | **< 1.000 ms** | **< 3.000 ms** | SendGrid SLA. No bloquea respuesta al usuario (Fire-and-Forget). |
| **Latencia Audit Insert (p95)** | **< 20 ms** | **< 50 ms** | INSERT en tabla particionada de PostgreSQL. |
| **Throughput Iniciación** | **500 req/seg** | **300 req/seg** | Limitado por Redis throughput y CPU de generación criptográfica. |
| **Throughput Completación** | **1.000 req/seg** | **500 req/seg** | Operaciones puramente en Redis, muy rápidas. |
| **Disponibilidad** | **99.99%** | **99.9%** | Crítico: si UC-09 cae, los usuarios no pueden registrarse, recuperar passwords ni hacer MFA. |
| **Tasa de Éxito** | **> 85%** | **> 75%** | Usuarios que completan verificación exitosamente dentro del TTL. |
| **Tasa de Detección Brute Force** | **100%** | **100%** | Todo intento de fuerza bruta debe ser detectado y bloqueado. No negociable. |

---

## 3. Cuellos de Botella y Estrategias de Optimización

### 3.1. Cuello de Botella #1: Redis Latency (Network Bound)

**Problema:** La completación de verificación depende 100% de Redis. Si Redis tiene latencia alta (network jitter, sobrecarga, failover), todo el flujo se degrada.

**Estrategia de Optimización:**

#### A. Redis en la Misma Zona de Disponibilidad
```yaml
# Kubernetes: Pod Affinity para co-localizar Verification Pods con Redis
affinity:
  podAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: redis-cluster
          topologyKey: topology.kubernetes.io/zone
```

#### B. Connection Pooling con StackExchange.Redis
```csharp
// Configuración optimizada de conexión Redis
var redisOptions = new ConfigurationOptions
{
    EndPoints = { "redis-cluster:6379" },
    Password = Environment.GetEnvironmentVariable("REDIS_PASSWORD"),
    ConnectTimeout = 2000,
    SyncTimeout = 1000,      // Timeout estricto para operaciones síncronas
    AsyncTimeout = 1000,     // Timeout estricto para operaciones asíncronas
    KeepAlive = 60,
    ConnectRetry = 3,
    AbortOnConnectFail = false,
    
    // Connection Pooling
    MinIdleConnections = 50,
    MaxConnections = 500,
    
    // Pipeline optimization
    AllowAdmin = false,
    DefaultDatabase = 0
};

var connection = ConnectionMultiplexer.Connect(redisOptions);
```

#### C. Pipelining para Operaciones Múltiples
```csharp
// En lugar de 3 round-trips separados, usar pipeline para 1 round-trip
public async Task<bool> ValidateOtpPipelinedAsync(string contactHash, string providedOtpHash, CancellationToken ct)
{
    var db = _redis.GetDatabase();
    var otpKey = $"otp:{contactHash}";
    var attemptsKey = $"attempts:{contactHash}";

    // Crear batch (pipeline)
    var batch = db.CreateBatch();
    
    var incrementTask = batch.StringIncrementAsync(attemptsKey);
    var getTask = batch.StringGetAsync(otpKey);
    
    batch.Execute(); // Un solo round-trip a Redis
    
    var newAttempts = await incrementTask;
    var storedHash = await getTask;
    
    // ... validación ...
}
```

### 3.2. Cuello de Botella #2: Generación Criptográfica (CPU Bound)

**Problema:** `RandomNumberGenerator.Create()` y `SHA256.HashData()` son operaciones CPU. Bajo carga extrema (500 req/seg), pueden saturar cores.

**Estrategia de Optimización:**

#### A. Reutilización de Instancias de RandomNumberGenerator
```csharp
// MALO: Crear nueva instancia por cada OTP (overhead de GC)
public OtpCode Generate()
{
    using var rng = RandomNumberGenerator.Create(); // Overhead!
    var bytes = new byte[4];
    rng.GetBytes(bytes);
    // ...
}

// BUENO: Usar RandomNumberGenerator.Fill (static, thread-safe, sin allocation)
public OtpCode Generate()
{
    var bytes = new byte[4];
    RandomNumberGenerator.Fill(bytes); // Static, thread-safe, zero allocation
    var otpValue = Math.Abs(BitConverter.ToInt32(bytes)) % 1000000;
    return new OtpCode(otpValue.ToString("D6"));
}
```

#### B. Object Pooling para SHA256
```csharp
// .NET 8+: SHA256.HashData es estático y optimizado internamente
// No requiere pooling manual
var hash = SHA256.HashData(Encoding.UTF8.GetBytes(otp)); // Ya está optimizado
```

### 3.3. Cuello de Botella #3: Envío a Gateways Externos (I/O Bound)

**Problema:** Twilio y SendGrid tienen latencias variables (100ms - 3000ms). Si el envío fuera síncrono, violaría el SLO de <50ms para iniciación.

**Estrategia de Optimización: Fire-and-Forget con Outbox Pattern**

```csharp
public async Task<VerificationRequestId> InitiateVerification(
    InitiateVerificationCommand command, CancellationToken ct)
{
    // 1-5: Validaciones, generación, persistencia en Redis (< 30ms total)
    // ...

    // 6: Envío asíncrono (Fire-and-Forget)
    // NO await - no bloquea la respuesta HTTP
    _ = Task.Run(async () =>
    {
        try
        {
            await _gatewayAdapter.SendAsync(
                command.ContactValue, 
                challengeValue, 
                CancellationToken.None);
            
            Metrics.GatewaySendSuccess.Add(1);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Gateway send failed for {ContactHash}", command.ContactHash);
            Metrics.GatewaySendFailure.Add(1);
            
            // Outbox Pattern: guardar en DB para reintento posterior
            await _outboxRepo.SaveAsync(new OutboxMessage
            {
                Type = "SEND_VERIFICATION",
                Payload = JsonSerializer.Serialize(new { command.ContactValue, challengeValue }),
                RetryCount = 0,
                NextRetryAt = DateTime.UtcNow.AddSeconds(30)
            });
        }
    });

    // 7: Responder inmediatamente (< 50ms)
    return verificationRequest.Id;
}
```

### 3.4. Cuello de Botella #4: Auditoría en PostgreSQL (Write Bound)

**Problema:** Cada verificación exitosa genera un INSERT en `verification_audit_logs`. Bajo picos de 500 req/seg, esto puede saturar la DB.

**Estrategia de Optimización: Escritura Asíncrona con Outbox**

```csharp
// La auditoría no bloquea la respuesta al usuario
// Se escribe en la misma transacción que el estado, pero de forma asíncrona
public async Task<bool> CompleteVerification(
    CompleteVerificationCommand command, CancellationToken ct)
{
    // 1-4: Validación en Redis (< 20ms)
    // ...

    // 5: Si éxito, marcar contacto como verificado + auditoría
    if (isValid)
    {
        // Outbox Pattern: insertar auditoría en misma transacción
        using var transaction = await _dbContext.Database.BeginTransactionAsync(ct);
        try
        {
            _dbContext.VerifiedContacts.Add(new VerifiedContact { /* ... */ });
            _dbContext.VerificationAuditLogs.Add(new VerificationAuditLog { /* ... */ });
            await _dbContext.SaveChangesAsync(ct);
            await transaction.CommitAsync(ct);
        }
        catch (Exception ex)
        {
            await transaction.RollbackAsync(ct);
            _logger.LogError(ex, "Audit log insert failed");
            // La verificación sigue siendo exitosa en Redis
            // El Outbox Pattern reintentará la auditoría
        }
    }

    return isValid;
}
```

---

## 4. Estrategia de Load Testing (Pruebas de Carga)

Utilizamos **k6** para simular escenarios realistas. Es crucial probar no solo el rendimiento, sino también la **atomicidad bajo concurrencia** y la **detección de fuerza bruta bajo carga**.

### 4.1. Escenario 1: Pico de Iniciación de Verificaciones

**Objetivo:** Validar que el sistema maneja 500 iniciaciones/segundo sin degradar la latencia por debajo del SLO de 50ms.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const initiationLatency = new Trend('initiation_latency', true);

export const options = {
  scenarios: {
    initiation_spike: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '1m', target: 50 },    // Calentamiento
        { duration: '3m', target: 500 },   // Pico de onboarding masivo
        { duration: '2m', target: 500 },   // Mantener pico
        { duration: '1m', target: 0 },     // Enfriamiento
      ],
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<100'],  // SLO de 100ms (SLA)
    errors: ['rate<0.01'],              // Menos del 1% de errores
  },
};

export default function () {
  const payload = JSON.stringify({
    contact_type: 'EMAIL',
    contact_value: `user_${__VU}_${__ITER}@khcs.com`,
    channel: 'EMAIL'
  });

  const res = http.post(`${__ENV.BASE_URL}/v1/verification/initiate`, payload, {
    headers: { 
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${__ENV.USER_TOKEN}`
    },
  });

  initiationLatency.add(res.timings.duration);

  const success = check(res, {
    'verification initiated (202)': (r) => r.status === 202,
    'has expires_in': (r) => r.json('expires_in_seconds') !== undefined,
    'latency < 50ms': (r) => r.timings.duration < 50,
  });

  if (!success) errorRate.add(1);

  sleep(0.1); // ~10 req/seg por VU
}
```

### 4.2. Escenario 2: Concurrencia Atómica en Redis (Race Condition Test)

**Objetivo:** Validar que el contador de intentos en Redis es verdaderamente atómico bajo carga concurrente extrema. Este es el test de seguridad más crítico.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Counter } from 'k6/metrics';

const bruteForceBypassed = new Counter('brute_force_bypassed');

export const options = {
  scenarios: {
    concurrent_validation: {
      executor: 'per-vu-iterations',
      vus: 50,           // 50 VUs simultáneos
      iterations: 10,    // Cada VU envía 10 requests = 500 requests totales
      maxDuration: '30s',
    },
  },
  thresholds: {
    // Todos deben recibir 400 (invalid) o 429 (brute force)
    // NUNCA 200 (success) porque el OTP es incorrecto
    http_req_failed: ['rate==0'], // Ningún request debe fallar a nivel de red
  },
};

// Todos los VUs intentan validar el MISMO OTP incorrecto simultáneamente
const SHARED_CONTACT_HASH = __ENV.TEST_CONTACT_HASH;
const WRONG_OTP = '000000';

export default function () {
  const payload = JSON.stringify({
    contact_hash: SHARED_CONTACT_HASH,
    provided_challenge: WRONG_OTP
  });

  const res = http.post(`${__ENV.BASE_URL}/v1/verification/complete`, payload, {
    headers: { 
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${__ENV.USER_TOKEN}`
    },
  });

  // Verificar que NINGÚN request recibió 200 (éxito con OTP incorrecto)
  if (res.status === 200) {
    bruteForceBypassed.add(1);
    console.error('CRITICAL: Brute force bypass detected! OTP validation succeeded with wrong code.');
  }

  check(res, {
    'status is 400 or 429': (r) => r.status === 400 || r.status === 429,
    'no success with wrong OTP': (r) => r.status !== 200,
  });

  sleep(0.01); // Intentos casi simultáneos (10ms entre cada uno)
}
```

**Resultado Esperado:**
- Los primeros 5 requests reciben HTTP 400 "Invalid code".
- Los requests 6+ reciben HTTP 429 "Too many invalid attempts" (brute force detected).
- **Ningún request recibe HTTP 200** (el OTP es incorrecto).
- Si algún request recibe 200, hay un **bug crítico de seguridad** (Race Condition en el contador de intentos).

### 4.3. Escenario 3: Degradación de Gateway Externo

**Objetivo:** Validar que si Twilio/SendGrid se vuelve lento o cae, el endpoint de iniciación sigue respondiendo en <100ms (gracias al Fire-and-Forget).

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    gateway_degradation: {
      executor: 'constant-vus',
      vus: 100,
      duration: '5m',
    },
  },
  thresholds: {
    // El endpoint de iniciación debe responder rápido
    // incluso si el gateway está caído (Fire-and-Forget)
    http_req_duration: ['p(95)<100'],
    http_req_failed: ['rate<0.05'],
  },
};

export default function () {
  const payload = JSON.stringify({
    contact_type: 'PHONE',
    contact_value: `+54911555${String(__VU).padStart(4, '0')}`,
    channel: 'SMS'
  });

  const res = http.post(`${__ENV.BASE_URL}/v1/verification/initiate`, payload, {
    headers: { 
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${__ENV.USER_TOKEN}`
    },
    timeout: '2s', // Timeout estricto
  });

  // Aceptar tanto 202 (éxito) como 500 (gateway falló pero se manejó)
  check(res, {
    'status is 202 or 500': (r) => r.status === 202 || r.status === 500,
    'latency < 100ms regardless of gateway': (r) => r.timings.duration < 100,
  });

  sleep(0.5);
}
```

**Resultado Esperado:**
- La latencia se mantiene <100ms p95 **incluso si el gateway está caído**.
- Si el gateway falla, el sistema usa Outbox Pattern para reintentar.
- El usuario recibe HTTP 202 (el envío es asíncrono).

### 4.4. Escenario 4: Ataque de SMS Bombing

**Objetivo:** Validar que el Rate Limiting previene el abuso masivo de SMS.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const rateLimited = new Rate('rate_limited');
const smsSent = new Rate('sms_sent');

export const options = {
  scenarios: {
    sms_bombing: {
      executor: 'constant-vus',
      vus: 20,
      duration: '2m',
    },
  },
};

// Todos los VUs atacan el MISMO número de teléfono
const VICTIM_PHONE = '+5491155556666';

export default function () {
  const payload = JSON.stringify({
    contact_type: 'PHONE',
    contact_value: VICTIM_PHONE,
    channel: 'SMS'
  });

  const res = http.post(`${__ENV.BASE_URL}/v1/verification/initiate`, payload, {
    headers: { 
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${__ENV.USER_TOKEN}`
    },
  });

  if (res.status === 429) {
    rateLimited.add(1);
  } else if (res.status === 202) {
    smsSent.add(1);
  }

  check(res, {
    'status is 202 or 429': (r) => r.status === 202 || r.status === 429,
  });

  sleep(0.1); // 10 requests/seg por VU = 200 requests/seg totales
}
```

**Resultado Esperado:**
- Los primeros 5 requests reciben HTTP 202 (SMS enviado).
- Los requests 6+ reciben HTTP 429 (rate limit excedido).
- **Máximo 5 SMS enviados al número de la víctima**, independientemente de cuántos requests se envíen.
- La tasa `rate_limited` debe ser >95% del total.

---

## 5. Capacity Planning y Escalabilidad

### 5.1. Cálculo de Recursos para Redis

Para soportar **1.000 validaciones/segundo** (operaciones puramente en Redis):

1. **Operaciones por validación:** 1 GET + 1 INCR + 1 DEL (éxito) o 1 GET + 1 INCR (fallo) = 2-3 ops.
2. **Total ops/seg:** 1.000 × 3 = 3.000 ops/seg.
3. **Redis throughput:** Un solo nodo Redis maneja ~100.000 ops/seg.

**Recomendación:**
- **Redis Cluster con 3 nodos** (master + 2 replicas) para alta disponibilidad.
- **Memoria:** 2GB por nodo (suficiente para millones de claves OTP con TTL).
- **Network:** Redis en la misma VPC/AZ que los pods de verificación.

### 5.2. Cálculo de Recursos para PostgreSQL (Auditoría)

Para soportar **500 verificaciones exitosas/segundo** (cada una genera 1 INSERT):

1. **IOPS de escritura:** 500 INSERTs/seg = 500 IOPS.
2. **IOPS de lectura:** Consultas forenses ocasionales = ~50 IOPS.
3. **Total:** ~550 IOPS.

**Recomendación:**
- **PostgreSQL db.r5.large** (2 vCPUs, 16GB RAM) con GP3 (3.000 IOPS base).
- **Particionamiento mensual** de `verification_audit_logs` para mantener índices pequeños.
- **Read Replica** para consultas forenses sin afectar escrituras.

### 5.3. Cálculo de CPU para Generación Criptográfica

Para soportar **500 iniciaciones/segundo**:

1. **RandomNumberGenerator.Fill:** ~0.5μs por llamada.
2. **SHA256.HashData:** ~2μs por llamada.
3. **Total por iniciación:** ~5μs de CPU.
4. **CPU total:** 500 × 5μs = 2.5ms de CPU por segundo = ~0.25% de un core.

**Recomendación:** La generación criptográfica **no es un cuello de botella**. Un solo core puede manejar >100.000 generaciones/segundo.

### 5.4. Escalado Horizontal de Verification Handlers

```yaml
# Kubernetes HPA para Verification Handlers
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kagami-verification-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kagami-verification
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
          name: kagami_verification_completion_latency
        target:
          type: AverageValue
          averageValue: "30" # Escalar si latencia p95 > 30ms
    - type: External
      external:
        metric:
          name: kagami_verification_initiated_total
        target:
          type: AverageValue
          averageValue: "500" # Escalar si >500 iniciaciones/seg por pod
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30  # Escalar rápido (servicio crítico)
      policies:
        - type: Percent
          value: 100  # Duplicar pods rápidamente
          periodSeconds: 30
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

---

## 6. Observabilidad y Profiling en Producción

### 6.1. Distributed Tracing (OpenTelemetry)

El Trace de una verificación debe descomponerse en spans granulares:

**Iniciación:**
```
HTTP POST /verification/initiate (Root Span) [Total: 45ms]
├── Rate Limit Check (Redis GET) [3ms]
├── OTP Generation (CPU) [1ms]
├── SHA256 Hashing (CPU) [1ms]
├── Redis SET with TTL [4ms]
├── Redis INCR (Rate Limit) [3ms]
├── Fire-and-Forget Gateway Send [0ms - async]
└── HTTP Response 202 [0ms]
```

**Completación:**
```
HTTP POST /verification/complete (Root Span) [Total: 25ms]
├── Redis GET (Challenge Hash) [3ms]
├── Redis INCR (Attempts - Atomic) [3ms]
├── SecureCompare (CPU) [0.5ms]
├── Redis DEL (Invalidate OTP) [3ms]
├── Redis DEL (Cleanup Attempts) [3ms]
├── PostgreSQL INSERT (Audit Log) [15ms]
└── HTTP Response 200 [0ms]
```

**Spans Críticos a Monitorear:**
1. **Redis INCR:** Si >5ms, hay contención en Redis o problema de red.
2. **SecureCompare:** Si >1ms, hay problema de CPU o implementación incorrecta.
3. **PostgreSQL INSERT:** Si >20ms, la tabla de auditoría necesita mantenimiento (VACUUM) o el particionamiento no funciona.
4. **Gateway Send (async):** Si falla >10% de las veces, revisar Circuit Breaker y estado de Twilio/SendGrid.

### 6.2. Dashboards de Grafana (Específicos para UC-09)

#### Panel 1: Latencia de Verificación (Iniciación vs Completación)
*   **Métricas:** `initiation_latency` (p50, p95, p99) vs `completion_latency` (p50, p95, p99).
*   **Objetivo:** Iniciación <50ms p95, Completación <30ms p95.
*   **Alerta:** Si completación >50ms p95, revisar Redis.

#### Panel 2: Atomicidad de Redis (Seguridad)
*   **Métricas:** `brute_force_bypassed_total` (debe ser SIEMPRE 0).
*   **Objetivo:** Cero bypasses de fuerza bruta. Cualquier valor >0 es una **alerta crítica de seguridad**.
*   **Alerta:** `brute_force_bypassed_total > 0` → **P0 Incident**.

#### Panel 3: Gateway Health
*   **Métricas:** `sms_gateway_latency` (p95), `email_gateway_latency` (p95), `gateway_errors_total`.
*   **Objetivo:** Latencia <2s p95, errores <1%.
*   **Alerta:** Si Circuit Breaker se abre, notificar a operaciones.

#### Panel 4: Tasa de Éxito de Verificación
*   **Métricas:** `verification_success_rate` por canal (SMS vs Email).
*   **Objetivo:** >85% de éxito.
*   **Alerta:** Si tasa <70%, revisar entregabilidad de emails/SMS o UX del formulario.

#### Panel 5: Rate Limiting Effectiveness
*   **Métricas:** `rate_limit_exceeded_total` vs `initiated_total`.
*   **Objetivo:** La mayoría de requests deben pasar. Si >50% son bloqueados, posible ataque.
*   **Alerta:** Si rate_limit_exceeded >100/min, revisar IPs ofensivas.

### 6.3. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Brute Force Bypass** | `brute_force_bypassed_total > 0` | **P0 - Crítica** | **BUG DE SEGURIDAD.** La atomicidad de Redis falló. Investigar inmediatamente. |
| **Redis Latency High** | `redis_operation_latency_p95 > 10ms` | Alta | Redis lento. Revisar conectividad, memoria, y CPU del cluster. |
| **SMS Gateway Down** | `sms_gateway_errors_total > 10` en 5 min | Alta | Twilio caído. Circuit Breaker debe abrirse automáticamente. |
| **Email Gateway Down** | `email_gateway_errors_total > 10` en 5 min | Alta | SendGrid caído. Circuit Breaker debe abrirse automáticamente. |
| **Audit Log Lag** | `audit_log_insert_latency_p95 > 50ms` | Media | PostgreSQL lento. Revisar particiones y VACUUM. |
| **Low Verification Rate** | `verification_success_rate < 0.70` en 1h | Media | Los usuarios no completan verificaciones. Revisar UX y entregabilidad. |
| **High Rate Limiting** | `rate_limit_exceeded_total > 100` en 5 min | Media | Posible ataque de SMS/Email Bombing. Revisar IPs ofensivas. |
| **OTP Generation Slow** | `otp_generation_latency_p95 > 5ms` | Baja | CPU saturado. Revisar carga del servidor. |

---

## 7. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los objetivos de rendimiento iniciales (<100ms p95 para iniciación, <50ms para validación) que este documento formaliza. |
| `08_Final_Class_Design.md` | Las clases `VerificationHandler`, `RateLimiter`, `SecureCompare`, y `OtpGenerator` deben implementar las optimizaciones definidas aquí. |
| `10_UC_Specific_Data_Model.md` | La estructura de claves Redis y las políticas de TTL definidas allí son críticas para cumplir los SLAs de latencia. |
| `11_UC_Specific_Security_Threats.md` | Las mitigaciones de "Race Conditions" y "OTP Brute-Forcing" se validan mediante el Escenario 2 de Load Testing (Concurrencia Atómica). |
| `SHUGO_Performance_Testing_Strategy.md` (Transversal) | Este documento aplica la estrategia global al contexto específico de operaciones atómicas en Redis y servicios transversales de alta demanda. |

---

## 8. Anexos: Configuración de Rendimiento

### 8.1. Configuración de Redis para Alto Throughput

```ini
# redis.conf - Optimizaciones para UC-09

# Memoria
maxmemory 2gb
maxmemory-policy allkeys-lru

# Persistencia deshabilitada (OTPs son efímeros)
save ""
appendonly no

# Performance
tcp-keepalive 300
timeout 0
tcp-backlog 511

# Cluster
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000

# Logging
loglevel notice
slowlog-log-slower-than 1000  # Log comandos > 1ms
slowlog-max-len 128
```

### 8.2. Configuración de PostgreSQL para Auditoría

```ini
# postgresql.conf - Optimizaciones para UC-09

# WAL y Checkpoints
wal_level = replica
max_wal_size = 4GB
min_wal_size = 1GB
checkpoint_completion_target = 0.9
checkpoint_timeout = 10min
wal_buffers = 64MB

# Autovacuum agresivo para tabla de auditoría
autovacuum_max_workers = 5
autovacuum_vacuum_scale_factor = 0.01
autovacuum_analyze_scale_factor = 0.005

# Memoria
shared_buffers = 8GB
effective_cache_size = 24GB
work_mem = 128MB
```

### 8.3. Configuración de Connection Pooling (PgBouncer)

```ini
# pgbouncer.ini

[databases]
shugo_kagami = host=postgres-primary port=5432 dbname=shugo_kagami

[pgbouncer]
pool_mode = transaction
max_client_conn = 2000
default_pool_size = 100
min_pool_size = 20
reserve_pool_size = 50
reserve_pool_timeout = 2
server_connect_timeout = 5
query_timeout = 10
```

### 8.4. Benchmark de Referencia (Resultados Esperados)

| Operación | p50 | p95 | p99 | Throughput Máximo |
| :--- | :--- | :--- | :--- | :--- |
| **Redis GET** | 0.5ms | 2ms | 5ms | 100.000 ops/seg |
| **Redis INCR** | 0.5ms | 2ms | 5ms | 100.000 ops/seg |
| **Redis SET + EXPIRE** | 0.8ms | 3ms | 8ms | 80.000 ops/seg |
| **Redis DEL** | 0.3ms | 1ms | 3ms | 150.000 ops/seg |
| **OTP Generation** | 0.3ms | 1ms | 2ms | 200.000 ops/seg |
| **SHA256 Hash** | 0.5ms | 1.5ms | 3ms | 150.000 ops/seg |
| **SecureCompare** | 0.1ms | 0.5ms | 1ms | 500.000 ops/seg |
| **PG INSERT (Audit)** | 5ms | 15ms | 30ms | 5.000 ops/seg |
| **Iniciación E2E** | 20ms | 45ms | 80ms | 500 req/seg |
| **Completación E2E** | 10ms | 25ms | 40ms | 1.000 req/seg |

---
*Fin del Documento 12_UC_Specific_Performance_Benchmarks para UC-09*