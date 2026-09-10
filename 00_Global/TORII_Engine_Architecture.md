# SHUGO - TORII Engine Architecture (Motor de Integración, API Gateway y Proveedores de Identidad)

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Documento:** | TORII_Engine_Architecture.md |
| **Metodología:** | Domain-Driven Design + Anti-Corruption Layer (ACL) + API Management |
| **Estándares:** | SCIM 2.0 (RFC 7643/7644), SAML 2.0, OpenID Connect (OIDC) 1.0, OAuth 2.0 (RFC 6749), HMAC SHA256 |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 30/08/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 30/08/2026 | Gastón Ruiz | **Creación inicial.** Arquitectura detallada del motor TORII. Cubre 7 UCs (UC-39 a UC-43, UC-48, UC-49) enfocados en integración externa (SCIM, SAML, OIDC), gestión de Webhooks, y control de acceso a la API (API Keys, Rate Limiting). Incluye el patrón Anti-Corruption Layer (ACL) y el motor de despacho asíncrono de eventos. | Documento de referencia técnica para la capa de integración y gateway de SHUGO. Define cómo el sistema interactúa de forma segura y estandarizada con el ecosistema externo. |

---

## 1. Introducción y Propósito

TORII (puerta o entrada en japonés) es el **motor de Integración, API Management y Proveedores de Identidad (IdP)** de SHUGO. Actúa como la "aduana" y el "embajador" de la plataforma: traduce protocolos externos al modelo de dominio interno (ACL) y expone las capacidades de SHUGO al mundo exterior mediante APIs seguras y estandarizadas.

TORII es crítico porque:
- **Habilita el Ecosistema:** Permite que SHUGO se integre con cualquier HRIS (Workday, BambooHR), IdP corporativo (Azure AD, Okta) o aplicación cliente.
- **Protege la Plataforma:** Gestiona el acceso programático mediante API Keys y Rate Limiting, previniendo abusos y DDoS.
- **Actúa como IdP:** Permite que aplicaciones de terceros confíen en SHUGO para autenticar a sus usuarios mediante SAML u OIDC.

### 1.1. Alcance

TORII cubre **7 Casos de Uso** organizados en 4 EPICs:

| EPIC | UCs | Descripción |
| :--- | :--- | :--- |
| **EPIC-10: Provisioning & Integration** | UC-39 a UC-41 | Sincronización SCIM, Directorios (AD/LDAP), Mapeo de Claims. |
| **EPIC-11: Identity Provider** | UC-42, UC-43 | SHUGO como IdP (SAML/OIDC), Identity Proofing (KYC). |
| **EPIC-14: Webhooks & Notifications** | UC-48 | Gestión y despacho de Webhooks para eventos de dominio. |
| **EPIC-15: API Management** | UC-49 | API Keys, Rate Limiting, Throttling. |

### 1.2. Objetivos de Rendimiento y Seguridad

| Métrica | Objetivo | Justificación |
| :--- | :--- | :--- |
| **Webhook Delivery Latency** | < 100ms (Queue time) | Despacho asíncrono para no bloquear el flujo principal. |
| **Webhook Reliability** | 99.9% (con reintentos) | Garantizar entrega mediante Exponential Backoff y DLQ. |
| **Rate Limiter Overhead** | < 2ms por request | El Rate Limiting no debe ser el cuello de botella de la API. |
| **IdP Token Generation** | < 50ms p95 | Generación rápida de SAML Assertions y OIDC ID Tokens. |
| **API Key Lookup** | < 5ms | Búsqueda rápida por prefijo y validación de hash. |

---

## 2. Arquitectura General de TORII

### 2.1. Vista de Alto Nivel

```plantuml
@startuml TORII_Architecture_Overview
skinparam componentStyle rectangle

title TORII Engine - Arquitectura de Alto Nivel

' ============================================
' EXTERNAL ACTORS
' ============================================
actor "Client Apps" as apps
actor "HR Systems\n(Workday, BambooHR)" as hr
actor "External IdPs\n(Azure AD, Okta)" as ext_idp

' ============================================
' API LAYER
' ============================================
package "API Layer (Gateway)" {
    [Public API\n(REST/GraphQL)] as PublicAPI
    [IdP Endpoints\n(OIDC/SAML)] as IdPEndpoints
    [SCIM Endpoint\n(RFC 7644)] as SCIMEndpoint
}

' ============================================
' APPLICATION LAYER
' ============================================
package "Application Layer" {
    [API Key Manager\n(UC-49)] as APIKeyManager
    [Rate Limiter\n(Sliding Window)] as RateLimiter
    [Webhook Dispatcher\n(UC-48)] as WebhookDispatcher
    [SCIM Adapter\n(UC-39)] as SCIMAdapter
    [IdP Engine\n(UC-42)] as IdPEngine
}

' ============================================
' DOMAIN LAYER (ACLs)
' ============================================
package "Domain Layer (Anti-Corruption Layers)" {
    [SCIM Translator\n(JSON -> Domain)] as SCIMTranslator
    [SAML/OIDC Builder\n(Domain -> XML/JWT)] as ProtocolBuilder
    [Claims Mapper\n(UC-41)] as ClaimsMapper
}

' ============================================
' INFRASTRUCTURE LAYER
' ============================================
package "Infrastructure Layer" {
    database "PostgreSQL\n(Configs, Deliveries)" as Postgres
    database "Redis\n(Rate Limits, Caches)" as Redis
    queue "Kafka\n(Outbound Events)" as Kafka
    entity "KAGAMI\n(Identities)" as KAGAMI
    entity "KAGE\n(Audit)" as KAGE
}

' ============================================
' RELACIONES
' ============================================
apps --> PublicAPI : HTTPS + API Key
hr --> SCIMEndpoint : HTTPS + Bearer Token
ext_idp --> IdPEndpoints : SAML/OIDC Flows

PublicAPI --> APIKeyManager : Validate Key
PublicAPI --> RateLimiter : Check Limit

WebhookDispatcher --> Kafka : Consume Events
WebhookDispatcher --> Postgres : Log Delivery

SCIMEndpoint --> SCIMAdapter
SCIMAdapter --> SCIMTranslator
SCIMTranslator --> KAGAMI : Provision/Update Identity

IdPEndpoints --> IdPEngine
IdPEngine --> ProtocolBuilder
ProtocolBuilder --> KAGAMI : Fetch User Attributes

Kafka --> WebhookDispatcher : Trigger Webhooks
WebhookDispatcher --> KAGE : Log Delivery Status

@enduml
```

### 2.2. Componentes Principales

| Componente | Responsabilidad | UCs Asociados |
| :--- | :--- | :--- |
| **API Key Manager** | Generación, rotación, hashing y validación de claves de API. | UC-49 |
| **Rate Limiter** | Algoritmo de Sliding Window en Redis para controlar throughput. | UC-49 |
| **Webhook Dispatcher** | Servicio asíncrono que consume eventos de Kafka y los entrega a endpoints externos con reintentos. | UC-48 |
| **SCIM Adapter** | Traduce payloads SCIM 2.0 (User/Group) al modelo de dominio de KAGAMI. | UC-39 |
| **IdP Engine** | Genera SAML Assertions y OIDC ID Tokens. Gestiona JWKS (JSON Web Key Set). | UC-42 |
| **Claims Mapper** | Transforma atributos internos de SHUGO a claims estándar (email, name, roles) para aplicaciones cliente. | UC-41 |

---

## 3. Modelo de Datos de TORII

### 3.1. Diagrama de Modelo de Datos

```plantuml
@startuml TORII_Data_Model
skinparam classAttributeIconSize 0

title TORII - Modelo de Datos (PostgreSQL)

class api_keys {
    id : UUID
    tenant_id : UUID
    name : VARCHAR
    prefix : VARCHAR -- e.g., "shugo_sk_live_abc123"
    key_hash : VARCHAR -- SHA256 hash of the full key
    scopes : JSONB -- e.g., ["read:identities", "write:authz"]
    rate_limit_tier : VARCHAR -- e.g., "enterprise", "standard"
    last_used_at : TIMESTAMP
    expires_at : TIMESTAMP
    status : VARCHAR -- ACTIVE, REVOKED
    --
    PK id
}

class webhook_subscriptions {
    id : UUID
    tenant_id : UUID
    endpoint_url : VARCHAR
    secret : VARCHAR -- HMAC SHA256 secret
    events : JSONB -- e.g., ["identity.created", "authz.denied"]
    status : VARCHAR -- ACTIVE, PAUSED, FAILED
    --
    PK id
}

class webhook_deliveries {
    id : UUID
    subscription_id : UUID
    event_type : VARCHAR
    payload : JSONB
    status : VARCHAR -- PENDING, SUCCESS, FAILED
    http_status : INT
    attempts : INT
    next_retry_at : TIMESTAMP
    --
    PK id
}

class idp_configurations {
    id : UUID
    tenant_id : UUID
    protocol : VARCHAR -- SAML, OIDC
    client_id : VARCHAR
    client_secret_hash : VARCHAR
    redirect_uris : JSONB
    claims_mapping : JSONB
    --
    PK id
}

api_keys ||--o{ webhook_subscriptions : "owned by tenant"
webhook_subscriptions ||--o{ webhook_deliveries : "generates"

note right of api_keys
  **Seguridad de API Keys**
  Nunca se almacena la clave completa.
  Solo el prefijo (para lookup) y el hash.
end note

note right of webhook_deliveries
  **Dead Letter Queue (DLQ)**
  Si attempts > max_retries,
  se marca como FAILED y
  se alerta al tenant.
end note

@enduml
```

### 3.2. Scripts de Creación de Tablas (Extracto Clave)

```sql
-- ============================================
-- API KEYS (Hashed & Prefixed)
-- ============================================
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    name VARCHAR(100) NOT NULL,
    prefix VARCHAR(20) NOT NULL, -- Primeros 15-20 chars para identificación visual
    key_hash VARCHAR(64) NOT NULL, -- SHA256
    scopes JSONB NOT NULL DEFAULT '[]',
    rate_limit_tier VARCHAR(20) DEFAULT 'standard',
    last_used_at TIMESTAMP WITH TIME ZONE,
    expires_at TIMESTAMP WITH TIME ZONE,
    status VARCHAR(20) DEFAULT 'ACTIVE',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    CONSTRAINT uq_api_key_hash UNIQUE (key_hash)
);

CREATE INDEX idx_api_keys_prefix ON api_keys(prefix);
CREATE INDEX idx_api_keys_tenant ON api_keys(tenant_id);

-- ============================================
-- WEBHOOK DELIVERIES (Audit & Retry Log)
-- ============================================
CREATE TABLE webhook_deliveries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    http_status INT,
    response_body TEXT,
    attempts INT DEFAULT 0,
    next_retry_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    CONSTRAINT fk_delivery_subscription FOREIGN KEY (subscription_id) REFERENCES webhook_subscriptions(id) ON DELETE CASCADE
);

-- Índice para el worker de reintentos
CREATE INDEX idx_deliveries_retry ON webhook_deliveries(status, next_retry_at) 
WHERE status = 'PENDING' AND next_retry_at <= NOW();
```

---

## 4. Core Engine Details

### 4.1. Webhook Dispatcher (UC-48)

El despacho de webhooks es completamente asíncrono. Cuando ocurre un evento en SHUGO (ej. `Identity.Created`), se publica en Kafka. TORII consume este evento y lo entrega a los suscriptores.

**Estrategia de Reintentos (Exponential Backoff with Jitter):**

```csharp
public class WebhookDispatcher : IHostedService
{
    private readonly IWebhookRepository _repo;
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly IHmacSigner _signer;

    public async Task ProcessPendingDeliveriesAsync()
    {
        var deliveries = await _repo.GetDueDeliveriesAsync();

        foreach (var delivery in deliveries)
        {
            try
            {
                var subscription = await _repo.GetSubscriptionAsync(delivery.SubscriptionId);
                
                // 1. Firmar payload con HMAC SHA256
                var signature = _signer.ComputeSignature(subscription.Secret, delivery.Payload);
                
                // 2. Enviar request
                var client = _httpClientFactory.CreateClient();
                var response = await client.PostAsync(subscription.EndpointUrl, 
                    new StringContent(delivery.Payload, Encoding.UTF8, "application/json"),
                    new CancellationTokenSource(TimeSpan.FromSeconds(10)).Token); // 10s timeout

                // 3. Actualizar estado
                if (response.IsSuccessStatusCode)
                {
                    await _repo.MarkSuccessAsync(delivery.Id, (int)response.StatusCode);
                }
                else
                {
                    await ScheduleRetryAsync(delivery, (int)response.StatusCode);
                }
            }
            catch (Exception ex)
            {
                await ScheduleRetryAsync(delivery, 0); // Network error
            }
        }
    }

    private async Task ScheduleRetryAsync(Delivery delivery, int httpStatus)
    {
        delivery.Attempts++;
        if (delivery.Attempts >= 5) // Max retries
        {
            await _repo.MarkFailedAsync(delivery.Id, httpStatus);
            // Trigger alert to tenant
        }
        else
        {
            // Exponential backoff: 1m, 2m, 4m, 8m, 16m
            var delay = TimeSpan.FromMinutes(Math.Pow(2, delivery.Attempts - 1));
            delivery.NextRetryAt = DateTime.UtcNow.Add(delay);
            await _repo.UpdateDeliveryAsync(delivery);
        }
    }
}
```

### 4.2. IdP Engine & JWKS (UC-42)

Cuando SHUGO actúa como Identity Provider (IdP), debe firmar tokens (JWT para OIDC, XML para SAML). TORII gestiona un par de claves RSA rotativo y expone un endpoint JWKS (JSON Web Key Set) público.

**Flujo de Firma OIDC:**
1. La aplicación cliente solicita el token en `/oauth/token`.
2. TORII valida el `client_id` y `client_secret`.
3. TORII consulta a KAGAMI los atributos del usuario.
4. TORII aplica el `Claims Mapper` (UC-41) para transformar atributos internos a claims estándar (ej. `custom:employee_id` -> `employee_id`).
5. TORII firma el JWT con la clave privada RSA actual.
6. TORII retorna el ID Token.

**Endpoint JWKS (`/.well-known/jwks.json`):**
```json
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "kid": "shugo-key-2023-10",
      "n": "0vx7agoebGcQSuuPiLJXZptN...",
      "e": "AQAB",
      "alg": "RS256"
    }
  ]
}
```

### 4.3. SCIM Adapter (UC-39)

El adaptador SCIM traduce el estándar RFC 7643 al modelo de KAGAMI.

*   **POST /scim/v2/Users:** Crea una identidad en KAGAMI. Mapea `userName` a `email`, `displayName` a `profile.display_name`.
*   **PATCH /scim/v2/Users/{id}:** Actualiza atributos. Soporta operaciones `add`, `remove`, `replace`.
*   **GET /scim/v2/Users?filter=userName eq "juan":** Consulta a KAGAMI y formatea la respuesta como `ListResponse` de SCIM.

---

## 5. Security & API Management (UC-49)

### 5.1. API Key Lifecycle

1. **Generación:** Se genera un token criptográficamente seguro (32 bytes -> base64url).
2. **Almacenamiento:** Se guarda el `prefix` (primeros 15 caracteres) y el `SHA256 hash` del token completo. El token completo se muestra al usuario *una sola vez*.
3. **Validación:**
   - El cliente envía `Authorization: Bearer shugo_sk_live_abc123...xyz`.
   - TORII extrae el prefijo `shugo_sk_live_abc123`.
   - Busca en DB por prefijo (índice rápido).
   - Hashea el token enviado y lo compara con el `key_hash` almacenado.

### 5.2. Rate Limiting (Sliding Window Log)

Para garantizar justicia y prevenir abusos, TORII implementa un Sliding Window Log en Redis.

```csharp
public class RateLimiter
{
    private readonly IDatabase _redis;

    public async Task<bool> IsAllowedAsync(string apiKeyHash, int limit, TimeSpan window)
    {
        var now = DateTimeOffset.UtcNow.ToUnixTimeMilliseconds();
        var windowStart = now - window.TotalMilliseconds;
        var key = $"ratelimit:{apiKeyHash}";

        // 1. Eliminar logs fuera de la ventana
        await _redis.SortedSetRemoveRangeByScoreAsync(key, 0, windowStart);

        // 2. Contar requests actuales
        var count = await _redis.SortedSetLengthAsync(key);

        if (count >= limit)
        {
            return false; // Rate limit exceeded
        }

        // 3. Añadir request actual
        await _redis.SortedSetAddAsync(key, now.ToString(), now);
        await _redis.KeyExpireAsync(key, window);

        return true;
    }
}
```

---

## 6. Decisiones de Arquitectura (ADR)

### 6.1. ADR-TORII-001: Almacenamiento de API Keys (Hash vs. Encrypted)

**Decisión:** Almacenar API Keys usando Hash (SHA256) + Prefijo, en lugar de encriptación simétrica.

**Contexto:** Las API Keys son secretos de larga duración. Si la DB se compromete, el atacante no debe poder usar las keys.

**Opciones Consideradas:**
1. Texto plano (Descartado inmediatamente).
2. Encriptación simétrica (AES-256) con clave maestra.
3. Hash criptográfico (SHA256) + Prefijo para lookup.

**Decisión Tomada:** Opción 3 - Hash + Prefijo

**Justificación:**
- **Seguridad:** Si la DB es comprometida, las keys son inútiles sin el original.
- **Performance:** El lookup por prefijo es extremadamente rápido en SQL.
- **UX:** El prefijo permite al usuario identificar qué key está usando en los logs ("Mi key que empieza por `shugo_sk_live_abc` falló").

### 6.2. ADR-TORII-002: Webhook Delivery (Síncrono vs. Asíncrono)

**Decisión:** Despacho de Webhooks completamente asíncrono mediante Kafka y base de datos de reintentos.

**Contexto:** Los webhooks son críticos para la integración, pero los endpoints de los clientes son inestables y lentos.

**Opciones Consideradas:**
1. Síncrono (bloquear la respuesta de la API hasta que el webhook se entregue).
2. Asíncrono "Fire and Forget" (sin garantía de entrega).
3. Asíncrono con Persistencia y Reintentos (Outbox pattern para webhooks).

**Decisión Tomada:** Opción 3 - Asíncrono con Persistencia

**Justificación:**
- **Performance:** La API de SHUGO responde en < 50ms independientemente de la lentitud del cliente.
- **Fiabilidad:** Garantiza "At-least-once delivery". Si el cliente está caído, reintentamos durante 24h.
- **Auditoría:** La tabla `webhook_deliveries` sirve como prueba de entrega para disputas.

### 6.3. ADR-TORII-003: IdP Protocol Support (SAML vs. OIDC)

**Decisión:** Soportar ambos protocolos mediante un patrón Adaptador, pero priorizar OIDC para nuevas integraciones.

**Contexto:** El mercado enterprise aún depende heavily de SAML 2.0 (legacy), pero las aplicaciones modernas (SPAs, Mobile) usan OIDC/OAuth2.

**Opciones Consideradas:**
1. Solo OIDC (Moderno, pero aliena clientes enterprise).
2. Solo SAML (Legacy, complejo de implementar en móviles).
3. Dual Stack con Adapter Pattern.

**Decisión Tomada:** Opción 3 - Dual Stack

**Justificación:**
- **Cobertura de Mercado:** Permite vender SHUGO tanto a startups (OIDC) como a bancos/gobiernos (SAML).
- **Mantenibilidad:** El `ProtocolBuilder` abstrae la complejidad. El dominio interno solo maneja "Claims" y "Attributes", no XML ni JWTs específicos.

---

## 7. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `Product_Vision.md` | Define la visión de TORII como el habilitador del ecosistema y la seguridad perimetral de SHUGO. |
| `Domain_Analysis.md` | Define el Bounded Context TORII y sus Aggregate Roots (APIKey, WebhookSubscription, IdPConfiguration). |
| `System_Architecture_Overview.md` | Define TORII como el punto de entrada de todo el tráfico externo. |
| `Security_Threat_Model.md` | Analiza amenazas como API Key leakage, Webhook hijacking, y SAML assertion forgery. |
| `Glossary.md` | Contiene definiciones de términos de integración (SCIM, JWKS, HMAC, Sliding Window). |
| `KHCS_SHUGO_Integration_Contract.md` | Primer caso de uso real que consume TORII para sincronizar empleados desde el HRIS de KHCS. |
| UC-39 a UC-43, UC-48, UC-49 (7 documentos) | Cada UC tiene su paquete completo de 9 documentos que detalla la implementación de TORII. |

---

## 8. Anexos

### 8.1. Glosario de Términos Técnicos de TORII

| Término | Definición |
| :--- | :--- |
| **ACL (Anti-Corruption Layer)** | Patrón de diseño que traduce modelos de datos externos al modelo de dominio interno, previniendo la contaminación del core. |
| **JWKS (JSON Web Key Set)** | Estándar JSON que representa un conjunto de claves públicas utilizadas para validar firmas JWT. |
| **HMAC (Hash-based Message Authentication Code)** | Mecanismo para calcular un código de autenticación de mensajes utilizando una función hash criptográfica en combinación con una clave secreta. |
| **Sliding Window Log** | Algoritmo de Rate Limiting que registra timestamps exactos de requests para calcular la tasa en una ventana deslizante precisa. |
| **DLQ (Dead Letter Queue)** | Cola donde se almacenan mensajes/eventos que no pudieron ser procesados después de múltiples reintentos. |
| **SCIM (System for Cross-domain Identity Management)** | Estándar abierto diseñado para simplificar la gestión de identidades de usuarios en la nube. |

### 8.2. Referencias

1. **SCIM 2.0 (RFC 7643/7644):** Core Schema & Protocol - https://scim.dev/
2. **OpenID Connect Core 1.0:** https://openid.net/specs/openid-connect-core-1_0.html
3. **SAML 2.0 Technical Overview:** https://docs.oasis-open.org/security/saml/v2.0/
4. **OAuth 2.0 (RFC 6749):** https://tools.ietf.org/html/rfc6749
5. **JSON Web Key (JWK) (RFC 7517):** https://tools.ietf.org/html/rfc7517
6. **OWASP API Security Top 10:** https://owasp.org/www-project-api-security/

---
*Fin del Documento TORII_Engine_Architecture v1.0 para SHUGO*