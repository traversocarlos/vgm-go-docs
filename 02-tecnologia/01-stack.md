# Stack tecnológico

**Versión:** 1.1
**Fecha:** 2026-03-14
**Estado:** Activo

---

## Backend

| Componente | Tecnología | Versión |
|---|---|---|
| Lenguaje | Kotlin | 2.1.10 |
| Framework | Spring Boot | 3.5.3 |
| JVM | Java | 21 |
| Persistencia | JPA / Hibernate | — |
| Migraciones BD | Flyway | — |
| Seguridad | Spring Security + OAuth2 Resource Server | — |
| Caché | Caffeine | 3.1.8 |
| Documentación API | springdoc-openapi | 2.8.4 |
| Build | Gradle + libs.versions.toml | — |
| Tests | Testcontainers | 1.20.4 |
| CI | GitHub Actions | — |
| Containers | Docker + Docker Compose | — |

**Base de datos:** PostgreSQL 17 (propia, no compartida con ningún otro producto)

---

## Frontend

| Componente | Tecnología | Versión |
|---|---|---|
| Framework | React | 18 |
| Lenguaje | TypeScript | 5 |
| Build tool | Vite | — |
| Estilos | Tailwind CSS | — |
| Componentes UI | Shadcn/ui | — |
| Mapas | Leaflet.js + OpenStreetMap | 1.9+ |

**OpenStreetMap reemplaza Google Maps** — sin API key, sin costo de licencia.

---

## Justificación del stack

El stack es intencionalmente idéntico al de VGM Core (desarrollado por Mauricio). Esto permite:

- Reutilizar convenciones, configuración de CI, Docker Compose
- El equipo no tiene que aprender tecnologías nuevas
- Componentes de seguridad (TenantContextFilter, GlobalExceptionHandler, TenantConnectionPreparer) se copian y adaptan directamente

---

## Componentes copiados de VGM Core

| Componente | Cambio para VGM Core Geo |
|---|---|
| `TenantContextFilter` | Agregar resolución de `id_sucursal` desde header `X-Sucursal-Id` |
| `TenantConnectionPreparer` | Copiar sin cambios (agregar `set_config` para `id_sucursal`) |
| `TenantExceptions` | Copiar sin cambios, agregar excepciones de sucursal |
| `GlobalExceptionHandler` | Copiar sin cambios |
| `TenantResolver` | Cambiar namespace a `https://vgmcoregeo.com` y claim a `tenant_id` |
| `SecurityConfig` | Etapa 1: emitir JWT propio. Etapa 2: Resource Server igual que VGM Core |

---

## Lo que NO se usa

| Tecnología | Decisión |
|---|---|
| Google Maps | Reemplazado por OpenStreetMap (costo) |
| Microservicios | Monolito modular, igual que VGM Core |
| RLS (Row Level Security) | A implementar en Etapa 2 — igual al patrón de Mauricio con `set_config` en PostgreSQL |
