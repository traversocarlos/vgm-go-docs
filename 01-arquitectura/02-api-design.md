# VGM Core Geo — Diseño de API

**Versión:** 3.0
**Fecha:** 2026-03-27

## Resumen ejecutivo

**VGM Core Geo** es el sistema de geolocalización en tiempo real para empresas de distribución y ventas. Reemplaza Ultra GEO (ASP.NET + Google Maps) con un stack moderno: Kotlin 2.1 + Spring Boot 3.5 + PostgreSQL 17 en el backend, y React 18 + TypeScript + Leaflet.js + OpenStreetMap en el frontend.

La API expone dos tipos de contrato:

1. **Ingesta GPS** — recibe posiciones desde fuentes externas (app móvil, bridge legado, trackers hardware, sistemas externos). Usa autenticación por API Key.
2. **Frontend / administración** — consulta de posiciones, gestión de entidades y zonas. Usa JWT Bearer con contexto de empresa y sucursal activa.

Todos los recursos siguen una jerarquía de multi-tenancy: `clientes_saas → empresas → sucursales → entidades`.

---

## Autenticación

### Esquemas disponibles

| Esquema | Header | Usado en |
|---|---|---|
| API Key | `X-Api-Key: {clave}` | Endpoints de ingesta GPS |
| JWT Bearer | `Authorization: Bearer {token}` | Endpoints de frontend/admin |

### Headers obligatorios en endpoints JWT

Además del token, los endpoints del frontend requieren que el cliente informe el contexto de empresa y sucursal activa:

```
Authorization: Bearer eyJ...
X-Empresa-Id: 7
X-Sucursal-Id: 3
```

El backend filtra todos los datos según estos headers. Una misma cuenta de usuario puede tener acceso a múltiples empresas o sucursales — la sesión indica cuáles tiene disponibles.

### Flujo de autenticación (Etapa 1 — JWT propio)

```
[Cliente]  →  POST /api/v1/auth/login  →  [Backend]
                email + password
                                        ← JWT (expires 8h)

[Cliente]  →  GET /api/v1/sesion/yo    →  [Backend]
                Bearer {jwt}
                                        ← empresas + sucursales disponibles

[Cliente]  →  Cualquier endpoint       →  [Backend]
                Bearer {jwt}
                X-Empresa-Id
                X-Sucursal-Id
```

> Etapa 2: se conecta Auth0 como proveedor. Solo cambia `application.yml` — el resto del código queda igual (ADR-003).

### Fuentes de datos GPS y su autenticación

| Fuente | Mecanismo |
|---|---|
| App móvil Android (GEMA) | API Key |
| Bridge VGMDIS (SQL Server legado) | API Key |
| GPS Tracker hardware | API Key |
| Sistema externo (Traccar, Wialon, etc.) | API Key |
| Importación CSV / Excel / GPX | Sesión JWT |
| Carga manual desde web | Sesión JWT |

---

## Módulos de la API

### 1. Autenticación

| Método | Path | Auth | Descripción |
|---|---|---|---|
| POST | `/api/v1/auth/login` | Ninguna | Obtiene JWT a partir de email y password |

### 2. Sesión

| Método | Path | Auth | Descripción |
|---|---|---|---|
| GET | `/api/v1/sesion/yo` | JWT | Devuelve el usuario autenticado con sus empresas y sucursales disponibles |
| GET | `/api/v1/sesion/sucursales` | JWT | Sucursales disponibles del usuario |

### 3. Ingesta de posiciones

| Método | Path | Auth | Descripción |
|---|---|---|---|
| POST | `/api/v1/posiciones` | API Key | Registra una posición GPS recibida de cualquier fuente |

### 4. Consulta de posiciones

| Método | Path | Auth | Descripción |
|---|---|---|---|
| GET | `/api/v1/posiciones/actuales` | JWT | Última posición de todas las entidades móviles de la sucursal activa |
| GET | `/api/v1/posiciones/actuales/{idPublicoEntidad}` | JWT | Última posición de una entidad específica |
| GET | `/api/v1/posiciones/historial/{idPublicoEntidad}` | JWT | Historial de posiciones de una entidad con filtros |

### 5. Entidades

| Método | Path | Auth | Descripción |
|---|---|---|---|
| GET | `/api/v1/entidades` | JWT | Listado paginado de entidades (filtrable por rol) |
| GET | `/api/v1/entidades/{idPublico}` | JWT | Detalle de una entidad |
| POST | `/api/v1/entidades` | JWT | Crea una nueva entidad |
| PUT | `/api/v1/entidades/{idPublico}` | JWT | Actualiza una entidad |
| DELETE | `/api/v1/entidades/{idPublico}` | JWT | Desactiva una entidad (soft delete) |

### 6. Zonas

| Método | Path | Auth | Descripción |
|---|---|---|---|
| GET | `/api/v1/zonas` | JWT | Listado de zonas |
| GET | `/api/v1/zonas/{idPublico}` | JWT | Detalle de una zona |
| POST | `/api/v1/zonas` | JWT | Crea una nueva zona con polígono |
| PUT | `/api/v1/zonas/{idPublico}` | JWT | Actualiza una zona |
| DELETE | `/api/v1/zonas/{idPublico}` | JWT | Desactiva una zona (soft delete) |

---

## Detalle de payloads

### Login

**Request** `POST /api/v1/auth/login`

```json
{
  "email": "gustavo@vgm.com.ar",
  "password": "********"
}
```

**Response 200**

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expira": "2026-03-18T10:00:00Z"
}
```

---

### Sesión

**Response 200** `GET /api/v1/sesion/yo`

```json
{
  "idPublico": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "nombre": "Gustavo Rodríguez",
  "email": "gustavo@vgm.com.ar",
  "sucursales": [
    {
      "idSucursal": 3,
      "deSucursal": "Casa Central",
      "idEmpresa": 7,
      "deEmpresa": "Distribuidora Sur S.A.",
      "coRol": "ADMIN",
      "activa": true
    }
  ]
}
```

---

### Ingesta de posición GPS

**Request** `POST /api/v1/posiciones`
**Header**: `X-Api-Key: {clave_de_fuente}`

El payload acepta identificar la entidad de dos formas alternativas:

- `idEntidad` (UUID) — cuando la fuente ya conoce el `id_publico` de la entidad en VGM Core Geo.
- `coEntidad` (string) — cuando la fuente envía un código funcional. Permite que el Bridge VGMDIS envíe directamente el `id_legajo` sin necesidad de lookup previo.

Al menos uno de los dos debe estar presente.

```json
{
  "idEntidad": "f7e6d5c4-b3a2-1098-7654-321098fedcba",
  "coEntidad": "1042",
  "latitud": -34.603722,
  "longitud": -58.381592,
  "fechaPosicion": "2026-03-17T09:45:00Z",
  "precision": 8.5,
  "velocidad": 45.2,
  "altitud": 25.0,
  "tipoOperacion": "VENTA"
}
```

| Campo | Tipo | Requerido | Descripción |
|---|---|---|---|
| `idEntidad` | UUID | Condicional | `id_publico` de la entidad. Requerido si no viene `coEntidad` |
| `coEntidad` | string (max 50) | Condicional | Código funcional. Requerido si no viene `idEntidad` |
| `latitud` | decimal | Sí | Latitud WGS84 |
| `longitud` | decimal | Sí | Longitud WGS84 |
| `fechaPosicion` | ISO 8601 | Sí | Timestamp del momento de la posición (con timezone) |
| `precision` | decimal | No | Precisión GPS en metros |
| `velocidad` | decimal | No | Velocidad en km/h |
| `altitud` | decimal | No | Altitud en metros sobre el nivel del mar |
| `tipoOperacion` | string | No | Tipo de operación registrada en ese momento |

**Tipos de operación válidos**: `VENTA`, `RECIBO`, `ENVIO_PERIODICO`, `NO_ATENCION`

**Response 201**

```json
{
  "idPosicion": "cc3d4e5f-6a7b-8c9d-0e1f-2a3b4c5d6e7f",
  "fechaRegistro": "2026-03-17T09:45:01Z"
}
```

---

### Consulta de posiciones actuales

**Response 200** `GET /api/v1/posiciones/actuales`

Devuelve la última posición conocida de cada entidad móvil activa (roles: `VENDEDOR`, `REPARTIDOR`, `SUPERVISOR`) en la sucursal indicada por `X-Sucursal-Id`.

```json
[
  {
    "idEntidad": "f7e6d5c4-b3a2-1098-7654-321098fedcba",
    "deNombre": "Juan Pérez",
    "coRol": "VENDEDOR",
    "latitud": -34.603722,
    "longitud": -58.381592,
    "fechaPosicion": "2026-03-17T09:45:00Z",
    "precision": 8.5,
    "velocidad": 45.2,
    "altitud": 25.0,
    "tipoOperacion": "VENTA",
    "minutosDesdeUltimaPosicion": 3
  }
]
```

---

### Historial de posiciones

**Parámetros de query** `GET /api/v1/posiciones/historial/{idPublicoEntidad}`

| Parámetro | Tipo | Descripción |
|---|---|---|
| `fecha` | YYYY-MM-DD | Filtra por día completo |
| `desde` | ISO 8601 datetime | Inicio del rango (alternativa a `fecha`) |
| `hasta` | ISO 8601 datetime | Fin del rango (alternativa a `fecha`) |
| `tipoOperacion` | string | Filtra por tipo de operación |

`fecha` y `desde`/`hasta` son mutuamente excluyentes.

---

### Entidades

**Request** `POST /api/v1/entidades` / `PUT /api/v1/entidades/{idPublico}`

```json
{
  "coEntidad": "1042",
  "deNombre": "Juan Pérez",
  "roles": ["VENDEDOR"],
  "nuLatitud": null,
  "nuLongitud": null,
  "deDireccion": null,
  "snActivo": true
}
```

> Para una entidad con rol `CLIENTE` (ex punto de venta), incluir `nuLatitud`, `nuLongitud` y `deDireccion`.

**Response 200/201**

```json
{
  "idPublico": "f7e6d5c4-b3a2-1098-7654-321098fedcba",
  "coEntidad": "1042",
  "deNombre": "Juan Pérez",
  "roles": ["VENDEDOR"],
  "nuLatitud": null,
  "nuLongitud": null,
  "deDireccion": null,
  "snActivo": true,
  "feAlta": "2026-01-15T10:00:00Z"
}
```

**Listado paginado** `GET /api/v1/entidades?rol=VENDEDOR&activo=true&pagina=0&tamanio=20`

| Parámetro | Descripción |
|---|---|
| `rol` | Filtra por rol: `VENDEDOR`, `REPARTIDOR`, `SUPERVISOR`, `CLIENTE` |
| `activo` | Default `true` |
| `buscar` | Búsqueda por nombre o código |
| `pagina` | Default `0` |
| `tamanio` | Default `20`, máximo `100` |

```json
{
  "contenido": [
    {
      "idPublico": "f7e6d5c4-b3a2-1098-7654-321098fedcba",
      "coEntidad": "1042",
      "deNombre": "Juan Pérez",
      "roles": ["VENDEDOR"],
      "snActivo": true
    }
  ],
  "pagina": 0,
  "tamanio": 20,
  "totalElementos": 150,
  "totalPaginas": 8
}
```

---

### Zonas

**Request** `POST /api/v1/zonas` / `PUT /api/v1/zonas/{idPublico}`

```json
{
  "coZona": "NORTE",
  "deZona": "Zona Norte - Vendedor Pérez",
  "deColor": "#FF5733",
  "coordenadas": [
    [-58.390000, -34.590000],
    [-58.370000, -34.590000],
    [-58.370000, -34.610000],
    [-58.390000, -34.610000],
    [-58.390000, -34.590000]
  ]
}
```

**Response 200/201**

```json
{
  "idPublico": "b9c8d7e6-f5a4-3b2c-1d0e-9f8a7b6c5d4e",
  "coZona": "NORTE",
  "deZona": "Zona Norte - Vendedor Pérez",
  "deColor": "#FF5733",
  "coordenadas": [...]
}
```

---

## Respuestas estándar

### Error

Todos los errores devuelven el mismo envelope independientemente del código HTTP:

```json
{
  "codigo": "ENTIDAD_NO_ENCONTRADA",
  "mensaje": "No se encontró la entidad con id f7e6d5c4-b3a2-1098-7654-321098fedcba",
  "correlationId": "a1b2c3d4-0000-0000-0000-000000000000",
  "timestamp": "2026-03-17T10:00:00Z"
}
```

| Campo | Descripción |
|---|---|
| `codigo` | Código de error legible por la app, en SCREAMING_SNAKE_CASE |
| `mensaje` | Descripción para logs o mensajes de error |
| `correlationId` | UUID de la request para rastrear en logs |
| `timestamp` | Momento en que ocurrió el error |

### Paginación

Los endpoints de listado devuelven siempre el mismo envelope paginado:

```json
{
  "contenido": [],
  "pagina": 0,
  "tamanio": 20,
  "totalElementos": 0,
  "totalPaginas": 0
}
```

La paginación es base 0. El tamaño por defecto es 20.

---

## Mapeo VGMDIS → VGM Core Geo

VGMDIS usa `id_legajo` (entero) como identificador de persona en `i_rrhh`.

VGM Core Geo usa `co_entidad` (varchar 50) como código funcional. Cuando se importan datos desde VGMDIS:

```
co_entidad = id_legajo.toString()
```

El Bridge VGMDIS puede enviar posiciones usando `coEntidad = "1042"` sin necesidad de conocer el UUID interno de la entidad. El backend resuelve la correspondencia internamente vía la función `buscar_entidad_por_codigo`.

---

## Convenciones generales

### Prefijos de columnas (heredado de VGM Core)

| Prefijo | Significado |
|---|---|
| `id_` | Identificador interno (bigint). **Nunca se expone en la API.** |
| `co_` | Código funcional |
| `de_` | Descripción o texto |
| `nu_` | Número o medida |
| `sn_` | Boolean (sí/no) |
| `fe_` | Fecha |

### IDs en la API

Toda entidad expone `idPublico` (UUID v4). Los IDs internos nunca aparecen en respuestas ni en rutas.

### Formato de fechas

Todas las fechas en ISO 8601 con timezone explícito (UTC, sufijo `Z`).

### Versionado

Todos los endpoints bajo `/api/v1/`. El versionado es por URL.

### Códigos HTTP usados

| Código | Uso |
|---|---|
| 200 | OK — GET, PUT exitoso |
| 201 | Created — POST exitoso |
| 204 | No Content — DELETE exitoso |
| 400 | Bad Request — payload inválido |
| 401 | Unauthorized — JWT ausente/inválido o API Key inválida |
| 403 | Forbidden — sin acceso al recurso |
| 404 | Not Found — recurso no encontrado |
| 409 | Conflict — código funcional duplicado |
| 500 | Internal Server Error |
