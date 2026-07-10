# Guía paso a paso — construir el wizard de captura (Canvas) en el navegador

Continúa después de [04-app-model-driven.md](04-app-model-driven.md) (Bloque 13).
Aquí: cómo construir la **app Canvas del wizard de captura** directamente en
`make.powerapps.com`, control por control, **sin depender de Git integration ni
de Power Platform CLI** — coherente con el resto de la construcción (todo
clicando en el navegador). Usa **controles Modern** (Fluent) — en el panel
**Insertar** del estudio, bajo la sección **Modern** (no los controles
clásicos). Mapeo completo clásico → Modern en
[docs/powerapps/05-canvas-yaml-captura.md](../powerapps/05-canvas-yaml-captura.md#controles-modern-usados--mapeo-y-diferencias-clave).

## Arquitectura final (8 pantallas: 3 con pestañas + 4 pasos del tablero + confirmación)

```
┌──────────────────────────────────────────────────────┐
│  [ Contacto y Proyecto ] [ Tableros ] [ Documentación ]│  ← barra de pestañas
├──────────────────────────────────────────────────────┤
│                  contenido de la pestaña activa        │
└──────────────────────────────────────────────────────┘
```

- **`scrContactoProyecto`** — Datos de contacto y del proyecto. Pestaña.
- **`scrTableros`** — Galería de tableros agregados (ancho completo), con
  Agregar/Editar/Eliminar. Pestaña.
- **`scrTableroForm` / `scrTableroForm2a` / `scrTableroForm2b` / `scrTableroForm3`**
  — las ~37 columnas del tablero repartidas en **4 pantallas** (una por grupo
  de campos: Identificación / Ubicación-ambiente-montaje / Eléctrico / Diseño
  constructivo). Llevan **doble barra apilada** con roles distintos: la barra
  de pestañas principal arriba (`navPestanas`, siempre marcando "Tableros")
  es **solo indicativa** (`DisplayMode: =DisplayMode.View` — sin clic, sin
  hover) porque salir del sub-flujo hacia otra sección abandonaría un
  tablero a medio llenar sin aviso; su propia **barra de sub-pestañas**
  debajo (`navSubPasos`) **sí es clickeable**, con el mismo patrón de gateo
  que `navPestanas` pero con su propia variable (`varPasoMaximoTablero`, 1 a
  4): retrocede libre a un paso ya completado, bloquea el salto adelante sin
  validar. Así no hay atajo que se salte la validación, ni desde las
  pestañas principales ni desde las sub-pestañas — solo que unas lo logran
  estando deshabilitadas y otras estando gateadas. Se entra y sale desde
  `scrTableros` (Agregar o Editar → paso 1; Guardar tablero en el paso 4 →
  vuelve a `scrTableros`; Cancelar/Atrás piden confirmación). Traen la **capa
  a prueba de tontos** (banner de pendientes, error por paso, Guardar que
  salta al primer hueco — ver la sección "Capa a prueba de tontos" del 06). Se
  decidió repartir en 4 pantallas (en vez de un solo panel con todo junto)
  porque 35 campos en un panel resultaban muy densos para el editor. YAML
  completo de las 4 en
  [docs/powerapps/06-yaml-completo-para-pegar.md](../powerapps/06-yaml-completo-para-pegar.md).
- **`scrDocumentacion`** — Adjuntos, observaciones y el botón **Enviar
  solicitud**. Pestaña.
- **`scrConfirmacion`** — **no es una pestaña**: aparece después de enviar,
  fuera del flujo de pestañas (no tiene forma de volver a ella navegando).

Las **3 pantallas con pestaña** (`scrContactoProyecto`, `scrTableros`,
`scrDocumentacion`) comparten la misma barra (Bloque 14b) para poder saltar
directo a cualquiera. Cada una de las 4 pantallas del tablero valida **solo
sus propios campos** antes de dejar avanzar al siguiente paso.

> **Guía de referencia de esta arquitectura de 8 pantallas:**
> [docs/powerapps/06-yaml-completo-para-pegar.md](../powerapps/06-yaml-completo-para-pegar.md)
> tiene el YAML completo y ya actualizado de las 8 pantallas (control por
> control, fórmulas incluidas). Los Bloques de este documento (14-19)
> describían la versión anterior de "maestro-detalle en una sola pantalla" —
> esa versión quedó **superseded** (su documento se eliminó del repo); la
> construcción real sigue el diseño de 4 pantallas descrito aquí.

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
   - `scrTableros`
   - `scrTableroForm`
   - `scrTableroForm2a`
   - `scrTableroForm2b`
   - `scrTableroForm3`
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

✅ *Verificable:* 8 pantallas creadas con esos nombres exactos; el conector
Dataverse aparece en el panel de Datos.

---

## Bloque 14b — Barra de pestañas compartida (`ModernTabList`)

Power Apps trae un control Modern dedicado exactamente para esto —
**`ModernTabList@1.0.0`** ("Tabs o tab list" en el panel Insertar → Modern) —
en vez de armar la barra a mano con botones. Se inserta **una copia en las 7
pantallas** que la llevan: las 3 principales (`scrContactoProyecto`,
`scrTableros`, `scrDocumentacion`, funcional) y las 4 del tablero
(`scrTableroForm`/`2a`/`2b`/`3`, solo indicativa — ver Bloque 16); la única
diferencia entre copias es el `Default` (qué pestaña aparece marcada al
entrar a esa pantalla) y, en las 4 del tablero, el `DisplayMode`.

1. En cada una de las 7 pantallas → **Insertar → Modern → Tabs o tab list** →
   nómbralo `navPestanas`.
2. **`Items`** (igual en las 7 copias — centralizado en una sola variable
   para no repetir el array literal 7 veces):
   ```
   varNavPestanas
   ```
   (definida en `App.OnStart`: `Set(varNavPestanas; ["Contacto y Proyecto";
   "Tableros"; "Documentación"]);;` — ver sección **App — colección e
   inicialización**).
3. **`Default`** — **distinto en cada pantalla**, la pestaña que corresponde a
   esa pantalla — referenciado por índice sobre `varNavPestanas` (no como
   texto literal, para no repetir el string y que quede sincronizado si
   cambia el texto de una pestaña):
   - En `scrContactoProyecto`: `Index(varNavPestanas; 1).Value`
   - En `scrTableros` y en las 4 pantallas del tablero: `Index(varNavPestanas; 2).Value`
   - En `scrDocumentacion`: `Index(varNavPestanas; 3).Value`
4. **`OnChange`** (igual en las 3 copias) — navega según la pestaña elegida,
   pero **gatea el avance hacia adelante** con `varPasoMaximo` (variable global
   que trackea hasta qué pestaña ya se validó: `1` = Contacto, `2` = Tableros,
   `3` = Documentación; se inicializa en `App.OnStart` — Bloque 14). Hacia
   **atrás** siempre navega libre, sin condición — solo lo de adelante se
   bloquea. Los casos del `Switch` también referencian `varNavPestanas` por
   índice (no el texto literal), por la misma razón que el `Default`.
   `OnChange` solo dispara cuando el usuario cambia de pestaña (no si toca
   la ya activa), así que no navega en falso a la misma pantalla:
   ```
   Switch(
     Self.Selected.Value;
     Index(varNavPestanas; 1).Value; Navigate(scrContactoProyecto; ScreenTransition.None);
     Index(varNavPestanas; 2).Value;
       If(
         varPasoMaximo >= 2;
         Navigate(scrTableros; ScreenTransition.None);
         Notify("Completa Contacto y Proyecto antes de continuar."; NotificationType.Warning);;
         Reset(navPestanas)
       );
     If(
       varPasoMaximo >= 3;
       Navigate(scrDocumentacion; ScreenTransition.None);
       Notify("Agrega al menos un tablero antes de continuar."; NotificationType.Warning);;
       Reset(navPestanas)
     )
   )
   ```
   El `Reset(navPestanas)` es necesario porque `ModernTabList` marca
   visualmente la pestaña tocada por su cuenta (estado interno del control);
   sin el `Reset`, si el usuario toca "Documentación" y se bloquea, la pestaña
   quedaría marcada como activa aunque la app siga en la pantalla anterior.
   `Reset` la vuelve a mostrar según su `Default`.
5. **`varPasoMaximo` se actualiza en dos lugares** (no en la barra de
   pestañas): en el `OnSelect` de **`btnSiguiente`** de `scrContactoProyecto`
   (Bloque 15) agrega `Set(varPasoMaximo; Max(varPasoMaximo; 2));;` justo antes
   del `Navigate(scrTableros; ...)` de la rama válida; en el `OnSelect` de
   **`btnSiguienteTableros`** de `scrTableros` (Bloque 16) agrega
   `Set(varPasoMaximo; Max(varPasoMaximo; 3));;` antes del
   `Navigate(scrDocumentacion; ...)`. El `Max(...)` evita que retroceder y
   volver a avanzar baje el valor ya alcanzado.
6. (Opcional) `Appearance: =TabListAppearance.Underline` para el estilo visual
   de subrayado bajo la pestaña activa.

✅ *Verificable:* las 3 pantallas con pestaña muestran la misma barra arriba;
la pestaña de la pantalla en la que estás parado aparece marcada como activa
(por el `Default` de esa copia); **retroceder** a una pestaña ya completada
navega libre sin perder los datos ya escritos; **saltar hacia adelante** antes
de completar la pestaña anterior muestra el `Notify` de advertencia y no
navega (la pestaña vuelve a mostrarse en la posición correcta gracias al
`Reset`).

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
`ddIngenieriaPor`, `btnSiguiente` — el `OnSelect` de `btnSiguiente` navega a
`Navigate(scrTableros; ScreenTransition.Cover)` (la galería, no un paso del
formulario del tablero).

Agrega también la **barra de pestañas** del Bloque 14b arriba de todo.

✅ *Verificable:* al tocar **Siguiente** con campos obligatorios vacíos, aparece
el `Notify` de error; con todo completo, navega a `scrTableros`.

---

## Bloque 16 — Pantalla "Tableros" (`scrTableros`, galería) + las 4 pantallas del formulario

`scrTableros` es la galería de tableros agregados, a **ancho completo** (ya
no comparte pantalla con el formulario). **Agregar** y **Editar** navegan al
sub-flujo de 4 pantallas (`scrTableroForm` → `2a` → `2b` → `3`), que se sale
por "Guardar tablero" en el paso 4, volviendo a `scrTableros`.

Construcción completa (galería, botones, las 4 pantallas del formulario con
`Default`/`Reset` de cada campo, validación repartida por paso, popup de
confirmar eliminar) en
[docs/powerapps/06-yaml-completo-para-pegar.md](../powerapps/06-yaml-completo-para-pegar.md) —
ese documento ya trae la arquitectura de 8 pantallas completa, control por
control. Agrega la **barra de pestañas** del Bloque 14b arriba de todo en
`scrTableros` (funcional, como en las otras 2 pantallas principales). En las
**4 pantallas del formulario** agrega **las dos barras apiladas**: una copia
más de `navPestanas` en `Y=0` (`Default: =Index(varNavPestanas; 2).Value`),
con `DisplayMode: =DisplayMode.View` (solo indicativa: sin clic, sin hover —
salir del sub-flujo hacia otra sección se hace solo por Cancelar); y debajo,
`navSubPasos` en `Y=40` (`Items: =varNavSubPasos`, `Default` por índice sobre
esa misma variable — igual patrón que `varNavPestanas`), que sí queda
clickeable y gateado por `varPasoMaximoTablero` (mismo patrón que
`varPasoMaximo` de las pestañas principales: retrocede libre, bloquea el
salto adelante sin validar — `Set(varPasoMaximoTablero; 1)` en Agregar
Tablero, `Set(varPasoMaximoTablero; 4)` en Editar, y
`Set(varPasoMaximoTablero; Max(...))` en cada Siguiente).
Baja el resto de los campos de cada pantalla ~40px para que no queden
tapados por la segunda barra (coordenadas ya ajustadas en el YAML del 06).

✅ *Verificable:* **Agregar Tablero** limpia los 4 pasos (modo "nuevo") y
navega al paso 1; seleccionar una fila de la galería y tocar **Editar** carga
sus valores y navega al paso 1; **Atrás/Siguiente** se mueve entre los 4
pasos validando solo los campos de cada uno; **Guardar tablero** en el paso 4
agrega/actualiza el tablero en la galería y vuelve a `scrTableros`;
**Siguiente** en `scrTableros` con la galería vacía muestra el error y no
navega; dentro de las 4 pantallas del tablero, tocar cualquier pestaña de
**`navPestanas`** no navega a ningún lado y no muestra hover (indicativa
pura); en **`navSubPasos`**, retroceder a un paso ya completado navega
libre, y saltar adelante sin haber validado el paso anterior muestra el
`Notify` de advertencia y no navega.

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
