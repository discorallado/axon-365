> ⚠️ **Reorientar a Model-driven sobre Dataverse ([ADR 0002](../adr/0002-pivote-a-dataverse.md)).**
> Con Dataverse, la bandeja interna conviene construirla como **app Model-driven**
> (vistas, formularios y subgrids casi automáticos) en vez de una Canvas hecha a
> mano. El gating de estados/roles deja de implementarse en la app: lo hacen el
> **Business Process Flow** y los **security roles**. Esta guía Canvas queda como
> alternativa/segundo plano. Pendiente: reescribir como model-driven.

# Guía de construcción — App de gestión interna (bandeja)

Reemplaza el `SubmissionRequestResource` de Filament: la bandeja donde el equipo
lista, filtra y gestiona solicitudes, ve respuestas y adjuntos, cambia estado y
comenta.

> **Licencia:** solo conector **SharePoint** (estándar) → incluido en M365.
> **Quién entra:** usuarios internos con cuenta MS, según roles del ADR.

> **Notación Power Fx:** separador decimal `,` en el entorno → argumentos de
> función con `;`, encadenado de instrucciones con `;;`.

## Decisión de formato: Canvas vs. vistas de lista

| Enfoque | Cuándo conviene |
|---|---|
| **A — Vistas de lista SharePoint** | Lo más barato y rápido. Filtros por `Estado`, agrupación, búsqueda. Suficiente si la gestión es "ver y cambiar estado". Sin maestro-detalle elegante ni acciones guiadas. |
| **B — Power App Canvas (bandeja)** | Maestro-detalle real: lista de solicitudes + panel con sus tableros y adjuntos. Botones de transición que llaman a la lógica de estados. Recomendado para reproducir la experiencia Filament. |

Esta guía detalla **B** (Canvas) y usa vistas de lista (A) como complemento para
reportes rápidos.

---

## 1. Pantalla "Bandeja" (lista maestra)

- **Galería vertical** `galSolicitudes`, `Items`:
  ```powerfx
  SortByColumns(
      Filter(
          Solicitudes;
          // filtro por estado seleccionado (o todos)
          (ddFiltroEstado.Selected.Value = "todos") || (Estado.Value = ddFiltroEstado.Selected.Value)
      );
      "EnviadoEl"; SortOrder.Descending
  )
  ```
- Cada fila: `Title` (código), `NombreProyecto`, `EmpresaCliente`, chip de
  `Estado` con color, `EnviadoEl`, `Asignado`.
- **Chip de estado con color** (reproduce los colores del enum original) —
  `Fill`/`Color` del control:
  ```powerfx
  Switch(ThisItem.Estado.Value;
      "nueva";       RGBA(59;130;246;1);   // info / azul
      "en_revision"; RGBA(245;158;11;1);   // warning / ámbar
      "cotizada";    RGBA(99;102;241;1);   // primary / índigo
      "aprobada";    RGBA(34;197;94;1);    // success / verde
      "rechazada";   RGBA(239;68;68;1);    // danger / rojo
      RGBA(120;120;120;1)
  )
  ```
- **Barra de filtros**: Dropdown `ddFiltroEstado` (todos + 5 estados), caja de
  búsqueda por `NombreProyecto`/`Title`/`EmpresaCliente`, filtro por `Asignado`.
- Al seleccionar una fila: `Set(varSolSel; ThisItem);; Navigate(scrDetalle)`.

> **Delegación:** `Filter`/`SortByColumns` sobre columnas Choice y Texto son
> delegables en SharePoint. Evitar `Search()` sobre muchos campos y funciones no
> delegables (advertencia azul) para no truncar a 500/2000 ítems.

---

## 2. Pantalla "Detalle" (maestro-detalle)

Reproduce `ViewSubmissionRequest`.

### 2.1 Cabecera
Mostrar todos los campos de `varSolSel`: contacto, proyecto, centro de costo,
fecha deseada, ingeniería por, observaciones, asignado, estado actual.

### 2.2 Tableros de la solicitud
- **Galería** `galTablerosDet`, `Items`:
  ```powerfx
  Filter(SolicitudTableros; Solicitud.Id = varSolSel.ID)
  ```
  (La columna Lookup `Solicitud` debe estar **indexada** — ver esquema.)
- Mostrar por tablero: `Title`, tipo (traducir código→etiqueta), cantidad,
  parámetros eléctricos clave, IP/IK, y un sub-detalle expandible con el resto.
- **Traducción código→etiqueta** (los Choice guardan el código corto):
  ```powerfx
  Switch(ThisItem.TipoTablero.Value;
      "fuerza"; "Fuerza / Potencia";
      "alumbrado"; "Alumbrado / Distribución BT";
      "control"; "Control / Automatización";
      "transfer"; "Transferencia (ATS/MTS)";
      "sincronizacion"; "Sincronización de Generadores";
      "remoto"; "Distribución Remoto";
      "pfcs"; "Factor de Potencia";
      "medicion"; "Medición / Centro de Carga";
      "variadores"; "Variadores de Frecuencia";
      "arrancadores"; "Arrancadores Suaves";
      "ups"; "UPS / Respaldo";
      "otro"; Coalesce(ThisItem.OtroTipoTablero; "Otro");
      ThisItem.TipoTablero.Value
  )
  ```

### 2.3 Adjuntos
- Mostrar adjuntos nativos del ítem (cabecera y de cada tablero). Si se adopta la
  biblioteca de documentos con metadato (decisión M-1), filtrar por la solicitud y
  agrupar por `TipoArchivo`.

### 2.4 Comentarios internos
Reemplaza los comentarios Parallax. Opciones:
- Lista `Comentarios` (Lookup → Solicitud, `Texto`, `Autor`, `Fecha`) + galería.
- O la columna de comentarios/actividad nativa si se usa una lista con esa feature.

---

## 3. Acciones de cambio de estado (el núcleo)

Botones que aparecen **según el estado actual y el rol del usuario**, replicando
`allowedNextStatuses()` + las reglas de rol del original.

### 3.1 Rol del usuario actual
SharePoint no expone roles de negocio directamente. Definir una lista
`RolesUsuarios` (`Usuario` Persona, `Rol` Choice) y resolver:
```powerfx
// App.OnStart
Set(varRol;
    LookUp(RolesUsuarios; Usuario.Email = User().Email; Rol.Value)
)
```
> Alternativa: usar grupos de SharePoint/Azure AD y `Office365Groups`. Decidir
> según cómo se administren los roles (ver A-3, decisión abierta).

### 3.2 Estados siguientes permitidos
```powerfx
// colАcciones: estados a los que se puede pasar desde el actual
With({s: varSolSel.Estado.Value};
    Switch(s;
        "nueva";       ["en_revision";"rechazada"];
        "en_revision"; ["cotizada";"rechazada"];
        "cotizada";    ["aprobada";"rechazada";"en_revision"];
        // terminales: solo reapertura
        "aprobada";    ["nueva"];
        "rechazada";   ["nueva"];
        []
    )
)
```

### 3.3 Gating por rol (antes de mostrar/permitir cada botón)
```powerfx
// ¿Puede el usuario hacer ESTA transición?
With({destino: thisDestino};   // p.ej. "rechazada"
    Switch(destino;
        "rechazada"; varRol in ["super_admin";"supervisor"];
        "nueva";     varRol = "super_admin";                       // reapertura
        // avanzar (en_revision/cotizada/aprobada)
        varRol in ["super_admin";"ingeniero";"supervisor"]
    )
)
```

### 3.4 Ejecutar la transición
El botón **solo** cambia `Estado`; **F-2 valida y audita** (no dupliques la
lógica en la app). Pedir comentario antes:
```powerfx
// OnSelect del botón de transición
Patch(Solicitudes; varSolSel; {
    Estado: {Value: thisDestino};
    Comentario: txtComentarioTransicion.Text   // si guardas el comentario en la solicitud
});;
Set(varSolSel; LookUp(Solicitudes; ID = varSolSel.ID));;  // refrescar
Notify("Estado actualizado."; NotificationType.Success)
```
> Importante: la validación dura vive en **F-2** (servidor), no en la app. La app
> solo evita ofrecer transiciones inválidas para UX. Si la app y F-2 difieren,
> manda F-2 (revierte lo inválido). Mantener ambas listas de transiciones en sync.

### 3.5 Asignar responsable
```powerfx
Patch(Solicitudes; varSolSel; {Asignado: pickerAsignado.Selected})
```
Solo visible si `varRol in ["super_admin";"ingeniero";"supervisor"]` (policy `assign`).

---

## 4. Exportación (reemplaza export Excel/PDF)

El original exporta una solicitud o lote a Excel/PDF. En M365:
- **Excel:** botón que llama a un flujo Power Automate "Exportar solicitud" →
  crea fila(s) en una plantilla Excel (Office Scripts) o genera CSV. O usar la
  exportación nativa de la **vista de lista** SharePoint para lotes.
- **PDF:** flujo Power Automate con **"Convertir a PDF"** (OneDrive/SharePoint) a
  partir de una plantilla Word rellenada con los datos de la solicitud.

> Esto es premium-free mientras use conectores estándar (SharePoint, OneDrive,
> Word/Excel Online). Confirmar que el flujo no agregue conectores premium.

---

## 5. Vistas de lista SharePoint (complemento, enfoque A)

Aun con la app Canvas, crear vistas en `Solicitudes` para reportes rápidos:
- **"Nuevas"** — filtro `Estado = nueva`, orden por `EnviadoEl` desc.
- **"En proceso"** — `Estado in (en_revision, cotizada)`.
- **"Cerradas"** — `Estado in (aprobada, rechazada)`.
- **"Mis asignadas"** — `Asignado = [Yo]`.
- Agrupar por `Estado`; columnas: Title, NombreProyecto, EmpresaCliente, Asignado, EnviadoEl.

---

## 6. Equivalencias con Filament

| Filament original | App de gestión Canvas |
|---|---|
| `SubmissionRequestResource` (tabla) | Pantalla Bandeja (`galSolicitudes`) + filtros |
| `ViewSubmissionRequest` (infolist) | Pantalla Detalle (cabecera + tableros + adjuntos) |
| `RepeatableEntry` de items | `galTablerosDet` filtrada por Lookup |
| Acción cambiar estado + comentario | Sección 3 (botones gateados + Patch + F-2) |
| `allowedNextStatuses()` | `colAcciones` (3.2) |
| Reglas de rol del `SubmissionStateMachine` | gating 3.3 + validación dura en F-2 |
| Asignar responsable (policy `assign`) | sección 3.5 |
| Comentarios Parallax | lista `Comentarios` (2.4) |
| Export Excel/PDF | flujos Power Automate (sección 4) |
| Badge de estado con color | `Switch` de color (sección 1) |
