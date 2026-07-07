# Sub-formulario del tablero en varias pantallas (sub-pestañas)

Rediseño de la **Pantalla 2 — Tableros**. Antes era **maestro-detalle en una
sola pantalla** (galería izquierda + los ~35 campos del tablero a la derecha en
un contenedor con scroll, ver [02-canvas-guia-construccion.md](02-canvas-guia-construccion.md)).
Aquí se separa el **sub-formulario del tablero en 4 pantallas reales**, una por
etapa, navegadas por una **barra de sub-pestañas** (`ModernTabList` propio del
sub-flujo). Al **Guardar** (o **Cancelar**) se vuelve a la **pantalla de
galería**.

> Este documento **reemplaza** el diseño de una sola pantalla de
> [02-canvas-guia-construccion.md](02-canvas-guia-construccion.md) §3 para la
> captura del tablero. La galería, el modelo en memoria (`colTableros`,
> `varEditIndex`), las colecciones `colOpc*`, la validación, el `Reset` y el
> `Patch` de envío **no cambian** — solo se reparte el formulario en pantallas
> y se agrega la navegación por sub-pestañas. Las fórmulas de `Items`,
> opciones y el envío a Dataverse siguen en
> [05-canvas-yaml-captura.md](05-canvas-yaml-captura.md).

---

## Arquitectura nueva

```
Barra de pestañas principal:  [ Contacto y Proyecto ] [ Tableros ] [ Documentación ]
                                                          │
                                                          ▼
                              scrTableros  ── galería de tableros (maestro)
                              [+ Agregar]  [Editar]  [Eliminar]  [Siguiente ▶]
                                                          │  Agregar / Editar
                                                          ▼
        ┌──────────────── SUB-FLUJO del tablero (sub-pestañas) ────────────────┐
        │  [ 1. Identificación ] [ 2. Ubicación ] [ 3. Eléctrico ] [ 4. Constr.] │
        ├───────────────────────────────────────────────────────────────────────┤
        │  scrTableroPaso1 → scrTableroPaso2 → scrTableroPaso3 → scrTableroPaso4  │
        │  [Cancelar]                        [◀ Atrás]  [Siguiente ▶] / [Guardar] │
        └───────────────────────────────────────────────────────────────────────┘
                                                          │  Guardar / Cancelar
                                                          ▼
                                                    vuelve a scrTableros
```

**Pantallas:**

| Pantalla | Rol | Contenido |
|---|---|---|
| `scrTableros` | Galería (maestro). Tiene la **barra de pestañas principal**. | Galería `galTableros`, Agregar/Editar/Eliminar, Siguiente a Documentación. |
| `scrTableroPaso1` | Sub-formulario etapa 1 — **Identificación** | Tipo de entrega, instalación, nombre, cantidad, tipo de tablero, función, cargas, circuitos. |
| `scrTableroPaso2` | Sub-formulario etapa 2 — **Ubicación, ambiente y montaje** | Ubicación, ambiente especial, IP/IK, montaje, restricciones de dimensión. |
| `scrTableroPaso3` | Sub-formulario etapa 3 — **Parámetros eléctricos** | Tensión, sistema, potencia, corriente calculada, frecuencia, protecciones, marcas. |
| `scrTableroPaso4` | Sub-formulario etapa 4 — **Diseño constructivo** + **Guardar** | Material, color, ventilación, expansión, observaciones, botón Guardar tablero. |

- Las **4 pantallas del sub-flujo llevan la barra de sub-pestañas**
  (`navSubPasos`), **no** la barra principal — estás "dentro" de un tablero, no
  saltando entre secciones de la solicitud.
- El **valor de cada control se conserva al navegar** entre pantallas dentro de
  la misma sesión, así que repartir los 35 campos en 4 pantallas no rompe ni el
  Guardar ni la validación: un control en `scrTableroPaso4` puede leer
  `txtNombreTablero.Text` de `scrTableroPaso1` sin problema (los controles se
  referencian por nombre, no por pantalla).
- **Volver a la galería:** el botón **Guardar tablero** (Paso 4) y el botón
  **Cancelar** (las 4 pantallas) terminan en `Navigate(scrTableros)`.

> **Renombre respecto al diseño anterior:** la pantalla maestro-detalle
> `scrTableroForm` se divide en `scrTableros` (galería) + `scrTableroPaso1..4`
> (sub-formulario). Si ya construiste `scrTableroForm`, puedes reutilizar sus
> controles moviéndolos a las nuevas pantallas por etapa.

---

## 0. Colección e inicialización — sin cambios

`App.OnStart` es idéntico: `varEditIndex`, `colTableros` y las ~17 colecciones
`colOpc*`. Ver [05-canvas-yaml-captura.md](05-canvas-yaml-captura.md#app--colección-e-inicialización).

---

## 1. Pantalla de galería (`scrTableros`)

Es la pantalla a la que se **vuelve** al terminar. Lleva la **barra de pestañas
principal** (Bloque 14b de [dataverse/05](../dataverse/05-construir-canvas-captura.md)).

```yaml
Screens:
  scrTableros:
    Properties:
      Fill: =RGBA(245; 246; 248; 1)
    Children:
      # --- Barra de pestañas principal (Contacto / Tableros / Documentación) ---
      # navPestanas: insertar según Bloque 14b de dataverse/05, con Default = "Tableros".

      - lblTituloTableros:
          Control: ModernText@1.0.0
          Properties:
            Text: ="Tableros de la solicitud"
            X: =48
            Y: =88
            Width: =Parent.Width - 96
            Size: =20
            FontWeight: =FontWeight.Semibold

      - lblContadorTableros:
          Control: ModernText@1.0.0
          Properties:
            Text: =CountRows(colTableros) & " tablero(s) agregado(s)"
            X: =48
            Y: =128
            Width: =600
            Size: =11
            Color: =RGBA(100; 116; 139; 1)

      # --- Acciones sobre la galería ---
      - btnAgregarTablero:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="+ Agregar tablero"
            Appearance: =ButtonAppearance.Primary
            X: =48
            Y: =168
            Width: =200
            # Abre el sub-flujo en modo "nuevo": limpia varEditIndex, resetea los
            # 35 controles (repartidos en las 4 pantallas) y navega al Paso 1.
            OnSelect: |
              =Set(varEditIndex; Blank());;
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
              Reset(txtObservacionesTablero);;
              Navigate(scrTableroPaso1; ScreenTransition.Cover)

      - btnEditarTablero:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Editar"
            X: =260
            Y: =168
            Width: =120
            DisplayMode: =If(IsBlank(galTableros.Selected); DisplayMode.Disabled; DisplayMode.Edit)
            # Carga la fila seleccionada en modo "edición": setea varEditIndex,
            # resetea los 35 controles (cada Default vuelve a leer varEditIndex) y
            # navega al Paso 1.
            OnSelect: |
              =Set(varEditIndex; galTableros.Selected);;
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
              Reset(txtObservacionesTablero);;
              Navigate(scrTableroPaso1; ScreenTransition.Cover)

      - btnEliminarTablero:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Eliminar"
            Appearance: =ButtonAppearance.Subtle
            X: =392
            Y: =168
            Width: =120
            DisplayMode: =If(IsBlank(galTableros.Selected); DisplayMode.Disabled; DisplayMode.Edit)
            OnSelect: =UpdateContext({locItemAEliminar: galTableros.Selected; mostrarConfirmarEliminar: true})

      # --- Galería de tableros (clásica; no hay Gallery Modern) ---
      - galTableros:
          Control: Gallery@2.15.0   # galería vertical clásica
          Properties:
            Items: =colTableros
            X: =48
            Y: =220
            Width: =Parent.Width - 96
            Height: =Parent.Height - 300
          Children:
            - lblNombreTablero:
                Control: ModernText@1.0.0
                Properties:
                  Text: =ThisItem.Nombre
                  Size: =14
                  FontWeight: =FontWeight.Semibold
            - lblResumenTablero:
                Control: ModernText@1.0.0
                Properties:
                  Text: ="×" & ThisItem.Cantidad
                  Size: =11
                  Color: =RGBA(100; 116; 139; 1)

      - lblSinTableros:
          Control: ModernText@1.0.0
          Properties:
            Text: ="Aún no hay tableros en esta solicitud."
            Visible: =CountRows(colTableros) = 0
            X: =48
            Y: =260
            Width: =600
            Color: =RGBA(100; 116; 139; 1)

      # --- Siguiente: valida ≥1 tablero y pasa a Documentación ---
      - btnSiguienteTableros:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Siguiente ▶"
            Appearance: =ButtonAppearance.Primary
            X: =Parent.Width - 248
            Y: =168
            Width: =200
            OnSelect: |
              =If(
                CountRows(colTableros) = 0;
                Notify("Agrega al menos un tablero antes de continuar."; NotificationType.Error);
                Navigate(scrDocumentacion; ScreenTransition.Cover)
              )

      # --- Popup de confirmación de borrado (Group clásico, oculto por defecto) ---
      # grpConfirmarEliminar (Visible: =mostrarConfirmarEliminar) con:
      #   btnConfirmarSi.OnSelect:
      #     =UpdateContext({mostrarConfirmarEliminar: false});;
      #      Remove(colTableros; locItemAEliminar)
      #     (si borras el que estabas editando, además Set(varEditIndex; Blank());; y el bloque de Reset;
      #      con este flujo, al editar estás en el sub-formulario, no en la galería, así que rara vez aplica.)
      #   btnConfirmarNo.OnSelect:
      #     =UpdateContext({mostrarConfirmarEliminar: false})
```

> `locItemAEliminar` se captura **antes** del `Remove` — al borrar, la galería
> se re-renderiza y `galTableros.Selected` puede quedar en blanco (ver
> [02-canvas-guia-construccion.md](02-canvas-guia-construccion.md) §3.7).

---

## 2. Barra de sub-pestañas (`navSubPasos`) — una copia por pantalla del sub-flujo

Se inserta **una copia en cada una de las 4 pantallas** del sub-flujo
(`scrTableroPaso1..4`). Lo único que cambia entre copias es el `Default` (qué
sub-pestaña aparece marcada al entrar).

1. **Insertar → Modern → Tabs o tab list** → nómbralo `navSubPasos`.
2. **`Items`** (igual en las 4 copias):
   ```
   ["1. Identificación"; "2. Ubicación y montaje"; "3. Eléctrico"; "4. Constructivo"]
   ```
3. **`Default`** — distinto en cada pantalla:
   - En `scrTableroPaso1`: `"1. Identificación"`
   - En `scrTableroPaso2`: `"2. Ubicación y montaje"`
   - En `scrTableroPaso3`: `"3. Eléctrico"`
   - En `scrTableroPaso4`: `"4. Constructivo"`
4. **`OnChange`** (igual en las 4 copias):
   ```
   Switch(Self.Selected.Value;
     "1. Identificación";      Navigate(scrTableroPaso1; ScreenTransition.None);
     "2. Ubicación y montaje"; Navigate(scrTableroPaso2; ScreenTransition.None);
     "3. Eléctrico";           Navigate(scrTableroPaso3; ScreenTransition.None);
     Navigate(scrTableroPaso4; ScreenTransition.None)
   )
   ```
5. (Opcional) `Appearance: =TabListAppearance.Underline`.

> **Las sub-pestañas navegan libre, sin validar** — igual que la barra
> principal (sirven para revisar/corregir cualquier etapa sin bloqueos). El
> valor de los controles se conserva al saltar entre sub-pestañas, así que
> puedes ir y volver sin perder lo escrito.
>
> **Validación en dos niveles:** cada botón **Siguiente ▶** valida los
> obligatorios de **su** etapa antes de avanzar (avisa temprano, etapa por
> etapa — ver §3, §4, §5); y el botón **Guardar tablero** (Paso 4) revalida
> **todo** el tablero como red de seguridad final, porque las sub-pestañas
> permiten saltarse los Siguiente. Los obligatorios del Paso 4 (material,
> ventilación, expansión) se validan solo en el Guardar, que es su etapa.

---

## 3. `scrTableroPaso1` — Identificación

`Default`/`Visible` de cada control salen de
[02-canvas-guia-construccion.md](02-canvas-guia-construccion.md) §3.4 y de
[05-canvas-yaml-captura.md](05-canvas-yaml-captura.md); aquí van **inline** para
que la pantalla sea copiar-pegar directo.

```yaml
  scrTableroPaso1:
    Properties:
      Fill: =RGBA(245; 246; 248; 1)
    Children:
      # navSubPasos — insertar según §2, Default = "1. Identificación", en X:=48 Y:=64 Width:=Parent.Width-96

      - lblTituloPaso1:
          Control: ModernText@1.0.0
          Properties:
            Text: ="Identificación del tablero"
            X: =48
            Y: =24
            Width: =Parent.Width - 300
            Size: =18
            FontWeight: =FontWeight.Semibold
      - lblModoEdicion:
          Control: ModernText@1.0.0
          Properties:
            Text: =If(IsBlank(varEditIndex); "Nuevo tablero"; "Editando: " & varEditIndex.Nombre)
            X: =Parent.Width - 360
            Y: =28
            Width: =312
            Align: =Align.Right
            Color: =RGBA(99; 102; 241; 1)

      - ddTipoEntrega:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcTipoEntrega
            ItemDisplayText: =ThisItem.Label
            Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTipoEntrega; Value = varEditIndex.TipoEntrega))
            X: =48
            Y: =150
            Width: =560
      - ddInstalacionNuevaReemplazo:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcInstalacionNuevaReemplazo
            ItemDisplayText: =ThisItem.Label
            Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcInstalacionNuevaReemplazo; Value = varEditIndex.InstalacionNuevaReemplazo))
            X: =648
            Y: =150
            Width: =560

      - txtNombreTablero:
          Control: ModernTextInput@1.0.0
          Properties:
            Placeholder: ="Nombre del tablero * (Ej.: TG Principal)"
            Default: =If(IsBlank(varEditIndex); ""; varEditIndex.Nombre)
            X: =48
            Y: =230
            Width: =560
      - numCantidad:
          Control: ModernNumberInput@1.0.0
          Properties:
            HintText: ="Cantidad"
            Default: =If(IsBlank(varEditIndex); 1; varEditIndex.Cantidad)
            Min: =1
            Max: =999
            Precision: =DecimalPrecision.'0'
            X: =648
            Y: =230
            Width: =560

      # Tipo de tablero: visible solo si TipoEntrega = "tablero"
      - ddTipoTablero:
          Control: ModernDropdown@1.0.0
          Properties:
            Visible: =ddTipoEntrega.Selected.Value = "tablero"
            Items: =colOpcTipoTablero
            ItemDisplayText: =ThisItem.Label
            Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTipoTablero; Value = varEditIndex.TipoTablero))
            X: =48
            Y: =310
            Width: =560
      # Otro tipo de tablero: visible solo si TipoTablero = "otro"
      - txtOtroTipoTablero:
          Control: ModernTextInput@1.0.0
          Properties:
            Visible: =ddTipoTablero.Selected.Value = "otro"
            Placeholder: ="Especifica el tipo de tablero"
            Default: =If(IsBlank(varEditIndex); ""; varEditIndex.OtroTipoTablero)
            X: =648
            Y: =310
            Width: =560

      - txtFuncionTablero:
          Control: ModernTextInput@1.0.0
          Properties:
            Type: =TextInputType.Multiline
            Placeholder: ="Función del tablero"
            Default: =If(IsBlank(varEditIndex); ""; varEditIndex.FuncionTablero)
            X: =48
            Y: =390
            Width: =560
            Height: =110
      - txtCargasAAlimentar:
          Control: ModernTextInput@1.0.0
          Properties:
            Type: =TextInputType.Multiline
            Placeholder: ="Cargas a alimentar"
            Default: =If(IsBlank(varEditIndex); ""; varEditIndex.CargasAAlimentar)
            X: =648
            Y: =390
            Width: =560
            Height: =110

      - numNumeroCircuitos:
          Control: ModernNumberInput@1.0.0
          Properties:
            HintText: ="Número de circuitos"
            Default: =If(IsBlank(varEditIndex); 0; varEditIndex.NumeroCircuitos)
            Min: =0
            Precision: =DecimalPrecision.'0'
            X: =48
            Y: =520
            Width: =560

      # --- Barra de navegación inferior ---
      - btnCancelarP1:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Cancelar"
            Appearance: =ButtonAppearance.Subtle
            X: =48
            Y: =Parent.Height - 88
            Width: =160
            OnSelect: =Set(varEditIndex; Blank());; Navigate(scrTableros; ScreenTransition.UnCover)
      - btnSiguienteP1:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Siguiente ▶"
            Appearance: =ButtonAppearance.Primary
            X: =Parent.Width - 248
            Y: =Parent.Height - 88
            Width: =200
            # Valida los obligatorios de esta etapa antes de avanzar.
            OnSelect: |
              =If(
                Or(
                  IsBlank(txtNombreTablero.Text);
                  IsBlank(ddTipoEntrega.Selected.Value);
                  IsBlank(ddInstalacionNuevaReemplazo.Selected.Value);
                  And(ddTipoEntrega.Selected.Value = "tablero"; IsBlank(ddTipoTablero.Selected.Value))
                );
                Notify("Completa los campos obligatorios de Identificación."; NotificationType.Error);
                Navigate(scrTableroPaso2; ScreenTransition.None)
              )
```

---

## 4. `scrTableroPaso2` — Ubicación, ambiente y montaje

```yaml
  scrTableroPaso2:
    Properties:
      Fill: =RGBA(245; 246; 248; 1)
    Children:
      # navSubPasos — Default = "2. Ubicación y montaje"
      - lblTituloPaso2:
          Control: ModernText@1.0.0
          Properties:
            Text: ="Ubicación, ambiente y montaje"
            X: =48
            Y: =24
            Width: =Parent.Width - 96
            Size: =18
            FontWeight: =FontWeight.Semibold

      - ddUbicacion:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcUbicacion
            ItemDisplayText: =ThisItem.Label
            Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcUbicacion; Value = varEditIndex.Ubicacion))
            X: =48
            Y: =150
            Width: =560
      - cmbAmbienteEspecial:
          Control: ModernCombobox@1.0.0
          Properties:
            Items: =colOpcAmbienteEspecial
            ItemDisplayText: =ThisItem.Label
            SelectMultiple: =true
            DefaultSelectedItems: =If(IsBlank(varEditIndex); Table(); varEditIndex.AmbienteEspecial)
            X: =648
            Y: =150
            Width: =560
      - txtOtroAmbiente:
          Control: ModernTextInput@1.0.0
          Properties:
            Visible: ="otro" in cmbAmbienteEspecial.SelectedItems.Value
            Placeholder: ="Especifica el ambiente especial"
            Default: =If(IsBlank(varEditIndex); ""; varEditIndex.OtroAmbienteEspecial)
            X: =648
            Y: =230
            Width: =560

      # Grado IP: subconjunto según interior/exterior. Default en {Value:...} porque
      # su Items es array de texto plano (sin colección colOpc*).
      - ddGradoIP:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: |
              =Switch(ddUbicacion.Selected.Value;
                "interior"; ["IP20";"IP31";"IP43";"IP54";"IP55";"IP65"];
                "exterior"; ["IP54";"IP55";"IP65";"IP66";"IP67";"IP68"];
                ["IP20";"IP31";"IP43";"IP54";"IP55";"IP65";"IP66";"IP67";"IP68"]
              )
            Default: =If(IsBlank(varEditIndex); Blank(); {Value: varEditIndex.GradoIP})
            X: =48
            Y: =310
            Width: =270
      - ddGradoIK:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =["IK06";"IK07";"IK08";"IK09";"IK10"]
            Default: =If(IsBlank(varEditIndex); Blank(); {Value: varEditIndex.GradoIK})
            X: =338
            Y: =310
            Width: =270
      - ddTipoMontaje:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcTipoMontaje
            ItemDisplayText: =ThisItem.Label
            Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTipoMontaje; Value = varEditIndex.TipoMontaje))
            X: =648
            Y: =310
            Width: =560

      - tglRestricciones:
          Control: ModernToggle@1.0.0
          Properties:
            Label: ="¿Tiene restricciones de dimensión?"
            Checked: =If(IsBlank(varEditIndex); false; varEditIndex.RestriccionesDimension)
            X: =48
            Y: =390
            Width: =560
      - numAltoMax:
          Control: ModernNumberInput@1.0.0
          Properties:
            Visible: =tglRestricciones.Checked
            HintText: ="Alto máximo (mm)"
            Default: =If(IsBlank(varEditIndex); 0; varEditIndex.AltoMaxMm)
            Min: =0
            Precision: =DecimalPrecision.'0'
            X: =48
            Y: =440
            Width: =180
      - numAnchoMax:
          Control: ModernNumberInput@1.0.0
          Properties:
            Visible: =tglRestricciones.Checked
            HintText: ="Ancho máximo (mm)"
            Default: =If(IsBlank(varEditIndex); 0; varEditIndex.AnchoMaxMm)
            Min: =0
            Precision: =DecimalPrecision.'0'
            X: =238
            Y: =440
            Width: =180
      - numFondoMax:
          Control: ModernNumberInput@1.0.0
          Properties:
            Visible: =tglRestricciones.Checked
            HintText: ="Fondo máximo (mm)"
            Default: =If(IsBlank(varEditIndex); 0; varEditIndex.FondoMaxMm)
            Min: =0
            Precision: =DecimalPrecision.'0'
            X: =428
            Y: =440
            Width: =180

      - txtCondicionesInstalacion:
          Control: ModernTextInput@1.0.0
          Properties:
            Type: =TextInputType.Multiline
            Placeholder: ="Condiciones adicionales de instalación"
            Default: =If(IsBlank(varEditIndex); ""; varEditIndex.CondicionesInstalacion)
            X: =648
            Y: =390
            Width: =560
            Height: =110

      # --- Navegación inferior ---
      - btnCancelarP2:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Cancelar"
            Appearance: =ButtonAppearance.Subtle
            X: =48
            Y: =Parent.Height - 88
            Width: =160
            OnSelect: =Set(varEditIndex; Blank());; Navigate(scrTableros; ScreenTransition.UnCover)
      - btnAtrasP2:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="◀ Atrás"
            Appearance: =ButtonAppearance.Secondary
            X: =Parent.Width - 468
            Y: =Parent.Height - 88
            Width: =200
            OnSelect: =Navigate(scrTableroPaso1; ScreenTransition.None)
      - btnSiguienteP2:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Siguiente ▶"
            Appearance: =ButtonAppearance.Primary
            X: =Parent.Width - 248
            Y: =Parent.Height - 88
            Width: =200
            # Valida los obligatorios de esta etapa antes de avanzar.
            OnSelect: |
              =If(
                Or(
                  IsBlank(ddUbicacion.Selected.Value);
                  IsBlank(ddGradoIP.Selected.Value);
                  IsBlank(ddTipoMontaje.Selected.Value)
                );
                Notify("Completa los campos obligatorios de Ubicación y montaje."; NotificationType.Error);
                Navigate(scrTableroPaso3; ScreenTransition.None)
              )
```

---

## 5. `scrTableroPaso3` — Parámetros eléctricos

```yaml
  scrTableroPaso3:
    Properties:
      Fill: =RGBA(245; 246; 248; 1)
    Children:
      # navSubPasos — Default = "3. Eléctrico"
      - lblTituloPaso3:
          Control: ModernText@1.0.0
          Properties:
            Text: ="Parámetros eléctricos"
            X: =48
            Y: =24
            Width: =Parent.Width - 96
            Size: =18
            FontWeight: =FontWeight.Semibold

      - ddTension:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcTension
            ItemDisplayText: =ThisItem.Label
            Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTension; Value = varEditIndex.TensionSuministro))
            X: =48
            Y: =150
            Width: =270
      - numOtraTension:
          Control: ModernNumberInput@1.0.0
          Properties:
            Visible: =ddTension.Selected.Value = "otro"
            HintText: ="Tensión (V)"
            Default: =If(IsBlank(varEditIndex); 0; varEditIndex.OtraTension)
            Min: =0
            X: =338
            Y: =150
            Width: =270
      - ddSistema:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcSistema
            ItemDisplayText: =ThisItem.Label
            Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcSistema; Value = varEditIndex.SistemaElectrico))
            X: =648
            Y: =150
            Width: =270
      - txtOtroSistema:
          Control: ModernTextInput@1.0.0
          Properties:
            Visible: =ddSistema.Selected.Value = "otro"
            Placeholder: ="Especifica el sistema eléctrico"
            Default: =If(IsBlank(varEditIndex); ""; varEditIndex.OtroSistemaElectrico)
            X: =938
            Y: =150
            Width: =270

      - numPotencia:
          Control: ModernNumberInput@1.0.0
          Properties:
            HintText: ="Potencia estimada"
            Default: =If(IsBlank(varEditIndex); Blank(); varEditIndex.PotenciaEstimada)
            Min: =0
            Precision: =DecimalPrecision.'2'
            X: =48
            Y: =230
            Width: =270
      - ddUnidadPotencia:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcUnidadPotencia
            ItemDisplayText: =ThisItem.Label
            Default: =If(IsBlank(varEditIndex); LookUp(colOpcUnidadPotencia; Value="kW"); LookUp(colOpcUnidadPotencia; Value = varEditIndex.UnidadPotencia))
            X: =338
            Y: =230
            Width: =270
      - lblCorriente:
          Control: ModernText@1.0.0
          Properties:
            Text: |
              =With(
                {
                  watts: numPotencia.Value * 1000;
                  v: If(ddTension.Selected.Value="otro"; numOtraTension.Value; Value(ddTension.Selected.Value))
                };
                If(Or(watts<=0; v<=0); "";
                  "Corriente nominal ≈ " & Text(Round(
                    Switch(ddSistema.Selected.Value;
                      "trifasico"; watts/(Sqrt(3)*v);
                      "monofasico"; watts/v;
                      "dc"; watts/v;
                      Blank()
                    ); 1)
                  ) & " A"
                )
              )
            X: =648
            Y: =240
            Width: =560
            Size: =12
            Color: =RGBA(30; 41; 59; 1)

      - ddFrecuencia:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcFrecuencia
            ItemDisplayText: =ThisItem.Label
            Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcFrecuencia; Value = varEditIndex.Frecuencia))
            X: =48
            Y: =310
            Width: =270
      - numOtraFrecuencia:
          Control: ModernNumberInput@1.0.0
          Properties:
            Visible: =ddFrecuencia.Selected.Value = "otro"
            HintText: ="Frecuencia (Hz)"
            Default: =If(IsBlank(varEditIndex); 0; varEditIndex.OtraFrecuencia)
            Min: =0
            X: =338
            Y: =310
            Width: =270

      - cmbProteccionesRequeridas:
          Control: ModernCombobox@1.0.0
          Properties:
            Items: =colOpcProteccionesRequeridas
            ItemDisplayText: =ThisItem.Label
            SelectMultiple: =true
            DefaultSelectedItems: =If(IsBlank(varEditIndex); Table(); varEditIndex.ProteccionesRequeridas)
            X: =48
            Y: =390
            Width: =560
      - cmbMarcasPreferidas:
          Control: ModernCombobox@1.0.0
          Properties:
            Items: =colOpcMarcasPreferidas
            ItemDisplayText: =ThisItem.Label
            SelectMultiple: =true
            DefaultSelectedItems: =If(IsBlank(varEditIndex); Table(); varEditIndex.MarcasPreferidas)
            X: =648
            Y: =390
            Width: =560

      # --- Navegación inferior ---
      - btnCancelarP3:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Cancelar"
            Appearance: =ButtonAppearance.Subtle
            X: =48
            Y: =Parent.Height - 88
            Width: =160
            OnSelect: =Set(varEditIndex; Blank());; Navigate(scrTableros; ScreenTransition.UnCover)
      - btnAtrasP3:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="◀ Atrás"
            Appearance: =ButtonAppearance.Secondary
            X: =Parent.Width - 468
            Y: =Parent.Height - 88
            Width: =200
            OnSelect: =Navigate(scrTableroPaso2; ScreenTransition.None)
      - btnSiguienteP3:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Siguiente ▶"
            Appearance: =ButtonAppearance.Primary
            X: =Parent.Width - 248
            Y: =Parent.Height - 88
            Width: =200
            # Valida los obligatorios de esta etapa antes de avanzar.
            OnSelect: |
              =If(
                Or(
                  IsBlank(ddTension.Selected.Value);
                  IsBlank(ddSistema.Selected.Value);
                  IsBlank(numPotencia.Value) || numPotencia.Value <= 0;
                  IsBlank(ddFrecuencia.Selected.Value);
                  CountRows(cmbProteccionesRequeridas.SelectedItems) = 0
                );
                Notify("Completa los campos obligatorios de Parámetros eléctricos."; NotificationType.Error);
                Navigate(scrTableroPaso4; ScreenTransition.None)
              )
```

---

## 6. `scrTableroPaso4` — Diseño constructivo + Guardar

La **última etapa** lleva el botón **Guardar tablero**, que valida **todos** los
campos (repartidos en las 4 pantallas), arma el registro, lo agrega/actualiza en
`colTableros` y **vuelve a la galería** (`scrTableros`).

```yaml
  scrTableroPaso4:
    Properties:
      Fill: =RGBA(245; 246; 248; 1)
    Children:
      # navSubPasos — Default = "4. Constructivo"
      - lblTituloPaso4:
          Control: ModernText@1.0.0
          Properties:
            Text: ="Diseño constructivo"
            X: =48
            Y: =24
            Width: =Parent.Width - 96
            Size: =18
            FontWeight: =FontWeight.Semibold

      - ddMaterialGabinete:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcMaterialGabinete
            ItemDisplayText: =ThisItem.Label
            Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcMaterialGabinete; Value = varEditIndex.MaterialGabinete))
            X: =48
            Y: =150
            Width: =560
      - ddColorGabinete:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcColorGabinete
            ItemDisplayText: =ThisItem.Label
            Default: =If(IsBlank(varEditIndex); LookUp(colOpcColorGabinete; Value="7035"); LookUp(colOpcColorGabinete; Value = varEditIndex.ColorGabinete))
            X: =648
            Y: =150
            Width: =560
      - ddTipoVentilacion:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcTipoVentilacion
            ItemDisplayText: =ThisItem.Label
            Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTipoVentilacion; Value = varEditIndex.TipoVentilacion))
            X: =48
            Y: =230
            Width: =560
      - ddExpansionFutura:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcExpansionFutura
            ItemDisplayText: =ThisItem.Label
            Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcExpansionFutura; Value = varEditIndex.ExpansionFutura))
            X: =648
            Y: =230
            Width: =560
      - txtObservacionesTablero:
          Control: ModernTextInput@1.0.0
          Properties:
            Type: =TextInputType.Multiline
            Placeholder: ="Observaciones del tablero"
            Default: =If(IsBlank(varEditIndex); ""; varEditIndex.ObservacionesTablero)
            X: =48
            Y: =310
            Width: =1160
            Height: =140

      # --- Navegación inferior + Guardar ---
      - btnCancelarP4:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Cancelar"
            Appearance: =ButtonAppearance.Subtle
            X: =48
            Y: =Parent.Height - 88
            Width: =160
            OnSelect: =Set(varEditIndex; Blank());; Navigate(scrTableros; ScreenTransition.UnCover)
      - btnAtrasP4:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="◀ Atrás"
            Appearance: =ButtonAppearance.Secondary
            X: =Parent.Width - 468
            Y: =Parent.Height - 88
            Width: =200
            OnSelect: =Navigate(scrTableroPaso3; ScreenTransition.None)
      - btnGuardarTablero:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Guardar tablero ✓"
            Appearance: =ButtonAppearance.Primary
            X: =Parent.Width - 248
            Y: =Parent.Height - 88
            Width: =200
            # Valida TODO el tablero (campos de las 4 pantallas se leen por nombre),
            # arma el registro, lo agrega/actualiza y VUELVE a la galería.
            OnSelect: |
              =If(
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
                Notify("Completa los campos obligatorios del tablero (revisa las 4 sub-pestañas)."; NotificationType.Error);
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
                      CorrienteNominal: Value(Substitute(Substitute(lblCorriente.Text; "Corriente nominal ≈ "; ""); " A"; ""));
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
                Navigate(scrTableros; ScreenTransition.UnCover)
              )
```

> **`CorrienteNominal`:** como `lblCorriente.Text` ahora trae el texto
> `"Corriente nominal ≈ 123 A"` (más legible en pantalla), se limpia con
> `Substitute(...)` antes de `Value(...)`. Si prefieres dejar `lblCorriente`
> como número puro (solo `Text(Round(...))`, como en el diseño anterior), usa
> directamente `CorrienteNominal: Value(lblCorriente.Text)` y quita los
> `Substitute`.

> **No hace falta `Reset` aquí:** al volver a la galería, el próximo
> **Agregar/Editar** vuelve a resetear los 35 controles antes de reentrar al
> sub-flujo, así que no queda estado viejo. Esto elimina una de las 4 copias
> del bloque de `Reset` que tenía el diseño de una sola pantalla.

---

## 7. Qué cambió respecto al diseño de una sola pantalla

| Antes (una pantalla) | Ahora (sub-formulario en 4 pantallas) |
|---|---|
| `scrTableroForm` = galería + formulario completo con scroll | `scrTableros` (galería) + `scrTableroPaso1..4` (una etapa por pantalla) |
| Contenedor derecho con scroll para ~35 campos | Cada pantalla muestra ~5-11 campos, sin scroll |
| Editar/Agregar solo cambia `varEditIndex` y resetea (misma pantalla) | Editar/Agregar resetea **y navega** a `scrTableroPaso1` |
| Guardar vuelve a limpiar el panel (misma pantalla), 4 copias del `Reset` | Guardar **vuelve a la galería** (`scrTableros`); el `Reset` vive solo en Agregar/Editar (2 copias) |
| Sin sub-pestañas | Barra `navSubPasos` (`ModernTabList`) en las 4 pantallas del sub-flujo |
| Barra de pestañas principal en la misma pantalla del formulario | Barra principal solo en la galería; el sub-flujo usa **sub-pestañas** |

## 8. Verificación

1. **Vista previa** (F5). En Tableros (`scrTableros`) → **Agregar tablero** →
   abre `scrTableroPaso1` con los campos vacíos y `lblModoEdicion` = "Nuevo
   tablero".
2. Recorre las 4 sub-pestañas (o los botones Siguiente/Atrás); confirma que lo
   escrito en el Paso 1 sigue ahí al volver desde el Paso 4. Toca **Siguiente ▶**
   con un obligatorio de esa etapa vacío → aparece el `Notify` de esa etapa y
   no avanza; con la etapa completa, avanza.
3. En el Paso 4 → **Guardar tablero**: con campos obligatorios vacíos muestra el
   `Notify` (aunque falte algo del Paso 1, la validación lo detecta porque lee
   por nombre); con todo completo, agrega la fila y **vuelve a la galería**.
4. En la galería, selecciona una fila → **Editar** → abre el Paso 1 con
   `lblModoEdicion` = "Editando: …" y los valores cargados en las 4 pantallas.
5. **Cancelar** en cualquier paso vuelve a la galería sin agregar/modificar
   nada.
6. **Siguiente ▶** de la galería con la lista vacía muestra el error y no
   navega a Documentación.
