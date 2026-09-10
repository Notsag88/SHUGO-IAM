# 📚 SHUGO - EPIC 01: Core Identity Lifecycle Management

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **EPIC:** | EPIC-01 - Core Identity Lifecycle Management (Motor KAGAMI) |
| **Versión del Documento:** | 1.0.0 |
| **Fecha:** | 03/09/2026 |
| **Estado:** | ✅ Completado (60/60 Documentos) |

---

## 1. Introducción y Propósito

Este directorio contiene la especificación de diseño arquitectónico y técnico completo para el **EPIC-01**, que abarca el ciclo de vida central de las identidades en la plataforma SHUGO. 

El motor **KAGAMI** (Identity Lifecycle Engine) es el núcleo del sistema, responsable de la creación, modificación, consulta, desactivación y purga irreversible de identidades, garantizando en todo momento el cumplimiento normativo (GDPR, SOC 2, ISO 27001, NIST SP 800-88), la seguridad criptográfica y el rendimiento enterprise.

### 1.1. Alcance del EPIC-01
Este EPIC cubre exclusivamente las operaciones fundamentales del ciclo de vida de una identidad:
1. **UC-01:** Provisionar Nueva Identidad (Creación)
2. **UC-02:** Actualizar Identidad (Mutación con OCC y Merge Patch)
3. **UC-03:** Desactivar Identidad (Soft Delete con Grace Period)
4. **UC-04:** Consultar Identidad (CQRS Query Side con Cache-Aside)
5. **UC-05:** Purgar Identidad (Hard Delete / Crypto-Shredding / GDPR Art. 17)

> **Nota sobre Numeración:** Originalmente, "Importar/Exportar Identidades" estaba previsto como UC-05. Tras un análisis arquitectónico, se determinó que esta operación pertenece al dominio de integración masiva. Por lo tanto, **UC-05 se asignó a "Purgar Identidad"** para mantener la coherencia del ciclo de vida core, y "Importar/Exportar" se reubicará en el **EPIC-10: Provisioning Integration**.

---

## 2. Estándar de Diseño Aplicado

Cada Caso de Uso (UC) dentro de este EPIC ha sido diseñado siguiendo un rigoroso estándar de **12 documentos** (Opción A+), divididos en dos fases:

### Fase 1: Diseño Funcional y de Dominio (Documentos 01-09)
| # | Documento | Metodología | Propósito |
| :--- | :--- | :--- | :--- |
| **01** | `Requirements_Specification.md` | ICONIX + BDD | Especificación analítica, actores, precondiciones y criterios de aceptación Gherkin. |
| **02** | `Use_Case_Diagram.md` | UML 2.5 | Límites del sistema, actores y relaciones `<<include>>` / `<<extend>>`. |
| **03** | `Activity_Diagram.md` | UML 2.5 + Swimlanes | Flujo de trabajo interno, decisiones lógicas y carriles de responsabilidad. |
| **04** | `Initial_Domain_Model.md` | DDD + ICONIX | Modelo conceptual de Aggregate Roots, Entities, Value Objects y Domain Services. |
| **05** | `State_Machine_Diagram.md` | UML 2.5 | Máquinas de estados, transiciones permitidas y guardas (guards) de dominio. |
| **06** | `Robustness_Diagram.md` | ICONIX | Validación de acoplamiento entre Boundary, Control y Entity objects. |
| **07** | `Sequence_Diagram.md` | UML 2.5 | Orden temporal exacto de las interacciones entre objetos y sistemas externos. |
| **08** | `Final_Class_Design.md` | DDD + Clean Arch | Firmas de métodos, visibilidad, patrones de diseño y código C# de referencia. |
| **09** | `Modern_Artifacts.md` | API-First + DevSecOps | Contratos OpenAPI 3.1, métricas OpenTelemetry, Feature Flags y Compliance. |

### Fase 2: Diseño Técnico Específico (Documentos 10-12)
| # | Documento | Metodología | Propósito |
| :--- | :--- | :--- | :--- |
| **10** | `UC_Specific_Data_Model.md` | Physical Data Modeling | DDL, índices optimizados, particionamiento, RLS y estrategias de almacenamiento. |
| **11** | `UC_Specific_Security_Threats.md` | STRIDE + DREAD + Attack Trees | Análisis de amenazas específico, mitigaciones de nivel elite y compliance mapping. |
| **12** | `UC_Specific_Performance_Benchmarks.md` | SRE + Load Testing | SLIs, SLOs, estrategias de optimización, capacity planning y scripts de k6. |

---

## 3. Índice de Casos de Uso (Status: 100% Completado)

| UC | Nombre del Caso de Uso | Dominio | Documentos | Estado |
| :--- | :--- | :--- | :---: | :---: |
| **UC-01** | Provisionar Nueva Identidad | Identity Creation | [12/12](./UC-01_Provisionar_Identidad/) | ✅ Completo |
| **UC-02** | Actualizar Identidad | Identity Mutation | [12/12](./UC-02_Actualizar_Identidad/) | ✅ Completo |
| **UC-03** | Desactivar Identidad | Lifecycle Management | [12/12](./UC-03_Desactivar_Identidad/) | ✅ Completo |
| **UC-04** | Consultar Identidad | CQRS Query Side | [12/12](./UC-04_Consultar_Identidad/) | ✅ Completo |
| **UC-05** | Purgar Identidad | Compliance & Deletion | [12/12](./UC-05_Purgar_Identidad/) | ✅ Completo |

**Total de Documentos Generados en EPIC-01: 60**

---

## 4. Principios Arquitectónicos Clave del EPIC-01

1. **Domain-Driven Design (DDD):** Lógica de negocio encapsulada en Aggregate Roots con métodos de intención explícita (ej. `Identity.Deactivate()`, no `identity.Status = DEACTIVATED`).
2. **CQRS (Command Query Responsibility Segregation):** Separación estricta entre el lado de escritura (UC-01, 02, 03, 05) y el lado de lectura optimizado (UC-04) con Cache-Aside y Read Replicas.
3. **Optimistic Concurrency Control (OCC):** Prevención de *Lost Updates* mediante `RowVersion` en todas las operaciones de mutación (UC-02).
4. **Privacy by Design & Crypto-Shredding:** Enmascaramiento automático de PII para Self-Service (UC-04) y anonimización criptográfica irreversible (HMAC-SHA256) durante la purga (UC-05).
5. **Defense-in-Depth:** Validaciones en API Gateway, Domain Services, y Triggers de Base de Datos (ej. bloqueo físico de purga si hay Legal Hold activo).
6. **Event-Driven Architecture:** Uso del **Outbox Pattern** para garantizar la consistencia eventual en la notificación a sistemas downstream (KAGI, KIZUNA, KAGE, TORII) sin comprometer la atomicidad de la transacción principal.

---

## 5. Cómo Navegar este Directorio

Cada Caso de Uso tiene su propio subdirectorio (ej. `UC-02_Actualizar_Identidad/`). Dentro de cada uno, los documentos están numerados del `01` al `12` para facilitar la lectura secuencial.

**Flujo de Lectura Recomendado para Desarrolladores:**
1. Leer `01_Requirements_Specification.md` para entender el "qué" y el "por qué".
2. Revisar `07_Sequence_Diagram.md` y `08_Final_Class_Design.md` para entender el "cómo" a nivel de flujo y código.
3. Consultar `10_UC_Specific_Data_Model.md` y `11_UC_Specific_Security_Threats.md` antes de escribir queries SQL o lógica de seguridad.

**Flujo de Lectura Recomendado para DBAs / SREs:**
1. Ir directamente a `10_UC_Specific_Data_Model.md` para DDL e índices.
2. Revisar `12_UC_Specific_Performance_Benchmarks.md` para SLOs, alertas y tuning.

---

## 6. Dependencias y Sistemas Relacionados

El Motor KAGAMI (EPIC-01) interactúa estrechamente con:
- **KAGI (Authentication Engine):** Valida JWTs, gestiona sesiones y Refresh Tokens (Consumidor de eventos de UC-03 y UC-05).
- **KIZUNA (Authorization Engine):** Evalúa permisos y gestiona caché de autorización (Consumidor de eventos de UC-02, UC-03 y UC-05).
- **KAGE (Audit Engine):** Ingesta y almacena logs de auditoría inmutables (Consumidor de todos los UCs).
- **TORII (Integration/Webhook Engine):** Notifica a sistemas externos (HRIS, SIEM) sobre cambios de ciclo de vida.
- **EPIC-10 (Provisioning Integration):** Futuro hogar de las operaciones de Importación/Exportación masiva (SCIM, CSV, LDAP).

---

## 7. Historial de Versiones del EPIC

| Versión | Fecha | Autor | Descripción de Cambios |
| :--- | :--- | :--- | :--- |
| **1.0.0** | [Fecha Actual] | Arquitecto de Software SHUGO | **Lanzamiento Inicial.** Completitud del 100% de los 60 documentos de diseño para los 5 Casos de Uso del ciclo de vida core. Resolución de la renumeración de UC-05 (Purga) y reubicación de Importar/Exportar al EPIC-10. |

---

## 8. Próximos Pasos (Roadmap)

Con el EPIC-01 completado, las siguientes fases de diseño son:

1. **EPIC-10: Provisioning Integration** 
   - UC-47: Importar/Exportar Identidades (Bulk Operations).
   - UC-48: Sincronización SCIM con HRIS (Workday, BambooHR).
2. **EPIC-02: Authentication & Session Management (Motor KAGI)**
   - UC-06: Autenticar Identidad (Login, MFA, Passwordless).
   - UC-07: Gestionar Sesiones y Refresh Tokens.
3. **Documentos Transversales (Master)**
   - `SHUGO_Data_Model_Master.md`
   - `SHUGO_Security_Architecture.md`
   - `SHUGO_Deployment_Architecture.md`

---

> *"La excelencia en la gestión de identidades no se mide por lo que el sistema permite hacer, sino por lo que garantiza que no se pueda hacer de forma insegura, no auditable o no compliant."* 
> — Principio de Diseño SHUGO
