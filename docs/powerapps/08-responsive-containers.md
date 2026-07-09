# Responsive — containers de auto-layout (pantallas grandes y chicas)

> **Estado: en construcción.** Este documento se va llenando a medida que se
> verifica cada paso en Studio real. Hasta ahora se probó `scrContactoProyecto`
> parcialmente (ver checklist abajo). El resto de las 7 pantallas se hace
> replicando el mismo patrón.

## Por qué containers de auto-layout (y no fórmulas de breakpoint)

Dos formas de hacer una app Canvas responsive:

1. **Fórmulas de breakpoint manuales** — cada `X`/`Y`/`Width` existente se
   reescribe como `If(App.Width < 700; valorMovil; valorEscritorio)`. Control
   total, pero duplica literalmente cada una de las ~150+ propiedades de
   posición ya documentadas en las 8 pantallas.
2. **Containers de auto-layout (elegido)** — se envuelven los controles en
   containers verticales/horizontales con `LayoutWrap`. Power Apps
   reacomoda y **apila automáticamente** cuando no entran uno al lado del
   otro — sin escribir una sola fórmula de breakpoint. Pantalla grande =
   lado a lado; pantalla chica = apilado, gratis, con el mismo container.

## ⚠️ Gotcha importante: no se puede pegar YAML de un container suelto en Studio

Se intentó pegar el YAML de un `GroupContainer` directo en el árbol/lienzo de
Studio (Ctrl+V) y dio dos errores distintos:

1. `YamlInvalidSyntax; Reason: Named object value cannot be null` — de
   indentación: al copiar solo una sección desde un bloque más grande, el
   editor/portapapeles a veces "dedenta" el texto y las propiedades quedan a
   la misma columna que el nombre del control en vez de más adentro (YAML las
   interpreta como hermanas, no como hijas → el control queda con valor
   `null`).
2. `Reason: Property '...' not found on type '...PaModule'` — este es más de
   fondo: **Studio no soporta pegar un nodo de control suelto vía Ctrl+V en
   el lienzo**. El parser espera un documento `.pa.yaml` completo (con
   `Screens:`, `ComponentDefinitions:`, etc.), no un fragmento aislado.

**Conclusión: los containers (igual que el resto de la app) se construyen
insertándolos manualmente desde el panel Insertar y seteando cada propiedad
una por una** — el mismo método "clic por clic" usado en toda la construcción
de esta app (ver [dataverse/05-construir-canvas-captura.md](../dataverse/05-construir-canvas-captura.md)).
El pegado masivo de YAML nunca estuvo garantizado para esta versión de
Studio (ya advertido en la cabecera de
[06-yaml-completo-para-pegar.md](06-yaml-completo-para-pegar.md)); para
containers, directamente no funciona.

## Controles disponibles en el panel Insertar

Buscando "contenedor" en Insertar aparecen 4 opciones nativas (confirmado en
Studio real):

- **Contenedor** — sin layout automático, contenedor simple.
- **Contenedor horizontal** — hijos en fila (`LayoutDirection.Horizontal`).
- **Contenedor vertical** — hijos en columna (`LayoutDirection.Vertical`).
- **Contenedor de cuadrícula** — grilla.

Usamos **Contenedor vertical** (el wrapper general de cada pantalla) y
**Contenedor horizontal** (cada "fila" de campos, con `LayoutWrap: =true`
para que se apile cuando no entra).

## Propiedades reales (confirmadas en el panel Avanzado de Studio)

Verificadas primero contra el código fuente de
[`microsoft/PowerApps-Language-Tooling`](https://github.com/microsoft/PowerApps-Language-Tooling/blob/master/src/PAModel/ControlTemplates/GlobalTemplates.cs)
y luego contra el panel **Avanzado** real de un Contenedor vertical en Studio
— coinciden exactamente:

| Propiedad | Para qué sirve | Valores típicos |
|---|---|---|
| `LayoutDirection` | Eje principal del container | `LayoutDirection.Vertical` / `LayoutDirection.Horizontal` (fijo según el tipo de container elegido en Insertar) |
| `LayoutWrap` | Si el contenido pasa a la siguiente fila/columna cuando no entra | `true` / `false` |
| `LayoutGap` | Espacio entre hijos (px) | `16` |
| `LayoutAlignItems` | Alineación en el eje transversal | `LayoutAlignItems.Start` / `.Center` / `.End` / `.Stretch` |
| `LayoutJustifyContent` | Alineación en el eje principal | `LayoutJustifyContent.Start` / `.Center` / `.End` / `.SpaceBetween` |
| `LayoutOverflowX` / `LayoutOverflowY` | Qué hacer si el contenido no cabe | `LayoutOverflow.Hide` / `LayoutOverflow.Scroll` |
| `PaddingTop`/`Left`/`Right`/`Bottom` | Márgenes internos | px |
| `LayoutMinWidth` (por hijo) | Ancho mínimo antes de que el container lo mande a la siguiente fila | px |

`X`/`Y` siguen existiendo pero **solo importan para el container raíz de
cada pantalla** (el primer container, hijo directo de la `Screen`) — una vez
que un control queda anidado *dentro* de un container de auto-layout, Studio
deja de usar su `X`/`Y` para posicionarlo.

## `scrContactoProyecto` — progreso verificado

### ✅ Paso 1 — Container exterior (`cntContenido`, Contenedor vertical)

Insertado directo sobre la pantalla (hermano de `navPestanas`, no dentro de
él). Propiedades seteadas:

| Propiedad | Valor |
|---|---|
| `X` | `0` |
| `Y` | `0` |
| `Width` | `Parent.Width` |
| `Height` | `Parent.Height` |
| `PaddingTop` | `92` |
| `PaddingLeft` | `40` |
| `PaddingRight` | `40` |
| `PaddingBottom` | `24` |
| `LayoutGap` | `16` |
| `LayoutAlignItems` | `LayoutAlignItems.Stretch` |
| `LayoutOverflowY` | `LayoutOverflow.Scroll` |

Sin cambios (ya vienen bien por defecto): `LayoutDirection.Vertical`,
`LayoutJustifyContent.Start`, `LayoutWrap = false`.

### ✅ Paso 2 — Título dentro del container

`lblTituloContacto` (ya existía en la pantalla) se arrastró para quedar como
primer hijo de `cntContenido`. Sin cambios de propiedades — al quedar dentro
de un container de auto-layout, Studio deja de usar su `X`/`Y`.

### ✅ Paso 3 — Primera fila (`cntFilaContacto`, Contenedor horizontal)

Insertado dentro de `cntContenido`, debajo del título.

| Propiedad | Valor |
|---|---|
| `LayoutWrap` | `true` |
| `LayoutGap` | `16` |
| `LayoutAlignItems` | `LayoutAlignItems.Start` |
| `Width` | `Parent.Width` |

Hijos arrastrados dentro (en este orden): `txtContactoNombre`,
`txtContactoEmail`, `txtContactoTelefono`. Cada uno con `LayoutMinWidth`
agregado (su `Width` existente no se toca):

| Control | `LayoutMinWidth` |
|---|---|
| `txtContactoNombre` | `300` |
| `txtContactoEmail` | `300` |
| `txtContactoTelefono` | `180` |

### ⏳ Pendiente

- **`lblEmailError`** — necesita ir en un container vertical propio junto
  con `txtContactoEmail` (para que el mensaje de error se mueva pegado al
  campo cuando la fila se reordena), en vez de quedar suelto dentro de
  `cntFilaContacto`.
- **Fila 2** (`cntFilaProyecto`): `txtNombreProyecto` + `txtEmpresaCliente`.
- **Fila 3** (`cntFilaUbicacion`): `txtUbicacion` + `txtCentroCosto` +
  `dtpEntrega`.
- **`ddIngenieriaPor`** — suelto, ancho completo, sin fila.
- **Fila de botones** (`cntBotones`): `btnSiguiente`, alineado a la derecha
  con `LayoutJustifyContent.End`.
- Repetir el mismo patrón en las **7 pantallas restantes**
  (`scrTableros`, `scrDocumentacion`, las 4 del tablero, `scrConfirmacion`).

## Referencias

- Construcción original (absoluta, sin responsive) de las 8 pantallas:
  [06-yaml-completo-para-pegar.md](06-yaml-completo-para-pegar.md).
- Guía clic-por-clic general: [dataverse/05-construir-canvas-captura.md](../dataverse/05-construir-canvas-captura.md).
- Fuente verificada de las propiedades de layout: [GlobalTemplates.cs](https://github.com/microsoft/PowerApps-Language-Tooling/blob/master/src/PAModel/ControlTemplates/GlobalTemplates.cs).
