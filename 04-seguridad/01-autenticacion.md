# Seguridad y autenticación

**Versión:** 2.0
**Fecha:** 2026-03-13
**Estado:** Activo

---

## Estrategia en dos etapas

### Etapa 1 — JWT propio (arranque)

VGM Go genera sus propios tokens JWT internamente.

- Endpoint de login: `POST /api/v1/auth/login`
- VGM Go valida usuario/contraseña contra su propia tabla `usuarios_go`
- Emite un JWT firmado con clave propia (`vgmgo.jwt.secret`)
- Expiración configurable (`vgmgo.jwt.expiration: 3600`)

**Ventaja:** funciona de forma completamente independiente, sin depender de Auth0.

### Etapa 2 — Auth0 (futuro)

Cuando haya necesidad (SSO, cliente enterprise, integración con VGM Core):

- El backend pasa a ser **Resource Server**
- Solo valida tokens emitidos por Auth0
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

### Etapa 1 (JWT propio)
```json
{
  "sub": "usuario@empresa.com",
  "https://vgmcoregeo.com/cliente_saas_id": 1,
  "https://vgmcoregeo.com/empresa_id": 5,
  "https://vgmcoregeo.com/sucursal_id": 3,
  "https://vgmcoregeo.com/rol": "ADMIN",
  "exp": 1234567890
}
```

### Etapa 2 (Auth0)
Mismo formato de claims, emitidos por Auth0 a través de un Action Post Login igual al de VGM Core pero con namespace `https://vgmcoregeo.com`.

---

## Flujo de autenticación

```
1. Usuario abre VGM Go → no tiene sesión → pantalla de login
2. Ingresa usuario y contraseña
3. POST /api/v1/auth/login → VGM Go valida y emite JWT
4. Frontend guarda el token EN MEMORIA (nunca en localStorage)
5. Cada request envía: Authorization: Bearer <token>
6. TenantContextFilter intercepta, valida JWT, carga contexto
   (cliente_saas, empresa, sucursal)
7. Controller procesa con contexto ya resuelto
```

---

## Componentes de seguridad

Copiados y adaptados del código de Mauricio (VGM Core):

| Componente | Origen | Cambio para VGM Go |
|---|---|---|
| `TenantContextFilter` | VGM Core | Agregar resolución de `id_sucursal` |
| `TenantConnectionPreparer` | VGM Core | Copiar sin cambios |
| `TenantExceptions` | VGM Core | Copiar sin cambios |
| `GlobalExceptionHandler` | VGM Core | Copiar sin cambios |
| `TenantResolver` | VGM Core | Cambiar namespace a `https://vgmcoregeo.com` |
| `SecurityConfig` | VGM Core | Etapa 1: generar JWT. Etapa 2: Resource Server |

---

## Diferencia con VGM Core

| | VGM Core | VGM Go |
|---|---|---|
| Niveles de tenancy | `clientes_saas → empresas → sucursales` | Igual — misma jerarquía |
| Namespace JWT | `https://vgmcore.com` | `https://vgmcoregeo.com` |
| Auth0 Application | VGM Core Web / Desktop | VGM Core Geo |
| RLS | Obligatorio desde el inicio | A definir — por ahora filtro por campo |
| IdP Etapa 1 | Auth0 desde el inicio | JWT propio |
| IdP Etapa 2 | — | Auth0 (`vgm-core-dev.us.auth0.com`) |

---

## Auth0 Action para Etapa 2

Cuando se implemente Auth0, el Action Post Login para VGM Go sería:

```javascript
exports.onExecutePostLogin = async (event, api) => {
  const NAMESPACE = 'https://vgmcoregeo.com';

  const orgMetadata = event.organization?.metadata || {};
  const tenantId = orgMetadata.tenant_id;

  if (!tenantId) {
    api.access.deny('No se pudo determinar el tenant.');
    return;
  }

  api.accessToken.setCustomClaim(`${NAMESPACE}/cliente_saas_id`, parseInt(tenantId, 10));

  if (event.user.email) {
    api.accessToken.setCustomClaim(`${NAMESPACE}/email`, event.user.email);
  }

  const roles = event.authorization?.roles || [];
  if (roles.length > 0) {
    api.accessToken.setCustomClaim(`${NAMESPACE}/roles`, roles);
  }
};
```

> Coordinarlo con Mauricio antes de implementar — tiene que vivir en el mismo tenant de Auth0.
