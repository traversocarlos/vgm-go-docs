# CONTEXT — VGM Go

> Archivo de contexto para Claude Code y cualquier IA que trabaje en este proyecto.
> Leer completo antes de hacer cualquier modificación.
> Actualizar cada vez que se tome una decisión importante.

**Última actualización:** 2026-03-13

---

## ¿Qué es VGM Go?

Sistema de geolocalización en tiempo real para empresas de distribución y ventas. Reemplaza **Ultra GEO** — sistema actual con crashes diarios, tecnología obsoleta (ASP.NET WebForms) y costos de licencia de Google Maps.

VGM Go es un **producto autónomo** — base de datos propia, ciclo de release independiente, se puede vender sin VGM Core.

---

## El equipo

**VGM Sistemas** — software de gestión empresarial.
- **Mauricio** — backend, autor de VGM Core
- **Gustavo** — equipo
- **Lucas** — equipo

---

## Los tres productos VGM — todos autónomos

| Producto | Qué es | Estado |
|---|---|---|
| **VGM Core** | ERP completo (reingeniería del legacy VGMDIS) | Backend funcionando, por Mauricio |
| **VGM Go** | Geolocalización en tiempo real | En diseño — este repositorio |
| **VGM GEMA** | App móvil Android de preventa/distribución | Existe, envía posiciones GPS hoy |

**Regla fundamental:** integración entre productos siempre por **API REST**. Nunca base de datos compartida.

---

## Sistema legacy

**VGMDIS** — ERP actual en PowerBuilder / SQL Server. Sigue operativo.

Flujo GPS actual:
```
Android GEMA → Tomcat → SQL Server VGMDIS
```

Bridge VGM Go: lee `v_posiciones_nuevas` en VGMDIS → llama a `POST /api/v1/posiciones`. Cuando VGMDIS migre a VGM Core, solo cambia el bridge — VGM Go no se toca.

---

## Stack tecnológico

### Backend
- Kotlin 2.1 + Spring Boot 3.5
- JPA / Hibernate + Flyway
- Spring Security (JWT propio Etapa 1, Auth0 Etapa 2)
- springdoc-openapi 2.x
- PostgreSQL 17 (base de datos propia)
- Docker + Docker Compose + GitHub Actions CI
- Tests con Testcontainers

### Frontend
- React 18 + TypeScript 5 + Vite
- Tailwind CSS + Shadcn/ui
- Leaflet.js + OpenStreetMap (reemplaza Google Maps — sin costo)

### Referencia directa
El backend de VGM Core (Mauricio) es la referencia. Mismo stack, mismas convenciones. Componentes a copiar y adaptar:
- `TenantContextFilter.kt`
- `TenantConnectionPreparer.kt`
- `TenantExceptions.kt`
- `GlobalExceptionHandler.kt`

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

**Campos puente para integración futura:**
- `empleados.id_publico_core UUID NULL`
- `puntos_venta.id_publico_core UUID NULL`

---

## Seguridad y autenticación

Ver detalle completo en `04-seguridad/01-autenticacion.md`

### Namespace JWT definido
```
https://vgmcoregeo.com
```
Diferente al de VGM Core (`https://vgmcore.com`) — productos independientes.

### Claims del token
```json
{
  "https://vgmcoregeo.com/cliente_saas_id": 1,
  "https://vgmcoregeo.com/empresa_id": 5,
  "https://vgmcoregeo.com/sucursal_id": 3,
  "https://vgmcoregeo.com/rol": "ADMIN"
}
```

### Auth0 — ya configurado por Mauricio
- Tenant: `vgm-core-dev.us.auth0.com`
- Aplicación: `VGM Core Geo` (Single Page Application)
- Etapa 1: JWT propio. Etapa 2: conectar Auth0.

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
- Namespace JWT: `https://vgmcoregeo.com`
- Integración entre productos siempre por API REST
- OpenStreetMap reemplaza Google Maps

---

## ⚠️ Pendiente — no implementar hasta resolver

- [ ] Definir contrato OpenAPI antes de escribir código de negocio
- [ ] Coordinar con Mauricio el Auth0 Action para Etapa 2

---

## Próximos pasos en orden

1. Definir contrato OpenAPI (`01-arquitectura/02-openapi.yaml`)
2. Crear repositorio `vgm-go-backend` copiando estructura de VGM Core
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
| `vgm-go-backend` | Backend Kotlin / Spring Boot (a crear) |
| `vgm-go-web` | Frontend React / TypeScript (a crear) |
| `vgm-core-docs` | Documentación VGM Core — referencia arquitectónica |

---

## Cómo usar este archivo desde Claude Code

Empezá siempre con:
```
Leé el CONTEXT.md y después [lo que necesitás hacer]
```

Cuando se tome una decisión nueva, actualizar este archivo es parte del proceso.
