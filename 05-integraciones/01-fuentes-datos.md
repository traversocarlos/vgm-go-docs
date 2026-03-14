# Integraciones y fuentes de datos

**Versión:** 1.1
**Fecha:** 2026-03-14
**Estado:** Activo

---

## Principio

VGM Core Geo recibe posiciones GPS desde múltiples fuentes a través de **un único endpoint REST**. No importa de dónde vengan los datos — todos entran por el mismo lugar.

```
POST /api/v1/posiciones
```

VGM Core Geo no sabe ni le importa quién mandó los datos. El origen se identifica por la API Key del request.

---

## Fuentes soportadas

| Fuente | Protocolo | Autenticación |
|---|---|---|
| App móvil Android (GEMA) | REST API | API Key |
| Bridge VGMDIS (SQL Server) | REST API | API Key |
| GPS Tracker hardware | Webhook | API Key |
| Sistema externo (Traccar, Wialon, etc.) | REST API | API Key |
| Importación de archivo | CSV / Excel / GPX | Sesión de usuario |
| Carga manual | Formulario web | Sesión de usuario |

---

## Payload del endpoint

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

---

## Bridge VGMDIS (flujo actual)

El flujo actual Android GEMA → Tomcat → SQL Server VGMDIS **no se toca**.

Un proceso bridge independiente hace:

```
1. Lee SQL Server via vista v_posiciones_nuevas
2. Llama a POST /api/v1/posiciones de VGM Core Geo
3. Marca las posiciones como procesadas en VGMDIS
```

**Punto clave:** el bridge llama al endpoint REST — nunca escribe directo en la base de datos de VGM Core Geo. Esto protege a VGM Core Geo del futuro: cuando VGMDIS migre a VGM Core, solo cambia el bridge, VGM Core Geo no se toca.

---

## Integración futura con VGM Core

Cuando un cliente usa VGM Core + VGM Core Geo, los datos de empleados y puntos de venta se pueden sincronizar. Modelos posibles ordenados por complejidad:

| Modelo | Cómo funciona | Cuándo usarlo |
|---|---|---|
| Sin integración | Cada sistema carga sus datos por separado | Clientes que usan solo uno de los dos |
| Exportación/importación | CSV manual o programado | Cambios ocasionales, volumen bajo |
| ETL programado | Script que sincroniza periódicamente | Cambios frecuentes, delay tolerable |
| API de integración | VGM Core expone endpoints que VGM Core Geo consulta | Sincronización casi en tiempo real |

**Recomendación para arrancar:** exportación/importación. Los empleados no cambian todos los días. Se evoluciona cuando haya un cliente real que lo necesite.

> Los campos `id_publico_core` en las tablas `empleados` y `puntos_venta` son el puente técnico para cualquiera de estos modelos.
