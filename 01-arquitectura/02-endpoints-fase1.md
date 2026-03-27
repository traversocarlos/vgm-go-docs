# Contrato de endpoints — Fase 1

**Versión:** 2.0
**Fecha:** 2026-03-27
**Estado:** Activo

> Diseño de referencia. La fuente de verdad es el código + `/v3/api-docs` (generado por springdoc-openapi).
> Ver convenciones generales en `01-arquitectura/01-convenciones-api.md`.

---

## Resumen de endpoints

| Método | Path | Descripción | Auth |
|---|---|---|---|
| `POST` | `/api/v1/auth/login` | Login — emite JWT | Público |
| `GET` | `/api/v1/sesion/yo` | Datos del usuario actual | JWT |
| `GET` | `/api/v1/sesion/sucursales` | Sucursales disponibles del usuario | JWT |
| `POST` | `/api/v1/posiciones` | Ingresar posición GPS | API Key |
| `GET` | `/api/v1/posiciones/actuales` | Última posición de entidades móviles | JWT + Empresa + Sucursal |
| `GET` | `/api/v1/posiciones/actuales/{idPublico}` | Última posición de una entidad | JWT + Empresa + Sucursal |
| `GET` | `/api/v1/posiciones/historial/{idPublico}` | Historial de posiciones de una entidad | JWT + Empresa + Sucursal |
| `GET` | `/api/v1/entidades` | Listar entidades (filtrable por rol) | JWT + Empresa + Sucursal |
| `GET` | `/api/v1/entidades/{idPublico}` | Detalle de entidad | JWT + Empresa + Sucursal |
| `POST` | `/api/v1/entidades` | Crear entidad | JWT + Empresa + Sucursal |
| `PUT` | `/api/v1/entidades/{idPublico}` | Actualizar entidad | JWT + Empresa + Sucursal |
| `DELETE` | `/api/v1/entidades/{idPublico}` | Baja lógica de entidad | JWT + Empresa + Sucursal |
| `GET` | `/api/v1/zonas` | Listar zonas | JWT + Empresa + Sucursal |
| `GET` | `/api/v1/zonas/{idPublico}` | Detalle de zona | JWT + Empresa + Sucursal |
| `POST` | `/api/v1/zonas` | Crear zona | JWT + Empresa + Sucursal |
| `PUT` | `/api/v1/zonas/{idPublico}` | Actualizar zona | JWT + Empresa + Sucursal |
| `DELETE` | `/api/v1/zonas/{idPublico}` | Baja lógica de zona | JWT + Empresa + Sucursal |
| `GET` | `/actuator/health` | Health check | Público |

---

## Módulo autenticación

### `POST /api/v1/auth/login`

**No requiere autenticación.**

**Request body:**
```json
{
  "email": "usuario@empresa.com",
  "password": "contraseña"
}
```

**Response 200:**
```json
{
  "token": "eyJ...",
  "expiracion": "2026-03-14T11:30:00Z"
}
```

**Errores:**

| Status | Código | Cuándo |
|---|---|---|
| 401 | `CREDENCIALES_INVALIDAS` | Email o contraseña incorrectos |
| 403 | `USUARIO_INACTIVO` | La cuenta existe pero está deshabilitada |

---

## Módulo sesión

### `GET /api/v1/sesion/yo`

No requiere `X-Empresa-Id` ni `X-Sucursal-Id` — opera solo con el `tenant_id` del JWT.

**Response 200:**
```json
{
  "idUsuarioGeo": 1,
  "email": "usuario@empresa.com",
  "sub": "auth0|abc123",
  "sucursales": [
    {
      "idSucursal": 3,
      "deSucursal": "Sucursal Centro",
      "idEmpresa": 10,
      "deEmpresa": "OV",
      "coRol": "ADMIN",
      "activa": true
    }
  ]
}
```

---

### `GET /api/v1/sesion/sucursales`

No requiere `X-Empresa-Id` ni `X-Sucursal-Id`.

**Response 200:**
```json
[
  {
    "idSucursal": 3,
    "deSucursal": "Sucursal Centro",
    "idEmpresa": 10,
    "deEmpresa": "OV",
    "coRol": "ADMIN"
  }
]
```

---

## Módulo posiciones

### `POST /api/v1/posiciones`

Ingesta GPS — autenticado por **API Key** (`X-Api-Key` header). Cada fuente tiene su propia clave.

```json
{
  "idEntidad": "uuid-publico-de-la-entidad",
  "coEntidad": "1042",
  "latitud": -27.4516,
  "longitud": -58.9867,
  "precision": 5.2,
  "velocidad": 0.0,
  "altitud": 150.0,
  "tipoOperacion": "VENTA",
  "fechaPosicion": "2026-03-14T09:30:00Z"
}
```

> `idEntidad` o `coEntidad` — al menos uno requerido. El Bridge VGMDIS usa `coEntidad` con el `id_legajo`.

**Response 201:**
```json
{
  "idPosicion": "uuid-de-la-posicion",
  "fechaRegistro": "2026-03-14T09:30:01Z"
}
```

**Errores:**

| Status | Código | Cuándo |
|---|---|---|
| 401 | `API_KEY_INVALIDA` | API Key ausente o inválida |
| 404 | `ENTIDAD_NO_ENCONTRADA` | La entidad no existe o no pertenece al tenant de la API Key |
| 400 | `VALIDACION_ERROR` | Campos obligatorios faltantes o formato inválido |

---

### `GET /api/v1/posiciones/actuales`

Última posición de todas las entidades móviles activas (roles `VENDEDOR`, `REPARTIDOR`, `SUPERVISOR`) de la sucursal activa.

**Response 200:**
```json
[
  {
    "idEntidad": "uuid-publico",
    "deNombre": "Juan García",
    "coRol": "VENDEDOR",
    "latitud": -27.4516,
    "longitud": -58.9867,
    "fechaPosicion": "2026-03-14T09:30:00Z",
    "minutosDesdeUltimaPosicion": 3
  }
]
```

---

### `GET /api/v1/posiciones/historial/{idPublico}`

**Query params:**

| Param | Tipo | Descripción |
|---|---|---|
| `fecha` | YYYY-MM-DD | Día completo (mutuamente excluyente con desde/hasta) |
| `desde` | ISO 8601 | Inicio del rango |
| `hasta` | ISO 8601 | Fin del rango |
| `tipoOperacion` | string | Filtra por tipo de operación |

---

## Módulo entidades

### `GET /api/v1/entidades`

**Query params opcionales:**

| Param | Tipo | Descripción |
|---|---|---|
| `rol` | string | `VENDEDOR`, `REPARTIDOR`, `SUPERVISOR`, `CLIENTE` |
| `buscar` | string | Búsqueda por nombre o código |
| `activo` | boolean | Default `true` |
| `pagina` | int | Default `0` |
| `tamanio` | int | Default `20`, máximo `100` |

**Response 200:**
```json
{
  "contenido": [
    {
      "idPublico": "uuid-publico",
      "coEntidad": "1042",
      "deNombre": "Juan García",
      "roles": ["VENDEDOR"],
      "snActivo": true,
      "feAlta": "2026-01-15T10:00:00Z"
    }
  ],
  "pagina": 0,
  "tamanio": 20,
  "totalElementos": 45,
  "totalPaginas": 3
}
```

---

### `GET /api/v1/entidades/{idPublico}`

**Response 200:**
```json
{
  "idPublico": "uuid-publico",
  "coEntidad": "1042",
  "deNombre": "Juan García",
  "roles": ["VENDEDOR"],
  "nuLatitud": null,
  "nuLongitud": null,
  "deDireccion": null,
  "snActivo": true,
  "feAlta": "2026-01-15T10:00:00Z",
  "idPublicoCore": null
}
```

**Errores:**

| Status | Código | Cuándo |
|---|---|---|
| 404 | `ENTIDAD_NO_ENCONTRADA` | No existe o no pertenece a la sucursal activa |

---

### `POST /api/v1/entidades`

**Request body:**
```json
{
  "coEntidad": "1043",
  "deNombre": "María López",
  "roles": ["REPARTIDOR"],
  "nuLatitud": null,
  "nuLongitud": null,
  "deDireccion": null
}
```

> Para un `CLIENTE` (ex punto de venta), incluir `nuLatitud`, `nuLongitud` y opcionalmente `deDireccion`.

**Response 201** + header `Location: /api/v1/entidades/{uuid}`.

**Errores:**

| Status | Código | Cuándo |
|---|---|---|
| 400 | `VALIDACION_ERROR` | Campos obligatorios faltantes |
| 409 | `CODIGO_DUPLICADO` | Ya existe una entidad con ese `coEntidad` en la sucursal |

---

### `PUT /api/v1/entidades/{idPublico}`

Envía el objeto completo. Response 200 con la entidad actualizada.

---

### `DELETE /api/v1/entidades/{idPublico}`

Baja lógica (`sn_activo = false`). Response 204 sin body.

---

## Módulo zonas

### `GET /api/v1/zonas`

Response 200 — array sin paginación (las zonas son pocas):
```json
[
  {
    "idPublico": "uuid-publico",
    "coZona": "NORTE",
    "deZona": "Zona Norte",
    "deColor": "#3B82F6",
    "coordenadas": [
      [-27.430, -58.970],
      [-27.430, -58.950],
      [-27.450, -58.950],
      [-27.450, -58.970]
    ],
    "snActivo": true
  }
]
```

---

### `POST /api/v1/zonas`

```json
{
  "coZona": "SUR",
  "deZona": "Zona Sur",
  "deColor": "#EF4444",
  "coordenadas": [
    [-27.470, -58.990],
    [-27.470, -58.960],
    [-27.490, -58.960],
    [-27.490, -58.990]
  ]
}
```

Response 201 + `Location`.

---

### `PUT /api/v1/zonas/{idPublico}` / `DELETE /api/v1/zonas/{idPublico}`

PUT actualiza completo. DELETE hace baja lógica (`sn_activo = false`), response 204.

---

## Permisos por rol de usuario

| Operación | ADMIN | OPERADOR | READONLY |
|---|---|---|---|
| Ver posiciones | ✓ | ✓ | ✓ |
| Ingresar posición (API Key) | ✓ | ✓ | — |
| Ver entidades / zonas | ✓ | ✓ | ✓ |
| Crear / editar entidades y zonas | ✓ | ✓ | — |
| Dar de baja entidades y zonas | ✓ | — | — |
| Gestionar usuarios | ✓ | — | — |

Los permisos se validan en el backend consultando `usuarios_geo_sucursales.co_rol` — nunca desde el JWT.
