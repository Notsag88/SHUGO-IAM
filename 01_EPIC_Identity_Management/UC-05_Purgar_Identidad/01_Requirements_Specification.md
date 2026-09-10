# UC-05: Purgar Identidad - Especificación de Requisitos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-05 - Purgar Identidad (Hard Delete / GDPR Right to Erasure) |
| **Documento:** | 01_Requirements_Specification.md |
| **Metodología:** | ICONIX (Deep Dive) + DDD + BDD + Batch Processing + GDPR Compliance |
| **Estándares:** | GDPR Art. 17 (Right to Erasure), NIST SP 800-88 (Media Sanitization), ISO 27001 A.8.10, SOC 2 CC7.4, Legal Hold Requirements |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 02/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 02/09/2026 | Gastón Ruiz | **Creación inicial.** Definición completa de la especificación analítica para la purga de identidades. Establece el flujo de Hard Delete irreversible, la anonimización de datos PII, la preservación de logs de auditoría para cumplimiento legal, el procesamiento batch para purgas masivas, y las garantías de cumplimiento GDPR Art. 17 con manejo de excepciones para Legal Hold. | Documento base para el paquete de diseño Tier 1 del UC-05. Es el caso de uso más crítico desde la perspectiva de compliance y privacidad, ejecutando el Derecho al Olvido de forma segura, auditable e irreversible. |

---

## 1. Introducción y Alcance

Este documento detalla la especificación de requisitos para el Caso de Uso **UC-05**, que representa la operación de **purga irreversible (Hard Delete)** de identidades en el motor **KAGAMI** de SHUGO.

A diferencia de la desactivación (UC-03) que es un Soft Delete reversible, la purga es una operación **destructiva e irreversible** que:
1. **Elimina físicamente** los datos PII de la base de datos (email, teléfono, atributos custom).
2. **Anonimiza** los registros de auditoría asociados (reemplaza PII con hashes irreversibles).
3. **Notifica** a todos los sistemas downstream (KAGI, KIZUNA, KAGE) para que eliminen sus copias.
4. **Preserva** la evidencia de que la identidad existió y fue purgada (para cumplimiento legal).
5. **Respeta** Legal Hold (retención legal) cuando existe una orden judicial o investigación en curso.

### 1.1. Importancia Estratégica

El UC-05 es **crítico para cumplimiento normativo** porque:

- **GDPR Art. 17 (Right to Erasure):** Los usuarios tienen derecho a solicitar la eliminación de sus datos personales. SHUGO debe poder cumplir esta solicitud de forma técnica y auditable.
- **NIST SP 800-88 (Media Sanitization):** Define estándares para la eliminación segura de datos sensibles.
- **ISO 27001 A.8.10 (Information Deletion):** Requiere procedimientos para la eliminación segura de información.
- **SOC 2 CC7.4 (System Operations):** Exige procesos para la eliminación de datos según políticas de retención.
- **Legal Hold:** En casos de litigio o investigación regulatoria, ciertos datos no pueden ser eliminados aunque el usuario lo solicite.

### 1.2. Relación con Otros Casos de Uso

| UC | Relación | Descripción |
| :--- | :--- | :--- |
| **UC-01 (Provisionar)** | Predecesor | Crea la identidad que eventualmente será purgada. |
| **UC-02 (Actualizar)** | Predecesor | Puede modificar datos antes de la purga. |
| **UC-03 (Desactivar)** | Predecesor Obligatorio | La identidad debe estar `DEACTIVATED` antes de ser purgada. Inicia el Grace Period. |
| **UC-04 (Consultar)** | Consumidor | Después de la purga, retorna 404 (ocultamiento de existencia). |
| **UC-06 (Autenticar)** | Bloqueado | KAGI debe rechazar cualquier intento de login de una identidad purgada. |
| **UC-46 (Auditoría)** | Consumidor | Registra el evento de purga y anonimiza los logs históricos. |

### 1.3. Objetivos de Rendimiento

| Métrica | Objetivo | Justificación |
| :--- | :--- | :--- |
| **Latencia p95 (individual)** | < 500ms | Purga individual debe ser rápida, pero no crítica (es batch). |
| **Throughput (batch)** | 1000 identidades/minuto | Suficiente para purgas masivas de tenants grandes. |
| **Disponibilidad** | 99.9% | Es un proceso batch, puede reintentarse si falla. |
| **Consistencia de Purga** | 100% | Todos los sistemas downstream deben ser notificados. |
| **Tasa de error** | < 0.01% | Fallos en purga son críticos (datos no eliminados = violación GDPR). |

---

## 2. Descripción Analítica (ICONIX Clásico)

| Campo | Detalle |
| :--- | :--- |
| **ID Y NOMBRE:** | UC-05 - Purgar Identidad (Hard Delete) |
| **ESTADO:** | Preliminar |
| **DESCRIPCIÓN:** | Un administrador (o proceso automatizado de cumplimiento GDPR) solicita la purga de una o más identidades que están en estado `DEACTIVATED` y han superado el Grace Period (ej. 30 días). El sistema valida permisos, verifica que no haya Legal Hold activo, anonimiza los datos PII en la base de datos, anonimiza los logs de auditoría asociados, publica un evento `Identity.Purged` para notificar a sistemas downstream, y retorna confirmación. La operación es irreversible. |
| **ACTOR PRINCIPAL:** | Administrador de Tenant (Humano), Sistema de Cumplimiento GDPR (Automatizado). |
| **ACTOR SECUNDARIO:** | KAGI (Eliminación de sesiones), KIZUNA (Eliminación de caché de autorización), KAGE (Anonimización de logs), Sistema de Legal Hold (Validación de retención). |
| **PRECONDICIONES:** | 1. La identidad debe existir y estar en estado `DEACTIVATED`.<br>2. Debe haber transcurrido el Grace Period (ej. `NOW() - deactivated_at > 30 days`).<br>3. El actor debe tener el permiso `identity:purge`.<br>4. No debe haber Legal Hold activo sobre la identidad.<br>5. La identidad no debe estar en estado `PURGED` (idempotencia). |
| **PUNTOS DE EXTENSIÓN:** | Si hay Legal Hold activo, el sistema bloquea la purga y notifica al oficial de cumplimiento. Si la purga falla, el sistema reintenta automáticamente hasta 3 veces. |
| **CONDICIÓN:** | La identidad es eliminada físicamente de la base de datos, los logs de auditoría son anonimizados, y todos los sistemas downstream son notificados. La operación es irreversible. |

### 2.1. Escenario Principal (Happy Path) - Purga Individual Exitosa

1. El administrador envía una solicitud `POST /identities/{id}/purge` con una razón obligatoria (ej. "GDPR Art. 17 - Solicitud del usuario").
2. El API Gateway valida el JWT y el permiso `identity:purge`.
3. El Handler carga la identidad y verifica que esté en estado `DEACTIVATED`.
4. El Handler verifica que haya transcurrido el Grace Period (ej. 30 días desde `deactivated_at`).
5. El Handler consulta el sistema de Legal Hold para verificar que no haya retención legal activa.
6. El Handler invoca el método `Purge(reason)` en el Aggregate Root `Identity`.
7. El Aggregate valida la transición de estado (`DEACTIVATED` → `PURGED`) y genera el evento de dominio `Identity.Purged`.
8. El Handler anonimiza los datos PII en la base de datos (reemplaza email/phone con hashes irreversibles).
9. El Handler anonimiza los logs de auditoría asociados (reemplaza PII con hashes en `audit_logs`).
10. El Handler persiste el cambio de estado y los logs de auditoría anonimizados en una transacción atómica.
11. El Handler publica el evento `Identity.Purged` en el Outbox.
12. El sistema retorna HTTP 200 OK con confirmación de purga.
13. **Asíncronamente:** KAGI consume el evento y elimina todas las sesiones y tokens.
14. **Asíncronamente:** KIZUNA consume el evento y elimina la caché de autorización.
15. **Asíncronamente:** KAGE consume el evento y completa la anonimización de logs históricos.

### 2.2. Escenario Alternativo - Purga Masiva (Batch Processing)

1. El sistema de cumplimiento GDPR envía una solicitud `POST /identities/purge-batch` con una lista de IDs (máximo 1000).
2. El sistema valida permisos y crea un job de purga batch.
3. El job procesa las identidades en lotes de 100, validando cada una (estado, Grace Period, Legal Hold).
4. Para cada identidad válida, ejecuta el flujo de purga individual (pasos 6-15 del escenario principal).
5. El job retorna un `batch_id` y una URL para consultar el progreso (`GET /purge-jobs/{batch_id}`).
6. El administrador puede consultar el progreso: cuántas identidades fueron purgadas, cuántas fallaron, y por qué.

### 2.3. Escenario Alternativo - Identidad Ya Purgada (Idempotencia)

1-5. (Idénticos al escenario principal)
6. El Handler detecta que el estado actual ya es `PURGED`.
7. El sistema retorna HTTP 200 OK inmediatamente sin generar eventos duplicados ni logs de auditoría redundantes.
8. Fin del Caso de Uso.

### 2.4. Flujos de Excepción

**[3.1] Identidad No Encontrada o Ya Purgada**
- 3.1.1 El ID no existe o la identidad ya tiene status `PURGED`.
- 3.1.2 Retorna HTTP 404 Not Found (ocultamiento de existencia).
- 3.1.3 Fin del Caso de Uso.

**[4.1] Identidad No Está Desactivada**
- 4.1.1 La identidad está en estado `ACTIVE` o `SUSPENDED` (no `DEACTIVATED`).
- 4.1.2 Retorna HTTP 400 Bad Request "Identity must be deactivated before purging".
- 4.1.3 Fin del Caso de Uso.

**[4.2] Grace Period No Ha Transcurrido**
- 4.2.1 La identidad está `DEACTIVATED` pero no ha pasado el Grace Period (ej. solo 15 días de 30).
- 4.2.2 Retorna HTTP 400 Bad Request "Grace period has not expired. Purge available after {date}".
- 4.2.3 Fin del Caso de Uso.

**[5.1] Legal Hold Activo**
- 5.1.1 El sistema de Legal Hold indica que la identidad está bajo retención legal.
- 5.1.2 Retorna HTTP 403 Forbidden "Identity is under legal hold. Contact compliance officer".
- 5.1.3 Se genera alerta al oficial de cumplimiento.
- 5.1.4 Fin del Caso de Uso.

**[8.1] Error de Anonimización**
- 8.1.1 Falla el proceso de anonimización de PII (ej. error de base de datos).
- 8.1.2 Retorna HTTP 500 Internal Server Error.
- 8.1.3 Se hace Rollback de la transacción (la identidad NO se purga).
- 8.1.4 Se genera alerta crítica al equipo de operaciones.
- 8.1.5 Fin del Caso de Uso.

**[11.1] Fallo en Notificación a Sistemas Downstream**
- 11.1.1 KAGI, KIZUNA, o KAGE no consumen el evento inmediatamente.
- 11.1.2 El evento permanece en el Outbox y se reintentará.
- 11.1.3 **Nota de Seguridad:** La identidad ya está purgada en KAGAMI (Source of Truth). Los sistemas downstream eventualmente sincronizarán su estado.

### 2.5. Postcondiciones

**De Éxito:**
1. El estado de la identidad es `PURGED`.
2. Los datos PII han sido anonimizados en la base de datos (email/phone reemplazados con hashes).
3. Los logs de auditoría asociados han sido anonimizados.
4. El log de auditoría registra la acción de purga con el actor y la razón.
5. El evento `Identity.Purged` está en la cola para sistemas downstream.
6. La operación es irreversible.

**De Falla:**
1. El estado de la identidad no ha cambiado.
2. Los datos PII no han sido anonimizados.
3. No se han notificado a sistemas downstream.

---

## 3. Reglas de Negocio e Invariantes de Dominio (DDD)

| ID | Invariante / Regla | Descripción Técnica |
| :--- | :--- | :--- |
| **INV-25** | Precondición de Estado | Solo identidades en estado `DEACTIVATED` pueden ser purgadas. Lanza `InvalidTransitionException` si está en otro estado. |
| **INV-26** | Grace Period Obligatorio | Debe transcurrir el Grace Period (configurable por tenant, ej. 30 días) desde `deactivated_at` antes de permitir la purga. |
| **INV-27** | Legal Hold Priority | Si hay Legal Hold activo, la purga se bloquea independientemente de otros factores. Legal Hold tiene prioridad sobre GDPR Art. 17. |
| **INV-28** | Anonimización Completa | Todos los campos PII (email, phone, custom_attributes sensibles) deben ser anonimizados antes de marcar como `PURGED`. |
| **INV-29** | Preservación de Evidencia | Los logs de auditoría NO se eliminan, solo se anonimizan. Debe quedar registro de que la identidad existió y fue purgada. |
| **INV-30** | Irreversibilidad | Una vez purgada, la identidad NO puede ser reactivada ni restaurada. La operación es permanente. |
| **RN-01** | Idempotencia | Purgar una identidad ya purgada es una operación segura (no-op) que retorna éxito. |
| **RN-02** | Procesamiento Batch | El sistema debe soportar purgas masivas (hasta 1000 identidades por batch) para cumplimiento GDPR a escala. |
| **RN-03** | Notificación Downstream | Todos los sistemas downstream (KAGI, KIZUNA, KAGE) deben ser notificados vía Outbox Pattern para garantizar consistencia eventual. |
| **RN-04** | Auditoría de Purga | Cada purga debe generar un log de auditoría con: actor, razón, timestamp, y lista de datos anonimizados. |
| **RN-05** | Reintento Automático | Si la purga falla, el sistema reintenta automáticamente hasta 3 veces con backoff exponencial. |
| **RN-06** | Alerta de Fallo | Si la purga falla después de 3 reintentos, se genera alerta crítica al equipo de operaciones y compliance. |

---

## 4. Criterios de Aceptación (BDD / Gherkin)

### Escenario 4.1: Purga exitosa de identidad desactivada (GDPR Art. 17)
```gherkin
DADO que la identidad "user-uuid-123" tiene status "DEACTIVATED"
Y la identidad fue desactivada hace 45 días (Grace Period de 30 días superado)
Y el administrador "admin-001" tiene el permiso "identity:purge"
Y no hay Legal Hold activo sobre la identidad
CUANDO el administrador envía POST /identities/user-uuid-123/purge con:
  | Campo  | Valor                                    |
  |--------|------------------------------------------|
  | reason | "GDPR Art. 17 - Solicitud del usuario"   |
ENTONCES el sistema retorna HTTP 200 OK
Y la identidad ahora tiene status "PURGED"
Y el campo "purged_at" está establecido
Y los datos PII han sido anonimizados:
  | Campo  | Antes              | Después (Anonimizado)         |
  |--------|--------------------|-------------------------------|
  | email  | "juan@khcs.com"    | "a1b2c3d4e5f6..." (HMAC)     |
  | phone  | "+5491123456789"   | "f9e8d7c6b5a4..." (HMAC)     |
Y los logs de auditoría asociados han sido anonimizados
Y se ha generado un evento "Identity.Purged"
Y se ha creado un log de purga con:
  | Campo  | Valor                                    |
  |--------|------------------------------------------|
  | actor  | "admin-001"                              |
  | reason | "GDPR Art. 17 - Solicitud del usuario"   |
  | timestamp | "2024-03-15T10:00:00Z"               |
```

### Escenario 4.2: Purga bloqueada por Grace Period no expirado
```gherkin
DADO que la identidad "user-uuid-456" tiene status "DEACTIVATED"
Y la identidad fue desactivada hace solo 15 días (Grace Period de 30 días NO superado)
Y el administrador tiene el permiso "identity:purge"
CUANDO el administrador envía POST /identities/user-uuid-456/purge con:
  | Campo  | Valor                  |
  |--------|------------------------|
  | reason | "Solicitud de purga"   |
ENTONCES el sistema retorna HTTP 400 Bad Request
Y el mensaje es "Grace period has not expired. Purge available after 2024-03-30T10:00:00Z"
Y la identidad permanece en status "DEACTIVATED"
Y los datos PII NO han sido modificados
```

### Escenario 4.3: Purga bloqueada por Legal Hold
```gherkin
DADO que la identidad "user-uuid-789" tiene status "DEACTIVATED"
Y el Grace Period ha transcurrido
Y existe un Legal Hold activo sobre la identidad (caso judicial #2024-0042)
CUANDO el administrador envía POST /identities/user-uuid-789/purge con:
  | Campo  | Valor                  |
  |--------|------------------------|
  | reason | "Solicitud de purga"   |
ENTONCES el sistema retorna HTTP 403 Forbidden
Y el mensaje es "Identity is under legal hold (Case #2024-0042). Contact compliance officer."
Y la identidad permanece en status "DEACTIVATED"
Y se genera una alerta al oficial de cumplimiento
Y se registra un log de intento de purga bloqueado
```

### Escenario 4.4: Purga de identidad no desactivada
```gherkin
DADO que la identidad "user-uuid-321" tiene status "ACTIVE"
Y el administrador tiene el permiso "identity:purge"
CUANDO el administrador envía POST /identities/user-uuid-321/purge con:
  | Campo  | Valor                  |
  |--------|------------------------|
  | reason | "Solicitud de purga"   |
ENTONCES el sistema retorna HTTP 400 Bad Request
Y el mensaje es "Identity must be deactivated before purging. Use UC-03 first."
Y la identidad permanece en status "ACTIVE"
```

### Escenario 4.5: Idempotencia (Identidad ya purgada)
```gherkin
DADO que la identidad "user-uuid-654" ya tiene status "PURGED"
Y fue purgada el "2024-02-15T10:00:00Z"
CUANDO el administrador envía POST /identities/user-uuid-654/purge con:
  | Campo  | Valor                  |
  |--------|------------------------|
  | reason | "Reintento de purga"   |
ENTONCES el sistema retorna HTTP 200 OK
Y NO se generan eventos de dominio duplicados
Y NO se crea un nuevo log de auditoría de purga
Y los datos PII permanecen anonimizados (sin cambios)
```

### Escenario 4.6: Purga masiva (Batch Processing)
```gherkin
DADO que existen 500 identidades en estado "DEACTIVATED" con Grace Period expirado
Y 10 identidades en estado "DEACTIVATED" con Grace Period NO expirado
Y 5 identidades con Legal Hold activo
Y el administrador tiene el permiso "identity:purge"
CUANDO el administrador envía POST /identities/purge-batch con:
  | Campo        | Valor                              |
  |--------------|------------------------------------|
  | identity_ids | [lista de 515 IDs]                 |
  | reason       | "GDPR batch purge - Q1 2024"      |
ENTONCES el sistema retorna HTTP 202 Accepted con:
  | Campo    | Valor                              |
  |----------|------------------------------------|
  | batch_id | "batch-uuid-001"                   |
  | status   | "PROCESSING"                       |
  | total    | 515                                |
Y al consultar GET /purge-jobs/batch-uuid-001 después de completar:
  | Campo           | Valor                              |
  |-----------------|------------------------------------|
  | status          | "COMPLETED"                        |
  | total           | 515                                |
  | purged          | 500                                |
  | skipped         | 15                                 |
  | skipped_reasons | {"grace_period": 10, "legal_hold": 5} |
```

### Escenario 4.7: Fallo de anonimización (Rollback)
```gherkin
DADO que la identidad "user-uuid-987" tiene status "DEACTIVATED"
Y el Grace Period ha transcurrido
Y no hay Legal Hold activo
Y la base de datos falla durante la anonimización de PII
CUANDO el administrador envía POST /identities/user-uuid-987/purge
ENTONCES el sistema retorna HTTP 500 Internal Server Error
Y la transacción se revierte (Rollback)
Y la identidad permanece en status "DEACTIVATED"
Y los datos PII NO han sido modificados
Y se genera una alerta crítica al equipo de operaciones
Y el sistema reintenta automáticamente (hasta 3 veces)
```

### Escenario 4.8: Consulta de identidad purgada (Post-purga)
```gherkin
DADO que la identidad "user-uuid-123" fue purgada exitosamente
CUANDO un administrador consulta GET /identities/user-uuid-123
ENTONCES el sistema retorna HTTP 404 Not Found
Y el mensaje es "Identity not found"
Y NO se revela que la identidad existió y fue purgada
```

### Escenario 4.9: Consulta de identidad purgada con permisos de auditoría
```gherkin
DADO que la identidad "user-uuid-123" fue purgada exitosamente
Y el administrador tiene el permiso "audit:read"
CUANDO el administrador consulta GET /identities/user-uuid-123
ENTONCES el sistema retorna HTTP 200 OK con campos limitados:
  | Campo      | Valor                              |
  |------------|------------------------------------|
  | id         | "user-uuid-123"                    |
  | status     | "PURGED"                           |
  | purged_at  | "2024-03-15T10:00:00Z"             |
Y NO incluye datos PII (fueron anonimizados):
  | Campo Excluido      | Razón                              |
  |---------------------|------------------------------------|
  | profile.email       | Anonimizado (hash irreversible)    |
  | profile.phone       | Anonimizado (hash irreversible)    |
  | custom_attributes   | Eliminados permanentemente         |
```

### Escenario 4.10: Intento de reactivación post-purga
```gherkin
DADO que la identidad "user-uuid-123" fue purgada exitosamente
CUANDO un administrador intenta reactivar la identidad vía UC-02
ENTONCES el sistema retorna HTTP 404 Not Found
Y el mensaje es "Identity not found"
Y la identidad NO puede ser reactivada (operación irreversible)
```

---

## 5. Eventos de Dominio Generados

### Evento 1: `Identity.Purged`
```json
{
  "eventId": "uuid-v4",
  "eventType": "Identity.Purged",
  "timestamp": "2024-03-15T10:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-123",
    "previousStatus": "DEACTIVATED",
    "purgedBy": "admin-001",
    "reason": "GDPR Art. 17 - Solicitud del usuario",
    "anonymizedFields": [
      "profile.email",
      "profile.phone",
      "profile.custom_attributes.ssn"
    ],
    "auditLogsAnonymized": 12,
    "deactivatedAt": "2024-01-30T10:00:00Z",
    "purgedAt": "2024-03-15T10:00:00Z"
  },
  "metadata": {
    "traceId": "opentelemetry-trace-id",
    "priority": "HIGH",
    "irreversible": true
  }
}
```

**Consumidores:**
1. **KAGI (Auth):** Elimina todos los `RefreshTokens`, sesiones en Redis, y entradas en la lista negra. La identidad ya no puede autenticarse.
2. **KIZUNA (AuthZ):** Elimina la caché de autorización y las tuplas de relación asociadas a la identidad en KIZUNA Graph.
3. **KAGE (Audit):** Completa la anonimización de logs históricos y marca la identidad como purgada en el índice de auditoría.
4. **TORII (Webhooks):** Notifica a sistemas externos (ej. "Usuario eliminado permanentemente por GDPR").
5. **Sistema de Legal Hold:** Actualiza el registro para indicar que la purga se completó (o fue bloqueada).

### Evento 2: `Identity.PurgeBlocked` (Solo si Legal Hold)
```json
{
  "eventId": "uuid-v4",
  "eventType": "Identity.PurgeBlocked",
  "timestamp": "2024-03-15T10:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "user-uuid-789",
    "requestedBy": "admin-001",
    "reason": "GDPR Art. 17 - Solicitud del usuario",
    "blockedBy": "Legal Hold - Case #2024-0042",
    "blockedAt": "2024-03-15T10:00:00Z"
  },
  "metadata": {
    "priority": "CRITICAL",
    "notifyComplianceOfficer": true
  }
}
```

---

## 6. Consideraciones de Arquitectura y Seguridad

### 6.1. Anonimización vs. Eliminación Física

El UC-05 implementa una estrategia híbrida de **anonimización + eliminación selectiva**:

| Tipo de Dato | Estrategia | Justificación |
| :--- | :--- | :--- |
| **Email** | Anonimización (HMAC-SHA256 irreversible) | Se reemplaza con hash. Permite verificar unicidad futura sin exponer el dato. |
| **Teléfono** | Anonimización (HMAC-SHA256 irreversible) | Mismo tratamiento que email. |
| **Display Name** | Reemplazo con valor genérico ("Anonymized User") | No es PII sensible, pero se anonimiza por consistencia. |
| **Custom Attributes (sensibles)** | Eliminación física (DELETE) | Campos como SSN, tax_id se eliminan completamente. |
| **Custom Attributes (no sensibles)** | Conservación anonimizada | Campos como "department" se conservan para estadísticas. |
| **Audit Logs** | Anonimización de PII, preservación de estructura | Los logs NO se eliminan, solo se anonimizan los valores PII dentro del `snapshot_json`. |
| **RowVersion, timestamps** | Conservación | No son PII. Se conservan para integridad referencial. |

**Justificación:**
- **GDPR Art. 17:** Requiere eliminación de datos personales, pero permite retención para cumplimiento legal (Art. 17.3).
- **NIST SP 800-88:** Define tres niveles de sanitización: Clear, Purge, Destroy. La anonimización con HMAC equivale a "Purge" (no recuperable con técnicas de laboratorio).
- **Auditoría Forense:** Los logs de auditoría deben preservarse para investigaciones futuras, pero sin exponer PII.

### 6.2. Legal Hold (Retención Legal)

El Legal Hold es un mecanismo que **bloquea la purga** de identidades cuando existe una obligación legal de retener datos:

**Escenarios de Legal Hold:**
- Orden judicial que requiere preservación de evidencia.
- Investigación regulatoria en curso.
- Litigio laboral donde el empleado es parte.
- Auditoría fiscal que requiere retención de registros.

**Flujo de Legal Hold:**
1. El oficial de cumplimiento activa Legal Hold sobre una identidad vía API.
2. Se registra el Legal Hold con: caso judicial, fecha de inicio, fecha estimada de fin, autoridad que lo emitió.
3. Cualquier intento de purga (UC-05) verifica Legal Hold antes de proceder.
4. Si hay Legal Hold, la purga se bloquea y se notifica al oficial de cumplimiento.
5. Cuando el Legal Hold se levanta, la identidad vuelve a ser elegible para purga.

**Prioridad:** Legal Hold tiene prioridad sobre GDPR Art. 17. Esto está contemplado en el propio GDPR Art. 17.3(b): "El derecho al olvido no aplica cuando el tratamiento es necesario para el cumplimiento de una obligación legal."

### 6.3. Procesamiento Batch para Purgas Masivas

Para tenants grandes que necesitan purgar miles de identidades (ej. cierre de cuenta de tenant, migración de datos), el sistema soporta procesamiento batch:

**Diseño del Batch Processor:**
1. **Recepción:** El endpoint `POST /identities/purge-batch` recibe hasta 1000 IDs.
2. **Validación Previa:** Se valida cada identidad individualmente (estado, Grace Period, Legal Hold).
3. **Segmentación:** Las identidades válidas se agrupan en lotes de 100 para procesamiento.
4. **Ejecución:** Cada lote se procesa en una transacción independiente.
5. **Reporte:** Se genera un reporte con: purgadas exitosamente, saltadas (con razón), fallidas (con error).
6. **Reintento:** Las identidades fallidas se reintentan hasta 3 veces con backoff exponencial.

**Justificación:**
- **Atomicidad por Lote:** Si un lote falla, solo ese lote se revierte. Los lotes exitosos no se ven afectados.
- **Progreso Visible:** El administrador puede consultar el progreso en tiempo real.
- **Resiliencia:** Si el proceso se interrumpe, puede reanudarse desde el último lote exitoso.

### 6.4. Garantía de Irreversibilidad

La purga es **irreversible por diseño**:

1. **Anonimización Criptográfica:** Los hashes HMAC-SHA256 son irreversibles sin la clave secreta. Incluso con acceso a la base de datos, no se puede recuperar el email/teléfono original.
2. **Eliminación de Claves de Cifrado:** Opcionalmente, se puede destruir la clave de cifrado específica de la identidad (crypto-shredding), haciendo imposible descifrar cualquier dato residual.
3. **Estado Terminal:** La máquina de estados no permite transiciones desde `PURGED` a ningún otro estado.
4. **Sin Backups de PII:** Los backups de la base de datos se cifran y tienen política de retención. Después de la retención, se destruyen.

### 6.5. Seguridad y Prevención de Ataques

| Ataque | Mitigación |
| :--- | :--- |
| **Purga no autorizada** | Permiso granular `identity:purge` separado de `identity:update` y `identity:deactivate`. |
| **Purga prematura (antes de Grace Period)** | Validación estricta de `NOW() - deactivated_at > retention_days`. |
| **Purga durante Legal Hold** | Bloqueo automático con alerta al oficial de cumplimiento. |
| **Purga masiva maliciosa** | Rate limiting para batch (máximo 1 batch/hora por admin). |
| **Rollback de purga** | Transacción atómica. Si falla la anonimización, se hace rollback completo. |
| **Acceso a datos purgados** | Los datos PII se reemplazan con hashes irreversibles. No hay forma de recuperarlos. |
| **Evadir auditoría de purga** | El log de purga se escribe en la misma transacción que la anonimización. Si la transacción tiene éxito, el log existe. |

---

## 7. Contratos de API (OpenAPI 3.0 Snippets)

### 7.1. Endpoint: POST /identities/{id}/purge

**Request:**
```http
POST /v1/identities/user-uuid-123/purge HTTP/1.1
Host: api.shugo.io
Authorization: Bearer <JWT>
Content-Type: application/json

{
  "reason": "GDPR Art. 17 - Solicitud del usuario recibida el 2024-03-01",
  "gdpr_request_id": "gdpr-req-2024-0042",
  "confirm_irreversible": true
}
```

**Response (200 OK):**
```json
{
  "id": "user-uuid-123",
  "status": "PURGED",
  "purged_at": "2024-03-15T10:00:00Z",
  "anonymized_fields": [
    "profile.email",
    "profile.phone",
    "profile.custom_attributes.ssn"
  ],
  "audit_logs_anonymized": 12,
  "message": "Identity purged successfully. This action is irreversible."
}
```

**Response (400 Bad Request - Grace Period):**
```json
{
  "error": {
    "code": "GRACE_PERIOD_NOT_EXPIRED",
    "message": "Grace period has not expired. Purge available after 2024-03-30T10:00:00Z",
    "details": {
      "deactivated_at": "2024-02-28T10:00:00Z",
      "retention_days": 30,
      "purge_available_at": "2024-03-30T10:00:00Z"
    }
  }
}
```

**Response (403 Forbidden - Legal Hold):**
```json
{
  "error": {
    "code": "LEGAL_HOLD_ACTIVE",
    "message": "Identity is under legal hold. Contact compliance officer.",
    "details": {
      "legal_hold_id": "lh-2024-0042",
      "case_reference": "Case #2024-0042",
      "issued_by": "Tribunal Nacional",
      "issued_at": "2024-02-01T10:00:00Z"
    }
  }
}
```

### 7.2. Endpoint: POST /identities/purge-batch

**Request:**
```http
POST /v1/identities/purge-batch HTTP/1.1
Host: api.shugo.io
Authorization: Bearer <JWT>
Content-Type: application/json

{
  "identity_ids": ["user-uuid-001", "user-uuid-002", "..."],
  "reason": "GDPR batch purge - Q1 2024",
  "confirm_irreversible": true
}
```

**Response (202 Accepted):**
```json
{
  "batch_id": "batch-uuid-001",
  "status": "PROCESSING",
  "total": 515,
  "status_url": "/v1/purge-jobs/batch-uuid-001"
}
```

### 7.3. Endpoint: GET /purge-jobs/{batch_id}

**Response (200 OK - Completado):**
```json
{
  "batch_id": "batch-uuid-001",
  "status": "COMPLETED",
  "total": 515,
  "purged": 500,
  "skipped": 15,
  "failed": 0,
  "skipped_details": {
    "grace_period_not_expired": 10,
    "legal_hold_active": 5
  },
  "started_at": "2024-03-15T10:00:00Z",
  "completed_at": "2024-03-15T10:05:30Z",
  "duration_seconds": 330
}
```

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `02_Use_Case_Diagram.md` | Mostrará los actores y la relación con KAGI/KIZUNA/KAGE y el sistema de Legal Hold. |
| `03_Activity_Diagram.md` | Modelará el flujo de validación (estado, Grace Period, Legal Hold), anonimización, persistencia y publicación de eventos. |
| `04_Initial_Domain_Model.md` | Definirá el método `Purge()` en el Aggregate Root `Identity` y el Domain Service `IdentityPurgeService`. |
| `05_State_Machine_Diagram.md` | (UC-02) Ya define la transición `DEACTIVATED` → `PURGED` como estado terminal. |
| `10_UC_Specific_Data_Model.md` | Definirá la estrategia de anonimización a nivel de columna y la estructura de la tabla de Legal Hold. |
| `UC-03/01_Requirements_Specification.md` | Documento hermano que define la desactivación (Soft Delete) como precondición obligatoria para la purga. |

---

## 9. Glosario de Términos Específicos de UC-05

| Término | Definición |
| :--- | :--- |
| **Hard Delete** | Eliminación física e irreversible de datos de la base de datos. A diferencia del Soft Delete, los datos no pueden ser recuperados. |
| **Anonimización** | Proceso de transformar datos personales de forma irreversible, de modo que el titular ya no pueda ser identificado. En SHUGO se usa HMAC-SHA256. |
| **Crypto-Shredding** | Técnica de eliminación de datos mediante la destrucción de la clave de cifrado, haciendo imposible descifrar los datos residuales. |
| **Legal Hold** | Orden legal o regulatoria que obliga a preservar datos específicos, bloqueando su eliminación incluso si el titular lo solicita. |
| **Grace Period** | Período de tiempo entre la desactivación (UC-03) y la elegibilidad para purga (UC-05). Permite reactivación en caso de error. |
| **GDPR Art. 17** | Artículo del Reglamento General de Protección de Datos que establece el "Derecho al Olvido" (Right to Erasure). |
| **NIST SP 800-88** | Estándar del NIST para sanitización de medios, que define niveles de eliminación de datos (Clear, Purge, Destroy). |
| **Batch Processing** | Procesamiento de múltiples identidades en lotes, optimizado para purgas masivas de tenants grandes. |
| **Irreversibilidad** | Propiedad de la purga que garantiza que los datos anonimizados no pueden ser recuperados por ningún medio técnico. |
| **Audit Log Anonymization** | Proceso de reemplazar valores PII dentro de los logs de auditoría con hashes irreversibles, preservando la estructura del log. |

---
*Fin del Documento 01_Requirements_Specification para UC-05*