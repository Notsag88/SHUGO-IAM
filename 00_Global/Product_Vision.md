# SHUGO - Visión del Producto (Product Vision)

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Documento:** | Product_Vision.md |
| **Metodología:** | Product Management Framework + Lean Canvas + OKRs |
| **Estándares:** | ISO/IEC 25010 (Calidad de Software), ISO/IEC 27001 (Seguridad), NIST SP 800-63 (Identidad Digital) |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 2.0.0 |
| **Fecha:** | 30/08/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 27/08/2026 | Gastón Ruiz | Versión inicial con 4 motores (KAGAMI, KAGI, MURA, KIZUNA) y 24 UCs. | Base de la visión del producto. |
| **2.0.0** | 30/08/2026 | Gastón Ruiz | **Refactorización completa.** Expansión a 49 UCs en 15 EPICs. Incorporación de 6 Bounded Contexts (KAGAMI, KAGI, MURA, KIZUNA, KAGE, TORII). Definición de 3 fases de implementación (MVP, Enterprise, Advanced). Actualización de roadmap, métricas de éxito, diferenciadores competitivos, y estrategia de monetización. Alineación con NIST SP 800-63 y ISO 27001. | Documento de referencia estratégica para todo el equipo de SHUGO. Define la visión, misión, estrategia y roadmap de la plataforma IAM. |

---

## 1. Visión del Producto

### 1.1. Declaración de Visión

> **"SHUGO será la plataforma de Identity & Access Management (IAM) más segura, flexible y agnóstica al dominio del mercado, permitiendo que cualquier organización—desde startups hasta enterprises—gestione identidades, autenticación y autorización con un enfoque Zero Trust, Relationship-Based Access Control (ReBAC) y cumplimiento normativo integral, todo en una solución unificada que se adapta a cualquier industria sin modificaciones al core."**

### 1.2. Misión

**Proveer una plataforma IAM enterprise-grade que:**

1. **Centralice la gestión de identidades** de usuarios, servicios y dispositivos en un solo lugar.
2. **Garantice la seguridad** mediante autenticación multi-factor, políticas de contraseña robustas, detección de brechas y acceso privilegiado.
3. **Habilite la autorización granular** mediante una combinación única de RBAC, ReBAC y ABAC que se adapta a cualquier modelo de negocio.
4. **Cumpla con normativas internacionales** (GDPR, CCPA, SOC 2, ISO 27001, NIST) desde el diseño.
5. **Se integre sin fricción** con sistemas existentes mediante estándares abiertos (SCIM, SAML, OIDC, LDAP).
6. **Escale horizontalmente** para soportar millones de identidades y miles de evaluaciones de permisos por segundo.

### 1.3. Valores Fundamentales

| Valor | Descripción | Manifestación en SHUGO |
| :--- | :--- | :--- |
| **Seguridad Ante Todo** | La seguridad no es negociable. Cada decisión de diseño prioriza la protección de datos y la prevención de accesos no autorizados. | Zero Trust por defecto, Fail Secure en todos los endpoints, auditoría completa de todas las acciones. |
| **Agnosticismo de Dominio** | SHUGO no asume ningún modelo de negocio específico. Es una plataforma horizontal que se adapta a cualquier industria. | El motor KIZUNA no conoce conceptos como "veterinario" o "equino". Solo conoce "subject", "relation", "object". |
| **Transparencia Radical** | Todas las decisiones de autorización son explicables y auditables. No hay "cajas negras". | La Check API retorna el camino de evaluación completo. Todos los eventos se registran en el log de auditoría. |
| **Desempeño sin Compromisos** | La seguridad no debe sacrificar la experiencia de usuario. SHUGO es rápido incluso bajo carga extrema. | Latencia p95 < 50ms para la Check API, caché multinivel, optimización de grafos con BFS limitado. |
| **Estándares Abiertos** | Evitar vendor lock-in. SHUGO implementa y promueve estándares abiertos de la industria. | SCIM, SAML, OIDC, OAuth 2.0, JWT, WebAuthn/FIDO2. |
| **Privacidad por Diseño** | La privacidad no es un add-on, es un requisito fundamental desde el primer día. | Consentimiento granular, derecho al olvido, portabilidad de datos, residencia de datos por región. |

---

## 2. Problema que Resuelve

### 2.1. El Problema

Las organizaciones enfrentan desafíos críticos en la gestión de identidades y accesos:

1. **Fragmentación de Sistemas IAM:** Múltiples soluciones puntuales (LDAP para directorio, OAuth para APIs, RBAC custom para aplicaciones) que no se integran entre sí, creando silos de identidad y aumentando la superficie de ataque.

2. **Modelos de Autorización Rígidos:** Los sistemas tradicionales de RBAC no pueden expresar relaciones complejas como "un veterinario solo puede rectificar registros médicos de los equinos que tiene asignados". Se requieren soluciones custom que son difíciles de mantener y auditar.

3. **Falta de Cumplimiento Normativo:** Las normativas como GDPR, CCPA, SOC 2 e ISO 27001 requieren capacidades específicas (consentimiento, derecho al olvido, auditoría completa) que los sistemas IAM tradicionales no ofrecen nativamente.

4. **Escalabilidad Limitada:** Los sistemas IAM legacy no pueden escalar horizontalmente para soportar millones de identidades y miles de evaluaciones de permisos por segundo sin degradar el rendimiento.

5. **Vendor Lock-in:** Las soluciones propietarias dificultan la migración y aumentan los costos a largo plazo. Las organizaciones necesitan soluciones basadas en estándares abiertos.

6. **Falta de Visibilidad:** Sin auditoría completa y trazabilidad end-to-end, es imposible detectar accesos no autorizados, investigar incidentes de seguridad o demostrar cumplimiento ante auditores.

### 2.2. Impacto del Problema

| Impacto | Descripción | Costo Estimado |
| :--- | :--- | :--- |
| **Brechas de Seguridad** | Accesos no autorizados debido a modelos de autorización inadecuados. | $4.45M promedio por brecha (IBM 2023) |
| **Multas Regulatorias** | Sanciones por incumplimiento de GDPR (hasta 4% de ingresos globales). | €20M o 4% de ingresos anuales |
| **Pérdida de Productividad** | Tiempo perdido en gestión manual de accesos y resolución de tickets. | 15-20 horas/mes por administrador |
| **Costos de Integración** | Desarrollo de integraciones custom entre sistemas IAM fragmentados. | $50K-$200K por integración |
| **Riesgo Reputacional** | Pérdida de confianza de clientes y socios tras incidentes de seguridad. | Difícil de cuantificar, pero potencialmente devastador |

### 2.3. Solución que Ofrece SHUGO

SHUGO resuelve estos problemas mediante:

1. **Plataforma Unificada:** Un solo sistema para gestión de identidades, autenticación, autorización, auditoría e integración. Elimina la fragmentación y reduce la superficie de ataque.

2. **Motor de Autorización Híbrido (KIZUNA):** Combina RBAC (roles tradicionales), ReBAC (relaciones entre sujetos y objetos) y ABAC (condiciones contextuales) en un solo motor que puede expresar cualquier modelo de autorización.

3. **Cumplimiento Nativo:** Capacidades integradas de consentimiento, derecho al olvido, portabilidad de datos, residencia de datos y auditoría completa que satisfacen GDPR, CCPA, SOC 2 e ISO 27001.

4. **Arquitectura Escalable:** Diseño distribuido con caché multinivel, particionamiento por tenant y procesamiento asíncrono que soporta millones de identidades y miles de evaluaciones por segundo.

5. **Estándares Abiertos:** Implementación de SCIM, SAML, OIDC, OAuth 2.0, JWT, WebAuthn/FIDO2 que evita vendor lock-in y facilita la integración con sistemas existentes.

6. **Observabilidad Completa:** Trazas distribuidas, métricas en tiempo real, logs estructurados y alertas que proporcionan visibilidad end-to-end del sistema.

---

## 3. Público Objetivo

### 3.1. Segmentos de Mercado

#### **Segmento 1: Startups y Scale-ups Tecnológicas**
- **Tamaño:** 10-500 empleados
- **Necesidades:** IAM rápido de implementar, escalable, basado en estándares abiertos.
- **Dolor:** No tienen recursos para construir un sistema IAM custom, pero necesitan algo más flexible que las soluciones enterprise tradicionales.
- **Presupuesto:** $500-$5,000/mes

#### **Segmento 2: Empresas Medianas en Crecimiento**
- **Tamaño:** 500-5,000 empleados
- **Necesidades:** Cumplimiento normativo (SOC 2, ISO 27001), integración con sistemas existentes (HR, CRM, ERP), autorización granular.
- **Dolor:** Están outgrowing soluciones básicas como Auth0 o Okta Starter, pero no necesitan (o no pueden pagar) soluciones enterprise como Ping Identity o SailPoint.
- **Presupuesto:** $5,000-$50,000/mes

#### **Segmento 3: Enterprises y Organizaciones Reguladas**
- **Tamaño:** 5,000+ empleados
- **Necesidades:** Cumplimiento regulatorio estricto (GDPR, HIPAA, SOX), acceso privilegiado (PAM), integración con directorios legacy (AD/LDAP), residencia de datos.
- **Dolor:** Necesitan una solución que pueda reemplazar múltiples sistemas IAM legacy, proporcionar visibilidad completa y satisfacer auditorías regulatorias.
- **Presupuesto:** $50,000-$500,000+/mes

#### **Segmento 4: Industrias Específicas con Modelos de Autorización Complejos**
- **Ejemplos:** Salud veterinaria (KHCS), e-commerce multi-vendor, SaaS B2B con multi-tenancy, plataformas de educación.
- **Necesidades:** Modelos de autorización que expresen relaciones complejas (ej. "un veterinario solo puede acceder a registros de equinos asignados").
- **Dolor:** Los sistemas IAM tradicionales no pueden expresar estas relaciones sin desarrollo custom extensivo.
- **Presupuesto:** Variable, pero alto valor percibido por la capacidad de ReBAC.

### 3.2. Personas (Buyer Personas)

#### **Persona 1: CTO / VP of Engineering**
- **Objetivos:** Reducir tiempo de desarrollo, mejorar seguridad, cumplir normativas, escalar el negocio.
- **Dolores:** Sistemas IAM fragmentados, falta de visibilidad, riesgo de brechas de seguridad, multas regulatorias.
- **Criterios de Decisión:** Seguridad, escalabilidad, cumplimiento normativo, costo total de propiedad (TCO).
- **Objeciones:** "¿Es seguro?", "¿Escala?", "¿Cumple con SOC 2/ISO 27001?", "¿Cuánto cuesta?"

#### **Persona 2: CISO / Head of Security**
- **Objetivos:** Proteger datos sensibles, prevenir accesos no autorizados, detectar y responder a incidentes, demostrar cumplimiento.
- **Dolores:** Superficie de ataque amplia, falta de visibilidad, incidentes de seguridad, auditorías regulatorias.
- **Criterios de Decisión:** Zero Trust, MFA, auditoría completa, detección de brechas, cumplimiento normativo.
- **Objeciones:** "¿Es realmente Zero Trust?", "¿Qué pasa si hay una brecha?", "¿Cómo demuestro cumplimiento?"

#### **Persona 3: Lead Developer / Architect**
- **Objetivos:** Implementar IAM rápidamente, integrar con sistemas existentes, mantener flexibilidad para cambios futuros.
- **Dolores:** APIs complejas, falta de documentación, vendor lock-in, dificultad para expresar modelos de autorización complejos.
- **Criterios de Decisión:** Calidad de API, documentación, SDKs, estándares abiertos, facilidad de integración.
- **Objeciones:** "¿Qué tan fácil es integrar?", "¿Hay SDKs para mi stack?", "¿Puedo expresar mi modelo de autorización?"

#### **Persona 4: Compliance Officer / DPO**
- **Objetivos:** Cumplir GDPR/CCPA/SOC 2/ISO 27001, gestionar consentimiento, ejercer derecho al olvido, generar reportes de auditoría.
- **Dolores:** Falta de capacidades de consentimiento, dificultad para ejercer derechos de datos, reportes de auditoría manuales.
- **Criterios de Decisión:** Capacidades de consentimiento, derecho al olvido, portabilidad de datos, reportes de auditoría.
- **Objeciones:** "¿Cumple con GDPR?", "¿Cómo manejo el derecho al olvido?", "¿Puedo generar reportes para auditores?"

---

## 4. Propuesta de Valor

### 4.1. Canvas de Propuesta de Valor

| Elemento | Descripción |
| :--- | :--- |
| **Customer Jobs** | - Gestionar identidades de usuarios, servicios y dispositivos<br>- Autenticar usuarios de forma segura<br>- Autorizar accesos granulares<br>- Cumplir normativas (GDPR, SOC 2, ISO 27001)<br>- Integrar con sistemas existentes |
| **Pains** | - Sistemas IAM fragmentados<br>- Modelos de autorización rígidos<br>- Falta de cumplimiento normativo<br>- Escalabilidad limitada<br>- Vendor lock-in<br>- Falta de visibilidad |
| **Gains** | - Plataforma unificada<br>- Autorización híbrida (RBAC + ReBAC + ABAC)<br>- Cumplimiento nativo<br>- Escalabilidad horizontal<br>- Estándares abiertos<br>- Observabilidad completa |
| **Pain Relievers** | - Un solo sistema para todo IAM<br>- Motor KIZUNA que expresa cualquier modelo de autorización<br>- Capacidades integradas de consentimiento, auditoría, derecho al olvido<br>- Arquitectura distribuida con caché multinivel<br>- SCIM, SAML, OIDC, OAuth 2.0<br>- Trazas, métricas, logs, alertas |
| **Gain Creators** | - Reduce complejidad operacional<br>- Permite modelos de autorización complejos sin código custom<br>- Satisface auditorías regulatorias<br>- Soporta millones de identidades<br>- Evita vendor lock-in<br>- Proporciona visibilidad end-to-end |

### 4.2. Diferenciadores Competitivos

| Diferenciador | SHUGO | Competidores (Auth0, Okta, Ping) | Ventaja Competitiva |
| :--- | :--- | :--- | :--- |
| **ReBAC Nativo** | ✅ Motor KIZUNA con grafo de relaciones | ❌ Solo RBAC o ABAC básico | Puede expresar relaciones complejas como "veterinario asignado a equino" sin código custom |
| **Agnosticismo de Dominio** | ✅ Core completamente agnóstico | ⚠️ Algunos tienen verticalizaciones | Se adapta a cualquier industria sin modificaciones al core |
| **Cumplimiento Nativo** | ✅ GDPR, CCPA, SOC 2, ISO 27001 integrados | ⚠️ Requieren add-ons o configuraciones complejas | Cumplimiento desde el día 1 sin costos adicionales |
| **Latencia de Evaluación** | ✅ < 50ms p95 (caché multinivel) | ⚠️ 100-500ms típico | Mejor experiencia de usuario para evaluaciones frecuentes |
| **Precio por Evaluación** | ✅ Modelo basado en evaluaciones de permisos | ⚠️ Modelo basado en MAU (Monthly Active Users) | Más justo para aplicaciones con muchos permisos pero pocos usuarios |
| **Open Source Core** | ✅ Core open source (futuro) | ❌ Propietario | Transparencia, comunidad, sin vendor lock-in |

### 4.3. Unique Value Proposition (UVP)

> **"SHUGO es la única plataforma IAM que combina Relationship-Based Access Control (ReBAC) nativo con cumplimiento normativo integral (GDPR, SOC 2, ISO 27001) en una solución agnóstica al dominio que escala horizontalmente para soportar millones de identidades con latencia < 50ms, todo basado en estándares abiertos para evitar vendor lock-in."**

---

## 5. Estrategia de Producto

### 5.1. Fases de Implementación

#### **Fase 1: MVP (Minimum Viable Product) - Meses 1-6**
**Objetivo:** Lanzar una plataforma IAM funcional con las capacidades core para validar el mercado.

| EPIC | UCs | Prioridad | Descripción |
| :--- | :--- | :--- | :--- |
| EPIC-01 | UC-01 a UC-04 | 🔴 Alta | Identity Management (CRUD básico) |
| EPIC-02 | UC-06 a UC-10 | 🔴 Alta | Authentication (Local, Federada, MFA) |
| EPIC-03 | UC-13, UC-14 | 🔴 Alta | Session Management (básico) |
| EPIC-04 | UC-16, UC-17 | 🔴 Alta | Tenant Management (básico) |
| EPIC-05 | UC-19 a UC-21 | 🔴 Alta | RBAC (roles y permisos) |
| EPIC-06 | UC-23, UC-27, UC-30 | 🔴 Alta | ReBAC Core (namespace, tupla, check) |
| EPIC-07 | UC-31, UC-32 | 🔴 Alta | ABAC (políticas con condiciones) |
| EPIC-08 | UC-33, UC-34 | 🔴 Alta | Password & Security Policies |
| EPIC-13 | UC-46 | 🔴 Alta | Audit Logs (consulta) |
| EPIC-14 | UC-48 | 🔴 Alta | Webhooks (básico) |
| EPIC-15 | UC-49 | 🔴 Alta | API Keys & Rate Limits |

**Total MVP:** 32 UCs de Prioridad Alta

**Criterios de Éxito:**
- ✅ 5 clientes beta activos
- ✅ 10,000 identidades gestionadas
- ✅ 1 millón de evaluaciones de permisos/día
- ✅ Latencia p95 < 100ms para Check API
- ✅ 99.9% de disponibilidad
- ✅ 0 brechas de seguridad

#### **Fase 2: Enterprise Features - Meses 7-12**
**Objetivo:** Agregar capacidades enterprise para satisfacer clientes medianos y grandes.

| EPIC | UCs | Prioridad | Descripción |
| :--- | :--- | :--- | :--- |
| EPIC-01 | UC-05 | 🟡 Media | Import/Export de identidades |
| EPIC-02 | UC-11 | 🟡 Media | Passwordless Authentication |
| EPIC-03 | UC-15 | 🟡 Media | Device Management |
| EPIC-04 | UC-18 | 🟡 Media | Delegated Administration |
| EPIC-05 | UC-22 | 🟡 Media | Access Reviews |
| EPIC-06 | UC-24 a UC-26, UC-28, UC-29 | 🟡 Media | ReBAC completo (CRUD namespaces y tuplas) |
| EPIC-08 | UC-35 | 🟡 Media | Breach Detection |
| EPIC-09 | UC-36 a UC-38 | 🟡 Media | Consent & Privacy (GDPR) |
| EPIC-10 | UC-39 a UC-41 | 🟡 Media | Provisioning & Integration (SCIM, AD/LDAP) |
| EPIC-11 | UC-42 | 🟡 Media | Identity Provider (SAML/OIDC) |
| EPIC-12 | UC-44, UC-45 | 🟡 Media | PAM (Break-Glass, JIT) |
| EPIC-13 | UC-47 | 🟡 Media | Compliance Reports |

**Total Fase 2:** 14 UCs de Prioridad Media

**Criterios de Éxito:**
- ✅ 20 clientes enterprise activos
- ✅ 100,000 identidades gestionadas
- ✅ 10 millones de evaluaciones/día
- ✅ Certificación SOC 2 Type II
- ✅ Cumplimiento GDPR demostrado
- ✅ Integración con 5+ sistemas HR (Workday, BambooHR, etc.)

#### **Fase 3: Advanced Features - Meses 13-18**
**Objetivo:** Agregar capacidades avanzadas para diferenciarse de la competencia.

| EPIC | UCs | Prioridad | Descripción |
| :--- | :--- | :--- | :--- |
| EPIC-02 | UC-12 | 🟢 Baja | Risk-Based Authentication |
| EPIC-11 | UC-43 | 🟢 Baja | Identity Proofing (KYC) |

**Total Fase 3:** 2 UCs de Prioridad Baja + mejoras continuas

**Criterios de Éxito:**
- ✅ 50 clientes enterprise activos
- ✅ 1 millón de identidades gestionadas
- ✅ 100 millones de evaluaciones/día
- ✅ Certificación ISO 27001
- ✅ Integración con 10+ sistemas externos
- ✅ Comunidad open source activa (si aplica)

### 5.2. Roadmap Visual

```
2024 Q1          Q2          Q3          Q4          2025 Q1         Q2
─────────────────────────────────────────────────────────────────────────────
│                                                     │
│  FASE 1: MVP (Meses 1-6)                           │
│  ├─ EPIC-01: Identity Management                   │
│  ├─ EPIC-02: Authentication                        │
│  ├─ EPIC-03: Session Management                    │
│  ├─ EPIC-04: Tenant Management                     │
│  ├─ EPIC-05: RBAC                                  │
│  ├─ EPIC-06: ReBAC Core (3 UCs)                    │
│  ├─ EPIC-07: ABAC                                  │
│  ├─ EPIC-08: Password & Security                   │
│  ├─ EPIC-13: Audit Logs                            │
│  ├─ EPIC-14: Webhooks                              │
│  └─ EPIC-15: API Keys                              │
│                                                     │
│  🎯 Hito: 5 clientes beta, 10K identidades         │
│                                                     │
│                          FASE 2: Enterprise (Meses 7-12)
│                          ├─ EPIC-06: ReBAC completo (5 UCs)
│                          ├─ EPIC-09: Consent & Privacy
│                          ├─ EPIC-10: Provisioning (SCIM, AD)
│                          ├─ EPIC-11: Identity Provider
│                          ├─ EPIC-12: PAM
│                          └─ Otros UCs media prioridad
│                          
│                          🎯 Hito: 20 clientes enterprise
│                                     Certificación SOC 2
│                          
│                                      FASE 3: Advanced (Meses 13-18)
│                                      ├─ Risk-Based Auth
│                                      ├─ Identity Proofing
│                                      └─ Mejoras continuas
│                                      
│                                      🎯 Hito: 50 clientes enterprise
│                                                 Certificación ISO 27001
```

### 5.3. OKRs (Objectives and Key Results)

#### **OKR 1: Validar el Producto en el Mercado (Fase 1)**

**Objective:** Lanzar el MVP y validar el product-market fit con clientes beta.

| Key Result | Target | Medición |
| :--- | :--- | :--- |
| KR1: Adquirir clientes beta | 5 clientes | Número de contratos firmados |
| KR2: Gestionar identidades | 10,000 identidades | Contador en dashboard |
| KR3: Procesar evaluaciones | 1M evaluaciones/día | Métrica de throughput |
| KR4: Mantener latencia | p95 < 100ms | Métrica de latencia |
| KR5: Garantizar disponibilidad | 99.9% uptime | Métrica de disponibilidad |
| KR6: Mantener seguridad | 0 brechas | Incidentes de seguridad |

#### **OKR 2: Escalar a Enterprise (Fase 2)**

**Objective:** Convertir SHUGO en una solución enterprise-grade con certificaciones de compliance.

| Key Result | Target | Medición |
| :--- | :--- | :--- |
| KR1: Adquirir clientes enterprise | 20 clientes | Número de contratos enterprise |
| KR2: Gestionar identidades | 100,000 identidades | Contador en dashboard |
| KR3: Procesar evaluaciones | 10M evaluaciones/día | Métrica de throughput |
| KR4: Obtener certificación SOC 2 | Type II | Auditoría externa |
| KR5: Demostrar cumplimiento GDPR | 100% | Reporte de auditoría |
| KR6: Integrar con sistemas HR | 5+ sistemas | Número de integraciones |

#### **OKR 3: Liderar el Mercado IAM (Fase 3)**

**Objective:** Posicionar a SHUGO como líder en el mercado IAM con capacidades avanzadas y comunidad open source.

| Key Result | Target | Medición |
| :--- | :--- | :--- |
| KR1: Adquirir clientes enterprise | 50 clientes | Número de contratos enterprise |
| KR2: Gestionar identidades | 1M identidades | Contador en dashboard |
| KR3: Procesar evaluaciones | 100M evaluaciones/día | Métrica de throughput |
| KR4: Obtener certificación ISO 27001 | Certificación | Auditoría externa |
| KR5: Integrar con sistemas externos | 10+ sistemas | Número de integraciones |
| KR6: Construir comunidad open source | 1,000+ stars | GitHub stars |

---

## 6. Estrategia de Monetización

### 6.1. Modelo de Pricing

SHUGO adopta un modelo de pricing basado en **consumo de evaluaciones de permisos** (no en MAU como Auth0/Okta), lo cual es más justo para aplicaciones con muchos permisos pero pocos usuarios.

#### **Plan Free (Starter)**
- **Precio:** $0/mes
- **Incluye:**
  - Hasta 1,000 identidades
  - Hasta 100,000 evaluaciones de permisos/mes
  - Authentication local + 1 IdP federado
  - RBAC básico
  - ReBAC básico (3 namespaces)
  - Audit logs (7 días de retención)
  - Community support

#### **Plan Professional**
- **Precio:** $500/mes + $0.001 por evaluación adicional
- **Incluye:**
  - Hasta 10,000 identidades
  - Hasta 10 millones de evaluaciones/mes
  - Authentication local + IdPs federados ilimitados
  - MFA (TOTP, SMS, Email)
  - RBAC + ReBAC completo
  - ABAC básico
  - Audit logs (30 días de retención)
  - Webhooks
  - Email support (24h response)

#### **Plan Enterprise**
- **Precio:** $5,000/mes + $0.0005 por evaluación adicional
- **Incluye:**
  - Identidades ilimitadas
  - Evaluaciones ilimitadas
  - Todas las capacidades de Professional +
  - Passwordless Authentication
  - SCIM Provisioning
  - Directory Sync (AD/LDAP)
  - Identity Provider (SAML/OIDC)
  - PAM (Break-Glass, JIT)
  - Consent Management (GDPR)
  - Audit logs (1 año de retención)
  - Compliance Reports
  - Priority support (4h response)
  - Dedicated account manager

#### **Plan Custom (Ultra-Enterprise)**
- **Precio:** Custom (starting at $50,000/mes)
- **Incluye:**
  - Todas las capacidades de Enterprise +
  - Data residency por región
  - Identity Proofing (KYC)
  - Risk-Based Authentication
  - Custom integrations
  - On-premise deployment option
  - 24/7 phone support
  - SLA 99.99%
  - Dedicated engineering team

### 6.2. Proyección de Ingresos

| Año | Clientes | ARR (Annual Recurring Revenue) | GMV (Gross Merchandise Value) |
| :--- | :--- | :--- | :--- |
| **Año 1** | 50 (5 Free, 30 Pro, 15 Enterprise) | $1.2M | $1.2M |
| **Año 2** | 200 (20 Free, 100 Pro, 70 Enterprise, 10 Custom) | $8.5M | $8.5M |
| **Año 3** | 500 (50 Free, 250 Pro, 180 Enterprise, 20 Custom) | $25M | $25M |

### 6.3. Unit Economics

| Métrica | Target | Justificación |
| :--- | :--- | :--- |
| **CAC (Customer Acquisition Cost)** | $5,000 (Pro), $50,000 (Enterprise) | Marketing + Sales + Onboarding |
| **LTV (Lifetime Value)** | $60,000 (Pro), $600,000 (Enterprise) | ARPU × Gross Margin × Avg Lifespan |
| **LTV:CAC Ratio** | 12:1 (Pro), 12:1 (Enterprise) | Saludable (> 3:1 es bueno) |
| **Gross Margin** | 80% | Típico de SaaS |
| **Net Revenue Retention** | 120% | Upsell + Expansion - Churn |
| **Payback Period** | 6 meses | Tiempo para recuperar CAC |

---

## 7. Análisis de Mercado

### 7.1. Tamaño del Mercado

| Segmento | TAM (Total Addressable Market) | SAM (Serviceable Addressable Market) | SOM (Serviceable Obtainable Market) |
| :--- | :--- | :--- | :--- |
| **Global IAM Market** | $25B (2025) | $5B (Mid-market + Enterprise) | $500M (5% de SAM en 3 años) |
| **ReBAC Market** | $500M (emergente) | $100M (early adopters) | $10M (10% de SAM en 3 años) |
| **Compliance IAM** | $2B (GDPR, SOC 2) | $400M (regulated industries) | $40M (10% de SAM en 3 años) |

### 7.2. Competencia

| Competidor | Fortaleza | Debilidad | Diferenciador SHUGO |
| :--- | :--- | :--- | :--- |
| **Auth0 (Okta)** | Brand recognition, ecosystem | Solo RBAC/ABAC, caro, vendor lock-in | ReBAC nativo, precio justo, open source |
| **Okta** | Enterprise features, integrations | Complejo, caro, no ReBAC | Simplicidad, ReBAC, precio |
| **Ping Identity** | Enterprise-grade, compliance | Legacy, difícil de integrar | Moderno, API-first, fácil integración |
| **Keycloak (Red Hat)** | Open source, gratis | Difícil de operar, no ReBAC | Managed service, ReBAC, mejor UX |
| **Oso (ReBAC)** | ReBAC nativo | Solo autorización, no IAM completo | IAM completo + ReBAC |
| **Permit.io** | ReBAC + ABAC | No es IAM completo, pricing complejo | IAM completo, pricing simple |

### 7.3. Tendencias del Mercado

1. **Zero Trust Architecture:** Adopción masiva de modelos Zero Trust que requieren autorización granular y continua.
2. **Relationship-Based Access Control (ReBAC):** Crecimiento de soluciones ReBAC (Oso, Permit.io) para expresar relaciones complejas.
3. **Passwordless Authentication:** Movimiento hacia autenticación sin contraseña (WebAuthn/FIDO2, magic links).
4. **Compliance Automation:** Demanda de soluciones que automaticen el cumplimiento normativo (GDPR, SOC 2, ISO 27001).
5. **Identity for Machines:** Gestión de identidades de servicios, APIs y dispositivos IoT, no solo humanos.
6. **Consolidation of IAM:** Tendencia a consolidar múltiples soluciones IAM en una sola plataforma unificada.

---

## 8. Riesgos y Mitigaciones

### 8.1. Riesgos Técnicos

| Riesgo | Probabilidad | Impacto | Mitigación |
| :--- | :--- | :--- | :--- |
| **Escalabilidad del grafo de relaciones** | Media | Alto | Caché multinivel, particionamiento por tenant, BFS limitado a 5 saltos |
| **Latencia de evaluación > 50ms** | Baja | Alto | Optimización de queries, índices en PostgreSQL, read replicas |
| **Brecha de seguridad** | Baja | Crítico | Zero Trust, auditoría completa, penetration testing trimestral |
| **Vendor lock-in de infraestructura** | Media | Medio | Multi-cloud design, estándares abiertos, abstracción de infraestructura |

### 8.2. Riesgos de Mercado

| Riesgo | Probabilidad | Impacto | Mitigación |
| :--- | :--- | :--- | :--- |
| **Competidores lanzzan ReBAC** | Alta | Medio | First-mover advantage, comunidad open source, pricing competitivo |
| **Adopción lenta de ReBAC** | Media | Alto | Educación del mercado, casos de uso claros, migración gradual desde RBAC |
| **Cambios regulatorios** | Media | Alto | Compliance team dedicado, monitoreo regulatorio, arquitectura flexible |
| **Consolidación de competidores** | Media | Medio | Diferenciación clara, enfoque en nichos específicos, partnerships estratégicos |

### 8.3. Riesgos de Negocio

| Riesgo | Probabilidad | Impacto | Mitigación |
| :--- | :--- | :--- | :--- |
| **Churn de clientes enterprise** | Baja | Alto | Customer success team, onboarding dedicado, roadmap transparente |
| **Dificultad para contratar talento** | Alta | Medio | Remote-first, compensación competitiva, cultura técnica fuerte |
| **Cash flow negativo prolongado** | Media | Alto | Funding adecuado, control de costos, focus en unit economics saludables |

---

## 9. Stakeholders y Equipo

### 9.1. Stakeholders Internos

| Stakeholder | Rol | Responsabilidades | Interés |
| :--- | :--- | :--- | :--- |
| **CEO / Founder** | Estrategia y visión | Definir dirección, fundraising, partnerships | Alto |
| **CTO** | Arquitectura técnica | Decisiones técnicas, calidad de código, escalabilidad | Alto |
| **CISO** | Seguridad y compliance | Seguridad, certificaciones, cumplimiento normativo | Alto |
| **VP of Product** | Gestión de producto | Roadmap, priorización, customer discovery | Alto |
| **VP of Engineering** | Ejecución técnica | Entrega de features, calidad, velocidad | Alto |
| **VP of Sales** | Ventas y revenue | Adquisición de clientes, revenue, partnerships | Alto |
| **VP of Customer Success** | Retención de clientes | Onboarding, soporte, retention, expansion | Medio |
| **Head of Marketing** | Marketing y brand | Brand awareness, lead generation, content | Medio |

### 9.2. Stakeholders Externos

| Stakeholder | Rol | Interés | Influencia |
| :--- | :--- | :--- | :--- |
| **Clientes Beta** | Early adopters | Alto | Alto |
| **Clientes Enterprise** | Revenue principal | Alto | Alto |
| **Inversores** | Funding y guidance | Alto | Alto |
| **Partners tecnológicos** | Integraciones y co-marketing | Medio | Medio |
| **Comunidad open source** | Contribuciones y advocacy | Medio | Medio |
| **Auditores (SOC 2, ISO)** | Certificaciones | Bajo | Alto |
| **Reguladores (GDPR, etc.)** | Cumplimiento normativo | Bajo | Alto |

### 9.3. Equipo Requerido (Fase 1)

| Rol | Cantidad | Responsabilidades |
| :--- | :--- | :--- |
| **Backend Engineers (Senior)** | 3 | Core IAM, KIZUNA, APIs |
| **Backend Engineers (Mid)** | 2 | Features, integrations, testing |
| **Frontend Engineer** | 1 | Admin portal, dashboards |
| **DevOps Engineer** | 1 | Infrastructure, CI/CD, monitoring |
| **Security Engineer** | 1 | Security reviews, penetration testing |
| **Product Manager** | 1 | Roadmap, prioritization, customer discovery |
| **Designer (UX/UI)** | 0.5 | Admin portal UX, documentation |
| **QA Engineer** | 1 | Testing, quality assurance |
| **Technical Writer** | 0.5 | Documentation, API docs |
| **Customer Success** | 1 | Onboarding, support, feedback |

**Total:** 11 FTEs (Full-Time Equivalents)

---

## 10. Métricas de Éxito (KPIs)

### 10.1. Métricas de Producto

| Métrica | Target (Año 1) | Target (Año 2) | Target (Año 3) | Fuente |
| :--- | :--- | :--- | :--- | :--- |
| **MAU (Monthly Active Users)** | 5,000 | 50,000 | 200,000 | Analytics |
| **Identidades gestionadas** | 10,000 | 100,000 | 1,000,000 | Database |
| **Evaluaciones de permisos/día** | 1M | 10M | 100M | Metrics |
| **Latencia p95 (Check API)** | < 100ms | < 50ms | < 50ms | Metrics |
| **Disponibilidad** | 99.9% | 99.95% | 99.99% | Monitoring |

### 10.2. Métricas de Negocio

| Métrica | Target (Año 1) | Target (Año 2) | Target (Año 3) | Fuente |
| :--- | :--- | :--- | :--- | :--- |
| **ARR (Annual Recurring Revenue)** | $1.2M | $8.5M | $25M | Finance |
| **Número de clientes** | 50 | 200 | 500 | CRM |
| **NRR (Net Revenue Retention)** | 110% | 120% | 125% | Finance |
| **CAC Payback Period** | 8 meses | 6 meses | 6 meses | Finance |
| **Gross Margin** | 75% | 80% | 82% | Finance |

### 10.3. Métricas de Seguridad y Compliance

| Métrica | Target (Año 1) | Target (Año 2) | Target (Año 3) | Fuente |
| :--- | :--- | :--- | :--- | :--- |
| **Brechas de seguridad** | 0 | 0 | 0 | Security |
| **Certificaciones** | - | SOC 2 Type II | ISO 27001 | Audit |
| **Tiempo de respuesta a incidentes** | < 1 hora | < 30 min | < 15 min | Security |
| **Cobertura de tests de seguridad** | 80% | 90% | 95% | Security |

### 10.4. Métricas de Satisfacción del Cliente

| Métrica | Target (Año 1) | Target (Año 2) | Target (Año 3) | Fuente |
| :--- | :--- | :--- | :--- | :--- |
| **NPS (Net Promoter Score)** | 40+ | 50+ | 60+ | Surveys |
| **CSAT (Customer Satisfaction)** | 4.0/5 | 4.5/5 | 4.5/5 | Surveys |
| **Churn rate (anual)** | < 10% | < 8% | < 5% | CRM |
| **Tiempo de onboarding** | < 1 semana | < 3 días | < 2 días | Customer Success |

---

## 11. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `Domain_Analysis.md` | Define los Bounded Contexts y Aggregate Roots que este documento materializa en features. |
| `System_Architecture_Overview.md` | Traduce la visión de producto en arquitectura técnica. |
| `KIZUNA_Engine_Architecture.md` | Detalla la arquitectura del motor de autorización, diferenciador clave de SHUGO. |
| `Security_Threat_Model.md` | Analiza las amenazas que las features de este roadmap deben mitigar. |
| `Glossary.md` | Contiene definiciones de términos usados en este documento. |
| `KHCS_SHUGO_Integration_Contract.md` | Primer caso de uso real que valida la visión de producto. |
| Cada documento de UC | Implementa las features definidas en el roadmap de este documento. |

---

## 12. Anexos

### 12.1. Glosario de Términos de Negocio

| Término | Definición |
| :--- | :--- |
| **ARR (Annual Recurring Revenue)** | Ingresos recurrentes anuales normalizados. |
| **CAC (Customer Acquisition Cost)** | Costo de adquirir un cliente (marketing + sales + onboarding). |
| **LTV (Lifetime Value)** | Valor total que un cliente genera durante su vida útil. |
| **NRR (Net Revenue Retention)** | Porcentaje de revenue retenido después de expansiones y churn. |
| **GMV (Gross Merchandise Value)** | Valor total de transacciones procesadas. |
| **TAM (Total Addressable Market)** | Mercado total disponible para el producto. |
| **SAM (Serviceable Addressable Market)** | Segmento del TAM que el producto puede servir. |
| **SOM (Serviceable Obtainable Market)** | Porción del SAM que se puede capturar realistically. |

### 12.2. Referencias

1. **NIST SP 800-63:** Digital Identity Guidelines - https://pages.nist.gov/800-63-3/
2. **ISO/IEC 27001:** Information Security Management - https://www.iso.org/standard/27001
3. **GDPR:** General Data Protection Regulation - https://gdpr.eu/
4. **SOC 2:** Service Organization Control 2 - https://www.aicpa.org/
5. **OAuth 2.0 (RFC 6749):** https://tools.ietf.org/html/rfc6749
6. **OpenID Connect:** https://openid.net/connect/
7. **SCIM (RFC 7643/7644):** https://scim.dev/
8. **SAML 2.0:** https://docs.oasis-open.org/security/saml/v2.0/
9. **WebAuthn / FIDO2:** https://www.w3.org/TR/webauthn/

---
*Fin del Documento Product_Vision v2.0 para SHUGO*