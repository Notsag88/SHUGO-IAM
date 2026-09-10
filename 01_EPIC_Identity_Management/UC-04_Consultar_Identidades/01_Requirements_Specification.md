# UC-04: Consultar Identidad - Especificación de Requisitos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-04 - Consultar Identidad |
| **Documento:** | 01_Requirements_Specification.md |
| **Metodología:** | ICONIX (Deep Dive) + DDD + CQRS + BDD + Performance Optimization |
| **Estándares:** | NIST SP 800-63A (Identity Verification), GDPR (Art. 15 - Right of Access), SOC 2 (CC6.1), ISO 27001 (A.9.1.2) |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 01/09/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 01/09/2026 | Gastón Ruiz | **Creación inicial.** Definición completa de la especificación analítica para la consulta de identidades. Establece el flujo de lectura optimizado con CQRS, los diferentes niveles de detalle según el actor (Admin vs Self-Service), la paginación y búsqueda avanzada, y las garantías de aislamiento multi-tenant. | Documento base para el paquete de diseño Tier 1 del UC-04. Es el caso de uso de lectura más consumido del sistema, utilizado por todos los demás UCs (UC-02 necesita RowVersion, UC-03 valida estado, UC-06 verifica existencia para autenticación). |

---

## 1. Introducción y Alcance

Este documento detalla la especificación de requisitos para el Caso de Uso **UC-04**, que representa el endpoint de **consulta y lectura de identidades** en el motor **KAGAMI** de SHUGO.

El UC-04 es responsable de:
1. **Proveer acceso de lectura optimizado** a las identidades del sistema mediante patrones CQRS (Command Query Responsibility Segregation).
2. **Diferenciar niveles de detalle** según el actor consultante (Admin ve todo, Usuario Self-Service ve solo su perfil enmascarado).
3. **Soportar búsquedas avanzadas** con filtros por estado, tipo, atributos custom, y paginación eficiente.
4. **Garantizar aislamiento multi-tenant** estricto en todas las consultas.
5. **Retornar el RowVersion** necesario para operaciones de actualización con OCC (UC-02).
6. **Manejar identidades en diferentes estados** (ACTIVE, DEACTIVATED, SUSPENDED, PURGED) con filtros apropiados.

### 1.1. Importancia Estratégica

El UC-04 es el **caso de uso más consumido** de SHUGO porque:

- **Es la base de todas las operaciones de escritura:** UC-02 (Actualizar) necesita obtener el RowVersion antes de modificar. UC-03 (Desactivar) valida el estado actual. UC-06 (Autenticar) verifica la existencia de la identidad.
- **Alto volumen de lecturas:** Por cada operación de escritura, hay ~10-50 lecturas (validaciones de permisos, renderizado de UI, auditoría).
- **Optimización crítica:** Las consultas deben ser extremadamente rápidas (< 50ms p95) para no degradar la experiencia de usuario.
- **Consumo transversal:** Es utilizado por la Admin UI, APIs de terceros, sistemas SCIM, y motores de autorización (KIZUNA).

### 1.2. Relación con Otros Casos de Uso

| UC | Relación | Descripción |
| :--- | :--- | :--- |
| **UC-01 (Provisionar)** | Consumidor | UC-04 permite verificar que una identidad fue creada exitosamente. |
| **UC-02 (Actualizar)** | Predecesor | UC-04 proporciona el RowVersion necesario para OCC en UC-02. |
| **UC-03 (Desactivar)** | Consumidor | UC-04 permite verificar el estado actual antes de desactivar. |
| **UC-05 (Purgar)** | Consumidor | UC-04 permite consultar identidades elegibles para purga (Grace Period). |
| **UC-06 (Autenticar)** | Consumidor | KAGI consulta UC-04 para verificar existencia y estado de la identidad durante el login. |
| **UC-19 (Definir Rol)** | Consumidor | MURA consulta UC-04 para asignar roles a identidades existentes. |
| **UC-30 (Evaluar Permiso)** | Consumidor | KIZUNA consulta UC-04 para obtener atributos de identidad durante evaluación de permisos. |

### 1.3. Objetivos de Rendimiento

| Métrica | Objetivo | Justificación |
| :--- | :--- | :--- |
| **Latencia p50** | < 20ms | La mayoría de consultas son simples (por ID) y deben ser casi instantáneas. |
| **Latencia p95** | < 50ms | Incluye consultas con filtros complejos y paginación. |
| **Latencia p99** | < 100ms | Casos extremos con búsquedas full-text en atributos custom. |
| **Throughput** | 500 req/seg | Suficiente para soportar 10x el volumen de escrituras. |
| **Disponibilidad** | 99.99% | Máximo 4.38 minutos de downtime/mes. |
| **Cache Hit Rate** | > 80% | La mayoría de consultas deben ser servidas desde caché (Redis). |
| **Tasa de error** | < 0.1% | Fallos solo en casos de infraestructura (DB down, Redis down). |

---

## 2. Descripción Analítica (ICONIX Clásico)

| Campo | Detalle |
| :--- | :--- |
| **ID Y NOMBRE:** | UC-04 - Consultar Identidad |
| **ESTADO:** | Preliminar |
| **DESCRIPCIÓN:** | Un actor (Administrador, Usuario Self-Service, o Sistema Externo) envía una solicitud de consulta a la API de KAGAMI especificando criterios de búsqueda (ID único, email, estado, tipo, atributos custom). El sistema valida el JWT del actor, extrae el tenant_id, determina el nivel de detalle permitido según el scope del actor (Admin vs User), ejecuta la consulta optimizada en PostgreSQL (o caché Redis si está disponible), aplica enmascaramiento de PII si el actor es Self-Service, y retorna la respuesta con los datos solicitados incluyendo el RowVersion para operaciones de actualización. |
| **ACTOR PRINCIPAL:** | Administrador de Tenant, Usuario Final (Self-Service), Sistema Externo (SCIM, KIZUNA, KAGI). |
| **ACTOR SECUNDARIO:** | Sistema SHUGO (KAGAMI Query Engine, Redis Cache, PostgreSQL Read Replicas). |
| **PRECONDICIONES:** | 1. El tenant debe estar registrado y activo en SHUGO.<br>2. El JWT presentado debe ser válido y no expirado.<br>3. El actor debe tener permisos de lectura (`identity:read` para Admin, `profile:read` para User).<br>4. La identidad consultada debe existir y no estar en estado `PURGED` (excepto para Admin con permisos especiales). |
| **PUNTOS DE EXTENSIÓN:** | Si la consulta es por email y no existe, se extiende a UC-01 (Provisionar) para crear la identidad. Si la identidad está `DEACTIVATED`, se extiende a UC-03 (Desactivar) para reactivarla. |
| **CONDICIÓN:** | La consulta debe retornar datos consistentes con el estado actual de la identidad, respetando el aislamiento multi-tenant y el nivel de detalle autorizado para el actor. |

### 2.1. Escenario Principal (Happy Path) - Consulta por ID (Admin)

1. El administrador envía una solicitud GET a `/identities/{id}` con el JWT en el header `Authorization`.
2. El API Gateway de SHUGO valida la firma del JWT usando las claves públicas del JWKS endpoint.
3. El API Gateway extrae el `tenant_id` y el `user_id` del claim del JWT y los inyecta en el contexto de la request.
4. El API Gateway enruta la solicitud al servicio KAGAMI (Query API).
5. El Query Handler recibe la solicitud y valida que el actor tenga el permiso `identity:read`.
6. El Query Handler consulta la caché Redis usando la clave `{tenant_id}:identity:{id}`.
7. Si hay cache hit, el Query Handler retorna la identidad desde caché (latencia < 5ms).
8. Si hay cache miss, el Query Handler consulta PostgreSQL usando el índice primario (`id`, `tenant_id`).
9. El Query Handler valida que la identidad pertenezca al `tenant_id` del actor (aislamiento multi-tenant).
10. El Query Handler valida que la identidad no esté en estado `PURGED` (a menos que el actor tenga permisos especiales de auditoría).
11. El Query Handler construye la respuesta incluyendo todos los campos (Admin ve todo: email, phone, custom_attributes, audit logs).
12. El Query Handler retorna la respuesta HTTP 200 OK con el JSON de la identidad incluyendo el `row_version`.
13. El Query Handler actualiza la caché Redis con TTL de 60 segundos para futuras consultas.
14. El administrador recibe la respuesta y puede proceder a actualizar (UC-02) o desactivar (UC-03) la identidad.

### 2.2. Escenario Alternativo - Consulta por ID (Usuario Self-Service)

1-5. (Idénticos al escenario principal hasta la validación de permisos)
6. El Query Handler detecta que el actor es un Usuario Self-Service (no Admin).
7. El Query Handler valida que el `user_id` del JWT coincida con el `id` de la identidad consultada (un usuario solo puede consultar su propia identidad).
8. El Query Handler consulta la identidad en PostgreSQL o caché.
9. El Query Handler aplica enmascaramiento de PII: el email se retorna parcialmente enmascarado (`j***@khcs.com`), el teléfono también (`+54 *** ** 6789`).
10. El Query Handler excluye campos sensibles: `audit_logs`, `deactivated_at`, `row_version` (no necesario para Self-Service).
11. El Query Handler retorna la respuesta HTTP 200 OK con el JSON enmascarado.
12. El usuario recibe su perfil parcial y puede actualizar campos permitidos (UC-02 con scope limitado).

### 2.3. Escenario Alternativo - Consulta con Filtros y Paginación (Admin)

1-5. (Idénticos al escenario principal hasta la validación de permisos)
6. El administrador envía una solicitud GET a `/identities` con query parameters: `?status=ACTIVE&type=USER&page=1&limit=50&search=juan`.
7. El Query Handler valida los parámetros de búsqueda (límites de paginación, filtros válidos).
8. El Query Handler construye una consulta SQL optimizada con filtros dinámicos:
   ```sql
   SELECT id, tenant_id, type, status, email_hash, created_at, row_version
   FROM identities
   WHERE tenant_id = @tenantId
     AND status = @status
     AND type = @type
     AND (display_name ILIKE @search OR email_hash = @searchHash)
   ORDER BY created_at DESC
   LIMIT @limit OFFSET @offset;
   ```
9. El Query Handler ejecuta la consulta en PostgreSQL (o en Read Replica si está configurado).
10. El Query Handler construye la respuesta con metadatos de paginación (`total_count`, `page`, `limit`, `has_next`).
11. El Query Handler retorna la respuesta HTTP 200 OK con el array de identidades y metadatos.

### 2.4. Escenario Alternativo - Consulta por Email (SCIM/HRIS)

1-5. (Idénticos al escenario principal hasta la validación de permisos)
6. El sistema SCIM envía una solicitud GET a `/identities?email=juan@khcs.com`.
7. El Query Handler calcula el hash HMAC-SHA256 del email proporcionado.
8. El Query Handler consulta PostgreSQL usando el índice `idx_identities_tenant_email_hash`.
9. El Query Handler retorna la identidad correspondiente (si existe).
10. El sistema SCIM recibe la identidad y puede sincronizar cambios (UC-02).

### 2.5. Flujos de Excepción

**[2.1] JWT Inválido o Expirado**
- 2.1.1 El API Gateway detecta que el JWT está expirado, tiene firma inválida, o falta el claim tenant_id.
- 2.1.2 El API Gateway retorna HTTP 401 Unauthorized con el mensaje "Invalid or expired token".
- 2.1.3 El actor recibe el error y solicita al usuario que inicie sesión nuevamente.
- 2.1.4 Fin del Caso de Uso.

**[5.1] Usuario sin Permisos de Lectura**
- 5.1.1 El Query Handler detecta que el usuario no tiene el permiso `identity:read` (Admin) o `profile:read` (User).
- 5.1.2 El Query Handler retorna HTTP 403 Forbidden con el mensaje "Insufficient permissions".
- 5.1.3 El actor recibe el error y contacta al administrador para obtener permisos.
- 5.1.4 Fin del Caso de Uso.

**[7.1] Self-Service Consulta Identidad de Otro Usuario**
- 7.1.1 El Query Handler detecta que el `user_id` del JWT no coincide con el `id` de la identidad consultada.
- 7.1.2 El Query Handler retorna HTTP 403 Forbidden con el mensaje "Cannot access other user's identity".
- 7.1.3 El usuario recibe el error y solo puede consultar su propia identidad.
- 7.1.4 Fin del Caso de Uso.

**[8.1] Identidad No Encontrada**
- 8.1.1 El Query Handler no encuentra la identidad con el ID proporcionado en el tenant del actor.
- 8.1.2 El Query Handler retorna HTTP 404 Not Found con el mensaje "Identity not found".
- 8.1.3 El actor recibe el error y verifica el ID o crea una nueva identidad (UC-01).
- 8.1.4 Fin del Caso de Uso.

**[9.1] Identidad en Estado PURGED**
- 9.1.1 El Query Handler encuentra la identidad pero está en estado `PURGED`.
- 9.1.2 Si el actor es Admin con permisos de auditoría, retorna la identidad con campos limitados.
- 9.1.3 Si el actor no tiene permisos especiales, retorna HTTP 404 Not Found (oculta la existencia de la identidad purgada).
- 9.1.4 Fin del Caso de Uso.

**[10.1] Error de Caché (Redis Down)**
- 10.1.1 El Query Handler intenta consultar Redis pero está caído.
- 10.1.2 El Query Handler hace fallback a PostgreSQL directamente.
- 10.1.3 La consulta se ejecuta en PostgreSQL con latencia ligeramente mayor (~20ms vs ~5ms).
- 10.1.4 El Query Handler retorna la respuesta normalmente (sin actualizar caché).
- 10.1.5 Fin del Caso de Uso.

**[11.1] Error de Base de Datos (PostgreSQL Down)**
- 11.1.1 El Query Handler intenta consultar PostgreSQL pero está caído.
- 11.1.2 El Query Handler intenta fallback a caché Redis (si está disponible).
- 11.1.3 Si hay cache hit, retorna datos potencialmente desactualizados (TTL de 60s).
- 11.1.4 Si hay cache miss, retorna HTTP 503 Service Unavailable con el mensaje "Database unavailable".
- 11.1.5 Fin del Caso de Uso.

### 2.6. Postcondiciones

**De Éxito:**
1. La identidad ha sido consultada exitosamente desde caché o PostgreSQL.
2. El nivel de detalle de la respuesta corresponde al scope del actor (Admin vs User).
3. El `row_version` está incluido en la respuesta (para Admin) para soportar OCC en UC-02.
4. La caché Redis ha sido actualizada (si había cache miss).
5. La respuesta HTTP 200 OK ha sido retornada al actor con los datos solicitados.

**De Falla:**
1. No se ha modificado ningún dato en la base de datos (operación de lectura).
2. No se ha actualizado la caché (si hubo error).
3. La respuesta HTTP 4xx/5xx ha sido retornada al actor con el mensaje de error apropiado.

---

## 3. Reglas de Negocio e Invariantes de Dominio (DDD)

| ID | Invariante / Regla | Descripción Técnica |
| :--- | :--- | :--- |
| **INV-19** | Tenant Isolation Estricto | Todas las consultas deben incluir `WHERE tenant_id = @actorTenantId`. Intentar consultar identidades de otro tenant retorna 404 (no 403, para no revelar existencia). |
| **INV-20** | Nivel de Detalle por Scope | Admin ve todos los campos (email, phone, custom_attributes, audit_logs). User Self-Service ve solo campos no sensibles con PII enmascarado. |
| **INV-21** | RowVersion para OCC | Todas las respuestas de Admin incluyen el `row_version` para soportar Control de Concurrencia Optimista en UC-02. |
| **INV-22** | Ocultamiento de Identidades PURGED | Las identidades en estado `PURGED` no son visibles para actores sin permisos especiales de auditoría. Retorna 404 en lugar de 403. |
| **INV-23** | Self-Service Restringido | Un usuario Self-Service solo puede consultar su propia identidad (`user_id == identity_id`). Intentar consultar otra identidad retorna 403. |
| **INV-24** | Paginación con Límites | Las consultas con filtros deben tener límites de paginación (máximo 100 resultados por página) para prevenir DoS. |
| **RN-01** | Cache-Aside Pattern | El sistema consulta caché primero. Si hay miss, consulta PostgreSQL y actualiza caché con TTL de 60s. |
| **RN-02** | Read Replicas para Escalabilidad | Las consultas de lectura pueden ser enrutadas a Read Replicas de PostgreSQL para distribuir carga. |
| **RN-03** | Enmascaramiento de PII para Self-Service | El email se retorna como `j***@khcs.com`, el teléfono como `+54 *** ** 6789`. Solo Admin ve PII en claro. |
| **RN-04** | Búsqueda por Email via Hash | Las búsquedas por email usan el hash HMAC-SHA256 (`email_hash`) en lugar del texto plano para mantener consistencia con el modelo de datos cifrado. |
| **RN-05** | Idempotencia de Lectura | Múltiples consultas idénticas retornan el mismo resultado (sin efectos secundarios). |
| **RN-06** | Filtros de Estado | Las consultas pueden filtrar por estado (`ACTIVE`, `DEACTIVATED`, `SUSPENDED`). Por defecto, solo se retornan identidades `ACTIVE` y `SUSPENDED` (no `DEACTIVATED` ni `PURGED`). |

---

## 4. Criterios de Aceptación (BDD / Gherkin)

### Escenario 4.1: Consulta por ID exitosa (Admin)
```gherkin
DADO que la identidad "user-uuid-123" existe en el tenant "khcs-001" con status "ACTIVE"
Y el usuario "admin-001" tiene el permiso "identity:read"
Y el JWT del admin es válido y contiene tenant_id="khcs-001"
CUANDO el administrador envía GET /identities/user-uuid-123
ENTONCES el sistema retorna HTTP 200 OK con:
  | Campo           | Valor                              |
  |-----------------|------------------------------------|
  | id              | "user-uuid-123"                    |
  | tenant_id       | "khcs-001"                         |
  | type            | "USER"                             |
  | status          | "ACTIVE"                           |
  | row_version     | "uuid-row-version"                 |
  | profile.email   | "juan@khcs.com" (en claro)         |
  | profile.phone   | "+5491123456789" (en claro)        |
  | profile.custom_attributes | {"department": "Vet"}    |
  | created_at      | "2024-01-30T10:00:00Z"             |
```

### Escenario 4.2: Consulta por ID (Usuario Self-Service con enmascaramiento)
```gherkin
DADO que la identidad "user-uuid-123" existe en el tenant "khcs-001"
Y el usuario "user-uuid-123" tiene el permiso "profile:read"
Y el JWT del usuario es válido y contiene user_id="user-uuid-123"
CUANDO el usuario envía GET /identities/user-uuid-123
ENTONCES el sistema retorna HTTP 200 OK con:
  | Campo           | Valor                              |
  |-----------------|------------------------------------|
  | id              | "user-uuid-123"                    |
  | type            | "USER"                             |
  | status          | "ACTIVE"                           |
  | profile.email   | "j***@khcs.com" (enmascarado)      |
  | profile.phone   | "+54 *** ** 6789" (enmascarado)    |
  | profile.display_name | "Juan Veterinario"            |
  | profile.custom_attributes | {"department": "Vet"}    |
Y NO incluye campos sensibles:
  | Campo Excluido  | Razón                              |
  |-----------------|------------------------------------|
  | row_version     | No necesario para Self-Service     |
  | audit_logs      | Solo visible para Admin            |
  | deactivated_at  | Solo visible para Admin            |
```

### Escenario 4.3: Self-Service intenta consultar identidad de otro usuario
```gherkin
DADO que la identidad "user-uuid-456" existe en el tenant "khcs-001"
Y el usuario "user-uuid-123" tiene el permiso "profile:read"
Y el JWT del usuario contiene user_id="user-uuid-123"
CUANDO el usuario envía GET /identities/user-uuid-456
ENTONCES el sistema retorna HTTP 403 Forbidden
Y el mensaje es "Cannot access other user's identity"
Y NO se revela información sobre la existencia de user-uuid-456
```

### Escenario 4.4: Consulta con filtros y paginación (Admin)
```gherkin
DADO que existen 150 identidades con status "ACTIVE" y type "USER" en el tenant "khcs-001"
Y el administrador tiene el permiso "identity:read"
CUANDO el administrador envía GET /identities?status=ACTIVE&type=USER&page=1&limit=50
ENTONCES el sistema retorna HTTP 200 OK con:
  | Campo           | Valor                              |
  |-----------------|------------------------------------|
  | data            | Array de 50 identidades            |
  | total_count     | 150                                |
  | page            | 1                                  |
  | limit           | 50                                 |
  | has_next        | true                               |
```

### Escenario 4.5: Identidad no encontrada
```gherkin
DADO que NO existe la identidad "user-uuid-999" en el tenant "khcs-001"
Y el administrador tiene el permiso "identity:read"
CUANDO el administrador envía GET /identities/user-uuid-999
ENTONCES el sistema retorna HTTP 404 Not Found
Y el mensaje es "Identity not found"
```

### Escenario 4.6: Identidad en estado PURGED (sin permisos de auditoría)
```gherkin
DADO que la identidad "user-uuid-789" existe en el tenant "khcs-001" con status "PURGED"
Y el administrador tiene el permiso "identity:read" pero NO "audit:read"
CUANDO el administrador envía GET /identities/user-uuid-789
ENTONCES el sistema retorna HTTP 404 Not Found
Y el mensaje es "Identity not found"
Y NO se revela que la identidad existe pero está purgada
```

### Escenario 4.7: Identidad en estado PURGED (con permisos de auditoría)
```gherkin
DADO que la identidad "user-uuid-789" existe en el tenant "khcs-001" con status "PURGED"
Y el administrador tiene el permiso "audit:read"
CUANDO el administrador envía GET /identities/user-uuid-789
ENTONCES el sistema retorna HTTP 200 OK con campos limitados:
  | Campo           | Valor                              |
  |-----------------|------------------------------------|
  | id              | "user-uuid-789"                    |
  | status          | "PURGED"                           |
  | purged_at       | "2024-02-15T10:00:00Z"             |
Y NO incluye campos sensibles:
  | Campo Excluido  | Razón                              |
  |-----------------|------------------------------------|
  | profile.email   | Ya fue anonimizado                 |
  | profile.phone   | Ya fue anonimizado                 |
  | custom_attributes | Ya fueron eliminados             |
```

### Escenario 4.8: Cache Hit (Redis disponible)
```gherkin
DADO que la identidad "user-uuid-123" está en caché Redis con TTL de 60s
Y el administrador envía GET /identities/user-uuid-123
CUANDO el Query Handler consulta Redis
ENTONCES el sistema retorna HTTP 200 OK en < 5ms
Y NO se consulta PostgreSQL
Y la respuesta incluye datos de caché (potencialmente desactualizados en ~60s)
```

### Escenario 4.9: Cache Miss (Redis disponible, datos en PostgreSQL)
```gherkin
DADO que la identidad "user-uuid-123" NO está en caché Redis
Y la identidad existe en PostgreSQL
Y el administrador envía GET /identities/user-uuid-123
CUANDO el Query Handler consulta Redis (miss) y luego PostgreSQL
ENTONCES el sistema retorna HTTP 200 OK en < 50ms
Y actualiza Redis con los datos consultados (TTL 60s)
```

### Escenario 4.10: Redis caído (Fallback a PostgreSQL)
```gherkin
DADO que Redis NO está disponible
Y la identidad "user-uuid-123" existe en PostgreSQL
Y el administrador envía GET /identities/user-uuid-123
CUANDO el Query Handler intenta consultar Redis (falla) y hace fallback a PostgreSQL
ENTONCES el sistema retorna HTTP 200 OK en < 100ms
Y NO actualiza caché (Redis está caído)
Y registra un warning en los logs: "Redis unavailable, falling back to PostgreSQL"
```

### Escenario 4.11: PostgreSQL caído (Fallback a caché si disponible)
```gherkin
DADO que PostgreSQL NO está disponible
Y la identidad "user-uuid-123" está en caché Redis (desactualizada)
Y el administrador envía GET /identities/user-uuid-123
CUANDO el Query Handler intenta consultar PostgreSQL (falla) y hace fallback a Redis
ENTONCES el sistema retorna HTTP 200 OK con datos de caché
Y incluye un header "X-Data-Source: cache" para indicar datos potencialmente desactualizados
Y registra un warning en los logs: "PostgreSQL unavailable, serving from cache"
```

### Escenario 4.12: PostgreSQL y Redis caídos (Service Unavailable)
```gherkin
DADO que PostgreSQL NO está disponible
Y Redis NO está disponible
Y el administrador envía GET /identities/user-uuid-123
CUANDO el Query Handler intenta consultar ambos (fallan)
ENTONCES el sistema retorna HTTP 503 Service Unavailable
Y el mensaje es "Database and cache unavailable. Please try again later."
```

### Escenario 4.13: Búsqueda por email (SCIM/HRIS)
```gherkin
DADO que la identidad con email "juan@khcs.com" existe en el tenant "khcs-001"
Y el sistema SCIM tiene el permiso "identity:read"
CUANDO el sistema SCIM envía GET /identities?email=juan@khcs.com
ENTONCES el sistema retorna HTTP 200 OK con la identidad correspondiente
Y la consulta usa el índice idx_identities_tenant_email_hash para optimización
```

### Escenario 4.14: Paginación con límite excedido
```gherkin
DADO que el administrador envía GET /identities?limit=500
CUANDO el Query Handler valida los parámetros
ENTONCES el sistema retorna HTTP 400 Bad Request
Y el mensaje es "Limit exceeds maximum allowed value (100)"
Y sugiere usar limit=100
```

---

## 5. Eventos de Dominio Generados

El UC-04 es una operación de **lectura pura** y **NO genera eventos de dominio**. No hay mutación de estado, por lo tanto no hay eventos que publicar.

**Nota:** Las consultas pueden generar **métricas de observabilidad** (OpenTelemetry) para monitorear rendimiento, pero no eventos de dominio.

---

## 6. Consideraciones de Arquitectura y Seguridad (Clean Architecture + CQRS)

### 6.1. Patrón CQRS (Command Query Responsibility Segregation)

El UC-04 implementa el lado **Query** de CQRS:
- **Separación de Responsabilidades:** Los handlers de lectura (`QueryHandler`) están separados de los handlers de escritura (`CommandHandler` de UC-01, UC-02, UC-03).
- **Optimización Independiente:** Las consultas pueden usar Read Replicas, caché Redis, e índices optimizados sin afectar las operaciones de escritura.
- **Escalabilidad Horizontal:** Los Query Handlers pueden escalar independientemente de los Command Handlers según la carga de lectura vs escritura.

### 6.2. Cache-Aside Pattern

El UC-04 implementa el patrón Cache-Aside para optimizar lecturas frecuentes:
1. **Consulta Caché:** El Query Handler consulta Redis primero.
2. **Cache Hit:** Si está en caché, retorna inmediatamente (< 5ms).
3. **Cache Miss:** Si no está en caché, consulta PostgreSQL (< 50ms).
4. **Actualización de Caché:** Tras consultar PostgreSQL, actualiza Redis con TTL de 60s.
5. **Invalidación de Caché:** Cuando UC-01, UC-02, o UC-03 modifican una identidad, invalidan la caché correspondiente.

**Justificación:**
- **Rendimiento:** 80%+ de las consultas son servidas desde caché, reduciendo la carga en PostgreSQL.
- **Consistencia Eventual:** Los datos en caché pueden estar desactualizados hasta 60 segundos, lo cual es aceptable para operaciones de lectura.

### 6.3. Multi-Tenancy y Aislamiento

Todas las consultas incluyen el `tenant_id` extraído del JWT:
- **Query Filters:** `WHERE tenant_id = @actorTenantId` en todas las consultas SQL.
- **Row-Level Security (RLS):** PostgreSQL RLS previene que un tenant acceda a identidades de otro tenant incluso si hay bugs en la aplicación.
- **Ocultamiento de Existencia:** Si una identidad no existe en el tenant del actor, retorna 404 (no 403) para no revelar que la identidad existe en otro tenant.

### 6.4. Seguridad y Prevención de Ataques

| Ataque | Mitigación |
| :--- | :--- |
| **Tenant Confusion** | Extracción de tenant_id desde JWT firmado (no desde headers), query filters globales, RLS |
| **Information Disclosure** | Self-Service solo ve su propia identidad, Admin ve todo pero con auditoría |
| **PII Leakage** | Enmascaramiento de PII para Self-Service (email, phone) |
| **DoS via Paginación** | Límite máximo de 100 resultados por página, rate limiting por tenant |
| **Cache Poisoning** | Validación de datos antes de almacenar en caché, TTL corto (60s) |
| **Enumeration Attack** | Retorna 404 genérico para identidades no encontradas o purgadas (sin permisos) |
| **SQL Injection** | Parameterized queries, ORM con sanitización automática |
| **Timing Attack** | Respuestas con latencia constante (no revelar si existe o no por tiempo de respuesta) |

### 6.5. Observabilidad y Métricas (OpenTelemetry)

| Métrica | Tipo | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- |
| `kagami.identity.query_total` | Counter | Total de consultas de identidad | `tenant_id`, `query_type` (by_id, by_email, filtered), `cache_hit` |
| `kagami.identity.query_latency` | Histogram | Latencia de consultas de identidad | `tenant_id`, `query_type`, `cache_hit`, `success` |
| `kagami.identity.cache_hit_rate` | Gauge | Tasa de aciertos de caché Redis | `tenant_id` |
| `kagami.identity.query_errors_total` | Counter | Total de errores en consultas | `tenant_id`, `error_type` (not_found, db_down, cache_down) |
| `kagami.identity.paginated_queries_total` | Counter | Total de consultas con paginación | `tenant_id`, `page_size` |

### 6.6. Resiliencia y Patrones

| Patrón | Implementación |
| :--- | :--- |
| **Cache-Aside** | Consulta caché primero, fallback a PostgreSQL, actualización de caché tras miss |
| **Circuit Breaker** | Si Redis o PostgreSQL fallan repetidamente, el Circuit Breaker se abre y retorna error inmediatamente |
| **Retry with Backoff** | Reintentos con backoff exponencial para consultas a PostgreSQL (fallos transitorios) |
| **Fallback** | Si Redis está caído, consulta PostgreSQL directamente. Si PostgreSQL está caído, sirve desde caché si está disponible. |
| **Timeout** | Timeout de 100ms para consultas a PostgreSQL, timeout de 10ms para consultas a Redis |
| **Bulkhead** | Separación de thread pools para Query API vs Command API |

---

## 7. Contratos de API (OpenAPI 3.0)

### 7.1. Endpoint: GET /identities/{id}

**Request:**
```http
GET /v1/identities/user-uuid-123 HTTP/1.1
Host: api.shugo.io
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
X-Tenant-ID: khcs-001
```

**Response (200 OK - Admin):**
```json
{
  "id": "user-uuid-123",
  "tenant_id": "khcs-001",
  "type": "USER",
  "status": "ACTIVE",
  "row_version": "uuid-row-version",
  "created_at": "2024-01-30T10:00:00Z",
  "updated_at": "2024-01-30T10:00:00Z",
  "profile": {
    "email": "juan@khcs.com",
    "phone": "+5491123456789",
    "display_name": "Juan Veterinario",
    "locale": "es-AR",
    "timezone": "America/Buenos_Aires",
    "custom_attributes": {
      "department": "Veterinaria",
      "clearance_level": 3
    }
  },
  "audit_logs": [
    {
      "action": "Identity.Provisioned",
      "actor_id": "admin-001",
      "timestamp": "2024-01-30T10:00:00Z",
      "reason": "Employee onboarding"
    }
  ]
}
```

**Response (200 OK - Self-Service):**
```json
{
  "id": "user-uuid-123",
  "type": "USER",
  "status": "ACTIVE",
  "created_at": "2024-01-30T10:00:00Z",
  "profile": {
    "email": "j***@khcs.com",
    "phone": "+54 *** ** 6789",
    "display_name": "Juan Veterinario",
    "locale": "es-AR",
    "timezone": "America/Buenos_Aires",
    "custom_attributes": {
      "department": "Veterinaria"
    }
  }
}
```

**Response (404 Not Found):**
```json
{
  "error": {
    "code": "IDENTITY_NOT_FOUND",
    "message": "Identity not found"
  }
}
```

**Response (403 Forbidden - Self-Service accessing other user):**
```json
{
  "error": {
    "code": "INSUFFICIENT_PERMISSIONS",
    "message": "Cannot access other user's identity"
  }
}
```

**Response (503 Service Unavailable):**
```json
{
  "error": {
    "code": "SERVICE_UNAVAILABLE",
    "message": "Database and cache unavailable. Please try again later."
  }
}
```

### 7.2. Endpoint: GET /identities (Con Filtros y Paginación)

**Request:**
```http
GET /v1/identities?status=ACTIVE&type=USER&page=1&limit=50&search=juan HTTP/1.1
Host: api.shugo.io
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
X-Tenant-ID: khcs-001
```

**Query Parameters:**
| Parámetro | Tipo | Requerido | Descripción | Ejemplo |
| :--- | :--- | :--- | :--- | :--- |
| `status` | string | No | Filtrar por estado (ACTIVE, DEACTIVATED, SUSPENDED) | `ACTIVE` |
| `type` | string | No | Filtrar por tipo (USER, SERVICE, DEVICE) | `USER` |
| `page` | integer | No | Número de página (default: 1) | `1` |
| `limit` | integer | No | Resultados por página (default: 20, max: 100) | `50` |
| `search` | string | No | Búsqueda por display_name o email | `juan` |
| `email` | string | No | Búsqueda exacta por email (SCIM) | `juan@khcs.com` |
| `sort` | string | No | Campo de ordenamiento (created_at, updated_at) | `created_at` |
| `order` | string | No | Dirección de ordenamiento (asc, desc) | `desc` |

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": "user-uuid-123",
      "tenant_id": "khcs-001",
      "type": "USER",
      "status": "ACTIVE",
      "row_version": "uuid-row-version-1",
      "created_at": "2024-01-30T10:00:00Z",
      "profile": {
        "email": "juan@khcs.com",
        "display_name": "Juan Veterinario"
      }
    },
    {
      "id": "user-uuid-456",
      "tenant_id": "khcs-001",
      "type": "USER",
      "status": "ACTIVE",
      "row_version": "uuid-row-version-2",
      "created_at": "2024-01-29T10:00:00Z",
      "profile": {
        "email": "maria@khcs.com",
        "display_name": "María Mozo"
      }
    }
  ],
  "pagination": {
    "total_count": 150,
    "page": 1,
    "limit": 50,
    "total_pages": 3,
    "has_next": true,
    "has_prev": false
  }
}
```

**Response (400 Bad Request - Limit excedido):**
```json
{
  "error": {
    "code": "INVALID_PAGINATION",
    "message": "Limit exceeds maximum allowed value (100)",
    "details": {
      "provided_limit": 500,
      "max_limit": 100
    }
  }
}
```

---

## 8. Consideraciones de Rendimiento y Escalabilidad

### 8.1. Estrategia de Caché

**Redis Cache Structure:**
```
Key: {tenant_id}:identity:{identity_id}
Value: JSON serializado de la identidad
TTL: 60 segundos
Eviction: LRU (Least Recently Used)
Max Memory: 2GB por instancia de Redis
```

**Cache Invalidation:**
- Cuando UC-01 crea una identidad: `SET {tenant_id}:identity:{id} {json} EX 60`
- Cuando UC-02 actualiza una identidad: `DEL {tenant_id}:identity:{id}` (invalidación perezosa)
- Cuando UC-03 desactiva una identidad: `DEL {tenant_id}:identity:{id}` (invalidación perezosa)

**Justificación:**
- **TTL Corto (60s):** Balance entre rendimiento y consistencia. Los datos pueden estar desactualizados hasta 60 segundos, lo cual es aceptable para la mayoría de casos de uso.
- **Invalidación Perezosa:** En lugar de actualizar caché en cada escritura, se invalida y se deja que la próxima consulta repueble la caché.

### 8.2. Optimizaciones de PostgreSQL

**Índices Optimizados:**
```sql
-- Índice primario compuesto (ya existe desde UC-01)
CREATE UNIQUE INDEX idx_identities_pkey ON identities(tenant_id, id);

-- Índice para búsquedas por email (hash HMAC-SHA256)
CREATE INDEX idx_identities_tenant_email_hash 
    ON identities(tenant_id, email_hash) 
    WHERE email_hash IS NOT NULL;

-- Índice para filtros por estado y tipo
CREATE INDEX idx_identities_tenant_status_type 
    ON identities(tenant_id, status, type);

-- Índice para ordenamiento por fecha de creación
CREATE INDEX idx_identities_tenant_created_at 
    ON identities(tenant_id, created_at DESC);

-- Índice para búsquedas full-text en display_name (opcional, para tenants grandes)
CREATE INDEX idx_identities_display_name_trgm 
    ON identities USING GIN (display_name gin_trgm_ops);
```

**Read Replicas:**
- Para tenants enterprise con >100k identidades, configurar Read Replicas de PostgreSQL.
- Las consultas de UC-04 pueden ser enrutadas a Read Replicas para distribuir carga.
- Las escrituras (UC-01, UC-02, UC-03) siempre van al Primary.

### 8.3. Escalabilidad Horizontal

- **Stateless Query Handlers:** Los Query Handlers son stateless, permitiendo escalado horizontal mediante Kubernetes HPA.
- **Particionamiento por Tenant:** Para tenants muy grandes (>1M identidades), se implementa sharding de la base de datos por `tenant_id`.
- **Redis Cluster:** Para alta disponibilidad de caché, se usa Redis Cluster con sharding automático.

### 8.4. Paginación Eficiente

**Offset-Based Pagination (para páginas iniciales):**
```sql
SELECT id, tenant_id, type, status, row_version, created_at
FROM identities
WHERE tenant_id = @tenantId
  AND status = @status
  AND type = @type
ORDER BY created_at DESC
LIMIT @limit OFFSET @offset;
```

**Cursor-Based Pagination (para páginas profundas, opcional):**
```sql
SELECT id, tenant_id, type, status, row_version, created_at
FROM identities
WHERE tenant_id = @tenantId
  AND status = @status
  AND type = @type
  AND created_at < @cursor_created_at
ORDER BY created_at DESC
LIMIT @limit;
```

**Justificación:**
- **Offset-Based:** Simple de implementar, pero ineficiente para páginas profundas (>1000).
- **Cursor-Based:** Más eficiente para páginas profundas, pero más complejo de implementar.

---

## 9. Trazabilidad con Otros Documentos del Paquete

| Documento | Relación |
| :--- | :--- |
| `02_Use_Case_Diagram.md` | Mostrará el límite del sistema KAGAMI para consultas de identidad, los actores (Admin, User, Sistemas externos), las relaciones <<include>> con caché y PostgreSQL, y las relaciones <<extend>> con enmascaramiento de PII. |
| `03_Activity_Diagram.md` | Visualizará el flujo de validación JWT, consulta de caché, fallback a PostgreSQL, enmascaramiento de PII, y construcción de respuesta. |
| `04_Initial_Domain_Model.md` | Definirá el Query Handler, el Cache Manager, y los Value Objects `IdentityQuery`, `IdentityQueryResult`, `PaginationMetadata`. |
| `06_Robustness_Diagram.md` | Mostrará la colaboración entre el Query Handler, el Cache Manager, el Identity Repository, y el PII Masker. |
| `07_Sequence_Diagram.md` | Detallará el orden temporal de las llamadas: validación JWT → consulta caché → consulta PostgreSQL → enmascaramiento PII → construcción de respuesta. |
| `08_Final_Class_Design.md` | Definirá las firmas exactas de los métodos: `QueryHandler.Execute()`, `CacheManager.Get()`, `IdentityRepository.GetById()`. |
| `KAGAMI_Engine_Architecture.md` | Documento global que define la arquitectura completa del motor KAGAMI (del cual UC-04 es el caso de uso de lectura fundamental). |
| `KHCS_SHUGO_Integration_Contract.md` | Define cómo KHCS consume la API de consulta de identidades para sincronización y verificación. |
| `UC-02/01_Requirements_Specification.md` | Documento hermano que define la actualización, que consume el RowVersion proporcionado por UC-04. |

---

## 10. Glosario de Términos Específicos de UC-04

| Término | Definición |
| :--- | :--- |
| **CQRS (Command Query Responsibility Segregation)** | Patrón arquitectónico que separa las operaciones de lectura (Queries) de las operaciones de escritura (Commands) para optimizar rendimiento y escalabilidad de forma independiente. |
| **Cache-Aside Pattern** | Patrón de caché donde la aplicación consulta la caché primero, y si hay miss, consulta la fuente de datos y actualiza la caché. |
| **RowVersion** | Valor (UUID o Timestamp) que cambia cada vez que la identidad se actualiza. Usado para Control de Concurrencia Optimista (OCC) en UC-02. |
| **Read Replica** | Copia de solo lectura de una base de datos PostgreSQL. Usada para distribuir carga de consultas y mejorar rendimiento de lectura. |
| **PII Masking** | Enmascaramiento de datos personales identificables (email, teléfono) para proteger privacidad. Ej: `juan@khcs.com` → `j***@khcs.com`. |
| **Tenant Isolation** | Garantía de que un tenant solo puede acceder a sus propios datos. Implementado mediante query filters (`WHERE tenant_id = ?`) y Row-Level Security (RLS). |
| **Pagination Metadata** | Metadatos incluidos en respuestas paginadas: `total_count`, `page`, `limit`, `total_pages`, `has_next`, `has_prev`. |
| **Cache Hit / Cache Miss** | Cache Hit: Los datos solicitados están en caché (respuesta rápida < 5ms). Cache Miss: Los datos no están en caché, se consulta PostgreSQL (respuesta más lenta < 50ms). |
| **Lazy Invalidation** | Estrategia de invalidación de caché donde se elimina la entrada de caché tras una escritura, y se deja que la próxima consulta repueble la caché. |
| **Cursor-Based Pagination** | Técnica de paginación que usa un cursor (timestamp o ID) en lugar de offset para navegar páginas profundas de forma eficiente. |

---
*Fin del Documento 01_Requirements_Specification para UC-04*