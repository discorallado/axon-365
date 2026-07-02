# Canvas — código YAML de la pantalla de captura (Dataverse)

Código fuente `.pa.yaml` de la **Pantalla 1 — Contacto y Proyecto** y de las
**~37 columnas del tablero** (Pantallas 2 y 3), enlazados a **Dataverse**.

## Cómo usarlo

Este YAML no se pega directo en cualquier vista del estudio. Tres vías:

1. **Manual en el navegador (recomendada para este proyecto — sin Git ni CLI):**
   inserta cada control desde el estudio y pega solo la fórmula (lo que va
   después del `=`) en la propiedad indicada. Guía clic por clic, con las
   pantallas y botones de navegación que faltan aquí (galería de tableros,
   Siguiente/Atrás), en
   [docs/dataverse/05-construir-canvas-captura.md](../dataverse/05-construir-canvas-captura.md).
2. **Git integration (para equipos con repo Git ya conectado a la solución):**
   los archivos `Src/*.pa.yaml` son editables y se sincronizan al estudio.
3. **Power Platform CLI:**
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

```yaml
App:
  Properties:
    OnStart: |
      =Set(varEditIndex; Blank());;
      ClearCollect(colTableros; [])
    # Si prefieres, define colTableros como Named Formula en lugar de OnStart.
```

---

## Pantalla 1 — Contacto y Proyecto

```yaml
Screens:
  scrContacto:
    Properties:
      Fill: =RGBA(245; 246; 248; 1)
    Children:
      - lblTituloContacto:
          Control: Text
          Properties:
            Text: ="Contacto y Proyecto"
            X: =40
            Y: =32
            Width: =Parent.Width - 80
            Size: =20
            FontWeight: =FontWeight.Semibold

      # --- Datos de contacto ---
      - txtContactoNombre:
          Control: TextInput
          Properties:
            HintText: ="Nombre del contacto *"
            X: =40
            Y: =88
            Width: =360
      - txtContactoEmail:
          Control: TextInput
          Properties:
            HintText: ="Correo electrónico *"
            X: =420
            Y: =88
            Width: =360
      - lblEmailError:
          Control: Text
          Properties:
            Text: ="Correo con formato inválido"
            X: =420
            Y: =128
            Size: =9
            Color: =RGBA(239; 68; 68; 1)
            Visible: =And(Len(txtContactoEmail.Text) > 0; Not(IsMatch(txtContactoEmail.Text; Match.Email)))
      - txtContactoTelefono:
          Control: TextInput
          Properties:
            HintText: ="Teléfono (+56)"
            X: =800
            Y: =88
            Width: =200

      # --- Datos del proyecto ---
      - txtNombreProyecto:
          Control: TextInput
          Properties:
            HintText: ="Nombre del proyecto/obra *"
            X: =40
            Y: =160
            Width: =360
      - txtEmpresaCliente:
          Control: TextInput
          Properties:
            HintText: ="Empresa/Cliente"
            X: =420
            Y: =160
            Width: =360
      - txtUbicacion:
          Control: TextInput
          Properties:
            HintText: ="Ubicación de la instalación"
            X: =40
            Y: =220
            Width: =360
      - txtCentroCosto:
          Control: TextInput
          Properties:
            HintText: ="Centro de costo (MO-12345)"
            X: =420
            Y: =220
            Width: =360
      - dtpEntrega:
          Control: DatePicker
          Properties:
            X: =800
            Y: =220
            Width: =200

      - ddIngenieriaPor:
          Control: DropDown
          Properties:
            Items: |
              =Table(
                {Value: "csenergy"; Label: "CSEnergy la provee"};
                {Value: "cliente";  Label: "Por parte del cliente"};
                {Value: "conjunta"; Label: "Conjunta (CSEnergy + Cliente)"}
              )
            Value: =Label
            X: =40
            Y: =292
            Width: =500

      # --- Navegación ---
      - btnSiguiente:
          Control: Button
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
          Control: DropDown
          Properties:
            Items: |
              =Table(
                {Value:"tablero"; Label:"Tablero Eléctrico"};
                {Value:"sala";    Label:"Sala Eléctrica"};
                {Value:"producto";Label:"Producto Eléctrico"}
              )
            Value: =Label

      - ddInstalacionNuevaReemplazo:
          Control: DropDown
          Properties:
            Items: |
              =Table(
                {Value:"nueva";     Label:"Instalación nueva"};
                {Value:"reemplazo"; Label:"Reemplazo de existente"}
              )
            Value: =Label

      - txtNombreTablero:
          Control: TextInput
          Properties:
            HintText: ="Nombre del tablero * (Ej.: TG Principal)"

      - txtCantidad:
          Control: TextInput
          Properties:
            HintText: ="Cantidad"
            Format: =TextFormat.Number
            Default: ="1"

      # Tipo de tablero: visible solo si TipoEntrega = "tablero"
      - ddTipoTablero:
          Control: DropDown
          Properties:
            Visible: =ddTipoEntrega.Selected.Value = "tablero"
            Items: |
              =Table(
                {Value:"fuerza";Label:"Fuerza/Potencia"};
                {Value:"alumbrado";Label:"Alumbrado/Distribución BT"};
                {Value:"control";Label:"Control/Automatización"};
                {Value:"transfer";Label:"Transferencia (ATS/MTS)"};
                {Value:"sincronizacion";Label:"Sincronización de Generadores"};
                {Value:"remoto";Label:"Distribución Remoto"};
                {Value:"pfcs";Label:"Factor de Potencia"};
                {Value:"medicion";Label:"Medición/Centro de Carga"};
                {Value:"variadores";Label:"Variadores de Frecuencia"};
                {Value:"arrancadores";Label:"Arrancadores Suaves"};
                {Value:"ups";Label:"UPS/Respaldo"};
                {Value:"otro";Label:"Otro"}
              )
            Value: =Label

      # Otro tipo de tablero: visible solo si TipoTablero = "otro"
      - txtOtroTipoTablero:
          Control: TextInput
          Properties:
            Visible: =ddTipoTablero.Selected.Value = "otro"
            HintText: ="Especifica el tipo de tablero"

      - txtFuncionTablero:
          Control: TextInput
          Properties:
            Mode: =TextMode.MultiLine
            HintText: ="Función del tablero"

      - txtCargasAAlimentar:
          Control: TextInput
          Properties:
            Mode: =TextMode.MultiLine
            HintText: ="Cargas a alimentar"

      - txtNumeroCircuitos:
          Control: TextInput
          Properties:
            HintText: ="Número de circuitos"
            Format: =TextFormat.Number
```

---

## Pantalla del tablero — paso 2a: Ubicación, ambiente y montaje

```yaml
  scrTableroForm2a:
    Children:
      - ddUbicacion:
          Control: DropDown
          Properties:
            Items: |
              =Table(
                {Value:"interior"; Label:"Interior"};
                {Value:"exterior"; Label:"Exterior"}
              )
            Value: =Label

      # Ambiente especial: multi-select
      - cmbAmbienteEspecial:
          Control: ComboBox
          Properties:
            SelectMultiple: =true
            Items: |
              =Table(
                {Value:"marino";      Label:"Ambiente marino"};
                {Value:"minero";      Label:"Ambiente minero"};
                {Value:"humedo";      Label:"Húmedo"};
                {Value:"corrosivo";   Label:"Corrosivo"};
                {Value:"polvoriento"; Label:"Polvoriento"};
                {Value:"explosivo";   Label:"Atmósfera explosiva"};
                {Value:"otro";        Label:"Otro"}
              )

      # Otro ambiente especial: visible si el multiselect incluye "otro"
      - txtOtroAmbiente:
          Control: TextInput
          Properties:
            Visible: ="otro" in cmbAmbienteEspecial.SelectedItems.Value
            HintText: ="Especifica el ambiente especial"

      # Grado IP: subconjunto según interior/exterior (ver 01-tablas.md)
      - ddGradoIP:
          Control: DropDown
          Properties:
            Items: |
              =Switch(ddUbicacion.Selected.Value;
                "interior"; ["IP20";"IP31";"IP43";"IP54";"IP55";"IP65"];
                "exterior"; ["IP54";"IP55";"IP65";"IP66";"IP67";"IP68"];
                ["IP20";"IP31";"IP43";"IP54";"IP55";"IP65";"IP66";"IP67";"IP68"]
              )

      - ddGradoIK:
          Control: DropDown
          Properties:
            # IK06=1J, IK07=2J, IK08=5J, IK09=10J, IK10=20J (IEC 62262)
            Items: |
              =["IK06";"IK07";"IK08";"IK09";"IK10"]

      - ddTipoMontaje:
          Control: DropDown
          Properties:
            Items: |
              =Table(
                {Value:"autosoportado"; Label:"Autosoportado"};
                {Value:"mural";         Label:"Mural"};
                {Value:"rack_19";       Label:"Rack 19"};
                {Value:"pedestal";      Label:"Pedestal"};
                {Value:"otro";          Label:"Otro"}
              )
            Value: =Label

      - tglRestricciones:
          Control: Toggle
          Properties:
            Text: ="¿Tiene restricciones de dimensión?"

      # Dimensiones máximas: visibles solo si hay restricciones
      - txtAltoMax:
          Control: TextInput
          Properties:
            Visible: =tglRestricciones.Value
            HintText: ="Alto máximo (mm)"
            Format: =TextFormat.Number
      - txtAnchoMax:
          Control: TextInput
          Properties:
            Visible: =tglRestricciones.Value
            HintText: ="Ancho máximo (mm)"
            Format: =TextFormat.Number
      - txtFondoMax:
          Control: TextInput
          Properties:
            Visible: =tglRestricciones.Value
            HintText: ="Fondo máximo (mm)"
            Format: =TextFormat.Number

      - txtCondicionesInstalacion:
          Control: TextInput
          Properties:
            Mode: =TextMode.MultiLine
            HintText: ="Condiciones adicionales de instalación"
```

---

## Pantalla del tablero — paso 2b: Parámetros eléctricos

```yaml
  scrTableroForm2b:
    Children:
      - ddTension:
          Control: DropDown
          Properties:
            Items: |
              =Table(
                {Value:"220";  Label:"220 V"};
                {Value:"380";  Label:"380 V"};
                {Value:"400";  Label:"400 V"};
                {Value:"440";  Label:"440 V"};
                {Value:"480";  Label:"480 V"};
                {Value:"690";  Label:"690 V"};
                {Value:"1000"; Label:"1000 V"};
                {Value:"otro"; Label:"Otro"}
              )
            Value: =Label

      - txtOtraTension:
          Control: TextInput
          Properties:
            Visible: =ddTension.Selected.Value = "otro"
            HintText: ="Tensión (V)"
            Format: =TextFormat.Number

      # Sistema eléctrico: SIN "bifasico" (decisión B-1)
      - ddSistema:
          Control: DropDown
          Properties:
            Items: |
              =Table(
                {Value:"trifasico";  Label:"Trifásico"};
                {Value:"monofasico"; Label:"Monofásico"};
                {Value:"dc";         Label:"Corriente continua (DC)"};
                {Value:"otro";       Label:"Otro"}
              )
            Value: =Label

      - txtOtroSistema:
          Control: TextInput
          Properties:
            Visible: =ddSistema.Selected.Value = "otro"
            HintText: ="Especifica el sistema eléctrico"

      - txtPotencia:
          Control: TextInput
          Properties:
            HintText: ="Potencia estimada"
            Format: =TextFormat.Number

      - ddUnidadPotencia:
          Control: DropDown
          Properties:
            Items: |
              =Table(
                {Value:"kW";  Label:"kW"};
                {Value:"kVA"; Label:"kVA"}
              )
            Value: =Label
            Default: ="kW"

      # Corriente nominal: calculada (sistema bifásico eliminado, decisión B-1)
      - lblCorriente:
          Control: Text
          Properties:
            Text: |
              =With(
                {
                  watts: Value(txtPotencia.Text) * 1000;
                  v: If(ddTension.Selected.Value="otro"; Value(txtOtraTension.Text); Value(ddTension.Selected.Value))
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
          Control: DropDown
          Properties:
            Items: |
              =Table(
                {Value:"50";   Label:"50 Hz"};
                {Value:"60";   Label:"60 Hz"};
                {Value:"otro"; Label:"Otra"}
              )
            Value: =Label

      - txtOtraFrecuencia:
          Control: TextInput
          Properties:
            Visible: =ddFrecuencia.Selected.Value = "otro"
            HintText: ="Frecuencia (Hz)"
            Format: =TextFormat.Number

      - cmbProteccionesRequeridas:
          Control: ComboBox
          Properties:
            SelectMultiple: =true
            Items: |
              =Table(
                {Value:"interruptor_automatico"; Label:"Interruptor automático"};
                {Value:"diferencial";            Label:"Diferencial"};
                {Value:"fusible";                Label:"Fusible"};
                {Value:"relevo_sobrecarga";       Label:"Relevo de sobrecarga"};
                {Value:"relevo_falla_tierra";     Label:"Relevo de falla a tierra"};
                {Value:"proteccion_tension";      Label:"Protección de tensión"};
                {Value:"proteccion_corriente";    Label:"Protección de corriente"};
                {Value:"descargador_tension";     Label:"Descargador de tensión"};
                {Value:"otro";                    Label:"Otro"}
              )

      - cmbMarcasPreferidas:
          Control: ComboBox
          Properties:
            SelectMultiple: =true
            Items: |
              =Table(
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
              )
```

---

## Pantalla del tablero — paso 3: Diseño constructivo

```yaml
  scrTableroForm3:
    Children:
      - ddMaterialGabinete:
          Control: DropDown
          Properties:
            Items: |
              =Table(
                {Value:"acero_pintado";     Label:"Acero pintado"};
                {Value:"acero_galvanizado"; Label:"Acero galvanizado"};
                {Value:"acero_inoxidable";  Label:"Acero inoxidable"};
                {Value:"acero_inox_316";    Label:"Acero inoxidable 316"};
                {Value:"fibra_vidrio";      Label:"Fibra de vidrio"};
                {Value:"poliester";         Label:"Poliéster"};
                {Value:"aluminio";          Label:"Aluminio"}
              )
            Value: =Label

      - ddColorGabinete:
          Control: DropDown
          Properties:
            Items: |
              =Table(
                {Value:"7035"; Label:"RAL 7035 (gris claro)"};
                {Value:"7016"; Label:"RAL 7016 (gris antracita)"};
                {Value:"9016"; Label:"RAL 9016 (blanco tráfico)"};
                {Value:"9005"; Label:"RAL 9005 (negro)"};
                {Value:"5010"; Label:"RAL 5010 (azul)"};
                {Value:"6005"; Label:"RAL 6005 (verde)"};
                {Value:"otro"; Label:"Otro"}
              )
            Value: =Label
            Default: ="7035"

      - ddTipoVentilacion:
          Control: DropDown
          Properties:
            Items: |
              =Table(
                {Value:"natural";     Label:"Natural"};
                {Value:"forzada";     Label:"Forzada (ventilador)"};
                {Value:"sellado";     Label:"Sellado (IP alto)"};
                {Value:"climatizado"; Label:"Climatizado (aire acondicionado)"}
              )
            Value: =Label

      - ddExpansionFutura:
          Control: DropDown
          Properties:
            Items: |
              =Table(
                {Value:"no";   Label:"Sin expansión"};
                {Value:"10";   Label:"10%"};
                {Value:"20";   Label:"20%"};
                {Value:"30";   Label:"30%"};
                {Value:"otro"; Label:"Otro"}
              )
            Value: =Label

      - txtObservacionesTablero:
          Control: TextInput
          Properties:
            Mode: =TextMode.MultiLine
            HintText: ="Observaciones del tablero"
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
entre `Collect` (tablero nuevo) y `Patch` (edición de uno existente).

```yaml
      - btnGuardarTablero:
          Control: Button
          Properties:
            Text: ="Guardar tablero"
            OnSelect: |
              =With(
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
opción una vez creadas las Choices).

```yaml
      - btnEnviar:
          Control: Button
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
  Si cambias una etiqueta allá, actualiza el `Switch` correspondiente aquí.
- Los nombres de columna citados (`'Tipo de Entrega (SolicitudTableros)'`, etc.)
  asumen que le diste esos **nombres para mostrar** a las columnas al crearlas.
  Si usaste otros nombres, ajusta las citas — el prefijo de editor (`csen_`) no
  se escribe en la fórmula, Power Fx resuelve por nombre para mostrar.
- Antes de conectar el botón **Enviar solicitud** a producción, prueba el
  recorrido completo en el entorno Developer: captura dos tableros con al
  menos un campo "otro" cada uno (para ejercitar todos los `Switch`), envía, y
  confirma en Dataverse (vista de tabla) que cada columna Choice quedó con la
  opción correcta y no en blanco.
