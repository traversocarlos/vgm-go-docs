# CONTEXT — VGM Core Geo

> Archivo de contexto para Claude Code y cualquier IA que trabaje en este proyecto.
> Leer completo antes de hacer cualquier modificación.
> Actualizar cada vez que se tome una decisión importante.

**Última actualización:** 2026-03-14

---

## ¿Qué es VGM Core Geo?

Sistema de geolocalización en tiempo real para empresas de distribución y ventas. Reemplaza **Ultra GEO** — sistema actual con crashes diarios, tecnología obsoleta (ASP.NET WebForms) y costos de licencia de Google Maps.

VGM Core Geo es un **producto autónomo** — base de datos propia, ciclo de release independiente, se puede vender sin VGM Core.

---

## El equipo

**VGM Sistemas** — software de gestión empresarial.
- **Mauricio** — backend, autor de VGM Core (referencia arquitectónica directa)
- **Gustavo** — equipo
- **Lucas** — equipo

---

## Los tres productos VGM — todos autónomos

| Producto | Qué es | Estado |
|---|---|---|
| **VGM Core** | ERP completo (reingeniería del legacy VGMDIS) | Backend funcionando, por Mauricio |
| **VGM Core Geo** | Geolocalización en tiempo real | En diseño — este repositorio |
| **VGM GEMA** | App móvil Android de preventa/distribución | Existe, envía posiciones GPS hoy |

**Regla fundamental:** integración entre productos siempre por **API REST**. Nunca base de datos compartida.

---

## Sistema legacy

**VGMDIS** — ERP actual en PowerBuilder / SQL Server. Sigue operativo.

Flujo GPS actual:
```
Android GEMA → Tomcat → SQL Server VGMDIS
```

Bridge VGM Core Geo: lee `v_posiciones_nuevas` en VGMDIS → llama a `POST /api/v1/posiciones`. Cuando VGMDIS migre a VGM Core, solo cambia el bridge — VGM Core Geo no se toca.

---

## Stack tecnológico

Idéntico a VGM Core (Mauricio es la referencia).

### Backend
- Kotlin **2.1.10** + Spring Boot **3.5.3** + Java **21**
- JPA / Hibernate + Flyway
- Spring Security + OAuth2 Resource Server
- Caffeine Cache **3.1.8**
- springdoc-openapi **2.8.4**
- PostgreSQL 17 (base de datos propia)
- Docker + Docker Compose + GitHub Actions CI
- Tests con Testcontainers **1.20.4**

### Frontend
- React 18 + TypeScript 5 + Vite
- Tailwind CSS + Shadcn/ui
- Leaflet.js + OpenStreetMap (reemplaza Google Maps — sin costo)

### Componentes copiados de VGM Core y adaptados

| Componente | Cambio para VGM Core Geo |
|---|---|
| `TenantContextFilter` | Agregar resolución de `id_sucursal` desde header `X-Sucursal-Id` |
| `TenantConnectionPreparer` | Agregar `set_config('app.id_sucursal', valor)` |
| `TenantExceptions` | Copiar sin cambios, agregar excepciones de sucursal |
| `GlobalExceptionHandler` | Copiar sin cambios |
| `TenantResolver` | Cambiar namespace a `https://vgmcoregeo.com`, claim `tenant_id` |
| `SecurityConfig` | Etapa 1: emitir JWT propio. Etapa 2: Resource Server igual que VGM Core |

---

## Jerarquía de tenancy — idéntica a VGM Core

```
clientes_saas
  └── empresas
        └── sucursales
              ├── empleados
              ├── puntos_venta
              └── zonas
```

Mismos nombres de tablas y columnas que VGM Core.

**Diferencia con VGM Core:** VGM Core resuelve `cliente_saas` + `empresa` en el TenantContext. VGM Core Geo **extiende** esto resolviendo también `sucursal` — porque la unidad operativa es la sucursal.

**Ejemplo real:**
```
D OUNI (cliente_saas)
  ├── OV (empresa)
  │     ├── Sucursal Centro
  │     └── Sucursal Norte
  ├── ZIRKA (empresa)
  └── OV2 (empresa)
```

---

## Modelo de datos — tablas principales

Ver detalle completo en `03-datos/01-modelo.md`

**Convenciones de nombres (igual que VGM Core):**
- `id_` identificador interno
- `co_` código funcional
- `de_` descripción/texto
- `nu_` número/medida
- `sn_` boolean (sí/no)
- `fe_` fecha
- Toda tabla tiene `id_publico UUID` — se expone en APIs, nunca el ID interno

**Modelo de usuarios (patrón de Mauricio):**
- `cuentas` — identidad global (co_sub_oidc para Auth0 en Etapa 2)
- `usuarios_geo` — membresía de cuenta en un tenant
- `usuarios_geo_sucursales` — acceso a sucursales con rol (`ADMIN`, `OPERADOR`, `READONLY`)

**Campos puente para integración futura:**
- `empleados.id_publico_core UUID NULL`
- `puntos_venta.id_publico_core UUID NULL`

---

## Seguridad y autenticación

Ver detalle completo en `04-seguridad/01-autenticacion.md`

### Patrón idéntico a VGM Core

Siguiendo exactamente el patrón de Mauricio:
- El JWT lleva **solo** `tenant_id` — empresa, sucursal y rol se resuelven dinámicamente
- Empresa se resuelve desde header `X-Empresa-Id`
- Sucursal se resuelve desde header `X-Sucursal-Id` (extensión de VGM Core Geo)
- Rol se carga desde `usuarios_geo_sucursales` en BD

### Namespace JWT
```
https://vgmcoregeo.com
```
Diferente al de VGM Core (`https://vgmcore.com`) — productos independientes.

### Token JWT
```json
{
  "sub": "usuario@empresa.com",
  "https://vgmcoregeo.com/tenant_id": 1,
  "exp": 1234567890
}
```

### Auth0 — ya configurado por Mauricio
- Tenant: `vgm-core-dev.us.auth0.com`
- Aplicación: `VGM Core Geo` (Single Page Application)
- Etapa 1: JWT propio. Etapa 2: conectar Auth0 (solo cambia `application.yml`)

---

## Ingesta de datos

Endpoint único: `POST /api/v1/posiciones`

Seis fuentes: app móvil, bridge VGMDIS, GPS tracker hardware, sistema externo, importación archivo, carga manual.
Autenticación por fuente: API Key única por origen.

Ver detalle en `05-integraciones/01-fuentes-datos.md`

---

## ADRs tomados

| ADR | Decisión |
|---|---|
| ADR-001 | Productos autónomos — cada producto VGM tiene su propia BD y ciclo de release |
| ADR-002 | OpenStreetMap + Leaflet en lugar de Google Maps |
| ADR-003 | JWT propio Etapa 1, Auth0 Etapa 2 sin reescribir código |

---

## ✅ Decisiones confirmadas — no reabrir

- Jerarquía `clientes_saas → empresas → sucursales` idéntica a VGM Core
- Mismos nombres de tablas y columnas que VGM Core
- Patrón de seguridad idéntico a VGM Core (tenant_id en JWT, empresa/sucursal por headers, rol desde BD)
- Namespace JWT: `https://vgmcoregeo.com`
- Claim de tenant: `tenant_id` (igual que VGM Core)
- Auth0: `vgm-core-dev.us.auth0.com`, aplicación `VGM Core Geo`
- Integración entre productos siempre por API REST
- OpenStreetMap reemplaza Google Maps

---

## ⚠️ Pendiente — no implementar hasta resolver

- [ ] Definir contrato OpenAPI antes de escribir código de negocio

---

## Próximos pasos en orden

1. Definir contrato OpenAPI (`01-arquitectura/02-openapi.yaml`)
2. Crear repositorio `vgm-core-geo-backend` copiando estructura de VGM Core
3. Migraciones Flyway iniciales (incluir `id_publico_core UUID NULL` desde el inicio)
4. Módulo de seguridad (copiar y adaptar código de Mauricio)
5. Módulo de posiciones (`POST /api/v1/posiciones`)
6. Módulo de administración (empleados, puntos de venta, zonas)
7. Frontend (en paralelo desde el paso 1)

---

## Repositorios

| Repositorio | Contenido |
|---|---|
| `vgm-go-docs` | Esta documentación |
| `vgm-core-geo-backend` | Backend Kotlin / Spring Boot (a crear) |
| `vgm-core-geo-web` | Frontend React / TypeScript (a crear) |
| `vgm-core-docs` | Documentación VGM Core — referencia arquitectónica |

---

## Cómo usar este archivo desde Claude Code

Empezá siempre con:
```
Leé el CONTEXT.md y después [lo que necesitás hacer]
```

Cuando se tome una decisión nueva, actualizar este archivo es parte del proceso.
