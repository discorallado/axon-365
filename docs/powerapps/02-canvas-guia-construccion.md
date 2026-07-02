# Guía de construcción — Power App Canvas de captura (maestro-detalle)

App que reemplaza al `PublicFormWizard` de Filament, sobre **Dataverse**.
Arquitectura final: **3 pantallas navegadas por pestañas** (Contacto y
Proyecto / Tableros / Documentación, con una barra de pestañas compartida que
permite saltar directo a cualquiera) + una **pantalla de confirmación** fuera
del flujo de pestañas. Este documento es la referencia completa de la
**Pantalla 2 — Tableros** (`scrTableroForm`), construida como **maestro-detalle
en una sola pantalla**: galería de tableros a la izquierda + formulario
completo a la derecha.

> **Dónde está el resto:** la app completa (las 4 pantallas, la barra de
> pestañas compartida, y las Pantallas 1 y 3) se arma siguiendo
> [dataverse/05-construir-canvas-captura.md](../dataverse/05-construir-canvas-captura.md) —
> ese documento manda a **este** para el detalle de la Pantalla 2. Las
> fórmulas exactas de cada campo (`Items`, tipos, opciones) están en
> [05-canvas-yaml-captura.md](05-canvas-yaml-captura.md) — aquí se referencian,
> no se duplican.

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
tableros), igual que el wizard original. Una **colección local** de tableros
y una variable para saber si se está editando uno existente:

```powerfx
// App.OnStart
ClearCollect(colTableros; []);;  // tableros de la solicitud en curso
Set(varEditIndex; Blank())       // Blank = nuevo tablero; record = editando uno existente
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

`scrTableroForm` tiene **dos `Container`**: `cntGaleria` (izquierda, ~30% del
ancho) y `cntFormulario` (derecha, ~70%, con scroll vertical — son ~37 campos,
no entran en el alto de una pantalla Tableta). Arriba de todo va la barra de
pestañas compartida (Bloque 14b de
[dataverse/05](../dataverse/05-construir-canvas-captura.md)).

### 3.1 Contenedor izquierdo — galería de tableros

- **`galTableros`** (Galería vertical), `Items`: `colTableros`.
- Dentro de la plantilla:
  - **`lblNombreTablero`** (Texto), `Text`: `ThisItem.Nombre`
  - **`lblResumenTablero`** (Texto), `Text`: `"×" & ThisItem.Cantidad`
  - **Resaltar la fila seleccionada** — `Fill` del contenedor de la plantilla:
    ```powerfx
    If(ThisItem = galTableros.Selected; RGBA(99;102;241;0.15); RGBA(255;255;255;1))
    ```
    (si tu versión de Studio ya resalta la fila tocada de forma nativa, esta
    fórmula es opcional.)
- **`lblSinTableros`** (Texto) — "Aún no hay tableros en esta solicitud."
  `Visible`: `CountRows(colTableros) = 0`
- **`lblContadorTableros`** (Texto, opcional) — `Text`:
  `CountRows(colTableros) & " tablero(s) agregado(s)"`
- **`btnEditarTablero`** y **`btnEliminarTablero`** — van **arriba de la
  galería**, no dentro de cada fila; actúan sobre `galTableros.Selected` (la
  fila tocada). Deshabilitados si no hay ninguna seleccionada:
  ```powerfx
  // DisplayMode de ambos botones
  If(IsBlank(galTableros.Selected); DisplayMode.Disabled; DisplayMode.Edit)
  ```
  `btnEditarTablero.OnSelect` — ver 3.3. `btnEliminarTablero.OnSelect` — ver 3.7.
- **`btnAgregarTablero`** (Botón) — "+ Agregar Tablero", bajo la galería — ver 3.6.
- **`btnSiguienteTableros`** (Botón) — "Siguiente", junto al anterior. Valida
  que haya al menos un tablero antes de pasar a Documentación:
  ```powerfx
  // btnSiguienteTableros.OnSelect
  If(
      CountRows(colTableros) = 0;
      Notify("Agrega al menos un tablero antes de continuar."; NotificationType.Error);
      Navigate(scrDocumentacion; ScreenTransition.Cover)
  )
  ```

### 3.2 Contenedor derecho — indicador de modo

**`lblModoEdicion`** (Texto), arriba del formulario:
```powerfx
If(IsBlank(varEditIndex); "Nuevo tablero"; "Editando: " & varEditIndex.Nombre)
```
Sin esto es fácil perder de vista si estás agregando uno nuevo o modificando
uno ya guardado.

### 3.3 Por qué hace falta `Reset()` — y el `OnSelect` de "Editar"

`Default` de un control **se lee una sola vez**, al crearse o al llamar
`Reset(control)`. Cambiar `varEditIndex` **no** refresca los controles solos.
Por eso, cada vez que se carga un tablero distinto en el panel derecho (o se
limpia el formulario), hay que resetear los ~35 controles.

Tocar una fila de `galTableros` solo la **selecciona** (la resalta); **no**
carga sus datos en el panel derecho todavía — eso lo hace explícitamente
`btnEditarTablero`, para que seleccionar y hojear la lista no pise el tablero
que se está editando por accidente:

```powerfx
// btnEditarTablero.OnSelect
Set(varEditIndex; galTableros.Selected);;
Reset(ddTipoEntrega);; Reset(ddInstalacionNuevaReemplazo);; Reset(txtNombreTablero);;
Reset(txtCantidad);; Reset(ddTipoTablero);; Reset(txtOtroTipoTablero);;
Reset(txtFuncionTablero);; Reset(txtCargasAAlimentar);; Reset(txtNumeroCircuitos);;
Reset(ddUbicacion);; Reset(cmbAmbienteEspecial);; Reset(txtOtroAmbiente);;
Reset(ddGradoIP);; Reset(ddGradoIK);; Reset(ddTipoMontaje);; Reset(tglRestricciones);;
Reset(txtAltoMax);; Reset(txtAnchoMax);; Reset(txtFondoMax);; Reset(txtCondicionesInstalacion);;
Reset(ddTension);; Reset(txtOtraTension);; Reset(ddSistema);; Reset(txtOtroSistema);;
Reset(txtPotencia);; Reset(ddUnidadPotencia);; Reset(ddFrecuencia);; Reset(txtOtraFrecuencia);;
Reset(cmbProteccionesRequeridas);; Reset(cmbMarcasPreferidas);; Reset(ddMaterialGabinete);;
Reset(ddColorGabinete);; Reset(ddTipoVentilacion);; Reset(ddExpansionFutura);;
Reset(txtObservacionesTablero)
```

`lblCorriente` **no** entra en la lista — es un `Text` calculado, se
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

### 3.4 `Default` de cada control — carga los valores al editar

| Control | Tipo | `Default` |
|---|---|---|
| `txtNombreTablero` | Texto | `If(IsBlank(varEditIndex); ""; varEditIndex.Nombre)` |
| `txtCantidad` | Número | `If(IsBlank(varEditIndex); ""; Text(varEditIndex.Cantidad))` |
| `ddTipoEntrega` | DropDown | `If(IsBlank(varEditIndex); Blank(); Switch(varEditIndex.TipoEntrega; "tablero"; "Tablero Eléctrico"; "sala"; "Sala Eléctrica"; "Producto Eléctrico"))` |
| `ddInstalacionNuevaReemplazo` | DropDown | `If(IsBlank(varEditIndex); Blank(); Switch(varEditIndex.InstalacionNuevaReemplazo; "nueva"; "Instalación nueva"; "Reemplazo de existente"))` |
| `ddTipoTablero` | DropDown | `If(IsBlank(varEditIndex); Blank(); Switch(varEditIndex.TipoTablero; "fuerza";"Fuerza/Potencia"; "alumbrado";"Alumbrado/Distribución BT"; "control";"Control/Automatización"; "transfer";"Transferencia (ATS/MTS)"; "sincronizacion";"Sincronización de Generadores"; "remoto";"Distribución Remoto"; "pfcs";"Factor de Potencia"; "medicion";"Medición/Centro de Carga"; "variadores";"Variadores de Frecuencia"; "arrancadores";"Arrancadores Suaves"; "ups";"UPS/Respaldo"; "Otro"))` |
| `txtOtroTipoTablero` | Texto | `If(IsBlank(varEditIndex); ""; varEditIndex.OtroTipoTablero)` |
| `txtFuncionTablero` | Texto multilínea | `If(IsBlank(varEditIndex); ""; varEditIndex.FuncionTablero)` |
| `txtCargasAAlimentar` | Texto multilínea | `If(IsBlank(varEditIndex); ""; varEditIndex.CargasAAlimentar)` |
| `txtNumeroCircuitos` | Número | `If(IsBlank(varEditIndex); ""; Text(varEditIndex.NumeroCircuitos))` |
| `ddUbicacion` | DropDown | `If(IsBlank(varEditIndex); Blank(); Switch(varEditIndex.Ubicacion; "interior"; "Interior"; "Exterior"))` |
| `cmbAmbienteEspecial` | ComboBox multi | `DefaultSelectedItems`: `If(IsBlank(varEditIndex); Table(); varEditIndex.AmbienteEspecial)` |
| `txtOtroAmbiente` | Texto | `If(IsBlank(varEditIndex); ""; varEditIndex.OtroAmbienteEspecial)` |
| `ddGradoIP` | DropDown (array plano) | `If(IsBlank(varEditIndex); Blank(); varEditIndex.GradoIP)` |
| `ddGradoIK` | DropDown (array plano) | `If(IsBlank(varEditIndex); Blank(); varEditIndex.GradoIK)` |
| `ddTipoMontaje` | DropDown | `If(IsBlank(varEditIndex); Blank(); Switch(varEditIndex.TipoMontaje; "autosoportado";"Autosoportado"; "mural";"Mural"; "rack_19";"Rack 19"; "pedestal";"Pedestal"; "Otro"))` |
| `tglRestricciones` | Toggle | `If(IsBlank(varEditIndex); false; varEditIndex.RestriccionesDimension)` |
| `txtAltoMax` | Número | `If(IsBlank(varEditIndex); ""; Text(varEditIndex.AltoMaxMm))` |
| `txtAnchoMax` | Número | `If(IsBlank(varEditIndex); ""; Text(varEditIndex.AnchoMaxMm))` |
| `txtFondoMax` | Número | `If(IsBlank(varEditIndex); ""; Text(varEditIndex.FondoMaxMm))` |
| `txtCondicionesInstalacion` | Texto multilínea | `If(IsBlank(varEditIndex); ""; varEditIndex.CondicionesInstalacion)` |
| `ddTension` | DropDown | `If(IsBlank(varEditIndex); Blank(); Switch(varEditIndex.TensionSuministro; "220";"220 V"; "380";"380 V"; "400";"400 V"; "440";"440 V"; "480";"480 V"; "690";"690 V"; "1000";"1000 V"; "Otro"))` |
| `txtOtraTension` | Número | `If(IsBlank(varEditIndex); ""; Text(varEditIndex.OtraTension))` |
| `ddSistema` | DropDown | `If(IsBlank(varEditIndex); Blank(); Switch(varEditIndex.SistemaElectrico; "trifasico";"Trifásico"; "monofasico";"Monofásico"; "dc";"Corriente continua (DC)"; "Otro"))` |
| `txtOtroSistema` | Texto | `If(IsBlank(varEditIndex); ""; varEditIndex.OtroSistemaElectrico)` |
| `txtPotencia` | Número | `If(IsBlank(varEditIndex); ""; Text(varEditIndex.PotenciaEstimada))` |
| `ddUnidadPotencia` | DropDown | `If(IsBlank(varEditIndex); "kW"; Switch(varEditIndex.UnidadPotencia; "kW";"kW"; "kVA"))` |
| `ddFrecuencia` | DropDown | `If(IsBlank(varEditIndex); Blank(); Switch(varEditIndex.Frecuencia; "50";"50 Hz"; "60";"60 Hz"; "Otra"))` |
| `txtOtraFrecuencia` | Número | `If(IsBlank(varEditIndex); ""; Text(varEditIndex.OtraFrecuencia))` |
| `cmbProteccionesRequeridas` | ComboBox multi | `DefaultSelectedItems`: `If(IsBlank(varEditIndex); Table(); varEditIndex.ProteccionesRequeridas)` |
| `cmbMarcasPreferidas` | ComboBox multi | `DefaultSelectedItems`: `If(IsBlank(varEditIndex); Table(); varEditIndex.MarcasPreferidas)` |
| `ddMaterialGabinete` | DropDown | `If(IsBlank(varEditIndex); Blank(); Switch(varEditIndex.MaterialGabinete; "acero_pintado";"Acero pintado"; "acero_galvanizado";"Acero galvanizado"; "acero_inoxidable";"Acero inoxidable"; "acero_inox_316";"Acero inoxidable 316"; "fibra_vidrio";"Fibra de vidrio"; "poliester";"Poliéster"; "Aluminio"))` |
| `ddColorGabinete` | DropDown | `If(IsBlank(varEditIndex); "RAL 7035 (gris claro)"; Switch(varEditIndex.ColorGabinete; "7035";"RAL 7035 (gris claro)"; "7016";"RAL 7016 (gris antracita)"; "9016";"RAL 9016 (blanco tráfico)"; "9005";"RAL 9005 (negro)"; "5010";"RAL 5010 (azul)"; "6005";"RAL 6005 (verde)"; "Otro"))` |
| `ddTipoVentilacion` | DropDown | `If(IsBlank(varEditIndex); Blank(); Switch(varEditIndex.TipoVentilacion; "natural";"Natural"; "forzada";"Forzada (ventilador)"; "sellado";"Sellado (IP alto)"; "Climatizado (aire acondicionado)"))` |
| `ddExpansionFutura` | DropDown | `If(IsBlank(varEditIndex); Blank(); Switch(varEditIndex.ExpansionFutura; "no";"Sin expansión"; "10";"10%"; "20";"20%"; "30";"30%"; "Otro"))` |
| `txtObservacionesTablero` | Texto multilínea | `If(IsBlank(varEditIndex); ""; varEditIndex.ObservacionesTablero)` |

> Las etiquetas usadas en cada `Switch` son las mismas que sus `Items` en
> [05-canvas-yaml-captura.md](05-canvas-yaml-captura.md) — si cambias una
> etiqueta ahí, cambia también su par aquí.

Lógica condicional de `Visible` (qué campo aparece según otro) — sin cambios
respecto a lo ya definido en el `Items`/`Visible` de cada control en
[05-canvas-yaml-captura.md](05-canvas-yaml-captura.md) §"Pantalla del tablero".

### 3.5 Guardar tablero

```powerfx
// btnGuardarTablero.OnSelect — antes de esto corre la validación de 3.4b
With(
    {
        registro: {
            Nombre: txtNombreTablero.Text;
            Cantidad: Clamp(Value(txtCantidad.Text); 1; 999);
            Orden: If(IsBlank(varEditIndex); CountRows(colTableros); varEditIndex.Orden);
            TipoEntrega: ddTipoEntrega.Selected.Value;
            InstalacionNuevaReemplazo: ddInstalacionNuevaReemplazo.Selected.Value;
            TipoTablero: If(ddTipoEntrega.Selected.Value <> "tablero"; Blank(); ddTipoTablero.Selected.Value);
            OtroTipoTablero: If(ddTipoTablero.Selected.Value <> "otro"; Blank(); txtOtroTipoTablero.Text);
            FuncionTablero: txtFuncionTablero.Text;
            CargasAAlimentar: txtCargasAAlimentar.Text;
            NumeroCircuitos: Value(txtNumeroCircuitos.Text);
            Ubicacion: ddUbicacion.Selected.Value;
            AmbienteEspecial: cmbAmbienteEspecial.SelectedItems;
            OtroAmbienteEspecial: If(Not("otro" in cmbAmbienteEspecial.SelectedItems.Value); Blank(); txtOtroAmbiente.Text);
            GradoIP: ddGradoIP.Selected.Value;
            GradoIK: ddGradoIK.Selected.Value;
            TipoMontaje: ddTipoMontaje.Selected.Value;
            RestriccionesDimension: tglRestricciones.Value;
            AltoMaxMm: If(Not(tglRestricciones.Value); Blank(); Value(txtAltoMax.Text));
            AnchoMaxMm: If(Not(tglRestricciones.Value); Blank(); Value(txtAnchoMax.Text));
            FondoMaxMm: If(Not(tglRestricciones.Value); Blank(); Value(txtFondoMax.Text));
            CondicionesInstalacion: txtCondicionesInstalacion.Text;
            TensionSuministro: ddTension.Selected.Value;
            OtraTension: If(ddTension.Selected.Value <> "otro"; Blank(); Value(txtOtraTension.Text));
            SistemaElectrico: ddSistema.Selected.Value;
            OtroSistemaElectrico: If(ddSistema.Selected.Value <> "otro"; Blank(); txtOtroSistema.Text);
            PotenciaEstimada: Value(txtPotencia.Text);
            UnidadPotencia: ddUnidadPotencia.Selected.Value;
            CorrienteNominal: Value(lblCorriente.Text);
            Frecuencia: ddFrecuencia.Selected.Value;
            OtraFrecuencia: If(ddFrecuencia.Selected.Value <> "otro"; Blank(); Value(txtOtraFrecuencia.Text));
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
Reset(txtCantidad);; Reset(ddTipoTablero);; Reset(txtOtroTipoTablero);;
Reset(txtFuncionTablero);; Reset(txtCargasAAlimentar);; Reset(txtNumeroCircuitos);;
Reset(ddUbicacion);; Reset(cmbAmbienteEspecial);; Reset(txtOtroAmbiente);;
Reset(ddGradoIP);; Reset(ddGradoIK);; Reset(ddTipoMontaje);; Reset(tglRestricciones);;
Reset(txtAltoMax);; Reset(txtAnchoMax);; Reset(txtFondoMax);; Reset(txtCondicionesInstalacion);;
Reset(ddTension);; Reset(txtOtraTension);; Reset(ddSistema);; Reset(txtOtroSistema);;
Reset(txtPotencia);; Reset(ddUnidadPotencia);; Reset(ddFrecuencia);; Reset(txtOtraFrecuencia);;
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
        IsBlank(txtPotencia.Text);
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

### 3.6 Nuevo tablero

```powerfx
// btnNuevoTablero.OnSelect
Set(varEditIndex; Blank());;
Reset(ddTipoEntrega);; Reset(ddInstalacionNuevaReemplazo);; Reset(txtNombreTablero);;
Reset(txtCantidad);; Reset(ddTipoTablero);; Reset(txtOtroTipoTablero);;
Reset(txtFuncionTablero);; Reset(txtCargasAAlimentar);; Reset(txtNumeroCircuitos);;
Reset(ddUbicacion);; Reset(cmbAmbienteEspecial);; Reset(txtOtroAmbiente);;
Reset(ddGradoIP);; Reset(ddGradoIK);; Reset(ddTipoMontaje);; Reset(tglRestricciones);;
Reset(txtAltoMax);; Reset(txtAnchoMax);; Reset(txtFondoMax);; Reset(txtCondicionesInstalacion);;
Reset(ddTension);; Reset(txtOtraTension);; Reset(ddSistema);; Reset(txtOtroSistema);;
Reset(txtPotencia);; Reset(ddUnidadPotencia);; Reset(ddFrecuencia);; Reset(txtOtraFrecuencia);;
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
// btnEliminarTablero.OnSelect
UpdateContext({locItemAEliminar: galTableros.Selected; mostrarConfirmarEliminar: true})
```

Y un **`Group`/popup** `grpConfirmarEliminar` (oculto por defecto, `Visible:
mostrarConfirmarEliminar`) con texto "¿Eliminar este tablero?" y dos botones:

```powerfx
// btnConfirmarSi.OnSelect
UpdateContext({mostrarConfirmarEliminar: false});;
Remove(colTableros; locItemAEliminar);;
If(
    locItemAEliminar = varEditIndex;
    Set(varEditIndex; Blank());;
    Reset(ddTipoEntrega);; Reset(ddInstalacionNuevaReemplazo);; Reset(txtNombreTablero);;
    Reset(txtCantidad);; Reset(ddTipoTablero);; Reset(txtOtroTipoTablero);;
    Reset(txtFuncionTablero);; Reset(txtCargasAAlimentar);; Reset(txtNumeroCircuitos);;
    Reset(ddUbicacion);; Reset(cmbAmbienteEspecial);; Reset(txtOtroAmbiente);;
    Reset(ddGradoIP);; Reset(ddGradoIK);; Reset(ddTipoMontaje);; Reset(tglRestricciones);;
    Reset(txtAltoMax);; Reset(txtAnchoMax);; Reset(txtFondoMax);; Reset(txtCondicionesInstalacion);;
    Reset(ddTension);; Reset(txtOtraTension);; Reset(ddSistema);; Reset(txtOtroSistema);;
    Reset(txtPotencia);; Reset(ddUnidadPotencia);; Reset(ddFrecuencia);; Reset(txtOtraFrecuencia);;
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
- Botón **Enviar solicitud** — fórmula completa (validación de "al menos un
  tablero", `Patch` de la cabecera, `ForAll`/`Patch` de cada tablero con el
  mapeo Choice → Dataverse, y navegación a confirmación) en la sección
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
// lblConfirmacion.Text
"Solicitud enviada correctamente. Código: " & varSolicitud.ReferenceCode
```
Botón **"Registrar otra solicitud"** → `Navigate(scrContactoProyecto)`. Esta
pantalla **no** lleva la barra de pestañas — es un estado terminal fuera del
flujo de las 3 secciones.

---

## 6. Qué más podrías necesitar

Revisando este diseño (galería + formulario en una pantalla, ~37 campos por
tablero), esto es lo que conviene resolver antes de darlo por terminado:

### Ya incorporado en esta guía
- **Validación de campos obligatorios** antes de guardar un tablero (3.4b) —
  sin esto se pueden guardar tableros vacíos.
- **Confirmación antes de eliminar** (3.7) — un ícono de basurero sin
  confirmación es fácil de tocar por error en una lista larga.
- **Indicador de modo** "Nuevo tablero" / "Editando: X" (3.2) — sin esto no
  se sabe si se está creando o modificando algo.
- **Corrección de `Orden`** al editar (no se recalcula, conserva el original).
- **Limpieza de campos "Otro"/condicionales** que quedaron ocultos pero con
  datos viejos (3.5) — si no, se guardan datos huérfanos e inconsistentes con
  el resto del registro.

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
- **Rangos numéricos adicionales** — ya se acotó `Cantidad` con `Clamp(...;
  1; 999)`; considera lo mismo para `NumeroCircuitos` (≥0) y validar
  `PotenciaEstimada > 0` en 3.4b, no solo que no esté vacío.

### Mejoras opcionales (no bloquean la construcción)
- **Colecciones de opciones reutilizables:** en vez de repetir cada par
  código/etiqueta en el `Items` del dropdown y de nuevo en su `Switch` de
  `Default` (como en la tabla de 3.4), define cada lista como colección en
  `App.OnStart` (`ClearCollect(colOpcTipoEntrega; Table(...))`) y usa
  `Items: colOpcTipoEntrega` + `Default: LookUp(colOpcTipoEntrega; Value =
  varEditIndex.TipoEntrega).Label`. Evita mantener el mismo texto en dos
  lugares. Dilo si quieres que lo reescriba así.
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

| Filament / Livewire original | Power Apps Canvas (maestro-detalle) |
|---|---|
| Wizard de 3 pasos | `scrContactoProyecto` → `scrTableroForm` (maestro-detalle) → `scrDocumentacion` → `scrConfirmacion`, con barra de pestañas para saltar entre las 3 primeras |
| Modal "Agregar/Editar tablero" repetible | Galería izquierda + formulario derecho en la misma pantalla, gateados por `varEditIndex`; Editar/Eliminar arriba de la galería actúan sobre `galTableros.Selected` |
| `->visible(fn($get)=>...)` condicionales | propiedad `Visible` de cada control (ver 05-canvas-yaml-captura.md) |
| `recalculateCurrent()` | `lblCorriente.Text` (fórmula `With(...)`, ver 05) |
| Opciones de IP según interior/exterior | `Switch()` en `ddGradoIP.Items` (ver 05) |
| `->required()` | validación 3.4b + `Notify` |
| Notificaciones al enviar | Power Automate (flujo F-1) |
