# Guía paso a paso — construir en make.powerapps.com

Construcción de la **Solución**, las **tablas** y los **security roles** en el
entorno Plan Developer. Sigue el orden; cada bloque deja algo verificable.

> Referencia de campos completa: [01-tablas.md](01-tablas.md). Aquí va la mecánica
> (qué clicar); el detalle de cada columna está en ese esquema.

---

## Bloque 0 — Entorno y auditoría

1. Entra a **make.powerapps.com** con tu cuenta.
2. Arriba a la derecha, en el **selector de entorno**, elige tu entorno **Developer**
   (suele decir tu nombre + "(Plan Developer)"). Confirma que NO sea el "default".
3. Activa auditoría del entorno: **Configuración** (⚙) → *Centro de administración*
   (Power Platform admin) → tu entorno → **Configuración** → *Auditoría y registros*
   → activar **Auditar**, **iniciar la auditoría de lectura** opcional. (Necesario
   para que el historial de estados se registre nativamente.)

✅ *Verificable:* el entorno Developer está seleccionado y la auditoría está ON.

---

## Bloque 1 — Crear la Solución

Todo va dentro de una Solución para poder exportar a producción después (ver
[03-plan-developer-y-despliegue.md](03-plan-developer-y-despliegue.md)).

1. Menú izquierdo → **Soluciones** → **+ Nueva solución**.
2. **Nombre para mostrar:** `Axon Solicitudes`. **Nombre:** `AxonSolicitudes`.
3. **Publicador:** **+ Nuevo publicador** →
   - Nombre para mostrar: `CSEnergy`
   - **Prefijo:** `csen` (queda como `csen_` en cada tabla/columna).
   - Guardar y seleccionar ese publicador.
4. **Versión:** 1.0.0.0. **Crear**.

✅ *Verificable:* existe la solución `Axon Solicitudes` con publicador prefijo `csen`.

> De aquí en adelante, **trabaja siempre dentro de la solución** (entra a ella y crea
> los componentes con **+ Nuevo**). No crees tablas sueltas fuera de la solución.

---

## Bloque 2 — Choices globales (option sets)

Créalos primero para reutilizarlos en las columnas. Dentro de la solución:
**+ Nuevo → Más → Opciones (Choice)**. Para cada uno, usa como **valor** el código
corto y como **etiqueta** el texto en español.

Mínimos a crear como globales (los más reutilizados):
- `csen_Estado`: nueva, en_revision, cotizada, aprobada, rechazada.
- `csen_IngenieriaPor`: csenergy, cliente, conjunta.

El resto (TipoTablero, GradoIP, etc.) puedes crearlos **locales** al vuelo al definir
cada columna Choice — es más rápido si solo los usa una tabla. Ver la lista completa
de opciones en [01-tablas.md](01-tablas.md).

✅ *Verificable:* el Choice `csen_Estado` existe con las 5 opciones.

---

## Bloque 3 — Tabla `Solicitud`

### 3.1 Crear la tabla
1. Dentro de la solución → **+ Nuevo → Tabla → Tabla (avanzadas)**.
2. **Nombre para mostrar:** `Solicitud`. Plural: `Solicitudes`.
3. Expande **Opciones avanzadas** → marca **Crear auditoría** (Audit) ✅.
4. **Columna principal:** déjala como texto; ponle nombre para mostrar `Asunto`
   (la usaremos solo como título legible). El **código** va aparte como Autonumber (3.2).
5. **Guardar**.

### 3.2 Columna `ReferenceCode` (Autonumber)
1. En la tabla → **Columnas → + Nueva columna**.
2. Nombre: `ReferenceCode`. **Tipo de datos: Numeración automática (Autonumber)**.
3. **Tipo de Autonumber:** con prefijo + número. Formato: `SOL-{SEQNUM:00000}`.
4. Guardar. (Dataverse asignará `SOL-00001`, `SOL-00002`… únicos, sin código ni flujo.)

### 3.3 Resto de columnas de `Solicitud`
Añade una por una (**+ Nueva columna**) según [01-tablas.md](01-tablas.md). Guía por tipo:

| Campo | Al crear la columna elige… |
|---|---|
| `Estado` | Tipo **Choice** → usar el global `csen_Estado` → valor por defecto `nueva` |
| `NombreProyecto` | **Texto** (una línea) → marcar **Requerido** |
| `IngenieriaPor` | **Choice** → global `csen_IngenieriaPor` → Requerido |
| `ContactoNombre` | Texto, Requerido |
| `ContactoEmail` | Texto, **Formato = Correo electrónico**, Requerido |
| `ContactoTelefono` | Texto, Formato = Teléfono |
| `EmpresaCliente`, `UbicacionInstalacion`, `CentroCosto` | Texto |
| `FechaEntregaDeseada` | **Fecha y hora → Formato = Solo fecha** |
| `EnviadoEl` | Fecha y hora |
| `ObservacionesProyecto` | **Texto → varias líneas** |
| `Asignado` | **Búsqueda (Lookup) → tabla Usuario** |
| `Organizacion` | Texto (single-tenant por ahora) |

✅ *Verificable:* `Solicitud` tiene `ReferenceCode` autonumber, `Estado` con default
`nueva`, y los campos de contacto/proyecto. Auditoría ON en la tabla.

---

## Bloque 4 — Tabla `SolicitudTablero` + relación 1:N

### 4.1 Crear la tabla
1. Solución → **+ Nuevo → Tabla (avanzadas)**. Nombre: `SolicitudTablero`,
   plural `SolicitudTableros`. **Crear auditoría** ✅.
2. Columna principal: `Nombre` (texto) → es el `label` del tablero (ej. "TG Principal").

### 4.2 La relación con `Solicitud` (1:N Parental)
1. En la tabla `Solicitud` → **Relaciones → + Nueva relación → Uno a varios**.
2. **Tabla relacionada:** `SolicitudTablero`. Esto crea automáticamente la columna
   **Lookup `Solicitud`** en el hijo.
3. En la configuración de la relación, **Comportamiento → Tipo = Parental** (o
   personalizado con **Eliminar = Cascada**) para que al borrar la solicitud se
   borren sus tableros (replica el cascade original).
4. Guardar.

### 4.3 Resto de columnas (~35) de `SolicitudTablero`
Mismo procedimiento, según [01-tablas.md](01-tablas.md). Atajos por tipo:

| Tipo de campo | Cómo |
|---|---|
| `Cantidad`, `NumeroCircuitos`, `AltoMaxMm`… | **Número entero** (Whole Number), mín/máx en opciones |
| `PotenciaEstimada`, `CorrienteNominal` | **Número decimal** |
| `RestriccionesDimension` | **Sí/No** (Yes/No) |
| `TipoTablero`, `GradoIP`, `MaterialGabinete`… | **Choice** (locales; opciones en el esquema) |
| `AmbienteEspecial`, `ProteccionesRequeridas`, `MarcasPreferidas` | **Choice → marcar "Seleccionar varios" (multi-select)** |
| `FuncionTablero`, `CargasAAlimentar` | Texto varias líneas → **Formato = Texto enriquecido** |
| `ObservacionesTablero`, `CondicionesInstalacion` | Texto varias líneas |

> **Recuerda:** `SistemaElectrico` SIN `bifasico` (decisión B-1). No crees columna
> `EstadoPrevio` (la máquina de estados usa el pre-image nativo).

### 4.4 Adjuntos (columnas File)
Añade columnas tipo **Archivo (File)**: `EspecificacionesTecnicas`, `ListaCargas`,
`DiagramaUnilineal`, `PlanosMecanicos`. Para `site_photos` (múltiples) ver §3 de
[01-tablas.md](01-tablas.md) (tabla hija de fotos si quieres varias).

✅ *Verificable:* `SolicitudTablero` tiene el Lookup `Solicitud`, las columnas
tipadas, los 3 multi-select y las columnas File.

---

## Bloque 5 — Tabla `HistorialEstado` (solo si quieres comentario por transición)

La auditoría nativa ya registra los cambios de `Estado`, pero NO el comentario libre.
Si lo quieres visible en la app:

1. Tabla `HistorialEstado` (auditoría opcional). Columna principal `Titulo` (texto).
2. Columnas: Lookup `Solicitud`; Choice `EstadoAnterior` y `EstadoNuevo` (global
   `csen_Estado`); Lookup `CambiadoPor` → Usuario; Texto varias líneas `Comentario`;
   Fecha y hora `Fecha`; Texto `Organizacion`.
3. Relación 1:N desde `Solicitud` (no parental necesariamente; referencial está bien).

✅ *Verificable:* existe `HistorialEstado` con su Lookup a `Solicitud`.

---

## Bloque 6 — Security roles (5)

1. Centro de administración → tu entorno → **Configuración → Usuarios y permisos →
   Roles de seguridad → + Nuevo rol** (o desde la solución en versiones nuevas).
2. Crea `csen_super_admin`, `csen_supervisor`, `csen_ingeniero`, `csen_calidad`,
   `csen_tecnico`.
3. Asigna privilegios sobre `Solicitud` / `SolicitudTablero` / `HistorialEstado`
   según la matriz de [02-bpf-y-roles.md](02-bpf-y-roles.md) §2 (nivel **Organización**).
4. En Developer eres único usuario: asígnate `csen_super_admin` para probar.

✅ *Verificable:* los 5 roles existen y tú tienes super_admin.

---

## Después de esto

Con tablas + roles listos:
1. Construir la **app Canvas de captura** (YAML en [../powerapps/05-canvas-yaml-captura.md](../powerapps/05-canvas-yaml-captura.md)).
2. Crear el **flujo real-time de estados** ([02-bpf-y-roles.md](02-bpf-y-roles.md) §3).
3. Crear los **flujos de Power Automate F-1** (notificaciones).
4. Construir la **app Model-driven** de gestión.

> Mantén todo dentro de la solución `Axon Solicitudes`. Cuando todo funcione,
> exporta como **managed** e importa en el entorno productivo licenciado.
