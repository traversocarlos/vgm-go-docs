# Roadmap de implementación

**Versión:** 1.1
**Fecha:** 2026-03-14
**Estado:** Activo

---

## Estado actual

El proyecto está en fase de **diseño y definición**. La arquitectura está decidida. Todavía no se escribió código de VGM Core Geo.

---

## Pendiente antes de escribir código

- [ ] **Contrato OpenAPI** — definir todos los endpoints antes de escribir código de negocio

---

## Etapa 1 — Reemplazo funcional de Ultra GEO

Objetivo: sistema funcionando que reemplaza lo que hace Ultra GEO hoy.

### Paso 1 — Contrato OpenAPI
Definir todos los endpoints antes de escribir código. Esto permite que backend, frontend y Android avancen en paralelo.

Endpoints mínimos:
- `POST /api/v1/auth/login`
- `GET /api/v1/sesion/yo`
- `POST /api/v1/posiciones`
- `GET /api/v1/posiciones` (con filtros de fecha y empleado)
- `GET /api/v1/empleados`
- `GET /api/v1/puntos-venta`
- `GET /api/v1/zonas`

### Paso 2 — Repositorio backend
Crear `vgm-core-geo-backend` copiando la estructura de Mauricio:
- Módulos: `seguridad`, `geo`, `administracion`
- Configuración de CI (GitHub Actions)
- Docker Compose con PostgreSQL

### Paso 3 — Migraciones Flyway
Crear las migraciones iniciales con el modelo de datos definido.
Incluir desde el inicio: `id_publico_core UUID NULL` en `empleados` y `puntos_venta`.

### Paso 4 — Módulo de seguridad
Copiar y adaptar el código de Mauricio:
- TenantContextFilter (con resolución de sucursal), TenantConnectionPreparer, TenantExceptions
- Login con JWT propio
- Endpoint `/api/v1/sesion/yo`

### Paso 5 — Módulo de posiciones
- Endpoint `POST /api/v1/posiciones` con autenticación por API Key
- Almacenamiento en tabla `posiciones`
- Consulta con filtros

### Paso 6 — Módulo de administración
- CRUD de empleados, puntos de venta, zonas
- Gestión de usuarios y empresas

### Paso 7 — Frontend (en paralelo desde Paso 1)
- Mapa en tiempo real con Leaflet + OpenStreetMap
- Vista de trayectorias
- Editor de puntos de venta y zonas
- Panel de administración

---

## Etapa 2 — Funcionalidades avanzadas

- Geofencing con alertas
- Notificaciones de inactividad
- Dashboard con KPIs
- Exportación Excel / CSV
- Conexión con Auth0 (`vgm-core-dev.us.auth0.com`)
- Integración con VGM Core (sincronización de empleados y puntos de venta)

---

## Repositorios

| Repositorio | Contenido |
|---|---|
| `vgm-go-docs` | Esta documentación |
| `vgm-core-geo-backend` | Backend Kotlin / Spring Boot |
| `vgm-core-geo-web` | Frontend React / TypeScript |
