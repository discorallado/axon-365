# Canvas — código YAML de la pantalla de captura (Dataverse, Modern controls)

Código fuente `.pa.yaml` de la **Pantalla 1 — Contacto y Proyecto** y de las
**~37 columnas del tablero** (Pantallas 2 y 3), enlazados a **Dataverse**, con
los **controles Modern** de Power Apps (Fluent) en vez de los controles
clásicos.

## Controles Modern usados — mapeo y diferencias clave

Verificado contra la documentación oficial de Microsoft (ver Fuentes al
final). Cada control Modern tiene un identificador de tipo distinto en YAML
(`Control: Modern<Nombre>@1.0.0`) y varias propiedades cambiaron de nombre o
de comportamiento:

| Clásico | Modern (`Control:` en YAML) | Cambios que importan aquí |
|---|---|---|
| `Text` (label) | `ModernText@1.0.0` | Igual (`Text`, `Size`, `Color`, `Visible`). Ahora también soporta `OnSelect` (no lo usamos). |
| `TextInput` (texto) | `ModernTextInput@1.0.0` | `HintText` → **`Placeholder`**. Multilínea: `Mode: =TextMode.MultiLine` → **`Type: =TextInputType.Multiline`**. El valor se sigue leyendo con `.Text`. |
| `TextInput` + `Format: =TextFormat.Number` | **`ModernNumberInput@1.0.0`** (control dedicado, no una variante de TextInput) | El valor se lee con **`.Value`** — ya es numérico, no hace falta `Value(control.Text)`. Tiene **`Min`/`Max`** nativos (reemplazan el `Clamp(...)` manual). Usa `HintText` (no `Placeholder`, a diferencia del TextInput — así lo documenta Microsoft, aunque no sea consistente entre controles). |
| `DropDown` | `ModernDropdown@1.0.0` | `Value: =Label` (truco para mostrar la etiqueta) → **`ItemDisplayText: =ThisItem.Label`**. `Default` ya **no acepta un string plano** — necesita el **record completo** de `Items` que corresponde a la opción (`{Value:"...";Label:"..."}` o el resultado de un `LookUp`). `.Selected.Value` se sigue leyendo igual. |
| `ComboBox` (multi-select) | `ModernCombobox@1.0.0` | Igual `SelectMultiple`, `DefaultSelectedItems`, `.SelectedItems`. Se agrega **`ItemDisplayText: =ThisItem.Label`** (antes el multiselect mostraba el texto por convención; ahora hay que decirlo explícito). |
| `Toggle` | `ModernToggle@1.0.0` | `Value` (leer) + `Default` (setear) se **unifican en una sola propiedad: `Checked`**. `Text` → **`Label`**. |
| `DatePicker` | `ModernDatePicker@1.0.0` | `.SelectedDate` se sigue leyendo igual. |
| `Button` | `ModernButton@1.0.0` | `Text`, `OnSelect`, `DisplayMode` iguales. **`Fill` ya no existe** para resaltar estado — se usa **`Appearance`** (`ButtonAppearance.Primary/Secondary/Outline/Subtle/Transparent`). |
| — | **`ModernTabList@1.0.0`** (nuevo) | Reemplaza la barra de pestañas armada a mano con 3 botones — ver [dataverse/05-construir-canvas-captura.md](../dataverse/05-construir-canvas-captura.md) Bloque 14b. |

> **`Group`/contenedor y `Gallery` NO tienen versión Modern** (no están en el
> catálogo oficial de controles Modern al momento de escribir esto). Se
> mantienen como controles clásicos — `galTableros` (la galería de tableros
> del maestro-detalle) y los `Group` de confirmación siguen exactamente igual
> que antes.

## Cómo usarlo

Este YAML no se pega directo en cualquier vista del estudio. Cuatro vías:

1. **Manual en el navegador (recomendada para este proyecto — sin Git ni CLI):**
   inserta cada control desde el estudio (los controles Modern están bajo
   **Insertar → Modern**) y pega solo la fórmula (lo que va después del `=`)
   en la propiedad indicada. Guía clic por clic en
   [docs/dataverse/05-construir-canvas-captura.md](../dataverse/05-construir-canvas-captura.md).
2. **Todo de una vez (best-effort):** [06-yaml-completo-para-pegar.md](06-yaml-completo-para-pegar.md)
   consolida la app entera (`App` + las 4 pantallas) en un solo YAML, para
   probar el pegado masivo de Studio (clic derecho en el árbol → Pegar) o para
   usar con Git integration/CLI de una vez. Lee el aviso al inicio de ese
   documento antes de intentarlo — no pude confirmar al 100% que el pegado
   masivo siga funcionando igual en la versión actual de Studio.
3. **Git integration (para equipos con repo Git ya conectado a la solución):**
   los archivos `Src/*.pa.yaml` son editables y se sincronizan al estudio.
4. **Power Platform CLI:**
   ```
   pac canvas pack --sources ./Src --msapp Captura.msapp
   ```
   y luego importas el `.msapp`.

## Requisitos previos

- En la app, **Agregar datos** → conector **Dataverse** → tablas `Solicitudes` y
  `SolicitudTableros`.
- Los nombres de columna de Choice de Dataverse se referencian como
  `'Columna (Tabla)'.Opcion` (sintaxis de option set), distinta de SharePoint.
  Cuando la etiqueta de la opción tiene espacios, tildes o símbolos, se cita
  también: `'Columna (Tabla)'.'Etiqueta exacta'` — funciona igual y evita
  adivinar cómo Dataverse sanea el texto. Este documento usa esa forma citada
  en todas las columnas nuevas para no depender de esa sanitización.
- **Notación Power Fx:** separador decimal `,` en el entorno → argumentos de
  función con `;`, encadenado de instrucciones con `;;`.
- Las tablas, columnas y Choices de `SolicitudTablero` deben existir ya
  (Bloque 2-4 de [00-construir-en-powerapps.md](../dataverse/00-construir-en-powerapps.md))
  antes de construir esta pantalla — los nombres de columna y las etiquetas de
  cada opción usadas aquí deben coincidir exactamente con lo creado allí.

---

## App — colección e inicialización

Además de la colección de tableros, se definen aquí las **colecciones de
opciones** de cada `ModernDropdown`/`ModernCombobox` (código corto + etiqueta
en español). Definirlas una sola vez evita repetir el mismo par código/etiqueta
en `Items` y de nuevo en el `Default` de edición (ver 02-canvas-guia-construccion.md §3.4).

```yaml
App:
  Properties:
    OnStart: |
      =Set(varEditIndex; Blank());;
      // colTableros vacía PERO tipada: define todas las columnas y sus tipos
      // (esquema de cada fila = esquema de varEditIndex al editar), para que
      // varEditIndex.Campo y los IsBlank/Default no den "columna inexistente"
      // antes de agregar el primer tablero. Filter(...; false) = 0 filas, con esquema.
      ClearCollect(colTableros; Filter(Table({
        Nombre:                    "";
        Cantidad:                  0;
        Orden:                     0;
        TipoEntrega:               "";
        InstalacionNuevaReemplazo: "";
        TipoTablero:               "";
        OtroTipoTablero:           "";
        FuncionTablero:            "";
        CargasAAlimentar:          "";
        NumeroCircuitos:           0;
        Ubicacion:                 "";
        AmbienteEspecial:          Filter(Table({Value:""; Label:""}); false);
        OtroAmbienteEspecial:      "";
        GradoIP:                   "";
        GradoIK:                   "";
        TipoMontaje:               "";
        RestriccionesDimension:    false;
        AltoMaxMm:                 0;
        AnchoMaxMm:                0;
        FondoMaxMm:                0;
        CondicionesInstalacion:    "";
        TensionSuministro:         "";
        OtraTension:               0;
        SistemaElectrico:          "";
        OtroSistemaElectrico:      "";
        PotenciaEstimada:          0;
        UnidadPotencia:            "";
        CorrienteNominal:          0;
        Frecuencia:                "";
        OtraFrecuencia:            0;
        ProteccionesRequeridas:    Filter(Table({Value:""; Label:""}); false);
        MarcasPreferidas:          Filter(Table({Value:""; Label:""}); false);
        MaterialGabinete:          "";
        ColorGabinete:             "";
        TipoVentilacion:           "";
        ExpansionFutura:           "";
        ObservacionesTablero:      ""
      }); false));;

      ClearCollect(colOpcIngenieriaPor; Table(
        {Value:"csenergy"; Label:"CSEnergy la provee"};
        {Value:"cliente";  Label:"Por parte del cliente"};
        {Value:"conjunta"; Label:"Conjunta (CSEnergy + Cliente)"}
      ));;
      ClearCollect(colOpcTipoEntrega; Table(
        {Value:"tablero";  Label:"Tablero Eléctrico"};
        {Value:"sala";     Label:"Sala Eléctrica"};
        {Value:"producto"; Label:"Producto Eléctrico"}
      ));;
      ClearCollect(colOpcInstalacionNuevaReemplazo; Table(
        {Value:"nueva";     Label:"Instalación nueva"};
        {Value:"reemplazo"; Label:"Reemplazo de existente"}
      ));;
      ClearCollect(colOpcTipoTablero; Table(
        {Value:"fuerza";         Label:"Fuerza/Potencia"};
        {Value:"alumbrado";      Label:"Alumbrado/Distribución BT"};
        {Value:"control";        Label:"Control/Automatización"};
        {Value:"transfer";       Label:"Transferencia (ATS/MTS)"};
        {Value:"sincronizacion"; Label:"Sincronización de Generadores"};
        {Value:"remoto";         Label:"Distribución Remoto"};
        {Value:"pfcs";           Label:"Factor de Potencia"};
        {Value:"medicion";       Label:"Medición/Centro de Carga"};
        {Value:"variadores";     Label:"Variadores de Frecuencia"};
        {Value:"arrancadores";   Label:"Arrancadores Suaves"};
        {Value:"ups";            Label:"UPS/Respaldo"};
        {Value:"otro";           Label:"Otro"}
      ));;
      ClearCollect(colOpcUbicacion; Table(
        {Value:"interior"; Label:"Interior"};
        {Value:"exterior"; Label:"Exterior"}
      ));;
      ClearCollect(colOpcAmbienteEspecial; Table(
        {Value:"marino";      Label:"Ambiente marino"};
        {Value:"minero";      Label:"Ambiente minero"};
        {Value:"humedo";      Label:"Húmedo"};
        {Value:"corrosivo";   Label:"Corrosivo"};
        {Value:"polvoriento"; Label:"Polvoriento"};
        {Value:"explosivo";   Label:"Atmósfera explosiva"};
        {Value:"otro";        Label:"Otro"}
      ));;
      ClearCollect(colOpcTipoMontaje; Table(
        {Value:"autosoportado"; Label:"Autosoportado"};
        {Value:"mural";         Label:"Mural"};
        {Value:"rack_19";       Label:"Rack 19"};
        {Value:"pedestal";      Label:"Pedestal"};
        {Value:"otro";          Label:"Otro"}
      ));;
      ClearCollect(colOpcTension; Table(
        {Value:"220";  Label:"220 V"};
        {Value:"380";  Label:"380 V"};
        {Value:"400";  Label:"400 V"};
        {Value:"440";  Label:"440 V"};
        {Value:"480";  Label:"480 V"};
        {Value:"690";  Label:"690 V"};
        {Value:"1000"; Label:"1000 V"};
        {Value:"otro"; Label:"Otro"}
      ));;
      ClearCollect(colOpcSistema; Table(
        {Value:"trifasico";  Label:"Trifásico"};
        {Value:"monofasico"; Label:"Monofásico"};
        {Value:"dc";         Label:"Corriente continua (DC)"};
        {Value:"otro";       Label:"Otro"}
      ));;
      ClearCollect(colOpcUnidadPotencia; Table(
        {Value:"kW";  Label:"kW"};
        {Value:"kVA"; Label:"kVA"}
      ));;
      ClearCollect(colOpcFrecuencia; Table(
        {Value:"50";   Label:"50 Hz"};
        {Value:"60";   Label:"60 Hz"};
        {Value:"otro"; Label:"Otra"}
      ));;
      ClearCollect(colOpcProteccionesRequeridas; Table(
        {Value:"interruptor_automatico"; Label:"Interruptor automático"};
        {Value:"diferencial";            Label:"Diferencial"};
        {Value:"fusible";                Label:"Fusible"};
        {Value:"relevo_sobrecarga";      Label:"Relevo de sobrecarga"};
        {Value:"relevo_falla_tierra";    Label:"Relevo de falla a tierra"};
        {Value:"proteccion_tension";     Label:"Protección de tensión"};
        {Value:"proteccion_corriente";   Label:"Protección de corriente"};
        {Value:"descargador_tension";    Label:"Descargador de tensión"};
        {Value:"otro";                   Label:"Otro"}
      ));;
      ClearCollect(colOpcMarcasPreferidas; Table(
        {Value:"schneider";  Label:"Schneider Electric"};
        {Value:"siemens";    Label:"Siemens"};
        {Value:"abb";        Label:"ABB"};
        {Value:"legrand";    Label:"Legrand"};
        {Value:"eaton";      Label:"Eaton"};
        {Value:"chint";      Label:"Chint"};
        {Value:"hager";      Label:"Hager"};
        {Value:"weidmuller"; Label:"Weidmüller"};
        {Value:"phoenix";    Label:"Phoenix Contact"};
        {Value:"otro";       Label:"Otro"}
      ));;
      ClearCollect(colOpcMaterialGabinete; Table(
        {Value:"acero_pintado";     Label:"Acero pintado"};
        {Value:"acero_galvanizado"; Label:"Acero galvanizado"};
        {Value:"acero_inoxidable";  Label:"Acero inoxidable"};
        {Value:"acero_inox_316";    Label:"Acero inoxidable 316"};
        {Value:"fibra_vidrio";      Label:"Fibra de vidrio"};
        {Value:"poliester";         Label:"Poliéster"};
        {Value:"aluminio";          Label:"Aluminio"}
      ));;
      ClearCollect(colOpcColorGabinete; Table(
        {Value:"7035"; Label:"RAL 7035 (gris claro)"};
        {Value:"7016"; Label:"RAL 7016 (gris antracita)"};
        {Value:"9016"; Label:"RAL 9016 (blanco tráfico)"};
        {Value:"9005"; Label:"RAL 9005 (negro)"};
        {Value:"5010"; Label:"RAL 5010 (azul)"};
        {Value:"6005"; Label:"RAL 6005 (verde)"};
        {Value:"otro"; Label:"Otro"}
      ));;
      ClearCollect(colOpcTipoVentilacion; Table(
        {Value:"natural";     Label:"Natural"};
        {Value:"forzada";     Label:"Forzada (ventilador)"};
        {Value:"sellado";     Label:"Sellado (IP alto)"};
        {Value:"climatizado"; Label:"Climatizado (aire acondicionado)"}
      ));;
      ClearCollect(colOpcExpansionFutura; Table(
        {Value:"no";   Label:"Sin expansión"};
        {Value:"10";   Label:"10%"};
        {Value:"20";   Label:"20%"};
        {Value:"30";   Label:"30%"};
        {Value:"otro"; Label:"Otro"}
      ))
    # Si prefieres, define colTableros/colOpc* como Named Formulas en lugar de OnStart.
```

> `GradoIP` (depende de `Ubicacion`, se arma con `Switch`) y `GradoIK` (lista
> fija sin etiqueta propia — el código es la etiqueta) **no** tienen colección:
> siguen como arrays de texto plano directamente en el `Items` del control.

---

## Pantalla 1 — Contacto y Proyecto

```yaml
Screens:
  scrContacto:
    Properties:
      Fill: =RGBA(245; 246; 248; 1)
    Children:
      - lblTituloContacto:
          Control: ModernText@1.0.0
          Properties:
            Text: ="Contacto y Proyecto"
            X: =40
            Y: =32
            Width: =Parent.Width - 80
            Size: =20
            FontWeight: =FontWeight.Semibold

      # --- Datos de contacto ---
      - txtContactoNombre:
          Control: ModernTextInput@1.0.0
          Properties:
            Placeholder: ="Nombre del contacto *"
            X: =40
            Y: =88
            Width: =360
      - txtContactoEmail:
          Control: ModernTextInput@1.0.0
          Properties:
            Placeholder: ="Correo electrónico *"
            X: =420
            Y: =88
            Width: =360
      - lblEmailError:
          Control: ModernText@1.0.0
          Properties:
            Text: ="Correo con formato inválido"
            X: =420
            Y: =128
            Size: =9
            Color: =RGBA(239; 68; 68; 1)
            Visible: =And(Len(txtContactoEmail.Text) > 0; Not(IsMatch(txtContactoEmail.Text; Match.Email)))
      - txtContactoTelefono:
          Control: ModernTextInput@1.0.0
          Properties:
            Placeholder: ="Teléfono (+56)"
            X: =800
            Y: =88
            Width: =200

      # --- Datos del proyecto ---
      - txtNombreProyecto:
          Control: ModernTextInput@1.0.0
          Properties:
            Placeholder: ="Nombre del proyecto/obra *"
            X: =40
            Y: =160
            Width: =360
      - txtEmpresaCliente:
          Control: ModernTextInput@1.0.0
          Properties:
            Placeholder: ="Empresa/Cliente"
            X: =420
            Y: =160
            Width: =360
      - txtUbicacion:
          Control: ModernTextInput@1.0.0
          Properties:
            Placeholder: ="Ubicación de la instalación"
            X: =40
            Y: =220
            Width: =360
      - txtCentroCosto:
          Control: ModernTextInput@1.0.0
          Properties:
            Placeholder: ="Centro de costo (MO-12345)"
            X: =420
            Y: =220
            Width: =360
      - dtpEntrega:
          Control: ModernDatePicker@1.0.0
          Properties:
            X: =800
            Y: =220
            Width: =200

      - ddIngenieriaPor:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcIngenieriaPor
            ItemDisplayText: =ThisItem.Label
            X: =40
            Y: =292
            Width: =500

      # --- Navegación ---
      - btnSiguiente:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Siguiente"
            X: =800
            Y: =360
            Width: =200
            OnSelect: |
              =If(
                Or(
                  IsBlank(txtContactoNombre.Text);
                  Not(IsMatch(txtContactoEmail.Text; Match.Email));
                  IsBlank(txtNombreProyecto.Text);
                  IsBlank(ddIngenieriaPor.Selected.Value)
                );
                Notify("Completa los campos obligatorios."; NotificationType.Error);
                Navigate(scrTableros; ScreenTransition.Cover)
              )
```

---

## Pantalla del tablero — paso 1: Identificación

```yaml
  scrTableroForm:
    Children:
      - ddTipoEntrega:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcTipoEntrega
            ItemDisplayText: =ThisItem.Label

      - ddInstalacionNuevaReemplazo:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcInstalacionNuevaReemplazo
            ItemDisplayText: =ThisItem.Label

      - txtNombreTablero:
          Control: ModernTextInput@1.0.0
          Properties:
            Placeholder: ="Nombre del tablero * (Ej.: TG Principal)"

      - numCantidad:
          Control: ModernNumberInput@1.0.0
          Properties:
            HintText: ="Cantidad"
            Default: =1
            Min: =1
            Max: =999
            Precision: =DecimalPrecision.'0'

      # Tipo de tablero: visible solo si TipoEntrega = "tablero"
      - ddTipoTablero:
          Control: ModernDropdown@1.0.0
          Properties:
            Visible: =ddTipoEntrega.Selected.Value = "tablero"
            Items: =colOpcTipoTablero
            ItemDisplayText: =ThisItem.Label

      # Otro tipo de tablero: visible solo si TipoTablero = "otro"
      - txtOtroTipoTablero:
          Control: ModernTextInput@1.0.0
          Properties:
            Visible: =ddTipoTablero.Selected.Value = "otro"
            Placeholder: ="Especifica el tipo de tablero"

      - txtFuncionTablero:
          Control: ModernTextInput@1.0.0
          Properties:
            Type: =TextInputType.Multiline
            Placeholder: ="Función del tablero"

      - txtCargasAAlimentar:
          Control: ModernTextInput@1.0.0
          Properties:
            Type: =TextInputType.Multiline
            Placeholder: ="Cargas a alimentar"

      - numNumeroCircuitos:
          Control: ModernNumberInput@1.0.0
          Properties:
            HintText: ="Número de circuitos"
            Min: =0
            Precision: =DecimalPrecision.'0'
```

---

## Pantalla del tablero — paso 2a: Ubicación, ambiente y montaje

```yaml
  scrTableroForm2a:
    Children:
      - ddUbicacion:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcUbicacion
            ItemDisplayText: =ThisItem.Label

      # Ambiente especial: multi-select
      - cmbAmbienteEspecial:
          Control: ModernCombobox@1.0.0
          Properties:
            Items: =colOpcAmbienteEspecial
            ItemDisplayText: =ThisItem.Label
            SelectMultiple: =true

      # Otro ambiente especial: visible si el multiselect incluye "otro"
      - txtOtroAmbiente:
          Control: ModernTextInput@1.0.0
          Properties:
            Visible: ="otro" in cmbAmbienteEspecial.SelectedItems.Value
            Placeholder: ="Especifica el ambiente especial"

      # Grado IP: subconjunto según interior/exterior (ver 01-tablas.md) — array
      # de texto plano, sin colección: ItemDisplayText no hace falta (el código
      # es la etiqueta).
      - ddGradoIP:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: |
              =Switch(ddUbicacion.Selected.Value;
                "interior"; ["IP20";"IP31";"IP43";"IP54";"IP55";"IP65"];
                "exterior"; ["IP54";"IP55";"IP65";"IP66";"IP67";"IP68"];
                ["IP20";"IP31";"IP43";"IP54";"IP55";"IP65";"IP66";"IP67";"IP68"]
              )

      - ddGradoIK:
          Control: ModernDropdown@1.0.0
          Properties:
            # IK06=1J, IK07=2J, IK08=5J, IK09=10J, IK10=20J (IEC 62262)
            Items: |
              =["IK06";"IK07";"IK08";"IK09";"IK10"]

      - ddTipoMontaje:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcTipoMontaje
            ItemDisplayText: =ThisItem.Label

      - tglRestricciones:
          Control: ModernToggle@1.0.0
          Properties:
            Label: ="¿Tiene restricciones de dimensión?"

      # Dimensiones máximas: visibles solo si hay restricciones
      - numAltoMax:
          Control: ModernNumberInput@1.0.0
          Properties:
            Visible: =tglRestricciones.Checked
            HintText: ="Alto máximo (mm)"
            Min: =0
            Precision: =DecimalPrecision.'0'
      - numAnchoMax:
          Control: ModernNumberInput@1.0.0
          Properties:
            Visible: =tglRestricciones.Checked
            HintText: ="Ancho máximo (mm)"
            Min: =0
            Precision: =DecimalPrecision.'0'
      - numFondoMax:
          Control: ModernNumberInput@1.0.0
          Properties:
            Visible: =tglRestricciones.Checked
            HintText: ="Fondo máximo (mm)"
            Min: =0
            Precision: =DecimalPrecision.'0'

      - txtCondicionesInstalacion:
          Control: ModernTextInput@1.0.0
          Properties:
            Type: =TextInputType.Multiline
            Placeholder: ="Condiciones adicionales de instalación"
```

---

## Pantalla del tablero — paso 2b: Parámetros eléctricos

```yaml
  scrTableroForm2b:
    Children:
      - ddTension:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcTension
            ItemDisplayText: =ThisItem.Label

      - numOtraTension:
          Control: ModernNumberInput@1.0.0
          Properties:
            Visible: =ddTension.Selected.Value = "otro"
            HintText: ="Tensión (V)"
            Min: =0

      # Sistema eléctrico: SIN "bifasico" (decisión B-1)
      - ddSistema:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcSistema
            ItemDisplayText: =ThisItem.Label

      - txtOtroSistema:
          Control: ModernTextInput@1.0.0
          Properties:
            Visible: =ddSistema.Selected.Value = "otro"
            Placeholder: ="Especifica el sistema eléctrico"

      - numPotencia:
          Control: ModernNumberInput@1.0.0
          Properties:
            HintText: ="Potencia estimada"
            Min: =0
            Precision: =DecimalPrecision.'2'

      - ddUnidadPotencia:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcUnidadPotencia
            ItemDisplayText: =ThisItem.Label
            Default: =LookUp(colOpcUnidadPotencia; Value = "kW")

      # Corriente nominal: calculada (sistema bifásico eliminado, decisión B-1)
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
                  Text(Round(
                    Switch(ddSistema.Selected.Value;
                      "trifasico"; watts/(Sqrt(3)*v);
                      "monofasico"; watts/v;
                      "dc"; watts/v;
                      Blank()
                    ); 1)
                  )
                )
              )

      - ddFrecuencia:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcFrecuencia
            ItemDisplayText: =ThisItem.Label

      - numOtraFrecuencia:
          Control: ModernNumberInput@1.0.0
          Properties:
            Visible: =ddFrecuencia.Selected.Value = "otro"
            HintText: ="Frecuencia (Hz)"
            Min: =0

      - cmbProteccionesRequeridas:
          Control: ModernCombobox@1.0.0
          Properties:
            Items: =colOpcProteccionesRequeridas
            ItemDisplayText: =ThisItem.Label
            SelectMultiple: =true

      - cmbMarcasPreferidas:
          Control: ModernCombobox@1.0.0
          Properties:
            Items: =colOpcMarcasPreferidas
            ItemDisplayText: =ThisItem.Label
            SelectMultiple: =true
```

---

## Pantalla del tablero — paso 3: Diseño constructivo

```yaml
  scrTableroForm3:
    Children:
      - ddMaterialGabinete:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcMaterialGabinete
            ItemDisplayText: =ThisItem.Label

      - ddColorGabinete:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcColorGabinete
            ItemDisplayText: =ThisItem.Label
            Default: =LookUp(colOpcColorGabinete; Value = "7035")

      - ddTipoVentilacion:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcTipoVentilacion
            ItemDisplayText: =ThisItem.Label

      - ddExpansionFutura:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcExpansionFutura
            ItemDisplayText: =ThisItem.Label

      - txtObservacionesTablero:
          Control: ModernTextInput@1.0.0
          Properties:
            Type: =TextInputType.Multiline
            Placeholder: ="Observaciones del tablero"
```

> **Documentación/adjuntos del tablero (`EspecificacionesTecnicas`, `ListaCargas`,
> `DiagramaUnilineal`, `PlanosMecanicos`):** el control nativo de adjuntos de
> Power Apps solo funciona sobre un registro **ya guardado** en Dataverse; como
> `colTableros` es una colección en memoria (el tablero aún no existe como fila
> real hasta el envío final), no se puede adjuntar archivo por tablero durante
> esta pantalla. Dos opciones: (a) subir los adjuntos del tablero después del
> envío, reabriendo cada fila creada en un formulario de edición con el control
> **Adjuntar archivo**; (b) gestionarlos directamente desde la app Model-driven
> ([docs/dataverse/04-app-model-driven.md](../dataverse/04-app-model-driven.md)
> Bloque 11.2 §6), que sí edita el registro ya guardado. Los adjuntos **a nivel
> solicitud** (specs generales, fotos de sitio) sí se cargan en la Pantalla 3
> de [02-canvas-guia-construccion.md](02-canvas-guia-construccion.md) §4.

---

## Guardar el tablero en la colección

Agrupa los ~36 campos en un solo registro con `With(...)` para no duplicarlos
entre `Collect` (tablero nuevo) y `Patch` (edición de uno existente). Los
campos numéricos ya no necesitan `Value(control.Text)` — `ModernNumberInput`
entrega `.Value` numérico directamente, y `Min`/`Max` del control reemplazan
el `Clamp(...)` manual.

```yaml
      - btnGuardarTablero:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Guardar tablero"
            OnSelect: |
              =With(
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
              Navigate(scrTableros)
```

> En este punto `colTableros` guarda cada campo Choice con su **código corto
> interno** (`"tablero"`, `"otro"`, `"IP54"`…), no con el valor de Dataverse
> todavía. La conversión a `'Columna (Tabla)'.'Opción'` ocurre una sola vez, al
> enviar (sección siguiente) — así el formulario de captura no depende de que
> las Choices de Dataverse ya existan mientras se prueba la lógica en memoria.

> **Dos correcciones frente a una versión ingenua de este `registro`:**
> `Orden` solo se recalcula con `CountRows` para un tablero **nuevo** — al
> editar uno existente conserva su `Orden` original, si no cada edición lo
> movería al final de la colección. Y los campos "Otro"/condicionales
> (`OtroTipoTablero`, `OtroAmbienteEspecial`, `AltoMaxMm`/`AnchoMaxMm`/
> `FondoMaxMm`, `OtraTension`, `OtroSistemaElectrico`, `OtraFrecuencia`) se
> limpian a `Blank()` cuando su condición deja de aplicar — si no, un campo
> oculto en pantalla puede seguir guardando un valor viejo que ya no
> corresponde (ej. `OtroTipoTablero` con texto aunque `TipoTablero` ya no sea
> `"otro"`).

---

## Envío — escribir en Dataverse (Patch)

Sintaxis de Choice de Dataverse: `'Columna (Tabla)'.Opcion`, o citada
`'Columna (Tabla)'.'Etiqueta exacta'` cuando la etiqueta tiene espacios,
tildes o símbolos (evita adivinar cómo Dataverse sanea el texto — usa el
autocompletado del editor de fórmulas para confirmar el nombre exacto de cada
opción una vez creadas las Choices). Esta sección no cambia con los controles
Modern — ya trabaja sobre `t.CampoX` (valores ya extraídos en `colTableros`),
no sobre los controles directamente.

```yaml
      - btnEnviar:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Enviar solicitud"
            OnSelect: |
              =If(CountRows(colTableros) = 0;
                Notify("Debe agregar al menos un tablero."; NotificationType.Error);
                // Paso 1 — Cabecera (Estado como Choice de Dataverse)
                Set(varSolicitud;
                  Patch(Solicitudes; Defaults(Solicitudes); {
                    Estado: 'Estado (Solicitudes)'.Nueva;
                    NombreProyecto: txtNombreProyecto.Text;
                    UbicacionInstalacion: txtUbicacion.Text;
                    CentroCosto: txtCentroCosto.Text;
                    FechaEntregaDeseada: dtpEntrega.SelectedDate;
                    IngenieriaPor: Switch(ddIngenieriaPor.Selected.Value;
                      "csenergy"; 'Ingeniería por (Solicitudes)'.CSEnergy;
                      "cliente";  'Ingeniería por (Solicitudes)'.Cliente;
                      'Ingeniería por (Solicitudes)'.Conjunta
                    );
                    ContactoNombre: txtContactoNombre.Text;
                    ContactoEmail: txtContactoEmail.Text;
                    ContactoTelefono: txtContactoTelefono.Text;
                    EmpresaCliente: txtEmpresaCliente.Text;
                    EnviadoEl: Now()
                  })
                );;
                // Paso 2 — Tableros vinculados (relación 1:N de Dataverse)
                ForAll(colTableros As t;
                  Patch(SolicitudTableros; Defaults(SolicitudTableros); {
                    Solicitud: varSolicitud;            // lookup directo al registro padre
                    Nombre: t.Nombre;
                    Cantidad: t.Cantidad;
                    Orden: t.Orden;

                    // --- Identificación ---
                    TipoEntrega: Switch(t.TipoEntrega;
                      "tablero";  'Tipo de Entrega (SolicitudTableros)'.'Tablero Eléctrico';
                      "sala";     'Tipo de Entrega (SolicitudTableros)'.'Sala Eléctrica';
                      'Tipo de Entrega (SolicitudTableros)'.'Producto Eléctrico'
                    );
                    InstalacionNuevaReemplazo: Switch(t.InstalacionNuevaReemplazo;
                      "nueva";     'Instalación Nueva o Reemplazo (SolicitudTableros)'.'Instalación nueva';
                      'Instalación Nueva o Reemplazo (SolicitudTableros)'.'Reemplazo de existente'
                    );
                    TipoTablero: Switch(t.TipoTablero;
                      "fuerza";         'Tipo de Tablero (SolicitudTableros)'.'Fuerza/Potencia';
                      "alumbrado";      'Tipo de Tablero (SolicitudTableros)'.'Alumbrado/Distribución BT';
                      "control";        'Tipo de Tablero (SolicitudTableros)'.'Control/Automatización';
                      "transfer";       'Tipo de Tablero (SolicitudTableros)'.'Transferencia (ATS/MTS)';
                      "sincronizacion"; 'Tipo de Tablero (SolicitudTableros)'.'Sincronización de Generadores';
                      "remoto";         'Tipo de Tablero (SolicitudTableros)'.'Distribución Remoto';
                      "pfcs";           'Tipo de Tablero (SolicitudTableros)'.'Factor de Potencia';
                      "medicion";       'Tipo de Tablero (SolicitudTableros)'.'Medición/Centro de Carga';
                      "variadores";     'Tipo de Tablero (SolicitudTableros)'.'Variadores de Frecuencia';
                      "arrancadores";   'Tipo de Tablero (SolicitudTableros)'.'Arrancadores Suaves';
                      "ups";            'Tipo de Tablero (SolicitudTableros)'.'UPS/Respaldo';
                      'Tipo de Tablero (SolicitudTableros)'.Otro
                    );
                    OtroTipoTablero: t.OtroTipoTablero;
                    FuncionTablero: t.FuncionTablero;
                    CargasAAlimentar: t.CargasAAlimentar;
                    NumeroCircuitos: t.NumeroCircuitos;

                    // --- Ubicación y ambiente ---
                    Ubicacion: Switch(t.Ubicacion;
                      "interior"; 'Ubicación (SolicitudTableros)'.Interior;
                      'Ubicación (SolicitudTableros)'.Exterior
                    );
                    AmbienteEspecial: ForAll(t.AmbienteEspecial As sel;
                      Switch(sel.Value;
                        "marino";      'Ambiente Especial (SolicitudTableros)'.'Ambiente marino';
                        "minero";      'Ambiente Especial (SolicitudTableros)'.'Ambiente minero';
                        "humedo";      'Ambiente Especial (SolicitudTableros)'.'Húmedo';
                        "corrosivo";   'Ambiente Especial (SolicitudTableros)'.Corrosivo;
                        "polvoriento"; 'Ambiente Especial (SolicitudTableros)'.Polvoriento;
                        "explosivo";   'Ambiente Especial (SolicitudTableros)'.'Atmósfera explosiva';
                        'Ambiente Especial (SolicitudTableros)'.Otro
                      )
                    );
                    OtroAmbienteEspecial: t.OtroAmbienteEspecial;
                    GradoIP: Switch(t.GradoIP;
                      "IP20"; 'Grado IP (SolicitudTableros)'.IP20;
                      "IP31"; 'Grado IP (SolicitudTableros)'.IP31;
                      "IP43"; 'Grado IP (SolicitudTableros)'.IP43;
                      "IP54"; 'Grado IP (SolicitudTableros)'.IP54;
                      "IP55"; 'Grado IP (SolicitudTableros)'.IP55;
                      "IP65"; 'Grado IP (SolicitudTableros)'.IP65;
                      "IP66"; 'Grado IP (SolicitudTableros)'.IP66;
                      "IP67"; 'Grado IP (SolicitudTableros)'.IP67;
                      'Grado IP (SolicitudTableros)'.IP68
                    );
                    GradoIK: Switch(t.GradoIK;
                      "IK06"; 'Grado IK (SolicitudTableros)'.IK06;
                      "IK07"; 'Grado IK (SolicitudTableros)'.IK07;
                      "IK08"; 'Grado IK (SolicitudTableros)'.IK08;
                      "IK09"; 'Grado IK (SolicitudTableros)'.IK09;
                      'Grado IK (SolicitudTableros)'.IK10
                    );

                    // --- Montaje y dimensiones ---
                    TipoMontaje: Switch(t.TipoMontaje;
                      "autosoportado"; 'Tipo de Montaje (SolicitudTableros)'.Autosoportado;
                      "mural";         'Tipo de Montaje (SolicitudTableros)'.Mural;
                      "rack_19";       'Tipo de Montaje (SolicitudTableros)'.'Rack 19';
                      "pedestal";      'Tipo de Montaje (SolicitudTableros)'.Pedestal;
                      'Tipo de Montaje (SolicitudTableros)'.Otro
                    );
                    RestriccionesDimension: t.RestriccionesDimension;
                    AltoMaxMm: t.AltoMaxMm;
                    AnchoMaxMm: t.AnchoMaxMm;
                    FondoMaxMm: t.FondoMaxMm;
                    CondicionesInstalacion: t.CondicionesInstalacion;

                    // --- Parámetros eléctricos ---
                    TensionSuministro: Switch(t.TensionSuministro;
                      "220";  'Tensión de Suministro (SolicitudTableros)'.'220 V';
                      "380";  'Tensión de Suministro (SolicitudTableros)'.'380 V';
                      "400";  'Tensión de Suministro (SolicitudTableros)'.'400 V';
                      "440";  'Tensión de Suministro (SolicitudTableros)'.'440 V';
                      "480";  'Tensión de Suministro (SolicitudTableros)'.'480 V';
                      "690";  'Tensión de Suministro (SolicitudTableros)'.'690 V';
                      "1000"; 'Tensión de Suministro (SolicitudTableros)'.'1000 V';
                      'Tensión de Suministro (SolicitudTableros)'.Otro
                    );
                    OtraTension: t.OtraTension;
                    SistemaElectrico: Switch(t.SistemaElectrico;
                      "trifasico";  'Sistema Eléctrico (SolicitudTableros)'.'Trifásico';
                      "monofasico"; 'Sistema Eléctrico (SolicitudTableros)'.'Monofásico';
                      "dc";         'Sistema Eléctrico (SolicitudTableros)'.'Corriente continua (DC)';
                      'Sistema Eléctrico (SolicitudTableros)'.Otro
                    );
                    OtroSistemaElectrico: t.OtroSistemaElectrico;
                    PotenciaEstimada: t.PotenciaEstimada;
                    UnidadPotencia: Switch(t.UnidadPotencia;
                      "kW";  'Unidad de Potencia (SolicitudTableros)'.kW;
                      'Unidad de Potencia (SolicitudTableros)'.kVA
                    );
                    CorrienteNominal: t.CorrienteNominal;
                    Frecuencia: Switch(t.Frecuencia;
                      "50";  'Frecuencia (SolicitudTableros)'.'50 Hz';
                      "60";  'Frecuencia (SolicitudTableros)'.'60 Hz';
                      'Frecuencia (SolicitudTableros)'.Otra
                    );
                    OtraFrecuencia: t.OtraFrecuencia;
                    ProteccionesRequeridas: ForAll(t.ProteccionesRequeridas As sel;
                      Switch(sel.Value;
                        "interruptor_automatico"; 'Protecciones Requeridas (SolicitudTableros)'.'Interruptor automático';
                        "diferencial";            'Protecciones Requeridas (SolicitudTableros)'.Diferencial;
                        "fusible";                'Protecciones Requeridas (SolicitudTableros)'.Fusible;
                        "relevo_sobrecarga";       'Protecciones Requeridas (SolicitudTableros)'.'Relevo de sobrecarga';
                        "relevo_falla_tierra";     'Protecciones Requeridas (SolicitudTableros)'.'Relevo de falla a tierra';
                        "proteccion_tension";      'Protecciones Requeridas (SolicitudTableros)'.'Protección de tensión';
                        "proteccion_corriente";    'Protecciones Requeridas (SolicitudTableros)'.'Protección de corriente';
                        "descargador_tension";     'Protecciones Requeridas (SolicitudTableros)'.'Descargador de tensión';
                        'Protecciones Requeridas (SolicitudTableros)'.Otro
                      )
                    );
                    MarcasPreferidas: ForAll(t.MarcasPreferidas As sel;
                      Switch(sel.Value;
                        "schneider";   'Marcas Preferidas (SolicitudTableros)'.'Schneider Electric';
                        "siemens";     'Marcas Preferidas (SolicitudTableros)'.Siemens;
                        "abb";         'Marcas Preferidas (SolicitudTableros)'.ABB;
                        "legrand";     'Marcas Preferidas (SolicitudTableros)'.Legrand;
                        "eaton";       'Marcas Preferidas (SolicitudTableros)'.Eaton;
                        "chint";       'Marcas Preferidas (SolicitudTableros)'.Chint;
                        "hager";       'Marcas Preferidas (SolicitudTableros)'.Hager;
                        "weidmuller";  'Marcas Preferidas (SolicitudTableros)'.'Weidmüller';
                        "phoenix";     'Marcas Preferidas (SolicitudTableros)'.'Phoenix Contact';
                        'Marcas Preferidas (SolicitudTableros)'.Otro
                      )
                    );

                    // --- Diseño constructivo ---
                    MaterialGabinete: Switch(t.MaterialGabinete;
                      "acero_pintado";     'Material de Gabinete (SolicitudTableros)'.'Acero pintado';
                      "acero_galvanizado"; 'Material de Gabinete (SolicitudTableros)'.'Acero galvanizado';
                      "acero_inoxidable";  'Material de Gabinete (SolicitudTableros)'.'Acero inoxidable';
                      "acero_inox_316";    'Material de Gabinete (SolicitudTableros)'.'Acero inoxidable 316';
                      "fibra_vidrio";      'Material de Gabinete (SolicitudTableros)'.'Fibra de vidrio';
                      "poliester";         'Material de Gabinete (SolicitudTableros)'.'Poliéster';
                      'Material de Gabinete (SolicitudTableros)'.Aluminio
                    );
                    ColorGabinete: Switch(t.ColorGabinete;
                      "7035"; 'Color de Gabinete (SolicitudTableros)'.'RAL 7035 (gris claro)';
                      "7016"; 'Color de Gabinete (SolicitudTableros)'.'RAL 7016 (gris antracita)';
                      "9016"; 'Color de Gabinete (SolicitudTableros)'.'RAL 9016 (blanco tráfico)';
                      "9005"; 'Color de Gabinete (SolicitudTableros)'.'RAL 9005 (negro)';
                      "5010"; 'Color de Gabinete (SolicitudTableros)'.'RAL 5010 (azul)';
                      "6005"; 'Color de Gabinete (SolicitudTableros)'.'RAL 6005 (verde)';
                      'Color de Gabinete (SolicitudTableros)'.Otro
                    );
                    TipoVentilacion: Switch(t.TipoVentilacion;
                      "natural";     'Tipo de Ventilación (SolicitudTableros)'.Natural;
                      "forzada";     'Tipo de Ventilación (SolicitudTableros)'.'Forzada (ventilador)';
                      "sellado";     'Tipo de Ventilación (SolicitudTableros)'.'Sellado (IP alto)';
                      'Tipo de Ventilación (SolicitudTableros)'.'Climatizado (aire acondicionado)'
                    );
                    ExpansionFutura: Switch(t.ExpansionFutura;
                      "no";  'Expansión Futura (SolicitudTableros)'.'Sin expansión';
                      "10";  'Expansión Futura (SolicitudTableros)'.'10%';
                      "20";  'Expansión Futura (SolicitudTableros)'.'20%';
                      "30";  'Expansión Futura (SolicitudTableros)'.'30%';
                      'Expansión Futura (SolicitudTableros)'.Otro
                    );
                    ObservacionesTablero: t.ObservacionesTablero
                  })
                );;
                Notify("Solicitud enviada."; NotificationType.Success);;
                Clear(colTableros);;
                Navigate(scrConfirmacion)
              )
```

> **Ventajas Dataverse vs SharePoint aquí:**
> - El lookup `Solicitud` se setea pasando el **registro** (`varSolicitud`), no un
>   `{Id, Value}` como en SharePoint.
> - `reference_code` lo genera una columna **Autonumber** de Dataverse — no se setea
>   desde la app ni desde un flujo.
> - `EstadoPrevio` ya **no es necesario**: el Business Process Flow gestiona la
>   transición; ver el rediseño de estados.
> - Los tres campos multi-select (`AmbienteEspecial`, `ProteccionesRequeridas`,
>   `MarcasPreferidas`) se convierten con `ForAll(...As sel; Switch(sel.Value; ...))`,
>   que arma la tabla de opciones que Dataverse espera para una columna
>   multi-select — mismo patrón para los tres.

---

## Verificación antes de implementar

- Todas las etiquetas (`Label`) citadas en los `Switch` de la sección de envío
  deben coincidir **exactamente** (mayúsculas, tildes, símbolos) con las
  etiquetas que crees para cada opción de Choice en Dataverse (Bloque 2-4 de
  [00-construir-en-powerapps.md](../dataverse/00-construir-en-powerapps.md)).
  Si cambias una etiqueta allá, actualiza el `Switch` correspondiente aquí —
  y también su par en las colecciones `colOpc*` de "App — colección e
  inicialización", que son la fuente de esa misma etiqueta para `Items`.
- Los nombres de columna citados (`'Tipo de Entrega (SolicitudTableros)'`, etc.)
  asumen que le diste esos **nombres para mostrar** a las columnas al crearlas.
  Si usaste otros nombres, ajusta las citas — el prefijo de editor (`csen_`) no
  se escribe en la fórmula, Power Fx resuelve por nombre para mostrar.
- Los controles Modern son relativamente nuevos y Microsoft los sigue
  actualizando — si al pegar una fórmula el estudio marca una propiedad como
  inexistente, revisa la página de ese control en Microsoft Learn (enlaces en
  Fuentes) por si el nombre cambió desde que se escribió esta guía.
- Antes de conectar el botón **Enviar solicitud** a producción, prueba el
  recorrido completo en el entorno Developer: captura dos tableros con al
  menos un campo "otro" cada uno (para ejercitar todos los `Switch`), envía, y
  confirma en Dataverse (vista de tabla) que cada columna Choice quedó con la
  opción correcta y no en blanco.

## Fuentes

- [Overview of modern controls and theming in canvas apps](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/controls/modern-controls/overview-modern-controls)
- [Modern controls and properties in canvas apps (índice)](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/controls/modern-controls/modern-controls-reference)
- [Button modern control](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/controls/modern-controls/modern-control-button)
- [Text input modern control](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/controls/modern-controls/modern-control-text-input)
- [Number input modern control](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/controls/modern-controls/modern-control-number-input)
- [Dropdown modern control](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/controls/modern-controls/modern-control-dropdown)
- [Combo box modern control](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/controls/modern-controls/modern-control-combobox)
- [Toggle modern control](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/controls/modern-controls/modern-control-toggle)
- [Text modern control](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/controls/modern-controls/modern-control-text)
- [Date picker modern control](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/controls/modern-controls/modern-controls-date-picker)
- [Tab list modern control](https://learn.microsoft.com/en-us/power-apps/maker/canvas-apps/controls/modern-controls/modern-control-tabs-or-tabs-list)
