# SESSION.md — Estado de la migración M365

> Estado de trabajo entre sesiones. Para continuar desde casa (claude.ai u otra
> máquina), pega este archivo + el README al inicio de la conversación.

---

## Última actualización
2026-06-24

## Qué es esto
Migración del **Módulo de Solicitudes de Tableros Eléctricos** desde el PMIS
Laravel/Filament (repo `axon`) a Microsoft 365. Motivo: la empresa no quiere
costear mantenimiento de app a medida; usa M365 que ya paga. El repo `axon`
original queda congelado como proyecto personal y NO se toca.

## Arquitectura aprobada
- **Captura y gestión:** Power Apps Canvas (conectores estándar → incluido en M365).
- **Datos:** SharePoint Lists (`Solicitudes`, `SolicitudTableros`, `HistorialEstados`).
- **Automatización:** Power Automate (notificaciones + máquina de estados).
- Decisiones cerradas: Choice múltiple nativo (opción A); usuarios internos con
  cuenta MS (sin acceso anónimo); `raw_data` JSON descartado; RichEditor → texto
  enriquecido SharePoint. Detalle en `docs/adr/0001-...md`.

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
  de adjuntos, umbral 5000. Quedan 3 decisiones abiertas (A-3 roles en flujos,
  M-1 tags de adjuntos, B-1 fórmula bifásica).

### Pendiente ⏳ (construcción real en el navegador, la hace el usuario)
- [ ] Crear las 3 listas SharePoint según el esquema.
- [ ] Construir la Power App Canvas de captura.
- [ ] Crear los flujos Power Automate.
- [ ] Construir la app/vista de gestión interna (bandeja) — guía ya escrita (`04`).
- [ ] Subir este repo a GitHub.
- [ ] Resolver las 3 decisiones abiertas de la revisión (A-3, M-1, B-1).

## Próximo paso concreto
Crear las 3 listas en SharePoint siguiendo `docs/sharepoint/01-listas-esquema.md`,
empezando por `Solicitudes` (incluir columna oculta `EstadoPrevio` por defecto
`nueva`), luego `SolicitudTableros` (con la columna Lookup `Solicitud` →
`Solicitudes`, **indexada**), luego `HistorialEstados` (con `Organizacion`).
Verificar que los valores internos de cada columna Choice coincidan con los
códigos del esquema.

## Notas de logística
- Trabajo futuro sin código local: se puede continuar en claude.ai pegando este
  SESSION.md + README. La construcción M365 es 100% en el navegador.
- Repo aún no está en GitHub; crear desde github.com o GitHub Desktop en Windows.
- Ruta local actual del repo: `\\wsl.localhost\ddev\home\ubuntu\axon-365`.
