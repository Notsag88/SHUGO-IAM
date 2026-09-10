# UC-09: Verificación de Contactos - Especificación de Requisitos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-09 - Verificación de Contactos (Out-of-Band Verification) |
| **Documento:** | 01_Requirements_Specification.md |
| **Metodología:** | ICONIX (Deep Dive) + DDD + Event-Driven Architecture + Security by Design |
| **Estándares:** | NIST SP 800-63A (Enrollment & Identity Proofing), OWASP ASVS V2 (Authentication), GDPR Art. 7 (Consent), SOC 2 CC6.1 |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 09/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 09/09/2026 | Gastón Ruiz | **Creación inicial.** Definición completa de la especificación analítica para la verificación de contactos (Email/Phone). Establece el flujo de generación de OTPs (One-Time Passwords) y Magic Links, las políticas de rate limiting para prevención de abuso (SMS/Email bombing), la protección contra fuerza bruta de OTPs, y la integración con UC-06 (Registro) y UC-08 (Cambio de Email) como motor de confianza criptográfica. | Documento base para el paquete de diseño Tier 1 del UC-09. Es el caso de uso transversal que garantiza la **integridad de los canales de comunicación** (Out-of-Band Verification), requisito fundamental para la recuperación de credenciales, MFA y cumplimiento de normativas de identidad. |

---

## 1. Introducción y Alcance

Este documento detalla la especificación de requisitos para el Caso de Uso **UC-09**, que representa la operación de **verificación de contactos (Out-of-Band Verification)** en el motor **KAGAMI** de SHUGO.

La verificación de contactos es el pilar de confianza en sistemas IAM modernos. Sin ella, no es posible garantizar que un usuario es el dueño legítimo del email o teléfono que afirma poseer. El UC-09 es un **servicio de dominio transversal** consumido por otros casos de uso:

1. **UC-06 (Registro Self-Service):** Al registrarse, el usuario debe verificar su email para que la identidad pase a estado `ACTIVE`.
2. **UC-08 (Self-Service Profile):** Si el usuario cambia su email, el cambio queda en estado `PENDING` hasta que UC-09 verifique el nuevo contacto.
3. **UC-18 (Recuperación de Credenciales):** El flujo de "Forgot Password" depende de UC-09 para enviar Magic Links o OTPs.
4. **UC-15 (MFA):** Envío de OTPs por SMS/Email como segundo factor de autenticación.

### 1.1. Importancia Estratégica

El UC-09 es **crítico para la seguridad y cumplimiento normativo** porque:

- **Integridad de Identidad (NIST SP 800-63A):** Garantiza que el canal de recuperación o notificación está bajo el control exclusivo del usuario (prueba de posesión).
- **Prevención de Abuso (Cost Control):** Los canales de envío (SMS vía Twilio, Emails vía SendGrid) tienen costos y reputación (IPs en blocklists). El UC-09 debe implementar rate limiting estricto para prevenir "SMS/Email Bombing" a víctimas inocentes.
- **Seguridad contra Fuerza Bruta:** Los OTPs numéricos (6 dígitos) tienen una entropía baja. El sistema debe limitar intentos y aplicar delays para mitigar ataques de adivinanza.
- **No Repudio:** El uso de Magic Links (tokens criptográficos de un solo uso) proporciona una prueba de auditoría inmutable de que el usuario completó la acción.

### 1.2. Relación con Otros Casos de Uso

| UC | Relación | Descripción |
| :--- | :--- | :--- |
| **UC-06 (Registro)** | Consumidor | Dispara UC-09 al crear la identidad. Bloquea el acceso hasta verificación. |
| **UC-08 (Perfil)** | Consumidor | Dispara UC-09 al solicitar cambio de email. Mantiene el email anterior activo hasta verificación. |
| **UC-18 (Recuperación)** | Consumidor | Usa UC-09 para enviar Magic Links de reseteo de password. |
| **UC-15 (MFA)** | Consumidor | Usa UC-09 para generar y validar OTPs por SMS/Email. |
| **UC-46 (Auditoría)** | Consumidor | Registra los eventos de emisión y validación de OTPs/Magic Links. |

### 1.3. Objetivos de Rendimiento

| Métrica | Objetivo | Justificación |
| :--- | :--- | :--- |
| **Latencia de Generación (p95)** | < 100ms | La generación del token y persistencia en Redis/DB debe ser casi instantánea. |
| **Latencia de Envío (p95)** | < 2.000ms | Depende del proveedor externo (SMTP/SMS Gateway). No bloquea la respuesta al usuario si se hace asíncrono. |
| **Latencia de Validación (p95)** | < 50ms | La validación del OTP/Token debe ser rápida (lookup en Redis + SecureCompare). |
| **Disponibilidad** | 99.99% | Si UC-09 cae, los usuarios no pueden registrarse, recuperar passwords ni hacer MFA. |

---

## 2. Descripción Analítica (ICONIX Clásico)

| Campo | Detalle |
| :--- | :--- |
| **ID Y NOMBRE:** | UC-09 - Verificación de Contactos (Out-of-Band Verification) |
| **ESTADO:** | Preliminar |
| **DESCRIPCIÓN:** | El sistema recibe una solicitud para verificar un contacto (email o teléfono) asociado a una identidad. Genera un código OTP (6 dígitos) o un Magic Link (token UUID v4 criptográfico), lo persiste con un TTL estricto (10 min para OTP, 24h para Magic Link) y lo envía a través del canal correspondiente (SMTP/SMS). El usuario recibe el código/link y lo ingresa en el sistema. El sistema valida el código (verificando TTL, intentos máximos y usando SecureCompare). Si es válido, marca el contacto como verificado (`is_verified = true`) y emite un evento de dominio. |
| **ACTOR PRINCIPAL:** | Usuario (Owner del contacto) o Sistema (Background Job para limpieza). |
| **ACTOR SECUNDARIO:** | Email Gateway (SendGrid/SES), SMS Gateway (Twilio), Redis (Cache de OTPs). |
| **PRECONDICIONES:** | 1. La identidad debe existir en el sistema.<br>2. El contacto no debe estar ya verificado para *otra* identidad (exclusividad).<br>3. El usuario no debe haber excedido el rate limit de envío (ej. 1 por minuto, 5 por hora). |
| **PUNTOS DE EXTENSIÓN:** | Si el usuario no recibe el código, puede solicitar un reenvío (sujeto a rate limiting). Si el canal es SMS, se puede ofrecer fallback a Email. |
| **CONDICIÓN:** | El contacto queda marcado como `VERIFIED`, el token/OTP se invalida (single-use), y se genera un log de auditoría. |

### 2.1. Escenario Principal (Happy Path) - Verificación por OTP

1. El usuario autenticado solicita verificar su número de teléfono (o email secundario).
2. El sistema valida que el teléfono no esté en uso por otra identidad.
3. El sistema valida el rate limiting (no se ha enviado otro OTP en los últimos 60 segundos).
4. El sistema genera un OTP criptográficamente seguro (6 dígitos).
5. El sistema persiste el OTP en Redis con TTL de 10 minutos y contador de intentos = 0.
6. El sistema envía el OTP vía SMS Gateway (de forma asíncrona).
7. El sistema retorna HTTP 202 Accepted ("OTP sent").
8. El usuario recibe el SMS e ingresa el OTP en la interfaz.
9. El sistema valida el OTP: verifica que exista, que no haya expirado, que no haya excedido 5 intentos, y compara usando `SecureCompare`.
10. El sistema marca el teléfono como `VERIFIED`, elimina el OTP de Redis y emite el evento `Contact.Verified`.
11. El sistema retorna HTTP 200 OK ("Contact verified successfully").

### 2.2. Escenario Alternativo - Verificación por Magic Link (Email)

1. El usuario solicita verificar su email (ej. durante el registro UC-06 o cambio en UC-08).
2. El sistema genera un Magic Link (URL con token UUID v4 de 128 bits).
3. El sistema persiste el token hash en DB/Redis con TTL de 24 horas.
4. El sistema envía el email con el link (ej. `https://app.shugo.io/verify?token=...`).
5. El usuario hace clic en el link desde su cliente de correo.
6. El sistema recibe el GET, valida el token (SecureCompare, TTL, single-use).
7. El sistema marca el email como `VERIFIED`, invalida el token y redirige al usuario al dashboard con un mensaje de éxito.

### 2.3. Flujos de Excepción

**[4.1] Rate Limit Excedido (Prevención de Abuso)**
- 4.1.1 El sistema detecta que se ha solicitado un OTP para este contacto/IP más de 5 veces en la última hora.
- 4.1.2 Retorna HTTP 429 Too Many Requests.
- 4.1.3 **No se envía ningún mensaje** (previene SMS/Email bombing).

**[9.1] OTP Expirado**
- 9.1.1 El usuario ingresa el OTP después de los 10 minutos.
- 9.1.2 Retorna HTTP 400 Bad Request ("OTP expired. Please request a new one").

**[9.2] Máximo de Intentos Alcanzado (Anti-Brute Force)**
- 9.2.1 El usuario ingresa un OTP incorrecto 5 veces consecutivas.
- 9.2.2 El sistema invalida el OTP inmediatamente (aún no haya expirado el TTL).
- 9.2.3 Retorna HTTP 429 Too Many Requests ("Too many invalid attempts. Please request a new OTP").

**[9.3] Token de Magic Link Ya Usado (Replay Attack)**
- 9.3.1 El usuario (o un atacante) intenta usar un Magic Link que ya fue consumido.
- 9.3.4 Retorna HTTP 400 Bad Request ("Link already used or invalid").

### 2.4. Postcondiciones

**De Éxito:**
1. El contacto en la tabla `identity_contacts` tiene `is_verified = true` y `verified_at = NOW()`.
2. El OTP/Token ha sido eliminado o invalidado.
3. Se ha emitido el evento `Contact.Verified`.

**De Falla:**
1. El contacto permanece `UNVERIFIED`.
2. Se ha registrado el intento fallido en los logs de seguridad (para detectar ataques de fuerza bruta).

---

## 3. Reglas de Negocio e Invariantes de Dominio (DDD)

| ID | Invariante / Regla | Descripción Técnica |
| :--- | :--- | :--- |
| **INV-55** | Exclusividad de Contacto Verificado | Un contacto (email/phone) solo puede estar verificado para **una** identidad a la vez. Si se verifica para una nueva identidad, la anterior debe desvincularse o marcarse como no verificada. |
| **INV-56** | TTL Estricto | OTPs: 10 minutos. Magic Links: 24 horas. No extendibles. |
| **INV-57** | Single-Use Tokens | Magic Links y OTPs se invalidan inmediatamente tras una validación exitosa (o tras 5 intentos fallidos). |
| **INV-58** | Rate Limiting de Emisión | Máximo 1 envío por minuto por contacto. Máximo 5 envíos por hora por contacto/IP. |
| **INV-59** | Anti-Brute Force | Máximo 5 intentos de validación por OTP. Tras el 5º fallo, el OTP se destruye. |
| **RN-06** | Generación Criptográfica | Los OTPs y Tokens deben generarse usando `RandomNumberGenerator` (C#) o `crypto.randomBytes` (Node), nunca `Math.random()`. |
| **RN-07** | SecureCompare | La validación de OTPs y Tokens debe usar comparación en tiempo constante para prevenir Timing Attacks. |
| **RN-08** | Mensajes Genéricos (Anti-Enumeration) | Si un atacante intenta verificar un email que no existe en el sistema, el sistema retorna el mismo mensaje de éxito ("If the email exists, a verification link was sent") para no revelar la base de datos de usuarios. |
| **RN-09** | Fallback de Canales | Si el envío por SMS falla (ej. número inválido, gateway caído), el sistema puede ofrecer fallback a Email si ambos están registrados. |

---

## 4. Criterios de Aceptación (BDD / Gherkin)

### Escenario 4.1: Emisión y validación exitosa de OTP
```gherkin
DADO que el usuario "user-123" solicita verificar el teléfono "+5491155556666"
Y el teléfono no está verificado para otra identidad
Y el usuario no ha excedido el rate limit de envío
CUANDO el sistema procesa la solicitud
ENTONCES genera un OTP de 6 dígitos
Y lo persiste en Redis con TTL de 10 minutos
Y envía el OTP vía SMS Gateway
Y retorna HTTP 202 Accepted
```

### Escenario 4.2: Validación de OTP exitosa
```gherkin
DADO que el usuario ingresa el OTP correcto "123456"
Y el OTP no ha expirado
Y el contador de intentos es < 5
CUANDO el sistema valida el OTP
ENTONCES marca el teléfono como VERIFIED
Y elimina el OTP de Redis
Y emite el evento "Contact.Verified"
Y retorna HTTP 200 OK
```

### Escenario 4.3: Prevención de Fuerza Bruta (Max Attempts)
```gherkin
DADO que el usuario ha ingresado un OTP incorrecto 5 veces
CUANDO intenta ingresar el OTP por 6ª vez
ENTONCES el sistema invalida el OTP inmediatamente
Y retorna HTTP 429 Too Many Requests
Y registra alerta de seguridad "Brute force attempt on OTP"
```

### Escenario 4.4: Prevención de Abuso (Rate Limiting)
```gherkin
DADO que se ha enviado un OTP al teléfono "+5491155556666" hace 30 segundos
CUANDO el usuario solicita un nuevo OTP
ENTONCES el sistema retorna HTTP 429 Too Many Requests
Y NO envía ningún SMS
Y el mensaje es "Please wait 60 seconds before requesting a new OTP"
```

### Escenario 4.5: Prevención de Enumeración de Emails
```gherkin
DADO que un atacante solicita verificar el email "inexistente@khcs.com"
Y el email NO existe en ninguna identidad del sistema
CUANDO el sistema procesa la solicitud
ENTONCES retorna HTTP 202 Accepted (mismo código que si existiera)
Y el mensaje es "If the email is registered, a verification link has been sent"
Y NO se envía ningún email
```

### Escenario 4.6: Magic Link de un solo uso (Replay Attack)
```gherkin
DADO que el usuario ya usó un Magic Link para verificar su email
CUANDO el usuario (o un atacante) intenta acceder a la misma URL nuevamente
ENTONCES el sistema retorna HTTP 400 Bad Request
Y el mensaje es "Link already used or expired"
```

---

## 5. Eventos de Dominio Generados

### Evento 1: `Contact.VerificationInitiated`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Contact.VerificationInitiated",
  "timestamp": "2026-09-05T10:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-123",
    "contactType": "EMAIL",
    "contactHash": "sha256_hash_of_email", 
    "channel": "SMTP",
    "tokenType": "MAGIC_LINK"
  },
  "metadata": {
    "traceId": "opentelemetry-trace-id",
    "ipAddress": "192.168.1.100"
  }
}
```
*Nota: Se usa `contactHash` en lugar del email en texto plano para proteger PII en los buses de eventos.*

### Evento 2: `Contact.Verified`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Contact.Verified",
  "timestamp": "2026-09-05T10:05:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-123",
    "contactType": "PHONE",
    "contactHash": "sha256_hash_of_phone",
    "verifiedAt": "2026-09-05T10:05:00Z"
  }
}
```
**Consumidores:**
1. **KAGE (Audit):** Registra la prueba de posesión del contacto.
2. **UC-06 (Registro):** Si todos los contactos requeridos están verificados, transiciona la identidad a `ACTIVE`.
3. **TORII (Webhooks):** Notifica a sistemas downstream (ej. CRM) que el email del usuario es confiable.

---

## 6. Consideraciones de Arquitectura y Seguridad

### 6.1. Generación y Almacenamiento de OTPs

Los OTPs no se almacenan en texto plano. Se almacenan en Redis (para TTL automático y alta velocidad) usando un hash o directamente como clave.

```csharp
public class OtpService : IOtpService
{
    public string GenerateOtp()
    {
        // Generación criptográficamente segura
        using var rng = RandomNumberGenerator.Create();
        var bytes = new byte[4];
        rng.GetBytes(bytes);
        var otp = Math.Abs(BitConverter.ToInt32(bytes)) % 1000000;
        return otp.ToString("D6"); // 6 dígitos con ceros a la izquierda
    }

    public async Task StoreOtpAsync(string contactHash, string otp, TimeSpan ttl, CancellationToken ct)
    {
        var key = $"otp:{contactHash}";
        var value = JsonSerializer.Serialize(new OtpData { 
            Hash = SHA256Hash(otp), // Almacenamos el hash, no el texto plano
            Attempts = 0,
            CreatedAt = DateTime.UtcNow 
        });
        
        await _redis.SetAsync(key, value, ttl, ct);
    }
}
```

### 6.2. Validación con SecureCompare y Anti-Brute Force

```csharp
public async Task<bool> ValidateOtpAsync(string contactHash, string providedOtp, CancellationToken ct)
{
    var key = $"otp:{contactHash}";
    var otpDataJson = await _redis.GetAsync(key, ct);
    
    if (otpDataJson == null)
        throw new DomainException("OTP expired or not found.");

    var otpData = JsonSerializer.Deserialize<OtpData>(otpDataJson);
    
    // Anti-Brute Force: Verificar intentos
    if (otpData.Attempts >= 5)
    {
        await _redis.DeleteAsync(key, ct); // Invalidar OTP
        throw new RateLimitExceededException("Too many invalid attempts.");
    }

    // Incrementar intentos
    otpData.Attempts++;
    await _redis.SetAsync(key, JsonSerializer.Serialize(otpData), TimeSpan.FromMinutes(10), ct);

    // SecureCompare para prevenir Timing Attacks
    var providedHash = SHA256Hash(providedOtp);
    if (!CryptographicOperations.FixedTimeEquals(
        Encoding.UTF8.GetBytes(otpData.Hash), 
        Encoding.UTF8.GetBytes(providedHash)))
    {
        return false; // El contador de intentos ya se incrementó
    }

    // Éxito: Invalidar OTP (Single-Use)
    await _redis.DeleteAsync(key, ct);
    return true;
}
```

### 6.3. Prevención de Enumeración (Anti-Enumeration)

En flujos donde el usuario **no está autenticado** (ej. "Forgot Password" o verificación inicial pública), el sistema **nunca** debe revelar si el contacto existe.

```csharp
public async Task<VerificationResult> InitiatePublicVerificationAsync(string email, CancellationToken ct)
{
    // 1. Buscar identidad (sin revelar si existe)
    var identity = await _identityRepo.GetByEmailAsync(email, ct);
    
    // 2. Si NO existe, retornar éxito falso (para no enumerar)
    if (identity == null)
    {
        _logger.LogInformation("Verification requested for non-existent email. Silently ignoring.");
        return new VerificationResult { Message = "If the email is registered, a link was sent." };
    }

    // 3. Si existe, generar y enviar token
    await GenerateAndSendMagicLinkAsync(identity, email, ct);
    return new VerificationResult { Message = "If the email is registered, a link was sent." };
}
```

---

## 7. Contratos de API (OpenAPI 3.0 Snippets)

### 7.1. Endpoint: POST /contacts/verify/initiate (Autenticado)

**Request:**
```http
POST /v1/contacts/verify/initiate HTTP/1.1
Authorization: Bearer <JWT>
Content-Type: application/json

{
  "contact_type": "PHONE",
  "contact_value": "+5491155556666",
  "channel": "SMS"
}
```

**Response (202 Accepted):**
```json
{
  "message": "Verification code sent.",
  "expires_in_seconds": 600
}
```

### 7.2. Endpoint: POST /contacts/verify/complete

**Request:**
```http
POST /v1/contacts/verify/complete HTTP/1.1
Authorization: Bearer <JWT>
Content-Type: application/json

{
  "contact_type": "PHONE",
  "contact_value": "+5491155556666",
  "otp": "123456"
}
```

**Response (200 OK):**
```json
{
  "contact_type": "PHONE",
  "contact_value": "+5491155556666",
  "is_verified": true,
  "verified_at": "2026-09-05T10:05:00Z"
}
```

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `02_Use_Case_Diagram.md` | Mostrará el límite del sistema, los actores (Usuario, Gateways), y las relaciones con UC-06, UC-08, UC-15, UC-18. |
| `03_Activity_Diagram.md` | Visualizará el flujo de generación, envío, validación, y los flujos de excepción (Rate Limit, Brute Force). |
| `04_Initial_Domain_Model.md` | Definirá el `ContactVerificationService`, `OtpGenerator`, `MagicLinkToken`, y las entidades `IdentityContact`. |
| `09_Modern_Artifacts.md` | Definirá los contratos OpenAPI completos, métricas de OpenTelemetry (OTP generation latency, validation failures), y Circuit Breakers para SMS/Email Gateways. |
| `11_UC_Specific_Security_Threats.md` | Analizará amenazas como OTP Brute-Forcing, SMS Bombing, Timing Attacks, y Token Replay. |

---

## 9. Glosario de Términos Específicos de UC-09

| Término | Definición |
| :--- | :--- |
| **Out-of-Band (OOB) Verification** | Proceso de verificar la posesión de un contacto enviando un desafío a través de un canal alternativo (ej. enviar un OTP al teléfono para verificar que el usuario posee el teléfono). |
| **OTP (One-Time Password)** | Código numérico (generalmente 6 dígitos) de un solo uso y corta duración (TTL). |
| **Magic Link** | URL criptográfica que contiene un token de un solo uso. Al hacer clic en ella, el sistema verifica al usuario sin necesidad de que copie/pegue un código. |
| **SMS/Email Bombing** | Ataque malicioso donde un atacante solicita miles de OTPs a la víctima para saturar su bandeja de entrada o generar costos masivos al tenant (SMS). Mitigado por Rate Limiting. |
| **SecureCompare (Fixed-Time Comparison)** | Algoritmo de comparación de strings que tarda el mismo tiempo sin importar en qué carácter difieran, previniendo Timing Attacks. |
| **Anti-Enumeration** | Técnica de seguridad donde el sistema retorna respuestas genéricas para evitar que un atacante descubra qué emails/teléfonos están registrados en la plataforma. |
| **TTL (Time-To-Live)** | Tiempo de vida máximo de un OTP o Magic Link antes de expirar automáticamente. |

---
*Fin del Documento 01_Requirements_Specification para UC-09*