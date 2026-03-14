# Seguridad y autenticación

**Versión:** 2.1
**Fecha:** 2026-03-14
**Estado:** Activo

---

## Estrategia en dos etapas

### Etapa 1 — JWT propio (arranque)

VGM Core Geo genera sus propios tokens JWT internamente.

- Endpoint de login: `POST /api/v1/auth/login`
- VGM Core Geo valida usuario/contraseña contra su propia tabla `cuentas` + `usuarios_geo`
- Emite un JWT firmado con clave propia (`vgmcoregeo.jwt.secret`)
- Expiración configurable (`vgmcoregeo.jwt.expiration: 3600`)

**Ventaja:** funciona de forma completamente independiente, sin depender de Auth0.

### Etapa 2 — Auth0 (futuro)

Cuando haya necesidad (SSO, cliente enterprise, integración con VGM Core):

- El backend pasa a ser **Resource Server** (igual que VGM Core hoy)
- Solo valida tokens emitidos por Auth0 (`vgm-core-dev.us.auth0.com`)
- El cambio en código es mínimo: configuración en `application.yml`
- Todo lo demás (filtros, servicios) queda igual

---

## Auth0 — configuración definida

VGM Core Geo ya está creada como aplicación en Auth0 dentro del tenant de Mauricio.

| Dato | Valor |
|---|---|
| Auth0 Tenant | `vgm-core-dev.us.auth0.com` |
| Aplicación | `VGM Core Geo` (Single Page Application) |
| Namespace de claims | `https://vgmcoregeo.com` |

> El namespace es diferente al de VGM Core (`https://vgmcore.com`) porque son productos independientes con sus propios tokens.

---

## Estructura del token JWT

Siguiendo el patrón de VGM Core: el token lleva **solo el tenant** (`tenant_id`). La empresa y la sucursal se resuelven dinámicamente desde headers HTTP y base de datos — no van embebidas en el token.

### Etapa 1 (JWT propio)
```json
{
  "sub": "usuario@empresa.com",
  "https://vgmcoregeo.com/tenant_id": 1,
  "exp": 1234567890
}
```

### Etapa 2 (Auth0)
Mismo claim `tenant_id`, emitido por Auth0 a través de un Action Post Login con namespace `https://vgmcoregeo.com`.

---

## Headers de contexto

Igual que VGM Core, la empresa y la sucursal activa se resuelven por headers:

| Header | Obligatorio | Descripción |
|---|---|---|
| `Authorization: Bearer <token>` | Sí | JWT del usuario |
| `X-Empresa-Id` | Condicional | Obligatorio si el usuario tiene acceso a más de una empresa |
| `X-Sucursal-Id` | Condicional | Obligatorio si el usuario tiene acceso a más de una sucursal |

Si el usuario solo tiene una empresa/sucursal asignada, el header es opcional — VGM Core Geo la resuelve automáticamente.

---

## Flujo de autenticación

```
1. Usuario abre VGM Core Geo → no tiene sesión → pantalla de login
2. Ingresa usuario y contraseña
3. POST /api/v1/auth/login → VGM Core Geo valida y emite JWT
4. Frontend guarda el token EN MEMORIA (nunca en localStorage)
5. Cada request envía:
   Authorization: Bearer <token>
   X-Empresa-Id: <id>      (si aplica)
   X-Sucursal-Id: <id>     (si aplica)
6. TenantContextFilter intercepta, valida JWT, carga contexto
   (cliente_saas, empresa, sucursal)
7. Controller procesa con contexto ya resuelto
```

---

## Componentes de seguridad

Copiados y adaptados del código de Mauricio (VGM Core). VGM Core Geo extiende el patrón agregando resolución de `id_sucursal`.

| Componente | Origen | Cambio para VGM Core Geo |
|---|---|---|
| `TenantContextFilter` | VGM Core | Agregar resolución de `id_sucursal` desde header `X-Sucursal-Id` |
| `TenantConnectionPreparer` | VGM Core | Agregar `set_config('app.id_sucursal', valor)` |
| `TenantExceptions` | VGM Core | Copiar sin cambios, agregar excepciones de sucursal |
| `GlobalExceptionHandler` | VGM Core | Copiar sin cambios |
| `TenantResolver` | VGM Core | Cambiar namespace a `https://vgmcoregeo.com` y claim a `tenant_id` |
| `SecurityConfig` | VGM Core | Etapa 1: emitir JWT propio. Etapa 2: Resource Server igual que VGM Core |

### TenantContext de VGM Core Geo

Extiende el de VGM Core agregando `idSucursal`:

```kotlin
data class DatosTenant(
    val idClienteSaas: Long,
    val idEmpresa: Long,
    val idSucursal: Long,
    val idUsuarioGeo: Long,
    val email: String,
    val sub: String,
)
```

---

## Resolución del contexto (paso a paso)

```
1. Spring Security valida el JWT
   - Etapa 1: valida firma con clave propia
   - Etapa 2: valida contra Auth0 (issuer-uri + audience)

2. TenantContextFilter (@Order(200)) corre después de Spring Security

3. TenantResolver extrae del JWT:
   - "https://vgmcoregeo.com/tenant_id" → idClienteSaas
   - "sub" → subject del usuario

4. Busca el usuario en BD (cuentas + usuarios_geo)
   - Si no existe → 403 FORBIDDEN

5. Resuelve empresa desde X-Empresa-Id:
   - Si header presente → valida acceso
   - Si una sola empresa → la usa automáticamente
   - Si múltiples y sin header → 400 BAD REQUEST

6. Resuelve sucursal desde X-Sucursal-Id:
   - Igual que empresa, pero dentro de la empresa resuelta

7. Carga rol desde usuarios_geo_sucursales (nunca desde el JWT)

8. TenantContext.establecer(DatosTenant) → guarda en ThreadLocal

9. TenantConnectionPreparer ejecuta set_config en PostgreSQL:
   - set_config('app.id_cliente_saas', valor)
   - set_config('app.id_empresa', valor)
   - set_config('app.id_sucursal', valor)

10. Controller ejecuta — contexto disponible vía TenantContext

11. finally: TenantContext.limpiar()
```

---

## Excepciones mapeadas a HTTP

| Excepción | HTTP |
|---|---|
| `TenantNoResueltaException` | 401 UNAUTHORIZED |
| `UsuarioNoEncontradoException` | 403 FORBIDDEN |
| `EmpresaNoAsignadaException` | 403 FORBIDDEN |
| `EmpresaInvalidaException` | 400 BAD REQUEST |
| `AccesoEmpresaDenegadoException` | 403 FORBIDDEN |
| `EmpresaNoSeleccionadaException` | 400 BAD REQUEST |
| `SucursalNoAsignadaException` | 403 FORBIDDEN |
| `SucursalNoSeleccionadaException` | 400 BAD REQUEST |

---

## Comparación con VGM Core

| | VGM Core | VGM Core Geo |
|---|---|---|
| Niveles de tenancy resueltos | `cliente_saas` + `empresa` | `cliente_saas` + `empresa` + `sucursal` |
| Namespace JWT | `https://vgmcore.com` | `https://vgmcoregeo.com` |
| Claim de tenant | `tenant_id` | `tenant_id` |
| Rol en JWT | No — se carga de BD | No — se carga de BD |
| Auth0 Application | VGM Core Web / Desktop | VGM Core Geo |
| IdP Etapa 1 | Auth0 desde el inicio | JWT propio |
| IdP Etapa 2 | — | Auth0 (`vgm-core-dev.us.auth0.com`) |
| Header empresa | `X-Empresa-Id` | `X-Empresa-Id` |
| Header sucursal | No aplica | `X-Sucursal-Id` |

---

## Auth0 Action para Etapa 2

Cuando se implemente Auth0, el Action Post Login para VGM Core Geo sería:

```javascript
exports.onExecutePostLogin = async (event, api) => {
  const NAMESPACE = 'https://vgmcoregeo.com';

  const orgMetadata = event.organization?.metadata || {};
  const tenantId = orgMetadata.tenant_id;

  if (!tenantId) {
    api.access.deny('No se pudo determinar el tenant.');
    return;
  }

  api.accessToken.setCustomClaim(`${NAMESPACE}/tenant_id`, parseInt(tenantId, 10));

  if (event.user.email) {
    api.accessToken.setCustomClaim(`${NAMESPACE}/email`, event.user.email);
  }
};
```

> Coordinarlo con Mauricio antes de implementar — tiene que vivir en el mismo tenant de Auth0.
