# Guía de construcción — Power App Canvas de captura (maestro-detalle, Modern controls)

> ⚠️ **Superseded.** El diseño de "maestro-detalle en una sola pantalla" que
> describe este documento (galería + ~35 campos en un panel al lado) resultó
> muy denso visualmente para trabajar en el editor de Power Apps. Se
> reemplazó por **4 pantallas separadas** para el formulario del tablero
> (una por grupo de campos), más una galería a ancho completo — ver
> [dataverse/05-construir-canvas-captura.md](../dataverse/05-construir-canvas-captura.md)
> y el YAML completo en
> [06-yaml-completo-para-pegar.md](06-yaml-completo-para-pegar.md). Este
> documento queda como referencia histórica de las fórmulas (`Default`,
> validación, `Reset`, el `With(){registro}` de guardar) — esas mismas
> fórmulas se reusaron, solo cambia en qué pantalla vive cada campo.

App que reemplaza al `PublicFormWizard` de Filament, sobre **Dataverse**, con
los **controles Modern** de Power Apps (Fluent). Arquitectura final: **3
pantallas navegadas por pestañas** (Contacto y Proyecto / Tableros /
Documentación, con `ModernTabList` compartido que permite saltar directo a
cualquiera) + una **pantalla de confirmación** fuera del flujo de pestañas.
Este documento es la referencia completa de la **Pantalla 2 — Tableros**
(`scrTableroForm`), construida como **maestro-detalle en una sola pantalla**:
galería de tableros a la izquierda + formulario completo a la derecha.

> **Dónde está el resto:** la app completa (las 4 pantallas, la barra de
> pestañas compartida, y las Pantallas 1 y 3) se arma siguiendo
> [dataverse/05-construir-canvas-captura.md](../dataverse/05-construir-canvas-captura.md) —
> ese documento manda a **este** para el detalle de la Pantalla 2. Las
> fórmulas exactas de cada campo (`Items`, tipos, opciones, las colecciones
> `colOpc*`) están en [05-canvas-yaml-captura.md](05-canvas-yaml-captura.md)
> — aquí se referencian, no se duplican.

> **Mapeo de controles clásico → Modern** (identificadores YAML, propiedades
> que cambiaron): tabla completa en
> [05-canvas-yaml-captura.md](05-canvas-yaml-captura.md#controles-modern-usados--mapeo-y-diferencias-clave).
> Lo más relevante para esta pantalla: `Toggle.Value` → **`Toggle.Checked`**;
> los campos numéricos usan **`ModernNumberInput`** (propiedad `.Value`, ya
> numérica — no más `Value(control.Text)`); el `Default` de un
> `ModernDropdown` necesita el **record** completo de `Items`, no un string
> — se resuelve con `LookUp(colOpcX; Value = codigo)`.

> **Licencia:** conector **Dataverse** — incluido en el Plan Developer /
> licencia de la empresa. No requiere Power Apps premium.

> **Notación Power Fx:** separador decimal `,` en el entorno → argumentos de
> función con `;`, encadenado de instrucciones con `;;`.

---

## 0. Preparación

1. En **make.powerapps.com**, dentro de la Solución `Axon Solicitudes` → **+
   Nuevo → App → App de lienzo en blanco**. **Formato:** Tableta.
2. **Datos → Agregar datos** → conector **Dataverse** → tablas `Solicitudes` y
   `SolicitudTableros` (deben existir ya — Bloques 2-4 de
   [00-construir-en-powerapps.md](../dataverse/00-construir-en-powerapps.md)).
3. Pantallas a crear: `scrContactoProyecto`, `scrTableroForm` (la
   maestro-detalle de este documento), `scrDocumentacion`, `scrConfirmacion` —
   ver [dataverse/05-construir-canvas-captura.md](../dataverse/05-construir-canvas-captura.md)
   Bloque 14 para la creación de las 4 y Bloque 14b para la barra de pestañas
   compartida entre las 3 primeras.

---

## 1. Modelo de datos en memoria

La solicitud se arma en memoria y se guarda toda al final (cabecera + N
tableros), igual que el wizard original. Una **colección local** de tableros,
una variable para saber si se está editando uno existente, y las
**colecciones de opciones** (`colOpc*`) que usan los `ModernDropdown`/
`ModernCombobox` de esta pantalla — todo esto se define una sola vez en
`App.OnStart`; fórmula completa en
[05-canvas-yaml-captura.md](05-canvas-yaml-captura.md#app--colección-e-inicialización).

```powerfx
// App.OnStart (resumen — ver 05 para las ~17 colecciones colOpc* completas)
Set(varEditIndex; Blank());;       // Blank = nuevo tablero; record = editando uno existente
ClearCollect(colTableros; []);;    // tableros de la solicitud en curso
ClearCollect(colOpcTipoEntrega; Table({Value:"tablero"; Label:"Tablero Eléctrico"}; /* ... */))
// ...resto de colOpc* (uno por cada ModernDropdown/ModernCombobox de tipo Choice)...
```

---

## 2. Pantalla 1 — Contacto y Proyecto (`scrContactoProyecto`)

Fórmulas exactas de cada control (incluida la validación de email y el botón
Siguiente) en la sección **"Pantalla 1 — Contacto y Proyecto"** de
[05-canvas-yaml-captura.md](05-canvas-yaml-captura.md). Botón **Siguiente** →
`Navigate(scrTableroForm; ScreenTransition.Cover)` (el YAML trae
`scrTableros`, cámbialo a `scrTableroForm`). Agrega también la barra de
pestañas compartida (Bloque 14b de
[dataverse/05](../dataverse/05-construir-canvas-captura.md)).

---

## 3. Pantalla 2 — Tableros (maestro-detalle en una sola pantalla)

`scrTableroForm` tiene **dos `Container`** (clásicos — no hay versión Modern
de contenedores): `cntGaleria` (izquierda, ~30% del ancho) y `cntFormulario`
(derecha, ~70%, con scroll vertical — son ~37 campos, no entran en el alto de
una pantalla Tableta). Arriba de todo va la barra de pestañas compartida
(Bloque 14b de [dataverse/05](../dataverse/05-construir-canvas-captura.md)).

### 3.1 Contenedor izquierdo — galería de tableros

- **`galTableros`** (Galería vertical — **clásica**, no hay `Gallery` Modern)
  `Items`: `colTableros`.
- Dentro de la plantilla:
  - **`lblNombreTablero`** (`ModernText@1.0.0`), `Text`: `ThisItem.Nombre`
  - **`lblResumenTablero`** (`ModernText@1.0.0`), `Text`: `"×" & ThisItem.Cantidad`
  - **Resaltar la fila seleccionada** — `Fill` del contenedor de la plantilla
    (el `Group`/container clásico sí tiene `Fill`, no cambia):
    ```powerfx
    If(ThisItem = galTableros.Selected; RGBA(99;102;241;0.15); RGBA(255;255;255;1))
    ```
    (si tu versión de Studio ya resalta la fila tocada de forma nativa, esta
    fórmula es opcional.)
- **`lblSinTableros`** (`ModernText@1.0.0`) — "Aún no hay tableros en esta solicitud."
  `Visible`: `CountRows(colTableros) = 0`
- **`lblContadorTableros`** (`ModernText@1.0.0`, opcional) — `Text`:
  `CountRows(colTableros) & " tablero(s) agregado(s)"`
- **`btnEditarTablero`** y **`btnEliminarTablero`** (`ModernButton@1.0.0`) —
  van **arriba de la galería**, no dentro de cada fila; actúan sobre
  `galTableros.Selected` (la fila tocada). Deshabilitados si no hay ninguna
  seleccionada (`DisplayMode` sigue existiendo igual en `ModernButton`):
  ```powerfx
  // DisplayMode de ambos botones
  If(IsBlank(galTableros.Selected); DisplayMode.Disabled; DisplayMode.Edit)
  ```
  `btnEditarTablero.OnSelect` — ver 3.3. `btnEliminarTablero.OnSelect` — ver 3.7.
- **`btnAgregarTablero`** (`ModernButton@1.0.0`) — "+ Agregar Tablero", bajo la
  galería — ver 3.6.
- **`btnSiguienteTableros`** (`ModernButton@1.0.0`) — "Siguiente", junto al
  anterior. Valida que haya al menos un tablero antes de pasar a Documentación:
  ```powerfx
  // btnSiguienteTableros.OnSelect
  If(
      CountRows(colTableros) = 0;
      Notify("Agrega al menos un tablero antes de continuar."; NotificationType.Error);
      Navigate(scrDocumentacion; ScreenTransition.Cover)
  )
  ```

### 3.2 Contenedor derecho — indicador de modo

**`lblModoEdicion`** (`ModernText@1.0.0`), arriba del formulario:
```powerfx
If(IsBlank(varEditIndex); "Nuevo tablero"; "Editando: " & varEditIndex.Nombre)
```
Sin esto es fácil perder de vista si estás agregando uno nuevo o modificando
uno ya guardado.

### 3.3 Por qué hace falta `Reset()` — y el `OnSelect` de "Editar"

`Default`/`Checked` de un control **se lee una sola vez**, al crearse o al
llamar `Reset(control)`. Cambiar `varEditIndex` **no** refresca los controles
solos. Por eso, cada vez que se carga un tablero distinto en el panel derecho
(o se limpia el formulario), hay que resetear los ~35 controles.

Tocar una fila de `galTableros` solo la **selecciona** (la resalta); **no**
carga sus datos en el panel derecho todavía — eso lo hace explícitamente
`btnEditarTablero`, para que seleccionar y hojear la lista no pise el tablero
que se está editando por accidente:

```powerfx
// btnEditarTablero.OnSelect
Set(varEditIndex; galTableros.Selected);;
Reset(ddTipoEntrega);; Reset(ddInstalacionNuevaReemplazo);; Reset(txtNombreTablero);;
Reset(numCantidad);; Reset(ddTipoTablero);; Reset(txtOtroTipoTablero);;
Reset(txtFuncionTablero);; Reset(txtCargasAAlimentar);; Reset(numNumeroCircuitos);;
Reset(ddUbicacion);; Reset(cmbAmbienteEspecial);; Reset(txtOtroAmbiente);;
Reset(ddGradoIP);; Reset(ddGradoIK);; Reset(ddTipoMontaje);; Reset(tglRestricciones);;
Reset(numAltoMax);; Reset(numAnchoMax);; Reset(numFondoMax);; Reset(txtCondicionesInstalacion);;
Reset(ddTension);; Reset(numOtraTension);; Reset(ddSistema);; Reset(txtOtroSistema);;
Reset(numPotencia);; Reset(ddUnidadPotencia);; Reset(ddFrecuencia);; Reset(numOtraFrecuencia);;
Reset(cmbProteccionesRequeridas);; Reset(cmbMarcasPreferidas);; Reset(ddMaterialGabinete);;
Reset(ddColorGabinete);; Reset(ddTipoVentilacion);; Reset(ddExpansionFutura);;
Reset(txtObservacionesTablero)
```

`lblCorriente` **no** entra en la lista — es un `ModernText` calculado, se
recalcula solo en cuanto cambian los controles de los que depende.

> **El orden de los `Reset()` importa** cuando un control depende de otro:
> `ddGradoIP.Items` depende de `ddUbicacion.Selected.Value` (filtro
> interior/exterior), así que `ddUbicacion` debe resetearse **antes** que
> `ddGradoIP` — como en la lista de arriba — o `ddGradoIP` puede intentar
> preseleccionar una opción que todavía no existe en su lista filtrada. Mismo
> cuidado entre `ddTipoEntrega`→`ddTipoTablero`→`txtOtroTipoTablero`.

Copia **este mismo bloque de 33 `Reset()`** en los otros tres lugares que lo
necesitan (3.6 Nuevo tablero, 3.5 Guardar tablero, 3.7 Eliminar tablero) — Power
Fx en Canvas no tiene funciones reutilizables definidas por el usuario, así
que se repite tal cual en cada `OnSelect`.

### 3.4 `Default`/`Checked` de cada control — carga los valores al editar

Con las colecciones `colOpc*` (definidas en `App.OnStart`, ver
[05-canvas-yaml-captura.md](05-canvas-yaml-captura.md#app--colección-e-inicialización)),
el `Default` de cada `ModernDropdown` es un `LookUp` — más simple y sin
repetir el par código/etiqueta que ya está en `Items`.

| Control | Tipo | Propiedad | Fórmula |
|---|---|---|---|
| `txtNombreTablero` | ModernTextInput | `Default` | `If(IsBlank(varEditIndex); ""; varEditIndex.Nombre)` |
| `numCantidad` | ModernNumberInput | `Default` | `If(IsBlank(varEditIndex); 1; varEditIndex.Cantidad)` |
| `ddTipoEntrega` | ModernDropdown | `Default` | `If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTipoEntrega; Value = varEditIndex.TipoEntrega))` |
| `ddInstalacionNuevaReemplazo` | ModernDropdown | `Default` | `If(IsBlank(varEditIndex); Blank(); LookUp(colOpcInstalacionNuevaReemplazo; Value = varEditIndex.InstalacionNuevaReemplazo))` |
| `ddTipoTablero` | ModernDropdown | `Default` | `If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTipoTablero; Value = varEditIndex.TipoTablero))` |
| `txtOtroTipoTablero` | ModernTextInput | `Default` | `If(IsBlank(varEditIndex); ""; varEditIndex.OtroTipoTablero)` |
| `txtFuncionTablero` | ModernTextInput | `Default` | `If(IsBlank(varEditIndex); ""; varEditIndex.FuncionTablero)` |
| `txtCargasAAlimentar` | ModernTextInput | `Default` | `If(IsBlank(varEditIndex); ""; varEditIndex.CargasAAlimentar)` |
| `numNumeroCircuitos` | ModernNumberInput | `Default` | `If(IsBlank(varEditIndex); 0; varEditIndex.NumeroCircuitos)` |
| `ddUbicacion` | ModernDropdown | `Default` | `If(IsBlank(varEditIndex); Blank(); LookUp(colOpcUbicacion; Value = varEditIndex.Ubicacion))` |
| `cmbAmbienteEspecial` | ModernCombobox | `DefaultSelectedItems` | `If(IsBlank(varEditIndex); Table(); varEditIndex.AmbienteEspecial)` |
| `txtOtroAmbiente` | ModernTextInput | `Default` | `If(IsBlank(varEditIndex); ""; varEditIndex.OtroAmbienteEspecial)` |
| `ddGradoIP` | ModernDropdown (array plano) | `Default` | `If(IsBlank(varEditIndex); Blank(); {Value: varEditIndex.GradoIP})` |
| `ddGradoIK` | ModernDropdown (array plano) | `Default` | `If(IsBlank(varEditIndex); Blank(); {Value: varEditIndex.GradoIK})` |
| `ddTipoMontaje` | ModernDropdown | `Default` | `If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTipoMontaje; Value = varEditIndex.TipoMontaje))` |
| `tglRestricciones` | ModernToggle | **`Checked`** | `If(IsBlank(varEditIndex); false; varEditIndex.RestriccionesDimension)` |
| `numAltoMax` | ModernNumberInput | `Default` | `If(IsBlank(varEditIndex); 0; varEditIndex.AltoMaxMm)` |
| `numAnchoMax` | ModernNumberInput | `Default` | `If(IsBlank(varEditIndex); 0; varEditIndex.AnchoMaxMm)` |
| `numFondoMax` | ModernNumberInput | `Default` | `If(IsBlank(varEditIndex); 0; varEditIndex.FondoMaxMm)` |
| `txtCondicionesInstalacion` | ModernTextInput | `Default` | `If(IsBlank(varEditIndex); ""; varEditIndex.CondicionesInstalacion)` |
| `ddTension` | ModernDropdown | `Default` | `If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTension; Value = varEditIndex.TensionSuministro))` |
| `numOtraTension` | ModernNumberInput | `Default` | `If(IsBlank(varEditIndex); 0; varEditIndex.OtraTension)` |
| `ddSistema` | ModernDropdown | `Default` | `If(IsBlank(varEditIndex); Blank(); LookUp(colOpcSistema; Value = varEditIndex.SistemaElectrico))` |
| `txtOtroSistema` | ModernTextInput | `Default` | `If(IsBlank(varEditIndex); ""; varEditIndex.OtroSistemaElectrico)` |
| `numPotencia` | ModernNumberInput | `Default` | `If(IsBlank(varEditIndex); Blank(); varEditIndex.PotenciaEstimada)` |
| `ddUnidadPotencia` | ModernDropdown | `Default` | `If(IsBlank(varEditIndex); LookUp(colOpcUnidadPotencia; Value="kW"); LookUp(colOpcUnidadPotencia; Value = varEditIndex.UnidadPotencia))` |
| `ddFrecuencia` | ModernDropdown | `Default` | `If(IsBlank(varEditIndex); Blank(); LookUp(colOpcFrecuencia; Value = varEditIndex.Frecuencia))` |
| `numOtraFrecuencia` | ModernNumberInput | `Default` | `If(IsBlank(varEditIndex); 0; varEditIndex.OtraFrecuencia)` |
| `cmbProteccionesRequeridas` | ModernCombobox | `DefaultSelectedItems` | `If(IsBlank(varEditIndex); Table(); varEditIndex.ProteccionesRequeridas)` |
| `cmbMarcasPreferidas` | ModernCombobox | `DefaultSelectedItems` | `If(IsBlank(varEditIndex); Table(); varEditIndex.MarcasPreferidas)` |
| `ddMaterialGabinete` | ModernDropdown | `Default` | `If(IsBlank(varEditIndex); Blank(); LookUp(colOpcMaterialGabinete; Value = varEditIndex.MaterialGabinete))` |
| `ddColorGabinete` | ModernDropdown | `Default` | `If(IsBlank(varEditIndex); LookUp(colOpcColorGabinete; Value="7035"); LookUp(colOpcColorGabinete; Value = varEditIndex.ColorGabinete))` |
| `ddTipoVentilacion` | ModernDropdown | `Default` | `If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTipoVentilacion; Value = varEditIndex.TipoVentilacion))` |
| `ddExpansionFutura` | ModernDropdown | `Default` | `If(IsBlank(varEditIndex); Blank(); LookUp(colOpcExpansionFutura; Value = varEditIndex.ExpansionFutura))` |
| `txtObservacionesTablero` | ModernTextInput | `Default` | `If(IsBlank(varEditIndex); ""; varEditIndex.ObservacionesTablero)` |

> `ddGradoIP`/`ddGradoIK` no tienen colección `colOpc*` (su `Items` es un
> array de texto plano) — por eso su `Default` envuelve el código en
> `{Value: ...}` a mano en vez de usar `LookUp`.

Lógica condicional de `Visible` (qué campo aparece según otro) — sin cambios
respecto a lo ya definido en el `Items`/`Visible` de cada control en
[05-canvas-yaml-captura.md](05-canvas-yaml-captura.md) §"Pantalla del tablero".

### 3.5 Guardar tablero

```powerfx
// btnGuardarTablero.OnSelect (ModernButton) — antes de esto corre la validación de 3.4b
With(
    {
        registro: {
            Nombre: txtNombreTablero.Text;
            Cantidad: numCantidad.Value;
            Orden: If(IsBlank(varEditIndex); CountRows(colTableros); varEditIndex.Orden);
            TipoEntrega: ddTipoEntrega.Selected.Value;
            InstalacionNuevaReemplazo: ddInstalacionNuevaReemplazo.Selected.Value;
            TipoTablero: If(ddTipoEntrega.Selected.Value <> "tablero"; Blank(); ddTipoTablero.Selected.Value);
            OtroTipoTablero: If(ddTipoTablero.Selected.Value <> "otro"; Blank(); txtOtroTipoTablero.Text);
            FuncionTablero: txtFuncionTablero.Text;
            CargasAAlimentar: txtCargasAAlimentar.Text;
            NumeroCircuitos: numNumeroCircuitos.Value;
            Ubicacion: ddUbicacion.Selected.Value;
            AmbienteEspecial: cmbAmbienteEspecial.SelectedItems;
            OtroAmbienteEspecial: If(Not("otro" in cmbAmbienteEspecial.SelectedItems.Value); Blank(); txtOtroAmbiente.Text);
            GradoIP: ddGradoIP.Selected.Value;
            GradoIK: ddGradoIK.Selected.Value;
            TipoMontaje: ddTipoMontaje.Selected.Value;
            RestriccionesDimension: tglRestricciones.Checked;
            AltoMaxMm: If(Not(tglRestricciones.Checked); Blank(); numAltoMax.Value);
            AnchoMaxMm: If(Not(tglRestricciones.Checked); Blank(); numAnchoMax.Value);
            FondoMaxMm: If(Not(tglRestricciones.Checked); Blank(); numFondoMax.Value);
            CondicionesInstalacion: txtCondicionesInstalacion.Text;
            TensionSuministro: ddTension.Selected.Value;
            OtraTension: If(ddTension.Selected.Value <> "otro"; Blank(); numOtraTension.Value);
            SistemaElectrico: ddSistema.Selected.Value;
            OtroSistemaElectrico: If(ddSistema.Selected.Value <> "otro"; Blank(); txtOtroSistema.Text);
            PotenciaEstimada: numPotencia.Value;
            UnidadPotencia: ddUnidadPotencia.Selected.Value;
            CorrienteNominal: Value(lblCorriente.Text);
            Frecuencia: ddFrecuencia.Selected.Value;
            OtraFrecuencia: If(ddFrecuencia.Selected.Value <> "otro"; Blank(); numOtraFrecuencia.Value);
            ProteccionesRequeridas: cmbProteccionesRequeridas.SelectedItems;
            MarcasPreferidas: cmbMarcasPreferidas.SelectedItems;
            MaterialGabinete: ddMaterialGabinete.Selected.Value;
            ColorGabinete: ddColorGabinete.Selected.Value;
            TipoVentilacion: ddTipoVentilacion.Selected.Value;
            ExpansionFutura: ddExpansionFutura.Selected.Value;
            ObservacionesTablero: txtObservacionesTablero.Text
        }
    };
    If(
        IsBlank(varEditIndex);
        Collect(colTableros; registro);
        Patch(colTableros; varEditIndex; registro)
    )
);;
Set(varEditIndex; Blank());;
Reset(ddTipoEntrega);; Reset(ddInstalacionNuevaReemplazo);; Reset(txtNombreTablero);;
Reset(numCantidad);; Reset(ddTipoTablero);; Reset(txtOtroTipoTablero);;
Reset(txtFuncionTablero);; Reset(txtCargasAAlimentar);; Reset(numNumeroCircuitos);;
Reset(ddUbicacion);; Reset(cmbAmbienteEspecial);; Reset(txtOtroAmbiente);;
Reset(ddGradoIP);; Reset(ddGradoIK);; Reset(ddTipoMontaje);; Reset(tglRestricciones);;
Reset(numAltoMax);; Reset(numAnchoMax);; Reset(numFondoMax);; Reset(txtCondicionesInstalacion);;
Reset(ddTension);; Reset(numOtraTension);; Reset(ddSistema);; Reset(txtOtroSistema);;
Reset(numPotencia);; Reset(ddUnidadPotencia);; Reset(ddFrecuencia);; Reset(numOtraFrecuencia);;
Reset(cmbProteccionesRequeridas);; Reset(cmbMarcasPreferidas);; Reset(ddMaterialGabinete);;
Reset(ddColorGabinete);; Reset(ddTipoVentilacion);; Reset(ddExpansionFutura);;
Reset(txtObservacionesTablero)
```

Dos correcciones respecto a una implementación ingenua:
- **`Orden`** se recalcula con `CountRows` solo si es un tablero **nuevo**; si
  estás editando uno existente conserva su `Orden` original (si no, cada
  edición lo movería al final de la lista).
- **Campos "Otro"/condicionales que quedaron ocultos** (`OtroTipoTablero`,
  `OtroAmbienteEspecial`, `AltoMaxMm`/`AnchoMaxMm`/`FondoMaxMm`, `OtraTension`,
  `OtroSistemaElectrico`, `OtraFrecuencia`) se limpian a `Blank()` si su
  condición ya no aplica — si no, un tablero que tuvo `TipoTablero="otro"` y
  luego se cambió a `"fuerza"` se guardaría con el texto viejo de
  `OtroTipoTablero` todavía dentro, aunque el campo esté oculto en pantalla.

Nota sobre `ModernNumberInput`: ya no hace falta `Clamp(...)` para `Cantidad`
— el `Min`/`Max` del propio control (`numCantidad`, ver
[05-canvas-yaml-captura.md](05-canvas-yaml-captura.md)) impide que el usuario
escriba fuera de rango, y `.Value` ya llega numérico.

### 3.4b Validación antes de guardar

Antes del `With(...)` de 3.5, agrega esta validación (mismo patrón que
Pantalla 1):

```powerfx
If(
    Or(
        IsBlank(txtNombreTablero.Text);
        IsBlank(ddTipoEntrega.Selected.Value);
        IsBlank(ddInstalacionNuevaReemplazo.Selected.Value);
        And(ddTipoEntrega.Selected.Value = "tablero"; IsBlank(ddTipoTablero.Selected.Value));
        IsBlank(ddUbicacion.Selected.Value);
        IsBlank(ddGradoIP.Selected.Value);
        IsBlank(ddTipoMontaje.Selected.Value);
        IsBlank(ddTension.Selected.Value);
        IsBlank(ddSistema.Selected.Value);
        IsBlank(numPotencia.Value) || numPotencia.Value <= 0;
        IsBlank(ddFrecuencia.Selected.Value);
        CountRows(cmbProteccionesRequeridas.SelectedItems) = 0;
        IsBlank(ddMaterialGabinete.Selected.Value);
        IsBlank(ddTipoVentilacion.Selected.Value);
        IsBlank(ddExpansionFutura.Selected.Value)
    );
    Notify("Completa los campos obligatorios del tablero antes de guardar."; NotificationType.Error);
    // si pasa la validación, continúa con el With(...) de 3.5
)
```

`numPotencia.Value <= 0` ya valida que la potencia sea mayor a cero, no solo
que no esté vacía — más estricto que el `IsBlank` que hacía el `TextInput`
clásico.

### 3.6 Nuevo tablero

```powerfx
// btnNuevoTablero.OnSelect (ModernButton)
Set(varEditIndex; Blank());;
Reset(ddTipoEntrega);; Reset(ddInstalacionNuevaReemplazo);; Reset(txtNombreTablero);;
Reset(numCantidad);; Reset(ddTipoTablero);; Reset(txtOtroTipoTablero);;
Reset(txtFuncionTablero);; Reset(txtCargasAAlimentar);; Reset(numNumeroCircuitos);;
Reset(ddUbicacion);; Reset(cmbAmbienteEspecial);; Reset(txtOtroAmbiente);;
Reset(ddGradoIP);; Reset(ddGradoIK);; Reset(ddTipoMontaje);; Reset(tglRestricciones);;
Reset(numAltoMax);; Reset(numAnchoMax);; Reset(numFondoMax);; Reset(txtCondicionesInstalacion);;
Reset(ddTension);; Reset(numOtraTension);; Reset(ddSistema);; Reset(txtOtroSistema);;
Reset(numPotencia);; Reset(ddUnidadPotencia);; Reset(ddFrecuencia);; Reset(numOtraFrecuencia);;
Reset(cmbProteccionesRequeridas);; Reset(cmbMarcasPreferidas);; Reset(ddMaterialGabinete);;
Reset(ddColorGabinete);; Reset(ddTipoVentilacion);; Reset(ddExpansionFutura);;
Reset(txtObservacionesTablero)
```

Es literalmente la misma acción que hace "Cancelar edición" — no hace falta un
botón aparte.

### 3.7 Eliminar tablero (con confirmación)

`btnEliminarTablero` (arriba de la galería, ver 3.1) actúa sobre
`galTableros.Selected`:

```powerfx
// btnEliminarTablero.OnSelect (ModernButton)
UpdateContext({locItemAEliminar: galTableros.Selected; mostrarConfirmarEliminar: true})
```

Y un **`Group`/popup** `grpConfirmarEliminar` (clásico — oculto por defecto,
`Visible: mostrarConfirmarEliminar`) con un `ModernText` "¿Eliminar este
tablero?" y dos `ModernButton`:

```powerfx
// btnConfirmarSi.OnSelect
UpdateContext({mostrarConfirmarEliminar: false});;
Remove(colTableros; locItemAEliminar);;
If(
    locItemAEliminar = varEditIndex;
    Set(varEditIndex; Blank());;
    Reset(ddTipoEntrega);; Reset(ddInstalacionNuevaReemplazo);; Reset(txtNombreTablero);;
    Reset(numCantidad);; Reset(ddTipoTablero);; Reset(txtOtroTipoTablero);;
    Reset(txtFuncionTablero);; Reset(txtCargasAAlimentar);; Reset(numNumeroCircuitos);;
    Reset(ddUbicacion);; Reset(cmbAmbienteEspecial);; Reset(txtOtroAmbiente);;
    Reset(ddGradoIP);; Reset(ddGradoIK);; Reset(ddTipoMontaje);; Reset(tglRestricciones);;
    Reset(numAltoMax);; Reset(numAnchoMax);; Reset(numFondoMax);; Reset(txtCondicionesInstalacion);;
    Reset(ddTension);; Reset(numOtraTension);; Reset(ddSistema);; Reset(txtOtroSistema);;
    Reset(numPotencia);; Reset(ddUnidadPotencia);; Reset(ddFrecuencia);; Reset(numOtraFrecuencia);;
    Reset(cmbProteccionesRequeridas);; Reset(cmbMarcasPreferidas);; Reset(ddMaterialGabinete);;
    Reset(ddColorGabinete);; Reset(ddTipoVentilacion);; Reset(ddExpansionFutura);;
    Reset(txtObservacionesTablero)
)
```
```powerfx
// btnConfirmarNo.OnSelect
UpdateContext({mostrarConfirmarEliminar: false})
```

`locItemAEliminar` se captura con `UpdateContext` **antes** de `Remove(...)`
porque, al borrar el registro de `colTableros`, la galería se re-renderiza al
instante y `galTableros.Selected` puede quedar en blanco para el resto de la
fórmula si se sigue referenciando después del `Remove`.

---

## 4. Pantalla 3 — Documentación y envío (`scrDocumentacion`)

- Barra de pestañas compartida (Bloque 14b de
  [dataverse/05](../dataverse/05-construir-canvas-captura.md)) arriba de todo.
- Adjuntos a nivel solicitud (specs técnicas, fotos del sitio) y
  `txtObservacionesProyecto`.
- **`btnEnviar`** (`ModernButton@1.0.0`) — fórmula completa (validación de "al
  menos un tablero", `Patch` de la cabecera, `ForAll`/`Patch` de cada tablero
  con el mapeo Choice → Dataverse, y navegación a confirmación) en la sección
  **"Envío — escribir en Dataverse (Patch)"** de
  [05-canvas-yaml-captura.md](05-canvas-yaml-captura.md).

> **Adjuntos:** el control nativo de adjuntos de Power Apps solo funciona
> sobre un registro **ya guardado** — no se puede adjuntar a nivel tablero
> mientras `colTableros` es una colección en memoria. Ver la nota de
> "Documentación/adjuntos del tablero" en 05-canvas-yaml-captura.md para las
> dos alternativas (reabrir tras el envío, o gestionarlo desde la app
> Model-driven).

---

## 5. Pantalla de confirmación (`scrConfirmacion`)

```powerfx
// lblConfirmacion.Text (ModernText)
"Solicitud enviada correctamente. Código: " & varSolicitud.ReferenceCode
```
Botón **"Registrar otra solicitud"** (`ModernButton@1.0.0`) →
`Navigate(scrContactoProyecto)`. Esta pantalla **no** lleva la barra de
pestañas — es un estado terminal fuera del flujo de las 3 secciones.

---

## 6. Qué más podrías necesitar

Revisando este diseño (galería + formulario en una pantalla, ~37 campos por
tablero), esto es lo que conviene resolver antes de darlo por terminado:

### Ya incorporado en esta guía
- **Validación de campos obligatorios** antes de guardar un tablero (3.4b),
  incluyendo `PotenciaEstimada > 0` (no solo "no vacío") — sin esto se pueden
  guardar tableros vacíos o con potencia cero.
- **Confirmación antes de eliminar** (3.7) — un ícono de basurero sin
  confirmación es fácil de tocar por error en una lista larga.
- **Indicador de modo** "Nuevo tablero" / "Editando: X" (3.2) — sin esto no
  se sabe si se está creando o modificando algo.
- **Corrección de `Orden`** al editar (no se recalcula, conserva el original).
- **Limpieza de campos "Otro"/condicionales** que quedaron ocultos pero con
  datos viejos (3.5) — si no, se guardan datos huérfanos e inconsistentes con
  el resto del registro.
- **Colecciones de opciones reutilizables** (`colOpc*`) — el `Default` de
  3.4 ya usa `LookUp` contra ellas en vez de repetir cada par código/etiqueta
  en un `Switch`; una sola fuente de verdad compartida con `Items`.
- **Rangos numéricos** — `Min`/`Max` nativos de `ModernNumberInput`
  (`numCantidad`: 1-999) reemplazan el `Clamp(...)` manual.

### Recomendado revisar al construir
- **El orden de los `Reset()` importa** cuando un control depende de otro
  (`ddUbicacion`→`ddGradoIP`, `ddTipoEntrega`→`ddTipoTablero`→
  `txtOtroTipoTablero`) — verifica que cada campo se resetee **después** del
  campo del que depende, o puede intentar preseleccionar una opción que su
  `Items` filtrado todavía no contiene.
- **Scroll vertical del contenedor derecho** — ~37 campos no entran en el
  alto de una pantalla Tableta. Dale al `Container` una altura fija y activa
  desbordamiento/scroll (la propiedad exacta depende de tu versión de Studio:
  contenedores de diseño automático suelen tener `Overflow`/scroll nativo; si
  el tuyo no, envuélvelo en un control de desplazamiento).
- **Controles Modern en evolución:** si al pegar una fórmula el estudio marca
  una propiedad como inexistente, revisa la página de ese control en
  Microsoft Learn (enlaces al final de 05-canvas-yaml-captura.md) — Microsoft
  los sigue actualizando y algún nombre puede haber cambiado.

### Mejoras opcionales (no bloquean la construcción)
- **Duplicar tablero** — un botón que hace `Collect(colTableros; {...los
  mismos campos del seleccionado..., Nombre: "Copia de " & ThisItem.Nombre})`;
  útil cuando un proyecto tiene varios tableros casi idénticos.
- **Reordenar tableros** (drag & drop o botones subir/bajar que intercambian
  `Orden` entre dos registros de `colTableros`).
- **`TabIndex`** explícito en los ~35 controles del panel derecho para
  navegación por teclado ordenada.

### Riesgo a tener en cuenta (no es un bug de la app, es inherente al diseño)
- **`colTableros` vive solo en memoria** hasta que se toca "Enviar solicitud".
  Si el usuario cierra la pestaña o pierde conexión a mitad de captura, se
  pierde todo lo no enviado — no hay borrador guardado. Si esto es un
  problema real para tu flujo de trabajo (formularios largos, interrupciones
  frecuentes), la solución sería guardar un borrador periódico en Dataverse
  (una `Solicitud` en estado provisional) en vez de mantener todo en memoria
  hasta el final — es un cambio de arquitectura más grande, coméntalo si
  quieres explorarlo.

---

## 7. Equivalencias con el formulario original

| Filament / Livewire original | Power Apps Canvas (maestro-detalle, Modern controls) |
|---|---|
| Wizard de 3 pasos | `scrContactoProyecto` → `scrTableroForm` (maestro-detalle) → `scrDocumentacion` → `scrConfirmacion`, con `ModernTabList` para saltar entre las 3 primeras |
| Modal "Agregar/Editar tablero" repetible | Galería izquierda (clásica) + formulario derecho (Modern) en la misma pantalla, gateados por `varEditIndex`; Editar/Eliminar arriba de la galería actúan sobre `galTableros.Selected` |
| `->visible(fn($get)=>...)` condicionales | propiedad `Visible` de cada control (ver 05-canvas-yaml-captura.md) |
| `recalculateCurrent()` | `lblCorriente.Text` (fórmula `With(...)`, ver 05) |
| Opciones de IP según interior/exterior | `Switch()` en `ddGradoIP.Items` (ver 05) |
| `->required()` | validación 3.4b + `Notify` |
| Notificaciones al enviar | Power Automate (flujo F-1) |
