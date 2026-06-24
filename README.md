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

## Arquitectura elegida

```
Captura:        Power Apps Canvas (conectores estándar → incluido en M365)
Datos:          SharePoint Lists (Solicitudes, SolicitudTableros, HistorialEstados)
Automatización: Power Automate (notificaciones + máquina de estados)
Gestión:        Power Apps Canvas (bandeja) + vistas de lista SharePoint
```

**Clave de licenciamiento:** una Power App Canvas que usa solo conectores
estándar (SharePoint, Outlook, Teams) está **incluida en la licencia M365** — no
requiere Power Apps premium ni Dataverse. Por eso esta combinación da costo cero
y, a la vez, conserva el patrón multi-tablero del formulario original.

Ver el detalle y las alternativas descartadas (Dataverse, Microsoft Forms,
Power Pages, Excel) en [docs/adr/0001-canvas-sharepoint-sobre-dataverse.md](docs/adr/0001-canvas-sharepoint-sobre-dataverse.md).

## Cómo está organizado

| Carpeta | Contenido |
|---|---|
| `docs/adr/` | Decisiones de arquitectura (por qué Canvas+SharePoint) |
| `docs/sharepoint/` | Esquema de las 3 listas: columnas, tipos, opciones de choice |
| `docs/powerapps/` | Guía paso a paso para construir la Power App Canvas |
| `docs/powerautomate/` | Definición de los flujos (notificaciones, estados) |

## Orden de construcción recomendado

1. **SharePoint** — crear las 3 listas según [docs/sharepoint/01-listas-esquema.md](docs/sharepoint/01-listas-esquema.md).
2. **Power Apps** — construir la app de captura siguiendo [docs/powerapps/02-canvas-guia-construccion.md](docs/powerapps/02-canvas-guia-construccion.md).
3. **Power Automate** — crear los flujos de [docs/powerautomate/03-flujos.md](docs/powerautomate/03-flujos.md).
4. **Gestión interna** — vistas de lista + app de bandeja.

## Estado

- [x] Arquitectura aprobada (Canvas + SharePoint, opción A: Choice múltiple nativo)
- [ ] Listas SharePoint creadas
- [ ] Power App Canvas de captura
- [ ] Flujos Power Automate
- [ ] App de gestión interna
