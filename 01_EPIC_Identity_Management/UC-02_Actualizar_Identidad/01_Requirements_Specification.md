# UC-02: Actualizar Identidad - Especificación de Requisitos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-02 - Actualizar Identidad |
| **Documento:** | 01_Requirements_Specification.md |
| **Metodología:** | ICONIX (Deep Dive) + DDD + BDD + Optimistic Concurrency Control |
| **Estándares:** | NIST SP 800-63A (Lifecycle), GDPR (Data Accuracy), RFC 7396 (JSON Merge Patch), SOC 2 (Audit Trails) |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 30/08/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 30/08/2026 | Gastón Ruiz | **Creación inicial.** Definición completa de la especificación analítica para la actualización de identidades. Establece el flujo de modificación de perfiles, atributos custom y transiciones de estado, con énfasis crítico en Control de Concurrencia Optimista (OCC), re-cifrado de PII, y generación de logs de auditoría con estados Before/After. | Documento base para el paquete de diseño Tier 1 del UC-02. Complementa al UC-01 (Creación) gestionando el ciclo de vida activo de la identidad, garantizando integridad de datos y auditabilidad forense. |

---

## 1. Introducción y Alcance

Este documento detalla la especificación de requisitos para el Caso de Uso **UC-02**, que representa el endpoint de **modificación y gestión del ciclo de vida** de una identidad existente en el motor **KAGAMI** de SHUGO.

El UC-02 es responsable de:
1. **Recibir solicitudes de actualización** (parciales o totales) sobre el perfil, atributos custom o estado de una identidad.
2. **Validar permisos y concurrencia:** Asegurar que el usuario tenga permisos y que no haya ocurrido una modificación simultánea (Race Condition).
3. **Re-cifrar PII si cambia:** Si el email o teléfono se actualizan, se deben volver a cifrar y actualizar el hash determinista de unicidad.
4. **Persistir los cambios atómicamente** y generar un registro de auditoría inmutable (Before/After).
5. **Publicar eventos de dominio** (`Identity.Updated`, `Profile.Modified`) para notificar a sistemas downstream.

### 1.1. Importancia Estratégica

Mientras que el UC-01 (Crear) es la puerta de entrada, el **UC-02 (Actualizar) es el motor del ciclo de vida diario**:
- **Precisión de Datos (GDPR Art. 5):** Garantiza que los datos personales sean exactos y estén actualizados.
- **Seguridad Operativa:** Permite suspender identidades comprometidas o reactivarlas tras una investigación.
- **Integridad Forense:** Cada cambio debe ser rastreable. ¿Quién cambió el rol de este usuario? ¿Cuándo se actualizó su email?

### 1.2. Relación con Otros Casos de Uso

| UC | Relación | Descripción |
| :--- | :--- | :--- |
| **UC-01 (Provisionar)** | Predecesor | Crea la identidad que UC-02 modificará. |
| **UC-03 (Desactivar)** | Especialización | UC-02 puede invocar transiciones de estado específicas (ej. `ACTIVE` -> `SUSPENDED`). |
| **UC-04 (Consultar)** | Complementario | Permite obtener el estado actual y el `RowVersion` necesario para actualizar. |
| **UC-46 (Auditoría)** | Consumidor | Consume los eventos de UC-02 para generar logs inmutables con estados Before/After. |

### 1.3. Objetivos de Rendimiento

| Métrica | Objetivo | Justificación |
| :--- | :--- | :--- |
| **Latencia p50** | < 80ms | La mayoría de actualizaciones son simples (ej. cambiar display_name). |
| **Latencia p95** | < 250ms | Incluye validación de concurrencia, re-cifrado de PII y escritura de auditoría. |
| **Throughput** | 150 req/seg | Las actualizaciones son más frecuentes que las creaciones en operación normal. |
| **Tasa de Conflictos (409)** | < 1% | Indica buena UX; si es alto, los usuarios están editando datos obsoletos. |

---

## 2. Descripción Analítica (ICONIX Clásico)

| Campo | Detalle |
| :--- | :--- |
| **ID Y NOMBRE:** | UC-02 - Actualizar Identidad |
| **ESTADO:** | Preliminar |
| **DESCRIPCIÓN:** | Un administrador (o sistema SCIM) envía una solicitud PATCH/PUT a `/identities/{id}` con los campos a modificar y el `RowVersion` (o `updated_at`) actual para control de concurrencia. El sistema valida el JWT, extrae el tenant_id, carga la identidad, verifica el `RowVersion` (Optimistic Locking), valida los nuevos datos (formato, schema), re-cifra el PII si cambió, actualiza las tablas `identities` y `profiles`, registra el cambio en el log de auditoría (Before/After), publica un evento `Identity.Updated`, y retorna HTTP 200 con la nueva versión. |
| **ACTOR PRINCIPAL:** | Administrador de Tenant, Sistema SCIM/HRIS, o el propio usuario (Self-service). |
| **ACTOR SECUNDARIO:** | Sistema SHUGO (KAGAMI Engine, KAGE Audit, KIZUNA Graph). |
| **PRECONDICIONES:** | 1. La identidad debe existir y estar en un estado modificable (no `PURGED`).<br>2. El JWT debe tener permisos `identity:update` o ser el propio usuario (para self-service limitado).<br>3. El cliente debe proveer el `RowVersion` o `If-Match` header actual. |
| **PUNTOS DE EXTENSIÓN:** | Si la actualización cambia el email, se extiende al flujo de re-verificación (UC-06). Si cambia el estado a `SUSPENDED`, se extiende a la revocación de sesiones en KAGI. |
| **CONDICIÓN:** | La identidad actualizada debe reflejar los nuevos datos manteniendo la integridad referencial y el aislamiento multi-tenant. |

### 2.1. Escenario Principal (Happy Path) - Actualización Exitosa

1. El cliente envía una solicitud `PATCH /identities/{id}` con el header `If-Match: <RowVersion>` y un payload JSON (RFC 7396 Merge Patch).
2. El API Gateway valida el JWT y los permisos (`identity:update`).
3. El Handler carga la identidad y el perfil desde PostgreSQL.
4. El Handler compara el `RowVersion` de la DB con el header `If-Match`. Si coinciden, continúa.
5. El Handler valida los nuevos datos (ej. formato de nuevo email, schema de nuevos custom attributes).
6. Si el email cambió, el Handler solicita al `PIIEncryptor` que recifre el nuevo email y calcule el nuevo `email_hash`.
7. El Handler aplica los cambios al Aggregate Root `Identity` (lo que genera un evento de dominio `Identity.Updated` con el snapshot Before/After).
8. El Handler persiste los cambios en una transacción atómica (actualiza `identities`, `profiles`, y escribe en `audit_logs`).
9. El Handler retorna HTTP 200 OK con el nuevo `RowVersion` y los datos actualizados.

### 2.2. Escenario Alternativo - Actualización Parcial de Atributos Custom (JSON Merge Patch)

1-4. (Idénticos al escenario principal)
5. El payload contiene solo `{"custom_attributes": {"clearance_level": 4}}`.
6. El Handler hace un *Deep Merge* del JSON existente con el nuevo, respetando el schema del tenant.
7-9. (Continúa con cifrado si aplica, persistencia y respuesta).

### 2.3. Flujos de Excepción

**[4.1] Conflicto de Concurrencia (Optimistic Locking Failure)**
- 4.1.1 El `RowVersion` proporcionado por el cliente es menor al actual en la base de datos.
- 4.1.2 El Handler aborta la operación para prevenir "Lost Updates".
- 4.1.3 Retorna HTTP 409 Conflict con el mensaje "Resource has been modified by another user. Please fetch the latest version."
- 4.1.4 Fin del Caso de Uso.

**[5.1] Identidad no Encontrada o Purgada**
- 5.1.1 El ID no existe o la identidad tiene status `PURGED`.
- 5.1.2 Retorna HTTP 404 Not Found.
- 5.1.3 Fin del Caso de Uso.

**[6.1] Error de Re-cifrado de PII**
- 6.1.1 El email cambió, pero el servicio KMS está caído.
- 6.1.2 Retorna HTTP 500 Internal Server Error. **Fail Secure:** No se actualiza ningún dato para mantener consistencia.
- 6.1.3 Fin del Caso de Uso.

**[8.1] Error de Persistencia o Auditoría**
- 8.1.1 Falla la transacción de DB o la escritura del log de auditoría.
- 8.1.2 Retorna HTTP 500. Se hace Rollback de toda la transacción.
- 8.1.3 Fin del Caso de Uso.

### 2.4. Postcondiciones

**De Éxito:**
1. Los campos solicitados han sido actualizados en `identities` y/o `profiles`.
2. Si el PII cambió, `email_cipher` y `email_hash` están actualizados.
3. Un registro inmutable en `audit_logs` documenta el cambio (Before/After).
4. El evento `Identity.Updated` está en la tabla `outbox_events` para su publicación.
5. El `RowVersion` de la identidad ha incrementado.

**De Falla:**
1. No se ha modificado ningún dato en la base de datos (Atomicidad).
2. No se han generado eventos de dominio ni logs de auditoría falsos.

---

## 3. Reglas de Negocio e Invariantes de Dominio (DDD)

| ID | Invariante / Regla | Descripción Técnica |
| :--- | :--- | :--- |
| **INV-09** | Optimistic Concurrency Control (OCC) | Toda actualización debe validar el `RowVersion` (o `updated_at` con precisión de microsegundos). Lanza `ConcurrencyConflictException`. |
| **INV-10** | Audit Trail (Before/After) | Toda modificación exitosa debe generar un log de auditoría con el estado anterior y posterior de los campos cambiados. |
| **INV-11** | PII Re-encryption | Si el email o teléfono cambian, deben ser re-cifrados y el `email_hash` actualizado en la misma transacción. |
| **INV-12** | JSON Merge Patch (RFC 7396) | Las actualizaciones de `custom_attributes` deben hacer merge, no reemplazo total, a menos que se envíe `null`. |
| **INV-13** | Immutable Fields | Los campos `id`, `tenant_id`, `type`, y `created_at` NUNCA pueden ser modificados vía UC-02. |
| **INV-14** | State Transition Rules | Las transiciones de estado (ej. `ACTIVE` -> `SUSPENDED`) deben respetar la máquina de estados definida en UC-01. |
| **RN-01** | Idempotencia Condicional | Si el cliente envía exactamente los mismos datos y el mismo `RowVersion`, el sistema puede retornar 200 sin cambios (o 409 si el versionado es estricto). |
| **RN-02** | Tenant Isolation | Solo se pueden actualizar identidades del `tenant_id` extraído del JWT. |

---

## 4. Criterios de Aceptación (BDD / Gherkin)

### Escenario 4.1: Actualización exitosa de Display Name (Happy Path)
```gherkin
DADO que existe la identidad "user-uuid-123" en el tenant "khcs-001" con status "ACTIVE" y RowVersion "v1"
Y el usuario "admin-001" tiene el permiso "identity:update"
CUANDO el administrador envía PATCH /identities/user-uuid-123 con:
  | Header        | Valor                              |
  |---------------|------------------------------------|
  | If-Match      | "v1"                               |
  | Body          | {"profile": {"display_name": "Juan Director"}} |
ENTONCES el sistema retorna HTTP 200 OK
Y la identidad ahora tiene display_name "Juan Director" y RowVersion "v2"
Y se ha generado un log de auditoría con:
  | Campo         | Valor                              |
  |---------------|------------------------------------|
  | field_changed | "profile.display_name"             |
  | old_value     | "Juan Veterinario"                 |
  | new_value     | "Juan Director"                    |
```

### Escenario 4.2: Actualización de Email (Re-cifrado y Hash)
```gherkin
DADO que la identidad tiene email "juan@khcs.com" (cifrado) y email_hash "hash_abc"
CUANDO el administrador actualiza el email a "juan.director@khcs.com"
ENTONCES el sistema re-cifra el nuevo email usando AES-256-GCM
Y actualiza el email_hash a "hash_xyz" (HMAC del nuevo email)
Y retorna HTTP 200 OK
```

### Escenario 4.3: Conflicto de Concurrencia (Optimistic Locking)
```gherkin
DADO que la identidad tiene RowVersion "v2" en la base de datos
Y el Cliente A y el Cliente B leen la identidad simultáneamente (ambos tienen "v1")
Y el Cliente A actualiza exitosamente (la DB ahora tiene "v2")
CUANDO el Cliente B intenta actualizar enviando If-Match: "v1"
ENTONCES el sistema retorna HTTP 409 Conflict
Y el mensaje es "Resource has been modified by another user"
Y los datos del Cliente B NO se han persistido
```

### Escenario 4.4: Actualización Parcial de Custom Attributes (Merge)
```gherkin
DADO que la identidad tiene custom_attributes: {"department": "Vet", "clearance": 3}
CUANDO el administrador envía PATCH con: {"profile": {"custom_attributes": {"clearance": 4}}}
ENTONCES el sistema realiza un Deep Merge
Y los custom_attributes finales son: {"department": "Vet", "clearance": 4}
Y NO se ha borrado el campo "department"
```

### Escenario 4.5: Intento de modificar campo inmutable
```gherkin
DADO que la identidad fue creada el "2024-01-01"
CUANDO el administrador envía PATCH con: {"created_at": "2024-01-02"}
ENTONCES el sistema ignora el campo o retorna HTTP 400 Bad Request
Y el campo "created_at" permanece inalterado
```

---

## 5. Eventos de Dominio Generados

### Evento 1: `Identity.Updated`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Identity.Updated",
  "timestamp": "2024-01-30T11:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "updatedBy": "admin-001",
    "changes": [
      {
        "field": "profile.display_name",
        "oldValue": "Juan Veterinario",
        "newValue": "Juan Director"
      },
      {
        "field": "profile.custom_attributes.clearance",
        "oldValue": 3,
        "newValue": 4
      }
    ]
  },
  "metadata": {
    "traceId": "opentelemetry-trace-id",
    "rowVersionBefore": "v1",
    "rowVersionAfter": "v2"
  }
}
```

### Evento 2: `Identity.StatusChanged` (Si la actualización modifica el estado)
```json
{
  "eventId": "uuid-v4",
  "eventType": "Identity.StatusChanged",
  "payload": {
    "identityId": "user-uuid-123",
    "oldStatus": "ACTIVE",
    "newStatus": "SUSPENDED",
    "reason": "Security investigation"
  }
}
```

---

## 6. Consideraciones de Arquitectura y Seguridad

### 6.1. Control de Concurrencia Optimista (OCC)
Para prevenir *Lost Updates* (cuando dos usuarios editan la misma identidad simultáneamente y uno sobrescribe al otro), se implementa OCC:
- **Opción A (Recomendada):** Columna `row_version` (UUID o Timestamp de alta precisión) que se actualiza en cada `UPDATE`.
- **Opción B:** Usar `updated_at` con precisión de microsegundos y validar en el `WHERE` clause (`WHERE id = ? AND updated_at = @original_updated_at`).

### 6.2. JSON Merge Patch (RFC 7396)
Para los `custom_attributes`, el endpoint debe soportar *Merge Patch*:
- `{"a": 1}` + `{"b": 2}` = `{"a": 1, "b": 2}`
- `{"a": 1}` + `{"a": null}` = `{}` (Borrado de campo)
Esto evita que el cliente tenga que enviar todo el objeto JSON cada vez que quiere cambiar un solo atributo.

### 6.3. Auditoría Forense (Before/After)
El log de auditoría no solo debe decir "Se actualizó la identidad X", sino **qué campos exactos cambiaron y cuáles eran sus valores anteriores**. Esto es crucial para investigaciones de seguridad y cumplimiento SOC 2.
*Nota de Privacidad:* Los valores `oldValue` y `newValue` de campos PII (email, phone) en el log de auditoría **deben estar cifrados o hasheados**, nunca en texto plano.

---

## 7. Contratos de API (OpenAPI 3.0 Snippets)

### 7.1. Endpoint: PATCH /identities/{id}

**Request:**
```http
PATCH /v1/identities/user-uuid-123 HTTP/1.1
Host: api.shugo.io
Authorization: Bearer <JWT>
If-Match: "v1"
Content-Type: application/merge-patch+json

{
  "profile": {
    "display_name": "Juan Director",
    "custom_attributes": {
      "clearance_level": 4
    }
  }
}
```

**Response (200 OK):**
```json
{
  "id": "user-uuid-123",
  "tenant_id": "khcs-001",
  "type": "USER",
  "status": "ACTIVE",
  "row_version": "v2",
  "updated_at": "2024-01-30T11:00:00Z",
  "profile": {
    "display_name": "Juan Director",
    "custom_attributes": {
      "department": "Vet",
      "clearance_level": 4
    }
  }
}
```

**Response (409 Conflict - Concurrencia):**
```json
{
  "error": {
    "code": "CONCURRENCY_CONFLICT",
    "message": "Resource has been modified by another user.",
    "details": {
      "expected_version": "v1",
      "current_version": "v2"
    }
  }
}
```

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `02_Use_Case_Diagram.md` | Mostrará los actores y las relaciones <<include>> con OCC y Auditoría. |
| `03_Activity_Diagram.md` | Modelará el flujo de validación de `RowVersion` y el Merge de JSON. |
| `04_Initial_Domain_Model.md` | Definirá el método `Update()` en el Aggregate Root `Identity` y la lógica de eventos de dominio. |
| `10_UC_Specific_Data_Model.md` | (UC-01) Definirá la columna `row_version` y la estructura de la tabla de auditoría. |
| `11_UC_Specific_Security_Threats.md` | (UC-01) Abordará las amenazas de "Lost Updates" y "Audit Log Tampering". |

---

## 9. Glosario de Términos Específicos de UC-02

| Término | Definición |
| :--- | :--- |
| **Optimistic Concurrency Control (OCC)** | Estrategia de control de concurrencia que asume que los conflictos son raros y los detecta al momento de guardar (usando versiones), en lugar de bloquear filas (Pessimistic Locking). |
| **RowVersion** | Un valor (UUID, Timestamp o número secuencial) que cambia cada vez que la fila se actualiza. Usado para OCC. |
| **JSON Merge Patch** | Estándar RFC 7396 que define cómo aplicar modificaciones parciales a un documento JSON. |
| **Lost Update** | Anomalía de concurrencia donde la transacción de un usuario sobrescribe silenciosamente los cambios de otro usuario. |
| **Before/After Snapshot** | Registro de auditoría que captura el estado de los datos antes y después de una transacción. |
| **Fail Secure (en actualizaciones)** | Si falla el cifrado o la auditoría, la actualización completa se revierte (Rollback). |

---
*Fin del Documento 01_Requirements_Specification para UC-02*