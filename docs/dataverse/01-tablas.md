# Esquema de tablas Dataverse

Reemplaza a `docs/sharepoint/01-listas-esquema.md` (superseded). Dos tablas con
relación 1:N nativa; la auditoría y el historial de estados los da la plataforma.

```
Solicitud (1) ──< SolicitudTablero (N)   ← relación 1:N, comportamiento Parental (cascade)
```

- **`reference_code`** → columna **Autonumber** (Primary Name), secuencial y única.
- **`organization_id`** → columna `Organizacion` en cada tabla (multi-tenant-ready).
- **Auditoría:** activar *Audit* en el entorno y en ambas tablas → registra
  quién/cuándo/valor anterior→nuevo. Reemplaza la lista `HistorialEstados`.
- **`EstadoPrevio`** ya **no existe** (lo gestiona el Business Process Flow).
- **Adjuntos:** columnas **File** por slot, o tabla relacionada `Archivo` con
  Choice `TipoArchivo` (recomendado si se quiere categoría; ver §3).

> Convención de nombres: Dataverse antepone el prefijo de editor de la solución
> (p. ej. `csen_`). Aquí se usan nombres lógicos legibles; el prefijo lo agrega la
> solución al crear.

---

## Tabla 1 — `Solicitud`

Equivale a `SubmissionRequest`.

| Columna (display) | Tipo Dataverse | Notas / origen |
|---|---|---|
| `ReferenceCode` (Primary Name) | **Autonumber** | `reference_code`. Formato `SOL-{SEQNUM:00000}`. Único, lo asigna la plataforma. |
| `Estado` | **Choice** | `nueva`, `en_revision`, `cotizada`, `aprobada`, `rechazada`. Default `nueva`. La controla el BPF. |
| `NombreProyecto` | Texto (1 línea) | `project_name`. Requerida. |
| `UbicacionInstalacion` | Texto (1 línea) | `installation_location` |
| `CentroCosto` | Texto (1 línea) | `cost_center` |
| `FechaEntregaDeseada` | Fecha (Date Only) | `desired_delivery_date` |
| `IngenieriaPor` | **Choice** | `csenergy`, `cliente`, `conjunta`. Requerida. |
| `ContactoNombre` | Texto (1 línea) | `submitter_name`. Requerida. |
| `ContactoEmail` | Texto (1 línea, formato Email) | `submitter_email`. Requerida. |
| `ContactoTelefono` | Texto (1 línea, formato Phone) | `submitter_phone` |
| `EmpresaCliente` | Texto (1 línea) | `submitter_company` |
| `ObservacionesProyecto` | Texto (varias líneas) | `project_observations` |
| `Asignado` | **Lookup → Usuario (systemuser)** | `assigned_to` |
| `EnviadoEl` | Fecha y hora | `submitted_at` |
| `Organizacion` | Texto (1 línea) o Lookup | `organization_id`. Single-tenant por ahora. |

> `raw_data` (JSON snapshot) se descarta: la auditoría nativa cubre el versionado.
> `ip_address` / `user_agent`: opcionales como Texto si se quiere trazabilidad anti-spam.

---

## Tabla 2 — `SolicitudTablero`

Equivale a `SubmissionItem`. **Modelo ancho** (1 fila por tablero, no EAV).

### Relación

| Columna | Tipo | Notas |
|---|---|---|
| `Solicitud` | **Lookup → Solicitud** | Creada por la relación 1:N. Comportamiento **Parental** (cascade delete/assign), replicando el cascade del original. |

### Identificación

| Columna | Tipo | Opciones / notas |
|---|---|---|
| `Nombre` (Primary Name) | Texto (1 línea) | `label`. Ej. "TG Principal". Requerida. |
| `Cantidad` | Número entero | `quantity`. Mín 1, máx 999. Default 1. |
| `Orden` | Número entero | `sort_order` |
| `TipoEntrega` | Choice | `tablero`, `sala`, `producto`. Requerida. |
| `InstalacionNuevaReemplazo` | Choice | `nueva`, `reemplazo`. Requerida. |
| `TipoTablero` | Choice | Solo si `TipoEntrega = tablero`. 12 opciones (abajo). |
| `OtroTipoTablero` | Texto (1 línea) | `other_board_type`. Si `TipoTablero = otro`. |
| `FuncionTablero` | Texto (varias líneas, formato enriquecido) | `board_function` |
| `CargasAAlimentar` | Texto (varias líneas, enriquecido) | `loads_to_feed` |
| `NumeroCircuitos` | Número entero | `number_of_circuits` |

**`TipoTablero`:** `fuerza`, `alumbrado`, `control`, `transfer`, `sincronizacion`,
`remoto`, `pfcs`, `medicion`, `variadores`, `arrancadores`, `ups`, `otro`.

### Ubicación y ambiente

| Columna | Tipo | Opciones / notas |
|---|---|---|
| `Ubicacion` | Choice | `interior`, `exterior`. Requerida. |
| `AmbienteEspecial` | **Choice (multi-select)** | `marino`, `minero`, `humedo`, `corrosivo`, `polvoriento`, `explosivo`, `otro`. |
| `OtroAmbienteEspecial` | Texto (1 línea) | Si incluye `otro`. |
| `GradoIP` | Choice | `IP20`…`IP68`. El subconjunto válido según `Ubicacion` se filtra en la app. Requerida. |
| `GradoIK` | Choice | `IK06`…`IK10`. |

### Montaje y dimensiones

| Columna | Tipo | Opciones / notas |
|---|---|---|
| `TipoMontaje` | Choice | `autosoportado`, `mural`, `rack_19`, `pedestal`, `otro`. Requerida. |
| `RestriccionesDimension` | Sí/No | `has_dimension_restrictions` |
| `AltoMaxMm` / `AnchoMaxMm` / `FondoMaxMm` | Número entero | mm. Si `RestriccionesDimension = Sí`. |
| `CondicionesInstalacion` | Texto (varias líneas) | `additional_installation_conditions` |

### Parámetros eléctricos

| Columna | Tipo | Opciones / notas |
|---|---|---|
| `TensionSuministro` | Choice | `220`,`380`,`400`,`440`,`480`,`690`,`1000`,`otro`. Requerida. |
| `OtraTension` | Número entero | V. Si `otro`. |
| `SistemaElectrico` | Choice | `trifasico`, `monofasico`, `dc`, `otro`. **Sin `bifasico`** (B-1). Requerida. |
| `OtroSistemaElectrico` | Texto (1 línea) | Si `otro`. |
| `PotenciaEstimada` | Número decimal | `estimated_power`. Requerida. |
| `UnidadPotencia` | Choice | `kW`, `kVA`. Default `kW`. |
| `CorrienteNominal` | Número decimal | Calculado en la app (o columna calculada). |
| `Frecuencia` | Choice | `50`, `60`, `otro`. Requerida. |
| `OtraFrecuencia` | Número entero | Hz. Si `otro`. |
| `ProteccionesRequeridas` | **Choice (multi-select)** | 9 opciones. Requerida. |
| `MarcasPreferidas` | **Choice (multi-select)** | 10 opciones. |

**`ProteccionesRequeridas`:** `interruptor_automatico`, `diferencial`, `fusible`,
`relevo_sobrecarga`, `relevo_falla_tierra`, `proteccion_tension`,
`proteccion_corriente`, `descargador_tension`, `otro`.
**`MarcasPreferidas`:** `schneider`, `siemens`, `abb`, `legrand`, `eaton`, `chint`,
`hager`, `weidmuller`, `phoenix`, `otro`.

### Diseño constructivo

| Columna | Tipo | Opciones / notas |
|---|---|---|
| `MaterialGabinete` | Choice | `acero_pintado`, `acero_galvanizado`, `acero_inoxidable`, `acero_inox_316`, `fibra_vidrio`, `poliester`, `aluminio`. Requerida. |
| `ColorGabinete` | Choice | `7035`,`7016`,`9016`,`9005`,`5010`,`6005`,`otro` (RAL). Default `7035`. |
| `TipoVentilacion` | Choice | `natural`, `forzada`, `sellado`, `climatizado`. Requerida. |
| `ExpansionFutura` | Choice | `no`, `10`, `20`, `30`, `otro`. Requerida. |
| `ObservacionesTablero` | Texto (varias líneas) | `additional_observations` |

---

## 3. Adjuntos

Dos opciones (decisión M-1 fue "nativos planos" en SharePoint; en Dataverse el
equivalente recomendado es):

- **A — Columnas File por slot:** `EspecificacionesTecnicas`, `ListaCargas`,
  `DiagramaUnilineal`, `PlanosMecanicos`, etc. (tipo **File**). Conserva la
  semántica de cada archivo (lo que se perdía con adjuntos planos de SharePoint),
  sin tabla extra. **Recomendado.**
- **B — Tabla `Archivo`** (Lookup a Solicitud/Tablero + Choice `TipoArchivo` +
  columna File). Más flexible si el número de archivos por tipo es variable
  (p. ej. múltiples fotos de sitio).

Para `site_photos` (múltiples) conviene B o una columna File en una tabla hija de
fotos; el resto encaja en A.

---

## 4. Choices: ¿locales o globales?

- **Globales (recomendado para los reutilizables):** `Estado` (la usan la tabla y
  la auditoría/BPF), unidades, RAL. Reutilizables y editables en un solo lugar.
- **Locales:** las muy específicas de una sola tabla.

> Mantener como **valor** de cada opción el código corto original (`fuerza`,
> `IP54`, etc.) para conservar el dato limpio; la etiqueta en español va en el label.

---

## 5. Auditoría e historial

- Activar **Audit** (entorno + ambas tablas + columna `Estado`). Da el historial de
  cambios de estado con usuario y timestamp — reemplaza `HistorialEstados`.
- **Si se requiere un comentario por transición visible en la app** (el original lo
  tenía), añadir una tabla ligera `HistorialEstado` (Lookup a Solicitud,
  `EstadoAnterior`, `EstadoNuevo`, `CambiadoPor`, `Comentario`, `Fecha`,
  `Organizacion`) que escribe el BPF/flow. La auditoría nativa no captura el
  comentario libre.

---

## Cambios vs. el esquema SharePoint (resumen)

| SharePoint (superseded) | Dataverse |
|---|---|
| 3 listas + Lookup | 2 tablas + relación 1:N Parental |
| `Title` reutilizado como código | Primary Name **Autonumber** |
| Columna oculta `EstadoPrevio` | Eliminada (BPF) |
| Lista `HistorialEstados` | Auditoría nativa (+ tabla ligera solo si hace falta el comentario) |
| Choice múltiple nativo SP | Choice multi-select Dataverse |
| Índice de Lookup + umbral 5.000 | No aplica |
| Adjuntos planos | Columnas File por slot (conserva semántica) |
