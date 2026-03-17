# VGM Core Geo — Diseño de API

## Resumen ejecutivo

**VGM Core Geo** es el sistema de geolocalización en tiempo real para empresas de distribución y ventas. Reemplaza Ultra GEO (ASP.NET + Google Maps) con un stack moderno: Kotlin 2.1 + Spring Boot 3.5 + PostgreSQL 17 en el backend, y React 18 + TypeScript + Leaflet.js + OpenStreetMap en el frontend.

La API expone dos tipos de contrato:

1. **Ingesta GPS** — recibe posiciones desde fuentes externas (app móvil, bridge legado, trackers hardware, sistemas externos). Usa autenticación por API Key.
2. **Frontend / administración** — consulta de posiciones, gestión de empleados, puntos de venta y zonas. Usa JWT Bearer con contexto de empresa y sucursal activa.

Todos los recursos siguen una jerarquía de multi-tenancy: `clientes_saas → empresas → sucursales → empleados`.

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

### Flujo de autenticación (Etapa 1)

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

### 3. Ingesta de posiciones

| Método | Path | Auth | Descripción |
|---|---|---|---|
| POST | `/api/v1/posiciones` | API Key | Registra una posición GPS recibida de cualquier fuente |

### 4. Consulta de posiciones

| Método | Path | Auth | Descripción |
|---|---|---|---|
| GET | `/api/v1/posiciones/actuales` | JWT | Última posición de todos los empleados de la sucursal activa |
| GET | `/api/v1/posiciones/actuales/{idPublicoEmpleado}` | JWT | Última posición de un empleado específico |
| GET | `/api/v1/posiciones/historial/{idPublicoEmpleado}` | JWT | Historial de posiciones de un empleado con filtros |

### 5. Empleados

| Método | Path | Auth | Descripción |
|---|---|---|---|
| GET | `/api/v1/empleados` | JWT | Listado paginado de empleados |
| GET | `/api/v1/empleados/{idPublico}` | JWT | Detalle de un empleado |
| POST | `/api/v1/empleados` | JWT | Crea un nuevo empleado |
| PUT | `/api/v1/empleados/{idPublico}` | JWT | Actualiza un empleado |
| DELETE | `/api/v1/empleados/{idPublico}` | JWT | Desactiva un empleado (soft delete) |

### 6. Puntos de venta

| Método | Path | Auth | Descripción |
|---|---|---|---|
| GET | `/api/v1/puntos-venta` | JWT | Listado paginado de puntos de venta |
| GET | `/api/v1/puntos-venta/{idPublico}` | JWT | Detalle de un punto de venta |
| POST | `/api/v1/puntos-venta` | JWT | Crea un nuevo punto de venta |
| PUT | `/api/v1/puntos-venta/{idPublico}` | JWT | Actualiza un punto de venta |
| DELETE | `/api/v1/puntos-venta/{idPublico}` | JWT | Elimina un punto de venta |

### 7. Zonas

| Método | Path | Auth | Descripción |
|---|---|---|---|
| GET | `/api/v1/zonas` | JWT | Listado de zonas |
| GET | `/api/v1/zonas/{id}` | JWT | Detalle de una zona |
| POST | `/api/v1/zonas` | JWT | Crea una nueva zona con polígono GeoJSON |
| PUT | `/api/v1/zonas/{id}` | JWT | Actualiza una zona |
| DELETE | `/api/v1/zonas/{id}` | JWT | Elimina una zona |

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
  "empresas": [
    {
      "id": 7,
      "descripcion": "Distribuidora Sur S.A.",
      "sucursales": [
        { "id": 3, "descripcion": "Casa Central" },
        { "id": 4, "descripcion": "Sucursal Norte" }
      ]
    }
  ]
}
```

---

### Ingesta de posición GPS

**Request** `POST /api/v1/posiciones`
**Header**: `X-Api-Key: {clave_de_fuente}`

El payload acepta identificar al empleado de dos formas alternativas:

- `idEmpleado` (UUID) — cuando la fuente ya conoce el ID público del empleado en VGM Core Geo.
- `coEmpleado` (string) — cuando la fuente envía un código funcional. Esto permite que el Bridge VGMDIS envíe directamente el `id_legajo` sin necesidad de hacer un lookup previo (ver sección sobre mapeo VGMDIS más abajo).

Al menos uno de los dos debe estar presente.

```json
{
  "idEmpleado": "f7e6d5c4-b3a2-1098-7654-321098fedcba",
  "coEmpleado": "1042",
  "latitud": -34.603722,
  "longitud": -58.381592,
  "fechaPosicion": "2026-03-17T09:45:00Z",
  "precision": 8.5,
  "velocidad": 45.2,
  "altitud": 25.0,
  "tipoOperacion": "VENTA"
}
```

Campos:

| Campo | Tipo | Requerido | Descripción |
|---|---|---|---|
| `idEmpleado` | UUID | Condicional | ID público del empleado. Requerido si no viene `coEmpleado` |
| `coEmpleado` | string (max 50) | Condicional | Código funcional del empleado. Requerido si no viene `idEmpleado` |
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

Devuelve la última posición conocida de cada empleado activo en la sucursal indicada por `X-Sucursal-Id`.

```json
[
  {
    "idEmpleado": "f7e6d5c4-b3a2-1098-7654-321098fedcba",
    "nombreEmpleado": "Juan Pérez",
    "tipoEmpleado": "VENDEDOR",
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

**Parámetros de query** `GET /api/v1/posiciones/historial/{idPublicoEmpleado}`

| Parámetro | Tipo | Descripción |
|---|---|---|
| `fecha` | YYYY-MM-DD | Filtra por día completo (conveniente para el mapa de recorrido diario) |
| `desde` | ISO 8601 datetime | Inicio del rango (alternativa a `fecha`) |
| `hasta` | ISO 8601 datetime | Fin del rango (alternativa a `fecha`) |
| `tipoOperacion` | string | Filtra por tipo de operación |

`fecha` y `desde`/`hasta` son mutuamente excluyentes. Si se envía `fecha`, se ignoran los otros dos.

**Response 200**

```json
[
  {
    "latitud": -34.603722,
    "longitud": -58.381592,
    "fechaPosicion": "2026-03-17T08:00:00Z",
    "precision": 6.0,
    "velocidad": 0.0,
    "altitud": 24.0,
    "tipoOperacion": "ENVIO_PERIODICO"
  },
  {
    "latitud": -34.612500,
    "longitud": -58.393100,
    "fechaPosicion": "2026-03-17T09:45:00Z",
    "precision": 8.5,
    "velocidad": 45.2,
    "altitud": 25.0,
    "tipoOperacion": "VENTA"
  }
]
```

---

### Empleados

**Request** `POST /api/v1/empleados` / `PUT /api/v1/empleados/{idPublico}`

```json
{
  "coEmpleado": "1042",
  "nombre": "Juan Pérez",
  "tipo": "VENDEDOR",
  "snActivo": true
}
```

**Response 200/201**

```json
{
  "idPublico": "f7e6d5c4-b3a2-1098-7654-321098fedcba",
  "coEmpleado": "1042",
  "nombre": "Juan Pérez",
  "tipo": "VENDEDOR",
  "snActivo": true
}
```

**Listado paginado** `GET /api/v1/empleados?activo=true&tipo=VENDEDOR&pagina=0&tamanio=20`

```json
{
  "contenido": [
    {
      "idPublico": "f7e6d5c4-b3a2-1098-7654-321098fedcba",
      "coEmpleado": "1042",
      "nombre": "Juan Pérez",
      "tipo": "VENDEDOR",
      "snActivo": true
    }
  ],
  "pagina": 0,
  "tamanio": 20,
  "totalElementos": 150,
  "totalPaginas": 8
}
```

Tipos de empleado válidos: `VENDEDOR`, `REPARTIDOR`, `SUPERVISOR`

---

### Puntos de venta

**Request** `POST /api/v1/puntos-venta` / `PUT /api/v1/puntos-venta/{idPublico}`

```json
{
  "coPuntoVenta": "PV-0041",
  "descripcion": "Supermercado El Sol",
  "latitud": -34.615000,
  "longitud": -58.400000,
  "direccion": "Av. San Martín 1234, Buenos Aires"
}
```

**Response 200/201**

```json
{
  "idPublico": "b9c8d7e6-f5a4-3b2c-1d0e-9f8a7b6c5d4e",
  "coPuntoVenta": "PV-0041",
  "descripcion": "Supermercado El Sol",
  "latitud": -34.615000,
  "longitud": -58.400000,
  "direccion": "Av. San Martín 1234, Buenos Aires"
}
```

---

### Zonas

Las zonas definen áreas geográficas (territorios de vendedor, zonas de reparto, etc.) usando polígonos GeoJSON.

**Request** `POST /api/v1/zonas` / `PUT /api/v1/zonas/{id}`

```json
{
  "descripcion": "Zona Norte - Vendedor Pérez",
  "color": "#FF5733",
  "coordenadas": {
    "type": "Polygon",
    "coordinates": [
      [
        [-58.390000, -34.590000],
        [-58.370000, -34.590000],
        [-58.370000, -34.610000],
        [-58.390000, -34.610000],
        [-58.390000, -34.590000]
      ]
    ]
  }
}
```

**Response 200/201**

```json
{
  "id": 12,
  "descripcion": "Zona Norte - Vendedor Pérez",
  "color": "#FF5733",
  "coordenadas": {
    "type": "Polygon",
    "coordinates": [
      [
        [-58.390000, -34.590000],
        [-58.370000, -34.590000],
        [-58.370000, -34.610000],
        [-58.390000, -34.610000],
        [-58.390000, -34.590000]
      ]
    ]
  }
}
```

---

## Respuestas estándar

### Error

Todos los errores devuelven el mismo envelope independientemente del código HTTP:

```json
{
  "codigo": "EMPLEADO_NO_ENCONTRADO",
  "mensaje": "No se encontró el empleado con id f7e6d5c4-b3a2-1098-7654-321098fedcba",
  "correlationId": "a1b2c3d4-0000-0000-0000-000000000000",
  "timestamp": "2026-03-17T10:00:00Z"
}
```

| Campo | Descripción |
|---|---|
| `codigo` | Código de error legible por la app, en SCREAMING_SNAKE_CASE |
| `mensaje` | Descripción para mostrar en logs o mensajes de error |
| `correlationId` | UUID de la request, útil para rastrear en logs |
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

La paginación es base 0 (la primera página es `pagina=0`). El tamaño por defecto es 20.

---

## Mapeo VGMDIS → VGM Core Geo

VGMDIS (el sistema legado SQL Server) usa `id_legajo` (entero) como identificador de persona en la tabla `i_rrhh`.

VGM Core Geo usa `co_empleado` (varchar 50) como código funcional. Cuando se importan empleados desde VGMDIS, el mapeo es directo:

```
co_empleado = id_legajo.toString()
```

Esto tiene una consecuencia importante en la ingesta GPS: el Bridge VGMDIS puede enviar posiciones usando `coEmpleado = "1042"` sin necesidad de conocer el UUID interno del empleado en VGM Core Geo. El backend resuelve la correspondencia internamente.

---

## Convenciones generales

### Prefijos de columnas (heredado de VGM Core)

| Prefijo | Significado |
|---|---|
| `id_` | Identificador interno (bigint autoincremental). **Nunca se expone en la API.** |
| `co_` | Código funcional (string legible por humanos o sistemas externos) |
| `de_` | Descripción o texto |
| `nu_` | Número o medida |
| `sn_` | Boolean (sí/no) |
| `fe_` | Fecha |

### IDs en la API

Toda entidad expone un campo `idPublico` de tipo UUID v4 generado por el backend. Los IDs internos (bigint) nunca aparecen en respuestas ni en rutas de la API, excepto en Zonas donde se usa `id` entero por ser una entidad de configuración sin requerimiento de UUID.

### Formato de fechas

Todas las fechas y timestamps en la API siguen ISO 8601 con timezone explícito (preferentemente UTC, sufijo `Z`).

### Versionado

Todos los endpoints están bajo `/api/v1/`. El versionado es por URL.

### Códigos HTTP usados

| Código | Uso |
|---|---|
| 200 | OK — respuesta exitosa de GET, PUT |
| 201 | Created — recurso creado exitosamente (POST) |
| 204 | No Content — operación exitosa sin cuerpo de respuesta (DELETE) |
| 400 | Bad Request — payload inválido o parámetros faltantes |
| 401 | Unauthorized — token JWT ausente o inválido, o API Key inválida |
| 403 | Forbidden — el usuario no tiene acceso al recurso con la empresa/sucursal indicada |
| 404 | Not Found — recurso no encontrado |
| 409 | Conflict — conflicto de datos (ej: código funcional duplicado) |
| 500 | Internal Server Error — error inesperado del servidor |
