# Guía paso a paso — construir el wizard de captura (Canvas) en el navegador

Continúa después de [04-app-model-driven.md](04-app-model-driven.md) (Bloque 13).
Aquí: cómo construir la **app Canvas del wizard de captura** directamente en
`make.powerapps.com`, control por control, **sin depender de Git integration ni
de Power Platform CLI** — coherente con el resto de la construcción (todo
clicando en el navegador). Usa **controles Modern** (Fluent) — en el panel
**Insertar** del estudio, bajo la sección **Modern** (no los controles
clásicos). Mapeo completo clásico → Modern en
[docs/powerapps/05-canvas-yaml-captura.md](../powerapps/05-canvas-yaml-captura.md#controles-modern-usados--mapeo-y-diferencias-clave).

## Arquitectura final (4 pantallas, navegación por pestañas)

```
┌──────────────────────────────────────────────────────┐
│  [ Contacto y Proyecto ] [ Tableros ] [ Documentación ]│  ← barra de pestañas
├──────────────────────────────────────────────────────┤
│                  contenido de la pestaña activa        │
└──────────────────────────────────────────────────────┘
```

- **`scrContactoProyecto`** — Pantalla 1. Datos de contacto y del proyecto.
- **`scrTableroForm`** — Pantalla 2. **Maestro-detalle en una sola pantalla**:
  galería de tableros a la izquierda, formulario completo (~37 campos) a la
  derecha. Diseño y fórmulas completas en
  [docs/powerapps/02-canvas-guia-construccion.md](../powerapps/02-canvas-guia-construccion.md) —
  este documento solo agrega lo que falta para encajarla en la barra de
  pestañas (Bloque 16).
- **`scrDocumentacion`** — Pantalla 3. Adjuntos, observaciones y el botón
  **Enviar solicitud**.
- **`scrConfirmacion`** — **no es una pestaña**: aparece después de enviar,
  fuera del flujo de pestañas (no tiene forma de volver a ella navegando).

Las **3 primeras pantallas comparten la misma barra de pestañas** (Bloque 14b)
para poder saltar directo a cualquiera. Además, cada pantalla tiene su propio
botón **Siguiente** que valida y avanza a la próxima en orden.

> Las fórmulas exactas de cada control (qué escribir en cada propiedad) de
> Contacto/Proyecto y del envío a Dataverse están en
> [docs/powerapps/05-canvas-yaml-captura.md](../powerapps/05-canvas-yaml-captura.md)
> ("el YAML"). Este documento dice **qué control insertar, cómo llamarlo y en
> qué propiedad pegar cada fórmula** — el YAML no se pega completo, solo el
> contenido de cada fórmula (lo que va después del `=`).

---

## Bloque 14 — Crear la app Canvas y las 4 pantallas

1. Dentro de la Solución `Axon Solicitudes` → **+ Nuevo → App → App de lienzo
   en blanco**.
2. **Nombre:** `Axon - Captura de Solicitudes`. **Formato:** Tableta.
3. **Datos → Agregar datos** → conector **Dataverse** → tablas `Solicitudes` y
   `SolicitudTableros`.
4. En el panel de árbol (izquierda) → **Insertar → Pantalla en blanco**, una
   por una, y renómbralas (clic derecho → **Cambiar nombre**, o doble clic
   sobre el nombre) **exactamente** así — los `Navigate(...)` dependen de
   estos nombres:
   - `scrContactoProyecto`
   - `scrTableroForm`
   - `scrDocumentacion`
   - `scrConfirmacion`
5. Selecciona **App** en el árbol (arriba de todo) → en la barra de fórmulas,
   elige la propiedad **OnStart** → pega el contenido completo de "App —
   colección e inicialización" del YAML (`varEditIndex`, `colTableros`, y las
   ~17 colecciones `colOpc*` que usan los `ModernDropdown`/`ModernCombobox` de
   la Pantalla 2):
   ```
   Set(varEditIndex; Blank());;
   ClearCollect(colTableros; []);;
   ClearCollect(colOpcTipoEntrega; Table({Value:"tablero"; Label:"Tablero Eléctrico"}; ...));;
   // ...resto de colOpc* — ver 05-canvas-yaml-captura.md completo...
   ```

✅ *Verificable:* 4 pantallas creadas con esos nombres exactos; el conector
Dataverse aparece en el panel de Datos.

---

## Bloque 14b — Barra de pestañas compartida (`ModernTabList`)

Power Apps trae un control Modern dedicado exactamente para esto —
**`ModernTabList@1.0.0`** ("Tabs o tab list" en el panel Insertar → Modern) —
en vez de armar la barra a mano con botones. Se inserta **una copia por
pantalla** (`scrContactoProyecto`, `scrTableroForm`, `scrDocumentacion`); la
única diferencia entre copias es el `Default` (qué pestaña aparece marcada al
entrar a esa pantalla).

1. En cada una de las 3 pantallas → **Insertar → Modern → Tabs o tab list** →
   nómbralo `navPestanas`.
2. **`Items`** (igual en las 3 copias):
   ```
   ["Contacto y Proyecto"; "Tableros"; "Documentación"]
   ```
3. **`Default`** — **distinto en cada pantalla**, la pestaña que corresponde a
   esa pantalla:
   - En `scrContactoProyecto`: `"Contacto y Proyecto"`
   - En `scrTableroForm`: `"Tableros"`
   - En `scrDocumentacion`: `"Documentación"`
4. **`OnChange`** (igual en las 3 copias) — navega según la pestaña elegida.
   `OnChange` solo dispara cuando el usuario cambia de pestaña (no si toca la
   ya activa), así que no navega en falso a la misma pantalla:
   ```
   Switch(Self.Selected.Value;
     "Contacto y Proyecto"; Navigate(scrContactoProyecto; ScreenTransition.None);
     "Tableros";            Navigate(scrTableroForm; ScreenTransition.None);
     Navigate(scrDocumentacion; ScreenTransition.None)
   )
   ```
5. (Opcional) `Appearance: =TabListAppearance.Underline` para el estilo visual
   de subrayado bajo la pestaña activa.

> **Las pestañas navegan libremente** (sin validar) — es intencional: sirven
> para revisar/editar cualquier sección sin bloqueos. La validación real de
> "¿puedo avanzar?" vive en el botón **Siguiente** de cada pantalla (Bloques
> 15-17), no en las pestañas. Si prefieres bloquear el salto a "Documentación"
> hasta que "Tableros" tenga al menos un tablero, envuelve esa rama del
> `Switch` con la misma condición del Bloque 16
> (`If(CountRows(colTableros) = 0; Notify(...); Navigate(scrDocumentacion; ...))`)
> en las 3 copias.

✅ *Verificable:* las 3 pantallas muestran la misma barra arriba; la pestaña
de la pantalla en la que estás parado aparece marcada como activa (por el
`Default` de esa copia); tocar otra pestaña navega ahí sin perder los datos
ya escritos (los controles retienen su valor al navegar entre pantallas
dentro de la misma sesión de la app).

---

## Bloque 15 — Pantalla 1: Contacto y Proyecto (`scrContactoProyecto`)

Para cada control de la sección **"Pantalla 1 — Contacto y Proyecto"** del
YAML: **Insertar → Modern** → el tipo de control indicado por el `Control:`
del YAML (`ModernText`→Text, `ModernTextInput`→Text input,
`ModernDatePicker`→Date picker, `ModernDropdown`→Dropdown,
`ModernButton`→Button), renómbralo con el nombre exacto de la clave YAML, y
en cada propiedad listada pega el contenido de la fórmula (sin el `=`).

Controles: `lblTituloContacto`, `txtContactoNombre`, `txtContactoEmail`,
`lblEmailError`, `txtContactoTelefono`, `txtNombreProyecto`,
`txtEmpresaCliente`, `txtUbicacion`, `txtCentroCosto`, `dtpEntrega`,
`ddIngenieriaPor`, `btnSiguiente` — su `OnSelect` del YAML ya termina en
`Navigate(scrTableros; ScreenTransition.Cover)`; **cámbialo** a
`Navigate(scrTableroForm; ScreenTransition.Cover)` para que apunte al nombre
de pantalla real de este documento.

Agrega también la **barra de pestañas** del Bloque 14b arriba de todo.

✅ *Verificable:* al tocar **Siguiente** con campos obligatorios vacíos, aparece
el `Notify` de error; con todo completo, navega a `scrTableroForm`.

---

## Bloque 16 — Pantalla 2: Tableros (`scrTableroForm`, maestro-detalle)

Esta pantalla **no se construye desde este documento** — el diseño completo
(galería izquierda + formulario derecho con los ~37 campos, `Default`/`Reset`
de cada control, validación, guardar/nuevo/eliminar) está en
[docs/powerapps/02-canvas-guia-construccion.md](../powerapps/02-canvas-guia-construccion.md)
§3. Constrúyela siguiendo ese documento completo, y agrega solo estas dos
cosas que le faltan para encajar en el flujo de pestañas:

1. **Barra de pestañas** del Bloque 14b, arriba de todo.
2. **Botón `btnSiguienteTableros`** (junto a "Agregar Tablero"), que valida
   que exista al menos un tablero antes de pasar a Documentación:
   ```
   If(
       CountRows(colTableros) = 0;
       Notify("Agrega al menos un tablero antes de continuar."; NotificationType.Error);
       Navigate(scrDocumentacion; ScreenTransition.Cover)
   )
   ```

> En el diseño de 02, los botones **Editar** y **Eliminar** van **arriba de
> la galería** (no uno por fila) y actúan sobre `galTableros.Selected` — ver
> §3.1 y §3.7 de ese documento para las fórmulas exactas, incluyendo el
> `DisplayMode` que los deshabilita cuando no hay ninguna fila seleccionada.

✅ *Verificable:* **Agregar Tablero** limpia el panel derecho (modo "nuevo");
seleccionar una fila de la galería y tocar **Editar** carga sus valores;
**Guardar tablero** lo agrega/actualiza en la galería; **Siguiente** con la
galería vacía muestra el error y no navega.

---

## Bloque 17 — Pantalla 3: Documentación (`scrDocumentacion`)

1. **Barra de pestañas** del Bloque 14b, arriba de todo.
2. Adjuntos a nivel solicitud y `txtObservacionesProyecto` — ver la nota de
   adjuntos en el YAML (el control nativo de adjuntos necesita un registro ya
   guardado; opciones alternativas ahí mismo).
3. **`btnEnviar`** — pega el `OnSelect` completo de la sección **"Envío —
   escribir en Dataverse (Patch)"** del YAML (valida que haya tableros, crea
   la `Solicitud`, crea cada `SolicitudTablero` con el mapeo de Choices, y
   termina en `Navigate(scrConfirmacion)`).

✅ *Verificable:* **Enviar solicitud** crea la cabecera y los tableros en
Dataverse y navega a `scrConfirmacion`.

---

## Bloque 18 — Pantalla de confirmación (`scrConfirmacion`)

No lleva la barra de pestañas — es un estado terminal, no una sección del
formulario.

1. **`lblConfirmacion`** (`ModernText@1.0.0`):
   ```
   "Solicitud enviada correctamente. Código: " & varSolicitud.ReferenceCode
   ```
2. **`btnNuevaSolicitud`** (`ModernButton@1.0.0`) — "Registrar otra solicitud"
   - `OnSelect`: `Navigate(scrContactoProyecto)`

✅ *Verificable:* tras **Enviar solicitud**, aparece esta pantalla mostrando el
`ReferenceCode` real (Autonumber, ej. `SOL-00001`) que Dataverse asignó a la
solicitud recién creada, y no muestra la barra de pestañas.

---

## Bloque 19 — Prueba end-to-end

1. **Vista previa** (F5 o el botón ▷ arriba a la derecha).
2. Completa Contacto/Proyecto → **Siguiente**. En Tableros, agrega dos
   tableros (marcando al menos un campo "otro" en cada uno, para ejercitar
   los `Switch` de conversión) → **Siguiente**. Prueba también saltar entre
   pestañas a mitad de captura y confirma que no se pierde lo ya escrito. En
   Documentación → **Enviar solicitud**.
3. En `make.powerapps.com` → tabla `Solicitudes` → confirma 1 fila nueva; tabla
   `SolicitudTableros` → confirma 2 filas nuevas con el Lookup `Solicitud`
   apuntando a esa fila y cada columna Choice con la opción correcta (no en
   blanco). Ver también la sección **"Verificación antes de implementar"** al
   final del YAML.

## Después de esto
Con el wizard de captura funcionando: conecta el **flujo en tiempo real de
estados** ([02-bpf-y-roles.md](02-bpf-y-roles.md) §3) para que el cambio de
`Estado` desde la app Model-driven quede validado, y el **Power Automate F-1**
de notificaciones ([powerautomate/03-flujos.md](../powerautomate/03-flujos.md)).
