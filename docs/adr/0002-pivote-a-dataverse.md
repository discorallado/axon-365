# ADR 0002 — Pivote a Dataverse (supersede ADR 0001)

- **Estado:** Aceptado
- **Fecha:** 2026-06-25
- **Supersede:** [ADR 0001](0001-canvas-sharepoint-sobre-dataverse.md) (Canvas + SharePoint).

## Contexto

El ADR 0001 eligió **SharePoint Lists** como backend porque se asumía que
Dataverse implicaba licencia premium no disponible. Posteriormente se confirmó que
**la organización sí tiene acceso a Dataverse**. Como aún **no se había construido
nada** en M365 (solo documentación), el costo de cambiar es mínimo.

## Decisión

Migrar todo el módulo a **Dataverse** como backend; **se elimina SharePoint** del
diseño. La app de captura sigue siendo **Power Apps Canvas** (reapuntada a
Dataverse); la gestión interna pasa a **app Model-driven**.

> Requiere confirmar que es **Dataverse completo** (no "Dataverse for Teams"). Sin
> BPF ni security roles ricos, varias ventajas abajo no aplican. Verificar en
> make.powerapps.com → Entorno / Roles de seguridad / Flujos de proceso de negocio.

## Qué cambia respecto a ADR 0001

| Pieza (ADR 0001, SharePoint) | Reemplazo (Dataverse) |
|---|---|
| 3 listas SharePoint | **Tablas Dataverse** `Solicitud`, `SolicitudTablero` (+ relación 1:N nativa) |
| Columnas Choice de lista | **Choices / option sets** (reutilizables globalmente) |
| Lista `HistorialEstados` | **Auditoría nativa** de Dataverse (quién/cuándo/valor anterior→nuevo). Se mantiene una tabla ligera de historial **solo** si se requiere comentario por transición visible en la app |
| Máquina de estados en Power Automate F-2 (+ `EstadoPrevio`, anti-bucle) | **Business Process Flow** nativo — enforce de transiciones, sin columna `EstadoPrevio`, sin riesgo de bucle |
| Roles vía permisos de lista + workaround A-3 en Flow | **Security roles** por tabla/columna/fila — replica fiel la matriz de policies |
| `reference_code` (guid + chequeo de unicidad) | **Columna Autonumber** secuencial y única |
| Adjuntos nativos planos (M-1) | Tabla relacionada `Archivo` con columna de tipo, o columnas File por slot |
| Bandeja Canvas hecha a mano | **App Model-driven** (formularios, vistas y subgrids casi automáticos) |
| Galería editable multi-tablero | **Subgrid 1:N** en el formulario |
| Umbral de 5.000 ítems (M-2) | No aplica — Dataverse escala a millones |

## Hallazgos de la revisión que quedan resueltos por la plataforma

- **C-1 / A-1 / A-2** (máquina de estados, anti-bucle, `EstadoPrevio`) → Business
  Process Flow nativo.
- **A-3 / A-4** (roles en estados, matriz) → security roles nativos.
- **M-2** (umbral 5.000) → desaparece.
- **M-3** (`reference_code`) → Autonumber.
- **M-4** (`organization_id`) → se mantiene como columna en cada tabla
  (multi-tenant-ready), igual que antes.
- **M-5 / M-6** (notificaciones, validación) → siguen vigentes; se implementan con
  Power Automate y business rules / BPF.

## Decisiones previas que se conservan

- **B-1:** sistema `bifasico` eliminado del Choice `SistemaElectrico` y del cálculo.
- **Modelo ancho (no EAV):** `SolicitudTablero` mantiene una fila por tablero con
  columnas tipadas. En Dataverse el argumento es aún más fuerte (relaciones e
  índices reales).

## Consecuencias

**Positivas:** fidelidad casi total al diseño Laravel original; se eliminan los
workarounds de SharePoint; escala sin el techo de 5.000; auditoría y RBAC nativos.

**Negativas / a validar:**
- Depende de **Dataverse completo**; si fuera Dataverse for Teams, reevaluar.
- Confirmar que la app Canvas siga usando **solo conectores estándar + Dataverse**
  para no incurrir en costos inesperados (el acceso a Dataverse ya implica
  licencia adecuada; validar el plan exacto).

## Estado de los documentos

Los docs de SharePoint quedan **marcados como superseded** y se reescriben a
Dataverse de forma incremental. Ver banner al inicio de cada uno.
