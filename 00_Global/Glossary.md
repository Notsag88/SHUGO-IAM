# SHUGO - Glosario de Términos (Glossary)

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Documento:** | Glossary.md |
| **Metodología:** | Domain-Driven Design (Ubiquitous Language) |
| **Estándares:** | NIST SP 800-63, ISO/IEC 27001, GDPR, SCIM 2.0, SAML 2.0, OIDC 1.0 |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 2.0.0 |
| **Fecha:** | 30/08/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 27/08/2026 | Gastón Ruiz | Versión inicial con ~50 términos básicos de IAM y los 4 motores originales. | Base del lenguaje ubicuo del proyecto. |
| **2.0.0** | 30/08/2026 | Gastón Ruiz | **Refactorización completa.** Expansión a 200+ términos organizados en 12 categorías: Core IAM, Bounded Contexts (6), Arquitectura, Seguridad, Compliance, Integración, Criptografía, Patrones de Diseño, Protocolos, Modelos de Autorización, Observabilidad, y Negocio. Alineación con 49 UCs, 15 EPICs, y 6 Bounded Contexts. | Documento de referencia para todo el equipo. Garantiza que desarrolladores, arquitectos, stakeholders y auditores compartan el mismo vocabulario preciso y no ambiguo. |

---

## 1. Introducción

Este documento define el **Lenguaje Ubicuo (Ubiquitous Language)** de SHUGO, el vocabulario compartido que todos los miembros del equipo (desarrolladores, arquitectos, product managers, stakeholders, auditores) utilizan para comunicarse sobre el dominio de Identity & Access Management.

Cada término tiene:
- **Definición precisa:** Sin ambigüedad, en el contexto de SHUGO.
- **Ejemplo:** Caso de uso real dentro de la plataforma.
- **Referencia cruzada:** Términos relacionados dentro del glosario.
- **Estándar asociado:** Norma o RFC que respalda la definición (cuando aplica).

---

## 2. Términos Core de IAM (Identity & Access Management)

| Término | Definición | Ejemplo en SHUGO | Estándar |
| :--- | :--- | :--- | :--- |
| **Identidad (Identity)** | Representación digital única de un sujeto (persona, servicio, dispositivo) dentro de SHUGO. Tiene un identificador global único y un ciclo de vida gestionado por KAGAMI. | `user:uuid-123`, `service:payment-api`, `device:iphone-15` | NIST 800-63A |
| **Sujeto (Subject)** | Entidad que solicita acceso a un recurso. Puede ser un usuario, un grupo, un servicio o un dispositivo. En KIZUNA, es el primer elemento de una tupla (subject, relation, object). | `user:veterinario-juan` solicita acceso a `registro_medico:uuid-123` | NIST 800-162 |
| **Recurso (Resource / Object)** | Entidad sobre la cual se evalúa un permiso. En KIZUNA, es el tercer elemento de una tupla. Tiene un tipo (namespace) y un identificador. | `registro_medico:uuid-123`, `documento:456` | NIST 800-162 |
| **Acción (Action)** | Operación que un sujeto quiere realizar sobre un recurso. Debe estar definida en una política de KIZUNA. | `rectificar`, `eliminar`, `leer`, `crear` | NIST 800-162 |
| **Credencial (Credential)** | Secreto o evidencia que demuestra la posesión de una identidad. Puede ser contraseña, certificado, clave pública, token, factor biométrico. | Contraseña Argon2id, TOTP secret, WebAuthn credential | NIST 800-63B |
| **Autenticación (Authentication)** | Proceso de verificar que un sujeto es quien dice ser, mediante la validación de una o más credenciales. Gestionado por KAGI. | Login con username/password + TOTP | NIST 800-63B |
| **Autorización (Authorization)** | Proceso de determinar si un sujeto autenticado tiene permiso para realizar una acción sobre un recurso específico. Gestionado por KIZUNA. | ¿Puede Juan rectificar el registro médico X? | NIST 800-162 |
| **Federación (Federation)** | Mecanismo por el cual SHUGO confía en la autenticación realizada por un proveedor de identidad externo (IdP). Gestionado por TORII. | Login con Google, GitHub, Microsoft Azure AD | SAML 2.0, OIDC |
| **Assurance Level** | Nivel de confianza en la identidad verificada o en el método de autenticación, según NIST SP 800-63. | IAL2: Identidad verificada con evidencia documental. AAL2: MFA requerido. | NIST 800-63 |
| **Perfil (Profile)** | Conjunto de atributos descriptivos asociados a una identidad (nombre, email, teléfono, atributos custom). Gestionado por KAGAMI. | `{ name: "Juan", email: "juan@khcs.com", department: "Veterinaria" }` | SCIM 2.0 |
| **Tenant** | Organización o unidad de negocio aislada dentro de SHUGO. Cada tenant tiene su propio conjunto de identidades, roles, namespaces y configuraciones. Gestionado por MURA. | `khcs-001`, `ecommerce-xyz` | ISO 27001 A.9 |
| **Rol (Role)** | Etiqueta asignada a un sujeto dentro de un tenant que determina un conjunto de permisos. Gestionado por MURA. | `admin`, `veterinario`, `mozo`, `lector` | NIST 800-53 AC-2 |
| **Permiso (Permission)** | Capacidad específica otorgada por un rol para realizar una acción sobre un tipo de recurso. | `create:registro_medico`, `delete:documento` | NIST 800-53 AC-3 |
| **Grupo (Group)** | Colección de identidades que comparten una membresía común. Puede ser estático (miembros explícitos) o dinámico (reglas basadas en atributos). Gestionado por MURA. | `group:veterinarios`, `group:mozos_turno_a` | SCIM 2.0 |
| **Sesión (Session)** | Período de interacción continua entre un sujeto autenticado y SHUGO. Tiene un inicio, un tiempo de vida y un fin. Gestionado por KAGI. | Sesión activa desde las 10:00 hasta las 10:30 | OWASP Session Management |
| **Token de Acceso (Access Token)** | Token JWT de corta duración (15 min) que el sujeto presenta para acceder a APIs protegidas. Emitido por KAGI. | `eyJhbGciOiJSUzI1NiIs...` | RFC 7519 (JWT) |
| **Token de Refresco (Refresh Token)** | Token opaco de larga duración (7 días) que el sujeto usa para obtener nuevos tokens de acceso sin re-autenticarse. Implementa Refresh Token Rotation (RTR). | `dGhpcyBpcyBhIHJlZnJlc2g...` | OAuth 2.0 RFC 6749 |
| **Factor de Autenticación (Auth Factor)** | Categoría de evidencia utilizada para autenticar un sujeto. Tres tipos: conocimiento (algo que sabes), posesión (algo que tienes), inherencia (algo que eres). | Contraseña (conocimiento), TOTP (posesión), huella (inherencia) | NIST 800-63B |
| **MFA (Multi-Factor Authentication)** | Autenticación que requiere dos o más factores de diferentes categorías. Requisito para AAL2 y AAL3. | Contraseña + TOTP, WebAuthn (biometría + hardware) | NIST 800-63B |
| **Dispositivo (Device)** | Hardware utilizado por un sujeto para interactuar con SHUGO. Puede ser registrado y gestionado. Tiene un fingerprint único. | iPhone 15, Laptop corporativo | NIST 800-63B |

---

## 3. Términos por Bounded Context

### 3.1. KAGAMI (Identity Lifecycle) - EPIC-01

| Término | Definición | Ejemplo |
| :--- | :--- | :--- |
| **Provisioning** | Proceso de crear una nueva identidad en SHUGO. Puede ser manual (admin) o automático (SCIM desde HRIS). | UC-01: Crear identidad `user:veterinario-juan` |
| **Soft Delete (Deactivation)** | Marcado lógico de una identidad como eliminada sin borrarla físicamente. Permite reactivación y mantiene integridad referencial. | `status: DEACTIVATED` |
| **Hard Delete (Purge)** | Borrado físico de una identidad y sus datos personales. Cumple con GDPR Art. 17 (Right to Erasure). | `status: PURGED`, PII anonimizada |
| **PII (Personally Identifiable Information)** | Datos que pueden identificar a un individuo (email, teléfono, nombre). Cifrados a nivel de campo en KAGAMI. | `email: "juan@khcs.com"` (AES-256-GCM) |
| **Custom Attributes** | Atributos personalizados definidos por el tenant en formato JSONB. Permiten extensibilidad sin alterar el schema. | `{"department": "Veterinaria", "clearance": 3}` |
| **Identity Lifecycle** | Máquina de estados que gobierna el ciclo de vida de una identidad: Provisioned → Active → Suspended → Deactivated → Purged. | Ver State Machine en KAGAMI Architecture |

### 3.2. KAGI (Authentication & Sessions) - EPIC-02, EPIC-03

| Término | Definición | Ejemplo |
| :--- | :--- | :--- |
| **AAL (Authenticator Assurance Level)** | Nivel de confianza en el método de autenticación. AAL1 (single factor), AAL2 (MFA), AAL3 (hardware-based). | WebAuthn = AAL3, TOTP = AAL2 |
| **Refresh Token Rotation (RTR)** | Patrón de seguridad donde cada uso de refresh token genera uno nuevo y revoca el anterior. Detecta reuso y revoca toda la familia. | Ver Token Service en KAGI Architecture |
| **Token Family** | Conjunto de refresh tokens relacionados por un `family_id`. Si se detecta reuso, toda la familia se revoca. | `family_id: uuid-abc` |
| **WebAuthn** | Estándar W3C para autenticación sin contraseña usando criptografía de clave pública. Soporta biometría y hardware tokens. | YubiKey, FaceID, TouchID |
| **TOTP (Time-based One-Time Password)** | Código de 6 dígitos que cambia cada 30 segundos, basado en RFC 6238. Usado como segundo factor MFA. | Google Authenticator, Authy |
| **Magic Link** | Enlace de un solo uso enviado por email para autenticación sin contraseña. Expira en 5 minutos. | Login por email |
| **Risk Score** | Puntuación de 0 a 100 que evalúa el riesgo de un intento de autenticación basado en dispositivo, IP, geo-velocidad, etc. | Score > 80 → BLOCK, > 40 → REQUIRE_MFA |
| **Geo-Velocity** | Cálculo de velocidad entre dos puntos de login para detectar viajes imposibles (ej. login en Argentina y Japón en 1 hora). | Velocidad > 800 km/h → Riesgo alto |
| **Device Fingerprint** | Hash único generado a partir de características del dispositivo (browser, OS, screen, fonts, etc.). | `fingerprint: sha256-abc123` |
| **JWKS (JSON Web Key Set)** | Conjunto de claves públicas expuestas por KAGI para que los clientes validen firmas JWT. | `/.well-known/jwks.json` |
| **Session Binding** | Vinculación de una sesión a dispositivo, IP y User-Agent para detectar session hijacking. | `device_id + ip + user_agent` |

### 3.3. MURA (Directory & Tenancy) - EPIC-04, EPIC-05

| Término | Definición | Ejemplo |
| :--- | :--- | :--- |
| **Multi-Tenancy** | Capacidad de servir múltiples tenants con aislamiento completo de datos. Implementado con particionamiento por `tenant_id` y RLS. | `khcs-001` no puede ver datos de `ecommerce-xyz` |
| **Row-Level Security (RLS)** | Característica de PostgreSQL que restringe el acceso a filas de una tabla basado en el rol o sesión. Previene fugas de datos. | `WHERE tenant_id = current_setting('app.current_tenant_id')` |
| **Dynamic Group** | Grupo cuya membresía se calcula automáticamente basándose en atributos de las identidades (JSON Logic). | `{"==": [{"var": "department"}, "Sales"]}` |
| **Access Review** | Proceso periódico de revisión y certificación de accesos por parte de los propietarios de los datos. Requisito SOC 2. | Campaña trimestral de revisión de roles admin |
| **Delegated Administration** | Capacidad de asignar responsabilidades administrativas a usuarios no globales, limitadas por scope. | Manager de Ventas administra solo su equipo |
| **Segregation of Duties (SoD)** | Principio de control interno que previene que una sola persona controle todas las fases de una transacción crítica. | No puede tener roles `APPROVE_PAYMENTS` y `CREATE_VENDOR` |
| **Snapshot (Review)** | Copia congelada del estado de las asignaciones al inicio de una campaña de revisión. | Revisores ven lista fija, no cambia en tiempo real |
| **Role Hierarchy** | Relación de herencia entre roles. En SHUGO, roles planos con herencia explícita de permisos. | Rol "Senior" incluye permisos de "Junior" |

### 3.4. KIZUNA (Authorization Engine) - EPIC-06, EPIC-07

| Término | Definición | Ejemplo |
| :--- | :--- | :--- |
| **ReBAC (Relationship-Based Access Control)** | Modelo de autorización que evalúa permisos basándose en las relaciones entre sujetos y objetos en un grafo. | ¿Tiene Juan relación "autor" con registro 123? |
| **ABAC (Attribute-Based Access Control)** | Modelo de autorización que evalúa permisos basándose en atributos contextuales del sujeto, recurso o entorno. | Solo permitir acceso desde IP corporativa |
| **Namespace** | Contenedor lógico dentro de un tenant que agrupa tipos de objetos y define las relations válidas. Es el vocabulario de autorización del tenant. | `registro_medico`, `equino`, `documento` |
| **Relation** | Tipo de vínculo semántico entre un sujeto y un objeto, definido dentro de un namespace. | `autor`, `veterinario_asignado`, `editor` |
| **Tupla (Tuple)** | Unidad atómica del grafo de relaciones. Es una tripleta (sujeto, relación, objeto) que representa un hecho de autorización. | `(user:juan, autor, registro_medico:123)` |
| **Grafo de Relaciones (Relation Graph)** | Estructura de datos tipo grafo dirigida formada por todas las tuplas de un tenant. Los nodos son sujetos y objetos; las aristas son relations. | Grafo completo de permisos de KHCS |
| **BFS (Breadth-First Search)** | Algoritmo de traversación de grafos que explora nivel por nivel. Usado por KIZUNA para encontrar caminos entre subject y object. | Máximo 5 saltos, timeout 100ms |
| **Policy** | Regla que define qué relaciones o condiciones otorgan permiso para una acción específica. Combina ReBAC y ABAC. | `action=rectificar → rule="autor OR director_medico"` |
| **Check API** | Endpoint de KIZUNA que evalúa si un sujeto tiene permiso para realizar una acción sobre un recurso. Es el endpoint más consumido de SHUGO. | `POST /authz/check` |
| **Write API** | Endpoint de KIZUNA que crea, actualiza o elimina tuplas en el grafo de relaciones. | `POST /relations` |
| **Fail Secure** | Principio de diseño que dicta que, ante error, el sistema debe denegar el acceso (`allowed: false`) en lugar de permitirlo. | Timeout, DB caído, excepción → `allowed: false` |
| **Cache-Aside Pattern** | Patrón de caché donde la aplicación consulta caché primero, y si hay MISS, consulta la fuente de datos y actualiza la caché. | L1 (in-memory) → L2 (Redis) → L3 (PostgreSQL) |
| **Outbox Pattern** | Patrón que garantiza atomicidad entre persistencia de tuplas y publicación de eventos. El evento se guarda en la misma transacción. | `INSERT tuple + INSERT event → COMMIT` |

### 3.5. KAGE (Security & Compliance) - EPIC-08, EPIC-09, EPIC-12, EPIC-13

| Término | Definición | Ejemplo |
| :--- | :--- | :--- |
| **Audit Log** | Registro inmutable de un evento de seguridad o cambio de estado. Append-only, no modificable ni borrable. | `Identity.Created`, `Check.Denied` |
| **Consent** | Aprobación explícita e informada del usuario para el procesamiento de sus datos personales con un propósito específico. | Consentir compartir email con terceros |
| **Data Residency** | Requisito de que los datos se almacenen y procesen en una región geográfica específica. | Datos de usuarios EU en Frankfurt |
| **Break-Glass Access** | Acceso de emergencia que permite a un usuario autorizado saltar las restricciones normales en situaciones críticas, con auditoría reforzada. | Acceso de emergencia al historial clínico durante cirugía |
| **Just-in-Time (JIT) Access** | Acceso temporal concedido bajo demanda con workflow de aprobación y revocación automática. | Acceso de 2 horas a la base de datos de producción |
| **Password Policy** | Conjunto de reglas que definen los requisitos de complejidad, expiración, historial y detección de brechas para las contraseñas. | Mínimo 12 caracteres, expira cada 90 días |
| **Account Lockout** | Mecanismo que bloquea temporalmente una cuenta tras N intentos fallidos de autenticación. | Bloqueo tras 5 intentos, desbloqueo en 30 minutos |
| **Breach Detection** | Detección de credenciales comprometidas mediante integración con bases de datos de brechas (HaveIBeenPwned). | Password encontrado en breach → forzar cambio |
| **WORM (Write Once Read Many)** | Modelo de almacenamiento donde los datos, una vez escritos, no pueden ser modificados ni borrados. | Audit logs en PostgreSQL con `GRANT INSERT ONLY` |
| **Erasure Orchestrator** | Componente de KAGE que coordina el borrado de datos (GDPR Art. 17) enviando comandos a KAGAMI, KIZUNA y KAGI. | Saga pattern para Right to Erasure |
| **Immutability Guarantee** | Garantía de que ningún log de auditoría puede ser modificado o borrado, ni siquiera por administradores. | RLS + permisos DB + auditoría de DBAs |

### 3.6. TORII (Integration & Gateway) - EPIC-10, EPIC-11, EPIC-14, EPIC-15

| Término | Definición | Ejemplo |
| :--- | :--- | :--- |
| **SCIM (System for Cross-domain Identity Management)** | Estándar RFC 7643/7644 para el provisionamiento automatizado de identidades entre sistemas. | Sincronización de usuarios desde Workday |
| **SAML (Security Assertion Markup Language)** | Estándar XML para intercambio de datos de autenticación y autorización entre un IdP y un SP. | SHUGO como IdP para Salesforce |
| **OIDC (OpenID Connect)** | Capa de identidad sobre OAuth 2.0 que permite a las aplicaciones verificar la identidad del usuario. | SHUGO como IdP OIDC para aplicaciones web |
| **IdP (Identity Provider)** | Sistema que autentica usuarios y emite tokens/assertions para otros sistemas (Service Providers). | SHUGO actuando como IdP |
| **SP (Service Provider)** | Sistema que confía en un IdP para autenticar usuarios y consume los tokens/assertions emitidos. | KHCS consumiendo tokens de SHUGO |
| **Webhook** | Callback HTTP configurado por el cliente que SHUGO invoca automáticamente cuando ocurre un evento específico. | Notificar a KHCS cuando se crea un usuario |
| **API Key** | Clave secreta de larga duración utilizada por aplicaciones para autenticarse ante la API de SHUGO. Almacenada como hash SHA256 + prefijo. | `shugo_sk_live_abc123...` |
| **Rate Limit** | Límite máximo de requests que un cliente puede realizar en una ventana de tiempo determinada. | 1000 requests/minuto para Plan Enterprise |
| **HMAC (Hash-based Message Authentication Code)** | Mecanismo para calcular un código de autenticación de mensajes utilizando una función hash criptográfica en combinación con una clave secreta. | Firma de webhooks con HMAC SHA256 |
| **Anti-Corruption Layer (ACL)** | Patrón de diseño que traduce modelos de datos externos al modelo de dominio interno, previniendo la contaminación del core. | SCIM JSON → KAGAMI Identity |
| **JWKS Endpoint** | Endpoint público que expone las claves públicas utilizadas para validar firmas JWT. | `/.well-known/jwks.json` |
| **Claims Mapping** | Transformación de atributos internos de SHUGO a claims estándar (email, name, roles) para aplicaciones cliente. | `custom:employee_id` → `employee_id` |
| **Sliding Window Log** | Algoritmo de Rate Limiting que registra timestamps exactos de requests para calcular la tasa en una ventana deslizante precisa. | Redis Sorted Set con timestamps |
| **Dead Letter Queue (DLQ)** | Cola donde se almacenan mensajes/eventos que no pudieron ser procesados después de múltiples reintentos. | Webhooks fallidos después de 5 intentos |

---

## 4. Términos de Arquitectura

| Término | Definición | Ejemplo en SHUGO |
| :--- | :--- | :--- |
| **Bounded Context** | Delimitación semántica dentro de un dominio donde un modelo de dominio es aplicable. Cada BC tiene su propio lenguaje ubicuo. | KAGAMI, KAGI, MURA, KIZUNA, KAGE, TORII |
| **Aggregate Root** | Entidad raíz de un agregado que controla el acceso a las entidades internas. Punto de entrada para modificaciones. | `Identity` en KAGAMI, `Namespace` en KIZUNA |
| **Value Object** | Objeto inmutable definido por sus atributos, no por identidad. Se compara por valor. | `EmailAddress`, `RiskScore`, `CheckRequest` |
| **Entity** | Objeto definido por su identidad, no por sus atributos. Tiene ciclo de vida. | `Identity`, `Session`, `RelationTuple` |
| **Domain Event** | Evento que representa algo que ocurrió en el dominio. Se publica para notificar a otros componentes. | `Identity.Created`, `Tuple.Deleted` |
| **CQRS (Command Query Responsibility Segregation)** | Patrón que separa operaciones de lectura (queries) y escritura (commands) para optimizar rendimiento y escalabilidad. | Check API (read) vs Write API (write) en KIZUNA |
| **Event Sourcing** | Patrón que almacena cambios de estado como secuencia de eventos, permitiendo reconstrucción y auditoría. | Audit logs en KAGE |
| **Microservice** | Servicio independiente, desplegables y escalables, que implementa un Bounded Context. | KAGAMI Service, KIZUNA Service, etc. |
| **Service Mesh** | Infraestructura dedicada para manejar comunicación entre servicios (mTLS, observabilidad, resiliencia). | Istio o Linkerd entre microservicios |
| **mTLS (Mutual TLS)** | Autenticación bidireccional entre cliente y servidor. Ambos presentan certificados. | Comunicación entre microservicios |
| **API Gateway** | Punto de entrada único para todas las requests externas. Gestiona autenticación, rate limiting, enrutamiento. | Kong o AWS API Gateway |
| **Horizontal Scaling** | Escalabilidad agregando más instancias, no más recursos por instancia. | Kubernetes HPA para KIZUNA |
| **Particionamiento (Partitioning)** | División de tablas de base de datos por criterios (ej. tenant_id) para mejorar rendimiento y aislamiento. | `relation_tuples` particionada por tenant_id |
| **Sharding** | Distribución de datos entre múltiples nodos de base de datos. | Redis Cluster con hash slots |
| **Read Replica** | Copia de solo lectura de una base de datos para distribuir carga de consultas. | PostgreSQL read replicas para KIZUNA |
| **Connection Pooling** | Reutilización de conexiones de base de datos para reducir overhead. | PgBouncer o HikariCP |

---

## 5. Términos de Seguridad

| Término | Definición | Ejemplo en SHUGO |
| :--- | :--- | :--- |
| **Zero Trust** | Modelo de seguridad que nunca confía, siempre verifica. Cada request debe ser autenticada y autorizada. | JWT + mTLS + validación de permisos en cada request |
| **Fail Secure** | Principio de diseño que dicta que, ante error, el sistema debe denegar el acceso en lugar de permitirlo. | Check API retorna `allowed: false` ante cualquier error |
| **Defense in Depth** | Múltiples capas de defensa, no confiar en una sola. | JWT + mTLS + RBAC + ReBAC + ABAC + auditoría |
| **Least Privilege** | Principio de otorgar solo los permisos mínimos necesarios para realizar una tarea. | Roles scoped, JIT access |
| **Separation of Duties (SoD)** | Principio de control interno que previene conflictos de interés. | No puede aprobar y ejecutar pagos |
| **Encryption at Rest** | Cifrado de datos almacenados en discos o bases de datos. | AES-256 TDE en PostgreSQL |
| **Encryption in Transit** | Cifrado de datos en movimiento entre componentes. | TLS 1.3 entre todos los servicios |
| **Field-Level Encryption** | Cifrado de campos específicos dentro de una tabla (ej. PII). | Email y teléfono cifrados con AES-256-GCM |
| **Hashing** | Función criptográfica unidireccional para almacenar secretos (passwords, API keys). | Argon2id para passwords, SHA256 para API keys |
| **Salt** | Valor aleatorio añadido a un password antes de hashear para prevenir rainbow tables. | Salt único por password en Argon2id |
| **Certificate Pinning** | Validación de que el certificado del servidor coincide con un hash esperado. Previene MITM. | Clientes móviles validan certificado de SHUGO |
| **WAF (Web Application Firewall)** | Filtrado de tráfico HTTP malicioso (SQL injection, XSS, etc.). | AWS WAF o Cloudflare en API Gateway |
| **DDoS Protection** | Mecanismos para prevenir ataques de denegación de servicio distribuidos. | Rate limiting, auto-scaling, CDN |
| **Penetration Testing** | Evaluación de seguridad simulando ataques reales para identificar vulnerabilidades. | Trimestral por pentesters externos |
| **Vulnerability Scanning** | Escaneo automático de dependencias y contenedores en busca de vulnerabilidades conocidas. | Snyk, Trivy en CI/CD |

---

## 6. Términos de Compliance y Privacidad

| Término | Definición | Ejemplo en SHUGO |
| :--- | :--- | :--- |
| **GDPR (General Data Protection Regulation)** | Reglamento europeo de protección de datos personales. Aplica a cualquier organización que procese datos de residentes EU. | Consentimiento, derecho al olvido, portabilidad |
| **CCPA (California Consumer Privacy Act)** | Ley de privacidad de California que otorga derechos similares a GDPR para residentes de California. | Opt-out de venta de datos |
| **SOC 2 (Service Organization Control 2)** | Estándar de auditoría para organizaciones de servicios en la nube. Enfocado en seguridad, disponibilidad, integridad, confidencialidad y privacidad. | Certificación Type II para SHUGO |
| **ISO/IEC 27001** | Estándar internacional para sistemas de gestión de seguridad de la información (ISMS). | Certificación para SHUGO |
| **NIST SP 800-63** | Guías de identidad digital del NIST. Define Assurance Levels para autenticación y verificación de identidad. | AAL1, AAL2, AAL3 en KAGI |
| **NIST SP 800-53** | Controles de seguridad y privacidad para sistemas de información federales. Adoptado como referencia enterprise. | AC-2, AC-3, AU-2, IA-2 |
| **Right to Erasure (Derecho al Olvido)** | Derecho del usuario a solicitar la eliminación de sus datos personales (GDPR Art. 17). | Purge en KAGAMI, anonimización de audit logs |
| **Data Portability** | Derecho del usuario a recibir sus datos en formato legible y transferirlos a otro proveedor (GDPR Art. 20). | Export JSON en KAGAMI |
| **Consent Management** | Gestión del consentimiento explícito e informado del usuario para procesamiento de datos. | Consent records en KAGE |
| **Data Residency** | Requisito de que los datos se almacenen en una región geográfica específica. | EU data in Frankfurt, US data in Virginia |
| **Audit Trail** | Registro cronológico e inmutable de eventos para reconstruir y examinar la secuencia de actividades. | Audit logs en KAGE |
| **Compliance Report** | Documento que demuestra el cumplimiento de normativas y estándares. Generado por KAGE. | Reporte SOC 2, reporte GDPR |
| **Access Certification** | Proceso de revisión periódica de accesos para verificar que siguen siendo apropiados. | Access Reviews en MURA |
| **Retention Policy** | Política que define cuánto tiempo se deben conservar los datos antes de eliminarlos. | Audit logs: 7 años, PII: hasta purga |

---

## 7. Términos de Integración y Protocolos

| Término | Definición | Ejemplo en SHUGO |
| :--- | :--- | :--- |
| **OAuth 2.0** | Framework de autorización que permite a aplicaciones obtener acceso limitado a recursos en nombre de un usuario. | Flujo de autorización para APIs de SHUGO |
| **OpenID Connect (OIDC)** | Capa de identidad sobre OAuth 2.0 que permite verificar la identidad del usuario. | SHUGO como IdP OIDC |
| **SAML 2.0** | Estándar XML para intercambio de datos de autenticación y autorización entre dominios de seguridad. | SHUGO como IdP SAML para Salesforce |
| **JWT (JSON Web Token)** | Estándar RFC 7519 para tokens compactos y autocontenidos. Usado como Access Token en SHUGO. | `eyJhbGciOiJSUzI1NiIs...` |
| **JWK (JSON Web Key)** | Estándar RFC 7517 para representar claves criptográficas en formato JSON. | Claves públicas en JWKS endpoint |
| **JWKS (JSON Web Key Set)** | Conjunto de JWKs expuestos por un IdP para validación de firmas JWT. | `/.well-known/jwks.json` |
| **SCIM 2.0** | Estándar RFC 7643/7644 para provisionamiento automatizado de identidades. | Sincronización desde Workday |
| **LDAP (Lightweight Directory Access Protocol)** | Protocolo para acceder y mantener servicios de directorio. | Integración con Active Directory |
| **Active Directory (AD)** | Servicio de directorio de Microsoft para gestionar identidades en redes Windows. | Directory Sync en TORII |
| **HRIS (Human Resources Information System)** | Sistema de gestión de recursos humanos. | Workday, BambooHR, SAP SuccessFactors |
| **SIEM (Security Information and Event Management)** | Sistema de análisis centralizado de logs de seguridad. | Splunk, QRadar consumiendo logs de KAGE |
| **SOAR (Security Orchestration, Automation and Response)** | Automatización de respuesta a incidentes de seguridad. | Playbooks para alertas de KAGE |
| **Webhook Signature** | Firma criptográfica (HMAC) adjunta a un webhook para verificar autenticidad. | `X-SHUGO-Signature: sha256=abc...` |
| **Idempotency** | Propiedad de una operación que produce el mismo resultado sin importar cuántas veces se ejecute. | Crear tupla duplicada retorna 409, no error |
| **Callback URL** | URL a la que un servicio envía notificaciones asíncronas. | Webhook endpoint del cliente |

---

## 8. Términos de Criptografía

| Término | Definición | Ejemplo en SHUGO |
| :--- | :--- | :--- |
| **AES-256** | Algoritmo de cifrado simétrico con clave de 256 bits. Estándar para cifrado de datos en reposo. | TDE en PostgreSQL, field-level encryption |
| **RSA** | Algoritmo de cifrado asimétrico basado en factorización de números primos. Usado para firmas JWT y SAML. | RS256 para JWT signing |
| **Argon2id** | Función de hashing de passwords resistente a GPU y side-channel attacks. Recomendado por OWASP. | Hashing de passwords en KAGAMI |
| **SHA-256** | Función hash criptográfica de 256 bits. Usada para integridad y hashing de API keys. | Hash de API keys en TORII |
| **HMAC SHA-256** | Código de autenticación de mensajes basado en hash. Usado para firmar webhooks. | Firma de payloads de webhooks |
| **TOTP (Time-based One-Time Password)** | Algoritmo RFC 6238 que genera códigos de un solo uso basados en tiempo. | Segundo factor MFA |
| **FIDO2/WebAuthn** | Estándar W3C para autenticación sin contraseña usando criptografía de clave pública. | Autenticación biométrica |
| **Public Key Infrastructure (PKI)** | Infraestructura de claves públicas para gestión de certificados digitales. | Certificados mTLS entre servicios |
| **Certificate Authority (CA)** | Entidad que emite y firma certificados digitales. | Let's Encrypt o CA interna |
| **Nonce** | Número usado una sola vez para prevenir replay attacks. | Nonce en OIDC authentication |
| **Salt** | Valor aleatorio añadido a un input antes de hashear para prevenir rainbow tables. | Salt único por password |
| **Key Rotation** | Proceso de reemplazar claves criptográficas periódicamente para limitar exposición. | Rotación de JWKS keys cada 90 días |

---

## 9. Términos de Patrones de Diseño

| Término | Definición | Ejemplo en SHUGO |
| :--- | :--- | :--- |
| **Repository Pattern** | Patrón que abstrae el acceso a datos, permitiendo testeo unitario con mocks. | `INamespaceRepository`, `IRelationTupleRepository` |
| **Unit of Work** | Patrón que gestiona transacciones atómicas para múltiples operaciones. | Creación de tupla + evento en misma transacción |
| **Factory Method** | Patrón que define una interfaz para crear objetos, pero deja que las subclases decidan qué clase instanciar. | `CheckResult.Allowed()`, `CheckResult.Denied()` |
| **Strategy Pattern** | Patrón que define una familia de algoritmos, encapsula cada uno y los hace intercambiables. | Diferentes estrategias de caché (L1, L2, L3) |
| **Observer Pattern** | Patrón que define una dependencia uno-a-muchos entre objetos, de modo que cuando uno cambia, todos sus dependientes son notificados. | Publicación de eventos de dominio |
| **Circuit Breaker** | Patrón que previene cascadas de fallos cuando un servicio está caído. | Circuit breaker para llamadas a Redis/Kafka |
| **Retry with Backoff** | Patrón que reintenta operaciones fallidas con delays crecientes. | Reintentos de webhooks con exponential backoff |
| **Bulkhead Pattern** | Patrón que aísla fallos limitando el impacto a un subconjunto del sistema. | Thread pools separados para Check API vs Write API |
| **Cache-Aside Pattern** | Patrón donde la aplicación consulta caché primero, y si hay MISS, consulta la fuente de datos. | L1 → L2 → L3 en KIZUNA |
| **Outbox Pattern** | Patrón que garantiza atomicidad entre persistencia y publicación de eventos. | INSERT tuple + INSERT event → COMMIT |
| **Saga Pattern** | Patrón de gestión de transacciones distribuidas mediante secuencia de transacciones locales y eventos compensatorios. | Erasure orchestrator en KAGE |
| **Anti-Corruption Layer (ACL)** | Patrón que traduce modelos de datos externos al modelo de dominio interno. | SCIM JSON → KAGAMI Identity |
| **CQRS (Command Query Responsibility Segregation)** | Patrón que separa operaciones de lectura y escritura. | Check API (read) vs Write API (write) |
| **Event Sourcing** | Patrón que almacena cambios de estado como secuencia de eventos. | Audit logs inmutables en KAGE |
| **Fail Safe** | Patrón que captura excepciones en efectos secundarios sin propagarlas. | Invalidación de caché no bloquea creación de tupla |

---

## 10. Términos de Observabilidad

| Término | Definición | Ejemplo en SHUGO |
| :--- | :--- | :--- |
| **OpenTelemetry** | Estándar vendor-neutral para recolección de métricas, traces y logs. | SDK instrumentado en todos los microservicios |
| **Distributed Tracing** | Seguimiento de una request a través de múltiples servicios. | Trace ID propagado de API Gateway a KIZUNA |
| **Span** | Unidad de trabajo dentro de un trace. Representa una operación en un servicio. | Span para "Check L1 cache", "BFS traversal" |
| **Trace ID** | Identificador único que correlaciona todos los spans de una request. | `trace_id: abc123def456` |
| **Prometheus** | Sistema de monitoreo y alertas basado en pull de métricas time-series. | Métricas de latencia, throughput, errores |
| **Grafana** | Plataforma de visualización y dashboards para métricas. | Dashboards de KIZUNA performance |
| **Jaeger** | Sistema de distributed tracing para visualizar flujos de requests. | Trace de Check API end-to-end |
| **ELK Stack** | Elasticsearch, Logstash, Kibana para logs centralizados y búsqueda. | Logs de auditoría en KAGE |
| **Metric** | Medición numérica de un aspecto del sistema. | `shugo.kizuna.check.latency` |
| **Histogram** | Tipo de métrica que distribuye observaciones en buckets. | Latencia p50, p95, p99 |
| **Counter** | Tipo de métrica que solo incrementa. | Total de checks evaluados |
| **Gauge** | Tipo de métrica que puede subir y bajar. | Sesiones activas, tamaño de caché |
| **Alert** | Notificación cuando una métrica cruza un umbral. | Latencia p95 > 100ms → alerta a PagerDuty |
| **SLA (Service Level Agreement)** | Acuerdo de nivel de servicio con el cliente. | 99.99% uptime para Plan Enterprise |
| **SLO (Service Level Objective)** | Objetivo interno de nivel de servicio. | Latencia p95 < 50ms |
| **SLI (Service Level Indicator)** | Medición real de un aspecto del servicio. | Latencia p95 actual = 45ms |

---

## 11. Términos de Negocio y Producto

| Término | Definición | Ejemplo en SHUGO |
| :--- | :--- | :--- |
| **ARR (Annual Recurring Revenue)** | Ingresos recurrentes anuales normalizados. | $1.2M ARR en Año 1 |
| **CAC (Customer Acquisition Cost)** | Costo de adquirir un cliente (marketing + sales + onboarding). | $5,000 para Plan Professional |
| **LTV (Lifetime Value)** | Valor total que un cliente genera durante su vida útil. | $60,000 para Plan Professional |
| **NRR (Net Revenue Retention)** | Porcentaje de revenue retenido después de expansiones y churn. | 120% (upsell > churn) |
| **Churn Rate** | Porcentaje de clientes que cancelan en un período. | < 10% anual en Año 1 |
| **MAU (Monthly Active Users)** | Número de usuarios únicos activos en un mes. | 5,000 MAU en Año 1 |
| **TAM (Total Addressable Market)** | Mercado total disponible para el producto. | $25B IAM market |
| **SAM (Serviceable Addressable Market)** | Segmento del TAM que el producto puede servir. | $5B mid-market + enterprise |
| **SOM (Serviceable Obtainable Market)** | Porción del SAM que se puede capturar realisticamente. | $500M en 3 años |
| **MVP (Minimum Viable Product)** | Versión mínima del producto con features core para validar el mercado. | 32 UCs de prioridad alta |
| **Product-Market Fit** | Cuando el producto satisface una fuerte demanda del mercado. | 5 clientes beta activos |
| **Buyer Persona** | Representación semi-ficticia del cliente ideal. | CTO, CISO, Lead Developer, Compliance Officer |
| **Use Case (UC)** | Descripción de una interacción entre un actor y el sistema para lograr un objetivo. | UC-30: Evaluar Permiso |
| **EPIC** | Grupo de casos de uso relacionados que entregan una capacidad de negocio. | EPIC-06: ReBAC - KIZUNA Core |
| **User Story** | Descripción corta de una funcionalidad desde la perspectiva del usuario. | "Como admin, quiero crear un namespace para..." |
| **Acceptance Criteria** | Condiciones que deben cumplirse para que una user story se considere completa. | "Dado X, cuando Y, entonces Z" |
| **Definition of Done (DoD)** | Criterios que definen cuando un trabajo está completo. | Código + tests + docs + review |

---

## 12. Términos de DevOps y Operaciones

| Término | Definición | Ejemplo en SHUGO |
| :--- | :--- | :--- |
| **CI/CD (Continuous Integration/Continuous Deployment)** | Automatización de build, test y deployment. | GitHub Actions → ECR → Kubernetes |
| **Blue/Green Deployment** | Estrategia de deployment con dos ambientes idénticos para zero-downtime. | Switch de tráfico entre blue y green |
| **Canary Release** | Rollout gradual de una nueva versión a un subconjunto de usuarios. | 10% → 50% → 100% |
| **Feature Flag** | Mecanismo para activar/desactivar features sin deployment. | LaunchDarkly o flags propios |
| **Kubernetes (K8s)** | Plataforma de orquestación de contenedores para deployment, scaling y management. | EKS en AWS |
| **HPA (Horizontal Pod Autoscaler)** | Escalado automático de pods basado en métricas (CPU, memoria, custom). | KIZUNA escala de 3 a 50 pods |
| **Docker** | Plataforma de contenedores para empaquetar aplicaciones y dependencias. | Imagen `shugo/kizuna:1.0.0` |
| **Helm** | Package manager para Kubernetes. | Charts para cada microservicio |
| **Terraform** | Infrastructure as Code para provisionar recursos cloud. | EKS, RDS, ElastiCache |
| **GitOps** | Metodología donde la infraestructura se gestiona mediante Git. | ArgoCD o Flux |
| **Runbook** | Documento de procedimientos para operaciones y troubleshooting. | Runbook para "Redis caído" |
| **Incident Response** | Proceso estructurado para manejar incidentes de seguridad o operativos. | Playbook para "Brecha de datos" |
| **Postmortem** | Análisis retrospectivo de un incidente para identificar causas y mejoras. | Postmortem de "Outage de 30 min" |
| **On-Call** | Rotación de ingenieros responsables de responder incidentes. | PagerDuty rotation |

---

## 13. Abreviaturas y Acrónimos

| Abreviatura | Significado Completo |
| :--- | :--- |
| **IAM** | Identity & Access Management |
| **IdP** | Identity Provider |
| **SP** | Service Provider |
| **MFA** | Multi-Factor Authentication |
| **SSO** | Single Sign-On |
| **RBAC** | Role-Based Access Control |
| **ReBAC** | Relationship-Based Access Control |
| **ABAC** | Attribute-Based Access Control |
| **PBAC** | Policy-Based Access Control |
| **ACL** | Access Control List / Anti-Corruption Layer |
| **PII** | Personally Identifiable Information |
| **GDPR** | General Data Protection Regulation |
| **CCPA** | California Consumer Privacy Act |
| **SOC** | Service Organization Control |
| **ISO** | International Organization for Standardization |
| **NIST** | National Institute of Standards and Technology |
| **RFC** | Request for Comments (IETF standards) |
| **JWT** | JSON Web Token |
| **JWK** | JSON Web Key |
| **JWKS** | JSON Web Key Set |
| **OIDC** | OpenID Connect |
| **SAML** | Security Assertion Markup Language |
| **SCIM** | System for Cross-domain Identity Management |
| **LDAP** | Lightweight Directory Access Protocol |
| **AD** | Active Directory |
| **HRIS** | Human Resources Information System |
| **SIEM** | Security Information and Event Management |
| **SOAR** | Security Orchestration, Automation and Response |
| **WAF** | Web Application Firewall |
| **DDoS** | Distributed Denial of Service |
| **MITM** | Man-in-the-Middle |
| **XSS** | Cross-Site Scripting |
| **CSRF** | Cross-Site Request Forgery |
| **SQLi** | SQL Injection |
| **TDE** | Transparent Data Encryption |
| **RLS** | Row-Level Security |
| **BFS** | Breadth-First Search |
| **DFS** | Depth-First Search |
| **LRU** | Least Recently Used |
| **TTL** | Time To Live |
| **RTR** | Refresh Token Rotation |
| **DLQ** | Dead Letter Queue |
| **SLA** | Service Level Agreement |
| **SLO** | Service Level Objective |
| **SLI** | Service Level Indicator |
| **RTO** | Recovery Time Objective |
| **RPO** | Recovery Point Objective |
| **ARR** | Annual Recurring Revenue |
| **CAC** | Customer Acquisition Cost |
| **LTV** | Lifetime Value |
| **NRR** | Net Revenue Retention |
| **MAU** | Monthly Active Users |
| **TAM** | Total Addressable Market |
| **SAM** | Serviceable Addressable Market |
| **SOM** | Serviceable Obtainable Market |
| **MVP** | Minimum Viable Product |
| **UC** | Use Case |
| **BC** | Bounded Context |
| **ADR** | Architecture Decision Record |
| **DoD** | Definition of Done |
| **CI/CD** | Continuous Integration / Continuous Deployment |
| **K8s** | Kubernetes |
| **HPA** | Horizontal Pod Autoscaler |
| **EKS** | Elastic Kubernetes Service (AWS) |
| **RDS** | Relational Database Service (AWS) |
| **ECR** | Elastic Container Registry (AWS) |

---

## 14. Referencias Cruzadas por Bounded Context

### KAGAMI (Identity Lifecycle)
- Identidad, Perfil, PII, Provisioning, Soft Delete, Hard Delete, Custom Attributes, Identity Lifecycle

### KAGI (Authentication & Sessions)
- Autenticación, Credencial, MFA, AAL, TOTP, WebAuthn, Magic Link, Risk Score, Geo-Velocity, Device Fingerprint, JWKS, Session Binding, Refresh Token Rotation, Token Family

### MURA (Directory & Tenancy)
- Tenant, Multi-Tenancy, Rol, Permiso, Grupo, Dynamic Group, Access Review, Delegated Administration, Segregation of Duties, Snapshot, Role Hierarchy, RLS

### KIZUNA (Authorization Engine)
- ReBAC, ABAC, Namespace, Relation, Tupla, Grafo de Relaciones, BFS, Policy, Check API, Write API, Fail Secure, Cache-Aside Pattern, Outbox Pattern

### KAGE (Security & Compliance)
- Audit Log, Consent, Data Residency, Break-Glass Access, JIT Access, Password Policy, Account Lockout, Breach Detection, WORM, Erasure Orchestrator, Immutability Guarantee

### TORII (Integration & Gateway)
- SCIM, SAML, OIDC, IdP, SP, Webhook, API Key, Rate Limit, HMAC, Anti-Corruption Layer, JWKS Endpoint, Claims Mapping, Sliding Window Log, Dead Letter Queue

---

## 15. Referencias y Estándares

1. **NIST SP 800-63:** Digital Identity Guidelines - https://pages.nist.gov/800-63-3/
2. **NIST SP 800-53:** Security and Privacy Controls - https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
3. **NIST SP 800-162:** Guide to Attribute Based Access Control - https://csrc.nist.gov/publications/detail/sp/800-162/final
4. **ISO/IEC 27001:2022:** Information Security Management - https://www.iso.org/standard/27001
5. **GDPR:** General Data Protection Regulation - https://gdpr.eu/
6. **SOC 2:** Service Organization Control 2 - https://www.aicpa.org/
7. **OAuth 2.0 (RFC 6749):** https://tools.ietf.org/html/rfc6749
8. **OpenID Connect Core 1.0:** https://openid.net/specs/openid-connect-core-1_0.html
9. **SAML 2.0:** https://docs.oasis-open.org/security/saml/v2.0/
10. **SCIM 2.0 (RFC 7643/7644):** https://scim.dev/
11. **JWT (RFC 7519):** https://tools.ietf.org/html/rfc7519
12. **JWK (RFC 7517):** https://tools.ietf.org/html/rfc7517
13. **TOTP (RFC 6238):** https://tools.ietf.org/html/rfc6238
14. **WebAuthn Level 2:** https://www.w3.org/TR/webauthn-2/
15. **OWASP Top 10:** https://owasp.org/www-project-top-ten/
16. **OWASP ASVS:** https://owasp.org/www-project-application-security-verification-standard/

---
*Fin del Documento Glossary v2.0 para SHUGO*