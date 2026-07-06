# YAML completo consolidado — para aplicar de una sola vez

Consolida en **un solo archivo** todo lo que ya está documentado por partes en
[00-...](../dataverse/00-construir-en-powerapps.md) (tablas/roles, aparte —
esto es solo la app Canvas), [05-canvas-yaml-captura.md](05-canvas-yaml-captura.md),
[02-canvas-guia-construccion.md](02-canvas-guia-construccion.md) y
[dataverse/05-construir-canvas-captura.md](../dataverse/05-construir-canvas-captura.md):
la app entera (`App` + 4 `Screens`) en formato **`.pa.yaml`** (el esquema de
código fuente actual de Power Apps, activo — no el formato retirado de
"preview").

## ⚠️ Léelo antes de pegar — qué sé con certeza y qué no

Investigué esto antes de escribirlo (no lo armé de memoria), y encontré una
contradicción real en la documentación de Microsoft que **no pude resolver
solo con búsqueda**:

- La función **"Ver código / Copiar código / Pegar código"** de Power Apps
  Studio (clic derecho en un control del árbol → Pegar) está documentada como
  activa (última actualización agosto 2025).
- Pero la página de esquemas de **`.pa.yaml`** dice que el formato que usaba
  esa función de copiar/pegar ("Preview") **está retirado y ya no se puede
  pegar código de ese formato**.

No tengo forma de confirmar cuál es el comportamiento real en tu Studio sin
probarlo ahí — puede que la función de pegado se haya actualizado para usar
este mismo esquema versionado (`Control: X@1.0.0`, el que uso abajo, igual al
de los ejemplos oficiales de cada control Modern), puede que no. **Por eso
este documento es best-effort:** el contenido (fórmulas, tipos de control,
propiedades) está verificado contra la documentación oficial de cada control;
lo que no puedo garantizar es que el **pegado masivo tal cual** funcione sin
ajustes en tu versión exacta de Studio.

### Cómo probarlo con el menor riesgo

1. **Empieza por lo más chico** — la sección "Pantalla de confirmación" al
   final (2 controles). Crea esa pantalla vacía, clic derecho en el nombre de
   la pantalla en el árbol (o en un área vacía del lienzo) → **Pegar**, con
   ese bloque en el portapapeles.
2. Si pega y crea los controles correctamente → sigue con las pantallas más
   grandes en el mismo documento.
3. Si falla (error de validación, o no pasa nada) → cae de vuelta a las guías
   clic por clic ya probadas: [dataverse/05](../dataverse/05-construir-canvas-captura.md)
   y [02](02-canvas-guia-construccion.md) — ese camino **sí** es 100% manual
   pero no depende de esta función ambigua.
4. **Vía alternativa confirmada como vigente:** si el pegado en Studio no
   funciona, la forma oficialmente soportada hoy de aplicar YAML de una vez es
   **Git integration** (conectar la solución a un repo Git — los archivos
   `Src/*.pa.yaml` se sincronizan al estudio) o **Power Platform CLI**
   (`pac canvas pack --sources ./Src --msapp Captura.msapp` y luego importar
   el `.msapp`). Ambas consumen exactamente este mismo contenido YAML, solo
   cambia cómo se lo entregas a Power Apps.

### Identificadores que no pude verificar al 100%

- **`GroupContainer`** y **`Gallery`** (controles clásicos, sin versión
  Modern) — los uso **sin sufijo `@versión`** porque la documentación dice
  que, si no especificas versión, Power Apps usa la más reciente
  automáticamente. Si Studio no reconoce el nombre exacto, créalos a mano
  (Insertar → Contenedor de grupo / Galería vertical clásica) y pega solo el
  contenido de `Properties`/`Children` dentro.
- Todo lo demás (`Modern*@1.0.0`, cada propiedad) **sí** está verificado
  contra la página oficial de ese control — ver Fuentes en
  [05-canvas-yaml-captura.md](05-canvas-yaml-captura.md#fuentes).

---

## App

```yaml
App:
  Properties:
    OnStart: |
      Set(varEditIndex; Blank());;
      ClearCollect(colTableros; []);;
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
```

> El código view/paste **no permite copiar/pegar el objeto App** (limitación
> documentada). Este bloque va directo en la propiedad `OnStart` de **App**
> desde el árbol (selecciona **App** arriba de todo → barra de fórmulas →
> `OnStart` → pega solo el contenido, sin la línea `App:`/`Properties:`/
> `OnStart: |`).

---

## Screens

```yaml
Screens:
  scrContactoProyecto:
    Properties:
      Fill: =RGBA(245; 246; 248; 1)
    Children:
      - navPestanas:
          Control: ModernTabList@1.0.0
          Properties:
            Items: =["Contacto y Proyecto"; "Tableros"; "Documentación"]
            Default: ="Contacto y Proyecto"
            Appearance: =TabListAppearance.Underline
            X: =0
            Y: =0
            Width: =Parent.Width
            OnChange: |
              Switch(Self.Selected.Value;
                "Contacto y Proyecto"; Navigate(scrContactoProyecto; ScreenTransition.None);
                "Tableros";            Navigate(scrTableroForm; ScreenTransition.None);
                Navigate(scrDocumentacion; ScreenTransition.None)
              )
      - lblTituloContacto:
          Control: ModernText@1.0.0
          Properties:
            Text: ="Contacto y Proyecto"
            X: =40
            Y: =92
            Width: =Parent.Width - 80
            Size: =20
            FontWeight: =FontWeight.Semibold
      - txtContactoNombre:
          Control: ModernTextInput@1.0.0
          Properties:
            Placeholder: ="Nombre del contacto *"
            X: =40
            Y: =148
            Width: =360
      - txtContactoEmail:
          Control: ModernTextInput@1.0.0
          Properties:
            Placeholder: ="Correo electrónico *"
            X: =420
            Y: =148
            Width: =360
      - lblEmailError:
          Control: ModernText@1.0.0
          Properties:
            Text: ="Correo con formato inválido"
            X: =420
            Y: =188
            Size: =9
            Color: =RGBA(239; 68; 68; 1)
            Visible: =And(Len(txtContactoEmail.Text) > 0; Not(IsMatch(txtContactoEmail.Text; Match.Email)))
      - txtContactoTelefono:
          Control: ModernTextInput@1.0.0
          Properties:
            Placeholder: ="Teléfono (+56)"
            X: =800
            Y: =148
            Width: =200
      - txtNombreProyecto:
          Control: ModernTextInput@1.0.0
          Properties:
            Placeholder: ="Nombre del proyecto/obra *"
            X: =40
            Y: =220
            Width: =360
      - txtEmpresaCliente:
          Control: ModernTextInput@1.0.0
          Properties:
            Placeholder: ="Empresa/Cliente"
            X: =420
            Y: =220
            Width: =360
      - txtUbicacion:
          Control: ModernTextInput@1.0.0
          Properties:
            Placeholder: ="Ubicación de la instalación"
            X: =40
            Y: =280
            Width: =360
      - txtCentroCosto:
          Control: ModernTextInput@1.0.0
          Properties:
            Placeholder: ="Centro de costo (MO-12345)"
            X: =420
            Y: =280
            Width: =360
      - dtpEntrega:
          Control: ModernDatePicker@1.0.0
          Properties:
            X: =800
            Y: =280
            Width: =200
      - ddIngenieriaPor:
          Control: ModernDropdown@1.0.0
          Properties:
            Items: =colOpcIngenieriaPor
            ItemDisplayText: =ThisItem.Label
            X: =40
            Y: =352
            Width: =500
      - btnSiguiente:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Siguiente"
            X: =800
            Y: =420
            Width: =200
            OnSelect: |
              If(
                Or(
                  IsBlank(txtContactoNombre.Text);
                  Not(IsMatch(txtContactoEmail.Text; Match.Email));
                  IsBlank(txtNombreProyecto.Text);
                  IsBlank(ddIngenieriaPor.Selected.Value)
                );
                Notify("Completa los campos obligatorios."; NotificationType.Error);
                Navigate(scrTableroForm; ScreenTransition.Cover)
              )

  scrTableroForm:
    Children:
      - navPestanas:
          Control: ModernTabList@1.0.0
          Properties:
            Items: =["Contacto y Proyecto"; "Tableros"; "Documentación"]
            Default: ="Tableros"
            Appearance: =TabListAppearance.Underline
            X: =0
            Y: =0
            Width: =Parent.Width
            OnChange: |
              Switch(Self.Selected.Value;
                "Contacto y Proyecto"; Navigate(scrContactoProyecto; ScreenTransition.None);
                "Tableros";            Navigate(scrTableroForm; ScreenTransition.None);
                Navigate(scrDocumentacion; ScreenTransition.None)
              )
      - lblModoEdicion:
          Control: ModernText@1.0.0
          Properties:
            Text: |
              If(IsBlank(varEditIndex); "Nuevo tablero"; "Editando: " & varEditIndex.Nombre)
            X: =Parent.Width * 0.3 + 16
            Y: =56
            Width: =Parent.Width * 0.7 - 32
            Size: =16
            FontWeight: =FontWeight.Semibold

      - cntGaleria:
          Control: GroupContainer
          Properties:
            X: =0
            Y: =92
            Width: =Parent.Width * 0.3
            Height: =Parent.Height - 92
          Children:
            - lblContadorTableros:
                Control: ModernText@1.0.0
                Properties:
                  Text: =CountRows(colTableros) & " tablero(s) agregado(s)"
                  X: =16
                  Y: =0
                  Width: =Parent.TemplateWidth - 32
            - lblSinTableros:
                Control: ModernText@1.0.0
                Properties:
                  Text: ="Aún no hay tableros en esta solicitud."
                  Visible: =CountRows(colTableros) = 0
                  X: =16
                  Y: =28
                  Width: =Parent.TemplateWidth - 32
            - btnEditarTablero:
                Control: ModernButton@1.0.0
                Properties:
                  Text: ="Editar"
                  DisplayMode: =If(IsBlank(galTableros.Selected); DisplayMode.Disabled; DisplayMode.Edit)
                  X: =16
                  Y: =60
                  Width: =120
                  Height: =32
                  OnSelect: |
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
            - btnEliminarTablero:
                Control: ModernButton@1.0.0
                Properties:
                  Text: ="Eliminar"
                  Appearance: =ButtonAppearance.Outline
                  DisplayMode: =If(IsBlank(galTableros.Selected); DisplayMode.Disabled; DisplayMode.Edit)
                  X: =144
                  Y: =60
                  Width: =120
                  Height: =32
                  OnSelect: |
                    UpdateContext({locItemAEliminar: galTableros.Selected; mostrarConfirmarEliminar: true})
            - galTableros:
                Control: Gallery
                Properties:
                  Items: =colTableros
                  TemplateSize: =64
                  TemplateFill: =If(ThisItem = galTableros.Selected; RGBA(99;102;241;0.15); RGBA(255;255;255;1))
                  X: =0
                  Y: =100
                  Width: =Parent.Width
                  Height: =Parent.Height - 180
                Children:
                  - lblNombreTablero:
                      Control: ModernText@1.0.0
                      Properties:
                        Text: =ThisItem.Nombre
                        X: =16
                        Y: =8
                        Width: =Parent.TemplateWidth - 32
                  - lblResumenTablero:
                      Control: ModernText@1.0.0
                      Properties:
                        Text: ="×" & ThisItem.Cantidad
                        X: =16
                        Y: =32
                        Width: =Parent.TemplateWidth - 32
                        Size: =10
            - btnAgregarTablero:
                Control: ModernButton@1.0.0
                Properties:
                  Text: ="+ Agregar Tablero"
                  X: =16
                  Y: =Parent.Height - 76
                  Width: =Parent.Width - 32
                  OnSelect: |
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
            - btnSiguienteTableros:
                Control: ModernButton@1.0.0
                Properties:
                  Text: ="Siguiente"
                  X: =16
                  Y: =Parent.Height - 36
                  Width: =Parent.Width - 32
                  OnSelect: |
                    If(
                      CountRows(colTableros) = 0;
                      Notify("Agrega al menos un tablero antes de continuar."; NotificationType.Error);
                      Navigate(scrDocumentacion; ScreenTransition.Cover)
                    )

      - cntFormulario:
          Control: GroupContainer
          Properties:
            X: =Parent.Width * 0.3
            Y: =92
            Width: =Parent.Width * 0.7
            Height: =Parent.Height - 92
          Children:
            # --- Identificación ---
            - ddTipoEntrega:
                Control: ModernDropdown@1.0.0
                Properties:
                  Items: =colOpcTipoEntrega
                  ItemDisplayText: =ThisItem.Label
                  Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTipoEntrega; Value = varEditIndex.TipoEntrega))
                  X: =16
                  Y: =0
                  Width: =400
            - ddInstalacionNuevaReemplazo:
                Control: ModernDropdown@1.0.0
                Properties:
                  Items: =colOpcInstalacionNuevaReemplazo
                  ItemDisplayText: =ThisItem.Label
                  Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcInstalacionNuevaReemplazo; Value = varEditIndex.InstalacionNuevaReemplazo))
                  X: =432
                  Y: =0
                  Width: =400
            - txtNombreTablero:
                Control: ModernTextInput@1.0.0
                Properties:
                  Placeholder: ="Nombre del tablero * (Ej.: TG Principal)"
                  Default: =If(IsBlank(varEditIndex); ""; varEditIndex.Nombre)
                  X: =16
                  Y: =44
                  Width: =400
            - numCantidad:
                Control: ModernNumberInput@1.0.0
                Properties:
                  HintText: ="Cantidad"
                  Default: =If(IsBlank(varEditIndex); 1; varEditIndex.Cantidad)
                  Min: =1
                  Max: =999
                  Precision: =DecimalPrecision.'0'
                  X: =432
                  Y: =44
                  Width: =200
            - ddTipoTablero:
                Control: ModernDropdown@1.0.0
                Properties:
                  Visible: =ddTipoEntrega.Selected.Value = "tablero"
                  Items: =colOpcTipoTablero
                  ItemDisplayText: =ThisItem.Label
                  Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTipoTablero; Value = varEditIndex.TipoTablero))
                  X: =16
                  Y: =88
                  Width: =400
            - txtOtroTipoTablero:
                Control: ModernTextInput@1.0.0
                Properties:
                  Visible: =ddTipoTablero.Selected.Value = "otro"
                  Placeholder: ="Especifica el tipo de tablero"
                  Default: =If(IsBlank(varEditIndex); ""; varEditIndex.OtroTipoTablero)
                  X: =432
                  Y: =88
                  Width: =400
            - txtFuncionTablero:
                Control: ModernTextInput@1.0.0
                Properties:
                  Type: =TextInputType.Multiline
                  Placeholder: ="Función del tablero"
                  Default: =If(IsBlank(varEditIndex); ""; varEditIndex.FuncionTablero)
                  X: =16
                  Y: =132
                  Width: =816
                  Height: =64
            - txtCargasAAlimentar:
                Control: ModernTextInput@1.0.0
                Properties:
                  Type: =TextInputType.Multiline
                  Placeholder: ="Cargas a alimentar"
                  Default: =If(IsBlank(varEditIndex); ""; varEditIndex.CargasAAlimentar)
                  X: =16
                  Y: =204
                  Width: =816
                  Height: =64
            - numNumeroCircuitos:
                Control: ModernNumberInput@1.0.0
                Properties:
                  HintText: ="Número de circuitos"
                  Default: =If(IsBlank(varEditIndex); 0; varEditIndex.NumeroCircuitos)
                  Min: =0
                  Precision: =DecimalPrecision.'0'
                  X: =16
                  Y: =276
                  Width: =200

            # --- Ubicación, ambiente y montaje ---
            - ddUbicacion:
                Control: ModernDropdown@1.0.0
                Properties:
                  Items: =colOpcUbicacion
                  ItemDisplayText: =ThisItem.Label
                  Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcUbicacion; Value = varEditIndex.Ubicacion))
                  X: =16
                  Y: =324
                  Width: =400
            - cmbAmbienteEspecial:
                Control: ModernCombobox@1.0.0
                Properties:
                  Items: =colOpcAmbienteEspecial
                  ItemDisplayText: =ThisItem.Label
                  SelectMultiple: =true
                  DefaultSelectedItems: =If(IsBlank(varEditIndex); Table(); varEditIndex.AmbienteEspecial)
                  X: =432
                  Y: =324
                  Width: =400
            - txtOtroAmbiente:
                Control: ModernTextInput@1.0.0
                Properties:
                  Visible: ="otro" in cmbAmbienteEspecial.SelectedItems.Value
                  Placeholder: ="Especifica el ambiente especial"
                  Default: =If(IsBlank(varEditIndex); ""; varEditIndex.OtroAmbienteEspecial)
                  X: =16
                  Y: =368
                  Width: =400
            - ddGradoIP:
                Control: ModernDropdown@1.0.0
                Properties:
                  Items: |
                    Switch(ddUbicacion.Selected.Value;
                      "interior"; ["IP20";"IP31";"IP43";"IP54";"IP55";"IP65"];
                      "exterior"; ["IP54";"IP55";"IP65";"IP66";"IP67";"IP68"];
                      ["IP20";"IP31";"IP43";"IP54";"IP55";"IP65";"IP66";"IP67";"IP68"]
                    )
                  Default: =If(IsBlank(varEditIndex); Blank(); {Value: varEditIndex.GradoIP})
                  X: =432
                  Y: =368
                  Width: =200
            - ddGradoIK:
                Control: ModernDropdown@1.0.0
                Properties:
                  # IK06=1J, IK07=2J, IK08=5J, IK09=10J, IK10=20J (IEC 62262)
                  Items: |
                    ["IK06";"IK07";"IK08";"IK09";"IK10"]
                  Default: =If(IsBlank(varEditIndex); Blank(); {Value: varEditIndex.GradoIK})
                  X: =648
                  Y: =368
                  Width: =184
            - ddTipoMontaje:
                Control: ModernDropdown@1.0.0
                Properties:
                  Items: =colOpcTipoMontaje
                  ItemDisplayText: =ThisItem.Label
                  Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTipoMontaje; Value = varEditIndex.TipoMontaje))
                  X: =16
                  Y: =412
                  Width: =400
            - tglRestricciones:
                Control: ModernToggle@1.0.0
                Properties:
                  Label: ="¿Tiene restricciones de dimensión?"
                  Checked: =If(IsBlank(varEditIndex); false; varEditIndex.RestriccionesDimension)
                  X: =432
                  Y: =412
                  Width: =300
            - numAltoMax:
                Control: ModernNumberInput@1.0.0
                Properties:
                  Visible: =tglRestricciones.Checked
                  HintText: ="Alto máximo (mm)"
                  Default: =If(IsBlank(varEditIndex); 0; varEditIndex.AltoMaxMm)
                  Min: =0
                  Precision: =DecimalPrecision.'0'
                  X: =16
                  Y: =456
                  Width: =200
            - numAnchoMax:
                Control: ModernNumberInput@1.0.0
                Properties:
                  Visible: =tglRestricciones.Checked
                  HintText: ="Ancho máximo (mm)"
                  Default: =If(IsBlank(varEditIndex); 0; varEditIndex.AnchoMaxMm)
                  Min: =0
                  Precision: =DecimalPrecision.'0'
                  X: =232
                  Y: =456
                  Width: =200
            - numFondoMax:
                Control: ModernNumberInput@1.0.0
                Properties:
                  Visible: =tglRestricciones.Checked
                  HintText: ="Fondo máximo (mm)"
                  Default: =If(IsBlank(varEditIndex); 0; varEditIndex.FondoMaxMm)
                  Min: =0
                  Precision: =DecimalPrecision.'0'
                  X: =448
                  Y: =456
                  Width: =200
            - txtCondicionesInstalacion:
                Control: ModernTextInput@1.0.0
                Properties:
                  Type: =TextInputType.Multiline
                  Placeholder: ="Condiciones adicionales de instalación"
                  Default: =If(IsBlank(varEditIndex); ""; varEditIndex.CondicionesInstalacion)
                  X: =16
                  Y: =500
                  Width: =816
                  Height: =64

            # --- Parámetros eléctricos ---
            - ddTension:
                Control: ModernDropdown@1.0.0
                Properties:
                  Items: =colOpcTension
                  ItemDisplayText: =ThisItem.Label
                  Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTension; Value = varEditIndex.TensionSuministro))
                  X: =16
                  Y: =576
                  Width: =300
            - numOtraTension:
                Control: ModernNumberInput@1.0.0
                Properties:
                  Visible: =ddTension.Selected.Value = "otro"
                  HintText: ="Tensión (V)"
                  Default: =If(IsBlank(varEditIndex); 0; varEditIndex.OtraTension)
                  Min: =0
                  X: =332
                  Y: =576
                  Width: =200
            - ddSistema:
                Control: ModernDropdown@1.0.0
                Properties:
                  Items: =colOpcSistema
                  ItemDisplayText: =ThisItem.Label
                  Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcSistema; Value = varEditIndex.SistemaElectrico))
                  X: =548
                  Y: =576
                  Width: =284
            - txtOtroSistema:
                Control: ModernTextInput@1.0.0
                Properties:
                  Visible: =ddSistema.Selected.Value = "otro"
                  Placeholder: ="Especifica el sistema eléctrico"
                  Default: =If(IsBlank(varEditIndex); ""; varEditIndex.OtroSistemaElectrico)
                  X: =16
                  Y: =620
                  Width: =400
            - numPotencia:
                Control: ModernNumberInput@1.0.0
                Properties:
                  HintText: ="Potencia estimada"
                  Default: =If(IsBlank(varEditIndex); Blank(); varEditIndex.PotenciaEstimada)
                  Min: =0
                  Precision: =DecimalPrecision.'2'
                  X: =432
                  Y: =620
                  Width: =200
            - ddUnidadPotencia:
                Control: ModernDropdown@1.0.0
                Properties:
                  Items: =colOpcUnidadPotencia
                  ItemDisplayText: =ThisItem.Label
                  Default: =If(IsBlank(varEditIndex); LookUp(colOpcUnidadPotencia; Value="kW"); LookUp(colOpcUnidadPotencia; Value = varEditIndex.UnidadPotencia))
                  X: =648
                  Y: =620
                  Width: =184
            - lblCorriente:
                Control: ModernText@1.0.0
                Properties:
                  Text: |
                    With(
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
                  X: =16
                  Y: =664
                  Width: =400
            - ddFrecuencia:
                Control: ModernDropdown@1.0.0
                Properties:
                  Items: =colOpcFrecuencia
                  ItemDisplayText: =ThisItem.Label
                  Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcFrecuencia; Value = varEditIndex.Frecuencia))
                  X: =432
                  Y: =664
                  Width: =200
            - numOtraFrecuencia:
                Control: ModernNumberInput@1.0.0
                Properties:
                  Visible: =ddFrecuencia.Selected.Value = "otro"
                  HintText: ="Frecuencia (Hz)"
                  Default: =If(IsBlank(varEditIndex); 0; varEditIndex.OtraFrecuencia)
                  Min: =0
                  X: =648
                  Y: =664
                  Width: =184
            - cmbProteccionesRequeridas:
                Control: ModernCombobox@1.0.0
                Properties:
                  Items: =colOpcProteccionesRequeridas
                  ItemDisplayText: =ThisItem.Label
                  SelectMultiple: =true
                  DefaultSelectedItems: =If(IsBlank(varEditIndex); Table(); varEditIndex.ProteccionesRequeridas)
                  X: =16
                  Y: =708
                  Width: =400
            - cmbMarcasPreferidas:
                Control: ModernCombobox@1.0.0
                Properties:
                  Items: =colOpcMarcasPreferidas
                  ItemDisplayText: =ThisItem.Label
                  SelectMultiple: =true
                  DefaultSelectedItems: =If(IsBlank(varEditIndex); Table(); varEditIndex.MarcasPreferidas)
                  X: =432
                  Y: =708
                  Width: =400

            # --- Diseño constructivo ---
            - ddMaterialGabinete:
                Control: ModernDropdown@1.0.0
                Properties:
                  Items: =colOpcMaterialGabinete
                  ItemDisplayText: =ThisItem.Label
                  Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcMaterialGabinete; Value = varEditIndex.MaterialGabinete))
                  X: =16
                  Y: =756
                  Width: =400
            - ddColorGabinete:
                Control: ModernDropdown@1.0.0
                Properties:
                  Items: =colOpcColorGabinete
                  ItemDisplayText: =ThisItem.Label
                  Default: =If(IsBlank(varEditIndex); LookUp(colOpcColorGabinete; Value="7035"); LookUp(colOpcColorGabinete; Value = varEditIndex.ColorGabinete))
                  X: =432
                  Y: =756
                  Width: =400
            - ddTipoVentilacion:
                Control: ModernDropdown@1.0.0
                Properties:
                  Items: =colOpcTipoVentilacion
                  ItemDisplayText: =ThisItem.Label
                  Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTipoVentilacion; Value = varEditIndex.TipoVentilacion))
                  X: =16
                  Y: =800
                  Width: =400
            - ddExpansionFutura:
                Control: ModernDropdown@1.0.0
                Properties:
                  Items: =colOpcExpansionFutura
                  ItemDisplayText: =ThisItem.Label
                  Default: =If(IsBlank(varEditIndex); Blank(); LookUp(colOpcExpansionFutura; Value = varEditIndex.ExpansionFutura))
                  X: =432
                  Y: =800
                  Width: =400
            - txtObservacionesTablero:
                Control: ModernTextInput@1.0.0
                Properties:
                  Type: =TextInputType.Multiline
                  Placeholder: ="Observaciones del tablero"
                  Default: =If(IsBlank(varEditIndex); ""; varEditIndex.ObservacionesTablero)
                  X: =16
                  Y: =844
                  Width: =816
                  Height: =64

            # --- Validación + Guardar ---
            - btnGuardarTablero:
                Control: ModernButton@1.0.0
                Properties:
                  Text: ="Guardar tablero"
                  X: =16
                  Y: =924
                  Width: =816
                  OnSelect: |
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
                    )

      - grpConfirmarEliminar:
          Control: GroupContainer
          Properties:
            Visible: =mostrarConfirmarEliminar
            X: =Parent.Width * 0.5 - 160
            Y: =Parent.Height * 0.5 - 60
            Width: =320
            Height: =120
            Fill: =RGBA(255;255;255;1)
            BorderColor: =RGBA(0;0;0;0.2)
          Children:
            - lblConfirmarEliminar:
                Control: ModernText@1.0.0
                Properties:
                  Text: ="¿Eliminar este tablero?"
                  X: =16
                  Y: =16
                  Width: =288
            - btnConfirmarSi:
                Control: ModernButton@1.0.0
                Properties:
                  Text: ="Sí, eliminar"
                  Appearance: =ButtonAppearance.Primary
                  X: =16
                  Y: =64
                  Width: =136
                  OnSelect: |
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
            - btnConfirmarNo:
                Control: ModernButton@1.0.0
                Properties:
                  Text: ="Cancelar"
                  Appearance: =ButtonAppearance.Outline
                  X: =168
                  Y: =64
                  Width: =136
                  OnSelect: |
                    UpdateContext({mostrarConfirmarEliminar: false})

  scrDocumentacion:
    Children:
      - navPestanas:
          Control: ModernTabList@1.0.0
          Properties:
            Items: =["Contacto y Proyecto"; "Tableros"; "Documentación"]
            Default: ="Documentación"
            Appearance: =TabListAppearance.Underline
            X: =0
            Y: =0
            Width: =Parent.Width
            OnChange: |
              Switch(Self.Selected.Value;
                "Contacto y Proyecto"; Navigate(scrContactoProyecto; ScreenTransition.None);
                "Tableros";            Navigate(scrTableroForm; ScreenTransition.None);
                Navigate(scrDocumentacion; ScreenTransition.None)
              )
      - txtObservacionesProyecto:
          Control: ModernTextInput@1.0.0
          Properties:
            Type: =TextInputType.Multiline
            Placeholder: ="Observaciones del proyecto"
            X: =40
            Y: =100
            Width: =800
            Height: =160
      - btnEnviar:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Enviar solicitud"
            X: =40
            Y: =280
            Width: =240
            OnSelect: |
              If(CountRows(colTableros) = 0;
                Notify("Debe agregar al menos un tablero."; NotificationType.Error);
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
                ForAll(colTableros As t;
                  Patch(SolicitudTableros; Defaults(SolicitudTableros); {
                    Solicitud: varSolicitud;
                    Nombre: t.Nombre;
                    Cantidad: t.Cantidad;
                    Orden: t.Orden;
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

  scrConfirmacion:
    Children:
      - lblConfirmacion:
          Control: ModernText@1.0.0
          Properties:
            Text: ="Solicitud enviada correctamente. Código: " & varSolicitud.ReferenceCode
            X: =40
            Y: =40
            Width: =600
            Size: =18
      - btnNuevaSolicitud:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Registrar otra solicitud"
            X: =40
            Y: =100
            Width: =240
            OnSelect: =Navigate(scrContactoProyecto)
```

---

## Notas finales

- Las coordenadas `X`/`Y`/`Width`/`Height` son un punto de partida razonable,
  no un layout final — muévelas visualmente en Studio después de pegar, es
  más rápido que ajustarlas a mano en YAML.
- `Solicitud`/`ContactoNombre`/etc. de `scrDocumentacion` referencian
  controles de `scrContactoProyecto` (`txtNombreProyecto`, `dtpEntrega`,
  etc.) — Power Fx puede referenciar controles de **cualquier pantalla**
  dentro de la misma app, así que `btnEnviar` funciona estando en
  `scrDocumentacion` aunque esos controles vivan en la pantalla 1. Esto ya lo
  usaba el diseño original, no es nuevo de este documento.
- Si el pegado masivo falla en algún punto puntual (por ejemplo, un control
  Modern específico no se reconoce), no reinicies desde cero: identifica cuál
  control falló y créalo a mano siguiendo
  [02-canvas-guia-construccion.md](02-canvas-guia-construccion.md) o
  [05-canvas-yaml-captura.md](05-canvas-yaml-captura.md) para ese campo en
  particular — el resto del pegado ya construido queda intacto.
