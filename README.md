# Axon 365 — Migración del Módulo de Solicitudes a Microsoft 365

Migración del **Módulo de Solicitudes de Tableros Eléctricos** desde el sistema
original Laravel/Filament (repo `axon`) hacia la plataforma Microsoft 365 que la
empresa ya paga, **sin costo de licencias adicionales ni mantenimiento de código
propio**.

## Por qué este repo existe

El sistema original (`axon`) es un PMIS construido en Laravel 12 + Filament. La
empresa decidió no costear el mantenimiento de una aplicación a medida y prefiere
operar sobre las herramientas de Microsoft 365 que ya tiene licenciadas. Este
repo documenta la migración de la primera pieza —el módulo de solicitudes— a
Power Platform.

> El repo `axon` original **no se toca**: queda congelado como proyecto personal.
> Esta migración vive aparte, aquí.

## Arquitectura (actual — Dataverse)

```
Captura:        Power Apps Canvas (reapuntada a Dataverse)
Datos:          Dataverse — tablas Solicitud, SolicitudTablero (relación 1:N)
Estados:        Business Process Flow + security roles (nativos)
Automatización: Power Automate (notificaciones + acuse)
Gestión:        App Model-driven (vistas, formularios, subgrids)
```

> **Pivote a Dataverse** (ver [docs/adr/0002-pivote-a-dataverse.md](docs/adr/0002-pivote-a-dataverse.md),
> supersede al 0001). El diseño previo sobre **SharePoint Lists** se eligió cuando
> se creía que Dataverse no estaba disponible; al confirmarse el acceso a Dataverse
> y sin nada construido aún, se pivotó. Dataverse elimina los workarounds de
> SharePoint (máquina de estados a mano, `EstadoPrevio`, umbral de 5.000) con
> auditoría y RBAC nativos. **Requiere Dataverse completo** (no Dataverse for Teams).

Historia de la decisión: [ADR 0001](docs/adr/0001-canvas-sharepoint-sobre-dataverse.md)
(SharePoint, superseded) → [ADR 0002](docs/adr/0002-pivote-a-dataverse.md) (Dataverse).

## Cómo está organizado

| Carpeta | Contenido |
|---|---|
| `docs/adr/` | Decisiones: `0001` SharePoint (superseded), `0002` pivote a Dataverse |
| `docs/dataverse/` | **Vigente** — `00` guía mecánica de construcción (Solución/tablas/roles), `01` tablas, `02` BPF + security roles, `03` Plan Developer y despliegue, `04` Business Rules/vistas/formularios/app Model-driven, `05` construir el wizard Canvas (4 pantallas + barra de pestañas compartida) en el navegador |
| `docs/sharepoint/` | ⚠️ Superseded — esquema de columnas/choices, útil solo como referencia del modelo |
| `docs/powerapps/` | `02` **Pantalla 2 (tableros), maestro-detalle en una sola pantalla — diseño final**, `04` gestión, `05` YAML/fórmulas de captura (Dataverse) |
| `docs/powerautomate/` | Flujos: F-1 vigente; F-2 → reemplazado por Business Process Flow |

## Orden de construcción recomendado (Dataverse)

1. **Crear la Solución** `Axon Solicitudes` (publisher + prefijo). Construir TODO
   dentro de ella para poder migrar a producción sin retrabajo — ver
   [docs/dataverse/03-plan-developer-y-despliegue.md](docs/dataverse/03-plan-developer-y-despliegue.md).
2. **Tablas Dataverse** — `Solicitud`, `SolicitudTablero` (relación 1:N), Choices,
   `reference_code` como Autonumber, auditoría activada. Esquema en
   [docs/dataverse/01-tablas.md](docs/dataverse/01-tablas.md).
3. **App Canvas de captura** — 4 pantallas navegadas por una barra de pestañas
   compartida (Contacto y Proyecto / Tableros / Documentación, + confirmación
   fuera del flujo). Construcción clic por clic en
   [docs/dataverse/05-construir-canvas-captura.md](docs/dataverse/05-construir-canvas-captura.md)
   (app, pestañas, Pantalla 1, Documentación, confirmación); la **Pantalla 2
   (Tableros)** es maestro-detalle en una sola pantalla, documentada completa
   en [docs/powerapps/02-canvas-guia-construccion.md](docs/powerapps/02-canvas-guia-construccion.md).
   Fórmulas exactas de cada campo en
   [docs/powerapps/05-canvas-yaml-captura.md](docs/powerapps/05-canvas-yaml-captura.md)
   (YAML enlazado a Dataverse).
4. **Estados + roles** — flujo real-time de transiciones + security roles (+ BPF
   opcional), según [docs/dataverse/02-bpf-y-roles.md](docs/dataverse/02-bpf-y-roles.md).
5. **Power Automate** — F-1 (notificaciones + acuse) de [docs/powerautomate/03-flujos.md](docs/powerautomate/03-flujos.md).
6. **App Model-driven** de gestión interna — Business Rules de validación,
   vistas, formularios y subgrid en
   [docs/dataverse/04-app-model-driven.md](docs/dataverse/04-app-model-driven.md).

## Estado

- [x] Arquitectura: pivote a **Dataverse** (ADR 0002, supersede 0001)
- [x] YAML de captura completo: ~37 columnas del tablero, con envío a Dataverse (`05`)
- [x] Esquema de tablas Dataverse documentado (`01-tablas.md`)
- [x] BPF + security roles documentado (`02-bpf-y-roles.md`)
- [x] Plan Developer y despliegue documentado (`03`)
- [x] Confirmado: **Dataverse completo** (Plan Developer)
- [x] Guía mecánica de construcción (Solución/tablas/roles) documentada (`00`)
- [x] Business Rules, vistas, formularios y app Model-driven documentados (`04`)
- [x] Guía clic por clic del wizard de captura Canvas documentada — arquitectura
      final de 4 pantallas + barra de pestañas compartida (`dataverse/05`)
- [x] Pantalla 2 (tableros) maestro-detalle en una sola pantalla — diseño final,
      Editar/Eliminar arriba de la galería, documentado completo con
      `Default`/`Reset` de los 35 campos, validación y dos bugs corregidos (`powerapps/02`)
- [ ] Solución `Axon Solicitudes` creada (construcción real pendiente)
- [ ] Tablas Dataverse creadas
- [ ] App Canvas de captura
- [ ] Flujo real-time de estados + security roles
- [ ] Power Automate F-1
- [ ] App Model-driven de gestión construida en el navegador
