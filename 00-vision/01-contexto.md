# Contexto y visión de VGM Core Geo

**Versión:** 1.1
**Fecha:** 2026-03-14
**Estado:** Activo

---

## 1. El problema que resuelve

VGM Sistemas opera **Ultra GEO**, el sistema actual de geolocalización. Tiene problemas críticos:

- Crashes diarios en producción
- Construido sobre ASP.NET WebForms / NHibernate / IIS — tecnología obsoleta
- Costo de licencias de Google Maps
- Código de trayectoria comentado (funcionalidad perdida)
- Sin posibilidad de evolución técnica

**VGM Core Geo es el reemplazo completo de Ultra GEO.**

---

## 2. Qué hace VGM Core Geo

Sistema de geolocalización en tiempo real para empresas de distribución y ventas. Permite:

- Ver en un mapa la posición en tiempo real de vendedores y repartidores
- Consultar el historial de trayectorias
- Gestionar puntos de venta con sus coordenadas
- Definir zonas geográficas de cobertura
- Recibir posiciones desde múltiples fuentes (app móvil, GPS tracker, sistemas externos)

---

## 3. Modelo de negocio

VGM Core Geo es un **producto autónomo**. Se puede vender:

- A clientes que ya usan VGM Core ERP
- A clientes que no usan ningún producto VGM

No requiere VGM Core para funcionar. Tiene su propia base de datos y su propio sistema de usuarios.

---

## 4. Relación con otros productos VGM

| Producto | Relación |
|---|---|
| **VGM Core** | Producto hermano. Mismo stack, mismas convenciones. Pueden integrarse pero son independientes. |
| **VGM GEMA** | App móvil Android que envía posiciones a VGM Core Geo. |
| **Ultra GEO** | Sistema legacy que VGM Core Geo reemplaza. |

La integración entre VGM Core Geo y VGM Core (cuando un cliente usa los dos) se hace por procesos separados — nunca comparten base de datos.

---

## 5. Decisión arquitectónica fundacional

En reunión del equipo (Mauricio, Gustavo, Lucas) se decidió:

> Cada producto VGM es completamente autónomo. Datos propios, release independiente, sin módulos compartidos. La integración se hace por procesos separados.

Esta decisión aplica a VGM Core Geo, VGM Core y VGM GEMA.
