# Guía paso a paso — Business Rules, vistas y app Model-driven

Continúa después de [00-construir-en-powerapps.md](00-construir-en-powerapps.md)
(Bloque 6 — security roles ya creados). Aquí: validaciones de campo
(Business Rules), vistas de lista, formularios, subgrid y la app Model-driven
de gestión que reemplaza a `SubmissionRequestResource`/`ViewSubmissionRequest`
de Filament (ver también [docs/powerapps/04-app-gestion-interna.md](../powerapps/04-app-gestion-interna.md),
la versión Canvas que esta guía reemplaza).

> Todo se crea **dentro de la solución** `Axon Solicitudes`.
> La validación de **transición de estado** (quién puede pasar de qué a qué) NO
> va aquí: vive en el flujo en tiempo real de [02-bpf-y-roles.md](02-bpf-y-roles.md) §3.
> Las Business Rules de este documento son **validaciones de formulario**
> (campos condicionales/requeridos), el equivalente a las reglas de validación
> de Laravel Form Request del original.

---

## Bloque 7 — Business Rules de `SolicitudTablero`

`Solicitud` no tiene campos condicionales (todo lo requerido se marcó a nivel
columna en el Bloque 3). `SolicitudTablero` sí: 7 reglas, una por campo "Otro…"
o dependencia. Se crean en **Tablas → SolicitudTablero → Reglas de negocio →
+ Nueva regla de negocio**. Cada una: alcance **Todos los formularios**, para
que aplique también en la app Model-driven además de en Canvas.

### 7.1 BR — Tipo de tablero condicional
- **Si** `TipoEntrega` = `tablero` → `TipoTablero`: **Mostrar campo** + **Establecer como
  obligatorio de la empresa**.
- **Si no** → `TipoTablero`: **Ocultar campo** + **Establecer como no obligatorio**.

### 7.2 BR — Otro tipo de tablero
- **Si** `TipoTablero` = `otro` → `OtroTipoTablero`: mostrar + obligatorio.
- **Si no** → ocultar + no obligatorio.

### 7.3 BR — Otro ambiente especial
- **Si** `AmbienteEspecial` **contiene valores** `otro` (operador de condición para
  Choice multi-select) → `OtroAmbienteEspecial`: mostrar + obligatorio.
- **Si no** → ocultar + no obligatorio.
- ⚠️ Si tu versión del diseñador de Business Rules no ofrece "contiene valores"
  para columnas multi-select, alternativa: un pequeño **JavaScript de formulario**
  (`OnChange` de `AmbienteEspecial` → `formContext.getAttribute("csen_ambienteespecial").getValue()`
  devuelve array de códigos → `.includes("otro")`) o validarlo en un flujo de
  Power Automate al guardar.

### 7.4 BR — Restricciones de dimensión
- **Si** `RestriccionesDimension` = **Sí** → `AltoMaxMm`, `AnchoMaxMm`, `FondoMaxMm`:
  mostrar + obligatorio (las 3 en la misma regla, una acción por campo).
- **Si no** → ocultar + no obligatorio + (opcional) **Establecer valor** = vacío.

### 7.5 BR — Otra tensión
- **Si** `TensionSuministro` = `otro` → `OtraTension`: mostrar + obligatorio.
- **Si no** → ocultar + no obligatorio.

### 7.6 BR — Otro sistema eléctrico
- **Si** `SistemaElectrico` = `otro` → `OtroSistemaElectrico`: mostrar + obligatorio.
- **Si no** → ocultar + no obligatorio.
- Recuerda: `SistemaElectrico` no tiene opción `bifasico` (decisión B-1).

### 7.7 BR — Otra frecuencia
- **Si** `Frecuencia` = `otro` → `OtraFrecuencia`: mostrar + obligatorio.
- **Si no** → ocultar + no obligatorio.

### 7.8 BR — Grado IP mínimo si es exterior (validación de ingeniería)
El original dejaba "el subconjunto válido se filtra en la app". Una Business
Rule no puede filtrar las opciones de un Choice, pero sí puede **bloquear el
guardado con un mensaje** si la combinación no es válida:

- **Si** `Ubicacion` = `exterior` **Y** `GradoIP` es uno de
  `IP20, IP21, IP30, IP31, IP40, IP41, IP42, IP44` (protección insuficiente
  contra agua) → **Mostrar mensaje de error** en `GradoIP`:
  *"Para instalación exterior el grado IP mínimo es IP54."*
- Condición inversa no hace falta: los grados `IP54`–`IP68` son válidos tanto
  interior como exterior.
- Confirma con ingeniería si el umbral exacto (`IP54`) es el correcto para tu
  catálogo de tableros antes de dejar la regla en producción.

✅ *Verificable:* las 7 reglas activadas (**Activar** en la barra superior del
editor); en el formulario, cambiar `TipoEntrega` a `sala` oculta `TipoTablero`,
y elegir `Ubicacion = exterior` + `GradoIP = IP20` muestra el error al guardar.

---

## Bloque 8 — Columna calculada `CorrienteNominal` (opcional)

Si quieres que Dataverse calcule la corriente en vez de hacerlo la app Canvas:

1. `SolicitudTablero` → **+ Nueva columna** → `CorrienteNominalCalc`, tipo
   **Columna de fórmula** (Power Fx).
2. **Notación:** separador decimal `,` en el entorno → argumentos de función
   con `;` (esta fórmula es una sola expresión, sin encadenado `;;`).
3. Fórmula (aprox. `I = P / (√3 × V × cos φ)` para trifásico, `I = P / V` para
   monofásico/DC; ajusta el factor de potencia y unidad de potencia real):
   ```powerfx
   If(
       SolicitudTablero.UnidadPotencia.Value = "kVA";
       Switch(SolicitudTablero.SistemaElectrico.Value;
           "trifasico";  (SolicitudTablero.PotenciaEstimada * 1000) / (Sqrt(3) * SolicitudTablero.TensionSuministroValor);
           (SolicitudTablero.PotenciaEstimada * 1000) / SolicitudTablero.TensionSuministroValor
       );
       Switch(SolicitudTablero.SistemaElectrico.Value;
           "trifasico";  (SolicitudTablero.PotenciaEstimada * 1000) / (Sqrt(3) * SolicitudTablero.TensionSuministroValor * 0.9);
           (SolicitudTablero.PotenciaEstimada * 1000) / (SolicitudTablero.TensionSuministroValor * 0.9)
       )
   )
   ```
   `TensionSuministroValor` no existe todavía: como `TensionSuministro` es un
   Choice con valores numéricos como etiqueta (`220`, `380`…), añade una columna
   número auxiliar (`TensionSuministroValor`, entero) que la Business Rule 7.5
   ya llena vía `OtraTension` cuando es `otro`, o iguala por defecto según el
   Choice. Si prefieres simplicidad, deja `CorrienteNominal` como columna
   **manual** (Número decimal) y dile al usuario que la calcule la app Canvas
   como hasta ahora — es la opción más simple y la que ya documenta
   [01-tablas.md](01-tablas.md). Este bloque es opcional.

✅ *Verificable:* al crear/editar un tablero con potencia y tensión, `CorrienteNominal`
se recalcula solo (si implementaste la fórmula) o queda editable (si la dejaste manual).

---

## Bloque 9 — Vistas de `Solicitud`

En la tabla `Solicitud` → pestaña **Vistas** → **+ Nueva vista** (o edita las
del sistema). Para cada una: **Editar columnas** (elige y ordena), **Editar
filtros** (criterios), **Ordenar por**.

| Vista | Filtro | Orden | Columnas |
|---|---|---|---|
| **Todas las solicitudes** (editar la vista de sistema *Active Solicitudes*) | Estado activo (no eliminadas) | `EnviadoEl` desc | `ReferenceCode`, `NombreProyecto`, `EmpresaCliente`, `Estado`, `Asignado`, `EnviadoEl`, `FechaEntregaDeseada` |
| **Nuevas** | `Estado` = `nueva` | `EnviadoEl` asc (FIFO) | igual que arriba |
| **En proceso** | `Estado` **en** (`en_revision`, `cotizada`) | `EnviadoEl` desc | + columna `IngenieriaPor` |
| **Cerradas** | `Estado` **en** (`aprobada`, `rechazada`) | `EnviadoEl` desc | igual que "Todas" |
| **Mis asignadas** | `Asignado` = **Usuario actual** (`Me()` / "Usuario que ha iniciado sesión") **Y** `Estado` **en** (`nueva`,`en_revision`,`cotizada`) | `FechaEntregaDeseada` asc | + `FechaEntregaDeseada` destacada |
| **Sin asignar** | `Asignado` = **vacío** **Y** `Estado` ≠ (`aprobada`,`rechazada`) | `EnviadoEl` asc | igual que "Todas" |

- **Colores/chip de estado:** en vistas de tabla nativa no hay formato
  condicional por celda como en Canvas; Dataverse sí permite **formato
  condicional de columnas** (icono/color) en vistas modernas → en **Editar
  columnas de la vista → columna `Estado` → Formato condicional** → mapear
  cada valor a un color (mismos que en la guía Canvas: azul=nueva,
  ámbar=en_revision, índigo=cotizada, verde=aprobada, rojo=rechazada).
- Marca **Mis asignadas** o **Todas** como vista por defecto de la app (Bloque 12).

✅ *Verificable:* las 6 vistas existen y filtran correctamente al probarlas
desde la tabla (botón **Vista previa**).

---

## Bloque 10 — Vista asociada de `SolicitudTablero`

Esta es la que se usa dentro del **subgrid** del formulario de `Solicitud`
(Bloque 11), no una pantalla aparte.

1. `SolicitudTablero` → **Vistas** → edita (o crea) **Vista asociada de
   SolicitudTablero** (la que Dataverse ofrece por defecto para el subgrid).
2. **Columnas:** `Nombre`, `TipoEntrega`, `TipoTablero`, `Cantidad`,
   `TensionSuministro`, `PotenciaEstimada`, `UnidadPotencia`, `GradoIP`.
3. **Ordenar por:** `Orden` asc (si la usas), si no `Nombre` asc.
4. (Opcional) Vista adicional **"Todos los tableros"** (general, no asociada)
   para reportes/exportar a Excel con todas las ~35 columnas.

✅ *Verificable:* al abrir una `Solicitud` con tableros cargados, el subgrid
muestra esas columnas en ese orden.

---

## Bloque 11 — Formularios

### 11.1 Formulario principal de `Solicitud`
`Solicitud` → **Formularios** → formulario principal (o **+ Nuevo → Formulario
principal**) → organiza en secciones:

1. **Identificación** — `ReferenceCode` (solo lectura), `Estado` (destacado,
   grande), `Asignado`.
2. **Datos del proyecto** — `NombreProyecto`, `UbicacionInstalacion`,
   `CentroCosto`, `FechaEntregaDeseada`, `IngenieriaPor`.
3. **Contacto** — `ContactoNombre`, `ContactoEmail`, `ContactoTelefono`,
   `EmpresaCliente`.
4. **Observaciones** — `ObservacionesProyecto` (ancho completo).
5. **Tableros** — agrega un **Subgrid** (componente → Subgrid) enlazado a la
   relación `SolicitudTablero`, usando la **vista asociada** del Bloque 10.
   Marca **"Mostrar barra de comandos"** para que desde ahí se puedan crear
   tableros sin salir de la solicitud.
6. Si adoptaste el **Business Process Flow** (opcional, [02-bpf-y-roles.md](02-bpf-y-roles.md) §4):
   arrástralo para que quede **arriba del todo** del formulario.
7. **Metadatos** (pestaña aparte, colapsada): `EnviadoEl`, `Organizacion`,
   info de creación/auditoría estándar.

Guarda y **publica**.

### 11.2 Formulario principal de `SolicitudTablero`
Organízalo igual que el esquema de [01-tablas.md](01-tablas.md), por bloques —
así las Business Rules del Bloque 7 muestran/ocultan campos dentro de la
misma sección sin saltos raros:

1. **Identificación** — `Nombre`, `Cantidad`, `Orden`, `TipoEntrega`,
   `InstalacionNuevaReemplazo`, `TipoTablero`, `OtroTipoTablero`,
   `FuncionTablero`, `CargasAAlimentar`, `NumeroCircuitos`.
2. **Ubicación y ambiente** — `Ubicacion`, `AmbienteEspecial`,
   `OtroAmbienteEspecial`, `GradoIP`, `GradoIK`.
3. **Montaje y dimensiones** — `TipoMontaje`, `RestriccionesDimension`,
   `AltoMaxMm`, `AnchoMaxMm`, `FondoMaxMm`, `CondicionesInstalacion`.
4. **Parámetros eléctricos** — `TensionSuministro`, `OtraTension`,
   `SistemaElectrico`, `OtroSistemaElectrico`, `PotenciaEstimada`,
   `UnidadPotencia`, `CorrienteNominal`, `Frecuencia`, `OtraFrecuencia`,
   `ProteccionesRequeridas`, `MarcasPreferidas`.
5. **Diseño constructivo** — `MaterialGabinete`, `ColorGabinete`,
   `TipoVentilacion`, `ExpansionFutura`, `ObservacionesTablero`.
6. **Adjuntos** — columnas File (`EspecificacionesTecnicas`, `ListaCargas`,
   `DiagramaUnilineal`, `PlanosMecanicos`).

✅ *Verificable:* al abrir un tablero de prueba y cambiar `TipoEntrega` a
`sala`, `TipoTablero` desaparece del formulario (Business Rule 7.1 en acción).

---

## Bloque 12 — La app Model-driven

1. Solución `Axon Solicitudes` → **+ Nuevo → App → App basada en modelo**.
2. **Nombre:** `Axon Solicitudes — Gestión`. Descripción breve.
3. **Selector de páginas → + Página → Tabla**: agrega `Solicitud` y
   `SolicitudTablero` (esta última puede ir oculta del mapa de sitio si solo
   se accede vía subgrid).
4. **Mapa del sitio** (editor visual):
   - Área **"Solicitudes"**:
     - Grupo **Bandeja** → subáreas: `Mis asignadas` (vista por defecto),
       `Nuevas`, `En proceso`, `Cerradas`, `Todas`. Cada subárea = la tabla
       `Solicitud` apuntando a la vista correspondiente del Bloque 9.
   - (Opcional) Área **"Administración"** → `SolicitudTablero` (tabla general),
     Roles de seguridad (acceso directo para super_admin).
5. **Guardar → Publicar**.
6. **Compartir la app:** *Compartir* → asigna a los 5 security roles del
   Bloque 6 (o a un Team con esos roles) para que el resto del equipo la vea
   en su Power Apps home sin ser `super_admin`.

✅ *Verificable:* la app aparece en `make.powerapps.com` → **Apps**; al
abrirla, el mapa de sitio muestra las vistas y el subgrid de tableros funciona
dentro de una solicitud.

---

## Bloque 13 — Dashboard (opcional, reemplaza los contadores de Filament)

1. Solución → **+ Nuevo → Más → Dashboard** (o **Dashboard interactivo**).
2. Un gráfico de **Solicitudes por Estado** (tipo dona/columnas) sobre la vista
   "Todas las solicitudes".
3. Una lista **"Mis asignadas pendientes"** embebida (misma vista del Bloque 9).
4. Agrega el dashboard como página del mapa de sitio (Bloque 12 §4) si quieres
   que sea la pantalla de inicio.

✅ *Verificable:* el dashboard se ve al entrar a la app y el gráfico refleja
los datos reales de prueba.

---

## Equivalencias con la guía Canvas anterior

| Canvas (`powerapps/04`, superseded) | Model-driven (esta guía) |
|---|---|
| Galería `galSolicitudes` + filtros a mano | Vistas del Bloque 9 (nativas, con formato condicional de color) |
| Galería `galTablerosDet` filtrada por Lookup | Subgrid con vista asociada (Bloque 10-11.1) |
| `colAcciones` + gating por rol en Power Fx | Security roles (02 §2) + flujo en tiempo real (02 §3); la app solo edita `Estado`, el servidor valida |
| Validaciones de campos "Otro…" a mano en pantallas Canvas | Business Rules del Bloque 7 |
| Lista `RolesUsuarios` para resolver rol | `systemuserroles` nativo (02 §3) |
| Exportar vía Power Automate | Exportar a Excel nativo desde cualquier vista (botón **Exportar a Excel**) |

## Después de esto
Con Business Rules + vistas + formularios + app listos: crear el **flujo en
tiempo real de estados** ([02-bpf-y-roles.md](02-bpf-y-roles.md) §3) y el
**flujo Power Automate F-1** ([powerautomate/03-flujos.md](../powerautomate/03-flujos.md)),
y probar el recorrido completo con datos reales antes de exportar como
**managed** a producción ([03-plan-developer-y-despliegue.md](03-plan-developer-y-despliegue.md)).
