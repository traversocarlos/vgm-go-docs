# ADR-002 — OpenStreetMap en lugar de Google Maps

**Estado:** Aceptado
**Fecha:** 2026-03-10
**Decisores:** Equipo VGM

---

## Contexto

Ultra GEO usa Google Maps, que tiene costos de licencia que aumentaron significativamente. Al rediseñar el sistema como VGM Core Geo, hay que decidir qué proveedor de mapas usar.

## Decisión

Usar **Leaflet.js + OpenStreetMap** como motor de mapas. Sin API key, sin costo de licencia.

## Alternativas evaluadas

| Alternativa | Descartada porque |
|---|---|
| Google Maps | Costo de licencia, fue el problema original de Ultra GEO |
| Mapbox | Freemium — gratuito hasta cierto volumen, después costo |
| HERE Maps | Similar a Mapbox, freemium |

## Consecuencias

**Positivas:**
- Elimina el costo que tenía Ultra GEO
- OpenStreetMap tiene cobertura global y calidad suficiente para casos de uso de distribución
- Leaflet es la librería de mapas open source más usada del mundo

**Trade-offs:**
- OpenStreetMap puede tener menos detalle en zonas rurales comparado con Google Maps
- Algunas funcionalidades avanzadas de Google Maps (geocoding preciso, Street View) no están disponibles

**Mitigaciones:**
- Para geocoding se puede usar Nominatim (servicio gratuito de OSM) o integrarlo después si hay necesidad
