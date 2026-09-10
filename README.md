#  SHUGO - Enterprise Identity & Access Management Platform

**Secure Human User & Group Orchestration**

---

## 📑 Tabla de Contenidos

- [ Visión del Producto](#-visión-del-producto)
- [🏗️ Arquitectura del Sistema](#️-arquitectura-del-sistema)
- [📂 Estructura del Proyecto](#-estructura-del-proyecto)
- [ EPICs y Casos de Uso](#-epics-y-casos-de-uso)
- [⚙️ Motores Core](#️-motores-core)
- [📐 Estándar de Diseño (12 Documentos)](#-estándar-de-diseño-12-documentos)
- [🚀 Quick Start](#-quick-start)

---

##  Visión del Producto

**SHUGO** es una plataforma enterprise de **Identity & Access Management (IAM)** diseñada para organizaciones que exigen seguridad de grado bancario, cumplimiento normativo estricto y una experiencia de usuario fluida.

**Capacidades Clave:**
- **Ciclo de Vida Completo:** Desde el registro self-service hasta la purga criptográfica (GDPR Art. 17).
- **Autenticación Adaptativa:** Soporte nativo para MFA, Passwordless (FIDO2/WebAuthn) y Risk-Based Authentication.
- **Autorización Granular:** Modelos híbridos RBAC, ReBAC (KIZUNA Core) y ABAC.
- **Compliance by Design:** Auditoría inmutable, Legal Holds, y Crypto-Shredding nativo.

---

## 🏗️ Arquitectura del Sistema

SHUGO implementa una arquitectura **Domain-Driven Design (DDD)** con separación estricta de Bounded Contexts y principios de **Clean Architecture**.

### Principios Fundamentales
1. **CQRS (Command Query Responsibility Segregation):** Separación total entre el lado de escritura (comandos) y el lado de lectura (queries optimizadas con Cache-Aside).
2. **Outbox Pattern:** Garantía de entrega de eventos de dominio a sistemas downstream (KAGI, KIZUNA, KAGE) sin comprometer la atomicidad transaccional.
3. **Optimistic Concurrency Control (OCC):** Prevención de *Lost Updates* mediante `RowVersion` en todas las mutaciones de estado.
4. **Defense-in-Depth:** Validaciones en API Gateway, Domain Services, y Triggers de Base de Datos.

---

## 📂 Estructura del Proyecto

```text
SHUGO/
├── 00_Global/                              # Documentación transversal y arquitectura
├── 01_EPIC_Identity_Management/            # Ciclo de vida de identidades (UC-01 a UC-10)
├── 02_EPIC_Authentication/                 # Autenticación y MFA (UC-11 a UC-19)
├── 03_EPIC_Session_Management/             # Gestión de sesiones (UC-20 a UC-23)
├── 04_EPIC_Tenant_Management/              # Multi-tenancy (UC-24 a UC-26)
├── 05_EPIC_RBAC/                           # Role-Based Access Control (UC-27 a UC-30)
├── 06_EPIC_ReBAC_KIZUNA_Core/              # Relationship-Based Access Control (UC-31 a UC-38)
├── 07_EPIC_ABAC/                           # Attribute-Based Access Control (UC-39 a UC-40)
├── 08_EPIC_Password_Security_Policies/     # Políticas de password y lockout (UC-41 a UC-43)
├── 09_EPIC_Consent_Privacy/                # Consentimiento y privacidad (UC-44 a UC-46)
├── 10_EPIC_Provisioning_Integration/       # Integración externa (UC-47 a UC-50)
├── 11_EPIC_Identity_Provider/              # Identity Provider (UC-51 a UC-52)
├── 12_EPIC_Privileged_Access_Management/   # PAM (UC-53 a UC-55)
├── 13_EPIC_Audit_Compliance/               # Auditoría y compliance (UC-56 a UC-57)
├── 14_EPIC_Webhooks_Notifications/         # Webhooks (UC-58)
├── 15_EPIC_API_Management/                 # API Management (UC-59 a UC-61)
└── 16_Integration_Examples/                # Ejemplos de integración
```

---

## 📊 EPICs y Casos de Uso

### 🟢 EPIC-01: Identity Management (10 UCs)
| UC | Nombre del Caso de Uso | Docs | Estado |
| :--- | :--- | :---: | :--- |
| **UC-01** | Provisionar Identidad | 12/12 | ✅ Completo |
| **UC-02** | Actualizar Identidad | 12/12 | ✅ Completo |
| **UC-03** | Desactivar Identidad | 12/12 | ✅ Completo |
| **UC-04** | Consultar Identidades | 12/12 | ✅ Completo |
| **UC-05** | Purgar Identidad | 12/12 | ✅ Completo |
| **UC-06** | Registro Self-Service | 12/12 | ✅ Completo |
| **UC-07** | Invitación de Usuarios | 12/12 | ✅ Completo |
| **UC-08** | Self-Service Profile | 12/12 | ✅ Completo |
| **UC-09** | Verificación de Contactos | 12/12 | ✅ Completo |
| **UC-10** | Import/Export Masivo | 12/12 | ✅ Completo |

### 🟢 EPIC-02: Authentication (9 UCs)
| UC | Nombre del Caso de Uso | Docs | Estado |
| :--- | :--- | :---: | :--- |
| **UC-11** | Autenticar Credenciales Locales | 12/12 | ✅ Completo |
| **UC-12** | Cerrar Sesión (Logout) | 12/12 | ✅ Completo |
| **UC-13** | Renovar Sesión (Refresh Token) | 12/12 | ✅ Completo |
| **UC-14** | Autenticar Federada (SSO) | 12/12 | ✅ Completo |
| **UC-15** | Multi-Factor Authentication (MFA) | 12/12 | ✅ Completo |
| **UC-16** | Passwordless Authentication | 12/12 | ✅ Completo |
| **UC-17** | Risk-Based Authentication | 12/12 | ✅ Completo |
| **UC-18** | Recuperación de Credenciales | 12/12 | ✅ Completo |
| **UC-19** | Cambio de Password (Self-Service) | 12/12 | ✅ Completo |

### 🟡 EPIC-03 a EPIC-16 (En Desarrollo)
*Nota: Los siguientes EPICs cuentan con la estructura base (01-09) y están en proceso de completarse con los documentos técnicos (10-12).*

| EPIC | Nombre | Rango UCs | Total UCs |
| :--- | :--- | :--- | :---: |
| **03** | Session Management | UC-20 a UC-23 | 4 |
| **04** | Tenant Management | UC-24 a UC-26 | 3 |
| **05** | RBAC | UC-27 a UC-30 | 4 |
| **06** | ReBAC (KIZUNA Core) | UC-31 a UC-38 | 8 |
| **07** | ABAC | UC-39 a UC-40 | 2 |
| **08** | Password Security Policies | UC-41 a UC-43 | 3 |
| **09** | Consent & Privacy | UC-44 a UC-46 | 3 |
| **10** | Provisioning Integration | UC-47 a UC-50 | 4 |
| **11** | Identity Provider | UC-51 a UC-52 | 2 |
| **12** | Privileged Access Management | UC-53 a UC-55 | 3 |
| **13** | Audit & Compliance | UC-56 a UC-57 | 2 |
| **14** | Webhooks & Notifications | UC-58 | 1 |
| **15** | API Management | UC-59 a UC-61 | 3 |
| **16** | Integration Examples | - | - |

**Total Global:** 61 Casos de Uso distribuidos en 16 EPICs.

---

## ⚙️ Motores Core

SHUGO se compone de 6 motores especializados que interactúan mediante eventos de dominio:

1. **KAGAMI (Identity Lifecycle Engine):** Source of Truth para el ciclo de vida de identidades. Maneja provisioning, mutaciones, y purgas con crypto-shredding.
2. **KAGI (Authentication Engine):** Gestiona credenciales, emisión de tokens (JWT), MFA, y sesiones.
3. **KIZUNA (Authorization Engine):** Motor de autorización basado en relaciones (ReBAC) y grafos, complementado por RBAC y ABAC.
4. **KAGE (Audit Engine):** Ingesta y almacena logs de auditoría inmutables (WORM storage) para compliance forense.
5. **MURA (RBAC Engine):** Gestión de roles, permisos y revisiones de acceso (Access Reviews).
6. **TORII (Integration Engine):** Orquestación de webhooks, notificaciones y sincronización con sistemas externos (SCIM, SIEM).

---

##  Estándar de Diseño (12 Documentos)

Cada Caso de Uso (UC) en SHUGO sigue un rigoroso estándar de diseño de **12 documentos**, dividido en dos fases:

### Fase 1: Diseño Funcional y de Dominio (01-09)
1. `01_Requirements_Specification.md` - Requisitos analíticos y criterios BDD (Gherkin).
2. `02_Use_Case_Diagram.md` - Límites del sistema y actores (UML).
3. `03_Activity_Diagram.md` - Flujos de trabajo y carriles de responsabilidad.
4. `04_Initial_Domain_Model.md` - Aggregate Roots, Entities y Value Objects (DDD).
5. `05_State_Machine_Diagram.md` - Transiciones de estado y guardas.
6. `06_Robustness_Diagram.md` - Validación de acoplamiento (ICONIX).
7. `07_Sequence_Diagram.md` - Orden temporal de interacciones.
8. `08_Final_Class_Design.md` - Firmas de métodos y patrones de diseño.
9. `09_Modern_Artifacts.md` - Contratos OpenAPI, métricas y Feature Flags.

### Fase 2: Diseño Técnico Específico (10-12)
10. `10_UC_Specific_Data_Model.md` - DDL, índices, RLS y estrategias de almacenamiento.
11. `11_UC_Specific_Security_Threats.md` - Threat Modeling (STRIDE/DREAD) y mitigaciones.
12. `12_UC_Specific_Performance_Benchmarks.md` - SLIs, SLOs y estrategias de Load Testing.

---

## 🚀 Quick Start

### Para Desarrolladores Backend
1. Dirígete al EPIC correspondiente a tu tarea.
2. Lee el `01_Requirements_Specification.md` para entender el "qué" y el "por qué".
3. Revisa el `07_Sequence_Diagram.md` y `08_Final_Class_Design.md` para la implementación.
4. Consulta el `10_UC_Specific_Data_Model.md` antes de escribir queries SQL.

### Para Arquitectos y DBAs
1. Revisa la carpeta `00_Global/` para la visión macro.
2. Analiza los `12_UC_Specific_Performance_Benchmarks.md` para entender los SLAs.
3. Revisa los `11_UC_Specific_Security_Threats.md` para las mitigaciones de infraestructura.

### Para Equipos de Seguridad y Compliance
1. Enfócate en los UCs de **EPIC-01** (Purga/Desactivación) y **EPIC-13** (Auditoría).
2. Revisa los Threat Models (`11_UC_Specific_Security_Threats.md`) de los UCs de autenticación (EPIC-02).

---

*Documento Maestro de SHUGO IAM. Última actualización: Septiembre 2026.*