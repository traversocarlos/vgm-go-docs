# ADR-001 — Productos autónomos sin dependencias entre sí

**Estado:** Aceptado
**Fecha:** 2026-03-10
**Decisores:** Mauricio, Gustavo, Lucas

---

## Contexto

VGM Sistemas desarrolla tres productos en paralelo: VGM Core (ERP), VGM Core Geo (geolocalización) y VGM GEMA (preventa móvil). La pregunta era si estos productos debían compartir módulos, base de datos o autenticación.

## Decisión

Cada producto VGM es **completamente autónomo**: base de datos propia, ciclo de release independiente, sin módulos compartidos. La integración entre productos se hace por procesos separados (API, ETL).

## Alternativas evaluadas

| Alternativa | Descartada porque |
|---|---|
| VGM Core como plataforma central con auth compartida | Crea dependencia: VGM Core Geo no puede funcionar ni venderse sin VGM Core |
| Módulos dentro del monolito de VGM Core | Acopla los ciclos de release, cualquier bug en Core afecta a Geo |
| Repositorios separados que comparten base de datos | La base de datos compartida es el acoplamiento más difícil de romper después |

## Consecuencias

**Positivas:**
- VGM Core Geo se puede vender a clientes que no usan VGM Core
- Cada producto puede evolucionar a su propio ritmo
- Un bug en VGM Core no afecta a VGM Core Geo en producción

**Trade-offs:**
- Los datos de empleados y puntos de venta pueden existir en los dos sistemas si un cliente usa los dos
- Requiere un proceso de sincronización cuando hay integración

**Mitigaciones:**
- Campos `id_publico_core` en tablas clave de VGM Core Geo para facilitar la integración futura
- Modelos de sincronización definidos (ver `05-integraciones/01-fuentes-datos.md`)
