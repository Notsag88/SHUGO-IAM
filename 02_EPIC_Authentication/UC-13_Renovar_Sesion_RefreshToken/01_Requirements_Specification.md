# UC-13: Renovar Sesión (Refresh Token) - Especificación de Requisitos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-13 - Renovar Sesión (Refresh Token Rotation & Reuse Detection) |
| **Documento:** | 01_Requirements_Specification.md |
| **Metodología:** | ICONIX (Deep Dive) + DDD + BDD + Security by Design (Token Family Chain Validation) |
| **Estándares:** | NIST SP 800-63B (Refresh Token Handling), OWASP ASVS 4.0 (V3 Session Management), RFC 6749 §1.5 (Refresh Token), RFC 6819 §5.2.2.3 (Reuse Detection), GDPR Art. 32, SOC 2 CC6.1 |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 20/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 20/09/2026 | Gastón Ruiz | **Creación inicial.** Definición completa de la especificación analítica para la renovación de sesiones mediante Refresh Token Rotation con detección de reuso (Reuse Detection) y revocación de familia (Token Family Chain Validation). Establece la ventana de gracia de concurrencia (Grace Window) para evitar falsos positivos multi-tab, la vida absoluta de sesión (Absolute Lifetime), el binding de contexto (dispositivo/IP/User-Agent), la frescura de MFA (step-up), y el claim `sid` para identificación de sesión a nivel de familia. | Documento base para el paquete de diseño Tier 1 del UC-13. Es el caso de uso que convierte las sesiones de SHUGO en sesiones *auto-curativas*: cada renovación rota el token, y cualquier reuso anómalo desencadena contención automática (revocación de familia) con notificación al usuario y alerta al SOC. |

---

## 1. Introducción y Alcance

Este documento detalla la especificación de requisitos para el Caso de Uso **UC-13**, que representa la operación de **Renovación de Sesión (Refresh Token Rotation)** en el motor **KAGI** de SHUGO.

El UC-13 es el mecanismo que mantiene vivas las sesiones de larga duración sin comprometer la seguridad: el cliente entrega un Refresh Token vigente y recibe un **par nuevo** (Access Token JWT + Refresh Token sucesor), mientras el token entregado queda marcado como `ROTATED` y nunca podrá usarse de nuevo como credencial primaria. Esta rotación convierte el robo de un refresh token en un evento **detectable**: si un token ya rotado se presenta nuevamente fuera de la ventana de gracia, el sistema asume compromiso confirmado y revoca la **familia completa** de tokens.

### 1.1. Importancia Estratégica

El UC-13 es **el punto de equilibrio entre usabilidad y seguridad** de toda la plataforma:

- **Sesiones largas sin passwords largos:** Permite sesiones de hasta 30 días (vida absoluta) con Access Tokens de 15 minutos, minimizando la ventana de replay sin fatigar al usuario con re-logins.
- **Detección de robo como característica nativa:** La rotación + detección de reuso (RFC 6819 §5.2.2.3) transforma un ataque silencioso (token robado usado en paralelo) en una alarma accionable con contención automática.
- **Resiliencia de concurrencia real:** Aplicaciones modernas (SPA multi-tab, mobile + desktop) renuevan en paralelo. Sin una **Grace Window**, el sistema revocaría familias legítimas por falsos positivos, generando soporte masivo y bloqueo de usuarios legítimos.
- **Contexto como señal de seguridad:** Cada renovación es una oportunidad de re-evaluar riesgo (UC-17): cambios abruptos de IP, User-Agent o geolocalización pueden forzar step-up (MFA) o revocación.
- **Cumplimiento:** NIST SP 800-63B exige que los refresh tokens sean almacenados de forma segura y vinculados al contexto; OWASP ASVS V3 exige gestión de ciclo de vida de sesión con timeouts y binding.

### 1.2. Relación con Otros Casos de Uso

| UC | Relación | Descripción |
| :--- | :--- | :--- |
| **UC-11 (Login)** | Proveedor | UC-11 crea la **familia de tokens** (raíz) y emite el primer par. UC-13 rota esa familia. |
| **UC-12 (Logout)** | Consumidor | UC-12 revoca tokens individuales o la familia completa (`CASCADE_LOGOUT`). UC-13 respeta esas revocaciones. |
| **UC-15 (MFA)** | Integración | Si la frescura de MFA expiró, UC-13 retorna `mfa_required` en lugar de tokens (step-up). |
| **UC-17 (Risk-Based Auth)** | Consumidor | UC-13 invoca la evaluación de riesgo en cada renovación con el delta de contexto (IP/UA/geo). |
| **UC-20 / UC-23 (Session Management)** | Consumidor | El claim `sid` (family root id) y la tabla `refresh_tokens` alimentan el listado de sesiones activas. |
| **UC-12 (Blacklist)** | Integración | En revocación de familia por `REUSE_DETECTED`, los JTIs emitidos (`issued_jti`) se blacklistean en Redis con TTL residual. |

### 1.3. Objetivos de Rendimiento

| Métrica | Objetivo | Justificación |
| :--- | :--- | :--- |
| **Latencia de Refresh (p95)** | < 150 ms | SELECT por hash único + transacción de rotación (UPDATE+INSERT) + firma JWT. |
| **Latencia de Refresh (p99)** | < 250 ms | Contención ocasional de locks de fila en rotaciones concurrentes. |
| **Throughput** | 800 refresh/segundo | Pico de renovación matutino (apertura de jornada) en tenants enterprise. |
| **Falsos Positivos de Reuse Detection** | < 0.01% | La Grace Window debe absorber toda concurrencia legítima multi-tab. |
| **Disponibilidad** | 99.99% | Si el refresh cae, todos los usuarios son expulsados al expirar su Access Token (15 min). |

---

## 2. Descripción Analítica (ICONIX Clásico)

| Campo | Detalle |
| :--- | :--- |
| **ID Y NOMBRE:** | UC-13 - Renovar Sesión (Refresh Token Rotation & Reuse Detection) |
| **ESTADO:** | Preliminar |
| **DESCRIPCIÓN:** | El cliente envía su Refresh Token vigente al endpoint de renovación. El sistema localiza el token por su hash SHA256, valida su estado (ACTIVE, no revocado, no expirado, familia dentro de vida absoluta) y ejecuta la rotación: marca el token como `ROTATED` con referencia a su sucesor, crea un nuevo Refresh Token opaco de alta entropía, y emite un nuevo Access Token JWT preservando el nivel de autenticación (`auth_level`) y el identificador de sesión (`sid` = family root). Si el token presentado ya estaba `ROTATED`, se evalúa la Grace Window: dentro de ella se responde idempotentemente con el sucesor existente; fuera de ella se declara `REUSE_DETECTED`, se revoca la familia completa, se blacklistean los JTIs emitidos y se notifica al usuario y al SOC. |
| **ACTOR PRINCIPAL:** | Cliente (SPA / Mobile App) en nombre del Usuario Autenticado. |
| **ACTOR SECUNDARIO:** | API Gateway, KAGAMI (Token Store DB), Redis (Grace Cache + Blacklist), UC-17 (Risk Engine), KAGE (Audit), Notification Service. |
| **PRECONDICIONES:** | 1. El cliente posee un Refresh Token opaco emitido por UC-11 o por una rotación previa de UC-13.<br>2. La familia de tokens no superó su vida absoluta (30 días).<br>3. El tenant no tiene sesiones globalmente suspendidas (incidente de seguridad). |
| **PUNTOS DE EXTENSIÓN:** | Si el delta de contexto (IP/UA/geo) supera el umbral de política, se extiende con step-up de MFA (UC-15) o revocación preventiva. Si la frescura de MFA del tenant expiró, se retorna `mfa_required`. |
| **CONDICIÓN:** | El cliente recibe un par de tokens nuevo; el token presentado queda inutilizable como credencial primaria (ROTATED). |

### 2.1. Escenario Principal (Happy Path) - Rotación Exitosa

1. El cliente envía `POST /auth/refresh` con `{ "refresh_token": "<opaco>" }`.
2. El API Gateway aplica Rate Limiting por identidad e IP y forward la request.
3. El Refresh Handler calcula `token_hash = SHA256(refresh_token)`.
4. Consulta `refresh_tokens` por `token_hash` (índice único) obteniendo la fila con su familia (`family_root_id`, `generation`, `absolute_expires_at`, `status`).
5. **Guardas de validez:** existe → no revocado → no expirado (sliding) → familia dentro de vida absoluta. Cualquier fallo retorna `invalid_grant` genérico.
6. Estando `status = ACTIVE`, inicia transacción con lock de fila (`SELECT ... FOR UPDATE`):
   a. Crea el **sucesor**: nuevo token opaco de 256 bits; almacena su hash; hereda `family_root_id`, `tenant_id`, `identity_id`, `device_id`, `absolute_expires_at`; `generation = N+1`; registra `issued_jti` del Access Token que se emitirá.
   b. Marca el token presentado como `ROTATED` con `rotated_at = NOW()` y `replaced_by_token_id = sucesor.id`.
   c. Escribe en Redis la clave de Grace Window: `rotgrace:{hash_presentado} → AES-GCM(sucesor_opaco)` con TTL = 60s.
   d. Commit.
7. Emite nuevo Access Token JWT (15 min) con claims `sub`, `tenant_id`, `auth_level` (preservado), `amr` (preservado), `sid = family_root_id`, `jti` nuevo.
8. Invoca UC-17 (Risk Engine) con el delta de contexto. Si el score supera el umbral de política: revocación preventiva de familia + `invalid_grant` (ver excepción 4.7).
9. Inserta evento `Session.Refreshed` en `auth_outbox` (misma transacción o transacción inmediata posterior).
10. Retorna HTTP 200 con `{ access_token, refresh_token, token_type, expires_in }`.

### 2.2. Escenario Alternativo - Concurrencia Legítima (Grace Window)

1-5. Idénticos al escenario principal, pero la fila presenta `status = ROTATED`.
6. El sistema verifica la Grace Window: `rotated_at + 60s > NOW()` **y** existe la clave `rotgrace:{hash}` en Redis.
7. **Rotación Idempotente:** Decodifica el sucesor opaco desde la cache cifrada y emite un **nuevo Access Token** asociado al sucesor existente, **sin crear una nueva rotación** (la cadena no se bifurca).
8. Retorna HTTP 200 con el mismo `refresh_token` sucesor (el que la otra pestaña ya recibió) y un Access Token fresco.
9. **Justificación:** Dos pestañas que renuevan casi simultáneamente no deben destruir la sesión del usuario. La ventana de 60s absorbe la concurrencia legítima sin abrir una ventana de ataque significativa (el atacante que robe un token rotado tiene exactamente la misma ventana que ya tenía el token original activo).

### 2.3. Escenario Alternativo - Detección de Reuso (Compromiso Confirmado)

1-5. Idénticos, con `status = ROTATED`.
6. La Grace Window **no** aplica: `rotated_at + 60s <= NOW()` o la clave `rotgrace` no existe.
7. **Contención automática:**
   a. Se revocan **todos** los tokens de la familia (`family_root_id`) con motivo `REUSE_DETECTED`.
   b. Los `issued_jti` de las filas activas de la familia se escriben en la Blacklist de Redis con TTL residual.
   c. Se publica `Session.ReuseDetected` (evento de seguridad) y `Session.FamilyRevoked`.
   d. El Notification Service alerta al usuario ("Detectamos un intento de uso de una sesión antigua…").
8. Se retorna HTTP 400 `invalid_grant` (genérico; la contención y la notificación son los canales de comunicación reales).

### 2.4. Flujos de Excepción

**[4.1] Token Malformado**
- 4.1.1 El body no contiene `refresh_token` o su formato no corresponde a un token opaco emitido por SHUGO.
- 4.1.2 Retornar HTTP 400 `invalid_request`. No se toca la base de datos.

**[4.2] Token Inexistente, Revocado o Expirado (Zero Info Leakage)**
- 4.2.1 Cualquiera de las guardas del paso 5 falla.
- 4.2.2 Retornar HTTP 400 `invalid_grant` con mensaje genérico. No se revela cuál guarda falló.
- 4.2.3 Se registra el intento en auditoría con motivo interno (no expuesto al cliente).

**[4.3] Vida Absoluta Superada**
- 4.3.1 `absolute_expires_at <= NOW()` (30 días desde el login original).
- 4.3.2 Retornar HTTP 400 `invalid_grant` + evento `Session.AbsoluteLifetimeExceeded`.
- 4.3.3 El cliente debe redirigir a login interactivo (UC-11). No hay renovación que evite la vida absoluta.

**[4.4] Riesgo Elevado en la Renovación (UC-17)**
- 4.4.1 El Risk Engine retorna score sobre umbral de política (ej. IP de otro país + UA nuevo).
- 4.4.2 Según política del tenant: (a) step-up → retornar `mfa_required` con `mfa_token`; o (b) revocación preventiva de familia + `invalid_grant`.
- 4.4.3 Se publica `Session.RiskEscalation` para auditoría y SIEM.

**[4.5] Frescura de MFA Expirada (Step-Up Programado)**
- 4.5.1 La política del tenant exige re-verificación de MFA cada N horas (ej. 12h) y `last_mfa_at + N > NOW()`.
- 4.5.2 Retornar HTTP 200 con `{ "mfa_required": true, "mfa_token": "..." }` (UC-15 completará y reanudará la renovación).

**[4.6] Base de Datos de Tokens No Disponible (Fail-Secure)**
- 4.6.1 KAGAMI DB timeout o caída durante la rotación.
- 4.6.2 Retornar HTTP 503 `service_unavailable` con `Retry-After`. **Nunca** emitir tokens sin validar y rotar en la fuente de verdad.

**[4.7] Redis de Grace Cache No Disponible**
- 4.7.1 Un token `ROTATED` se presenta dentro de la ventana teórica, pero `rotgrace:{hash}` no puede leerse (Redis caído).
- 4.7.2 **Decisión conservadora:** retornar `invalid_grant` **sin revocar la familia** (evita falsos positivos destructivos durante outage de Redis).
- 4.7.3 Loguear warning `GraceCacheUnavailable`; la pestaña que posee el sucesor legítimo puede continuar normalmente.

### 2.5. Postcondiciones

**De Éxito (Rotación):**
1. El token presentado queda `ROTATED` con enlace a su sucesor (`replaced_by_token_id`).
2. Existe un nuevo token `ACTIVE` en la familia con `generation = N+1`.
3. El cliente posee un Access Token fresco (15 min) y el sucesor opaco.
4. Evento `Session.Refreshed` encolado en Outbox con contexto (IP, UA, generation).

**De Éxito (Grace Window):**
1. La cadena de la familia **no se bifurca**: no se crea un segundo sucesor.
2. El cliente concurrente recibe el mismo sucesor opaco que la otra pestaña.

**De Contención (Reuse Detected):**
1. Toda la familia está `REVOKED` con motivo `REUSE_DETECTED`.
2. Los JTIs emitidos de la familia están en la Blacklist de Redis con TTL residual.
3. El usuario fue notificado y el SOC recibió evento de seguridad.

**De Falla:**
1. No se emiten tokens; el estado de la familia permanece intacto (salvo contención).
2. El intento queda auditado con motivo interno.

---

## 3. Reglas de Negocio e Invariantes de Dominio (DDD)

| ID | Invariante / Regla | Descripción Técnica |
| :--- | :--- | :--- |
| **INV-82** | Rotación Obligatoria (Single-Use) | Todo Refresh Token es de un solo uso. Tras una rotación exitosa, su estado es `ROTATED` y nunca vuelve a `ACTIVE`. |
| **INV-83** | Reuse Detection con Revocación de Familia | Presentar un token `ROTATED` fuera de la Grace Window implica `REUSE_DETECTED`: se revoca la familia completa (`family_root_id`) y se blacklistean los `issued_jti` activos. |
| **INV-84** | Grace Window de Concurrencia (60s) | Un token `ROTATED` dentro de `rotated_at + 60s` con clave `rotgrace` vigente responde idempotentemente con el sucesor existente, sin bifurcar la cadena ni revocar la familia. |
| **INV-85** | Vida Absoluta de Sesión (30 días) | Toda familia tiene `absolute_expires_at = login_time + 30d`. Ninguna rotación la extiende. Al superarla, solo login interactivo (UC-11). |
| **INV-86** | Binding de Contexto | Cada fila de `refresh_tokens` conserva `device_id`, `user_agent`, `ip_address` de origen. Deltas significativos se evalúan contra UC-17 según política del tenant. |
| **INV-87** | Opacidad y Entropía del Refresh Token | Tokens opacos de 256 bits (32 bytes, Base64url). Nunca JWTs. Almacenados únicamente como hash SHA256. |
| **INV-88** | Preservación de Nivel de Autenticación | El Access Token renovado preserva `auth_level` y `amr` del login original. La renovación **nunca** eleva privilegios; solo los mantiene o degrada vía step-up. |
| **RN-23** | Expiración Deslizante con Tope Absoluto | `expires_at` de cada sucesor = `NOW() + 30d` deslizante, pero nunca mayor que `absolute_expires_at` de la familia. |
| **RN-24** | Auditoría de cada Rotación | Cada rotación (y cada rechazo) genera evento en Outbox: `Session.Refreshed`, `Session.ReuseDetected`, `Session.FamilyRevoked`, `Session.AbsoluteLifetimeExceeded`. |
| **RN-25** | Zero Info Leakage en Rechazos | Todos los rechazos de refresh retornan `invalid_grant` genérico. Los motivos internos viven solo en auditoría y métricas. |
| **RN-26** | Notificación al Usuario en Contención | Toda `REUSE_DETECTED` o revocación preventiva por riesgo dispara notificación al usuario (email/push) con contexto (IP, geo, tiempo). |

---

## 4. Criterios de Aceptación (BDD / Gherkin)

### Escenario 4.1: Renovación exitosa con rotación
```gherkin
DADO que el cliente posee un Refresh Token ACTIVE de la familia "fam-100" (generation 3)
CUANDO envía POST /auth/refresh con ese token
ENTONCES el sistema retorna HTTP 200 con un access_token nuevo (jti distinto, sid = fam-100)
Y un refresh_token opaco nuevo y distinto al presentado
Y el token presentado queda status = ROTATED con replaced_by_token_id = <nuevo id>
Y el nuevo token queda status = ACTIVE con generation = 4
Y se publica el evento "Session.Refreshed"
```

### Escenario 4.2: Concurrencia legítima dentro de la Grace Window
```gherkin
DADO que la pestaña A renovó hace 10 segundos y el token T quedó ROTATED con sucesor S
CUANDO la pestaña B envía POST /auth/refresh con el mismo token T
ENTONCES el sistema retorna HTTP 200
Y el refresh_token de la respuesta es exactamente S (el sucesor existente)
Y NO se crea una segunda rotación (generation no incrementa)
Y la familia NO es revocada
```

### Escenario 4.3: Reuso malicioso fuera de la Grace Window
```gherkin
DADO que el token T quedó ROTATED hace 5 minutos (fuera de la ventana de 60s)
CUANDO un atacante envía POST /auth/refresh con T
ENTONCES el sistema retorna HTTP 400 invalid_grant
Y TODOS los tokens de la familia quedan REVOKED con motivo REUSE_DETECTED
Y los issued_jti activos de la familia se añaden a la Blacklist de Redis
Y se publican "Session.ReuseDetected" y "Session.FamilyRevoked"
Y el usuario recibe una notificación de seguridad
```

### Escenario 4.4: Token revocado por logout (UC-12)
```gherkin
DADO que el usuario cerró sesión y su token quedó REVOKED
CUANDO el cliente intenta renovar con ese token
ENTONCES el sistema retorna HTTP 400 invalid_grant (genérico)
Y no se emiten tokens
Y el intento queda auditado con motivo interno "REVOKED_TOKEN_PRESENTED"
```

### Escenario 4.5: Expiración deslizante superada
```gherkin
DADO que el token ACTIVE tiene expires_at en el pasado (30 días sin uso)
CUANDO el cliente intenta renovar
ENTONCES el sistema retorna HTTP 400 invalid_grant
Y el cliente debe iniciar login interactivo (UC-11)
```

### Escenario 4.6: Vida absoluta de la familia superada
```gherkin
DADO que la familia fue creada hace 30 días y 2 horas (absolute_expires_at superado)
Y el token actual aún está ACTIVE y dentro de su expiración deslizante
CUANDO el cliente intenta renovar
ENTONCES el sistema retorna HTTP 400 invalid_grant
Y se publica "Session.AbsoluteLifetimeExceeded"
```

### Escenario 4.7: Delta de contexto de alto riesgo (step-up por política)
```gherkin
DADO que el tenant tiene política "strict_context" habilitada
Y el request de renovación proviene de una IP de otro país y User-Agent desconocido
CUANDO el Risk Engine (UC-17) retorna score 85 (umbral 70)
ENTONCES el sistema retorna HTTP 200 con { mfa_required: true, mfa_token: "..." }
Y no se emiten tokens de sesión hasta completar UC-15
```

### Escenario 4.8: Frescura de MFA expirada (step-up programado)
```gherkin
DADO que la política del tenant exige MFA cada 12 horas
Y el último MFA del usuario fue hace 13 horas
CUANDO el cliente renueva con un token ACTIVE válido
ENTONCES el sistema retorna HTTP 200 con { mfa_required: true, mfa_token: "..." }
Y tras completar UC-15, la renovación se reanuda emitiendo el par de tokens
```

---

## 5. Eventos de Dominio Generados

### Evento 1: `Session.Refreshed`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Session.Refreshed",
  "timestamp": "2026-09-06T09:15:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "familyRootId": "fam-100",
    "oldTokenId": "rt-uuid-77",
    "newTokenId": "rt-uuid-78",
    "generation": 4,
    "deviceId": "dev-hash-abc",
    "ipAddress": "192.168.1.100",
    "userAgent": "Mozilla/5.0...",
    "viaGraceWindow": false
  }
}
```
**Consumidores:** KAGE (Audit), KIZUNA (refresco de caché de sesión), Métricas (rotations por familia).

### Evento 2: `Session.ReuseDetected` (Evento de Seguridad)
```json
{
  "eventId": "uuid-v4",
  "eventType": "Session.ReuseDetected",
  "timestamp": "2026-09-06T09:20:00Z",
  "tenantId": "khcs-001",
  "severity": "SECURITY",
  "payload": {
    "identityId": "user-uuid-123",
    "familyRootId": "fam-100",
    "reusedTokenId": "rt-uuid-77",
    "rotatedAt": "2026-09-06T09:15:00Z",
    "successorId": "rt-uuid-78",
    "attackerIp": "203.0.113.66",
    "attackerUserAgent": "python-requests/2.31",
    "legitimateDeviceId": "dev-hash-abc"
  }
}
```
**Consumidores:** KAGE (Audit forense), SIEM (alerta crítica), Notification Service (aviso al usuario), TORII (webhook de seguridad).

### Evento 3: `Session.FamilyRevoked`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Session.FamilyRevoked",
  "timestamp": "2026-09-06T09:20:01Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "familyRootId": "fam-100",
    "tokensRevokedCount": 2,
    "jtisBlacklisted": ["jti-aaa", "jti-bbb"],
    "reason": "REUSE_DETECTED"
  }
}
```
**Consumidores:** KIZUNA (invalidar cachés de autorización por `sid`), KAGE, métricas de contención.

### Evento 4: `Session.AbsoluteLifetimeExceeded`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Session.AbsoluteLifetimeExceeded",
  "timestamp": "2026-10-06T09:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "familyRootId": "fam-100",
    "familyCreatedAt": "2026-09-06T09:00:00Z",
    "absoluteExpiresAt": "2026-10-06T09:00:00Z"
  }
}
```

---

## 6. Consideraciones de Arquitectura y Seguridad

### 6.1. Token Family Chain Validation

La familia de tokens se modela como una **cadena** enlazada por `replaced_by_token_id` con raíz `family_root_id` (= claim `sid` del JWT).

```sql
-- Rotación atómica: lock de fila previene doble rotación concurrente
BEGIN;
SELECT * FROM kagi_auth.refresh_tokens
WHERE id = :presented_id FOR UPDATE;

-- Insert sucesor (hereda familia y vida absoluta)
INSERT INTO kagi_auth.refresh_tokens
  (id, token_hash, identity_id, tenant_id, device_id, family_root_id,
   generation, expires_at, absolute_expires_at, issued_jti, status)
VALUES
  (:new_id, :new_hash, :identity, :tenant, :device, :family_root,
   :gen_plus_1, LEAST(NOW() + INTERVAL '30 days', :absolute_expires_at),
   :absolute_expires_at, :new_jti, 'ACTIVE');

-- Marcar presentado como ROTADO
UPDATE kagi_auth.refresh_tokens
SET status = 'ROTATED', rotated_at = NOW(), replaced_by_token_id = :new_id
WHERE id = :presented_id;
COMMIT;
```

**Justificación:** El `SELECT ... FOR UPDATE` serializa rotaciones concurrentes sobre la misma fila: la segunda petición espera y luego observa `ROTATED`, entrando al camino de Grace Window o Reuse Detection. Sin este lock, dos rotaciones paralelas bifurcarían la cadena y generarían falsos positivos de robo.

### 6.2. Grace Window: Cache Efímera Cifrada

```csharp
// Al rotar: publicar el sucesor para concurrencia legítima
var graceKey = $"rotgrace:{presentedTokenHash}";
var encryptedSuccessor = _aead.Encrypt(successorOpaqueToken); // AES-256-GCM con clave KAGI
await _redis.StringSetAsync(graceKey, encryptedSuccessor, TimeSpan.FromSeconds(60));

// Al recibir un token ROTADO dentro de la ventana:
var cached = await _redis.StringGetAsync(graceKey);
if (!cached.IsNullOrEmpty)
{
    var successor = _aead.Decrypt(cached);      // Rotación idempotente
    return IssuePairForExistingSuccessor(successor);
}
// Fuera de ventana o sin cache → Reuse Detection (o fallback conservador si Redis caído)
```

**Justificación de seguridad:** El valor en cache vive **cifrado** y solo 60 segundos. Un atacante con lectura de Redis obtiene un ciphertext AES-GCM cuya clave reside en KMS/KAGI, no en Redis. Un atacante con el token rotado en mano ya podía usarlo dentro de esa misma ventana antes de la rotación: la Grace Window **no amplía** la superficie de ataque original.

### 6.3. Vida Absoluta vs. Expiración Deslizante

- **Deslizante (`expires_at`):** Se renueva con cada rotación (`NOW() + 30d`), permitiendo sesiones activas continuas.
- **Absoluta (`absolute_expires_at`):** Fija desde el login original. Ninguna rotación la mueve.
- **Justificación:** Sin vida absoluta, un token robado y rotado periódicamente por un atacante mantendría una sesión viva **para siempre** (zombie session). La vida absoluta fuerza re-autenticación interactiva periódica, cumpliendo el espíritu de NIST SP 800-63B sobre re-verificación.

### 6.4. Binding de Contexto y Re-evaluación de Riesgo

Cada renovación compara el contexto actual (IP, UA, device fingerprint, geo) contra el contexto de origen de la fila. Deltas significativos se envían a UC-17:

| Delta de Contexto | Acción por Política |
| :--- | :--- |
| IP misma /8, UA idéntico | Renovar normalmente |
| IP distinta /16, UA idéntico | Renovar + log de contexto (score bajo) |
| IP distinto país + UA nuevo | Step-up MFA o revocación preventiva (score alto) |
| Device fingerprint desconocido | Step-up MFA obligatorio |

### 6.5. Preservación de `auth_level` y Degradación Controlada

El Access Token renovado **hereda** `auth_level` y `amr`. La renovación nunca eleva nivel (no se puede pasar de AAL1 a AAL2 renovando). La única transformación permitida es la **degradación vía step-up**: si la política exige MFA y la frescura expiró, no se emiten tokens hasta completar UC-15.

---

## 7. Contratos de API (OpenAPI 3.0 Snippets)

### 7.1. Endpoint: POST /auth/refresh

**Request:**
```http
POST /v1/auth/refresh HTTP/1.1
Host: auth.shugo.io
Content-Type: application/json

{
  "refresh_token": "o39xKd82mQwZr7LpVbNcXtYhFjA5sEuG..."
}
```

**Response (200 OK - Rotación):**
```json
{
  "token_type": "Bearer",
  "access_token": "eyJhbGciOiJFZERTQSIs...",
  "expires_in": 900,
  "refresh_token": "nuevoTokenOpacoSucesor..."
}
```

**Response (200 OK - Step-up MFA):**
```json
{
  "mfa_required": true,
  "mfa_token": "mfa_session_uuid...",
  "allowed_methods": ["totp", "webauthn"]
}
```

**Response (400 Bad Request - Rechazo genérico):**
```json
{
  "error": "invalid_grant",
  "error_description": "The provided refresh token is not valid."
}
```

**Response (503 Service Unavailable - Fail-Secure):**
```json
{
  "error": "service_unavailable",
  "error_description": "Session service temporarily unavailable."
}
```

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `02_Use_Case_Diagram.md` | Mostrará la interacción entre Cliente, Refresh Handler, Token Store, Grace Cache, Risk Engine y Outbox. |
| `04_Initial_Domain_Model.md` | Definirá la Entity `RefreshToken` extendida (family_root_id, generation, absolute_expires_at, issued_jti) y el Domain Service `TokenRotator`. |
| `05_State_Machine_Diagram.md` | Modelará la cadena de estados ACTIVE → ROTATED → (REVOKED/EXPIRED) y la máquina de la familia. |
| `10_UC_Specific_Data_Model.md` | Definirá la migración que añade `family_root_id`, `generation`, `absolute_expires_at`, `issued_jti` y los índices de familia. |
| `11_UC_Specific_Security_Threats.md` | Analizará Token Theft paralelo, Zombie Sessions, Grace Window Abuse y Family Revocation Bypass. |
| `UC-11/01_Requirements_Specification.md` | Documento hermano que crea la familia raíz y emite el primer par de tokens. |
| `UC-12/01_Requirements_Specification.md` | Documento hermano que consume `REUSE_DETECTED` para revocación y define la Blacklist donde se escriben los `issued_jti`. |

---

## 9. Glosario de Términos Específicos de UC-13

| Término | Definición |
| :--- | :--- |
| **Token Family** | Conjunto de Refresh Tokens encadenados por rotaciones sucesivas desde un login original. Identificado por `family_root_id` (= claim `sid`). |
| **Rotation (Rotación)** | Acto de invalidar el token presentado (ROTATED) y emitir un sucesor ACTIVE en la misma transacción. |
| **Successor Token** | El nuevo Refresh Token opaco creado durante una rotación; enlazado desde el presentado vía `replaced_by_token_id`. |
| **Reuse Detection** | Mecanismo (RFC 6819 §5.2.2.3) que interpreta la presentación de un token ya ROTADO (fuera de Grace Window) como evidencia de robo y revoca la familia. |
| **Grace Window (Leeway)** | Ventana de 60s tras una rotación durante la cual el token rotado responde idempotentemente con el sucesor existente, absorbiendo concurrencia legítima multi-tab. |
| **Absolute Lifetime** | Vida máxima fija de una familia (30 días desde el login). Inextensible por rotación; previene zombie sessions. |
| **Sliding Expiration** | Expiración del token individual que se renueva con cada rotación, topada por la vida absoluta. |
| **issued_jti** | JTI del Access Token emitido junto a cada Refresh Token. Permite blacklisteado quirúrgico de Access Tokens vivos al revocar una familia. |
| **Step-up** | Degradación controlada de la renovación: en lugar de emitir tokens, se exige re-verificación de MFA (UC-15) por riesgo o frescura expirada. |
| **Zombie Session** | Sesión mantenida viva indefinidamente por un atacante que rota periódicamente un token robado. Prevenida por la vida absoluta. |

---
*Fin del Documento 01_Requirements_Specification para UC-13*