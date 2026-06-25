> ⚠️ **SUPERSEDED por el pivote a Dataverse ([ADR 0002](../adr/0002-pivote-a-dataverse.md)).**
> El backend ya NO es SharePoint. Este esquema de columnas/choices sigue siendo
> útil como **referencia del modelo de datos** (los campos y opciones son los
> mismos), pero las "listas" pasan a ser **tablas Dataverse**, los Lookup a
> **relaciones 1:N nativas**, `HistorialEstados` a **auditoría nativa**, y
> `EstadoPrevio` se elimina. Pendiente: reescribir como `dataverse/01-tablas.md`.

# Esquema de listas SharePoint

Tres listas reproducen el modelo relacional del sistema original:

```
Solicitudes (1) ──< SolicitudTableros (N)
     │
     └──< HistorialEstados (N)   ← auditoría de cambios de estado
```

- Adjuntos: **adjuntos nativos** de cada ítem de lista (no se necesita tabla aparte).
- `organization_id` original → columna `Organizacion` (texto fijo por ahora,
  single-tenant; se mantiene para multi-tenant-ready, igual que en el diseño Laravel).
- Las relaciones 1:N se implementan con columnas **Lookup**.

> Notación: **Title** es la columna obligatoria que SharePoint crea por defecto en
> toda lista. Cuando se reutiliza para otro fin, se indica.

---

## Lista 1 — `Solicitudes` (cabecera)

Equivale a `SubmissionRequest`.

| Columna | Tipo SharePoint | Notas / origen |
|---|---|---|
| `Title` | Una línea de texto | Reutilizada como **código de referencia** `SOL-XXXXXXXX`. Se rellena por Power Automate al crear (ver flujos). |
| `Estado` | Elección | `nueva`, `en_revision`, `cotizada`, `aprobada`, `rechazada`. Valor por defecto: `nueva`. |
| `EstadoPrevio` | Elección (oculta) | Mismas 5 opciones. Por defecto `nueva`. La usa F-2 para conocer el estado anterior y validar la transición. No mostrar en formularios de usuario. |
| `NombreProyecto` | Una línea de texto | `project_name`. Obligatoria. |
| `UbicacionInstalacion` | Una línea de texto | `installation_location` |
| `CentroCosto` | Una línea de texto | `cost_center` (ej. `MO-12345`) |
| `FechaEntregaDeseada` | Fecha | `desired_delivery_date` (solo fecha) |
| `IngenieriaPor` | Elección | `csenergy` (CSEnergy la provee), `cliente` (Por parte del cliente), `conjunta` (Conjunta). Obligatoria. |
| `ContactoNombre` | Una línea de texto | `submitter_name`. Obligatoria. |
| `ContactoEmail` | Una línea de texto | `submitter_email`. Obligatoria. Validación de formato email en la app. |
| `ContactoTelefono` | Una línea de texto | `submitter_phone` (prefijo +56 en la app) |
| `EmpresaCliente` | Una línea de texto | `submitter_company` |
| `ObservacionesProyecto` | Varias líneas de texto (texto plano) | `project_observations` |
| `Asignado` | Persona o grupo | `assigned_to`. Quién gestiona la solicitud internamente. |
| `EnviadoEl` | Fecha y hora | `submitted_at`. Se setea por la app/flujo al enviar. |
| `Organizacion` | Una línea de texto | `organization_id`. Valor fijo por ahora (single-tenant). |

> **Adjuntos de cabecera** (`technical_specs`, `site_photos`): usar los adjuntos
> nativos de SharePoint en el ítem de `Solicitudes`. No requieren columna.

> **`raw_data` (JSON snapshot):** se descarta — el versionado se confía al
> Historial de versiones nativo de SharePoint (decisión R-3 del ADR).

---

## Lista 2 — `SolicitudTableros` (los tableros)

Equivale a `SubmissionItem`. Relación 1:N con `Solicitudes`.

> **Modelo ancho (decidido), no EAV.** Se evaluó un modelo Entidad-Atributo-Valor
> (una lista `id, campo, respuesta, fecha` con ~37 filas por tablero) y se descartó:
> en SharePoint multiplicaría las filas ~37× y chocaría el umbral de 5.000 ítems
> con apenas ~45 solicitudes, además de perder el tipado (Number/Date/Choice), el
> cálculo de corriente, y la delegación en Power Apps. Los 37 campos son fijos y
> conocidos, así que el modelo ancho (1 fila por tablero) es el correcto.

### Columna de relación

| Columna | Tipo SharePoint | Notas |
|---|---|---|
| `Solicitud` | **Búsqueda (Lookup)** → `Solicitudes` | Vincula cada tablero a su solicitud. Mostrar columna `Title` (código). Obligatoria. **Indexar esta columna** (ver nota de umbral 5.000 abajo). |

### Identificación

| Columna | Tipo | Opciones / notas |
|---|---|---|
| `Title` | Una línea de texto | Reutilizada como **nombre del tablero** (`label`). Ej.: "TG Principal". Obligatoria. |
| `Cantidad` | Número (entero) | `quantity`. Mín. 1, máx. 999. Por defecto 1. |
| `Orden` | Número (entero) | `sort_order`. Orden dentro de la solicitud. |
| `TipoEntrega` | Elección | `delivery_type`: `tablero` (Tablero Eléctrico), `sala` (Sala Eléctrica), `producto` (Producto Eléctrico). Obligatoria. |
| `InstalacionNuevaReemplazo` | Elección | `is_new_installation`: `nueva`, `reemplazo`. Obligatoria. |
| `TipoTablero` | Elección | `board_type`. Solo aplica si `TipoEntrega = tablero`. Opciones abajo. |
| `OtroTipoTablero` | Una línea de texto | `other_board_type`. Solo si `TipoTablero = otro`. |
| `FuncionTablero` | Varias líneas de texto (**enriquecido**) | `board_function`. Conserva formato (RichEditor original). |
| `CargasAAlimentar` | Varias líneas de texto (**enriquecido**) | `loads_to_feed` |
| `NumeroCircuitos` | Número (entero) | `number_of_circuits` |

**Opciones de `TipoTablero`** (`board_type`):
`fuerza` (Tablero de Fuerza/Potencia), `alumbrado` (Alumbrado/Distribución BT),
`control` (Control/Automatización), `transfer` (Transferencia ATS/MTS),
`sincronizacion` (Sincronización de Generadores), `remoto` (Distribución Remoto),
`pfcs` (Factor de Potencia), `medicion` (Medición/Centro de Carga),
`variadores` (Variadores de Frecuencia VFD), `arrancadores` (Arrancadores Suaves SS),
`ups` (UPS/Respaldo), `otro` (Otro).

### Ubicación y ambiente

| Columna | Tipo | Opciones / notas |
|---|---|---|
| `Ubicacion` | Elección | `location_type`: `interior`, `exterior`. Obligatoria. |
| `AmbienteEspecial` | **Elección múltiple** | `special_environment`: `marino`, `minero`, `humedo`, `corrosivo`, `polvoriento`, `explosivo`, `otro`. (Opción A aprobada.) |
| `OtroAmbienteEspecial` | Una línea de texto | `other_special_environment`. Solo si incluye `otro`. |
| `GradoIP` | Elección | `ip_rating`: `IP20`,`IP31`,`IP43`,`IP54`,`IP55`,`IP65`,`IP66`,`IP67`,`IP68`. Obligatoria. El subconjunto válido depende de `Ubicacion` (se filtra en la app, no en SharePoint). |
| `GradoIK` | Elección | `ik_rating`: `IK06`,`IK07`,`IK08`,`IK09`,`IK10`. |

### Montaje y dimensiones

| Columna | Tipo | Opciones / notas |
|---|---|---|
| `TipoMontaje` | Elección | `mounting_type`: `autosoportado`, `mural`, `rack_19`, `pedestal`, `otro`. Obligatoria. |
| `RestriccionesDimension` | Sí/No | `has_dimension_restrictions` |
| `AltoMaxMm` | Número | `max_height` (mm). Solo si `RestriccionesDimension = Sí`. |
| `AnchoMaxMm` | Número | `max_width` (mm) |
| `FondoMaxMm` | Número | `max_depth` (mm) |
| `CondicionesInstalacion` | Varias líneas de texto | `additional_installation_conditions` |

### Parámetros eléctricos

| Columna | Tipo | Opciones / notas |
|---|---|---|
| `TensionSuministro` | Elección | `supply_voltage`: `220`,`380`,`400`,`440`,`480`,`690`,`1000`,`otro`. Obligatoria. |
| `OtraTension` | Número | `supply_voltage_other` (V). Solo si `otro`. |
| `SistemaElectrico` | Elección | `electrical_system`: `trifasico`, `monofasico`, `dc`, `otro`. Obligatoria. **Sin `bifasico`** (eliminado en M365, decisión B-1; el repo original lo conserva). |
| `OtroSistemaElectrico` | Una línea de texto | `electrical_system_other`. Solo si `otro`. |
| `PotenciaEstimada` | Número (decimal) | `estimated_power`. Obligatoria. |
| `UnidadPotencia` | Elección | `power_unit`: `kW`, `kVA`. Por defecto `kW`. |
| `CorrienteNominal` | Número (decimal) | `nominal_current`. **Calculado automáticamente** en la app (ver guía Power Fx). |
| `Frecuencia` | Elección | `frequency`: `50`, `60`, `otro`. Obligatoria. |
| `OtraFrecuencia` | Número | `other_frequency` (Hz). Solo si `otro`. |
| `ProteccionesRequeridas` | **Elección múltiple** | `required_protections`: `interruptor_automatico`, `diferencial`, `fusible`, `relevo_sobrecarga`, `relevo_falla_tierra`, `proteccion_tension`, `proteccion_corriente`, `descargador_tension`, `otro`. Obligatoria. |
| `MarcasPreferidas` | **Elección múltiple** | `preferred_brands`: `schneider`, `siemens`, `abb`, `legrand`, `eaton`, `chint`, `hager`, `weidmuller`, `phoenix`, `otro`. |

### Diseño constructivo

| Columna | Tipo | Opciones / notas |
|---|---|---|
| `MaterialGabinete` | Elección | `cabinet_material`: `acero_pintado`, `acero_galvanizado`, `acero_inoxidable` (304), `acero_inox_316`, `fibra_vidrio`, `poliester`, `aluminio`. Obligatoria. |
| `ColorGabinete` | Elección | `special_color`: `7035`,`7016`,`9016`,`9005`,`5010`,`6005`,`otro` (RAL). Por defecto `7035`. |
| `TipoVentilacion` | Elección | `ventilation_type`: `natural`, `forzada`, `sellado`, `climatizado`. Obligatoria. |
| `ExpansionFutura` | Elección | `future_expansion`: `no`, `10`, `20`, `30`, `otro`. Obligatoria. |
| `ObservacionesTablero` | Varias líneas de texto | `additional_observations` |

> **Adjuntos del tablero** (`load_list_file`, `unilineal_diagram`,
> `mechanical_plans`): adjuntos nativos del ítem de `SolicitudTableros`.

---

## Lista 3 — `HistorialEstados` (auditoría)

Equivale a `SubmissionStatusHistory`. La escribe Power Automate en cada cambio de
estado (no se edita a mano).

| Columna | Tipo | Notas |
|---|---|---|
| `Solicitud` | Búsqueda (Lookup) → `Solicitudes` | A qué solicitud pertenece. |
| `Organizacion` | Una línea de texto | `organization_id`. Multi-tenant-ready, igual que `SubmissionStatusHistory`. |
| `EstadoAnterior` | Elección | `from_status`. Mismas 5 opciones de `Estado`. |
| `EstadoNuevo` | Elección | `to_status`. |
| `CambiadoPor` | Persona o grupo | `changed_by`. |
| `Comentario` | Varias líneas de texto | `comment`. Motivo de la transición. |
| `FechaCambio` | Fecha y hora | `created_at`. |

---

## Notas de implementación

- **Valores internos vs. etiquetas:** SharePoint guarda el texto visible de cada
  Choice. Para que el dato siga siendo limpio, usar como valor de cada opción el
  **código corto original** (`fuerza`, `IP54`, etc.) y poner la etiqueta legible
  en la app. Alternativa más amigable: usar etiquetas en español directamente como
  opción (ej. "Interior") — decisión menor, ver `docs/powerapps`.
- **Filtrado de `GradoIP` según `Ubicacion`:** SharePoint no soporta choices
  condicionales; el subconjunto válido (interior vs. exterior) se filtra en la
  Power App.
- **Cálculo de `CorrienteNominal`:** no se calcula en SharePoint; se hace en la
  app con Power Fx (fórmula en la guía de Power Apps).
- **Permisos:** ver matriz de roles en el ADR. Se implementan como permisos de
  lista SharePoint (grupos: Administradores, Gestores, Lectores). **Limitación:**
  los permisos de lista son gruesos y NO replican el detalle por acción/rol de las
  policies originales (p. ej. "solo super_admin reabre"). Esa granularidad, de
  necesitarse, debe ir en los flujos Power Automate (ver A-3 de la revisión).
- **Umbral de 5.000 ítems (list view threshold):** `SolicitudTableros` crece N por
  solicitud y puede superar el umbral. Indexar la columna Lookup `Solicitud` y
  cualquier columna usada para filtrar/ordenar en vistas. Sin índice, las vistas y
  consultas fallan pasado ese límite.
- **Adjuntos (decisión M-1 resuelta: adjuntos nativos planos).** Se usan los
  adjuntos nativos del ítem de lista. El sistema original etiquetaba cada archivo
  (`technical_specs`, `site_photo`, `load_list`, `unilineal_diagram`,
  `mechanical_plans`) y los agrupaba por tag; con adjuntos nativos **se pierde esa
  categorización** (se sabe cuántos archivos hay, no cuál es el unilineal salvo por
  el nombre). Mitigación: **convención de nombres** al subir (p. ej.
  `unilineal_<codigo>.pdf`). La alternativa fiel —biblioteca de documentos con
  columna `TipoArchivo` + Lookup— se descartó por complejidad: el control de
  adjuntos de Canvas no escribe metadatos, obligaría a subir vía conector
  `Crear archivo` + `Actualizar propiedades`. Reconsiderar solo si la falta de
  categoría molesta en operación.
