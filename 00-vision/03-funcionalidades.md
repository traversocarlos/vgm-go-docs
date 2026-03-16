# Especificación funcional — VGM Core Geo

**Versión:** 1.0
**Fecha:** 2026-03-16
**Estado:** Activo

> Este documento describe **qué hace** el sistema y **cómo se comporta** desde el punto de vista del usuario.
> No describe implementación técnica — eso está en los documentos de arquitectura.

---

## Módulos del sistema

```
VGM Core Geo
├── 1. Seguimiento en tiempo real
├── 2. Historial y trayectorias
├── 3. Visitas detectadas
├── 4. Alertas y eventos
├── 5. Zonas geográficas
├── 6. Cobertura y KPIs
├── 7. Administración
└── 8. Ingesta de posiciones (API)
```

---

## Módulo 1 — Seguimiento en tiempo real

### Qué hace
El supervisor abre VGM Core Geo y ve un mapa con la posición actual de todos los empleados de su sucursal. Los puntos se actualizan automáticamente sin recargar la página.

### Funcionalidades

**Mapa principal**
- Cada empleado aparece como un marcador con su nombre o código
- El color del marcador indica el estado actual (ver Estados)
- Al hacer click en un empleado se abre un panel con: nombre, estado, última posición recibida, hace cuánto fue la última señal, velocidad
- El mapa centra automáticamente la vista en la sucursal activa

**Panel lateral de empleados**
- Lista de todos los empleados de la sucursal con su estado actual
- Hace cuánto se recibió la última posición
- Cuántos puntos de venta visitaron en el día
- Al hacer click en uno, el mapa va a su posición

**Estados del empleado**

| Estado | Color | Condición |
|---|---|---|
| `EN_RUTA` | Verde | En movimiento (velocidad > 3 km/h) |
| `VISITANDO` | Azul | Dentro del radio de un punto de venta |
| `DETENIDO` | Naranja | Sin movimiento fuera de un punto de venta, < 30 min |
| `INACTIVO` | Gris | Sin movimiento fuera de un punto de venta, > 30 min |
| `SIN_SEÑAL` | Rojo | Sin posiciones recibidas hace más de 15 min |

Los umbrales de tiempo son configurables por sucursal.

**Actualización automática**
- Las posiciones se refrescan cada N segundos (configurable, por defecto 30s)
- Indicador visual cuando hay nuevas posiciones
- Sin necesidad de recargar la página (polling o WebSocket según implementación)

---

## Módulo 2 — Historial y trayectorias

### Qué hace
El supervisor puede ver el recorrido que hizo un empleado en un día determinado: dónde estuvo, qué ruta tomó, cuánto tiempo pasó en cada lugar.

### Funcionalidades

**Selector de fecha y empleado**
- Elegir empleado + fecha
- El mapa muestra la trayectoria del día como una línea

**Replay animado**
- Botón de play que anima el recorrido en el mapa
- Control de velocidad (1x, 2x, 5x)
- Barra de progreso temporal — se puede avanzar o retroceder

**Timeline del día**
Vista de lista (no mapa) que muestra el día como una secuencia de eventos:

```
08:45  Salió de Sucursal Centro
09:12  Llegó a Supermercado Don Mario    → estuvo 22 min
09:34  En ruta
10:05  Llegó a Almacén López             → estuvo 8 min
10:15  Detenido en Av. Belgrano 1234     → 38 min (fuera de punto de venta)
10:53  En ruta
11:20  Llegó a Despensa El Trébol        → estuvo 15 min
...
18:30  Sin señal
```

**Datos de la trayectoria**
- Distancia total recorrida en el día
- Tiempo en movimiento vs tiempo detenido
- Cantidad de puntos de venta visitados
- Primera y última posición del día

---

## Módulo 3 — Visitas detectadas

### Qué hace
VGM Core Geo detecta automáticamente cuando un empleado visita un punto de venta, sin que el empleado haga nada. El sistema compara la posición GPS con las coordenadas del punto de venta y registra la visita si se cumplen las condiciones configuradas.

Este módulo responde la pregunta más frecuente del supervisor:
> *"¿Realmente fue al cliente, o marcó que fue pero no fue?"*

### Cómo funciona la detección

```
Cada vez que llega una posición GPS:
  1. Buscar puntos de venta de la sucursal del empleado
  2. Para cada punto, calcular distancia entre posición y coordenadas del PV
  3. Si distancia < radio_visita (default: 100 metros):
     - Si no hay visita abierta → abrir visita (registrar hora_entrada)
     - Si hay visita abierta → actualizar hora_ultima_posicion
  4. Si distancia >= radio_visita y hay visita abierta:
     - Si (hora_ultima_posicion - hora_entrada) >= duracion_minima (default: 3 min):
       → Cerrar visita como CONFIRMADA
     - Si menor a duracion_minima:
       → Cerrar visita como DESCARTADA (pasó de largo)
```

### Tabla `visitas` — qué se registra

| Campo | Descripción |
|---|---|
| `id_empleado` | Quién realizó la visita |
| `id_punto_venta` | Qué punto de venta visitó |
| `fe_entrada` | Cuándo entró al radio |
| `fe_salida` | Cuándo salió del radio |
| `nu_duracion_min` | Minutos que estuvo |
| `nu_radio_metros` | Radio configurado al momento de la visita |
| `co_estado` | `CONFIRMADA`, `DESCARTADA`, `EN_CURSO` |

### Vista de visitas del día

El supervisor puede ver para cada empleado:
- Lista de puntos de venta visitados con hora de entrada, salida y duración
- Puntos de venta **no visitados** en el día
- Visitas muy cortas (sospechosas) resaltadas

### Configuración por sucursal

| Parámetro | Default | Descripción |
|---|---|---|
| `radio_visita_metros` | 100 | Radio para considerar que está en el punto |
| `duracion_minima_minutos` | 3 | Mínimo para confirmar visita |
| `tiempo_inactividad_minutos` | 30 | Para estado INACTIVO |
| `tiempo_sin_senal_minutos` | 15 | Para estado SIN_SEÑAL |

---

## Módulo 4 — Alertas y eventos

### Qué hace
El sistema detecta situaciones que requieren atención y las registra. En Etapa 1 se registran y muestran en la interfaz. En Etapa 2 se envían notificaciones.

### Tipos de alerta

| Alerta | Condición | Prioridad |
|---|---|---|
| `INACTIVIDAD` | Empleado detenido fuera de PV por más de N min | Media |
| `SIN_SEÑAL` | Sin posiciones recibidas hace más de N min | Alta |
| `FUERA_DE_ZONA` | Empleado fuera de su zona asignada | Media |
| `VISITA_CORTA` | Visita en PV menor a duración mínima | Baja |
| `INICIO_TARDIO` | No se recibió primera posición antes de hora configurada | Media |

### Tabla `eventos` — historial de alertas

Cada alerta genera un evento que queda registrado con: empleado, tipo, fecha, dato adicional (ej: nombre de la zona de la que salió).

### Visualización
- En el panel lateral, los empleados con alertas activas aparecen resaltados
- Badge con cantidad de alertas activas
- Log de eventos del día filtrable por tipo y empleado

---

## Módulo 5 — Zonas geográficas

### Qué hace
Permite definir áreas geográficas en el mapa y asignarlas a empleados o grupos. Se usan para asignar territorios de venta y detectar cuando un empleado sale de su zona.

### Funcionalidades

**Editor de zonas**
- Dibujar polígonos directamente en el mapa (click para agregar vértices)
- Editar vértices de una zona existente
- Asignar nombre y color a cada zona
- La zona queda guardada como polígono GeoJSON en la base de datos

**Asignación de zonas**
- Una zona pertenece a una sucursal
- Se puede asignar empleados a una zona
- Un empleado puede estar en varias zonas (territorio compartido)

**Visualización**
- Las zonas se muestran como polígonos semitransparentes en el mapa
- Se pueden activar/desactivar por capa
- Al ver la posición de un empleado, se resalta si está dentro o fuera de su zona

---

## Módulo 6 — Cobertura y KPIs

### Qué hace
Provee métricas sobre el trabajo en campo. No reemplaza el ERP — complementa con información de presencia física.

### Dashboard de cobertura del día

Para cada sucursal, el dashboard muestra:

**Por empleado:**
- Puntos de venta visitados vs total asignados (% cobertura)
- Distancia recorrida
- Tiempo en campo (primera a última posición)
- Tiempo en movimiento vs detenido
- Cantidad de visitas confirmadas

**Por sucursal:**
- % de cobertura general del día
- Mapa de calor de visitas (qué zonas tuvieron más actividad)
- Empleados con más y menos cobertura
- Puntos de venta nunca visitados en los últimos N días

### Historial
- Los mismos KPIs disponibles para cualquier fecha pasada
- Comparación entre días o semanas
- Exportación a CSV / Excel

---

## Módulo 7 — Administración

### Empleados

| Campo | Descripción |
|---|---|
| Código | Identificador interno (ej: `VEN001`) |
| Nombre | Nombre completo |
| Tipo | `VENDEDOR`, `REPARTIDOR`, `SUPERVISOR` |
| Sucursal | A qué sucursal pertenece |
| Registra coordenadas | Si está activo para seguimiento |
| id_publico_core | Puente con VGM Core (opcional) |

**Alta de empleado:** formulario web. En Etapa 2 se puede sincronizar desde VGM Core.

### Puntos de venta

| Campo | Descripción |
|---|---|
| Código | Identificador interno |
| Nombre | Nombre del local |
| Coordenadas | Latitud / longitud (se pone en el mapa) |
| Sucursal | A qué sucursal pertenece |
| id_publico_core | Puente con VGM Core (clientes del ERP) |

**Carga de coordenadas:** click en el mapa o ingreso manual. En Etapa 2 importación masiva desde CSV.

### Usuarios y accesos

- Un usuario puede tener acceso a una o varias sucursales
- Roles: `ADMIN`, `OPERADOR`, `READONLY`
  - `ADMIN`: puede crear/editar empleados, PV, zonas y usuarios
  - `OPERADOR`: puede ver todo, editar PV y zonas
  - `READONLY`: solo visualización

### Empresas y sucursales
Gestión de la jerarquía de tenancy. Solo accesible para `ADMIN` del cliente.

---

## Módulo 8 — Ingesta de posiciones (API)

### Qué hace
Recibe posiciones GPS desde cualquier fuente. Es el único punto de entrada de datos de posición.

### Endpoint

```
POST /api/v1/posiciones
Authorization: ApiKey <clave>
```

```json
{
  "idEmpleado": "uuid-del-empleado",
  "latitud": -27.4516,
  "longitud": -58.9867,
  "precision": 5.2,
  "velocidad": 12.5,
  "tipoOperacion": "VENTA",
  "fechaPosicion": "2026-03-16T09:30:00Z"
}
```

### Fuentes soportadas

| Fuente | Cómo se integra |
|---|---|
| GEMA (app Android) | Llama directamente al endpoint con su API Key |
| Bridge VGMDIS | Lee `v_posiciones_nuevas` en SQL Server, llama al endpoint |
| GPS Tracker hardware | Webhook configurado apuntando al endpoint |
| Sistema externo (Traccar, etc.) | REST API con su API Key |
| Importación archivo | CSV / GPX subido desde la interfaz web |
| Carga manual | Formulario en la web |

### Procesamiento al recibir una posición

```
1. Validar API Key → 401 si inválida
2. Validar que el empleado existe y pertenece al tenant → 404 si no
3. Guardar en tabla posiciones
4. Evaluar detección de visita (Módulo 3)
5. Evaluar generación de alertas (Módulo 4)
6. Responder 201 Created
```

---

## Etapas de implementación

### Etapa 1 — Reemplazo de Ultra GEO + diferenciadores clave

| Módulo | Alcance Etapa 1 |
|---|---|
| Seguimiento tiempo real | Completo: mapa + panel + estados |
| Historial trayectorias | Completo: mapa + replay + timeline |
| Visitas detectadas | Completo: detección automática + vista del día |
| Alertas | Registro en interfaz, sin notificaciones externas |
| Zonas geográficas | Completo: editor + asignación + detección |
| Cobertura y KPIs | Dashboard básico: cobertura del día por empleado |
| Administración | Completo: empleados, PV, zonas, usuarios |
| Ingesta API | Completo: todas las fuentes |

### Etapa 2 — Funcionalidades avanzadas

- Notificaciones (email, push, WhatsApp)
- Dashboard de KPIs avanzado con comparativas
- Exportación Excel / PDF
- Importación masiva de PV desde CSV
- Sincronización con VGM Core (empleados y PV)
- Auth0 (solo cambio de configuración)
- Row Level Security en PostgreSQL

### Etapa 3 — Diferenciadores premium

- Optimización de rutas (integración con OSRM o Valhalla — open source)
- Planificación de visitas (el supervisor define el recorrido del día)
- App móvil propia (si GEMA no es suficiente)
- API pública para integraciones de terceros
