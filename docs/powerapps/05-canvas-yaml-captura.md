# Canvas — código YAML de la pantalla de captura (Dataverse)

Código fuente `.pa.yaml` de la **Pantalla 1 — Contacto y Proyecto** y patrones de
la pantalla del tablero, enlazados a **Dataverse**.

## Cómo usarlo

No se pega directo en cualquier vista del estudio. Vías válidas:

1. **Git integration (recomendado con Dataverse):** conecta la solución que
   contiene la app a un repo Git; los archivos `Src/*.pa.yaml` son editables y se
   sincronizan al estudio.
2. **Power Platform CLI:**
   ```
   pac canvas pack --sources ./Src --msapp Captura.msapp
   ```
   y luego importas el `.msapp`.

## Requisitos previos

- En la app, **Agregar datos** → conector **Dataverse** → tablas `Solicitudes` y
  `SolicitudTableros`.
- Los nombres de columna de Choice de Dataverse se referencian como
  `'Columna (Tabla)'.Opcion` (sintaxis de option set), distinta de SharePoint.

---

## App — colección e inicialización

```yaml
App:
  Properties:
    OnStart: |
      =Set(varEditIndex, Blank());
      ClearCollect(colTableros, [])
    # Si prefieres, define colTableros como Named Formula en lugar de OnStart.
```

---

## Pantalla 1 — Contacto y Proyecto

```yaml
Screens:
  scrContacto:
    Properties:
      Fill: =RGBA(245, 246, 248, 1)
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
            Color: =RGBA(239, 68, 68, 1)
            Visible: =And(Len(txtContactoEmail.Text) > 0, Not(IsMatch(txtContactoEmail.Text, Match.Email)))
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
                {Value: "csenergy", Label: "CSEnergy la provee"},
                {Value: "cliente",  Label: "Por parte del cliente"},
                {Value: "conjunta", Label: "Conjunta (CSEnergy + Cliente)"}
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
                  IsBlank(txtContactoNombre.Text),
                  Not(IsMatch(txtContactoEmail.Text, Match.Email)),
                  IsBlank(txtNombreProyecto.Text),
                  IsBlank(ddIngenieriaPor.Selected.Value)
                ),
                Notify("Completa los campos obligatorios.", NotificationType.Error),
                Navigate(scrTableros, ScreenTransition.Cover)
              )
```

---

## Pantalla del tablero — paso "Identificación" (con lógica condicional)

Patrón para los campos clave; el resto de los ~37 campos siguen la misma forma.

```yaml
  scrTableroForm:
    Children:
      - ddTipoEntrega:
          Control: DropDown
          Properties:
            Items: |
              =Table(
                {Value:"tablero", Label:"Tablero Eléctrico"},
                {Value:"sala",    Label:"Sala Eléctrica"},
                {Value:"producto",Label:"Producto Eléctrico"}
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
                {Value:"fuerza",Label:"Fuerza/Potencia"},
                {Value:"alumbrado",Label:"Alumbrado/Distribución BT"},
                {Value:"control",Label:"Control/Automatización"},
                {Value:"transfer",Label:"Transferencia (ATS/MTS)"},
                {Value:"sincronizacion",Label:"Sincronización de Generadores"},
                {Value:"remoto",Label:"Distribución Remoto"},
                {Value:"pfcs",Label:"Factor de Potencia"},
                {Value:"medicion",Label:"Medición/Centro de Carga"},
                {Value:"variadores",Label:"Variadores de Frecuencia"},
                {Value:"arrancadores",Label:"Arrancadores Suaves"},
                {Value:"ups",Label:"UPS/Respaldo"},
                {Value:"otro",Label:"Otro"}
              )
            Value: =Label

      # Grado IP: opciones según interior/exterior
      - ddGradoIP:
          Control: DropDown
          Properties:
            Items: |
              =Switch(ddUbicacion.Selected.Value,
                "interior", ["IP20","IP31","IP43","IP54","IP55","IP65"],
                "exterior", ["IP54","IP55","IP65","IP66","IP67","IP68"],
                ["IP20","IP31","IP43","IP54","IP55","IP65","IP66","IP67","IP68"]
              )

      # Corriente nominal: calculada (sistema bifásico eliminado, decisión B-1)
      - lblCorriente:
          Control: Text
          Properties:
            Text: |
              =With(
                {
                  watts: Value(txtPotencia.Text) * 1000,
                  v: If(ddTension.Selected.Value="otro", Value(txtOtraTension.Text), Value(ddTension.Selected.Value))
                },
                If(Or(watts<=0, v<=0), "",
                  Text(Round(
                    Switch(ddSistema.Selected.Value,
                      "trifasico", watts/(Sqrt(3)*v),
                      "monofasico", watts/v,
                      "dc", watts/v,
                      Blank()
                    ), 1)
                  )
                )
              )
```

---

## Envío — escribir en Dataverse (Patch)

Sintaxis de Choice de Dataverse: `'Columna (Tabla)'.Opcion`.

```yaml
      - btnEnviar:
          Control: Button
          Properties:
            Text: ="Enviar solicitud"
            OnSelect: |
              =If(CountRows(colTableros) = 0,
                Notify("Debe agregar al menos un tablero.", NotificationType.Error),
                // 1) Cabecera (Estado y EstadoPrevio como Choice de Dataverse)
                Set(varSolicitud,
                  Patch(Solicitudes, Defaults(Solicitudes), {
                    Estado: 'Estado (Solicitudes)'.Nueva,
                    NombreProyecto: txtNombreProyecto.Text,
                    UbicacionInstalacion: txtUbicacion.Text,
                    CentroCosto: txtCentroCosto.Text,
                    FechaEntregaDeseada: dtpEntrega.SelectedDate,
                    IngenieriaPor: Switch(ddIngenieriaPor.Selected.Value,
                      "csenergy", 'Ingeniería por (Solicitudes)'.CSEnergy,
                      "cliente",  'Ingeniería por (Solicitudes)'.Cliente,
                      'Ingeniería por (Solicitudes)'.Conjunta
                    ),
                    ContactoNombre: txtContactoNombre.Text,
                    ContactoEmail: txtContactoEmail.Text,
                    ContactoTelefono: txtContactoTelefono.Text,
                    EmpresaCliente: txtEmpresaCliente.Text,
                    EnviadoEl: Now()
                  })
                );
                // 2) Tableros vinculados (relación 1:N de Dataverse)
                ForAll(colTableros As t,
                  Patch(SolicitudTableros, Defaults(SolicitudTableros), {
                    Solicitud: varSolicitud,            // lookup directo al registro padre
                    Title: t.Title,
                    Cantidad: t.Cantidad
                    // ... resto de columnas de t ...
                  })
                );
                Notify("Solicitud enviada.", NotificationType.Success);
                Clear(colTableros);
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

---

## Pendiente

Falta el YAML completo de los ~37 campos del tablero (instalación, parámetros
eléctricos, diseño constructivo, documentación). Siguen exactamente el mismo
patrón mostrado. Pídelo y lo genero completo.
