# Modelo de datos

**Versión:** 3.0
**Fecha:** 2026-03-27
**Estado:** Activo

---

## Principios

VGM Core Geo tiene su **propia base de datos PostgreSQL**. No comparte datos con VGM Core ni con ningún otro sistema.

La jerarquía de tenancy es **idéntica a VGM Core** — mismos nombres de tablas y columnas. Esto facilita que el equipo lea código de ambos productos sin fricción y simplifica la integración futura.

---

## Jerarquía de tenancy

```
clientes_saas (tenant — el cliente que contrató VGM Core Geo)
  └── empresas (las empresas del cliente)
        └── sucursales (las sucursales de cada empresa)
              ├── entidades (personas físicas o jurídicas — vendedores, repartidores, clientes)
              │     └── entidades_roles (rol que ejerce: VENDEDOR, REPARTIDOR, SUPERVISOR, CLIENTE)
              ├── zonas
              └── posiciones
```

**Ejemplo real:**
```
D OUNI (cliente_saas)
  ├── OV (empresa)
  │     ├── Sucursal Centro → entidades (vendedores, clientes), zonas
  │     └── Sucursal Norte  → entidades (vendedores, clientes), zonas
  ├── ZIRKA (empresa)
  │     └── Sucursal única  → entidades, zonas
  └── OV2 (empresa)
        └── Sucursal única  → entidades, zonas
```

---

## Party Model (ADR-009 — alineado con VGM Core)

VGM Core Geo implementa el mismo Party Model que VGM Core: una sola tabla `entidades` unifica lo que antes eran `empleados` y `puntos_venta`. El rol de cada entidad se define en `entidades_roles`.

**Ventaja clave:** si "Juan García" es vendedor y también es un cliente de la sucursal, sus datos se almacenan una sola vez.

| Rol | Descripción | Equivalente legacy |
|---|---|---|
| `VENDEDOR` | Reporta posiciones GPS en movimiento | ex-empleados tipo VENDEDOR |
| `REPARTIDOR` | Reporta posiciones GPS en movimiento | ex-empleados tipo REPARTIDOR |
| `SUPERVISOR` | Reporta posiciones GPS en movimiento | ex-empleados tipo SUPERVISOR |
| `CLIENTE` | Tiene coordenadas fijas (punto de visita) | ex-puntos_venta |

Sin CHECK constraint en `co_rol`: nuevos roles (ej: `VEHICULO`) se agregan sin modificar el schema. La validación vive en la aplicación.

---

## Tablas de tenancy

### `clientes_saas`
```sql
CREATE TABLE clientes_saas (
    id_cliente_saas     bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    id_publico          uuid NOT NULL DEFAULT gen_random_uuid(),
    co_cliente          varchar(50) NOT NULL,
    de_cliente          varchar(200) NOT NULL,
    sn_activo           boolean NOT NULL DEFAULT true,
    fe_alta             timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_cliente_saas_publico UNIQUE (id_publico)
);
```

### `empresas`
```sql
CREATE TABLE empresas (
    id_empresa          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    id_publico          uuid NOT NULL DEFAULT gen_random_uuid(),
    id_cliente_saas     bigint NOT NULL REFERENCES clientes_saas(id_cliente_saas),
    co_empresa          varchar(50) NOT NULL,
    de_empresa          varchar(200) NOT NULL,
    sn_activo           boolean NOT NULL DEFAULT true,
    fe_alta             timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_empresa_publico UNIQUE (id_publico)
);
```

### `sucursales`
```sql
CREATE TABLE sucursales (
    id_sucursal         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    id_publico          uuid NOT NULL DEFAULT gen_random_uuid(),
    id_cliente_saas     bigint NOT NULL REFERENCES clientes_saas(id_cliente_saas),
    id_empresa          bigint NOT NULL REFERENCES empresas(id_empresa),
    co_sucursal         varchar(50) NOT NULL,
    de_sucursal         varchar(200) NOT NULL,
    sn_activo           boolean NOT NULL DEFAULT true,
    fe_alta             timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_sucursal_publico UNIQUE (id_publico)
);
```

---

## Tablas de seguridad

Siguen el patrón de VGM Core: identidad global separada de la membresía en el tenant.

### `cuentas`
Identidad global. Un usuario puede tener membresía en múltiples tenants.
```sql
CREATE TABLE cuentas (
    id_cuenta           bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    id_publico          uuid NOT NULL DEFAULT gen_random_uuid(),
    de_email            varchar(255) NOT NULL,
    de_nombre           varchar(200),
    co_sub_oidc         varchar(255),          -- subject OIDC (Auth0 en Etapa 2)
    sn_activo           boolean NOT NULL DEFAULT true,
    fe_alta             timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_cuenta_email UNIQUE (de_email),
    CONSTRAINT uq_cuenta_publico UNIQUE (id_publico)
);
```

### `usuarios_geo`
Membresía de una cuenta dentro de un tenant (cliente_saas).
```sql
CREATE TABLE usuarios_geo (
    id_usuario_geo      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    id_cuenta           bigint NOT NULL REFERENCES cuentas(id_cuenta),
    id_cliente_saas     bigint NOT NULL REFERENCES clientes_saas(id_cliente_saas),
    sn_activo           boolean NOT NULL DEFAULT true,
    fe_alta             timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT uq_usuario_geo_cuenta_tenant UNIQUE (id_cuenta, id_cliente_saas)
);
```

### `usuarios_geo_sucursales`
Acceso de un usuario a una sucursal específica con su rol.
```sql
CREATE TABLE usuarios_geo_sucursales (
    id_usuario_geo      bigint NOT NULL REFERENCES usuarios_geo(id_usuario_geo),
    id_cliente_saas     bigint NOT NULL REFERENCES clientes_saas(id_cliente_saas),
    id_empresa          bigint NOT NULL REFERENCES empresas(id_empresa),
    id_sucursal         bigint NOT NULL REFERENCES sucursales(id_sucursal),
    co_rol              varchar(30) NOT NULL,  -- 'ADMIN', 'OPERADOR', 'READONLY'
    sn_activo           boolean NOT NULL DEFAULT true,
    PRIMARY KEY (id_usuario_geo, id_sucursal)
);
```

### `api_keys`
Claves de autenticación para fuentes externas de ingesta GPS.
```sql
CREATE TABLE api_keys (
    id_api_key          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    id_cliente_saas     bigint NOT NULL REFERENCES clientes_saas(id_cliente_saas),
    id_empresa          bigint NOT NULL REFERENCES empresas(id_empresa),
    co_api_key          varchar(100) NOT NULL UNIQUE,
    de_descripcion      varchar(200),
    co_fuente           varchar(50) NOT NULL,  -- 'GEMA', 'BRIDGE_VGMDIS', 'TRACKER', etc.
    sn_activo           boolean NOT NULL DEFAULT true,
    fe_alta             timestamptz NOT NULL DEFAULT now()
);
```

---

## Tablas operativas

### `entidades`
Party Model: unifica lo que antes eran `empleados` y `puntos_venta`.
```sql
CREATE TABLE entidades (
    id_entidad          bigserial    PRIMARY KEY,
    id_publico          uuid         NOT NULL DEFAULT gen_random_uuid() UNIQUE,
    id_cliente_saas     bigint       NOT NULL REFERENCES clientes_saas(id_cliente_saas),
    id_empresa          bigint       NOT NULL REFERENCES empresas(id_empresa),
    id_sucursal         bigint       NOT NULL REFERENCES sucursales(id_sucursal),
    co_entidad          varchar(50)  NOT NULL,  -- código funcional (id_legajo cuando viene de Bridge VGMDIS)
    de_nombre           varchar(300) NOT NULL,
    -- Coordenadas fijas: solo aplican para rol CLIENTE (ex punto de venta)
    nu_latitud          decimal(10, 7),
    nu_longitud         decimal(10, 7),
    de_direccion        varchar(300),
    -- Puente a VGM Core para integración futura
    id_publico_core     uuid,
    sn_activo           boolean      NOT NULL DEFAULT true,
    fe_alta             timestamptz  NOT NULL DEFAULT now(),
    fe_modificacion     timestamptz,
    UNIQUE (id_empresa, co_entidad)
);
```

### `entidades_roles`
Roles que ejerce una entidad. Una entidad puede tener múltiples roles.
```sql
CREATE TABLE entidades_roles (
    id_entidad_rol  bigserial    PRIMARY KEY,
    id_entidad      bigint       NOT NULL REFERENCES entidades(id_entidad),
    id_cliente_saas bigint       NOT NULL,
    id_empresa      bigint       NOT NULL,
    id_sucursal     bigint       NOT NULL,
    co_rol          varchar(20)  NOT NULL,  -- VENDEDOR, REPARTIDOR, SUPERVISOR, CLIENTE, VEHICULO, ...
    sn_activo       boolean      NOT NULL DEFAULT true,
    fe_alta         timestamptz  NOT NULL DEFAULT now(),
    UNIQUE (id_entidad, co_rol)
);
```

### `zonas`
Áreas geográficas definidas por polígono (territorios de vendedor, zonas de reparto, etc.).
```sql
CREATE TABLE zonas (
    id_zona             bigserial    PRIMARY KEY,
    id_publico          uuid         NOT NULL DEFAULT gen_random_uuid() UNIQUE,
    id_cliente_saas     bigint       NOT NULL REFERENCES clientes_saas(id_cliente_saas),
    id_empresa          bigint       NOT NULL REFERENCES empresas(id_empresa),
    id_sucursal         bigint       NOT NULL REFERENCES sucursales(id_sucursal),
    co_zona             varchar(20)  NOT NULL,
    de_zona             varchar(100) NOT NULL,
    de_color            varchar(7),             -- hex color para el mapa, ej: #FF5733
    sn_activo           boolean      NOT NULL DEFAULT true,
    fe_alta             timestamptz  NOT NULL DEFAULT now(),
    fe_modificacion     timestamptz,
    UNIQUE (id_empresa, co_zona)
);
```

### `posiciones`
Historial de posiciones GPS recibidas de todas las fuentes.
```sql
CREATE TABLE posiciones (
    id_posicion         bigserial      PRIMARY KEY,
    id_publico          uuid           NOT NULL DEFAULT gen_random_uuid() UNIQUE,
    id_entidad          bigint         NOT NULL REFERENCES entidades(id_entidad),
    id_cliente_saas     bigint         NOT NULL,
    id_empresa          bigint         NOT NULL,
    id_sucursal         bigint         NOT NULL,
    nu_latitud          decimal(10, 7) NOT NULL,
    nu_longitud         decimal(10, 7) NOT NULL,
    fe_posicion         timestamptz    NOT NULL,
    nu_precision        decimal(6, 2),
    nu_velocidad        decimal(6, 2),
    nu_altitud          decimal(8, 2),
    co_tipo_operacion   varchar(30),
    co_fuente           varchar(50)    NOT NULL,  -- origen del dato (co_fuente de api_keys)
    fe_recepcion        timestamptz    NOT NULL DEFAULT now()
);
```

---

## Funciones SECURITY DEFINER (ingesta)

El flujo de ingesta GPS necesita acceder a `api_keys` y `entidades` **antes** de que el tenant context esté seteado en sesión. Para resolver este bootstrap sin conceder `BYPASSRLS` al usuario de aplicación, se usan funciones `SECURITY DEFINER`:

| Función | Propósito |
|---|---|
| `buscar_api_key(co_api_key)` | Valida y retorna la API Key sin requerir tenant context |
| `buscar_entidad_ingesta(id_publico, id_empresa, id_cliente_saas)` | Resuelve entidad por UUID público |
| `buscar_entidad_por_codigo(co_entidad, id_empresa, id_cliente_saas)` | Resuelve entidad por `co_entidad` (permite que el Bridge VGMDIS envíe `id_legajo` directamente) |

---

## Convenciones de nombres

Idénticas a VGM Core:

| Prefijo | Significado | Ejemplo |
|---|---|---|
| `id_` | Identificador interno | `id_entidad` |
| `co_` | Código funcional | `co_entidad`, `co_rol` |
| `de_` | Descripción o texto | `de_nombre` |
| `nu_` | Número o medida | `nu_latitud` |
| `sn_` | Sí/No (boolean) | `sn_activo` |
| `fe_` | Fecha | `fe_alta` |

Toda tabla tiene `id_publico UUID` para exponer en APIs. El ID interno (`bigint`) **nunca** se expone en APIs.

---

## Integración entre productos

VGM Core Geo, VGM Core y VGM GEMA se comunican **únicamente por API REST**. Nunca por base de datos compartida.

El campo `id_publico_core` en `entidades` es el puente técnico para sincronización futura con VGM Core.
