# UC-03: Desactivar Identidad - Especificación de Requisitos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-03 - Desactivar Identidad (Soft Delete) |
| **Documento:** | 01_Requirements_Specification.md |
| **Metodología:** | ICONIX (Deep Dive) + DDD + BDD + Event-Driven Architecture |
| **Estándares:** | NIST SP 800-63A (Lifecycle), ISO/IEC 27001 (Access Control), SOC 2 (Offboarding), GDPR (Data Retention) |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 30/08/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 30/08/2026 | Gastón Ruiz | **Creación inicial.** Definición completa de la especificación analítica para la desactivación de identidades. Establece el flujo de "Soft Delete", la transición de estado a `DEACTIVATED`, la revocación inmediata de sesiones (vía KAGI) y el inicio del período de gracia para el borrado físico (UC-05). | Documento base para el paquete de diseño Tier 1 del UC-03. Es crítico para procesos de *Offboarding* y respuesta a incidentes de seguridad, garantizando que el acceso se deniegue inmediatamente mientras se preserva la data para auditoría. |

---

## 1. Introducción y Alcance

Este documento detalla la especificación de requisitos para el Caso de Uso **UC-03**, que representa la operación de **desactivación lógica (Soft Delete)** de una identidad en el motor **KAGAMI** de SHUGO.

A diferencia de la actualización general (UC-02), la desactivación es una **acción de ciclo de vida crítica** con efectos secundarios inmediatos y de alto impacto en la seguridad:
1. **Bloqueo de Acceso:** La identidad no puede autenticarse ni ser autorizada.
2. **Revocación de Sesiones:** Todas las sesiones activas y tokens de refresco deben ser invalidados inmediatamente.
3. **Retención de Datos:** Los datos no se borran, sino que se marcan para auditoría y cumplimiento de políticas de retención (hacia UC-05 Purgar).

### 1.1. Importancia Estratégica

El UC-03 es la **primera línea de defensa** en situaciones de riesgo:
- **Offboarding de Empleados:** Cuando un empleado deja la organización (KHCS, Enterprise), su acceso debe cortarse en segundos, no en horas.
- **Respuesta a Incidentes:** Si se detecta una cuenta comprometida, la desactivación es la acción de contención inmediata.
- **Cumplimiento Normativo:** SOC 2 y ISO 27001 exigen procesos formales de revocación de acceso tras la terminación del empleo.

### 1.2. Relación con Otros Casos de Uso

| UC | Relación | Descripción |
| :--- | :--- | :--- |
| **UC-01 (Provisionar)** | Predecesor | Crea la identidad que eventualmente será desactivada. |
| **UC-02 (Actualizar)** | Alternativa General | UC-02 puede cambiar el estado, pero UC-03 es la acción especializada con permisos y flujos específicos. |
| **UC-04 (Consultar)** | Consumidor | Permite ver identidades desactivadas (con filtros) para auditoría. |
| **UC-05 (Purgar)** | Sucesor | UC-03 inicia el "Período de Gracia". UC-05 ejecuta el borrado físico (Hard Delete) tras expirar este período. |
| **UC-06 (Autenticar)** | Bloqueado | KAGI debe rechazar cualquier intento de login de una identidad `DEACTIVATED`. |

### 1.3. Objetivos de Rendimiento

| Métrica | Objetivo | Justificación |
| :--- | :--- | :--- |
| **Latencia p95** | < 100ms | Acción de seguridad crítica. Debe ser casi instantánea. |
| **Throughput** | 50 req/seg | Suficiente para despidos masivos o bloqueos de emergencia. |
| **Disponibilidad** | 99.99% | No puede fallar durante una crisis de seguridad. |
| **Consistencia de Sesiones** | < 5 segundos | Tiempo máximo para que todas las sesiones activas sean revocadas globalmente. |

---

## 2. Descripción Analítica (ICONIX Clásico)

| Campo | Detalle |
| :--- | :--- |
| **ID Y NOMBRE:** | UC-03 - Desactivar Identidad |
| **ESTADO:** | Preliminar |
| **DESCRIPCIÓN:** | Un administrador (o sistema automatizado de RRHH) solicita la desactivación de una identidad. El sistema valida permisos, verifica que la identidad no esté ya purgada, transiciona el estado a `DEACTIVATED`, registra el timestamp `deactivated_at`, persiste el cambio atómicamente con un log de auditoría, y publica un evento `Identity.Deactivated`. Este evento es consumido por KAGI para revocar sesiones y por KIZUNA para denegar futuras autorizaciones. |
| **ACTOR PRINCIPAL:** | Administrador de Tenant (Humano), Sistema de Offboarding (SCIM/HRIS). |
| **ACTOR SECUNDARIO:** | KAGI (Revocación de Sesiones), KIZUNA (Denegación de Acceso), KAGE (Auditoría). |
| **PRECONDICIONES:** | 1. La identidad existe y su estado actual permite la desactivación (ej. `ACTIVE`, `SUSPENDED`).<br>2. El actor tiene el permiso `identity:deactivate`.<br>3. La identidad no está en estado `PURGED`. |
| **PUNTOS DE EXTENSIÓN:** | Si la identidad tiene roles críticos o es la última Admin del tenant, el sistema puede requerir una confirmación adicional o bloqueo preventivo. |
| **CONDICIÓN:** | La identidad queda inaccesible inmediatamente, pero sus datos permanecen para auditoría. |

### 2.1. Escenario Principal (Happy Path) - Desactivación Exitosa

1. El administrador envía una solicitud `POST /identities/{id}/deactivate` (o `PATCH` con cambio de estado) con una razón opcional (ej. "Empleado renunció").
2. El API Gateway valida el JWT y el permiso `identity:deactivate`.
3. El Handler carga la identidad y verifica que no esté `PURGED`.
4. El Handler invoca el método `Deactivate(reason)` en el Aggregate Root `Identity`.
5. El Aggregate valida la transición de estado (ej. `ACTIVE` -> `DEACTIVATED`) y genera el evento de dominio `Identity.Deactivated`.
6. El Handler persiste el nuevo estado y el log de auditoría en una transacción atómica.
7. El Handler publica el evento `Identity.Deactivated` en el Outbox.
8. El sistema retorna HTTP 200 OK (o 204 No Content).
9. **Asíncronamente:** KAGI consume el evento y revoca todos los `RefreshTokens` y sesiones activas.

### 2.2. Escenario Alternativo - Identidad Ya Desactivada (Idempotencia)

1-3. (Idénticos al principal)
4. El Handler detecta que el estado actual ya es `DEACTIVATED`.
5. El sistema retorna HTTP 200 OK inmediatamente sin generar eventos duplicados ni logs de auditoría redundantes.
6. Fin del Caso de Uso.

### 2.3. Flujos de Excepción

**[3.1] Identidad No Encontrada o Purgada**
- 3.1.1 El ID no existe o la identidad tiene status `PURGED`.
- 3.1.2 Retorna HTTP 404 Not Found.
- 3.1.3 Fin del Caso de Uso.

**[4.1] Transición de Estado Inválida**
- 4.1.1 La identidad está en un estado que no permite desactivación (ej. ya está `DEACTIVATED` y se intenta forzar, o lógica de negocio específica lo impide).
- 4.1.2 Retorna HTTP 400 Bad Request "Invalid state transition".
- 4.1.3 Fin del Caso de Uso.

**[6.1] Error de Persistencia**
- 6.1.1 Falla la transacción de DB.
- 6.1.2 Retorna HTTP 500 Internal Server Error.
- 6.1.3 Fin del Caso de Uso.

**[9.1] Fallo en Revocación de Sesiones (KAGI Down)**
- 9.1.1 KAGI no consume el evento inmediatamente.
- 9.1.2 El evento permanece en el Outbox y se reintentará.
- 9.1.3 **Nota de Seguridad:** La identidad está desactivada en KAGAMI (Source of Truth). Si KAGI tiene caché de sesiones, estas expirarán por TTL o serán invalidadas en el próximo intento de uso (Fail Secure en Check API).

### 2.4. Postcondiciones

**De Éxito:**
1. El estado de la identidad es `DEACTIVATED`.
2. El campo `deactivated_at` tiene un timestamp válido.
3. El log de auditoría registra la acción y el actor.
4. El evento `Identity.Deactivated` está en la cola para KAGI/KIZUNA.

**De Falla:**
1. El estado de la identidad no ha cambiado.
2. No se han revocado sesiones prematuramente.

---

## 3. Reglas de Negocio e Invariantes de Dominio (DDD)

| ID | Invariante / Regla | Descripción Técnica |
| :--- | :--- | :--- |
| **INV-15** | Transición de Estado Válida | Solo identidades en estado `ACTIVE` o `SUSPENDED` pueden pasar a `DEACTIVATED`. Lanza `InvalidTransitionException`. |
| **INV-16** | Inmutabilidad Post-Desactivación | Una identidad `DEACTIVATED` no puede ser modificada (UC-02 bloquea updates) excepto para reactivación o purga. |
| **INV-17** | Registro de Timestamp | El campo `deactivated_at` es obligatorio y debe ser la hora exacta de la transición. Usado para calcular el período de gracia para UC-05. |
| **INV-18** | Prevención de Auto-Desactivación | Un usuario no puede desactivar su propia identidad (previene lockouts accidentales o maliciosos). |
| **RN-01** | Garantía de Revocación | La desactivación *debe* disparar la revocación de sesiones. Si el sistema de sesiones está caído, se usa el patrón Outbox para reintentar. |
| **RN-02** | Idempotencia | Desactivar una identidad ya desactivada es una operación segura (no-op) que retorna éxito. |
| **RN-03** | Retención de Datos | Los datos de una identidad desactivada se retienen por `TenantConfig.GracePeriodDays` antes de ser elegibles para Purga (UC-05). |

---

## 4. Criterios de Aceptación (BDD / Gherkin)

### Escenario 4.1: Desactivación exitosa de empleado (Offboarding)
```gherkin
DADO que la identidad "user-uuid-123" tiene status "ACTIVE"
Y el administrador "admin-001" tiene el permiso "identity:deactivate"
CUANDO el administrador envía POST /identities/user-uuid-123/deactivate con:
  | Campo  | Valor                        |
  |--------|------------------------------|
  | reason | "Contrato finalizado"        |
ENTONCES el sistema retorna HTTP 200 OK
Y la identidad ahora tiene status "DEACTIVATED"
Y el campo "deactivated_at" está establecido
Y se ha generado un evento "Identity.Deactivated"
```

### Escenario 4.2: Idempotencia (Ya desactivada)
```gherkin
DADO que la identidad "user-uuid-123" ya tiene status "DEACTIVATED"
CUANDO el administrador envía POST /identities/user-uuid-123/deactivate
ENTONCES el sistema retorna HTTP 200 OK
Y NO se generan eventos de dominio duplicados
Y NO se crea un nuevo log de auditoría
```

### Escenario 4.3: Prevención de Auto-Desactivación
```gherkin
DADO que el administrador "admin-001" intenta desactivar su propia identidad "admin-uuid-001"
CUANDO envía POST /identities/admin-uuid-001/deactivate
ENTONCES el sistema retorna HTTP 400 Bad Request
Y el mensaje es "Cannot deactivate own identity"
Y el estado de la identidad permanece "ACTIVE"
```

### Escenario 4.4: Desactivación de identidad inexistente
```gherkin
DADO que la identidad "user-uuid-999" no existe o está "PURGED"
CUANDO el administrador envía POST /identities/user-uuid-999/deactivate
ENTONCES el sistema retorna HTTP 404 Not Found
```

---

## 5. Eventos de Dominio Generados

### Evento 1: `Identity.Deactivated`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Identity.Deactivated",
  "timestamp": "2024-01-30T12:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "previousStatus": "ACTIVE",
    "reason": "Contrato finalizado",
    "deactivatedBy": "admin-001"
  },
  "metadata": {
    "traceId": "opentelemetry-trace-id",
    "priority": "HIGH" // Para asegurar procesamiento rápido en KAGI
  }
}
```

**Consumidores:**
1. **KAGI (Auth):** Revoca `RefreshTokens`, invalida sesiones en Redis, añade a lista negra temporal.
2. **KIZUNA (AuthZ):** Actualiza caché para denegar checks futuros (Fail Secure).
3. **KAGE (Audit):** Registra el evento en logs inmutables.
4. **TORII (Webhooks):** Notifica a sistemas externos (ej. "Usuario dado de baja").

---

## 6. Consideraciones de Arquitectura y Seguridad

### 6.1. Patrón de Comando vs. Actualización de Estado
Aunque UC-02 permite cambiar el estado vía `PATCH`, UC-03 define un **endpoint de acción explícito** (`POST /identities/{id}/deactivate`).

**Justificación:**
- **Semántica Clara:** "Desactivar" es una acción de negocio, no solo un cambio de campo.
- **Permisos Granulares:** Permite asignar el permiso `identity:deactivate` sin otorgar `identity:update` completo.
- **Validación Específica:** Permite reglas de negocio únicas (ej. "No puedes desactivar al último Admin del tenant").

### 6.2. Consistencia Eventual en la Revocación de Sesiones
Si KAGI está caído al momento de la desactivación:
1. La identidad se marca como `DEACTIVATED` en KAGAMI (Source of Truth).
2. El evento se encola en el Outbox.
3. **Fail Secure:** Si un usuario intenta usar un token viejo, KAGI (al recuperarse) o KIZUNA (al hacer un check) validará el estado actual en KAGAMI (o caché) y denegará el acceso.
4. **Mitigación:** Los Access Tokens tienen TTL corto (15 min). El daño máximo es una ventana de 15 minutos si la caché de KAGI no se invalida inmediatamente.

### 6.3. Período de Gracia (Grace Period)
El campo `deactivated_at` inicia un temporizador.
- **Configuración:** `TenantConfig.GracePeriodDays` (ej. 30 días).
- **Función:** Permite reactivar la identidad si fue un error (UC-02/UC-04).
- **Finalización:** Tras 30 días, un proceso batch (UC-05) elegirá esta identidad para Purga (Hard Delete).

---

## 7. Contratos de API (OpenAPI 3.0 Snippets)

### 7.1. Endpoint: POST /identities/{id}/deactivate

**Request:**
```http
POST /v1/identities/user-uuid-123/deactivate HTTP/1.1
Host: api.shugo.io
Authorization: Bearer <JWT>
Content-Type: application/json

{
  "reason": "Empleado renunció voluntariamente",
  "notify_user": false
}
```

**Response (200 OK):**
```json
{
  "id": "user-uuid-123",
  "status": "DEACTIVATED",
  "deactivated_at": "2024-01-30T12:00:00Z",
  "message": "Identity deactivated successfully"
}
```

**Response (400 Bad Request - Auto-deactivation):**
```json
{
  "error": {
    "code": "SELF_DEACTIVATION_NOT_ALLOWED",
    "message": "Cannot deactivate own identity"
  }
}
```

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `02_Use_Case_Diagram.md` | Mostrará los actores y la relación con KAGI/KIZUNA. |
| `03_Activity_Diagram.md` | Modelará el flujo de validación, transición de estado y publicación de eventos. |
| `04_Initial_Domain_Model.md` | Definirá el método `Deactivate()` en el Aggregate Root `Identity`. |
| `05_State_Machine_Diagram.md` | (UC-02) Ya define la transición `ACTIVE` -> `DEACTIVATED`. |
| `10_UC_Specific_Data_Model.md` | (UC-01/02) Define la columna `deactivated_at` y políticas de retención. |

---

## 9. Glosario de Términos Específicos de UC-03

| Término | Definición |
| :--- | :--- |
| **Soft Delete** | Marcado lógico de un registro como eliminado/inactivo sin borrarlo físicamente de la base de datos. |
| **Offboarding** | Proceso de salida de un usuario de una organización, que incluye la revocación de accesos. |
| **Grace Period** | Período de tiempo tras la desactivación durante el cual los datos se retienen antes del borrado físico. |
| **Session Revocation** | Invalidación forzosa de tokens y sesiones activas, cerrando la sesión del usuario en todos los dispositivos. |
| **Fail Secure** | Principio de diseño donde, ante una falla (ej. KAGI caído), el sistema deniega el acceso por defecto. |
| **Idempotencia** | Propiedad de una operación que produce el mismo resultado sin importar cuántas veces se ejecute. |

---
*Fin del Documento 01_Requirements_Specification para UC-03*