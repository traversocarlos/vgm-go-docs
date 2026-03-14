# Modelo de datos

**Versión:** 2.1
**Fecha:** 2026-03-14
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
              ├── empleados (vendedores, repartidores)
              ├── puntos_venta
              └── zonas
```

**Ejemplo real:**
```
D OUNI (cliente_saas)
  ├── OV (empresa)
  │     ├── Sucursal Centro → vendedores, puntos de venta
  │     └── Sucursal Norte  → vendedores, puntos de venta
  ├── ZIRKA (empresa)
  │     └── Sucursal única  → vendedores, puntos de venta
  └── OV2 (empresa)
        └── Sucursal única  → vendedores, puntos de venta
```

---

## Tablas de tenancy

### `clientes_saas`
```sql
CREATE TABLE clientes_saas (
    id_cliente_saas     bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    id_publico          uuid NOT NULL DEFAULT gen_random_uuid(),
    co_cliente_saas     varchar(50) NOT NULL,
    de_cliente_saas     varchar(200) NOT NULL,
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
Identidad global. Un usuario puede tener cuentas en múltiples tenants.
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

---

## Tablas operativas

### `empleados`
```sql
CREATE TABLE empleados (
    id_empleado         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    id_publico          uuid NOT NULL DEFAULT gen_random_uuid(),
    id_cliente_saas     bigint NOT NULL REFERENCES clientes_saas(id_cliente_saas),
    id_empresa          bigint NOT NULL REFERENCES empresas(id_empresa),
    id_sucursal         bigint NOT NULL REFERENCES sucursales(id_sucursal),
    co_empleado         varchar(50) NOT NULL,
    de_nombre           varchar(200) NOT NULL,
    co_tipo             varchar(20) NOT NULL,  -- 'VENDEDOR', 'REPARTIDOR', 'SUPERVISOR'
    sn_registra_coords  boolean NOT NULL DEFAULT true,
    sn_activo           boolean NOT NULL DEFAULT true,
    fe_alta             timestamptz NOT NULL DEFAULT now(),
    id_publico_core     uuid NULL,             -- puente para integración futura con VGM Core
    CONSTRAINT uq_empleado_publico UNIQUE (id_publico)
);
```

### `posiciones`
```sql
CREATE TABLE posiciones (
    id_posicion         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    id_cliente_saas     bigint NOT NULL,
    id_empresa          bigint NOT NULL,
    id_sucursal         bigint NOT NULL,
    id_empleado         bigint NOT NULL REFERENCES empleados(id_empleado),
    nu_latitud          numeric(10,7) NOT NULL,
    nu_longitud         numeric(10,7) NOT NULL,
    nu_precision        numeric(8,2),
    nu_velocidad        numeric(8,2),
    co_tipo_operacion   varchar(30),
    fe_posicion         timestamptz NOT NULL,
    fe_recibida         timestamptz NOT NULL DEFAULT now()
);
```

### `puntos_venta`
```sql
CREATE TABLE puntos_venta (
    id_punto_venta      bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    id_publico          uuid NOT NULL DEFAULT gen_random_uuid(),
    id_cliente_saas     bigint NOT NULL REFERENCES clientes_saas(id_cliente_saas),
    id_empresa          bigint NOT NULL REFERENCES empresas(id_empresa),
    id_sucursal         bigint NOT NULL REFERENCES sucursales(id_sucursal),
    co_punto_venta      varchar(50) NOT NULL,
    de_nombre           varchar(200) NOT NULL,
    nu_latitud          numeric(10,7),
    nu_longitud         numeric(10,7),
    sn_activo           boolean NOT NULL DEFAULT true,
    fe_alta             timestamptz NOT NULL DEFAULT now(),
    id_publico_core     uuid NULL,             -- puente para integración futura con VGM Core
    CONSTRAINT uq_punto_venta_publico UNIQUE (id_publico)
);
```

### `zonas`
```sql
CREATE TABLE zonas (
    id_zona             bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    id_cliente_saas     bigint NOT NULL REFERENCES clientes_saas(id_cliente_saas),
    id_empresa          bigint NOT NULL REFERENCES empresas(id_empresa),
    id_sucursal         bigint NOT NULL REFERENCES sucursales(id_sucursal),
    de_nombre           varchar(200) NOT NULL,
    de_color            varchar(7),
    de_coordenadas      jsonb NOT NULL,
    sn_activo           boolean NOT NULL DEFAULT true,
    fe_alta             timestamptz NOT NULL DEFAULT now()
);
```

---

## Convenciones de nombres

Idénticas a VGM Core:

| Prefijo | Significado | Ejemplo |
|---|---|---|
| `id_` | Identificador interno | `id_empleado` |
| `co_` | Código funcional | `co_tipo` |
| `de_` | Descripción o texto | `de_nombre` |
| `nu_` | Número o medida | `nu_latitud` |
| `sn_` | Sí/No (boolean) | `sn_activo` |
| `fe_` | Fecha | `fe_alta` |

Toda tabla tiene `id_publico UUID` para exponer en APIs (nunca se expone el ID interno).

---

## Integración entre productos

VGM Core Geo, VGM Core y VGM GEMA se comunican **únicamente por API REST**. Nunca por base de datos compartida.

Los campos `id_publico_core` en `empleados` y `puntos_venta` son el puente técnico para sincronización futura entre productos.
