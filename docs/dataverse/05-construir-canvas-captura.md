# Guía paso a paso — construir el wizard de captura (Canvas) en el navegador

Continúa después de [04-app-model-driven.md](04-app-model-driven.md) (Bloque 13).
Aquí: cómo construir la **app Canvas del wizard de captura** directamente en
`make.powerapps.com`, control por control, **sin depender de Git integration ni
de Power Platform CLI** — coherente con el resto de la construcción (todo
clicando en el navegador).

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
   elige la propiedad **OnStart** → pega:
   ```
   Set(varEditIndex; Blank());;
   ClearCollect(colTableros; [])
   ```

✅ *Verificable:* 4 pantallas creadas con esos nombres exactos; el conector
Dataverse aparece en el panel de Datos.

---

## Bloque 14b — Barra de pestañas compartida

Se construye **una sola vez** y se **copia/pega igual** en `scrContactoProyecto`,
`scrTableroForm` y `scrDocumentacion` — la fórmula de resaltado usa
`App.ActiveScreen`, así que el mismo botón se ve "activo" o no según en qué
pantalla esté pegado, sin tener que cambiar nada al copiarlo.

1. Crea un **`Group`** llamado `grpPestanas` con 3 botones dentro:
   `btnTabContacto` ("Contacto y Proyecto"), `btnTabTableros` ("Tableros"),
   `btnTabDocumentacion` ("Documentación").
2. Para **cada uno** de los 3 botones:
   - `OnSelect` (navega directo a esa pantalla, sin animación):
     ```
     // btnTabContacto
     Navigate(scrContactoProyecto; ScreenTransition.None)

     // btnTabTableros
     Navigate(scrTableroForm; ScreenTransition.None)

     // btnTabDocumentacion
     Navigate(scrDocumentacion; ScreenTransition.None)
     ```
   - `Fill` (resalta la pestaña activa):
     ```
     // btnTabContacto.Fill
     If(App.ActiveScreen = scrContactoProyecto; RGBA(30;64;175;1); RGBA(255;255;255;1))

     // btnTabTableros.Fill
     If(App.ActiveScreen = scrTableroForm; RGBA(30;64;175;1); RGBA(255;255;255;1))

     // btnTabDocumentacion.Fill
     If(App.ActiveScreen = scrDocumentacion; RGBA(30;64;175;1); RGBA(255;255;255;1))
     ```
3. Selecciona `grpPestanas` completo → **Copiar** (Ctrl+C) → pégalo (Ctrl+V) en
   `scrTableroForm` y en `scrDocumentacion`, en la misma posición.

> **Las pestañas navegan libremente** (sin validar) — es intencional: sirven
> para revisar/editar cualquier sección sin bloqueos. La validación real de
> "¿puedo avanzar?" vive en el botón **Siguiente** de cada pantalla (Bloques
> 15-17), no en las pestañas. Si prefieres bloquear el salto a "Documentación"
> hasta que "Tableros" tenga al menos un tablero, agrega la misma condición
> del Bloque 16 (`CountRows(colTableros) = 0`) al `OnSelect` de
> `btnTabDocumentacion` en las 3 copias.

✅ *Verificable:* las 3 pantallas muestran la misma barra arriba; la pestaña
de la pantalla en la que estás parado se ve resaltada; tocar otra pestaña
navega ahí sin perder los datos ya escritos (los controles retienen su valor
al navegar entre pantallas dentro de la misma sesión de la app).

---

## Bloque 15 — Pantalla 1: Contacto y Proyecto (`scrContactoProyecto`)

Para cada control de la sección **"Pantalla 1 — Contacto y Proyecto"** del
YAML: **Insertar** el tipo de control indicado (`Control:` → Texto, Entrada
de texto, Selector de fecha, Lista desplegable o Botón), renómbralo con el
nombre exacto de la clave YAML, y en cada propiedad listada pega el contenido
de la fórmula (sin el `=`).

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

1. **`lblConfirmacion`** (Texto):
   ```
   "Solicitud enviada correctamente. Código: " & varSolicitud.ReferenceCode
   ```
2. **`btnNuevaSolicitud`** (Botón) — "Registrar otra solicitud"
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
