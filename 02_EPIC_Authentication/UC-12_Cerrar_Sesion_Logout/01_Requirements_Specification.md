# UC-12: Cerrar Sesión (Logout) - Especificación de Requisitos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-12 - Cerrar Sesión (Logout & Token Revocation) |
| **Documento:** | 01_Requirements_Specification.md |
| **Metodología:** | ICONIX (Deep Dive) + DDD + Event-Driven Architecture + Security by Design |
| **Estándares:** | NIST SP 800-63B (Session Management), OWASP ASVS V3 (Session Management), GDPR Art. 32, SOC 2 CC6.1 |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 17/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 17/09/2026 | Gastón Ruiz | **Creación inicial.** Definición completa de la especificación analítica para el cierre de sesión. Establece el flujo de revocación de Refresh Tokens, la invalidación de Access Tokens (JWT) mediante Blacklist en Redis, la propagación de eventos a sistemas downstream (KIZUNA, KAGE), y el soporte para "Logout Everywhere" (Cascade Logout). | Documento base para el paquete de diseño Tier 1 del UC-12. Es el caso de uso crítico que garantiza que una sesión no pueda ser reutilizada tras su finalización, mitigando riesgos de Session Hijacking y cumpliendo con requisitos de seguridad en dispositivos compartidos. |

---

## 1. Introducción y Alcance

Este documento detalla la especificación de requisitos para el Caso de Uso **UC-12**, que representa la operación de **Cerrar Sesión (Logout)** en el motor **KAGI** de SHUGO.

A diferencia del Login (UC-11), que se centra en la validación criptográfica, el Logout se centra en la **invalidación de estado y la propagación de la revocación**. En una arquitectura basada en JWT (Stateless), el logout no es trivial: requiere mechanisms explícitos para "matar" tokens que técnicamente siguen siendo válidos por su firma criptográfica hasta que expire su tiempo de vida.

### 1.1. Importancia Estratégica

El UC-12 es **crítico para la seguridad de la sesión y el cumplimiento normativo** porque:

- **Prevención de Session Hijacking:** Si un token es robado, el usuario legítimo debe poder invalidarlo inmediatamente desde otro dispositivo.
- **Dispositivos Compartidos:** En quioscos, computadoras públicas o dispositivos móviles corporativos, un logout robusto es la única barrera entre un usuario saliente y el siguiente.
- **Cumplimiento (SOC 2 / ISO 27001):** Requiere mecanismos explícitos para terminar sesiones y asegurar que los permisos de acceso sean revocados inmediatamente.
- **Sincronización de Estado:** El logout debe propagarse al motor de autorización (KIZUNA) para limpiar cachés de permisos y al motor de auditoría (KAGE) para registrar la finalización de la sesión.

### 1.2. Relación con Otros Casos de Uso

| UC | Relación | Descripción |
| :--- | :--- | :--- |
| **UC-11 (Login)** | Complementario | UC-11 crea la sesión; UC-12 la destruye. |
| **UC-13 (Refresh Token)** | Consumidor | UC-12 invalida el Refresh Token actual, deteniendo el ciclo de rotación. |
| **UC-21 (Force Logout)** | Integración | Un administrador puede invocar la lógica de UC-12 remotamente contra un usuario específico. |
| **UC-20 (Gestionar Sesiones)** | Consumidor | Permite al usuario ver y terminar sesiones específicas (dispositivos activos). |

### 1.3. Objetivos de Rendimiento

| Métrica | Objetivo | Justificación |
| :--- | :--- | :--- |
| **Latencia de Logout (p95)** | < 100 ms | El logout debe ser percibido como instantáneo por el usuario. |
| **Throughput** | 1.000 logouts/segundo | Debe soportar picos (ej. final de turno en empresas grandes). |
| **Disponibilidad** | 99.99% | Si el logout falla, el usuario queda "atrapado" o expuesto. |
| **Propagación a Downstream** | < 5 segundos | KIZUNA y KAGE deben recibir la notificación casi en tiempo real. |

---

## 2. Descripción Analítica (ICONIX Clásico)

| Campo | Detalle |
| :--- | :--- |
| **ID Y NOMBRE:** | UC-12 - Cerrar Sesión (Logout & Token Revocation) |
| **ESTADO:** | Preliminar |
| **DESCRIPCIÓN:** | El usuario (o un administrador en su nombre) solicita finalizar su sesión activa. El sistema recibe el Refresh Token (o el Access Token), verifica su validez, y procede a revocarlo. Para los Access Tokens (JWT), se añade su identificador único (JTI) a una "Blacklist" en Redis con un TTL igual al tiempo de vida restante del token. Para los Refresh Tokens, se marcan como revocados en la base de datos. Finalmente, se publican eventos de dominio para notificar a los sistemas downstream y se retorna una respuesta exitosa al cliente. |
| **ACTOR PRINCIPAL:** | Usuario Autenticado (Resource Owner), Administrador (para Force Logout). |
| **ACTOR SECUNDARIO:** | API Gateway, Redis (Blacklist), KAGAMI (DB de Tokens), KAGE (Audit), KIZUNA (Auth Cache). |
| **PRECONDICIONES:** | 1. El usuario posee un Access Token y un Refresh Token válidos (o el token está expirado pero se desea limpiar el estado).<br>2. El sistema de Redis está operativo para la blacklist. |
| **PUNTOS DE EXTENSIÓN:** | Si se solicita "Logout Everywhere" (Cascade), el sistema itera sobre todos los Refresh Tokens activos de la identidad y los revoca. |
| **CONDICIÓN:** | Los tokens son inválidos para futuros usos. El cliente limpia su almacenamiento local (cookies/localStorage). |

### 2.1. Escenario Principal (Happy Path) - Logout de Dispositivo Actual

1. El cliente envía `POST /auth/logout` con el `refresh_token` en el cuerpo (o en el header de autorización).
2. El API Gateway valida que el request esté autenticado (opcional, pero recomendado para auditoría).
3. El Logout Handler extrae el `refresh_token` y el `access_token` (si se proporciona para blacklist inmediata).
4. **Validación de Idempotencia:** Se verifica si el token ya fue revocado. Si es así, se retorna éxito inmediatamente (204 No Content).
5. **Revocación de Refresh Token:** Se marca el token como `revoked` en la base de datos (o se elimina).
6. **Blacklist de Access Token (JWT):** Se extrae el `jti` (JWT ID) y el `exp` (expiration) del Access Token. Se guarda en Redis: `SET blacklist:{jti} 1 EX {tiempo_restante}`.
7. **Publicación de Eventos:** Se genera el evento `Session.Terminated`.
8. El sistema retorna HTTP 204 No Content.
9. El cliente elimina los tokens de su almacenamiento local.

### 2.2. Escenario Alternativo - Logout Everywhere (Cascade)

1. El usuario solicita "Cerrar sesión en todos los dispositivos" desde su panel de seguridad.
2. El Logout Handler identifica al usuario mediante el Access Token actual.
3. El sistema consulta todos los `refresh_tokens` activos asociados a esa `identity_id`.
4. Para cada token encontrado:
   a. Se marca como `revoked` en la base de datos.
   b. (Opcional) Se envía una señal de "kill" a los WebSockets/push channels si existen.
5. Se publica el evento `Session.TerminatedEverywhere`.
6. Se retorna HTTP 204 No Content.

### 2.3. Flujos de Excepción

**[4.1] Token Inválido o Malformado**
- 4.1.1 El cliente envía un token que no es un JWT válido o no existe en la DB.
- 4.1.2 Por seguridad (para no revelar información), el sistema retorna HTTP 204 No Content (Idempotencia).
- 4.1.3 Se registra un evento de auditoría de tipo "LogoutAttempt_InvalidToken" para análisis de seguridad.

**[4.2] Fallo en Redis (Blacklist)**
- 4.2.1 El sistema intenta añadir el JTI a la blacklist de Redis, pero Redis está caído.
- 4.2.2 **Estrategia Fail-Secure:** El sistema registra el intento en una cola de reintentos local o en la tabla `auth_outbox`.
- 4.2.3 El logout se considera exitoso para el Refresh Token, pero el Access Token podría seguir siendo válido hasta su expiración natural (riesgo aceptado temporalmente). Se alerta a operaciones.

### 2.4. Postcondiciones

**De Éxito:**
1. El Refresh Token no puede ser usado para obtener nuevos Access Tokens (UC-13 lo rechazará).
2. El Access Token es rechazado por el API Gateway o Middleware de autorización si se presenta nuevamente (gracias a la Blacklist).
3. El evento `Session.Terminated` es ingerido por KAGE.
4. Las cachés de autorización en KIZUNA asociadas a esa sesión son invalidadas.

---

## 3. Reglas de Negocio e Invariantes de Dominio (DDD)

| ID | Invariante / Regla | Descripción Técnica |
| :--- | :--- | :--- |
| **INV-77** | Idempotencia del Logout | Llamar al endpoint de logout múltiples veces con el mismo token debe retornar siempre 204 No Content, sin generar errores. |
| **INV-78** | Revocación Inmediata de Refresh Token | Un Refresh Token revocado no puede ser usado para el flujo de UC-13 (Renovación). |
| **INV-79** | Blacklist de Access Token (TTL Dinámico) | Los JTIs de los Access Tokens se almacenan en Redis con un TTL exacto igual a `exp - now()`. Esto previene el crecimiento infinito de la blacklist. |
| **INV-80** | Cascade Logout (Logout Everywhere) | Si se solicita, debe revocar *todos* los Refresh Tokens activos de la identidad, no solo el actual. |
| **INV-81** | Auditoría Obligatoria | Todo logout (exitoso, fallido, o por token inválido) debe generar un evento de auditoría. |
| **RN-20** | Fail-Safe en Downstream | Si KIZUNA o KAGE no responden, el logout local (revocación de tokens) debe completarse. La propagación se maneja vía Outbox Pattern. |
| **RN-21** | Limpieza del Lado del Cliente | El servidor no puede forzar la limpieza del `localStorage` del navegador. El cliente es responsable de borrar los tokens tras recibir el 204. |
| **RN-22** | Validación de Propiedad | Solo el propietario del token (o un admin con permisos explícitos) puede revocarlo. No se puede revocar el token de otro usuario usando solo el token. |

---

## 4. Criterios de Aceptación (BDD / Gherkin)

### Escenario 4.1: Logout exitoso de dispositivo actual
```gherkin
DADO que el usuario tiene un Access Token y un Refresh Token válidos
CUANDO envía POST /auth/logout con el refresh_token
ENTONCES el sistema retorna HTTP 204 No Content
Y el refresh_token es marcado como revocado en la base de datos
Y el jti del access_token es añadido a la blacklist de Redis con TTL adecuado
Y se publica el evento "Session.Terminated"
```

### Escenario 4.2: Idempotencia (Logout repetido)
```gherkin
DADO que el usuario ya cerró sesión previamente con el token "XYZ"
CUANDO envía POST /auth/logout nuevamente con el token "XYZ"
ENTONCES el sistema retorna HTTP 204 No Content
Y no se generan errores ni eventos duplicados
```

### Escenario 4.3: Logout con token expirado o inválido
```gherkin
DADO que el usuario envía un token que ya expiró o es malformado
CUANDO envía POST /auth/logout
ENTONCES el sistema retorna HTTP 204 No Content (para no revelar información)
Y se registra un evento de auditoría "LogoutAttempt_InvalidToken"
```

### Escenario 4.4: Logout Everywhere (Cascade)
```gherkin
DADO que el usuario tiene sesiones activas en 3 dispositivos diferentes
CUANDO solicita "Cerrar sesión en todos los dispositivos"
ENTONCES el sistema retorna HTTP 204 No Content
Y los 3 refresh_tokens asociados a su identidad son marcados como revocados
Y se publica el evento "Session.TerminatedEverywhere"
```

### Escenario 4.5: Fallo en Redis durante Blacklist
```gherkin
DADO que Redis está caído temporalmente
CUANDO el usuario solicita logout
ENTONCES el refresh_token es revocado en la base de datos
Y el access_token no se añade a la blacklist (fallback a expiración natural)
Y se genera una alerta de operaciones "RedisBlacklistFailure"
Y el sistema retorna HTTP 204 No Content al usuario
```

---

## 5. Eventos de Dominio Generados

### Evento 1: `Session.Terminated`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Session.Terminated",
  "timestamp": "2026-09-05T12:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "tokenId": "refresh-token-uuid",
    "deviceFingerprint": "sha256-hash-of-device",
    "ipAddress": "192.168.1.100",
    "reason": "USER_INITIATED" 
  }
}
```
**Consumidores:**
1. **KAGE (Audit):** Registro forense de la finalización de sesión.
2. **KIZUNA (Auth):** Invalidation de cachés de permisos asociadas a esa sesión específica.
3. **TORII (Webhooks):** Notificación a sistemas externos (ej. SIEM) para correlación de seguridad.

### Evento 2: `Session.TerminatedEverywhere`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Session.TerminatedEverywhere",
  "timestamp": "2026-09-05T12:05:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "tokensRevokedCount": 4,
    "initiatedBy": "user-uuid-123",
    "reason": "SECURITY_COMPROMISE" 
  }
}
```

---

## 6. Consideraciones de Arquitectura y Seguridad

### 6.1. El Problema de la Revocación de JWT
Los Access Tokens en SHUGO son JWTs (JSON Web Tokens) firmados criptográficamente. Por diseño, son *stateless*: el servidor no necesita consultar la DB para validarlos, solo verifica la firma.
*   **El Conflicto:** Si un usuario hace logout, el JWT sigue siendo matemáticamente válido hasta que expire (ej. 15 minutos).
*   **La Solución (Blacklist de JTI):** Cada JWT emitido contiene un claim `jti` (JWT ID) único. Al hacer logout, extraemos el `jti` y lo guardamos en Redis: `SET blacklist:{jti} 1 EX 900`. El middleware de autorización, antes de aceptar un JWT, verifica si su `jti` está en Redis.
*   **Optimización:** El TTL de la clave en Redis es exactamente el tiempo restante hasta que el JWT expire naturalmente. Así, la blacklist se limpia sola y no crece infinitamente.

### 6.2. Idempotencia y Seguridad
Es vital que el endpoint de logout sea idempotente. Si un cliente envía el request dos veces por un error de red, no debe recibir un error 400 o 500. Debe recibir 204. Esto simplifica la lógica del cliente y previene que un atacante use respuestas de error para inferir el estado de un token.

### 6.3. Propagación Asíncrona (Outbox Pattern)
La revocación del token es la operación crítica y debe ser síncrona. Sin embargo, notificar a KIZUNA (para limpiar caché) y a KAGE (para auditar) no debe bloquear la respuesta al usuario.
*   Se utiliza el patrón **Outbox**: se inserta el evento en la tabla `auth_outbox` en la misma transacción que revoca el token. Un worker en background lee la tabla y publica en el bus de eventos.

---

## 7. Contratos de API (OpenAPI 3.0 Snippets)

### 7.1. Endpoint: POST /auth/logout

**Request:**
```http
POST /v1/auth/logout HTTP/1.1
Host: auth.shugo.io
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4...",
  "logout_everywhere": false 
}
```

**Response (204 No Content):**
```http
HTTP/1.1 204 No Content
Cache-Control: no-store
Pragma: no-cache
```
*Nota: Se incluyen headers de seguridad para evitar que proxies o navegadores cacheen la respuesta.*

**Response (400 Bad Request - Solo si falta el token en el body):**
```json
{
  "error": "invalid_request",
  "error_description": "Refresh token is required in the request body."
}
```

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `02_Use_Case_Diagram.md` | Mostrará la interacción entre el Cliente, KAGI, Redis (Blacklist) y la DB de Tokens. |
| `04_Initial_Domain_Model.md` | Definirá la entidad `RefreshToken` (estado `REVOKED`) y el servicio `TokenRevoker`. |
| `05_State_Machine_Diagram.md` | Mostrará la transición del `RefreshToken` de `ACTIVE` a `REVOKED`. |
| `10_UC_Specific_Data_Model.md` | Definirá la estructura de la tabla de Refresh Tokens y la estrategia de índices para la blacklist. |
| `11_UC_Specific_Security_Threats.md` | Analizará amenazas como Session Fixation y Token Replay post-logout. |

---

## 9. Glosario de Términos Específicos de UC-12

| Término | Definición |
| :--- | :--- |
| **JTI (JWT ID)** | Claim estándar de JWT que proporciona un identificador único para el token. Usado como clave en la Blacklist. |
| **Blacklist (Denylist)** | Lista de tokens (JTIs) que, aunque criptográficamente válidos, son rechazados por el sistema porque fueron revocados antes de su expiración. |
| **Idempotencia** | Propiedad de una operación que produce el mismo resultado sin importar cuántas veces se ejecute. Crítico para el Logout. |
| **Cascade Logout** | Acción de revocar todas las sesiones activas de un usuario en todos sus dispositivos simultáneamente. |
| **Session Hijacking** | Ataque donde un atacante roba una sesión válida (token) para suplantar al usuario. El Logout mitiga el impacto si el usuario detecta la anomalía. |
| **Fail-Safe** | Principio de diseño donde, ante un fallo de componentes secundarios (Redis, KAGE), la operación principal (Logout) sigue completándose para garantizar la seguridad. |

---
*Fin del Documento 01_Requirements_Specification para UC-12*