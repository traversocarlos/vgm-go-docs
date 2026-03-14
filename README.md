# VGM Core Geo — Documentación técnica

Sistema de geolocalización en tiempo real. Producto autónomo de VGM Sistemas.

**Repositorios relacionados**
- `vgm-core-geo-backend` — Backend Kotlin / Spring Boot
- `vgm-core-geo-web` — Frontend React / TypeScript
- `vgm-core-docs` — Documentación de VGM Core ERP
- `vgm-go-docs` — https://github.com/traversocarlos/vgm-go-docs

---

## Estructura

| Carpeta | Contenido |
|---|---|
| `00-vision/` | Qué es VGM Core Geo, objetivos, contexto |
| `01-arquitectura/` | Decisiones de arquitectura, diagramas |
| `02-tecnologia/` | Stack tecnológico, versiones, justificaciones |
| `03-datos/` | Modelo de datos, migraciones, convenciones |
| `04-seguridad/` | Autenticación, tokens, roles |
| `05-integraciones/` | Fuentes de datos, ETL, bridge VGMDIS |
| `06-adr/` | Registros de decisiones arquitectónicas |
| `07-roadmap/` | Fases de desarrollo, próximos pasos |

---

## Estado del proyecto

**Fase actual:** Diseño y definición — pendiente inicio de implementación

**Pendiente antes de escribir código:**
- [ ] Definir contrato OpenAPI (endpoints de posiciones, empleados, puntos de venta, zonas)

---

## Principio rector

VGM Core Geo es un producto **completamente autónomo**: base de datos propia, ciclo de release independiente, sin dependencia de VGM Core para operar. La integración con otros productos VGM se hace por procesos separados (API, ETL).
