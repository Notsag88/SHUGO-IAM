# UC-01: Provisionar Nueva Identidad - Especificación de Requisitos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-01 - Provisionar Nueva Identidad |
| **Documento:** | 01_Requirements_Specification.md |
| **Metodología:** | ICONIX (Deep Dive) + DDD + BDD + NIST 800-63A |
| **Estándares:** | NIST SP 800-63A (Enrollment), GDPR (PII Protection), SCIM 2.0 (RFC 7643), ISO/IEC 27001 |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 30/08/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 30/08/2026 | Gastón Ruiz | **Creación inicial.** Definición completa de la especificación analítica para el caso de uso de provisionamiento de identidades. Establece el flujo de creación de identidades (usuarios, servicios, dispositivos), las invariantes de validación, el manejo de multi-tenancy, el cifrado de PII, la publicación de eventos de dominio, y los criterios de aceptación para el endpoint fundamental del motor KAGAMI. | Documento base para el paquete de diseño Tier 1 del UC-01. Este es el primer caso de uso del EPIC-01 (Identity Management) y representa la puerta de entrada al sistema: sin identidades, no hay autenticación (KAGI), ni autorización (KIZUNA), ni gestión de roles (MURA). |

---

## 1. Introducción y Alcance

Este documento detalla la especificación de requisitos para el Caso de Uso **UC-01**, que representa el endpoint fundamental de **provisionamiento de identidades** en el motor **KAGAMI** de SHUGO.

El UC-01 es responsable de:
1. **Recibir solicitudes de creación de identidad** desde administradores de tenant o sistemas externos (SCIM, HRIS).
2. **Validar que la identidad sea válida** según las reglas del tenant (formato, unicidad, atributos requeridos).
3. **Cifrar datos PII** (email, teléfono) antes de persistirlos.
4. **Persistir la identidad** con su perfil y atributos custom en PostgreSQL.
5. **Publicar eventos** para notificar a KAGI (crear credenciales), KIZUNA (inicializar grafo), y sistemas de auditoría.
6. **Retornar confirmación** de la creación exitosa con el ID generado.

### 1.1. Importancia Estratégica

El UC-01 es **el caso de uso más crítico de SHUGO** porque:

- **Es la base de todo:** Sin identidades, ningún otro motor puede funcionar. KAGI no puede autenticar, KIZUNA no puede autorizar, MURA no puede asignar roles.
- **Determina la calidad de los datos:** Una identidad mal provisionada (email inválido, duplicado, sin tenant_id) corrompe todo el sistema downstream.
- **Es el punto de entrada de datos:** Toda la información de usuarios, servicios y dispositivos entra por aquí.
- **Tiene implicaciones legales:** El manejo de PII debe cumplir con GDPR, CCPA, y normativas de privacidad.
- **Define la experiencia de onboarding:** La velocidad y confiabilidad del provisionamiento afecta directamente la productividad de los clientes.

### 1.2. Relación con Otros Casos de Uso

| UC | Relación | Descripción |
| :--- | :--- | :--- |
| **UC-02 (Actualizar Identidad)** | Complementario | Permite modificar atributos de una identidad existente. |
| **UC-03 (Desactivar Identidad)** | Complementario | Permite desactivar temporalmente una identidad. |
| **UC-04 (Consultar Identidad)** | Complementario | Permite consultar detalles de una identidad. |
| **UC-05 (Purgar Identidad)** | Complementario | Permite eliminar permanentemente una identidad (GDPR). |
| **UC-06 (Autenticar)** | Consumidor | KAGI consume identidades creadas por UC-01 para autenticación. |
| **UC-19 (Definir Rol)** | Consumidor | MURA asigna roles a identidades creadas por UC-01. |
| **UC-27 (Crear Tupla)** | Consumidor | KIZUNA crea relaciones donde el subject es una identidad de UC-01. |
| **UC-39 (SCIM Provisioning)** | Alternativa | TORII expone UC-01 vía protocolo SCIM 2.0. |

### 1.3. Objetivos de Rendimiento

| Métrica | Objetivo | Justificación |
| :--- | :--- | :--- |
| **Latencia p50** | < 50ms | La mayoría de provisionamientos son simples y deben ser rápidos. |
| **Latencia p95** | < 200ms | Incluye validaciones, cifrado de PII, y publicación de eventos. |
| **Latencia p99** | < 500ms | Casos extremos con validaciones complejas o eventos de dominio. |
| **Throughput** | 100 req/seg | Suficiente para onboarding masivo y sincronización SCIM. |
| **Disponibilidad** | 99.99% | Máximo 4.38 minutos de downtime/mes. |
| **Tasa de error** | < 0.5% | Fallos solo en casos de validación o errores de infraestructura. |
| **PII Encryption Time** | < 5ms | El cifrado de campos PII no debe impactar significativamente la latencia. |

---

## 2. Descripción Analítica (ICONIX Clásico)

| Campo | Detalle |
| :--- | :--- |
| **ID Y NOMBRE:** | UC-01 - Provisionar Nueva Identidad |
| **ESTADO:** | Preliminar |
| **DESCRIPCIÓN:** | Un administrador de tenant (o sistema externo vía SCIM) envía una solicitud de creación de identidad a la API de KAGAMI especificando el tipo de identidad (usuario, servicio, dispositivo), el tenant_id, y los atributos del perfil (email, nombre, teléfono, atributos custom). El sistema valida el JWT del administrador, extrae el tenant_id, valida que el email no exista ya en el tenant, valida el formato de los atributos, cifra los campos PII (email, teléfono), persiste la identidad y su perfil en PostgreSQL, publica un evento `Identity.Provisioned` en Kafka para notificar a KAGI (crear credenciales), KIZUNA (inicializar grafo vacío), y sistemas de auditoría, y retorna confirmación de la creación exitosa con el ID generado. |
| **ACTOR PRINCIPAL:** | Administrador de Tenant (humano o sistema automatizado vía SCIM/API) |
| **ACTOR SECUNDARIO:** | Sistema SHUGO (KAGAMI Engine, KAGE Audit, KAGI Credentials, KIZUNA Graph) |
| **PRECONDICIONES:** | 1. El tenant debe estar registrado y activo en SHUGO.<br>2. El JWT presentado debe ser válido, no expirado, y tener rol de administrador o permisos de `identity:create`.<br>3. El tipo de identidad debe ser válido (USER, SERVICE, DEVICE).<br>4. El email (si aplica) debe tener formato válido y no existir ya en el tenant.<br>5. Los atributos custom deben cumplir con el schema definido por el tenant. |
| **PUNTOS DE EXTENSIÓN:** | Si la identidad requiere credenciales inmediatas, se extiende a UC-06 (Autenticar) para generar password temporal o magic link. Si la identidad es un dispositivo, se extiende a UC-15 (Device Management) para registrar fingerprint. |
| **CONDICIÓN:** | La identidad creada debe ser consistente con las reglas del tenant y estar lista para ser autenticada y autorizada. |

### 2.1. Escenario Principal (Happy Path) - Creación Exitosa de Identidad

1. El administrador envía una solicitud POST a `/identities` con el JWT del admin en el header `Authorization` y el payload JSON con type, tenant_id, y profile (email, display_name, phone, custom_attributes).
2. El API Gateway de SHUGO valida la firma del JWT usando las claves públicas del JWKS endpoint.
3. El API Gateway extrae el `tenant_id` y el `user_id` del claim del JWT y los inyecta en el contexto de la request.
4. El API Gateway enruta la solicitud al servicio KAGAMI (Identity API).
5. El Handler de KAGAMI recibe la solicitud y valida el formato del payload (type válido, email formato RFC 5322, phone formato E.164).
6. El Handler valida que el email no exista ya en el tenant (unicidad).
7. El Handler valida que los atributos custom cumplan con el schema definido por el tenant (si existe).
8. El Handler cifra los campos PII (email, phone) usando AES-256-GCM con una clave maestra rotativa.
9. El Handler crea el objeto Identity con status `PROVISIONED` y el objeto Profile con los atributos cifrados.
10. El Handler persiste la identidad y el perfil en PostgreSQL en una transacción atómica.
11. El Handler publica un evento `Identity.Provisioned` en Kafka con los datos de la identidad (sin PII en claro).
12. El Handler retorna la respuesta HTTP 201 Created con el JSON de la identidad creada (id, type, status, created_at).
13. El administrador recibe la respuesta y puede proceder a asignar roles (UC-20) o credenciales (UC-06).

### 2.2. Escenario Alternativo - Creación de Identidad con Atributos Custom

1-7. (Idénticos al escenario principal hasta la validación de schema)
8. El Handler valida que los atributos custom cumplan con el JSON Schema definido por el tenant.
9-13. (Idénticos al escenario principal)

### 2.3. Escenario Alternativo - Creación de Identidad de Servicio (Service Account)

1. El administrador envía una solicitud POST a `/identities` con type=`SERVICE` y sin email (los servicios no tienen email).
2-4. (Idénticos al escenario principal)
5. El Handler valida que el type sea SERVICE y que no se requiera email.
6. El Handler valida que el nombre del servicio sea único en el tenant.
7-13. (Idénticos al escenario principal, pero sin cifrado de email)

### 2.4. Flujos de Excepción

**[2.1] JWT Inválido o Expirado**
- 2.1.1 El API Gateway detecta que el JWT está expirado, tiene firma inválida, o falta el claim tenant_id.
- 2.1.2 El API Gateway retorna HTTP 401 Unauthorized con el mensaje "Invalid or expired token".
- 2.1.3 El administrador recibe el error y solicita al usuario que inicie sesión nuevamente.
- 2.1.4 Fin del Caso de Uso.

**[3.1] Usuario sin Permisos de Administrador**
- 3.1.1 El API Gateway detecta que el usuario no tiene rol de administrador o el permiso `identity:create`.
- 3.1.2 El API Gateway retorna HTTP 403 Forbidden con el mensaje "Insufficient permissions. Admin role or identity:create permission required".
- 3.1.3 El administrador recibe el error y contacta al super administrador para obtener permisos.
- 3.1.4 Fin del Caso de Uso.

**[5.1] Payload Malformado**
- 5.1.1 El Handler detecta que el payload JSON no tiene los campos requeridos (type) o tienen formato inválido (email inválido, type desconocido).
- 5.1.2 El Handler retorna HTTP 400 Bad Request con el mensaje "Invalid payload format" y detalles de validación.
- 5.1.3 El administrador recibe el error y corrige la solicitud.
- 5.1.4 Fin del Caso de Uso.

**[6.1] Email Duplicado en el Tenant**
- 6.1.1 El Handler detecta que ya existe una identidad con el mismo email en el tenant.
- 6.1.2 El Handler retorna HTTP 409 Conflict con el mensaje "Email already exists in tenant".
- 6.1.3 El administrador recibe el error y decide si usar el email existente o crear la identidad con otro email.
- 6.1.4 Fin del Caso de Uso.

**[7.1] Atributos Custom Inválidos**
- 7.1.1 El Handler detecta que los atributos custom no cumplen con el JSON Schema definido por el tenant.
- 7.1.2 El Handler retorna HTTP 400 Bad Request con el mensaje "Custom attributes do not match schema" y detalles de los campos inválidos.
- 7.1.3 El administrador recibe el error y corrige los atributos custom.
- 7.1.4 Fin del Caso de Uso.

**[8.1] Error de Cifrado de PII**
- 8.1.1 El Handler intenta cifrar los campos PII pero ocurre un error (clave maestra no disponible, algoritmo fallido).
- 8.1.2 El Handler retorna HTTP 500 Internal Server Error con el mensaje "Failed to encrypt PII data".
- 8.1.3 El Handler publica un evento `Identity.ProvisioningFailed` en Kafka para alertar al equipo de operaciones.
- 8.1.4 El administrador recibe el error y reintenta la operación después de un backoff exponencial.
- 8.1.5 Fin del Caso de Uso.

**[10.1] Error de Persistencia**
- 10.1.1 El Handler intenta persistir la identidad en PostgreSQL pero ocurre un error (conexión perdida, constraint violation, deadlock).
- 10.1.2 El Handler retorna HTTP 500 Internal Server Error con el mensaje "Failed to create identity".
- 10.1.3 El Handler publica un evento `Identity.ProvisioningFailed` en Kafka para alertar al equipo de operaciones.
- 10.1.4 El administrador recibe el error y reintenta la operación después de un backoff exponencial.
- 10.1.5 Fin del Caso de Uso.

**[11.1] Error de Publicación de Evento**
- 11.1.1 El Handler intenta publicar el evento `Identity.Provisioned` en Kafka pero Kafka no está disponible.
- 11.1.2 El Handler registra un warning en los logs pero continúa con la operación (la identidad ya fue creada).
- 11.1.3 El evento se reintentará desde el outbox o dead-letter queue.
- 11.1.4 La operación continúa normalmente (la identidad fue creada exitosamente).
- 11.1.5 Fin del Caso de Uso.

### 2.5. Postcondiciones

**De Éxito:**
1. La identidad ha sido persistida en PostgreSQL con status `PROVISIONED` o `ACTIVE`.
2. El perfil ha sido persistido con los campos PII cifrados.
3. El evento `Identity.Provisioned` ha sido publicado en Kafka.
4. La respuesta HTTP 201 Created ha sido retornada al administrador.
5. KAGI ha recibido el evento y está listo para crear credenciales.
6. KIZUNA ha recibido el evento y ha inicializado el grafo de relaciones vacío para la identidad.

**De Falla:**
1. La identidad NO ha sido persistida en PostgreSQL.
2. No se ha modificado la caché.
3. El evento `Identity.ProvisioningFailed` ha sido publicado en Kafka (si aplica).
4. La respuesta HTTP 4xx/5xx ha sido retornada al administrador.

---

## 3. Reglas de Negocio e Invariantes de Dominio (DDD)

Estas reglas viven en la Capa de Dominio de KAGAMI y deben ser validadas antes de la persistencia. Su violación lanza `DomainException`.

| ID | Invariante / Regla | Descripción Técnica |
| :--- | :--- | :--- |
| **INV-01** | Tipo de Identidad Válido | El tipo de identidad debe ser USER, SERVICE, o DEVICE. Lanza `InvalidIdentityTypeException`. |
| **INV-02** | Formato de Email | El email debe cumplir con RFC 5322. Lanza `InvalidEmailFormatException`. |
| **INV-03** | Formato de Teléfono | El teléfono debe cumplir con E.164 (ej. +5491123456789). Lanza `InvalidPhoneFormatException`. |
| **INV-04** | Tenant Isolation | La identidad solo puede crearse para el tenant_id extraído del JWT. Intentar crear identidades para otro tenant lanza `TenantIsolationViolationException`. |
| **INV-05** | Unicidad de Email | No puede existir otra identidad con el mismo email en el mismo tenant. Lanza `DuplicateEmailException`. |
| **INV-06** | Unicidad de Nombre de Servicio | Para identidades de tipo SERVICE, el nombre debe ser único en el tenant. Lanza `DuplicateServiceNameException`. |
| **INV-07** | Schema de Atributos Custom | Los atributos custom deben cumplir con el JSON Schema definido por el tenant. Lanza `InvalidCustomAttributesException`. |
| **INV-08** | PII Encryption | Los campos email y phone deben cifrarse antes de persistir. Si el cifrado falla, no se persiste. Lanza `PIIEncryptionException`. |
| **RN-01** | Idempotencia | La API es idempotente: múltiples llamadas con los mismos parámetros retornan el mismo resultado (la identidad se crea una sola vez). |
| **RN-02** | Audit Trail | Toda creación de identidad debe ser registrada en el log de auditoría (KAGE) vía evento de Kafka. |
| **RN-03** | Atomicity | La creación de la identidad y su perfil debe ser atómica (si una falla, todo falla). |
| **RN-04** | Eventual Consistency | La publicación de eventos puede fallar sin afectar la creación de la identidad (consistencia eventual). |
| **RN-05** | Default Status | Toda identidad nueva se crea con status `PROVISIONED` hasta que se verifique el email o se apruebe manualmente. |
| **RN-06** | Immutable Fields | Los campos `id`, `tenant_id`, `type`, y `created_at` son inmutables una vez creados. |

---

## 4. Criterios de Aceptación (BDD / Gherkin)

### Escenario 4.1: Creación exitosa de identidad de usuario (Happy Path)
```gherkin
DADO que el tenant "khcs-001" está registrado y activo en SHUGO
Y el usuario "user:admin-001" tiene rol de administrador en el tenant
Y NO existe una identidad con email "juan@khcs.com" en el tenant
Y el JWT del admin es válido y contiene tenant_id="khcs-001" y role="admin"
CUANDO el administrador envía POST /identities con:
  | Campo           | Valor                              |
  |-----------------|------------------------------------|
  | type            | "USER"                             |
  | profile.email   | "juan@khcs.com"                    |
  | profile.display_name | "Juan Veterinario"            |
  | profile.phone   | "+5491123456789"                   |
  | profile.custom_attributes | {"department": "Veterinaria", "clearance": 3} |
ENTONCES el sistema retorna HTTP 201 Created con:
  | Campo           | Valor                              |
  |-----------------|------------------------------------|
  | id              | "uuid-generado"                    |
  | type            | "USER"                             |
  | status          | "PROVISIONED"                      |
  | created_at      | "2024-01-30T10:00:00Z"             |
Y la identidad es persistida en PostgreSQL
Y el perfil es persistido con email y phone cifrados
Y el evento "Identity.Provisioned" es publicado en Kafka
```

### Escenario 4.2: Creación de identidad de servicio (Service Account)
```gherkin
DADO que el tenant "khcs-001" está registrado y activo en SHUGO
Y el usuario "user:admin-001" tiene rol de administrador
Y NO existe un servicio con name "payment-api" en el tenant
CUANDO el administrador envía POST /identities con:
  | Campo           | Valor                              |
  |-----------------|------------------------------------|
  | type            | "SERVICE"                          |
  | profile.name    | "payment-api"                      |
  | profile.description | "API de pagos"                |
ENTONCES el sistema retorna HTTP 201 Created
Y la identidad es persistida con type="SERVICE"
Y NO se requiere email ni phone
```

### Escenario 4.3: Creación de identidad de dispositivo (Device)
```gherkin
DADO que el tenant "khcs-001" está registrado y activo en SHUGO
Y el usuario "user:admin-001" tiene rol de administrador
CUANDO el administrador envía POST /identities con:
  | Campo           | Valor                              |
  |-----------------|------------------------------------|
  | type            | "DEVICE"                           |
  | profile.name    | "laptop-juan-001"                  |
  | profile.custom_attributes | {"os": "macOS", "model": "MacBook Pro"} |
ENTONCES el sistema retorna HTTP 201 Created
Y la identidad es persistida con type="DEVICE"
```

### Escenario 4.4: Email duplicado en el tenant
```gherkin
DADO que ya existe una identidad con email "juan@khcs.com" en el tenant "khcs-001"
CUANDO el administrador envía POST /identities con email "juan@khcs.com"
ENTONCES el sistema retorna HTTP 409 Conflict
Y el mensaje es "Email already exists in tenant"
Y NO se crea una nueva identidad
Y NO se publica evento en Kafka
```

### Escenario 4.5: JWT inválido o expirado
```gherkin
DADO que el administrador envía POST /identities con un JWT expirado
CUANDO el API Gateway valida el JWT
ENTONCES el sistema retorna HTTP 401 Unauthorized
Y el mensaje es "Invalid or expired token"
Y NO se crea ninguna identidad
```

### Escenario 4.6: Usuario sin permisos de administrador
```gherkin
DADO que el usuario "user:veterinario-juan" NO tiene rol de administrador
Y NO tiene el permiso "identity:create"
CUANDO el usuario envía POST /identities con datos válidos
ENTONCES el sistema retorna HTTP 403 Forbidden
Y el mensaje es "Insufficient permissions. Admin role or identity:create permission required"
Y NO se crea ninguna identidad
```

### Escenario 4.7: Payload malformado (email inválido)
```gherkin
DADO que el administrador envía POST /identities con:
  | Campo           | Valor                              |
  |-----------------|------------------------------------|
  | type            | "USER"                             |
  | profile.email   | "juan-invalido" (sin @ ni dominio) |
CUANDO el Handler valida el formato del email
ENTONCES el sistema retorna HTTP 400 Bad Request
Y el mensaje es "Invalid email format"
Y los detalles indican que el email debe cumplir con RFC 5322
```

### Escenario 4.8: Atributos custom inválidos
```gherkin
DADO que el tenant "khcs-001" tiene un schema de atributos custom que requiere "department" como string
Y el administrador envía POST /identities con:
  | Campo           | Valor                              |
  |-----------------|------------------------------------|
  | type            | "USER"                             |
  | profile.email   | "juan@khcs.com"                    |
  | profile.custom_attributes | {"department": 123} (número en lugar de string) |
CUANDO el Handler valida los atributos custom contra el schema
ENTONCES el sistema retorna HTTP 400 Bad Request
Y el mensaje es "Custom attributes do not match schema"
Y los detalles indican que "department" debe ser string, no number
```

### Escenario 4.9: Error de cifrado de PII
```gherkin
DADO que el servicio de cifrado de PII no está disponible (clave maestra no accesible)
CUANDO el administrador envía POST /identities con datos válidos
ENTONCES el sistema retorna HTTP 500 Internal Server Error
Y el mensaje es "Failed to encrypt PII data"
Y el evento "Identity.ProvisioningFailed" es publicado en Kafka
Y NO se persiste la identidad
```

### Escenario 4.10: Error de persistencia en PostgreSQL
```gherkin
DADO que PostgreSQL no está disponible
CUANDO el administrador envía POST /identities con datos válidos
ENTONCES el sistema retorna HTTP 500 Internal Server Error
Y el mensaje es "Failed to create identity"
Y el evento "Identity.ProvisioningFailed" es publicado en Kafka
Y NO se persiste la identidad
```

### Escenario 4.11: Error de publicación de evento (Kafka caído)
```gherkin
DADO que Kafka no está disponible
CUANDO el administrador envía POST /identities con datos válidos
ENTONCES el sistema retorna HTTP 201 Created (la identidad se crea exitosamente)
Y se registra un warning en los logs: "Failed to publish Identity.Provisioned event"
Y el evento se reintentará desde el outbox o dead-letter queue
Y la identidad es persistida en PostgreSQL
```

### Escenario 4.12: Creación de identidad con atributos custom válidos
```gherkin
DADO que el tenant "khcs-001" tiene un schema de atributos custom definido
Y el administrador envía POST /identities con atributos custom que cumplen el schema
CUANDO el Handler valida los atributos custom
ENTONCES el sistema retorna HTTP 201 Created
Y los atributos custom se persisten en el perfil
```

---

## 5. Eventos de Dominio Generados

El UC-01 genera eventos de dominio para notificar a otros componentes del sistema:

### Evento 1: `Identity.Provisioned` (Creación Exitosa)
```json
{
  "eventId": "uuid-v4",
  "eventType": "Identity.Provisioned",
  "timestamp": "2024-01-30T10:00:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "identityId": "uuid-v4",
    "type": "USER",
    "status": "PROVISIONED",
    "email": "juan@khcs.com",
    "displayName": "Juan Veterinario",
    "createdAt": "2024-01-30T10:00:00Z",
    "createdBy": "user:admin-001"
  },
  "metadata": {
    "traceId": "opentelemetry-trace-id",
    "correlationId": "uuid-v4"
  }
}
```

**Consumidores:**
1. **KAGI (Credentials):** Crea credenciales temporales o envía magic link.
2. **KIZUNA (Graph):** Inicializa grafo de relaciones vacío para la identidad.
3. **KAGE (Audit):** Registra el evento en el log de auditoría.
4. **TORII (Webhooks):** Notifica a aplicaciones cliente suscritas.

### Evento 2: `Identity.ProvisioningFailed` (Error de Creación)
```json
{
  "eventId": "uuid-v4",
  "eventType": "Identity.ProvisioningFailed",
  "timestamp": "2024-01-30T10:05:00Z",
  "tenantId": "khcs-001",
  "payload": {
    "type": "USER",
    "email": "juan@khcs.com",
    "errorType": "PIIEncryptionException",
    "errorMessage": "Failed to encrypt email: master key unavailable",
    "createdBy": "user:admin-001"
  },
  "metadata": {
    "traceId": "opentelemetry-trace-id",
    "severity": "error"
  }
}
```

**Consumidores:**
1. **KAGE (Alerts):** Genera alerta para el equipo de operaciones.
2. **TORII (Webhooks):** Notifica fallos a aplicaciones cliente.

---

## 6. Consideraciones de Arquitectura y Seguridad (Clean Architecture)

### 6.1. Ubicación en Clean Architecture
- **Capa de Dominio (KAGAMI):** Contiene el `IdentityValidator` (servicio de dominio que valida identidades), el `PIIEncryptor` (servicio de cifrado), y los Value Objects (`CreateIdentityRequest`, `CreateIdentityResult`, `Identity`, `Profile`).
- **Capa de Aplicación:** Contiene el `ProvisionIdentityHandler` (Command Handler que orquesta la creación), y los contratos de entrada/salida (`ProvisionIdentityCommand`, `CreateIdentityResult`).
- **Capa de Infraestructura:** Implementa los repositorios (`IdentityRepository` para PostgreSQL), el cliente de cifrado (AWS KMS o HashiCorp Vault), y el publisher de Kafka (eventos de dominio).

### 6.2. Multi-Tenancy y Aislamiento
- **Tenant Isolation:** Cada identidad incluye el `tenant_id` extraído del JWT. Esto se garantiza mediante:
  1. **Extracción del JWT:** El `tenant_id` se extrae del claim del JWT (no desde headers).
  2. **Query Filters en EF Core:** Todas las queries a PostgreSQL incluyen automáticamente `WHERE tenant_id = @currentTenantId`.
  3. **Validación de Unicidad:** La validación de email único incluye el `tenant_id` para prevenir colisiones entre tenants.
  4. **Row-Level Security (RLS):** PostgreSQL RLS previene que un tenant acceda a identidades de otro tenant.

### 6.3. Seguridad y Prevención de Ataques

| Ataque | Mitigación |
| :--- | :--- |
| **Identidad Maliciosa** | Validación estricta de formato (email RFC 5322, phone E.164) y schema de atributos custom |
| **Tenant Confusion** | Extracción de tenant_id desde JWT firmado (no desde header), query filters globales, RLS |
| **Duplicate Email Abuse** | Constraint UNIQUE en PostgreSQL + validación previa en el Handler |
| **PII Leakage** | Cifrado AES-256-GCM de email y phone antes de persistir, claves rotativas |
| **Replay Attacks** | Idempotencia: múltiples llamadas con mismos parámetros retornan el mismo resultado (HTTP 409) |
| **Privilege Escalation** | Validación de rol admin o permiso `identity:create` en el JWT |
| **Information Disclosure** | Los errores de validación no revelan información sensible sobre la estructura interna |
| **Brute Force de Emails** | Rate limiting por IP y tenant, CAPTCHA después de 5 intentos |

### 6.4. Observabilidad y Métricas (OpenTelemetry)

| Métrica | Tipo | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- |
| `kagami.identity.provisioned` | Counter | Total de identidades creadas | `tenant_id`, `type`, `status` |
| `kagami.identity.provisioning_failed` | Counter | Total de fallos de provisionamiento | `tenant_id`, `error_type` |
| `kagami.identity.latency` | Histogram | Latencia de provisionamiento en ms | `tenant_id`, `success` |
| `kagami.identity.pii_encryption_time` | Histogram | Tiempo de cifrado de PII en ms | `tenant_id` |
| `kagami.identity.duplicate_email_attempts` | Counter | Intentos de crear emails duplicados | `tenant_id` |

### 6.5. Resiliencia y Patrones

| Patrón | Implementación |
| :--- | :--- |
| **Circuit Breaker** | Si PostgreSQL o Kafka fallan repetidamente, el Circuit Breaker se abre y retorna error inmediatamente |
| **Retry with Backoff** | No aplica para creación de identidades (no es idempotente si hay error de persistencia) |
| **Timeout** | Timeout de 5s para queries a PostgreSQL, timeout de 1s para cifrado de PII |
| **Bulkhead** | Separación de thread pools para Provisioning API vs Query API |
| **Fallback** | Si Kafka está caído, la identidad se crea pero el evento se reintentará desde el outbox |
| **Compensation** | Si la publicación de evento falla después de crear la identidad, se publica a un dead-letter queue para retry posterior |

---

## 7. Contratos de API (OpenAPI 3.0)

### 7.1. Endpoint: POST /identities

**Request:**
```http
POST /v1/identities HTTP/1.1
Host: api.shugo.io
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
X-Tenant-ID: khcs-001
Content-Type: application/json

{
  "type": "USER",
  "profile": {
    "email": "juan@khcs.com",
    "display_name": "Juan Veterinario",
    "phone": "+5491123456789",
    "locale": "es-AR",
    "timezone": "America/Buenos_Aires",
    "custom_attributes": {
      "department": "Veterinaria",
      "clearance_level": 3,
      "manager_id": "user:admin-001"
    }
  }
}
```

**Response (201 Created):**
```json
{
  "id": "uuid-v4",
  "tenant_id": "khcs-001",
  "type": "USER",
  "status": "PROVISIONED",
  "created_at": "2024-01-30T10:00:00Z",
  "created_by": "user:admin-001",
  "profile": {
    "email": "juan@khcs.com",
    "display_name": "Juan Veterinario",
    "phone": "+5491123456789",
    "locale": "es-AR",
    "timezone": "America/Buenos_Aires",
    "custom_attributes": {
      "department": "Veterinaria",
      "clearance_level": 3,
      "manager_id": "user:admin-001"
    }
  }
}
```

**Response (400 Bad Request):**
```json
{
  "error": {
    "code": "INVALID_EMAIL_FORMAT",
    "message": "Invalid email format",
    "details": {
      "email": "juan-invalido",
      "expected_format": "RFC 5322"
    }
  }
}
```

**Response (401 Unauthorized):**
```json
{
  "error": {
    "code": "INVALID_TOKEN",
    "message": "Invalid or expired token"
  }
}
```

**Response (403 Forbidden):**
```json
{
  "error": {
    "code": "INSUFFICIENT_PERMISSIONS",
    "message": "Insufficient permissions. Admin role or identity:create permission required"
  }
}
```

**Response (409 Conflict):**
```json
{
  "error": {
    "code": "DUPLICATE_EMAIL",
    "message": "Email already exists in tenant",
    "details": {
      "email": "juan@khcs.com",
      "tenant_id": "khcs-001"
    }
  }
}
```

**Response (500 Internal Server Error):**
```json
{
  "error": {
    "code": "PII_ENCRYPTION_FAILED",
    "message": "Failed to encrypt PII data",
    "details": {
      "error_type": "MasterKeyUnavailableException",
      "trace_id": "abc123def456"
    }
  }
}
```

### 7.2. Endpoint: POST /identities/bulk (Creación Masiva)

**Request:**
```http
POST /v1/identities/bulk HTTP/1.1
Host: api.shugo.io
Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
X-Tenant-ID: khcs-001
Content-Type: application/json

{
  "identities": [
    {
      "type": "USER",
      "profile": {
        "email": "juan@khcs.com",
        "display_name": "Juan Veterinario"
      }
    },
    {
      "type": "USER",
      "profile": {
        "email": "maria@khcs.com",
        "display_name": "María Mozo"
      }
    },
    {
      "type": "SERVICE",
      "profile": {
        "name": "payment-api",
        "description": "API de pagos"
      }
    }
  ]
}
```

**Response (201 Created):**
```json
{
  "created_identities": [
    {
      "id": "uuid-v4-1",
      "type": "USER",
      "status": "PROVISIONED",
      "created_at": "2024-01-30T10:00:00Z"
    },
    {
      "id": "uuid-v4-2",
      "type": "USER",
      "status": "PROVISIONED",
      "created_at": "2024-01-30T10:00:01Z"
    },
    {
      "id": "uuid-v4-3",
      "type": "SERVICE",
      "status": "ACTIVE",
      "created_at": "2024-01-30T10:00:02Z"
    }
  ],
  "total_created": 3,
  "total_failed": 0,
  "errors": []
}
```

**Response (207 Multi-Status - Algunos fallos):**
```json
{
  "created_identities": [
    {
      "id": "uuid-v4-1",
      "type": "USER",
      "status": "PROVISIONED",
      "created_at": "2024-01-30T10:00:00Z"
    }
  ],
  "total_created": 1,
  "total_failed": 2,
  "errors": [
    {
      "index": 1,
      "code": "DUPLICATE_EMAIL",
      "message": "Email already exists in tenant",
      "identity": {
        "type": "USER",
        "profile": {
          "email": "maria@khcs.com"
        }
      }
    },
    {
      "index": 2,
      "code": "INVALID_SERVICE_NAME",
      "message": "Service name already exists in tenant",
      "identity": {
        "type": "SERVICE",
        "profile": {
          "name": "payment-api"
        }
      }
    }
  ]
}
```

---

## 8. Consideraciones de Rendimiento y Escalabilidad

### 8.1. Estrategia de Cifrado de PII

Los campos PII (email, phone) se cifran usando **AES-256-GCM** con una clave maestra rotativa:

1. **Clave Maestra:** Almacenada en AWS KMS o HashiCorp Vault, rotada cada 90 días.
2. **Clave de Datos:** Generada aleatoriamente para cada identidad, cifrada con la clave maestra, y almacenada junto con los datos cifrados.
3. **Algoritmo:** AES-256-GCM proporciona confidencialidad y autenticación (AEAD).
4. **Performance:** El cifrado toma < 5ms por campo, impactando mínimamente la latencia total.

**Justificación:**
- **Seguridad:** Si la base de datos es comprometida, los atacantes no pueden leer los PII sin la clave maestra.
- **Compliance:** Cumple con GDPR Art. 32 (Security of Processing) y ISO 27001 A.10.1 (Cryptographic Controls).
- **Performance:** AES-NI instructions en CPUs modernos hacen el cifrado muy rápido.

### 8.2. Optimizaciones de Persistencia

1. **Índices Optimizados en PostgreSQL:**
   ```sql
   CREATE UNIQUE INDEX idx_identities_tenant_email 
   ON identities(tenant_id, email) 
   WHERE email IS NOT NULL;
   
   CREATE UNIQUE INDEX idx_identities_tenant_type_name 
   ON identities(tenant_id, type, name) 
   WHERE type = 'SERVICE';
   
   CREATE INDEX idx_identities_tenant_status 
   ON identities(tenant_id, status);
   ```

2. **Transacciones Atómicas:** La creación de la identidad y su perfil se ejecuta en una sola transacción.

3. **Connection Pooling:** Se usa un pool de conexiones a PostgreSQL para reutilizar conexiones entre requests.

### 8.3. Escalabilidad Horizontal

- **Stateless Services:** El servicio KAGAMI es stateless, permitiendo escalado horizontal mediante Kubernetes HPA.
- **Particionamiento por Tenant:** Para tenants muy grandes (> 1M identidades), se implementa sharding de la base de datos por `tenant_id`.
- **Kafka Partitioning:** Los eventos se particionan por `tenant_id` para paralelismo en el consumo.

### 8.4. Bulk Operations

Para creación masiva de identidades (UC-05):

1. **Chunking:** El archivo se divide en chunks de 100 identidades.
2. **Parallel Processing:** Múltiples workers procesan chunks en paralelo.
3. **Progress Reporting:** El progreso se reporta vía WebSocket o polling.
4. **Error Handling:** Errores individuales no bloquean el procesamiento de otros chunks.

---

## 9. Trazabilidad con Otros Documentos del Paquete

| Documento | Relación |
| :--- | :--- |
| `02_Use_Case_Diagram.md` | Muestra el límite del sistema y los actores (administrador de tenant, KAGAMI Engine) |
| `03_Activity_Diagram.md` | Visualiza el flujo de validación, cifrado de PII, persistencia, y publicación de eventos |
| `04_Initial_Domain_Model.md` | Define el Servicio de Dominio `IdentityValidator`, el `PIIEncryptor`, y los Value Objects `Identity`, `Profile`, `CreateIdentityRequest` |
| `06_Robustness_Diagram.md` | Muestra la colaboración entre el Handler, el IdentityValidator, el PIIEncryptor, y los repositorios |
| `07_Sequence_Diagram.md` | Detalla el orden temporal de las llamadas: validación JWT → validación formato → validación unicidad → cifrado PII → persistencia → publicación evento |
| `08_Final_Class_Design.md` | Define las firmas exactas de los métodos: `IdentityValidator.Validate()`, `PIIEncryptor.Encrypt()`, `IdentityRepository.Create()` |
| `09_Modern_Artifacts.md` | Especifica los contratos OpenAPI, las métricas OpenTelemetry, y los patrones de resiliencia |
| `KIZUNA_Engine_Architecture.md` | Documento global que define la arquitectura completa del motor KAGAMI |
| `KHCS_SHUGO_Integration_Contract.md` | Define cómo KHCS consume la API de provisionamiento de identidades |
| `UC-06/01_Requirements_Specification.md` | Documento hermano que define la autenticación, que consume identidades creadas por UC-01 |

---

## 10. Glosario de Términos Específicos de UC-01

| Término | Definición |
| :--- | :--- |
| **Identity** | Representación digital única de un sujeto (usuario, servicio, dispositivo) dentro de SHUGO. Tiene un identificador global único y un ciclo de vida gestionado por KAGAMI. |
| **Profile** | Conjunto de atributos descriptivos asociados a una identidad (email, nombre, teléfono, atributos custom). Los campos PII se cifran antes de persistir. |
| **PII (Personally Identifiable Information)** | Información que puede usarse por sí sola o con otra información para identificar a un individuo. En SHUGO: email, teléfono, nombre completo. |
| **Provisioning** | Proceso de crear una nueva identidad en SHUGO. Puede ser manual (admin) o automático (SCIM desde HRIS). |
| **AES-256-GCM** | Algoritmo de cifrado simétrico con clave de 256 bits y modo GCM (Galois/Counter Mode). Proporciona confidencialidad y autenticación (AEAD). |
| **JSON Schema** | Estándar para validar la estructura de documentos JSON. Usado para validar atributos custom de identidades. |
| **RFC 5322** | Estándar de Internet para el formato de direcciones de email. |
| **E.164** | Estándar de la ITU-T para números de teléfono internacionales. Formato: +[código país][número] (ej. +5491123456789). |
| **Idempotencia** | Propiedad de una operación que produce el mismo resultado sin importar cuántas veces se ejecute. |
| **Consistencia Eventual** | Garantía de que, eventualmente, todos los nodos verán los mismos datos (después de publicación de eventos). |
| **Row-Level Security (RLS)** | Característica de PostgreSQL que restringe el acceso a filas de una tabla basado en el rol o sesión. Previene fugas de datos entre tenants. |
| **Master Key Rotation** | Proceso de reemplazar la clave maestra de cifrado periódicamente para limitar la exposición en caso de compromiso. |

---
*Fin del Documento 01_Requirements_Specification para UC-01*