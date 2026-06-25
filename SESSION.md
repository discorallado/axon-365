# SESSION.md — Estado de la migración M365

> Estado de trabajo entre sesiones. Para continuar desde casa (claude.ai u otra
> máquina), pega este archivo + el README al inicio de la conversación.

---

## Última actualización
2026-06-25

## Qué es esto
Migración del **Módulo de Solicitudes de Tableros Eléctricos** desde el PMIS
Laravel/Filament (repo `axon`) a Microsoft 365. Motivo: la empresa no quiere
costear mantenimiento de app a medida; usa M365 que ya paga. El repo `axon`
original queda congelado como proyecto personal y NO se toca.

## Arquitectura aprobada — PIVOTE A DATAVERSE (2026-06-25, ADR 0002 supersede 0001)
- **Datos:** **Dataverse** — tablas `Solicitud`, `SolicitudTablero` (relación 1:N).
  Ya NO SharePoint.
- **Captura:** Power Apps Canvas reapuntada a Dataverse (YAML en `docs/powerapps/05`).
- **Estados:** Business Process Flow + security roles (nativos) — reemplazan F-2.
- **Automatización:** Power Automate F-1 (notificaciones + acuse).
- **Gestión:** app Model-driven.
- **Requiere Dataverse completo** (NO Dataverse for Teams) — verificar en
  make.powerapps.com (Entorno / Roles de seguridad / BPF). Si fuera Teams, reevaluar.
- Decisiones que se conservan: usuarios internos con cuenta MS; `bifasico` eliminado
  (B-1); modelo ancho (no EAV). Dataverse resuelve A-3/A-4 (roles), M-2 (umbral 5000),
  M-3 (Autonumber), HistorialEstados (auditoría nativa), EstadoPrevio (lo hace el BPF).

## Estado actual

### Completado ✅ (documentación de diseño/ingeniería)
- `README.md` — propósito y orden de construcción.
- `docs/adr/0001-canvas-sharepoint-sobre-dataverse.md` — decisión + alternativas.
- `docs/sharepoint/01-listas-esquema.md` — esquema completo de las 3 listas
  (todas las columnas, tipos y opciones de choice, mapeadas 1:1 al modelo Laravel).
- `docs/powerapps/02-canvas-guia-construccion.md` — guía de la app Canvas de
  captura con Power Fx (cálculo de corriente, lógica condicional, galería multi-tablero).
- `docs/powerapps/04-app-gestion-interna.md` — guía de la app de gestión (bandeja):
  maestro-detalle, acciones de estado gateadas por rol, exportación, vistas de lista.
- `docs/powerautomate/03-flujos.md` — flujos F-1 (alta) y F-2 (estados), revisados.
- Pasó por `/revisor` (commit 50def5a): se corrigió la máquina de estados (era
  incompleta), anti-bucle de F-2, init de EstadoPrevio, matriz de permisos, tags
  de adjuntos, umbral 5000.
- Decisiones de la revisión **resueltas** (2026-06-24): A-3 → reglas de rol en F-2
  (gatea rechazar/reabrir, rol vía grupo AAD o lista RolesUsuarios); M-1 → adjuntos
  nativos planos + convención de nombres; B-1 → eliminado `bifasico` en M365.

### Pivote a Dataverse — hecho esta sesión (2026-06-25)
- `docs/adr/0002-pivote-a-dataverse.md` creado (supersede 0001).
- `docs/powerapps/05-canvas-yaml-captura.md` — YAML `.pa.yaml` de la pantalla de
  captura, enlazado a Dataverse (Patch a tablas, Choice `'Col (Tabla)'.Opcion`,
  lookup por registro).
- Banners de "superseded/actualizar a Dataverse" en los 4 docs de SharePoint/Canvas.
- README y SESSION actualizados al nuevo rumbo.

### Pendiente ⏳
- [ ] **Confirmar Dataverse completo vs. for Teams** (bloquea BPF/security roles).
- [x] Esquema reescrito: `docs/dataverse/01-tablas.md` (tablas, relación 1:N Parental,
      Choices/multi-select, Autonumber, auditoría, adjuntos como columnas File). ✅
- [ ] Reescribir F-2 como Business Process Flow + matriz de security roles.
- [ ] Reescribir la gestión interna como app Model-driven.
- [ ] Completar el YAML de los ~37 campos del tablero (`05`).
- [ ] Construir en M365: tablas → Canvas → BPF → F-1 → model-driven.
- [x] Repo en GitHub: `discorallado/axon-365` (privado). ✅

## Próximo paso concreto
Confirmar en make.powerapps.com que el entorno tiene **Dataverse completo** (ver
Roles de seguridad y Flujos de proceso de negocio). Con eso, escribir
`docs/dataverse/02-bpf-y-roles.md`: el Business Process Flow de la máquina de estados
(transiciones reales: nueva→en_revision/rechazada; en_revision→cotizada/rechazada;
cotizada→aprobada/rechazada/en_revision; reapertura terminal→nueva solo super_admin)
y la matriz de security roles (super_admin, supervisor, ingeniero, calidad, tecnico)
por tabla/acción. El esquema de tablas ya está en `docs/dataverse/01-tablas.md`.

## Notas de logística
- Continuable en claude.ai pegando este SESSION.md + README. Construcción 100% en navegador.
- Repo en GitHub: https://github.com/discorallado/axon-365 (privado). Remote `origin` configurado.
- Ruta local: `\\wsl.localhost\ddev\home\ubuntu\axon-365` (rama `main`).
