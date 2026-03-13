# CONTEXT — VGM Go

> Documento de contexto para Claude Code y cualquier IA que trabaje en este proyecto.
> Leer completo antes de cualquier modificación. Refleja el estado actual del proyecto.
> Actualizar cada vez que se tome una decisión importante.
>
> **Última actualización:** 2026-03-13

---

## Qué es VGM Go

Sistema de geolocalización en tiempo real para empresas de distribución y ventas.
Reemplaza **Ultra GEO** — el sistema actual, que tiene crashes diarios, tecnología obsoleta (ASP.NET WebForms) y costos de licencia de Google Maps.

VGM Go es un **producto autónomo**: base de datos propia, usuarios propios, ciclo de release independiente. Se puede vender a clientes que no usan ningún otro producto VGM.

---

## La empresa y el equipo

**VGM Sistemas** — software de gestión empresarial.

- **Mauricio** — backend, autor de VGM Core (referencia arquitectónica directa)
- **Gustavo** — parte del equipo
- **Lucas** — parte del equipo

---

## Los tres productos VGM (todos autónomos, sin base de datos compartida)

| Producto | Qué es | Estado |
|---|---|---|
| **VGM Core** | ERP completo (reingeniería del legacy VGMDIS) | Backend funcionando, desarrollado por Mauricio |
| **VGM Go** | Geolocalización en tiempo real | En diseño — este repositorio |
| **VGM GEMA** | App móvil Android de preventa/distribución | Existe, envía posiciones GPS a VGMDIS hoy |

**ADR-001:** Cada producto tiene su propia base de datos y ciclo de release. La integración se hace por API o ETL, nunca por base de datos compartida.

---

## Sistema legacy relevante

**VGMDIS** — ERP actual en PowerBuilder / SQL Server. Sigue operativo durante la transición.

Flujo GPS actual:
```
Android GEMA → Tomcat → SQL Server VGMDIS
```

VGM Go recibirá esas posiciones a través de un **bridge** que lee la vista `v_posiciones_nuevas` de VGMDIS y llama al endpoint REST de VGM Go. Cuando VGMDIS migre a VGM Core, solo cambia el bridge — VGM Go no se toca.

---

## Stack tecnológico

### Backend
- **Kotlin 2.1 + Spring Boot 3.5** — mismo stack que VGM Core
- JPA / Hibernate + Flyway
- Spring Security (JWT propio en Etapa 1, Auth0 en Etapa 2)
- springdoc-openapi 2.x
- PostgreSQL 17 (base de datos propia)
- Docker + Docker Compose / GitHub Actions CI / Testcontainers

### Frontend
- React 18 + TypeScript 5 + Vite
- Tailwind CSS + Shadcn/ui
- **Leaflet.js + OpenStreetMap** — reemplaza Google Maps, sin costo. **ADR-002.**

### Por qué el mismo stack que VGM Core

El equipo no aprende tecnologías nuevas. Los componentes de seguridad de Mauricio se copian y adaptan directamente:

| Componente | Cambio al adaptar |
|---|---|
| `TenantContextFilter.kt` | Eliminar lógica de `id_cliente_saas` |
| `TenantConnectionPreparer.kt` | Copiar sin cambios |
| `TenantExceptions.kt` | Copiar sin cambios |
| `GlobalExceptionHandler.kt` | Copiar sin cambios |
| `TenantResolver` | Cambiar claim de `https://vgmcore.com/tenant_id` al namespace acordado |
| `SecurityConfig` | Etapa 1: generar JWT. Etapa 2: Resource Server |

**Diferencia clave con VGM Core:** VGM Go tiene **1 nivel de tenancy** (empresas). VGM Core tiene 3 (clientes_saas → empresas → sucursales). Eliminar toda la lógica de `id_cliente_saas` al copiar.

---

## Modelo de datos

PostgreSQL propio. Multi-tenancy por campo `id_empresa` en cada tabla. Sin RLS en Etapa 1.

### Convenciones de nombres (igual que VGM Core)

| Prefijo | Significado | Ejemplo |
|---|---|---|
| `id_` | Identificador interno | `id_empleado` |
| `co_` | Código funcional | `co_tipo` |
| `de_` | Descripción o texto | `de_nombre` |
| `nu_` | Número o medida | `nu_latitud` |
| `sn_` | Boolean (sí/no) | `sn_activo` |
| `fe_` | Fecha | `fe_alta` |

### Tablas principales

**`empresas`** — cada empresa cliente de VGM Go

**`empleados`** — vendedores y repartidores
- `id_publico_core UUID NULL` — puente para integración futura con VGM Core. NULL por ahora.

**`posiciones`** — coordenadas GPS recibidas
- Campos relevantes: `nu_latitud`, `nu_longitud`, `nu_precision`, `nu_velocidad`, `co_tipo_operacion`, `fe_posicion`, `fe_recibida`

**`puntos_venta`** — comercios con coordenadas
- `id_publico_core UUID NULL` — puente con `entidades_direcciones` de VGM Core (Fase 2+)

**`zonas`** — polígonos geográficos (coordenadas en JSONB, campo `de_color` hex para el mapa)

**`usuarios_go`** + **`usuarios_go_empresas`** — usuarios del sistema, acceso por empresa
- Roles: `ADMIN`, `OPERADOR`, `READONLY`
- `co_sub_oidc` — preparado para Auth0 en Etapa 2

El DDL completo está en [03-datos/01-modelo.md](03-datos/01-modelo.md).

---

## Seguridad y autenticación

**ADR-003:** JWT propio en Etapa 1, diseñado para conectar Auth0 en Etapa 2 sin reescribir código.

### Etapa 1 — JWT propio

- Login: `POST /api/v1/auth/login`
- VGM Go valida contra su propia tabla `usuarios_go` y emite JWT firmado con `vgmgo.jwt.secret`
- Frontend guarda token **en memoria** (nunca en localStorage)
- Cada request envía: `Authorization: Bearer <token>` + `X-Empresa-Id: <id>`

Claims del token:
```json
{
  "sub": "usuario@empresa.com",
  "https://vgmgo.com/empresa_id": 5,
  "https://vgmgo.com/rol": "ADMIN",
  "exp": 1234567890
}
```

> ⚠️ **BLOQUEANTE:** El namespace `https://vgmgo.com/` debe acordarse con Mauricio antes de escribir código de autenticación. Si Auth0 emite tokens para VGM Core + VGM Go juntos, el namespace tiene que ser compatible desde el día uno. Pregunta: ¿`https://vgm.com/` compartido o `https://vgmgo.com/` separado?

### Etapa 2 — Auth0

Backend pasa a ser Resource Server. Solo cambian 4 líneas en `application.yml`. Todo lo demás igual.

---

## Ingesta de posiciones

Endpoint único: `POST /api/v1/posiciones`. Autenticación por API Key (una por fuente de origen).

| Fuente | Protocolo |
|---|---|
| App móvil Android (GEMA) | REST API |
| Bridge VGMDIS | Lee `v_posiciones_nuevas` de SQL Server → llama al endpoint REST |
| GPS Tracker hardware | Webhook |
| Sistema externo (Traccar, Wialon, etc.) | REST API |
| Importación de archivo | CSV / Excel / GPX |
| Carga manual | Formulario web |

Payload mínimo:
```json
{
  "idEmpleado": "uuid-del-empleado",
  "latitud": -27.4516,
  "longitud": -58.9867,
  "precision": 5.2,
  "velocidad": 0.0,
  "tipoOperacion": "VENTA",
  "fechaPosicion": "2026-03-13T09:30:00Z"
}
```

El bridge nunca escribe directo en la base de datos de VGM Go — siempre pasa por el endpoint REST.

---

## Decisiones tomadas (ADRs)

| ADR | Decisión | Detalle |
|---|---|---|
| ADR-001 | Productos autónomos | Base de datos y release separados. Integración solo por API/ETL |
| ADR-002 | OpenStreetMap + Leaflet | Reemplaza Google Maps — sin API key, sin costo de licencia |
| ADR-003 | JWT propio → Auth0 | Arranca independiente, migra a Auth0 con solo cambiar configuración |

---

## Pendiente antes de escribir código

- [ ] **URGENTE — Reunión con Mauricio:** acordar namespace de JWT claims
- [ ] **Contrato OpenAPI:** definir todos los endpoints (`01-arquitectura/02-openapi.yaml`)

---

## Próximos pasos en orden

1. Reunión con Mauricio → acordar namespace JWT
2. Contrato OpenAPI (`01-arquitectura/02-openapi.yaml`)
3. Crear repositorio `vgm-go-backend` copiando estructura de VGM Core
4. Migraciones Flyway (incluir `id_publico_core UUID NULL` desde el inicio)
5. Módulo de seguridad (copiar y adaptar código de Mauricio)
6. Módulo de posiciones (`POST /api/v1/posiciones`)
7. Módulo de administración (empleados, puntos de venta, zonas)
8. Frontend (puede arrancar en paralelo desde el paso 2)

### Endpoints mínimos Etapa 1

- `POST /api/v1/auth/login`
- `GET /api/v1/sesion/yo`
- `POST /api/v1/posiciones`
- `GET /api/v1/posiciones` (filtros por fecha y empleado)
- `GET /api/v1/empleados`
- `GET /api/v1/puntos-venta`
- `GET /api/v1/zonas`

---

## Repositorios

| Repositorio | Contenido |
|---|---|
| `vgm-go-docs` | Esta documentación (estás acá) |
| `vgm-go-backend` | Backend Kotlin / Spring Boot (a crear) |
| `vgm-go-web` | Frontend React / TypeScript (a crear) |
| `vgm-core-docs` | Documentación de VGM Core — referencia arquitectónica |

---

## Estructura de este repositorio

| Carpeta | Contenido |
|---|---|
| `00-vision/` | Qué es VGM Go, objetivos, contexto de negocio |
| `01-arquitectura/` | Decisiones de arquitectura, OpenAPI (pendiente) |
| `02-tecnologia/` | Stack tecnológico, versiones, justificaciones |
| `03-datos/` | Modelo de datos con DDL completo |
| `04-seguridad/` | Autenticación, tokens, roles, flujos |
| `05-integraciones/` | Fuentes de datos, bridge VGMDIS, integración con VGM Core |
| `06-adr/` | Registros de decisiones arquitectónicas |
| `07-roadmap/` | Fases de implementación y próximos pasos |

---

## Cómo usar este archivo

Cuando uses Claude Code en VS Code, empezá siempre con:

```
Leé el archivo CONTEXT.md y después [lo que necesitás hacer]
```
