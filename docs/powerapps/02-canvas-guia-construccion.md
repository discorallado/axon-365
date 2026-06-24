# Guía de construcción — Power App Canvas de captura

App que reemplaza al `PublicFormWizard` de Filament. Tres pantallas, una galería
editable para el patrón multi-tablero, y fórmulas Power Fx que reproducen la
lógica condicional y el cálculo automático de corriente del formulario original.

> **Licencia:** usa solo el conector **SharePoint** (estándar). No requiere
> Power Apps premium. Confirmar que la app no agregue conectores premium.

---

## 0. Preparación

1. En **make.powerapps.com** → **Crear** → **App de lienzo en blanco** →
   formato **Tableta** (más cómodo para los ~35 campos por tablero).
2. **Datos** → **Agregar datos** → conector **SharePoint** → seleccionar el sitio
   y las 3 listas: `Solicitudes`, `SolicitudTableros`, `HistorialEstados`.

---

## 1. Modelo de datos en memoria

El formulario original arma la solicitud en memoria y guarda todo al final
(cabecera + N tableros). Replicamos eso con una **colección local** de tableros
y variables de contexto para la cabecera.

```powerfx
// App.OnStart
ClearCollect(colTableros, []);   // tableros de la solicitud en curso
Set(varEditIndex, Blank());      // índice del tablero en edición (Blank = nuevo)
```

---

## 2. Pantalla 1 — Contacto y Proyecto

Controles ligados a variables de contexto (no se guardan aún). Campos:

| Control | Variable | Validación |
|---|---|---|
| Texto `txtContactoNombre` | — | Requerido |
| Texto `txtContactoEmail` | — | Requerido + formato email |
| Texto `txtContactoTelefono` | — | Prefijo "+56" como etiqueta |
| Texto `txtNombreProyecto` | — | Requerido |
| Texto `txtEmpresaCliente` | — | |
| Texto `txtUbicacion` | — | |
| Texto `txtCentroCosto` | — | placeholder `MO-12345` |
| DatePicker `dtpEntrega` | — | |
| Dropdown `ddIngenieriaPor` | — | Requerido: CSEnergy / Cliente / Conjunta |

**Validación de email** (en `OnSelect` del botón Siguiente o en `DisplayMode`):

```powerfx
IsMatch(txtContactoEmail.Text, Match.Email)
```

Botón **Siguiente** → `Navigate(scrTableros, ScreenTransition.Cover)`.

---

## 3. Pantalla 2 — Tableros (galería editable, el patrón multi-tablero)

Esta es la pieza que Microsoft Forms no podía replicar. Una galería lista los
tableros agregados; un botón abre un formulario para agregar/editar.

### 3.1 Galería de tableros

- **Galería vertical** `galTableros`, `Items = colTableros`.
- En cada fila mostrar: `ThisItem.Title` (nombre), etiqueta de tipo, `×Cantidad`,
  y botones **Editar** / **Eliminar**.

**Eliminar** (`OnSelect`):
```powerfx
Remove(colTableros, ThisItem)
```

**Editar** (`OnSelect`):
```powerfx
Set(varEditIndex, ThisItem);
// cargar los valores del tablero en los controles del formulario (sección 3.2)
Navigate(scrTableroForm)
```

**Agregar tablero** (botón bajo la galería):
```powerfx
Set(varEditIndex, Blank());
Navigate(scrTableroForm)
```

Mensaje cuando está vacía: "Aún no hay tableros en esta solicitud."

### 3.2 Pantalla/formulario del tablero (`scrTableroForm`)

Reproduce los 3 pasos del modal original (Identificación / Instalación y
Eléctrico / Documentación). Controles para cada columna de `SolicitudTableros`.

**Lógica condicional (campos que aparecen/desaparecen)** — `Visible` de cada control:

```powerfx
// TipoTablero: solo si es un tablero
ddTipoTablero.Visible = (ddTipoEntrega.Selected.Value = "tablero")

// OtroTipoTablero: solo si eligió "otro"
txtOtroTipoTablero.Visible = (ddTipoTablero.Selected.Value = "otro")

// OtroAmbienteEspecial: solo si el multiselect incluye "otro"
txtOtroAmbiente.Visible = ("otro" in ddAmbienteEspecial.SelectedItems.Value)

// Dimensiones máximas: solo si hay restricciones
grpDimensiones.Visible = tglRestricciones.Value

// OtraTension / OtroSistema / OtraFrecuencia
txtOtraTension.Visible    = (ddTension.Selected.Value = "otro")
txtOtroSistema.Visible    = (ddSistema.Selected.Value = "otro")
txtOtraFrecuencia.Visible = (ddFrecuencia.Selected.Value = "otro")
```

**Filtrado de `GradoIP` según `Ubicacion`** (el original cambia las opciones
válidas):

```powerfx
// ddGradoIP.Items
Switch(
    ddUbicacion.Selected.Value,
    "interior", ["IP20","IP31","IP43","IP54","IP55","IP65"],
    "exterior", ["IP54","IP55","IP65","IP66","IP67","IP68"],
    ["IP20","IP31","IP43","IP54","IP55","IP65","IP66","IP67","IP68"]
)
```

**Cálculo automático de `CorrienteNominal`** (reproduce `recalculateCurrent`):

```powerfx
// CorrienteNominal — valor calculado (Default del control, o en OnChange de los inputs)
With(
    {
        watts: Value(txtPotencia.Text) * 1000,
        v: If(ddTension.Selected.Value = "otro",
               Value(txtOtraTension.Text),
               Value(ddTension.Selected.Value))
    },
    If(
        watts <= 0 || v <= 0,
        Blank(),
        Round(
            Switch(
                ddSistema.Selected.Value,
                "trifasico", watts / (Sqrt(3) * v),
                "bifasico",  watts / (v * 2),
                "monofasico", watts / v,
                "dc",         watts / v,
                Blank()
            ),
            1
        )
    )
)
```

> Nota: la fórmula original usa potencia en kW/kVA × 1000. Si `UnidadPotencia` es
> kVA, el resultado es corriente aparente; se mantiene igual que el sistema
> original (no convierte kVA↔kW).

### 3.3 Guardar el tablero en la colección

Botón **Guardar tablero** (`OnSelect`):

```powerfx
If(
    IsBlank(varEditIndex),
    // nuevo
    Collect(colTableros, {
        Title: txtNombreTablero.Text,
        Cantidad: Value(txtCantidad.Text),
        Orden: CountRows(colTableros),
        TipoEntrega: ddTipoEntrega.Selected.Value,
        TipoTablero: ddTipoTablero.Selected.Value,
        // ... resto de columnas ...
        CorrienteNominal: lblCorrienteCalculada.Text
    }),
    // edición: reemplazar el item existente
    Patch(colTableros, varEditIndex, {
        Title: txtNombreTablero.Text,
        // ... resto de columnas ...
    })
);
Set(varEditIndex, Blank());
Navigate(scrTableros)
```

---

## 4. Pantalla 3 — Documentación y envío

- Controles de adjuntos a nivel solicitud (specs técnicas, fotos del sitio).
  En Canvas + SharePoint, los adjuntos se suben con el control **Adjuntar
  archivo** del formulario de edición de SharePoint, o se cargan tras crear el
  ítem (ver nota de adjuntos abajo).
- Textarea `txtObservacionesProyecto`.
- Botón **Enviar solicitud**.

### 4.1 Validación antes de enviar

```powerfx
// El original exige al menos un tablero
If(
    CountRows(colTableros) = 0,
    Notify("Debe agregar al menos un tablero antes de enviar.", NotificationType.Error),
    // ... continuar con el guardado (4.2)
)
```

### 4.2 Guardado: cabecera + tableros

```powerfx
// 1) Crear la cabecera y capturar el registro creado
Set(varSolicitud,
    Patch(Solicitudes, Defaults(Solicitudes), {
        // Title (reference_code) lo completa Power Automate; dejar provisional
        Title: "PENDIENTE",
        Estado: {Value: "nueva"},
        EstadoPrevio: {Value: "nueva"},   // necesario para F-2 (ver revisión A-2)
        NombreProyecto: txtNombreProyecto.Text,
        UbicacionInstalacion: txtUbicacion.Text,
        CentroCosto: txtCentroCosto.Text,
        FechaEntregaDeseada: dtpEntrega.SelectedDate,
        IngenieriaPor: {Value: ddIngenieriaPor.Selected.Value},
        ContactoNombre: txtContactoNombre.Text,
        ContactoEmail: txtContactoEmail.Text,
        ContactoTelefono: txtContactoTelefono.Text,
        EmpresaCliente: txtEmpresaCliente.Text,
        ObservacionesProyecto: txtObservacionesProyecto.Text,
        EnviadoEl: Now(),
        Organizacion: "CSEnergy"
    })
);

// 2) Crear cada tablero vinculado a la cabecera
ForAll(
    colTableros As t,
    Patch(SolicitudTableros, Defaults(SolicitudTableros),
        {
            Solicitud: {
                Id: varSolicitud.ID,
                Value: varSolicitud.Title
            }
        },
        // mapear todas las columnas de t a las columnas de la lista
        Sustituir_por_columnas_de_t
    )
);

// 3) Limpiar y confirmar
Notify("Solicitud enviada. Recibirás un correo de confirmación.", NotificationType.Success);
Clear(colTableros);
Navigate(scrConfirmacion)
```

> **Patrón de referencia (`reference_code`):** lo genera Power Automate al crear
> la cabecera (ver flujo F-1), reemplazando "PENDIENTE" por `SOL-XXXXXXXX`. Así no
> dependemos de lógica de aleatoriedad en la app.

> **Adjuntos en Canvas + SharePoint:** el control nativo de adjuntos solo funciona
> sobre un control **Editar formulario** ligado a un ítem existente. Dos enfoques:
> (a) tras crear el ítem, navegar a un formulario de edición con el control
> Adjuntos para subir archivos; (b) usar `'Crear archivo'` del conector SharePoint
> hacia una biblioteca de documentos. Recomendado (a) por simplicidad.

---

## 5. Pantalla de confirmación

Muestra el código de referencia (releer `varSolicitud` tras el flujo, o mostrar
"Tu solicitud fue registrada"). Botón para iniciar otra solicitud
(`Clear(colTableros); Navigate(scrInicio)`).

---

## 6. Equivalencias con el formulario original

| Filament / Livewire original | Power Apps Canvas |
|---|---|
| Wizard de 3 pasos | 3 pantallas + navegación |
| Modal "Agregar/Editar tablero" repetible | Galería editable + `scrTableroForm` |
| `->visible(fn($get)=>...)` condicionales | propiedad `Visible` con Power Fx |
| `recalculateCurrent()` | fórmula `With(...)` de la sección 3.2 |
| Opciones de IP según interior/exterior | `Switch()` en `ddGradoIP.Items` |
| `->required()` | validación en `OnSelect` + `Notify` |
| Notificaciones al enviar | Power Automate (flujo F-1) |
