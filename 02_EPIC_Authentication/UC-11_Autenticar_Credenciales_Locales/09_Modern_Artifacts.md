# UC-11: Autenticar Credenciales Locales - Artefactos Modernos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-11 - Autenticar Credenciales Locales (Local Authentication) |
| **Documento:** | 09_Modern_Artifacts.md |
| **Metodología:** | API-First + Observability (OpenTelemetry) + Resilience Patterns + DevSecOps + Zero Trust |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 15/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 15/09/2026 | Gastón Ruiz | **Creación inicial.** Definición de los artefactos modernos para UC-11: contratos OpenAPI 3.1 con prevención explícita de enumeración, catálogo de métricas OpenTelemetry (enfocado en detección de Timing Attacks, Credential Stuffing y rendimiento de Argon2id), patrones de resiliencia (Circuit Breaker para Redis/KAGAMI con estrategia Fail-Secure), Feature Flags para agilidad criptográfica, y mapeo de cumplimiento (NIST SP 800-63B, OWASP ASVS). | Cierra el paquete de diseño Tier 1 funcional del UC-11, proporcionando todos los artefactos necesarios para la implementación, operación, monitoreo y cumplimiento normativo de la puerta de entrada crítica del motor KAGI. |

---

## 1. Introducción y Propósito

Este documento define los **Artefactos Modernos** necesarios para operar el UC-11 (Autenticar Credenciales Locales) en un entorno de producción enterprise de alta seguridad. A diferencia de los UCs de gestión de identidad, el UC-11 es el punto de mayor exposición a ataques automatizados y requiere artefactos específicos para:

1. **Contratos API Anti-Enumeración:** Definir respuestas HTTP que no filtren información sobre la existencia de usuarios.
2. **Observabilidad de Seguridad:** Métricas y trazas que detecten ataques de Credential Stuffing, Brute Force y Timing Attacks en tiempo real.
3. **Resiliencia Fail-Secure:** Patrones que garanticen que, si los sistemas de apoyo (Redis, KAGAMI) fallan, el sistema deniegue el acceso por defecto para proteger al usuario.
4. **Agilidad Criptográfica:** Feature Flags para ajustar los parámetros de Argon2id o las políticas de Lockout sin redeploy.

---

## 2. Contratos OpenAPI Completos

### 2.1. Especificación OpenAPI 3.1 (Extracto Crítico)

```yaml
openapi: 3.1.0
info:
  title: SHUGO KAGI - Local Authentication API
  description: |
    Endpoint de autenticación local basado en credenciales (Email/Password).
    Diseñado con principios Zero Trust y prevención de enumeración de usuarios.
  version: 1.0.0

paths:
  /v1/auth/login:
    post:
      operationId: authenticateLocal
      summary: Autenticar credenciales locales
      description: |
        Valida email y password. Si es exitoso y no requiere MFA, emite tokens.
        Si requiere MFA, retorna un challenge token.
        **Nota de Seguridad:** Los errores 401 son genéricos para prevenir enumeración.
      tags:
        - Authentication
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/LoginRequest'
      responses:
        '200':
          description: Autenticación exitosa o Challenge de MFA requerido
          content:
            application/json:
              schema:
                oneOf:
                  - $ref: '#/components/schemas/SuccessResponse'
                  - $ref: '#/components/schemas/MfaChallengeResponse'
        '401':
          description: Credenciales inválidas o cuenta no encontrada
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/GenericErrorResponse'
        '423':
          description: Cuenta bloqueada temporalmente
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/LockedErrorResponse'
        '429':
          description: Demasiados intentos (Rate Limit de IP)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/RateLimitErrorResponse'
        '503':
          description: Servicio no disponible (Fallo en sistemas de seguridad)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/GenericErrorResponse'

components:
  schemas:
    LoginRequest:
      type: object
      required: [email, password]
      properties:
        email:
          type: string
          format: email
          example: "user@khcs.com"
        password:
          type: string
          format: password
          minLength: 12
          example: "SecureP@ssw0rd!"
        device_id:
          type: string
          description: "Fingerprint único del dispositivo para vincular Refresh Token"
          example: "d4f5a6b7-c8d9-e0f1-a2b3-c4d5e6f7a8b9"

    SuccessResponse:
      type: object
      properties:
        access_token:
          type: string
          description: "JWT Bearer Token (AAL1 o AAL2)"
        token_type:
          type: string
          enum: [Bearer]
        expires_in:
          type: integer
          description: "Tiempo de vida en segundos (ej. 900)"
        refresh_token:
          type: string
          description: "Token opaco de larga duración (solo se entrega una vez)"

    MfaChallengeResponse:
      type: object
      properties:
        mfa_required:
          type: boolean
          example: true
        mfa_token:
          type: string
          description: "Token de sesión temporal (5 min) para completar UC-15"
        allowed_methods:
          type: array
          items:
            type: string
            enum: [totp, webauthn, email_otp]

    GenericErrorResponse:
      type: object
      properties:
        error:
          type: string
          enum: [invalid_grant]
        error_description:
          type: string
          example: "Invalid credentials" # NUNCA "User not found"

    LockedErrorResponse:
      type: object
      properties:
        error:
          type: string
          enum: [account_locked]
        error_description:
          type: string
          example: "Account temporarily locked due to multiple failed attempts."
        retry_after_seconds:
          type: integer
          example: 900

    RateLimitErrorResponse:
      type: object
      properties:
        error:
          type: string
          enum: [rate_limit_exceeded]
        error_description:
          type: string
          example: "Too many login attempts from this IP."
```

### 2.2. Reglas de Oro del Contrato API

1. **Consistencia de Errores 401:** El cuerpo de la respuesta para "Usuario no existe" y "Password incorrecto" debe ser **idéntico** a nivel de bytes.
2. **Consistencia de Tiempo:** El tiempo de respuesta para un 401 debe ser constante (~150ms), independientemente de la causa (garantizado por el Dummy Hash).
3. **Exposición de Refresh Token:** El `refresh_token` solo se incluye en la respuesta 200 inicial. No se renueva en este endpoint (ver UC-13).

---

## 3. Catálogo de Métricas OpenTelemetry

### 3.1. Métricas de Seguridad y Detección de Ataques

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagi.auth.login.total` | Counter | attempts | Total de intentos de login | `tenant_id`, `result` (success, fail, locked, mfa_required) |
| `kagi.auth.lockout.triggered` | Counter | events | Cuentas bloqueadas por umbral de fallos | `tenant_id` |
| `kagi.auth.timing_attack.protection` | Counter | executions | Veces que se ejecutó el Dummy Hash (indica intentos de enumeración) | `tenant_id` |
| `kagi.auth.credential_stuffing.score` | Gauge | score | Puntuación de riesgo de IP basada en fallos por minuto | `ip_address`, `tenant_id` |

### 3.2. Métricas de Rendimiento Criptográfico

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagi.crypto.argon2.duration` | Histogram | ms | Tiempo de ejecución de Argon2id | `type` (real, dummy), `tenant_id` |
| `kagi.auth.login.duration` | Histogram | ms | Latencia end-to-end del endpoint `/auth/login` | `result`, `tenant_id` |
| `kagi.cache.lockout.duration` | Histogram | ms | Latencia de operaciones en Redis (Lockout) | `operation` (get, incr, set), `tenant_id` |

### 3.3. Spans de Trazabilidad Distribuida (Traces)

El Trace de una autenticación exitosa debe descomponerse en:
```text
POST /v1/auth/login (Root Span) [Total: 180ms]
├── cache.lockout.check [5ms] (Redis GET)
├── db.identity.fetch [15ms] (KAGAMI Read Model)
├── crypto.argon2.verify [150ms] (CPU Bound - Real o Dummy)
├── cache.lockout.reset [5ms] (Redis DEL)
├── token.jwt.issue [3ms]
└── audit.event.publish [2ms] (Outbox Pattern)
```

### 3.4. Alertas Sugeridas (SRE & Security)

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Credential Stuffing Detectado** | `kagi.auth.login.total{result=fail} > 1000` en 5 min por IP | **Crítica** | Bloquear IP en WAF/API Gateway. Notificar al equipo de seguridad. |
| **Timing Attack Anomaly** | `kagi.crypto.argon2.duration{type=dummy}` diverge > 20ms de `type=real` | Alta | Revisar parámetros de Argon2id o carga de CPU del nodo. El Dummy Hash no está siendo efectivo. |
| **Lockout Storm** | `kagi.auth.lockout.triggered > 50` en 10 min | Alta | Posible ataque dirigido a usuarios específicos. Revisar logs de auditoría. |
| **Argon2 CPU Saturation** | `kagi.crypto.argon2.duration_p95 > 300ms` | Media | Escalar horizontalmente pods de KAGI (HPA por CPU). |

---

## 4. Patrones de Resiliencia

### 4.1. Estrategia Fail-Secure para Dependencias Críticas

Dado que UC-11 es la puerta de entrada, no podemos permitirnos degradar la seguridad por disponibilidad.

```csharp
public class ResilientLockoutManager : ILockoutManager
{
    private readonly IDistributedCache _cache;
    private readonly ILogger<ResilientLockoutManager> _logger;

    public async Task<bool> IsLockedOutAsync(string identifier, CancellationToken ct)
    {
        try
        {
            // Timeout estricto de 50ms para Redis
            using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
            cts.CancelAfter(TimeSpan.FromMilliseconds(50));
            
            var status = await _cache.GetStringAsync($"lockout:{identifier}", cts.Token);
            return status == "locked";
        }
        catch (Exception ex) when (ex is OperationCanceledException or RedisConnectionException)
        {
            // FAIL SECURE: Si no podemos verificar el lockout, denegamos el acceso
            // para proteger al usuario de un posible brute force mientras Redis está caído.
            _logger.LogCritical(ex, "Redis unavailable during lockout check. Failing secure.");
            return true; 
        }
    }
}
```

### 4.2. Circuit Breaker para KAGAMI (Identity Repo)

Si el motor de identidades (KAGAMI) está caído, no tiene sentido intentar autenticar.

```csharp
// Configuración Polly
services.AddPolicyHandler(Policy<IdentitySnapshot?>
    .Handle<HttpRequestException>()
    .Or<TimeoutException>()
    .CircuitBreakerAsync(
        exceptionsAllowedBeforeBreaking: 3,
        durationOfBreak: TimeSpan.FromMinutes(2),
        onBreak: (ex, duration) => _logger.LogWarning("KAGAMI Circuit Breaker OPEN. Auth endpoint returning 503.")
    ));
```

### 4.3. Rate Limiting en Capas

1. **Capa 1 (API Gateway):** Rate Limit por IP (ej. 20 requests/minuto). Previene DDoS y scanning de IPs.
2. **Capa 2 (Aplicación - LockoutManager):** Rate Limit por Identificador (ej. 5 fallos consecutivos). Previene Brute Force dirigido.

---

## 5. Feature Flags y Agilidad Criptográfica

Permitir cambios en la configuración de seguridad sin redeploy ni downtime.

| Feature Flag | Tipo | Descripción | Valor por Defecto |
| :--- | :--- | :--- | :--- |
| `kagi.auth.local.enabled` | Boolean | Kill switch global para autenticación local. | `true` |
| `kagi.auth.mfa.enforce_all` | Boolean | Fuerza MFA para todos los usuarios (override de políticas de tenant). | `false` |
| `kagi.auth.lockout.max_attempts` | Integer | Umbral de fallos para bloqueo. | `5` |
| `kagi.auth.lockout.duration_minutes` | Integer | Duración del bloqueo temporal. | `15` |
| `kagi.crypto.argon2.memory_cost_mb` | Integer | Costo de memoria para Argon2id (Agilidad Criptográfica). | `64` |
| `kagi.crypto.argon2.iterations` | Integer | Iteraciones de Argon2id. | `3` |

**Implementación de Agilidad Criptográfica:**
```csharp
public class PasswordHasher : IPasswordHasher
{
    private readonly IFeatureManager _featureManager;

    public PasswordHash GenerateDummyHash()
    {
        // Leer parámetros dinámicamente de Feature Flags / Configuración
        var memoryCost = _featureManager.GetFeatureValue<int>("kagi.crypto.argon2.memory_cost_mb") * 1024;
        var iterations = _featureManager.GetFeatureValue<int>("kagi.crypto.argon2.iterations");
        
        // ... ejecutar Argon2id con estos parámetros ...
    }
}
```

---

## 6. Compliance y Auditoría (Mapeo Normativo)

### 6.1. NIST SP 800-63B (Digital Identity Guidelines)
*   **Requisito:** Los autenticadores memorizados (passwords) deben ser verificados contra un salt y un hash criptográfico aprobado.
*   **Implementación UC-11:** Uso de **Argon2id** (ganador del Password Hashing Competition) con parámetros configurables.
*   **Requisito:** El sistema debe implementar un mecanismo de rate limiting o lockout.
*   **Implementación UC-11:** `LockoutManager` con Redis y TTL automático.

### 6.2. OWASP ASVS 4.0
*   **V2.4.2:** Verificar que las contraseñas se almacenen de forma que resista ataques offline (hashing + salt). -> **Cumplido (Argon2id + Salt por usuario).**
*   **V2.4.4:** Verificar que no haya diferencias en el tiempo de respuesta o mensajes de error que permitan la enumeración de usuarios. -> **Cumplido (Dummy Hash + Mensajes genéricos 401).**

### 6.3. GDPR Art. 32 (Security of Processing)
*   **Requisito:** Capacidad para garantizar la confidencialidad, integridad, disponibilidad y resiliencia de los sistemas.
*   **Implementación UC-11:** Patrones Fail-Secure, Circuit Breakers, y auditoría inmutable de todos los intentos (éxito y fallo) vía Outbox Pattern a KAGE.

---

## 7. Estrategia de Deployment y Escalado

### 7.1. Kubernetes HPA (Horizontal Pod Autoscaler)

Dado que Argon2id es intensivo en CPU, el escalado no debe basarse solo en memoria o requests por segundo, sino en la latencia de la operación criptográfica.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kagi-auth-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kagi-auth
  minReplicas: 3 # Alta disponibilidad mínima
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60 # Mantener headroom para picos de Argon2id
    - type: Pods
      pods:
        metric:
          name: kagi_crypto_argon2_duration_p95
        target:
          type: AverageValue
          averageValue: "200" # Escalar si el hashing tarda más de 200ms
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30 # Escalar rápido ante ataques
      policies:
        - type: Percent
          value: 100
          periodSeconds: 30
```

### 7.2. Consideraciones de Seguridad en el Cluster
*   **Network Policies:** El pod de KAGI solo debe tener acceso de salida a KAGAMI (puerto DB), Redis (puerto 6379) y KAGE (puerto de mensajería). No debe tener acceso directo a internet.
*   **Secrets Management:** Las claves de firma de JWT (RS256 Private Key) deben inyectarse desde un Vault (HashiCorp Vault / Azure Key Vault) y rotarse automáticamente.

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los requisitos de anti-enumeración y lockout que estos contratos y métricas monitorean. |
| `08_Final_Class_Design.md` | Las clases `AuthenticationHandler` y `PasswordHasher` implementan la lógica que genera estas métricas y respeta estos contratos. |
| `10_UC_Specific_Data_Model.md` | Define la estructura de la tabla `outbox` para la auditoría asíncrona y la tabla de `refresh_tokens`. |
| `11_UC_Specific_Security_Threats.md` | Las mitigaciones de "Timing Attacks" y "Credential Stuffing" se validan mediante las métricas `timing_attack.protection` y alertas de rate limiting. |
| `12_UC_Specific_Performance_Benchmarks.md` | Los SLAs de latencia (<300ms) se monitorean mediante la métrica `auth.login.duration`. |

---

## 9. Cierre del Paquete de Diseño Funcional (UC-11)

Con este documento, completamos los **9 archivos del Paquete de Diseño Profundo Funcional (Tier 1)** para el UC-11:

1. ✅ `01_Requirements_Specification.md` - Especificación completa con escenarios BDD.
2. ✅ `02_Use_Case_Diagram.md` - Diagrama de caso de uso UML con actores y sistemas.
3. ✅ `03_Activity_Diagram.md` - Diagrama de actividades con flujo de seguridad y Dummy Hash.
4. ✅ `04_Initial_Domain_Model.md` - Modelo de dominio conceptual (Aggregate Roots, VOs).
5. ✅ `05_State_Machine_Diagram.md` - Máquinas de estados (AuthenticationAttempt, RefreshToken).
6. ✅ `06_Robustness_Diagram.md` - Diagrama de robustez ICONIX.
7. ✅ `07_Sequence_Diagram.md` - Diagrama de secuencia con flujos alternativos.
8. ✅ `08_Final_Class_Design.md` - Diseño de clases con firmas y patrones.
9. ✅ `09_Modern_Artifacts.md` - Contratos OpenAPI, métricas, resiliencia, compliance.

### 9.1. Resumen de lo Logrado

Hemos diseñado el endpoint de autenticación local más seguro y observable del mercado IAM:
- **Seguridad Criptográfica de Elite:** Argon2id con Dummy Hash para prevenir Timing Attacks.
- **Resiliencia Fail-Secure:** Si Redis o KAGAMI fallan, el sistema deniega el acceso para proteger al usuario.
- **Observabilidad de Seguridad:** Métricas específicas para detectar enumeración de usuarios y Credential Stuffing en tiempo real.
- **Agilidad Criptográfica:** Feature Flags para ajustar parámetros de hashing sin redeploy.
- **Cumplimiento Normativo:** Mapeo directo a NIST SP 800-63B, OWASP ASVS y GDPR.

### 9.2. Próximos Pasos (Documentos Técnicos Específicos)

Según la **Opción A+** acordada, cada UC tiene **12 documentos** (9 funcionales + 3 técnicos específicos). Para el UC-11, aún faltan:

1. **`10_UC_Specific_Data_Model.md`** - Modelo de datos físico específico (tablas `outbox`, `refresh_tokens`, índices para búsqueda de email normalizado).
2. **`11_UC_Specific_Security_Threats.md`** - Análisis de amenazas específico (Timing Attacks, Credential Stuffing, Brute Force, Memory Dumping).
3. **`12_UC_Specific_Performance_Benchmarks.md`** - Benchmarks y SLAs específicos para latencia de Argon2id, throughput de logins, y capacidad de Redis.

---
*Fin del Documento 09_Modern_Artifacts para UC-11*