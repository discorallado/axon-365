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
- `docs/powerapps/02-canvas-guia-construccion.md` — guía de la app Canvas con
  fórmulas Power Fx (cálculo de corriente, lógica condicional, galería multi-tablero).
- `docs/powerautomate/03-flujos.md` — flujos F-1 (alta) y F-2 (estados).

### Pendiente ⏳ (construcción real en el navegador, la hace el usuario)
- [ ] Crear las 3 listas SharePoint según el esquema.
- [ ] Construir la Power App Canvas de captura.
- [ ] Crear los flujos Power Automate.
- [ ] Construir la app/vista de gestión interna (bandeja).
- [ ] Subir este repo a GitHub.

## Próximo paso concreto
Crear las 3 listas en SharePoint siguiendo `docs/sharepoint/01-listas-esquema.md`,
empezando por `Solicitudes`, luego `SolicitudTableros` (con la columna Lookup
`Solicitud` → `Solicitudes`), luego `HistorialEstados`. Verificar que los valores
internos de cada columna Choice coincidan con los códigos del esquema.

## Notas de logística
- Trabajo futuro sin código local: se puede continuar en claude.ai pegando este
  SESSION.md + README. La construcción M365 es 100% en el navegador.
- Repo aún no está en GitHub; crear desde github.com o GitHub Desktop en Windows.
- Ruta local actual del repo: `\\wsl.localhost\ddev\home\ubuntu\axon-365`.
