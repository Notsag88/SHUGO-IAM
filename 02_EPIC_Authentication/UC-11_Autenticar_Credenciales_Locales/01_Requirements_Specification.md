# UC-11: Autenticar Credenciales Locales - Especificación de Requisitos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-11 - Autenticar Credenciales Locales (Local Authentication) |
| **Documento:** | 01_Requirements_Specification.md |
| **Metodología:** | ICONIX (Deep Dive) + DDD + BDD + Zero Trust Architecture + Security by Design |
| **Estándares:** | NIST SP 800-63B (AAL2/AAL3), OWASP ASVS V2 (Authentication), GDPR Art. 32, SOC 2 CC6.1 |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 15/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 15/09/2026 | Gastón Ruiz | **Creación inicial.** Definición completa de la especificación analítica para la autenticación local basada en credenciales (Email/Username + Password). Establece el flujo de validación criptográfica (Argon2id), prevención de ataques de fuerza bruta y enumeración, gestión de bloqueos de cuenta (Account Lockout), y la emisión inicial de tokens JWT. Incluye la integración condicional con MFA (UC-15) y la evaluación de riesgo (UC-17). | Documento base para el paquete de diseño Tier 1 del UC-11. Es la "puerta de entrada" crítica del motor KAGI, definiendo cómo se valida la identidad del usuario de forma segura, performante y auditable. |

---

## 1. Introducción y Alcance

Este documento detalla la especificación de requisitos para el Caso de Uso **UC-11**, que representa la operación de **Autenticación de Credenciales Locales** en el motor **KAGI** de SHUGO.

A diferencia de la autenticación federada (SSO) o passwordless, este UC se centra en el flujo tradicional pero hardened de "algo que el usuario sabe" (password). Dado que es el vector de ataque más común, el diseño prioriza la **seguridad criptográfica** y la **resiliencia ante ataques automatizados** sin sacrificar la experiencia de usuario legítima.

### 1.1. Importancia Estratégica

El UC-11 es **el punto de control más crítico** de la plataforma porque:

- **Validación de Identidad:** Es el primer paso para establecer confianza (Trust) en el sistema.
- **Protección de Datos:** Un fallo aquí compromete la confidencialidad de todos los datos del usuario.
- **Rendimiento vs. Seguridad:** El hashing de passwords (Argon2id) es intencionalmente lento. El sistema debe balancear la seguridad (resistencia a GPU cracking) con la latencia aceptable para el usuario (<300ms).
- **Prevención de Abuso:** Debe resistir ataques de Brute Force, Credential Stuffing y Timing Attacks.

### 1.2. Relación con Otros Casos de Uso

| UC | Relación | Descripción |
| :--- | :--- | :--- |
| **UC-12 (Cerrar Sesión)** | Complementario | UC-11 crea la sesión; UC-12 la destruye. |
| **UC-13 (Refresh Token)** | Consumidor | UC-11 emite el Refresh Token inicial que UC-13 renovará. |
| **UC-15 (MFA)** | Integración | Si el usuario tiene MFA habilitado, UC-11 valida el password y retorna un estado "MFA_PENDING", delegando la verificación del segundo factor a UC-15. |
| **UC-17 (Risk-Based Auth)** | Consumidor | UC-11 envía el contexto (IP, User-Agent, Geo) a UC-17 para evaluar el riesgo antes de emitir el token final. |
| **UC-42 (Account Lockout)** | Integración | UC-11 consulta y actualiza el estado de bloqueo de la cuenta tras fallos consecutivos. |
| **UC-43 (Breach Detection)** | Integración | UC-11 puede verificar si la contraseña ha sido comprometida en brechas conocidas (Have I Been Pwned) durante el login o cambio. |

### 1.3. Objetivos de Rendimiento

| Métrica | Objetivo | Justificación |
| :--- | :--- | :--- |
| **Latencia de Autenticación (p95)** | < 300ms | Incluye hashing Argon2id (~100-150ms) + DB lookup + Token generation. |
| **Throughput (Pico)** | 500 logins/segundo | Suficiente para picos de "mañana laboral" en tenants enterprise. |
| **Tasa de Falsos Positivos (Lockout)** | < 0.1% | Usuarios legítimos bloqueados por error deben ser mínimos. |
| **Disponibilidad** | 99.99% | Si el login cae, la plataforma es inaccesible. |

---

## 2. Descripción Analítica (ICONIX Clásico)

| Campo | Detalle |
| :--- | :--- |
| **ID Y NOMBRE:** | UC-11 - Autenticar Credenciales Locales |
| **ESTADO:** | Preliminar |
| **DESCRIPCIÓN:** | El usuario (Resource Owner) envía sus credenciales (email/username y password) al endpoint de autenticación. El sistema valida el formato, verifica que la cuenta no esté bloqueada ni desactivada, recupera el hash de la contraseña almacenada, y realiza una comparación segura en tiempo constante. Si es exitoso, evalúa si se requiere MFA. Si no se requiere MFA (o ya fue verificado), emite un par de tokens (Access Token + Refresh Token) y registra el evento de auditoría. Si falla, incrementa el contador de fallos y puede bloquear la cuenta temporalmente. |
| **ACTOR PRINCIPAL:** | Usuario (Resource Owner), Aplicación Cliente (SPA/Mobile). |
| **ACTOR SECUNDARIO:** | API Gateway (Rate Limiting), KAGAMI (Identity Repo), KAGE (Audit), Redis (Lockout/Cache). |
| **PRECONDICIONES:** | 1. La identidad existe y está en estado `ACTIVE`.<br>2. El tenant tiene habilitada la autenticación local.<br>3. El usuario no está en una lista negra global (IP reputation). |
| **PUNTOS DE EXTENSIÓN:** | Si el usuario tiene MFA obligatorio, el flujo se pausa y retorna un challenge. Si el riesgo es alto (UC-17), se puede solicitar MFA step-up aunque no sea obligatorio. |
| **CONDICIÓN:** | El usuario recibe tokens JWT válidos o un error genérico "Invalid credentials" para prevenir enumeración. |

### 2.1. Escenario Principal (Happy Path) - Login Exitoso sin MFA

1. El usuario envía `POST /auth/login` con `email` y `password`.
2. El API Gateway valida Rate Limiting (prevención de DDoS/Brute Force).
3. El Authentication Handler normaliza el email (lowercase, trim).
4. El Handler consulta la identidad en KAGAMI.
5. **Guarda de Seguridad:** Si la identidad no existe, el sistema simula un retraso y retorna "Invalid credentials" (Anti-Enumeration).
6. **Guarda de Estado:** Se verifica que la cuenta no esté `LOCKED` o `DEACTIVATED`.
7. El Handler recupera el `PasswordHash` y los parámetros de Argon2id (salt, memory, iterations).
8. Se ejecuta `Argon2id.Verify(providedPassword, storedHash)`.
9. **Si falla:** Se incrementa el contador de fallos en Redis. Si supera el umbral (ej. 5), se bloquea la cuenta (UC-42). Se retorna "Invalid credentials".
10. **Si es exitoso:**
    a. Se resetea el contador de fallos.
    b. Se evalúa si requiere MFA (UC-15).
    c. Si NO requiere MFA: Se generan Access Token (JWT) y Refresh Token.
    d. Se publica evento `Authentication.Succeeded`.
    e. Se retorna HTTP 200 con los tokens.

### 2.2. Escenario Alternativo - Login con MFA Obligatorio

1-9. (Idénticos al escenario principal hasta la verificación del password).
10. El sistema detecta que el usuario tiene `MFA_Enabled = true`.
11. En lugar de emitir tokens de sesión, se genera un `MFA_Session_Token` (de corta duración, ej. 5 min).
12. Se retorna HTTP 200 con `{ "mfa_required": true, "mfa_token": "...", "methods": ["totp", "webauthn"] }`.
13. El cliente debe invocar UC-15 para completar la autenticación.

### 2.3. Flujos de Excepción

**[4.1] Cuenta Bloqueada (Account Lockout)**
- 4.1.1 El usuario intenta login pero la cuenta está en estado `LOCKED`.
- 4.1.2 El sistema retorna HTTP 423 Locked (o 401 con mensaje específico si la política lo permite, aunque se prefiere ofuscar).
- 4.1.3 Se registra intento de acceso a cuenta bloqueada en auditoría.

**[4.2] Credential Stuffing Detectado**
- 4.2.1 El sistema detecta que el password ha sido visto en brechas públicas (integración UC-43 o lista local).
- 4.2.2 El login es exitoso, pero se fuerza al usuario a cambiar el password en el siguiente request (flag `password_expired`).
- 4.2.3 Se notifica al usuario por email.

**[4.3] Timing Attack Prevention**
- 4.3.1 El usuario ingresa un email que NO existe.
- 4.3.2 Para evitar que un atacante mida el tiempo de respuesta y sepa que el email no existe, el sistema ejecuta un "dummy hash" (falso cálculo de Argon2id) que toma el mismo tiempo que un hash real.
- 4.3.3 Retorna "Invalid credentials".

### 2.4. Postcondiciones

**De Éxito:**
1. Sesión iniciada (tokens emitidos).
2. Contador de fallos reseteado a 0.
3. Evento `Authentication.Succeeded` publicado con metadata (IP, User-Agent, Geo).
4. LastLoginAt actualizado en la identidad.

**De Falla:**
1. Contador de fallos incrementado.
2. Si umbral alcanzado -> Cuenta bloqueada.
3. Evento `Authentication.Failed` publicado.
4. No se revela si el error fue "usuario no encontrado" o "password incorrecto".

---

## 3. Reglas de Negocio e Invariantes de Dominio (DDD)

| ID | Invariante / Regla | Descripción Técnica |
| :--- | :--- | :--- |
| **INV-71** | Hashing Criptográfico (Argon2id) | Las contraseñas NUNCA se almacenan en texto plano. Se usa Argon2id con parámetros configurables (Memory: 64MB, Iterations: 3, Parallelism: 4). |
| **INV-72** | Comparación en Tiempo Constante | La validación del hash debe usar funciones `ConstantTimeEquals` para prevenir Timing Attacks. |
| **INV-73** | Anti-Enumeración de Usuarios | El mensaje de error debe ser genérico ("Invalid credentials") tanto para usuario no encontrado como para password incorrecto. El tiempo de respuesta debe ser constante. |
| **INV-74** | Account Lockout Policy | Tras N intentos fallidos consecutivos (ej. 5), la cuenta se bloquea temporalmente (ej. 15 min) o hasta reseteo admin. |
| **INV-75** | Token Structure (JWT) | El Access Token debe incluir claims estándar: `sub` (IdentityId), `tenant_id`, `auth_level` (AAL1/AAL2), `iat`, `exp`. |
| **INV-76** | Refresh Token Security | Los Refresh Tokens deben ser opacos (no JWT), almacenados como hashes SHA256 en DB, con rotación obligatoria (UC-13). |
| **RN-16** | Normalización de Identificadores | El email se normaliza (lowercase, trim) antes de la búsqueda para evitar errores de "User not found" por mayúsculas. |
| **RN-17** | Dummy Hash Execution | Si el usuario no existe, se debe ejecutar un hash dummy con los mismos parámetros de Argon2id para mantener el tiempo de respuesta constante. |
| **RN-18** | Audit Context | Todo intento (éxito o fallo) debe registrar: IP Address, User-Agent, Device Fingerprint (si disponible), y Geo-location. |
| **RN-19** | MFA Enforcement | Si el tenant o el usuario tiene MFA obligatorio, el flujo de UC-11 se detiene tras validar el password y retorna el challenge de MFA. |

---

## 4. Criterios de Aceptación (BDD / Gherkin)

### Escenario 4.1: Autenticación exitosa con credenciales válidas
```gherkin
DADO que el usuario "user@khcs.com" existe y está activo
Y la contraseña almacenada es "SecureP@ss123"
CUANDO el usuario envía POST /auth/login con email "user@khcs.com" y password "SecureP@ss123"
ENTONCES el sistema retorna HTTP 200 OK
Y la respuesta contiene un "access_token" válido (JWT)
Y la respuesta contiene un "refresh_token" opaco
Y el evento "Authentication.Succeeded" es publicado
Y el contador de fallos de la cuenta se resetea a 0
```

### Escenario 4.2: Autenticación fallida (Password incorrecto)
```gherkin
DADO que el usuario "user@khcs.com" existe
Y la contraseña correcta es "SecureP@ss123"
CUANDO el usuario envía POST /auth/login con password "WrongPassword"
ENTONCES el sistema retorna HTTP 401 Unauthorized
Y el mensaje de error es "Invalid credentials" (genérico)
Y el contador de fallos de la cuenta se incrementa en 1
Y el evento "Authentication.Failed" es publicado
```

### Escenario 4.3: Prevención de enumeración (Usuario no existe)
```gherkin
DADO que el usuario "nonexistent@khcs.com" NO existe en el sistema
CUANDO el usuario envía POST /auth/login con ese email
ENTONCES el sistema retorna HTTP 401 Unauthorized
Y el mensaje de error es "Invalid credentials" (igual que password incorrecto)
Y el tiempo de respuesta es similar al de un login con usuario existente (Dummy Hash)
Y NO se revela que el usuario no existe
```

### Escenario 4.4: Account Lockout tras múltiples fallos
```gherkin
DADO que la política de lockout es 5 intentos fallidos
Y el usuario "user@khcs.com" ha fallado 4 veces consecutivas
CUANDO el usuario envía el 5º intento fallido
ENTONCES el sistema retorna HTTP 401 Unauthorized
Y la cuenta pasa a estado "LOCKED"
Y el siguiente intento (aunque sea correcto) retorna HTTP 423 Locked
```

### Escenario 4.5: Login con MFA obligatorio
```gherkin
DADO que el usuario "user@khcs.com" tiene MFA habilitado (TOTP)
Y el password es correcto
CUANDO el usuario envía POST /auth/login
ENTONCES el sistema retorna HTTP 200 OK
Y la respuesta NO contiene access_token
Y la respuesta contiene "mfa_required": true
Y la respuesta contiene un "mfa_token" temporal para completar el flujo en UC-15
```

### Escenario 4.6: Credential Stuffing / Breached Password
```gherkin
DADO que el password del usuario ha sido comprometido en una brecha pública
Y el usuario se loguea exitosamente
CUANDO el sistema valida las credenciales
ENTONCES el login es exitoso (se emiten tokens)
Y el token incluye un claim o flag "password_compromised": true
Y el usuario es redirigido a forzar el cambio de password en el próximo request
```

---

## 5. Eventos de Dominio Generados

### Evento 1: `Authentication.Succeeded`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Authentication.Succeeded",
  "timestamp": "2026-09-05T10:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "authMethod": "LOCAL_PASSWORD",
    "authLevel": "AAL2",
    "mfaUsed": false,
    "ipAddress": "192.168.1.100",
    "userAgent": "Mozilla/5.0...",
    "geoLocation": "AR"
  }
}
```
**Consumidores:**
1. **KAGE (Audit):** Registro forense.
2. **UC-17 (Risk-Based Auth):** Actualiza el perfil de comportamiento del usuario.
3. **TORII (Webhooks):** Notifica a sistemas externos (ej. "User logged in").

### Evento 2: `Authentication.Failed`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Authentication.Failed",
  "timestamp": "2026-09-05T10:05:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identifier": "user@khcs.com",
    "failureReason": "INVALID_PASSWORD",
    "ipAddress": "192.168.1.100",
    "attemptCount": 3
  }
}
```
**Consumidores:**
1. **KAGE (Audit):** Registro de intentos fallidos.
2. **UC-42 (Lockout):** Evalúa si se debe bloquear la cuenta.
3. **SIEM/Security Monitor:** Alerta de posible Brute Force si hay muchos fallos de distintas IPs.

---

## 6. Consideraciones de Arquitectura y Seguridad

### 6.1. Argon2id Parameter Tuning
Argon2id es el algoritmo recomendado por OWASP y NIST. Es "memory-hard", lo que lo hace resistente a ataques con GPUs y ASICs.
- **Configuración recomendada (SHUGO):**
  - `Type`: Argon2id
  - `Version`: 1.3
  - `MemoryCost`: 64 MB (65536 KB)
  - `TimeCost`: 3 iterations
  - `Parallelism`: 4 threads
- **Impacto:** Esto genera un hash en ~100-150ms en hardware moderno. Es aceptable para login, pero no para operaciones frecuentes.

### 6.2. Prevención de Timing Attacks (Dummy Hash)
El mayor riesgo de enumeración no es el mensaje de error, sino el **tiempo de respuesta**.
- Si el usuario NO existe: El sistema retorna en 5ms.
- Si el usuario EXISTE pero password mal: El sistema retorna en 150ms (tiempo de Argon2id).
- **Solución:** Si el usuario no existe, el sistema **debe** ejecutar un hash Argon2id "dummy" (con un salt falso) antes de retornar el error. Esto iguala el tiempo a ~150ms.

### 6.3. Estructura del JWT (Access Token)
El token emitido debe ser minimalista pero contener la información necesaria para la autorización (KIZUNA) sin consultar DB en cada request.
```json
{
  "sub": "user-uuid-123",
  "tenant_id": "khcs-001",
  "auth_level": "AAL2", 
  "mfa_verified": true,
  "iat": 1693900800,
  "exp": 1693901700, 
  "iss": "https://auth.shugo.io"
}
```

### 6.4. Refresh Token Storage
Los Refresh Tokens NO son JWTs. Son strings opacos de alta entropía (ej. 64 bytes hex).
- Se almacenan en DB como **Hash SHA256** (nunca el token completo).
- Tienen una vida larga (ej. 30 días) pero con **Rotación Obligatoria** (UC-13).

---

## 7. Contratos de API (OpenAPI 3.0 Snippets)

### 7.1. Endpoint: POST /auth/login

**Request:**
```http
POST /v1/auth/login HTTP/1.1
Host: auth.shugo.io
Content-Type: application/json

{
  "username": "user@khcs.com",
  "password": "SecureP@ss123",
  "captcha_token": "optional-if-risk-high"
}
```

**Response (200 OK - Success):**
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 900,
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4...",
  "scope": "openid profile"
}
```

**Response (200 OK - MFA Required):**
```json
{
  "mfa_required": true,
  "mfa_token": "mfa_session_uuid...",
  "allowed_methods": ["totp", "webauthn"]
}
```

**Response (401 Unauthorized - Generic Error):**
```json
{
  "error": "invalid_grant",
  "error_description": "Invalid credentials"
}
```

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `02_Use_Case_Diagram.md` | Mostrará la interacción entre el Usuario, API Gateway, KAGI y KAGAMI. |
| `04_Initial_Domain_Model.md` | Definirá el Aggregate `AuthenticationSession` y el Service `PasswordHasher`. |
| `05_State_Machine_Diagram.md` | Mostrará los estados de la sesión: `PENDING` -> `AUTHENTICATED` -> `MFA_CHALLENGE`. |
| `10_UC_Specific_Data_Model.md` | Definirá la tabla de `RefreshTokens` y la estructura de hashes Argon2id. |
| `11_UC_Specific_Security_Threats.md` | Analizará amenazas como Brute Force, Credential Stuffing, y Timing Attacks. |

---

## 9. Glosario de Términos Específicos de UC-11

| Término | Definición |
| :--- | :--- |
| **Argon2id** | Algoritmo de hashing de passwords ganador del Password Hashing Competition. Resistente a GPU y side-channel attacks. |
| **Timing Attack** | Ataque que analiza el tiempo de respuesta del servidor para inferir información (ej. si un usuario existe). |
| **Credential Stuffing** | Ataque que usa listas de credenciales filtradas de otros sitios para probar login automatizado. |
| **Account Lockout** | Mecanismo de seguridad que bloquea una cuenta tras N intentos fallidos para prevenir Brute Force. |
| **Dummy Hash** | Cálculo de hash falso ejecutado cuando el usuario no existe, para mantener el tiempo de respuesta constante. |
| **AAL (Authentication Assurance Level)** | Nivel de seguridad de la autenticación (NIST). AAL1: Password, AAL2: Password + MFA, AAL3: Hardware Key. |
| **Refresh Token Rotation** | Patrón de seguridad donde cada uso de un Refresh Token genera uno nuevo e invalida el anterior. |
| **Resource Owner** | El usuario humano dueño de la cuenta (OAuth2 terminology). |

---
*Fin del Documento 01_Requirements_Specification para UC-11*