# UC-02: Actualizar Identidad - Artefactos Modernos

| Metadatos del Documento | |
| :--- | :--- |
| **Proyecto:** | SHUGO - Identity & Access Platform |
| **Caso de Uso:** | UC-02 - Actualizar Identidad |
| **Documento:** | 09_Modern_Artifacts.md |
| **Metodología:** | API-First + Observability + Resilience Patterns + DevSecOps + Privacy by Design |
| **Autor:** | Gastón Ruiz |
| **Versión Actual:** | 1.0.0 |
| **Fecha:** | 30/08/2026 |

---

## Historial de Versiones

| Versión | Fecha | Autor | Descripción de Cambios | Impacto |
| :--- | :--- | :--- | :--- | :--- |
| **1.0.0** | 30/08/2026 | Gastón Ruiz | **Creación inicial.** Definición de los artefactos modernos para UC-02: contratos OpenAPI 3.1 con soporte para JSON Merge Patch (RFC 7396) y Control de Concurrencia Optimista (OCC), catálogo de métricas OpenTelemetry (enfocado en conflictos de concurrencia y latencia de re-cifrado), patrones de resiliencia (Circuit Breaker para KMS), estrategias de enmascaramiento de PII en logs de auditoría, y pruebas de carga para escenarios de alta concurrencia. | Cierra el paquete de diseño Tier 1 del UC-02, proporcionando todos los artefactos necesarios para la implementación, operación, monitoreo y cumplimiento normativo (SOC 2 / GDPR) del endpoint de actualización del motor KAGAMI. |

---

## 1. Introducción y Propósito

Este documento define los **Artefactos Modernos** necesarios para operar el UC-02 (Actualizar Identidad) en un entorno de producción enterprise. A diferencia del UC-01 (Creación), el UC-02 introduce desafíos únicos que requieren artefactos específicos:

1. **Control de Concurrencia Optimista (OCC):** Manejo de conflictos cuando múltiples actores (Admin, SCIM, Usuario) editan la misma identidad simultáneamente.
2. **JSON Merge Patch (RFC 7396):** Soporte nativo para actualizaciones parciales y borrado de campos mediante `null`.
3. **Auditoría Forense con Privacidad:** Generación de logs de auditoría (Before/After) que **nunca** almacenan PII en texto plano.
4. **Re-cifrado Condicional:** Optimización de recursos al invocar el servicio de cifrado (KMS) solo cuando es estrictamente necesario.

---

## 2. Contratos OpenAPI Completos

### 2.1. Especificación OpenAPI 3.1

```yaml
openapi: 3.1.0
info:
  title: SHUGO KAGAMI - Identity Update API
  description: |
    API de actualización del ciclo de vida de identidades del motor KAGAMI.
    
    ## Características Clave
    - Control de Concurrencia Optimista (OCC) mediante header `If-Match`.
    - Soporte nativo para JSON Merge Patch (RFC 7396) para actualizaciones parciales.
    - Auditoría forense automática (Before/After) con enmascaramiento de PII.
    - Re-cifrado automático de PII (Email, Teléfono) si se modifican.
    
    ## Autenticación y Autorización
    Requiere JWT válido con permisos `identity:update`.
    El scope del actor (Admin, SCIM, User) determina qué campos pueden modificarse.
  version: 1.0.0

servers:
  - url: https://api.shugo.io/v1
    description: Producción

paths:
  /identities/{id}:
    patch:
      operationId: updateIdentity
      summary: Actualizar identidad existente
      description: |
        Modifica parcialmente una identidad existente.
        Utiliza JSON Merge Patch (RFC 7396) para `custom_attributes`.
        Requiere header `If-Match` con el `RowVersion` actual para prevenir Lost Updates.
      tags:
        - Identities
      security:
        - BearerAuth: []
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
        - name: If-Match
          in: header
          required: true
          description: "RowVersion actual de la identidad (para OCC). Obtenido vía GET /identities/{id}."
          schema:
            type: string
            format: uuid
      requestBody:
        required: true
        content:
          application/merge-patch+json: # RFC 7396
            schema:
              $ref: '#/components/schemas/IdentityMergePatch'
            examples:
              updateDisplayName:
                summary: Actualizar solo el nombre
                value:
                  profile:
                    display_name: "Juan Director"
              updateCustomAttributes:
                summary: Actualizar y borrar atributos custom
                value:
                  profile:
                    custom_attributes:
                      clearance_level: 4
                      old_attribute: null # Esto borra la clave
              suspendIdentity:
                summary: Suspender identidad
                value:
                  status: "SUSPENDED"
                  reason: "Investigación de seguridad"
      responses:
        '200':
          description: Identidad actualizada exitosamente
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/IdentityUpdateResponse'
        '400':
          description: Error de validación (Schema, Transición de estado inválida)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '403':
          description: Prohibido (Actor no tiene scope para editar estos campos)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '404':
          description: Identidad no encontrada o está PURGED
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '409':
          description: Conflicto de Concurrencia (OCC Failure)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '500':
          description: Error interno (Ej. Fallo de cifrado KMS, Error de DB)
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

  schemas:
    IdentityMergePatch:
      type: object
      properties:
        status:
          type: string
          enum: [ACTIVE, SUSPENDED, DEACTIVATED]
          description: "Solo Admin puede cambiar esto."
        reason:
          type: string
          description: "Requerido si se cambia el status."
        profile:
          type: object
          properties:
            display_name:
              type: string
            email:
              type: string
              format: email
              description: "Si cambia, se re-cifra y se dispara re-verificación."
            phone:
              type: string
              pattern: '^\+[1-9]\d{1,14}$'
            custom_attributes:
              type: object
              additionalProperties: true
              description: "Soporta Merge Patch. Usar 'null' para borrar claves."

    IdentityUpdateResponse:
      type: object
      properties:
        id:
          type: string
          format: uuid
        row_version:
          type: string
          format: uuid
          description: "Nuevo RowVersion. Debe usarse en el próximo PATCH."
        updated_at:
          type: string
          format: date-time
        profile:
          type: object
          properties:
            display_name:
              type: string
            custom_attributes:
              type: object

    ErrorResponse:
      type: object
      properties:
        error:
          type: object
          properties:
            code:
              type: string
              enum: [CONCURRENCY_CONFLICT, INVALID_TRANSITION, INSUFFICIENT_SCOPE, PII_ENCRYPTION_FAILED, INVALID_SCHEMA]
            message:
              type: string
            details:
              type: object
              properties:
                expected_version:
                  type: string
                  format: uuid
                current_version:
                  type: string
                  format: uuid
```

### 2.2. Códigos de Error Específicos de UC-02

| Código HTTP | Código de Error | Descripción | Cuándo Ocurre |
| :--- | :--- | :--- | :--- |
| **400** | `INVALID_TRANSITION` | Cambio de estado no permitido (ej. `PURGED` -> `ACTIVE`). | `IdentityValidator` detecta transición inválida. |
| **400** | `INVALID_SCHEMA` | `custom_attributes` no cumple el JSON Schema del tenant. | `IdentityValidator` post-Merge Patch. |
| **403** | `INSUFFICIENT_SCOPE` | El actor no puede editar los campos solicitados (ej. User editando `clearance_level`). | `IdentityValidator` valida el `ActorScope`. |
| **409** | `CONCURRENCY_CONFLICT` | El `RowVersion` enviado en `If-Match` no coincide con el de la DB. | `UpdateHandler` (Validación en memoria) o `IdentityRepository` (Validación en DB). |
| **500** | `PII_ENCRYPTION_FAILED` | Fallo al re-cifrar PII (KMS down). **Fail Secure:** No se actualiza nada. | `PiiEncryptor` durante el re-cifrado condicional. |

---

## 3. Catálogo de Métricas OpenTelemetry

### 3.1. Métricas de Negocio y Concurrencia

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.identity.updated_total` | Counter | updates | Total de identidades actualizadas exitosamente | `tenant_id`, `actor_type` (Admin/SCIM/User) |
| `kagami.identity.update_concurrency_conflicts_total` | Counter | conflicts | Total de conflictos de OCC (409). **Alerta si es alto** (indica problemas de UX o scripts mal configurados). | `tenant_id`, `endpoint` |
| `kagami.identity.pii_re_encrypted_total` | Counter | operations | Total de re-cifrados de PII (solo cuando email/phone cambia). | `tenant_id`, `field_type` |

### 3.2. Métricas de Rendimiento

| Nombre de Métrica | Tipo | Unidad | Descripción | Etiquetas |
| :--- | :--- | :--- | :--- | :--- |
| `kagami.identity.update_latency` | Histogram | ms | Latencia end-to-end del endpoint PATCH. | `tenant_id`, `success`, `has_pii_changes` |
| `kagami.identity.audit_snapshot_generation_latency` | Histogram | ms | Latencia específica de la generación del Diff y enmascaramiento de PII. | `tenant_id`, `num_fields_changed` |
| `kagami.identity.pii_re_encryption_latency` | Histogram | ms | Latencia del re-cifrado condicional (KMS call). | `tenant_id` |

### 3.3. Alertas Sugeridas

| Alerta | Condición | Severidad | Acción |
| :--- | :--- | :--- | :--- |
| **Alta Tasa de Conflictos OCC** | `update_concurrency_conflicts_total > 50` en 5 min | Media | Posible script SCIM mal configurado haciendo polling y update constante. Revisar lógica del cliente. |
| **Degradación de KMS** | `pii_re_encryption_latency_p95 > 100ms` | Alta | El servicio de cifrado está lento. Afecta solo a updates que modifican PII. |
| **Fallo de Cifrado (Fail Secure)** | `PII_ENCRYPTION_FAILED > 0` | **Crítica** | El KMS está caído. **Ninguna actualización que modifique PII está funcionando.** Revisar AWS KMS/Vault. |
| **Audit Logger Lento** | `audit_snapshot_generation_latency_p95 > 50ms` | Media | El enmascaramiento de PII o la generación de diffs está consumiendo mucha CPU. |

---

## 4. Patrones de Resiliencia

### 4.1. Circuit Breaker para KMS (Re-cifrado Condicional)

Dado que el re-cifrado de PII es opcional (solo si cambia el email/phone), el Circuit Breaker debe configurarse para fallar rápido y permitir que las actualizaciones que **no** modifican PII sigan funcionando.

**Implementación (Polly - C#):**

```csharp
public class ResilientKmsClient : IKmsClient
{
    private readonly IKmsClient _inner;
    private readonly AsyncCircuitBreakerPolicy _circuitBreaker;

    public ResilientKmsClient(IKmsClient inner)
    {
        _inner = inner;
        _circuitBreaker = Policy
            .Handle<KmsServiceUnavailableException>()
            .CircuitBreakerAsync(
                exceptionsAllowedBeforeBreaking: 3,
                durationOfBreak: TimeSpan.FromSeconds(30),
                onBreak: (ex, duration) => Log.Warning("KMS Circuit Breaker OPEN. PII Re-encryption disabled."),
                onReset: () => Log.Information("KMS Circuit Breaker CLOSED.")
            );
    }

    public async Task<EncryptionResult> EncryptAsync(string plainText)
    {
        // Si el circuito está abierto, lanzamos excepción inmediatamente (Fail Secure)
        // El UpdateHandler capturará esto y retornará HTTP 500 sin tocar la DB.
        return await _circuitBreaker.ExecuteAsync(() => _inner.EncryptAsync(plainText));
    }
}
```

### 4.2. Manejo de Race Conditions (Doble Validación OCC)

El sistema valida el OCC en dos niveles para máxima robustez:
1. **Nivel Aplicación (Memoria):** El `UpdateHandler` compara el `If-Match` header con el `RowVersion` cargado de la DB. Falla rápido (HTTP 409) antes de procesar.
2. **Nivel Base de Datos (SQL):** El `UPDATE` generado por EF Core incluye `WHERE row_version = @old_version`. Si otro proceso ganó la carrera, afectará 0 filas y EF Core lanzará `DbUpdateConcurrencyException`, que el Repository traduce a `ConcurrencyConflictException`.

---

## 5. Seguridad y Privacidad: Enmascaramiento de PII en Auditoría

### 5.1. Estrategia de Enmascaramiento (IPiiMasker)

El `AuditLogger` utiliza un servicio `IPiiMasker` para garantizar que **nunca** se almacene PII en texto plano en la tabla `audit_logs`.

**Implementación de Enmascaramiento:**

| Tipo de Campo | Estrategia de Enmascaramiento | Ejemplo (Before -> After) |
| :--- | :--- | :--- |
| **Email** | Hash Determinista (HMAC-SHA256) | `juan@test.com` -> `a1b2c3d4...` |
| **Teléfono** | Enmascaramiento Parcial (Redaction) | `+5491123456789` -> `+54911****6789` |
| **Display Name** | Sin enmascaramiento (No es PII sensible) | `Juan` -> `Juan Director` |
| **Custom Attributes** | Depende del Schema del Tenant | Si el schema marca el campo como `sensitive: true`, se hashea. |

**Código de Ejemplo (C#):**

```csharp
public class PiiMasker : IPiiMasker
{
    private readonly IHmacService _hmacService;
    private readonly ITenantSchemaService _schemaService;

    public object Mask(string fieldName, object value, Guid tenantId)
    {
        if (value == null) return null;

        // Lista de campos PII conocidos
        if (fieldName.Equals("email", StringComparison.OrdinalIgnoreCase))
        {
            return _hmacService.ComputeHash(value.ToString());
        }
        
        if (fieldName.Equals("phone", StringComparison.OrdinalIgnoreCase))
        {
            var str = value.ToString();
            return str.Length > 4 ? $"{str.Substring(0, str.Length - 4)}****" : "****";
        }

        // Verificar si el custom attribute es sensible según el schema del tenant
        if (fieldName.StartsWith("custom_attributes.") && _schemaService.IsFieldSensitive(tenantId, fieldName))
        {
            return _hmacService.ComputeHash(value.ToString());
        }

        // Por defecto, no enmascarar
        return value;
    }
}
```

### 5.2. Cumplimiento Normativo (SOC 2 / GDPR)

- **SOC 2 (CC6.1 - Logical Access):** Los analistas de seguridad que revisan los logs de auditoría no tienen acceso a la tabla `profiles` (donde está el PII cifrado). Solo ven los hashes en `audit_logs`, lo cual es suficiente para correlacionar eventos sin exponer datos sensibles.
- **GDPR (Art. 32 - Security of Processing):** El enmascaramiento automático previene fugas de datos a través de los propios sistemas de logging y monitoreo.

---

## 6. Testing Strategy

### 6.1. Unit Tests (Dominio y Lógica de Negocio)

```csharp
[Test]
public void Identity_ChangeStatus_WhenTransitionInvalid_ThrowsException()
{
    // Arrange
    var identity = new Identity(/* Status = PURGED */);
    
    // Act & Assert
    var act = () => identity.ChangeStatus(IdentityStatus.ACTIVE, "Reactivación");
    act.Should().Throw<InvalidTransitionException>()
       .WithMessage("Transition from PURGED to ACTIVE is not allowed.");
}

[Test]
public void Profile_ApplyMergePatch_WhenNullValue_RemovesKey()
{
    // Arrange
    var profile = new Profile(/* custom_attributes = { "rol": "admin", "dept": "Vet" } */);
    var patch = new ProfileMergePatch { CustomAttributes = new Dictionary<string, object> { { "rol", null } } };
    
    // Act
    profile.ApplyMergePatch(patch, mockSchema);
    
    // Assert
    profile.CustomAttributes.Should().NotContainKey("rol");
    profile.CustomAttributes.Should().ContainKey("dept"); // No se borra
}
```

### 6.2. Integration Tests (Concurrencia y Transacciones)

```csharp
[Test]
public async Task UpdateIdentity_WhenConcurrentUpdates_OnlyOneSucceeds()
{
    // Arrange: Crear identidad inicial
    var identityId = await CreateTestIdentityAsync();
    var initialRowVersion = await GetRowVersionAsync(identityId);

    // Act: Dos handlers intentan actualizar simultáneamente con el mismo RowVersion
    var task1 = UpdateDisplayNameAsync(identityId, initialRowVersion, "Name 1");
    var task2 = UpdateDisplayNameAsync(identityId, initialRowVersion, "Name 2");

    // Assert
    var results = await Task.WhenAll(
        Task.Run(async () => { try { await task1; return true; } catch { return false; } }),
        Task.Run(async () => { try { await task2; return true; } catch { return false; } })
    );

    results.Should().Contain(true); // Uno debe tener éxito
    results.Should().Contain(false); // El otro debe fallar con 409
    
    // Verificar que solo un cambio se persistió
    var finalName = await GetDisplayNameAsync(identityId);
    finalName.Should().BeOneOf("Name 1", "Name 2");
}
```

### 6.3. Load Testing (Escenario de Alta Concurrencia)

**Objetivo:** Validar que el sistema maneja correctamente múltiples intentos de actualización sobre la misma identidad sin degradar el rendimiento ni corromper datos.

**Script k6 (Extracto):**

```javascript
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  scenarios: {
    concurrent_updates: {
      executor: 'constant-vus',
      vus: 50, // 50 usuarios virtuales intentando actualizar la MISMA identidad
      duration: '1m',
    },
  },
  thresholds: {
    http_req_duration: ['p(95)<250'],
    http_req_failed: ['rate<0.1'], // Esperamos muchos 409, pero pocos 500
  },
};

export default function () {
  // 1. GET para obtener el RowVersion actual
  const getRes = http.get(`${__ENV.BASE_URL}/v1/identities/${__ENV.TEST_IDENTITY_ID}`);
  const rowVersion = getRes.json('row_version');

  // 2. PATCH con el RowVersion obtenido
  const patchRes = http.patch(
    `${__ENV.BASE_URL}/v1/identities/${__ENV.TEST_IDENTITY_ID}`,
    JSON.stringify({ profile: { display_name: `User_${Date.now()}` } }),
    {
      headers: {
        'Content-Type': 'application/merge-patch+json',
        'Authorization': `Bearer ${__ENV.JWT_TOKEN}`,
        'If-Match': rowVersion,
      },
    }
  );

  // 3. Validar respuestas (Esperamos mezcla de 200 y 409)
  check(patchRes, {
    'is success (200)': (r) => r.status === 200,
    'is conflict (409)': (r) => r.status === 409,
    'is not server error': (r) => r.status < 500,
  });
}
```

---

## 7. Deployment y Operaciones

### 7.1. Migraciones de Base de Datos

Para soportar UC-02, se requieren las siguientes migraciones en PostgreSQL:

```sql
-- 1. Agregar columna RowVersion a la tabla identities
ALTER TABLE identities 
ADD COLUMN row_version UUID NOT NULL DEFAULT gen_random_uuid();

-- 2. Crear tabla de logs de auditoría forense
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    identity_id UUID NOT NULL,
    tenant_id UUID NOT NULL,
    actor_id UUID NOT NULL,
    action VARCHAR(50) NOT NULL, -- 'UPDATED', 'STATUS_CHANGED'
    snapshot_json JSONB NOT NULL, -- Contiene el AuditSnapshot enmascarado
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    FOREIGN KEY (identity_id) REFERENCES identities(id)
);

CREATE INDEX idx_audit_logs_identity ON audit_logs(identity_id, created_at DESC);
CREATE INDEX idx_audit_logs_tenant ON audit_logs(tenant_id, created_at DESC);

-- 3. Agregar trigger para actualizar row_version automáticamente (Opcional, si no lo maneja EF Core)
-- CREATE OR REPLACE FUNCTION update_row_version() ...
```

### 7.2. Feature Flags

Para permitir una rollout gradual de las nuevas capacidades de UC-02:

| Feature Flag | Descripción | Valor por Defecto |
| :--- | :--- | :--- |
| `kagami.identity.update.occ_enabled` | Activa la validación de `If-Match` y el manejo de 409. | `true` (Siempre activo en Prod) |
| `kagami.identity.update.audit_forensic_enabled` | Activa la generación del `AuditSnapshot` con enmascaramiento de PII. | `true` |
| `kagami.identity.update.merge_patch_enabled` | Activa el soporte para `application/merge-patch+json`. | `true` |

---

## 8. Trazabilidad

| Documento | Relación |
| :--- | :--- |
| `01_Requirements_Specification.md` | Define los requisitos de OCC, Merge Patch y Auditoría que estos artefactos implementan. |
| `08_Final_Class_Design.md` | Las clases `UpdateIdentityHandler`, `AuditLogger`, y `PiiMasker` se implementan según los contratos y métricas definidos aquí. |
| `10_UC_Specific_Data_Model.md` (UC-01) | Define la estructura física de las tablas `identities` (con `row_version`) y `audit_logs`. |
| `11_UC_Specific_Security_Threats.md` (UC-01) | Las mitigaciones de "Audit Log Tampering" y "PII Leakage" se validan mediante el enmascaramiento automático definido en este documento. |
| `SHUGO_Security_Architecture.md` (Transversal) | Este documento aplica los estándares globales de enmascaramiento de PII al contexto específico de UC-02. |

---
*Fin del Documento 09_Modern_Artifacts para UC-02*