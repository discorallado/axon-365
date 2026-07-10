# YAML completo consolidado — para aplicar de una sola vez

Consolida en **un solo archivo** todo lo que ya está documentado por partes en
[00-...](../dataverse/00-construir-en-powerapps.md) (tablas/roles, aparte —
esto es solo la app Canvas), [05-canvas-yaml-captura.md](05-canvas-yaml-captura.md) y
[dataverse/05-construir-canvas-captura.md](../dataverse/05-construir-canvas-captura.md):
la app entera (`App` + 8 `Screens`) en formato **`.pa.yaml`** (el esquema de
código fuente actual de Power Apps, activo — no el formato retirado de
"preview").

## Arquitectura de pantallas (revisión de esta versión)

La versión anterior metía las ~37 columnas del tablero en **un solo panel**
(`cntFormulario`) al lado de la galería — visualmente muy denso para
trabajar en el editor. Esta versión separa el formulario del tablero en
**sus propias 4 pantallas** (una por grupo de campos), con Atrás/Siguiente,
igual que un wizard. La galería vuelve a ser una pantalla propia
(`scrTableros`), de ancho completo.

```
scrContactoProyecto ─Siguiente→ scrTableros ─Agregar/Editar→ scrTableroForm (1/4)
       ↑                            ↑  ↑                           │ Siguiente
       └──────── pestañas ──────────┘  └────Guardar tablero────scrTableroForm2a (2/4)
                                                                    │ Siguiente
                                              scrTableroForm3 (4/4) ← scrTableroForm2b (3/4)
                                                    │ Siguiente (scrTableros)
scrTableros ─Siguiente→ scrDocumentacion ─Enviar→ scrConfirmacion
```

- **Pestañas principales** (`ModernTabList`, `navPestanas`) en las 3 pantallas
  "principales" (`scrContactoProyecto`, `scrTableros`, `scrDocumentacion`)
  **y también** en las 4 pantallas del tablero (siempre marcando "Tableros"
  como activa) — mismo control en las 7 pantallas, solo cambia el `Default`
  (y el `DisplayMode`, ver abajo).
- Las **4 pantallas del tablero** (`scrTableroForm`, `2a`, `2b`, `3`) llevan
  **doble barra apilada**, con roles distintos:
  - **`navPestanas`** (arriba, Y=0) es **solo indicativa**
    (`DisplayMode: =DisplayMode.View` — sin clic, sin hover): siempre marca
    "Tableros", pero no navega. Salir del sub-flujo del tablero hacia otra
    sección abandonaría un tablero a medio llenar sin aviso, así que esa
    salida solo se hace por **Cancelar** (con confirmación).
  - **`navSubPasos`** (debajo, Y=40) **sí es clickeable**, con el mismo patrón
    de gateo que `navPestanas` pero con su propia variable
    (`varPasoMaximoTablero`, 1 a 4): retroceder a un paso ya completado
    navega libre; saltar adelante sin haber validado el paso anterior muestra
    un aviso y no navega. `varPasoMaximoTablero` se resetea a `1` en
    **Agregar Tablero** (nuevo, arranca en el paso 1) y a `4` en **Editar**
    (tablero ya guardado, se asume completo → navegación libre entre sus 4
    pasos); cada **Siguiente** lo sube con `Max(...)` al validar su paso.
  Se entra desde `scrTableros` (Agregar o Editar) y se sale por `scrTableros`
  (Guardar tablero en el paso 4, o **Cancelar** con confirmación en cualquier
  paso).
- Los valores de los controles **se conservan** al navegar entre las 4
  pantallas del tablero — Power Apps no destruye los controles de pantallas
  no visibles, así que un control del paso 4 puede leer uno del paso 1, y no
  hace falta guardar nada en variables intermedias.

### Capa a prueba de tontos (foolproof)

Difícil equivocarse: (1) fórmulas con nombre en **`App.Formulas`**
(`pasoIdentificacionOK`…`tableroCompleto`) — ver la sección **App — Formulas**;
(2) **banner de pendientes** (`lblPendientes`) en cada paso, en vivo, que dice
qué etapas faltan y se pone verde al completar; (3) **error por paso**
(`lblErroresPaso`) que, tras un intento fallido (`varValidarTablero`), lista en
rojo los campos que faltan **en esa pantalla**; (4) cada **Siguiente** valida su
paso y enciende los rojos; (5) el **Guardar** revalida todo, enciende los rojos
y **navega al primer paso incompleto**; (6) **Cancelar** (y el Atrás del paso 1,
que sale a la galería) piden **confirmación** para no perder un tablero a medio
llenar (`grpConfirmarCancelar`); (7) `ModernNumberInput` (Min/Max) impide teclear
cantidades/dimensiones fuera de rango. Así no se puede **guardar** un tablero
incompleto ni **perder** uno sin querer.

## ⚠️ Léelo antes de pegar

Ya probaste el pegado masivo en tu Studio y **funcionó** — encontraste
versiones concretas de control (`ModernDropdown@1.0.2`, `ModernTextInput@1.1.1`,
`ModernNumberInput@1.1.1`, `ModernCombobox@1.1.1`, `Gallery@2.15.0` +
`Variant`, `GroupContainer` + `Variant: "ManualLayout"`) que Studio aceptó;
las mantengo tal cual en este documento en vez de mis versiones genéricas
`@1.0.0` anteriores, porque las tuyas están confirmadas contra tu entorno
real. Si al pegar alguna pantalla nueva Studio pide una versión distinta,
usa la que él te ofrezca — es información más confiable que la de este doc.

- **`GroupContainer`** (`Variant: "ManualLayout"`) se usa para el popup de
  confirmar eliminación (`grpConfirmarEliminar`, en `scrTableros`) y para el de
  confirmar descartar el tablero (`grpConfirmarCancelar`, uno por cada una de
  las 4 pantallas del tablero).
- **`navSubPasos`** es otro `ModernTabList@1.0.0` (una copia por pantalla del
  tablero; solo cambia el `Default`). Si Studio pide otra versión al pegarlo,
  usa la que ofrezca.
- **`App.Formulas`** (fórmulas con nombre) debe pegarse aparte de `OnStart`
  (sección **App — Formulas**). Si tu entorno no las tiene habilitadas, pega
  cada expresión inline donde se usa (banner, Guardar).
- El resto de identificadores (`Modern*`, cada propiedad) está verificado
  contra la página oficial de ese control — ver Fuentes en
  [05-canvas-yaml-captura.md](05-canvas-yaml-captura.md#fuentes).

---

## App

```yaml
App:
  Properties:
    OnStart: |
      Set(varEditIndex; Blank());;
      Set(varValidarTablero; false);;
      // Items de navPestanas centralizados: una sola definición para las 7
      // copias del control (3 pantallas principales + 4 del tablero) en vez
      // de repetir el array literal en cada una.
      Set(varNavPestanas; ["Contacto y Proyecto"; "Tableros"; "Documentación"]);;
      // Pestaña más lejana ya alcanzada válidamente (1=Contacto, 2=Tableros,
      // 3=Documentación). Gatea el avance por pestañas: hacia atrás siempre
      // se permite; hacia adelante solo hasta este número.
      Set(varPasoMaximo; 1);;
      // Igual que varPasoMaximo, pero para las sub-pestañas (navSubPasos)
      // dentro del sub-flujo del tablero (1=Identificación...4=Constructivo).
      // Se resetea a 1 en btnAgregarTablero (tablero nuevo) y a 4 en
      // btnEditarTablero (tablero ya guardado, se asume completo → navegación
      // libre entre sus 4 pasos).
      Set(varPasoMaximoTablero; 1);;
      // Contador para TableroId: identifica cada tablero de forma única y
      // estable (a diferencia de Orden, nunca se reutiliza tras un Eliminar).
      // Necesario porque AmbienteEspecial/ProteccionesRequeridas/MarcasPreferidas
      // son tablas anidadas dentro de la fila, y Power Fx no puede comparar
      // registros completos por igualdad ni hacer Remove/Patch por registro
      // cuando contienen columnas de tipo tabla — hay que identificar cada
      // fila por un campo simple (Number) en vez de por el registro entero.
      Set(varNextTableroId; 1);;
      // colTableros vacía PERO tipada: define todas las columnas y sus tipos
      // (esquema de cada fila = esquema de varEditIndex al editar), para que
      // varEditIndex.Campo y los IsBlank/Default no den "columna inexistente"
      // antes de agregar el primer tablero. Filter(...; false) = 0 filas, con esquema.
      ClearCollect(colTableros; Filter(Table({
        Nombre:                    "";
        Cantidad:                  0;
        TableroId:                 0;
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
      // varEditIndex tipado (queda Blank, pero con el esquema de la fila de colTableros)
      // para que varEditIndex.Campo resuelva; IsBlank(varEditIndex) sigue siendo true.
      Set(varEditIndex; First(colTableros));;
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
      ));;
      ClearCollect(colOpcSiNo; Table(
        {Value: true;  Label: "Sí"};
        {Value: false; Label: "No"}
      ))
```

> El código view/paste **no permite copiar/pegar el objeto App** (limitación
> documentada). Este bloque va directo en la propiedad `OnStart` de **App**
> desde el árbol (selecciona **App** arriba de todo → barra de fórmulas →
> `OnStart` → pega solo el contenido, sin la línea `App:`/`Properties:`/
> `OnStart: |`).

---

## App — Formulas (fórmulas con nombre)  ← capa a prueba de tontos

Selecciona **App** en el árbol → propiedad **Formulas** → pega esto. Son la
única fuente de verdad de "¿está completa cada etapa?" y las reusan el banner
de pendientes, el botón Guardar y el salto al primer paso incompleto. (Si tu
entorno no tiene *Named formulas*, pega cada expresión inline donde se usa.)

```powerfx
pasoIdentificacionOK =
    And(
        Not(IsBlank(txtNombreTablero.Text));
        Not(IsBlank(ddTipoEntrega.Selected.Value));
        Not(IsBlank(ddInstalacionNuevaReemplazo.Selected.Value));
        Or(ddTipoEntrega.Selected.Value <> "tablero"; Not(IsBlank(ddTipoTablero.Selected.Value)))
    );
pasoUbicacionOK =
    And(
        Not(IsBlank(ddUbicacion.Selected.Value));
        Not(IsBlank(ddGradoIP.Selected.Value));
        Not(IsBlank(ddTipoMontaje.Selected.Value))
    );
pasoElectricoOK =
    And(
        Not(IsBlank(ddTension.Selected.Value));
        Not(IsBlank(ddSistema.Selected.Value));
        numPotencia.Value > 0;
        Not(IsBlank(ddFrecuencia.Selected.Value));
        CountRows(cmbProteccionesRequeridas.SelectedItems) > 0
    );
pasoConstructivoOK =
    And(
        Not(IsBlank(ddMaterialGabinete.Selected.Value));
        Not(IsBlank(ddTipoVentilacion.Selected.Value));
        Not(IsBlank(ddExpansionFutura.Selected.Value))
    );
tableroCompleto =
    And(pasoIdentificacionOK; pasoUbicacionOK; pasoElectricoOK; pasoConstructivoOK)
```

---

## El bloque `Reset()` — se repite igual en 4 lugares

Los ~35 controles del formulario del tablero ahora viven repartidos en 4
pantallas distintas, pero `Reset(control)` funciona **sin importar en qué
pantalla esté** ese control (Power Apps no destruye los controles de
pantallas no visibles). Este bloque se pega tal cual en:
`btnAgregarTablero` y `btnEditarTablero` (en `scrTableros`),
`btnGuardarTablero` (en `scrTableroForm3`) y `btnConfirmarSi` (popup eliminar,
en `scrTableros`):

```
Reset(ddTipoEntrega);; Reset(ddInstalacionNuevaReemplazo);; Reset(txtNombreTablero);;
Reset(numCantidad);; Reset(ddTipoTablero);; Reset(txtOtroTipoTablero);;
Reset(txtFuncionTablero);; Reset(txtCargasAAlimentar);; Reset(numNumeroCircuitos);;
Reset(ddUbicacion);; Reset(cmbAmbienteEspecial);; Reset(txtOtroAmbiente);;
Reset(ddGradoIP);; Reset(ddGradoIK);; Reset(ddTipoMontaje);; Reset(ddRestricciones);;
Reset(numAltoMax);; Reset(numAnchoMax);; Reset(numFondoMax);; Reset(txtCondicionesInstalacion);;
Reset(ddTension);; Reset(numOtraTension);; Reset(ddSistema);; Reset(txtOtroSistema);;
Reset(numPotencia);; Reset(ddUnidadPotencia);; Reset(ddFrecuencia);; Reset(numOtraFrecuencia);;
Reset(cmbProteccionesRequeridas);; Reset(cmbMarcasPreferidas);; Reset(ddMaterialGabinete);;
Reset(ddColorGabinete);; Reset(ddTipoVentilacion);; Reset(ddExpansionFutura);;
Reset(txtObservacionesTablero)
```

(Ya aparece completo, con `;;` al final para encadenar con lo que sigue, en
cada `OnSelect` de abajo — esta sección solo lo explica una vez.)

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
            Items: =varNavPestanas
            Default: ="Contacto y Proyecto"
            Appearance: =TabListAppearance.Underline
            X: =0
            Y: =0
            Width: =Parent.Width
            OnChange: |
              Switch(
                Self.Selected.Value;
                "Contacto y Proyecto"; Navigate(scrContactoProyecto; ScreenTransition.None);
                "Tableros";
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
          Control: ModernTextInput@1.1.1
          Properties:
            Placeholder: ="Nombre del contacto *"
            X: =40
            Y: =148
            Width: =360
      - txtContactoEmail:
          Control: ModernTextInput@1.1.1
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
          Control: ModernTextInput@1.1.1
          Properties:
            Placeholder: ="Teléfono (+56)"
            X: =800
            Y: =148
            Width: =200
      - txtNombreProyecto:
          Control: ModernTextInput@1.1.1
          Properties:
            Placeholder: ="Nombre del proyecto/obra *"
            X: =40
            Y: =220
            Width: =360
      - txtEmpresaCliente:
          Control: ModernTextInput@1.1.1
          Properties:
            Placeholder: ="Empresa/Cliente"
            X: =420
            Y: =220
            Width: =360
      - txtUbicacion:
          Control: ModernTextInput@1.1.1
          Properties:
            Placeholder: ="Ubicación de la instalación"
            X: =40
            Y: =280
            Width: =360
      - txtCentroCosto:
          Control: ModernTextInput@1.1.1
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
          Control: ModernDropdown@1.0.2
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
                Set(varPasoMaximo; Max(varPasoMaximo; 2));;
                Navigate(scrTableros; ScreenTransition.Cover)
              )

  scrTableros:
    Children:
      - navPestanas:
          Control: ModernTabList@1.0.0
          Properties:
            Items: =varNavPestanas
            Default: ="Tableros"
            Appearance: =TabListAppearance.Underline
            X: =0
            Y: =0
            Width: =Parent.Width
            OnChange: |
              Switch(
                Self.Selected.Value;
                "Contacto y Proyecto"; Navigate(scrContactoProyecto; ScreenTransition.None);
                "Tableros";
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
      - lblTituloTableros:
          Control: ModernText@1.0.0
          Properties:
            Text: ="Tableros de la solicitud"
            X: =40
            Y: =92
            Width: =Parent.Width - 80
            Size: =20
            FontWeight: =FontWeight.Semibold
      - lblContadorTableros:
          Control: ModernText@1.0.0
          Properties:
            Text: =CountRows(colTableros) & " tablero(s) agregado(s)"
            X: =40
            Y: =136
            Width: =Parent.Width - 80
      - lblSinTableros:
          Control: ModernText@1.0.0
          Properties:
            Text: ="Aún no hay tableros en esta solicitud."
            Visible: =CountRows(colTableros) = 0
            X: =40
            Y: =164
            Width: =Parent.Width - 80
      - btnEditarTablero:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Editar"
            DisplayMode: =If(IsBlank(galTableros.Selected); DisplayMode.Disabled; DisplayMode.Edit)
            X: =40
            Y: =196
            Width: =120
            Height: =32
            OnSelect: |
              Set(varEditIndex; galTableros.Selected);;
              Set(varValidarTablero; false);;
              Set(varPasoMaximoTablero; 4);;
              Reset(ddTipoEntrega);; Reset(ddInstalacionNuevaReemplazo);; Reset(txtNombreTablero);;
              Reset(numCantidad);; Reset(ddTipoTablero);; Reset(txtOtroTipoTablero);;
              Reset(txtFuncionTablero);; Reset(txtCargasAAlimentar);; Reset(numNumeroCircuitos);;
              Reset(ddUbicacion);; Reset(cmbAmbienteEspecial);; Reset(txtOtroAmbiente);;
              Reset(ddGradoIP);; Reset(ddGradoIK);; Reset(ddTipoMontaje);; Reset(ddRestricciones);;
              Reset(numAltoMax);; Reset(numAnchoMax);; Reset(numFondoMax);; Reset(txtCondicionesInstalacion);;
              Reset(ddTension);; Reset(numOtraTension);; Reset(ddSistema);; Reset(txtOtroSistema);;
              Reset(numPotencia);; Reset(ddUnidadPotencia);; Reset(ddFrecuencia);; Reset(numOtraFrecuencia);;
              Reset(cmbProteccionesRequeridas);; Reset(cmbMarcasPreferidas);; Reset(ddMaterialGabinete);;
              Reset(ddColorGabinete);; Reset(ddTipoVentilacion);; Reset(ddExpansionFutura);;
              Reset(txtObservacionesTablero);;
              Navigate(scrTableroForm; ScreenTransition.Cover)
      - btnEliminarTablero:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Eliminar"
            Appearance: =ButtonAppearance.Outline
            DisplayMode: =If(IsBlank(galTableros.Selected); DisplayMode.Disabled; DisplayMode.Edit)
            X: =168
            Y: =196
            Width: =120
            Height: =32
            OnSelect: |
              UpdateContext({locItemAEliminar: galTableros.Selected.TableroId; mostrarConfirmarEliminar: true})
      - galTableros:
          Control: Gallery@2.15.0
          Variant: BrowseLayout_Vertical_TwoTextOneImageVariant_ver5.0
          Properties:
            Items: =colTableros
            TemplateSize: =64
            TemplateFill: =If(ThisItem.TableroId = galTableros.Selected.TableroId; RGBA(99;102;241;0.15); RGBA(255;255;255;1))
            X: =40
            Y: =240
            Width: =Parent.Width - 80
            Height: =Parent.Height - 340
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
            X: =40
            Y: =Parent.Height - 84
            Width: =300
            OnSelect: |
              Set(varEditIndex; Blank());;
              Set(varValidarTablero; false);;
              Set(varPasoMaximoTablero; 1);;
              Reset(ddTipoEntrega);; Reset(ddInstalacionNuevaReemplazo);; Reset(txtNombreTablero);;
              Reset(numCantidad);; Reset(ddTipoTablero);; Reset(txtOtroTipoTablero);;
              Reset(txtFuncionTablero);; Reset(txtCargasAAlimentar);; Reset(numNumeroCircuitos);;
              Reset(ddUbicacion);; Reset(cmbAmbienteEspecial);; Reset(txtOtroAmbiente);;
              Reset(ddGradoIP);; Reset(ddGradoIK);; Reset(ddTipoMontaje);; Reset(ddRestricciones);;
              Reset(numAltoMax);; Reset(numAnchoMax);; Reset(numFondoMax);; Reset(txtCondicionesInstalacion);;
              Reset(ddTension);; Reset(numOtraTension);; Reset(ddSistema);; Reset(txtOtroSistema);;
              Reset(numPotencia);; Reset(ddUnidadPotencia);; Reset(ddFrecuencia);; Reset(numOtraFrecuencia);;
              Reset(cmbProteccionesRequeridas);; Reset(cmbMarcasPreferidas);; Reset(ddMaterialGabinete);;
              Reset(ddColorGabinete);; Reset(ddTipoVentilacion);; Reset(ddExpansionFutura);;
              Reset(txtObservacionesTablero);;
              Navigate(scrTableroForm; ScreenTransition.Cover)
      - btnSiguienteTableros:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Siguiente"
            X: =Parent.Width - 240
            Y: =Parent.Height - 84
            Width: =200
            OnSelect: |
              If(
                CountRows(colTableros) = 0;
                Notify("Agrega al menos un tablero antes de continuar."; NotificationType.Error);
                Set(varPasoMaximo; Max(varPasoMaximo; 3));;
                Navigate(scrDocumentacion; ScreenTransition.Cover)
              )
      - grpConfirmarEliminar:
          Control: GroupContainer
          Variant: "ManualLayout"
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
                    RemoveIf(colTableros; TableroId = locItemAEliminar);;
                    If(
                      Not(IsBlank(varEditIndex)) And locItemAEliminar = varEditIndex.TableroId;
                      Set(varEditIndex; Blank());;
                      Reset(ddTipoEntrega);; Reset(ddInstalacionNuevaReemplazo);; Reset(txtNombreTablero);;
                      Reset(numCantidad);; Reset(ddTipoTablero);; Reset(txtOtroTipoTablero);;
                      Reset(txtFuncionTablero);; Reset(txtCargasAAlimentar);; Reset(numNumeroCircuitos);;
                      Reset(ddUbicacion);; Reset(cmbAmbienteEspecial);; Reset(txtOtroAmbiente);;
                      Reset(ddGradoIP);; Reset(ddGradoIK);; Reset(ddTipoMontaje);; Reset(ddRestricciones);;
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

  scrTableroForm:
    Children:
      - navPestanas:
          Control: ModernTabList@1.0.0
          Properties:
            Items: =varNavPestanas
            Default: ="Tableros"
            Appearance: =TabListAppearance.Underline
            DisplayMode: =DisplayMode.View
            X: =0
            Y: =0
            Width: =Parent.Width
      - navSubPasos:
          Control: ModernTabList@1.0.0
          Properties:
            Items: =["1. Identificación"; "2. Ubicación y montaje"; "3. Eléctrico"; "4. Constructivo"]
            Default: ="1. Identificación"
            Appearance: =TabListAppearance.Underline
            X: =0
            Y: =40
            Width: =Parent.Width
            OnChange: |
              Switch(
                Self.Selected.Value;
                "1. Identificación";
                  If(
                    varPasoMaximoTablero >= 1;
                    Navigate(scrTableroForm; ScreenTransition.None);
                    Notify("Completa el paso anterior antes de continuar."; NotificationType.Warning);;
                    Reset(navSubPasos)
                  );
                "2. Ubicación y montaje";
                  If(
                    varPasoMaximoTablero >= 2;
                    Navigate(scrTableroForm2a; ScreenTransition.None);
                    Notify("Completa el paso 1 antes de continuar."; NotificationType.Warning);;
                    Reset(navSubPasos)
                  );
                "3. Eléctrico";
                  If(
                    varPasoMaximoTablero >= 3;
                    Navigate(scrTableroForm2b; ScreenTransition.None);
                    Notify("Completa el paso 2 antes de continuar."; NotificationType.Warning);;
                    Reset(navSubPasos)
                  );
                If(
                  varPasoMaximoTablero >= 4;
                  Navigate(scrTableroForm3; ScreenTransition.None);
                  Notify("Completa el paso 3 antes de continuar."; NotificationType.Warning);;
                  Reset(navSubPasos)
                )
              )
      - lblProgreso:
          Control: ModernText@1.0.0
          Properties:
            Text: |
              (If(IsBlank(varEditIndex); "Nuevo tablero"; "Editando: " & varEditIndex.Nombre)) & " — Paso 1 de 4: Identificación"
            X: =40
            Y: =96
            Width: =Parent.Width - 80
            Size: =16
            FontWeight: =FontWeight.Semibold
      - ddTipoEntrega:
          Control: ModernDropdown@1.0.2
          Properties:
            Items: =colOpcTipoEntrega
            ItemDisplayText: =ThisItem.Label
            Default: |
              If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTipoEntrega; Value = varEditIndex.TipoEntrega))
            X: =40
            Y: =124
            Width: =400
      - ddInstalacionNuevaReemplazo:
          Control: ModernDropdown@1.0.2
          Properties:
            Items: =colOpcInstalacionNuevaReemplazo
            ItemDisplayText: =ThisItem.Label
            Default: |
              If(IsBlank(varEditIndex); Blank(); LookUp(colOpcInstalacionNuevaReemplazo; Value = varEditIndex.InstalacionNuevaReemplazo))
            X: =460
            Y: =124
            Width: =400
      - txtNombreTablero:
          Control: ModernTextInput@1.1.1
          Properties:
            Placeholder: ="Nombre del tablero * (Ej.: TG Principal)"
            Default: =If(IsBlank(varEditIndex); ""; varEditIndex.Nombre)
            X: =40
            Y: =180
            Width: =400
      - numCantidad:
          Control: ModernNumberInput@1.1.1
          Properties:
            HintText: ="Cantidad"
            Default: =If(IsBlank(varEditIndex); 1; varEditIndex.Cantidad)
            Min: =1
            Max: =999
            Precision: =DecimalPrecision.'0'
            X: =460
            Y: =180
            Width: =200
      - ddTipoTablero:
          Control: ModernDropdown@1.0.2
          Properties:
            Visible: =ddTipoEntrega.Selected.Value = "tablero"
            Items: =colOpcTipoTablero
            ItemDisplayText: =ThisItem.Label
            Default: |
              If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTipoTablero; Value = varEditIndex.TipoTablero))
            X: =40
            Y: =236
            Width: =400
      - txtOtroTipoTablero:
          Control: ModernTextInput@1.1.1
          Properties:
            Visible: =ddTipoTablero.Selected.Value = "otro"
            Placeholder: ="Especifica el tipo de tablero"
            Default: =If(IsBlank(varEditIndex); ""; varEditIndex.OtroTipoTablero)
            X: =460
            Y: =236
            Width: =400
      - txtFuncionTablero:
          Control: ModernTextInput@1.1.1
          Properties:
            Type: =TextInputType.Multiline
            Placeholder: ="Función del tablero"
            Default: =If(IsBlank(varEditIndex); ""; varEditIndex.FuncionTablero)
            X: =40
            Y: =292
            Width: =820
            Height: =72
      - txtCargasAAlimentar:
          Control: ModernTextInput@1.1.1
          Properties:
            Type: =TextInputType.Multiline
            Placeholder: ="Cargas a alimentar"
            Default: =If(IsBlank(varEditIndex); ""; varEditIndex.CargasAAlimentar)
            X: =40
            Y: =376
            Width: =820
            Height: =72
      - numNumeroCircuitos:
          Control: ModernNumberInput@1.1.1
          Properties:
            HintText: ="Número de circuitos"
            Default: =If(IsBlank(varEditIndex); 0; varEditIndex.NumeroCircuitos)
            Min: =0
            Precision: =DecimalPrecision.'0'
            X: =40
            Y: =460
            Width: =200
      - lblPendientes:
          Control: ModernText@1.0.0
          Properties:
            Text: |
              If(tableroCompleto; "✓ Tablero completo — puedes guardar en el paso 4"; "⚠ Aún faltan obligatorios en: " & Concat(Filter(Table({p:"1. Identif."; ok:pasoIdentificacionOK}; {p:"2. Ubicación"; ok:pasoUbicacionOK}; {p:"3. Eléctrico"; ok:pasoElectricoOK}; {p:"4. Constructivo"; ok:pasoConstructivoOK}); Not(ok)); p; ",  "))
            X: =40
            Y: =Parent.Height - 150
            Width: =Parent.Width - 80
            Size: =12
            FontWeight: =FontWeight.Semibold
            Color: =If(tableroCompleto; RGBA(22;163;74;1); RGBA(217;119;6;1))
      - lblErroresPaso:
          Control: ModernText@1.0.0
          Properties:
            Visible: =And(varValidarTablero; Not(pasoIdentificacionOK))
            Text: |
              "Corrige en este paso: " & Concat(Filter(Table({n:"Nombre del tablero"; ok:Not(IsBlank(txtNombreTablero.Text))}; {n:"Tipo de entrega"; ok:Not(IsBlank(ddTipoEntrega.Selected.Value))}; {n:"Instalación nueva/reemplazo"; ok:Not(IsBlank(ddInstalacionNuevaReemplazo.Selected.Value))}; {n:"Tipo de tablero"; ok:Or(ddTipoEntrega.Selected.Value<>"tablero"; Not(IsBlank(ddTipoTablero.Selected.Value)))}); Not(ok)); n; ", ")
            X: =40
            Y: =Parent.Height - 124
            Width: =Parent.Width - 80
            Size: =11
            Color: =RGBA(239; 68; 68; 1)
      - grpConfirmarCancelar:
          Control: GroupContainer
          Variant: "ManualLayout"
          Properties:
            Visible: =mostrarConfirmarCancelar
            X: =Parent.Width * 0.5 - 180
            Y: =Parent.Height * 0.5 - 70
            Width: =360
            Height: =140
            Fill: =RGBA(255;255;255;1)
            BorderColor: =RGBA(0;0;0;0.2)
          Children:
            - lblConfirmarCancelar:
                Control: ModernText@1.0.0
                Properties:
                  Text: ="¿Descartar este tablero? Se perderán los datos no guardados."
                  X: =16
                  Y: =16
                  Width: =328
            - btnCancelarSi:
                Control: ModernButton@1.0.0
                Properties:
                  Text: ="Sí, descartar"
                  Appearance: =ButtonAppearance.Primary
                  X: =16
                  Y: =84
                  Width: =160
                  OnSelect: |
                    UpdateContext({mostrarConfirmarCancelar: false});;
                    Set(varEditIndex; Blank());;
                    Set(varValidarTablero; false);;
                    Navigate(scrTableros; ScreenTransition.UnCover)
            - btnCancelarNo:
                Control: ModernButton@1.0.0
                Properties:
                  Text: ="No, seguir"
                  Appearance: =ButtonAppearance.Outline
                  X: =184
                  Y: =84
                  Width: =160
                  OnSelect: =UpdateContext({mostrarConfirmarCancelar: false})
      - btnAtras1:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Cancelar"
            Appearance: =ButtonAppearance.Subtle
            X: =40
            Y: =Parent.Height - 76
            Width: =160
            # Salir del sub-flujo = perder el tablero a medio llenar: confirma antes.
            OnSelect: =UpdateContext({mostrarConfirmarCancelar: true})
      - btnSiguiente1:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Siguiente"
            X: =Parent.Width - 240
            Y: =Parent.Height - 76
            Width: =200
            OnSelect: |
              If(
                Not(pasoIdentificacionOK);
                Set(varValidarTablero; true);; Notify("Completa los obligatorios marcados en rojo."; NotificationType.Error);
                Set(varPasoMaximoTablero; Max(varPasoMaximoTablero; 2));;
                Navigate(scrTableroForm2a; ScreenTransition.Cover)
              )

  scrTableroForm2a:
    Children:
      - navPestanas:
          Control: ModernTabList@1.0.0
          Properties:
            Items: =varNavPestanas
            Default: ="Tableros"
            Appearance: =TabListAppearance.Underline
            DisplayMode: =DisplayMode.View
            X: =0
            Y: =0
            Width: =Parent.Width
      - navSubPasos:
          Control: ModernTabList@1.0.0
          Properties:
            Items: =["1. Identificación"; "2. Ubicación y montaje"; "3. Eléctrico"; "4. Constructivo"]
            Default: ="2. Ubicación y montaje"
            Appearance: =TabListAppearance.Underline
            X: =0
            Y: =40
            Width: =Parent.Width
            OnChange: |
              Switch(
                Self.Selected.Value;
                "1. Identificación";
                  If(
                    varPasoMaximoTablero >= 1;
                    Navigate(scrTableroForm; ScreenTransition.None);
                    Notify("Completa el paso anterior antes de continuar."; NotificationType.Warning);;
                    Reset(navSubPasos)
                  );
                "2. Ubicación y montaje";
                  If(
                    varPasoMaximoTablero >= 2;
                    Navigate(scrTableroForm2a; ScreenTransition.None);
                    Notify("Completa el paso 1 antes de continuar."; NotificationType.Warning);;
                    Reset(navSubPasos)
                  );
                "3. Eléctrico";
                  If(
                    varPasoMaximoTablero >= 3;
                    Navigate(scrTableroForm2b; ScreenTransition.None);
                    Notify("Completa el paso 2 antes de continuar."; NotificationType.Warning);;
                    Reset(navSubPasos)
                  );
                If(
                  varPasoMaximoTablero >= 4;
                  Navigate(scrTableroForm3; ScreenTransition.None);
                  Notify("Completa el paso 3 antes de continuar."; NotificationType.Warning);;
                  Reset(navSubPasos)
                )
              )
      - lblProgreso:
          Control: ModernText@1.0.0
          Properties:
            Text: |
              (If(IsBlank(varEditIndex); "Nuevo tablero"; "Editando: " & varEditIndex.Nombre)) & " — Paso 2 de 4: Ubicación, ambiente y montaje"
            X: =40
            Y: =96
            Width: =Parent.Width - 80
            Size: =16
            FontWeight: =FontWeight.Semibold
      - ddUbicacion:
          Control: ModernDropdown@1.0.2
          Properties:
            Items: =colOpcUbicacion
            ItemDisplayText: =ThisItem.Label
            Default: |
              If(IsBlank(varEditIndex); Blank(); LookUp(colOpcUbicacion; Value = varEditIndex.Ubicacion))
            X: =40
            Y: =124
            Width: =400
      - cmbAmbienteEspecial:
          Control: ModernCombobox@1.1.1
          Properties:
            Items: =colOpcAmbienteEspecial
            ItemDisplayText: =ThisItem.Label
            SelectMultiple: =true
            DefaultSelectedItems: =If(IsBlank(varEditIndex); Table(); varEditIndex.AmbienteEspecial)
            X: =460
            Y: =124
            Width: =400
      - txtOtroAmbiente:
          Control: ModernTextInput@1.1.1
          Properties:
            Visible: ="otro" in cmbAmbienteEspecial.SelectedItems.Value
            Placeholder: ="Especifica el ambiente especial"
            Default: =If(IsBlank(varEditIndex); ""; varEditIndex.OtroAmbienteEspecial)
            X: =40
            Y: =180
            Width: =400
      - ddGradoIP:
          Control: ModernDropdown@1.0.2
          Properties:
            Items: |
              Switch(ddUbicacion.Selected.Value;
                "interior"; ["IP20";"IP31";"IP43";"IP54";"IP55";"IP65"];
                "exterior"; ["IP54";"IP55";"IP65";"IP66";"IP67";"IP68"];
                ["IP20";"IP31";"IP43";"IP54";"IP55";"IP65";"IP66";"IP67";"IP68"]
              )
            Default: |
              If(IsBlank(varEditIndex); Blank(); {Value: varEditIndex.GradoIP})
            X: =460
            Y: =180
            Width: =200
      - ddGradoIK:
          Control: ModernDropdown@1.0.2
          Properties:
            # IK06=1J, IK07=2J, IK08=5J, IK09=10J, IK10=20J (IEC 62262)
            Items: |
              ["IK06";"IK07";"IK08";"IK09";"IK10"]
            Default: |
              If(IsBlank(varEditIndex); Blank(); {Value: varEditIndex.GradoIK})
            X: =676
            Y: =180
            Width: =184
      - ddTipoMontaje:
          Control: ModernDropdown@1.0.2
          Properties:
            Items: =colOpcTipoMontaje
            ItemDisplayText: =ThisItem.Label
            Default: |
              If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTipoMontaje; Value = varEditIndex.TipoMontaje))
            X: =40
            Y: =236
            Width: =400
      - ddRestricciones:
          Control: ModernDropdown@1.0.2
          Properties:
            Items: =colOpcSiNo
            ItemDisplayText: =ThisItem.Label
            Default: |
              If(IsBlank(varEditIndex); LookUp(colOpcSiNo; Value = false); LookUp(colOpcSiNo; Value = varEditIndex.RestriccionesDimension))
            X: =460
            Y: =236
            Width: =300
      - numAltoMax:
          Control: ModernNumberInput@1.1.1
          Properties:
            Visible: =ddRestricciones.Selected.Value
            HintText: ="Alto máximo (mm)"
            Default: =If(IsBlank(varEditIndex); 0; varEditIndex.AltoMaxMm)
            Min: =0
            Precision: =DecimalPrecision.'0'
            X: =40
            Y: =292
            Width: =200
      - numAnchoMax:
          Control: ModernNumberInput@1.1.1
          Properties:
            Visible: =ddRestricciones.Selected.Value
            HintText: ="Ancho máximo (mm)"
            Default: =If(IsBlank(varEditIndex); 0; varEditIndex.AnchoMaxMm)
            Min: =0
            Precision: =DecimalPrecision.'0'
            X: =256
            Y: =292
            Width: =200
      - numFondoMax:
          Control: ModernNumberInput@1.1.1
          Properties:
            Visible: =ddRestricciones.Selected.Value
            HintText: ="Fondo máximo (mm)"
            Default: =If(IsBlank(varEditIndex); 0; varEditIndex.FondoMaxMm)
            Min: =0
            Precision: =DecimalPrecision.'0'
            X: =472
            Y: =292
            Width: =200
      - txtCondicionesInstalacion:
          Control: ModernTextInput@1.1.1
          Properties:
            Type: =TextInputType.Multiline
            Placeholder: ="Condiciones adicionales de instalación"
            Default: =If(IsBlank(varEditIndex); ""; varEditIndex.CondicionesInstalacion)
            X: =40
            Y: =348
            Width: =820
            Height: =72
      - lblPendientes:
          Control: ModernText@1.0.0
          Properties:
            Text: |
              If(tableroCompleto; "✓ Tablero completo — puedes guardar en el paso 4"; "⚠ Aún faltan obligatorios en: " & Concat(Filter(Table({p:"1. Identif."; ok:pasoIdentificacionOK}; {p:"2. Ubicación"; ok:pasoUbicacionOK}; {p:"3. Eléctrico"; ok:pasoElectricoOK}; {p:"4. Constructivo"; ok:pasoConstructivoOK}); Not(ok)); p; ",  "))
            X: =40
            Y: =Parent.Height - 150
            Width: =Parent.Width - 80
            Size: =12
            FontWeight: =FontWeight.Semibold
            Color: =If(tableroCompleto; RGBA(22;163;74;1); RGBA(217;119;6;1))
      - lblErroresPaso:
          Control: ModernText@1.0.0
          Properties:
            Visible: =And(varValidarTablero; Not(pasoUbicacionOK))
            Text: |
              "Corrige en este paso: " & Concat(Filter(Table({n:"Ubicación"; ok:Not(IsBlank(ddUbicacion.Selected.Value))}; {n:"Grado IP"; ok:Not(IsBlank(ddGradoIP.Selected.Value))}; {n:"Tipo de montaje"; ok:Not(IsBlank(ddTipoMontaje.Selected.Value))}); Not(ok)); n; ", ")
            X: =40
            Y: =Parent.Height - 124
            Width: =Parent.Width - 80
            Size: =11
            Color: =RGBA(239; 68; 68; 1)
      - btnCancelar:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Cancelar"
            Appearance: =ButtonAppearance.Subtle
            X: =220
            Y: =Parent.Height - 76
            Width: =160
            OnSelect: =UpdateContext({mostrarConfirmarCancelar: true})
      - grpConfirmarCancelar:
          Control: GroupContainer
          Variant: "ManualLayout"
          Properties:
            Visible: =mostrarConfirmarCancelar
            X: =Parent.Width * 0.5 - 180
            Y: =Parent.Height * 0.5 - 70
            Width: =360
            Height: =140
            Fill: =RGBA(255;255;255;1)
            BorderColor: =RGBA(0;0;0;0.2)
          Children:
            - lblConfirmarCancelar:
                Control: ModernText@1.0.0
                Properties:
                  Text: ="¿Descartar este tablero? Se perderán los datos no guardados."
                  X: =16
                  Y: =16
                  Width: =328
            - btnCancelarSi:
                Control: ModernButton@1.0.0
                Properties:
                  Text: ="Sí, descartar"
                  Appearance: =ButtonAppearance.Primary
                  X: =16
                  Y: =84
                  Width: =160
                  OnSelect: |
                    UpdateContext({mostrarConfirmarCancelar: false});;
                    Set(varEditIndex; Blank());;
                    Set(varValidarTablero; false);;
                    Navigate(scrTableros; ScreenTransition.UnCover)
            - btnCancelarNo:
                Control: ModernButton@1.0.0
                Properties:
                  Text: ="No, seguir"
                  Appearance: =ButtonAppearance.Outline
                  X: =184
                  Y: =84
                  Width: =160
                  OnSelect: =UpdateContext({mostrarConfirmarCancelar: false})
      - btnAtras2a:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Atrás"
            Appearance: =ButtonAppearance.Outline
            X: =40
            Y: =Parent.Height - 76
            Width: =160
            OnSelect: =Navigate(scrTableroForm; ScreenTransition.UnCover)
      - btnSiguiente2a:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Siguiente"
            X: =Parent.Width - 240
            Y: =Parent.Height - 76
            Width: =200
            OnSelect: |
              If(
                Not(pasoUbicacionOK);
                Set(varValidarTablero; true);; Notify("Completa los obligatorios marcados en rojo."; NotificationType.Error);
                Set(varPasoMaximoTablero; Max(varPasoMaximoTablero; 3));;
                Navigate(scrTableroForm2b; ScreenTransition.Cover)
              )

  scrTableroForm2b:
    Children:
      - navPestanas:
          Control: ModernTabList@1.0.0
          Properties:
            Items: =varNavPestanas
            Default: ="Tableros"
            Appearance: =TabListAppearance.Underline
            DisplayMode: =DisplayMode.View
            X: =0
            Y: =0
            Width: =Parent.Width
      - navSubPasos:
          Control: ModernTabList@1.0.0
          Properties:
            Items: =["1. Identificación"; "2. Ubicación y montaje"; "3. Eléctrico"; "4. Constructivo"]
            Default: ="3. Eléctrico"
            Appearance: =TabListAppearance.Underline
            X: =0
            Y: =40
            Width: =Parent.Width
            OnChange: |
              Switch(
                Self.Selected.Value;
                "1. Identificación";
                  If(
                    varPasoMaximoTablero >= 1;
                    Navigate(scrTableroForm; ScreenTransition.None);
                    Notify("Completa el paso anterior antes de continuar."; NotificationType.Warning);;
                    Reset(navSubPasos)
                  );
                "2. Ubicación y montaje";
                  If(
                    varPasoMaximoTablero >= 2;
                    Navigate(scrTableroForm2a; ScreenTransition.None);
                    Notify("Completa el paso 1 antes de continuar."; NotificationType.Warning);;
                    Reset(navSubPasos)
                  );
                "3. Eléctrico";
                  If(
                    varPasoMaximoTablero >= 3;
                    Navigate(scrTableroForm2b; ScreenTransition.None);
                    Notify("Completa el paso 2 antes de continuar."; NotificationType.Warning);;
                    Reset(navSubPasos)
                  );
                If(
                  varPasoMaximoTablero >= 4;
                  Navigate(scrTableroForm3; ScreenTransition.None);
                  Notify("Completa el paso 3 antes de continuar."; NotificationType.Warning);;
                  Reset(navSubPasos)
                )
              )
      - lblProgreso:
          Control: ModernText@1.0.0
          Properties:
            Text: |
              (If(IsBlank(varEditIndex); "Nuevo tablero"; "Editando: " & varEditIndex.Nombre)) & " — Paso 3 de 4: Parámetros eléctricos"
            X: =40
            Y: =96
            Width: =Parent.Width - 80
            Size: =16
            FontWeight: =FontWeight.Semibold
      - ddTension:
          Control: ModernDropdown@1.0.2
          Properties:
            Items: =colOpcTension
            ItemDisplayText: =ThisItem.Label
            Default: |
              If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTension; Value = varEditIndex.TensionSuministro))
            X: =40
            Y: =124
            Width: =300
      - numOtraTension:
          Control: ModernNumberInput@1.1.1
          Properties:
            Visible: =ddTension.Selected.Value = "otro"
            HintText: ="Tensión (V)"
            Default: =If(IsBlank(varEditIndex); 0; varEditIndex.OtraTension)
            Min: =0
            X: =356
            Y: =124
            Width: =200
      - ddSistema:
          Control: ModernDropdown@1.0.2
          Properties:
            Items: =colOpcSistema
            ItemDisplayText: =ThisItem.Label
            Default: |
              If(IsBlank(varEditIndex); Blank(); LookUp(colOpcSistema; Value = varEditIndex.SistemaElectrico))
            X: =572
            Y: =124
            Width: =288
      - txtOtroSistema:
          Control: ModernTextInput@1.1.1
          Properties:
            Visible: =ddSistema.Selected.Value = "otro"
            Placeholder: ="Especifica el sistema eléctrico"
            Default: =If(IsBlank(varEditIndex); ""; varEditIndex.OtroSistemaElectrico)
            X: =40
            Y: =180
            Width: =400
      - numPotencia:
          Control: ModernNumberInput@1.1.1
          Properties:
            HintText: ="Potencia estimada"
            Default: =If(IsBlank(varEditIndex); Blank(); varEditIndex.PotenciaEstimada)
            Min: =0
            Precision: =DecimalPrecision.'2'
            X: =460
            Y: =180
            Width: =200
      - ddUnidadPotencia:
          Control: ModernDropdown@1.0.2
          Properties:
            Items: =colOpcUnidadPotencia
            ItemDisplayText: =ThisItem.Label
            Default: |
              If(IsBlank(varEditIndex); LookUp(colOpcUnidadPotencia; Value="kW"); LookUp(colOpcUnidadPotencia; Value = varEditIndex.UnidadPotencia))
            X: =676
            Y: =180
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
            X: =40
            Y: =236
            Width: =400
      - ddFrecuencia:
          Control: ModernDropdown@1.0.2
          Properties:
            Items: =colOpcFrecuencia
            ItemDisplayText: =ThisItem.Label
            Default: |
              If(IsBlank(varEditIndex); Blank(); LookUp(colOpcFrecuencia; Value = varEditIndex.Frecuencia))
            X: =460
            Y: =236
            Width: =200
      - numOtraFrecuencia:
          Control: ModernNumberInput@1.1.1
          Properties:
            Visible: =ddFrecuencia.Selected.Value = "otro"
            HintText: ="Frecuencia (Hz)"
            Default: =If(IsBlank(varEditIndex); 0; varEditIndex.OtraFrecuencia)
            Min: =0
            X: =676
            Y: =236
            Width: =184
      - cmbProteccionesRequeridas:
          Control: ModernCombobox@1.1.1
          Properties:
            Items: =colOpcProteccionesRequeridas
            ItemDisplayText: =ThisItem.Label
            SelectMultiple: =true
            DefaultSelectedItems: =If(IsBlank(varEditIndex); Table(); varEditIndex.ProteccionesRequeridas)
            X: =40
            Y: =292
            Width: =400
      - cmbMarcasPreferidas:
          Control: ModernCombobox@1.1.1
          Properties:
            Items: =colOpcMarcasPreferidas
            ItemDisplayText: =ThisItem.Label
            SelectMultiple: =true
            DefaultSelectedItems: =If(IsBlank(varEditIndex); Table(); varEditIndex.MarcasPreferidas)
            X: =460
            Y: =292
            Width: =400
      - lblPendientes:
          Control: ModernText@1.0.0
          Properties:
            Text: |
              If(tableroCompleto; "✓ Tablero completo — puedes guardar en el paso 4"; "⚠ Aún faltan obligatorios en: " & Concat(Filter(Table({p:"1. Identif."; ok:pasoIdentificacionOK}; {p:"2. Ubicación"; ok:pasoUbicacionOK}; {p:"3. Eléctrico"; ok:pasoElectricoOK}; {p:"4. Constructivo"; ok:pasoConstructivoOK}); Not(ok)); p; ",  "))
            X: =40
            Y: =Parent.Height - 150
            Width: =Parent.Width - 80
            Size: =12
            FontWeight: =FontWeight.Semibold
            Color: =If(tableroCompleto; RGBA(22;163;74;1); RGBA(217;119;6;1))
      - lblErroresPaso:
          Control: ModernText@1.0.0
          Properties:
            Visible: =And(varValidarTablero; Not(pasoElectricoOK))
            Text: |
              "Corrige en este paso: " & Concat(Filter(Table({n:"Tensión"; ok:Not(IsBlank(ddTension.Selected.Value))}; {n:"Sistema"; ok:Not(IsBlank(ddSistema.Selected.Value))}; {n:"Potencia (>0)"; ok:numPotencia.Value>0}; {n:"Frecuencia"; ok:Not(IsBlank(ddFrecuencia.Selected.Value))}; {n:"Protecciones (≥1)"; ok:CountRows(cmbProteccionesRequeridas.SelectedItems)>0}); Not(ok)); n; ", ")
            X: =40
            Y: =Parent.Height - 124
            Width: =Parent.Width - 80
            Size: =11
            Color: =RGBA(239; 68; 68; 1)
      - btnCancelar:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Cancelar"
            Appearance: =ButtonAppearance.Subtle
            X: =220
            Y: =Parent.Height - 76
            Width: =160
            OnSelect: =UpdateContext({mostrarConfirmarCancelar: true})
      - grpConfirmarCancelar:
          Control: GroupContainer
          Variant: "ManualLayout"
          Properties:
            Visible: =mostrarConfirmarCancelar
            X: =Parent.Width * 0.5 - 180
            Y: =Parent.Height * 0.5 - 70
            Width: =360
            Height: =140
            Fill: =RGBA(255;255;255;1)
            BorderColor: =RGBA(0;0;0;0.2)
          Children:
            - lblConfirmarCancelar:
                Control: ModernText@1.0.0
                Properties:
                  Text: ="¿Descartar este tablero? Se perderán los datos no guardados."
                  X: =16
                  Y: =16
                  Width: =328
            - btnCancelarSi:
                Control: ModernButton@1.0.0
                Properties:
                  Text: ="Sí, descartar"
                  Appearance: =ButtonAppearance.Primary
                  X: =16
                  Y: =84
                  Width: =160
                  OnSelect: |
                    UpdateContext({mostrarConfirmarCancelar: false});;
                    Set(varEditIndex; Blank());;
                    Set(varValidarTablero; false);;
                    Navigate(scrTableros; ScreenTransition.UnCover)
            - btnCancelarNo:
                Control: ModernButton@1.0.0
                Properties:
                  Text: ="No, seguir"
                  Appearance: =ButtonAppearance.Outline
                  X: =184
                  Y: =84
                  Width: =160
                  OnSelect: =UpdateContext({mostrarConfirmarCancelar: false})
      - btnAtras2b:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Atrás"
            Appearance: =ButtonAppearance.Outline
            X: =40
            Y: =Parent.Height - 76
            Width: =160
            OnSelect: =Navigate(scrTableroForm2a; ScreenTransition.UnCover)
      - btnSiguiente2b:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Siguiente"
            X: =Parent.Width - 240
            Y: =Parent.Height - 76
            Width: =200
            OnSelect: |
              If(
                Not(pasoElectricoOK);
                Set(varValidarTablero; true);; Notify("Completa los obligatorios marcados en rojo."; NotificationType.Error);
                Set(varPasoMaximoTablero; Max(varPasoMaximoTablero; 4));;
                Navigate(scrTableroForm3; ScreenTransition.Cover)
              )

  scrTableroForm3:
    Children:
      - navPestanas:
          Control: ModernTabList@1.0.0
          Properties:
            Items: =varNavPestanas
            Default: ="Tableros"
            Appearance: =TabListAppearance.Underline
            DisplayMode: =DisplayMode.View
            X: =0
            Y: =0
            Width: =Parent.Width
      - navSubPasos:
          Control: ModernTabList@1.0.0
          Properties:
            Items: =["1. Identificación"; "2. Ubicación y montaje"; "3. Eléctrico"; "4. Constructivo"]
            Default: ="4. Constructivo"
            Appearance: =TabListAppearance.Underline
            X: =0
            Y: =40
            Width: =Parent.Width
            OnChange: |
              Switch(
                Self.Selected.Value;
                "1. Identificación";
                  If(
                    varPasoMaximoTablero >= 1;
                    Navigate(scrTableroForm; ScreenTransition.None);
                    Notify("Completa el paso anterior antes de continuar."; NotificationType.Warning);;
                    Reset(navSubPasos)
                  );
                "2. Ubicación y montaje";
                  If(
                    varPasoMaximoTablero >= 2;
                    Navigate(scrTableroForm2a; ScreenTransition.None);
                    Notify("Completa el paso 1 antes de continuar."; NotificationType.Warning);;
                    Reset(navSubPasos)
                  );
                "3. Eléctrico";
                  If(
                    varPasoMaximoTablero >= 3;
                    Navigate(scrTableroForm2b; ScreenTransition.None);
                    Notify("Completa el paso 2 antes de continuar."; NotificationType.Warning);;
                    Reset(navSubPasos)
                  );
                If(
                  varPasoMaximoTablero >= 4;
                  Navigate(scrTableroForm3; ScreenTransition.None);
                  Notify("Completa el paso 3 antes de continuar."; NotificationType.Warning);;
                  Reset(navSubPasos)
                )
              )
      - lblProgreso:
          Control: ModernText@1.0.0
          Properties:
            Text: |
              (If(IsBlank(varEditIndex); "Nuevo tablero"; "Editando: " & varEditIndex.Nombre)) & " — Paso 4 de 4: Diseño constructivo"
            X: =40
            Y: =96
            Width: =Parent.Width - 80
            Size: =16
            FontWeight: =FontWeight.Semibold
      - ddMaterialGabinete:
          Control: ModernDropdown@1.0.2
          Properties:
            Items: =colOpcMaterialGabinete
            ItemDisplayText: =ThisItem.Label
            Default: |
              If(IsBlank(varEditIndex); Blank(); LookUp(colOpcMaterialGabinete; Value = varEditIndex.MaterialGabinete))
            X: =40
            Y: =124
            Width: =400
      - ddColorGabinete:
          Control: ModernDropdown@1.0.2
          Properties:
            Items: =colOpcColorGabinete
            ItemDisplayText: =ThisItem.Label
            Default: |
              If(IsBlank(varEditIndex); LookUp(colOpcColorGabinete; Value="7035"); LookUp(colOpcColorGabinete; Value = varEditIndex.ColorGabinete))
            X: =460
            Y: =124
            Width: =400
      - ddTipoVentilacion:
          Control: ModernDropdown@1.0.2
          Properties:
            Items: =colOpcTipoVentilacion
            ItemDisplayText: =ThisItem.Label
            Default: |
              If(IsBlank(varEditIndex); Blank(); LookUp(colOpcTipoVentilacion; Value = varEditIndex.TipoVentilacion))
            X: =40
            Y: =180
            Width: =400
      - ddExpansionFutura:
          Control: ModernDropdown@1.0.2
          Properties:
            Items: =colOpcExpansionFutura
            ItemDisplayText: =ThisItem.Label
            Default: |
              If(IsBlank(varEditIndex); Blank(); LookUp(colOpcExpansionFutura; Value = varEditIndex.ExpansionFutura))
            X: =460
            Y: =180
            Width: =400
      - txtObservacionesTablero:
          Control: ModernTextInput@1.1.1
          Properties:
            Type: =TextInputType.Multiline
            Placeholder: ="Observaciones del tablero"
            Default: =If(IsBlank(varEditIndex); ""; varEditIndex.ObservacionesTablero)
            X: =40
            Y: =236
            Width: =820
            Height: =72
      - lblPendientes:
          Control: ModernText@1.0.0
          Properties:
            Text: |
              If(tableroCompleto; "✓ Tablero completo — puedes guardar en el paso 4"; "⚠ Aún faltan obligatorios en: " & Concat(Filter(Table({p:"1. Identif."; ok:pasoIdentificacionOK}; {p:"2. Ubicación"; ok:pasoUbicacionOK}; {p:"3. Eléctrico"; ok:pasoElectricoOK}; {p:"4. Constructivo"; ok:pasoConstructivoOK}); Not(ok)); p; ",  "))
            X: =40
            Y: =Parent.Height - 150
            Width: =Parent.Width - 80
            Size: =12
            FontWeight: =FontWeight.Semibold
            Color: =If(tableroCompleto; RGBA(22;163;74;1); RGBA(217;119;6;1))
      - lblErroresPaso:
          Control: ModernText@1.0.0
          Properties:
            Visible: =And(varValidarTablero; Not(pasoConstructivoOK))
            Text: |
              "Corrige en este paso: " & Concat(Filter(Table({n:"Material del gabinete"; ok:Not(IsBlank(ddMaterialGabinete.Selected.Value))}; {n:"Ventilación"; ok:Not(IsBlank(ddTipoVentilacion.Selected.Value))}; {n:"Expansión futura"; ok:Not(IsBlank(ddExpansionFutura.Selected.Value))}); Not(ok)); n; ", ")
            X: =40
            Y: =Parent.Height - 124
            Width: =Parent.Width - 80
            Size: =11
            Color: =RGBA(239; 68; 68; 1)
      - btnCancelar:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Cancelar"
            Appearance: =ButtonAppearance.Subtle
            X: =220
            Y: =Parent.Height - 76
            Width: =160
            OnSelect: =UpdateContext({mostrarConfirmarCancelar: true})
      - grpConfirmarCancelar:
          Control: GroupContainer
          Variant: "ManualLayout"
          Properties:
            Visible: =mostrarConfirmarCancelar
            X: =Parent.Width * 0.5 - 180
            Y: =Parent.Height * 0.5 - 70
            Width: =360
            Height: =140
            Fill: =RGBA(255;255;255;1)
            BorderColor: =RGBA(0;0;0;0.2)
          Children:
            - lblConfirmarCancelar:
                Control: ModernText@1.0.0
                Properties:
                  Text: ="¿Descartar este tablero? Se perderán los datos no guardados."
                  X: =16
                  Y: =16
                  Width: =328
            - btnCancelarSi:
                Control: ModernButton@1.0.0
                Properties:
                  Text: ="Sí, descartar"
                  Appearance: =ButtonAppearance.Primary
                  X: =16
                  Y: =84
                  Width: =160
                  OnSelect: |
                    UpdateContext({mostrarConfirmarCancelar: false});;
                    Set(varEditIndex; Blank());;
                    Set(varValidarTablero; false);;
                    Navigate(scrTableros; ScreenTransition.UnCover)
            - btnCancelarNo:
                Control: ModernButton@1.0.0
                Properties:
                  Text: ="No, seguir"
                  Appearance: =ButtonAppearance.Outline
                  X: =184
                  Y: =84
                  Width: =160
                  OnSelect: =UpdateContext({mostrarConfirmarCancelar: false})
      - btnAtras3:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Atrás"
            Appearance: =ButtonAppearance.Outline
            X: =40
            Y: =Parent.Height - 76
            Width: =160
            OnSelect: =Navigate(scrTableroForm2b; ScreenTransition.UnCover)
      - btnGuardarTablero:
          Control: ModernButton@1.0.0
          Properties:
            Text: ="Guardar tablero"
            X: =Parent.Width - 240
            Y: =Parent.Height - 76
            Width: =200
            OnSelect: |
              If(
                Not(tableroCompleto);
                Set(varValidarTablero; true);;
                Notify("Faltan obligatorios. Te llevo al primer paso incompleto (en rojo)."; NotificationType.Error);;
                If(
                  Not(pasoIdentificacionOK); Navigate(scrTableroForm; ScreenTransition.None);
                  Not(pasoUbicacionOK);      Navigate(scrTableroForm2a; ScreenTransition.None);
                  Not(pasoElectricoOK);      Navigate(scrTableroForm2b; ScreenTransition.None);
                  Navigate(scrTableroForm3; ScreenTransition.None)
                );
                With(
                  {
                    registro: {
                      Nombre: txtNombreTablero.Text;
                      Cantidad: numCantidad.Value;
                      TableroId: If(IsBlank(varEditIndex); varNextTableroId; varEditIndex.TableroId);
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
                      RestriccionesDimension: ddRestricciones.Selected.Value;
                      AltoMaxMm: If(Not(ddRestricciones.Selected.Value); Blank(); numAltoMax.Value);
                      AnchoMaxMm: If(Not(ddRestricciones.Selected.Value); Blank(); numAnchoMax.Value);
                      FondoMaxMm: If(Not(ddRestricciones.Selected.Value); Blank(); numFondoMax.Value);
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
                    Collect(colTableros; registro);; Set(varNextTableroId; varNextTableroId + 1);
                    UpdateIf(colTableros; TableroId = varEditIndex.TableroId; registro)
                  )
                );;
                Set(varEditIndex; Blank());;
                Set(varValidarTablero; false);;
                Reset(ddTipoEntrega);; Reset(ddInstalacionNuevaReemplazo);; Reset(txtNombreTablero);;
                Reset(numCantidad);; Reset(ddTipoTablero);; Reset(txtOtroTipoTablero);;
                Reset(txtFuncionTablero);; Reset(txtCargasAAlimentar);; Reset(numNumeroCircuitos);;
                Reset(ddUbicacion);; Reset(cmbAmbienteEspecial);; Reset(txtOtroAmbiente);;
                Reset(ddGradoIP);; Reset(ddGradoIK);; Reset(ddTipoMontaje);; Reset(ddRestricciones);;
                Reset(numAltoMax);; Reset(numAnchoMax);; Reset(numFondoMax);; Reset(txtCondicionesInstalacion);;
                Reset(ddTension);; Reset(numOtraTension);; Reset(ddSistema);; Reset(txtOtroSistema);;
                Reset(numPotencia);; Reset(ddUnidadPotencia);; Reset(ddFrecuencia);; Reset(numOtraFrecuencia);;
                Reset(cmbProteccionesRequeridas);; Reset(cmbMarcasPreferidas);; Reset(ddMaterialGabinete);;
                Reset(ddColorGabinete);; Reset(ddTipoVentilacion);; Reset(ddExpansionFutura);;
                Reset(txtObservacionesTablero);;
                Navigate(scrTableros; ScreenTransition.Cover)
              )

  scrDocumentacion:
    Children:
      - navPestanas:
          Control: ModernTabList@1.0.0
          Properties:
            Items: =varNavPestanas
            Default: ="Documentación"
            Appearance: =TabListAppearance.Underline
            X: =0
            Y: =0
            Width: =Parent.Width
            OnChange: |
              Switch(
                Self.Selected.Value;
                "Contacto y Proyecto"; Navigate(scrContactoProyecto; ScreenTransition.None);
                "Tableros";
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
      - txtObservacionesProyecto:
          Control: ModernTextInput@1.1.1
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
                      "csenergy"; 'IngenieriaPor (Solicitudes)'.CSEnergy;
                      "cliente";  'IngenieriaPor (Solicitudes)'.Cliente;
                      'IngenieriaPor (Solicitudes)'.Conjunta
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
                      "tablero";  'TipoEntrega (SolicitudTableros)'.'Tablero Eléctrico';
                      "sala";     'TipoEntrega (SolicitudTableros)'.'Sala Eléctrica';
                      'TipoEntrega (SolicitudTableros)'.'Producto Eléctrico'
                    );
                    InstalacionNuevaReemplazo: Switch(t.InstalacionNuevaReemplazo;
                      "nueva";     'InstalacionNuevaReemplazo (SolicitudTableros)'.'Instalación nueva';
                      'InstalacionNuevaReemplazo (SolicitudTableros)'.'Reemplazo de existente'
                    );
                    TipoTablero: Switch(t.TipoTablero;
                      "fuerza";         'TipoTablero (SolicitudTableros)'.'Fuerza/Potencia';
                      "alumbrado";      'TipoTablero (SolicitudTableros)'.'Alumbrado/Distribución BT';
                      "control";        'TipoTablero (SolicitudTableros)'.'Control/Automatización';
                      "transfer";       'TipoTablero (SolicitudTableros)'.'Transferencia (ATS/MTS)';
                      "sincronizacion"; 'TipoTablero (SolicitudTableros)'.'Sincronización de Generadores';
                      "remoto";         'TipoTablero (SolicitudTableros)'.'Distribución Remoto';
                      "pfcs";           'TipoTablero (SolicitudTableros)'.'Factor de Potencia';
                      "medicion";       'TipoTablero (SolicitudTableros)'.'Medición/Centro de Carga';
                      "variadores";     'TipoTablero (SolicitudTableros)'.'Variadores de Frecuencia';
                      "arrancadores";   'TipoTablero (SolicitudTableros)'.'Arrancadores Suaves';
                      "ups";            'TipoTablero (SolicitudTableros)'.'UPS/Respaldo';
                      'TipoTablero (SolicitudTableros)'.Otro
                    );
                    OtroTipoTablero: t.OtroTipoTablero;
                    FuncionTablero: t.FuncionTablero;
                    CargasAAlimentar: t.CargasAAlimentar;
                    NumeroCircuitos: t.NumeroCircuitos;
                    Ubicacion: Switch(t.Ubicacion;
                      "interior"; 'Ubicacion (SolicitudTableros)'.Interior;
                      'Ubicacion (SolicitudTableros)'.Exterior
                    );
                    AmbienteEspecial: ForAll(t.AmbienteEspecial As sel;
                      Switch(sel.Value;
                        "marino";      'AmbienteEspecial (SolicitudTableros)'.'Ambiente marino';
                        "minero";      'AmbienteEspecial (SolicitudTableros)'.'Ambiente minero';
                        "humedo";      'AmbienteEspecial (SolicitudTableros)'.'Húmedo';
                        "corrosivo";   'AmbienteEspecial (SolicitudTableros)'.Corrosivo;
                        "polvoriento"; 'AmbienteEspecial (SolicitudTableros)'.Polvoriento;
                        "explosivo";   'AmbienteEspecial (SolicitudTableros)'.'Atmósfera explosiva';
                        'AmbienteEspecial (SolicitudTableros)'.Otro
                      )
                    );
                    OtroAmbienteEspecial: t.OtroAmbienteEspecial;
                    GradoIP: Switch(t.GradoIP;
                      "IP20"; 'GradoIP (SolicitudTableros)'.IP20;
                      "IP31"; 'GradoIP (SolicitudTableros)'.IP31;
                      "IP43"; 'GradoIP (SolicitudTableros)'.IP43;
                      "IP54"; 'GradoIP (SolicitudTableros)'.IP54;
                      "IP55"; 'GradoIP (SolicitudTableros)'.IP55;
                      "IP65"; 'GradoIP (SolicitudTableros)'.IP65;
                      "IP66"; 'GradoIP (SolicitudTableros)'.IP66;
                      "IP67"; 'GradoIP (SolicitudTableros)'.IP67;
                      'GradoIP (SolicitudTableros)'.IP68
                    );
                    GradoIK: Switch(t.GradoIK;
                      "IK06"; 'GradoIK (SolicitudTableros)'.IK06;
                      "IK07"; 'GradoIK (SolicitudTableros)'.IK07;
                      "IK08"; 'GradoIK (SolicitudTableros)'.IK08;
                      "IK09"; 'GradoIK (SolicitudTableros)'.IK09;
                      'GradoIK (SolicitudTableros)'.IK10
                    );
                    TipoMontaje: Switch(t.TipoMontaje;
                      "autosoportado"; 'TipoMontaje (SolicitudTableros)'.Autosoportado;
                      "mural";         'TipoMontaje (SolicitudTableros)'.Mural;
                      "rack_19";       'TipoMontaje (SolicitudTableros)'.'Rack 19';
                      "pedestal";      'TipoMontaje (SolicitudTableros)'.Pedestal;
                      'TipoMontaje (SolicitudTableros)'.Otro
                    );
                    RestriccionesDimension: t.RestriccionesDimension;
                    AltoMaxMm: t.AltoMaxMm;
                    AnchoMaxMm: t.AnchoMaxMm;
                    FondoMaxMm: t.FondoMaxMm;
                    CondicionesInstalacion: t.CondicionesInstalacion;
                    TensionSuministro: Switch(t.TensionSuministro;
                      "220";  'TensionSuministro (SolicitudTableros)'.'220 V';
                      "380";  'TensionSuministro (SolicitudTableros)'.'380 V';
                      "400";  'TensionSuministro (SolicitudTableros)'.'400 V';
                      "440";  'TensionSuministro (SolicitudTableros)'.'440 V';
                      "480";  'TensionSuministro (SolicitudTableros)'.'480 V';
                      "690";  'TensionSuministro (SolicitudTableros)'.'690 V';
                      "1000"; 'TensionSuministro (SolicitudTableros)'.'1000 V';
                      'TensionSuministro (SolicitudTableros)'.Otro
                    );
                    OtraTension: t.OtraTension;
                    SistemaElectrico: Switch(t.SistemaElectrico;
                      "trifasico";  'SistemaElectrico (SolicitudTableros)'.'Trifásico';
                      "monofasico"; 'SistemaElectrico (SolicitudTableros)'.'Monofásico';
                      "dc";         'SistemaElectrico (SolicitudTableros)'.'Corriente continua (DC)';
                      'SistemaElectrico (SolicitudTableros)'.Otro
                    );
                    OtroSistemaElectrico: t.OtroSistemaElectrico;
                    PotenciaEstimada: t.PotenciaEstimada;
                    UnidadPotencia: Switch(t.UnidadPotencia;
                      "kW";  'UnidadPotencia (SolicitudTableros)'.kW;
                      'UnidadPotencia (SolicitudTableros)'.kVA
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
                        "interruptor_automatico"; 'ProteccionesRequeridas (SolicitudTableros)'.'Interruptor automático';
                        "diferencial";            'ProteccionesRequeridas (SolicitudTableros)'.Diferencial;
                        "fusible";                'ProteccionesRequeridas (SolicitudTableros)'.Fusible;
                        "relevo_sobrecarga";       'ProteccionesRequeridas (SolicitudTableros)'.'Relevo de sobrecarga';
                        "relevo_falla_tierra";     'ProteccionesRequeridas (SolicitudTableros)'.'Relevo de falla a tierra';
                        "proteccion_tension";      'ProteccionesRequeridas (SolicitudTableros)'.'Protección de tensión';
                        "proteccion_corriente";    'ProteccionesRequeridas (SolicitudTableros)'.'Protección de corriente';
                        "descargador_tension";     'ProteccionesRequeridas (SolicitudTableros)'.'Descargador de tensión';
                        'ProteccionesRequeridas (SolicitudTableros)'.Otro
                      )
                    );
                    MarcasPreferidas: ForAll(t.MarcasPreferidas As sel;
                      Switch(sel.Value;
                        "schneider";   'MarcasPreferidas (SolicitudTableros)'.'Schneider Electric';
                        "siemens";     'MarcasPreferidas (SolicitudTableros)'.Siemens;
                        "abb";         'MarcasPreferidas (SolicitudTableros)'.ABB;
                        "legrand";     'MarcasPreferidas (SolicitudTableros)'.Legrand;
                        "eaton";       'MarcasPreferidas (SolicitudTableros)'.Eaton;
                        "chint";       'MarcasPreferidas (SolicitudTableros)'.Chint;
                        "hager";       'MarcasPreferidas (SolicitudTableros)'.Hager;
                        "weidmuller";  'MarcasPreferidas (SolicitudTableros)'.'Weidmüller';
                        "phoenix";     'MarcasPreferidas (SolicitudTableros)'.'Phoenix Contact';
                        'MarcasPreferidas (SolicitudTableros)'.Otro
                      )
                    );
                    MaterialGabinete: Switch(t.MaterialGabinete;
                      "acero_pintado";     'MaterialGabinete (SolicitudTableros)'.'Acero pintado';
                      "acero_galvanizado"; 'MaterialGabinete (SolicitudTableros)'.'Acero galvanizado';
                      "acero_inoxidable";  'MaterialGabinete (SolicitudTableros)'.'Acero inoxidable';
                      "acero_inox_316";    'MaterialGabinete (SolicitudTableros)'.'Acero inoxidable 316';
                      "fibra_vidrio";      'MaterialGabinete (SolicitudTableros)'.'Fibra de vidrio';
                      "poliester";         'MaterialGabinete (SolicitudTableros)'.'Poliéster';
                      'MaterialGabinete (SolicitudTableros)'.Aluminio
                    );
                    ColorGabinete: Switch(t.ColorGabinete;
                      "7035"; 'ColorGabinete (SolicitudTableros)'.'RAL 7035 (gris claro)';
                      "7016"; 'ColorGabinete (SolicitudTableros)'.'RAL 7016 (gris antracita)';
                      "9016"; 'ColorGabinete (SolicitudTableros)'.'RAL 9016 (blanco tráfico)';
                      "9005"; 'ColorGabinete (SolicitudTableros)'.'RAL 9005 (negro)';
                      "5010"; 'ColorGabinete (SolicitudTableros)'.'RAL 5010 (azul)';
                      "6005"; 'ColorGabinete (SolicitudTableros)'.'RAL 6005 (verde)';
                      'ColorGabinete (SolicitudTableros)'.Otro
                    );
                    TipoVentilacion: Switch(t.TipoVentilacion;
                      "natural";     'TipoVentilacion (SolicitudTableros)'.Natural;
                      "forzada";     'TipoVentilacion (SolicitudTableros)'.'Forzada (ventilador)';
                      "sellado";     'TipoVentilacion (SolicitudTableros)'.'Sellado (IP alto)';
                      'TipoVentilacion (SolicitudTableros)'.'Climatizado (aire acondicionado)'
                    );
                    ExpansionFutura: Switch(t.ExpansionFutura;
                      "no";  'ExpansionFutura (SolicitudTableros)'.'Sin expansión';
                      "10";  'ExpansionFutura (SolicitudTableros)'.'10%';
                      "20";  'ExpansionFutura (SolicitudTableros)'.'20%';
                      "30";  'ExpansionFutura (SolicitudTableros)'.'30%';
                      'ExpansionFutura (SolicitudTableros)'.Otro
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
  no un layout final — muévelas visualmente en Studio después de pegar.
- **8 pantallas en total** ahora: `scrContactoProyecto`, `scrTableros`,
  `scrTableroForm`/`2a`/`2b`/`3` (las 4 etapas del tablero), `scrDocumentacion`,
  `scrConfirmacion`. Créalas todas antes de empezar a pegar.
- La validación de "campos obligatorios" quedó repartida por pantalla (cada
  `Siguiente` valida solo los campos de su propio paso) en vez de todo junto
  al final, como estaba en la versión de un solo panel — es más fácil de
  ubicar el error para quien está llenando el formulario.
- `Solicitud`/`ContactoNombre`/etc. de `scrDocumentacion` referencian
  controles de `scrContactoProyecto` — Power Fx puede referenciar controles
  de **cualquier pantalla** dentro de la misma app, esto ya lo usaba el
  diseño original.
- Si el pegado masivo falla en algún punto puntual, no reinicies desde cero:
  identifica cuál control falló y créalo a mano — el resto del pegado ya
  construido queda intacto.
- **Por qué existe `TableroId`:** `AmbienteEspecial`, `ProteccionesRequeridas`
  y `MarcasPreferidas` son columnas de tipo **tabla anidada** dentro de cada
  fila de `colTableros` (vienen de un `ModernCombobox` multiselección). Power
  Fx **no puede comparar dos registros con `=`** ni hacer `Remove`/`Patch`
  **por registro completo** cuando el registro contiene una columna de tipo
  tabla — da error de tipo o "no se pueden comparar estos tipos: Record,
  Record". Por eso cada fila lleva un campo `TableroId` (Number, asignado con
  el contador `varNextTableroId` que nunca se reutiliza, ni siquiera tras un
  Eliminar) y todas las operaciones que necesitan identificar "esta fila
  puntual" (resaltado de selección, Eliminar, Guardar en modo edición) se
  hacen comparando `TableroId` (un Number, sí comparable) en vez del registro
  entero — con `RemoveIf`/`UpdateIf` en lugar de `Remove`/`Patch` por
  registro. Si en el futuro agregas otra columna de tabla anidada al tablero,
  recuerda este mismo patrón.
