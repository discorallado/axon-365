# Axon 365 — Instrucciones de proyecto (para Cowork / Claude)

> Instrucciones persistentes para asistentes de IA que trabajen en este repo.
> Pégalas como instrucciones de proyecto en Cowork, o léelas al inicio de cada
> sesión. Equivalen al `CLAUDE.md` del repo original, adaptadas a la fase actual
> (construir en Power Platform, no codificar).

## Qué es este proyecto
Migración del **Módulo de Solicitudes de Tableros Eléctricos** desde un PMIS
Laravel/Filament (repo `axon`, congelado, NO se toca) hacia **Microsoft 365 /
Power Platform**. Motivo: la empresa no quiere costear mantenimiento de código a
medida; usa lo que ya licencia. Repo de la migración: `discorallado/axon-365`
(privado, en GitHub y en la carpeta local de Windows). Dominio: construcción
eléctrica e infraestructura crítica (tableros, salas eléctricas). Idioma es-CL,
zona horaria America/Santiago.

## Fase actual: construcción no-code en el navegador
Ya NO se escribe código. Se construye en **make.powerapps.com** (Dataverse, Power
Apps, Power Automate). El "código" del repo son documentos de diseño y guías.

## Arquitectura DECIDIDA — no relitigar
- **Backend: Dataverse completo** (Plan Developer). `ADR 0002` supersede al `0001`
  (que era SharePoint, descartado).
- **Captura:** Power Apps **Canvas**. **Gestión interna:** app **Model-driven**.
- **Máquina de estados:** flujo **real-time** de Dataverse (usa el pre-image
  nativo; SIN columna `EstadoPrevio`) + **security roles**. BPF solo UX opcional,
  no enforcement.
- **Notificaciones:** Power Automate (flujo F-1).
- **`reference_code`:** columna **Autonumber** (`SOL-{SEQNUM:00000}`).
- **Modelo de datos ancho** (1 fila por tablero), NO EAV.
- **Sin sistema bifásico** (decisión B-1).
- **Usuarios internos con cuenta MS** (sin acceso anónimo).
- Transiciones de estado (verificadas contra el sistema original):
  nueva→{en_revision,rechazada}; en_revision→{cotizada,rechazada};
  cotizada→{aprobada,rechazada,en_revision}; reapertura terminal→nueva solo
  super_admin. Roles: rechazar=super_admin/supervisor; reabrir=super_admin;
  avanzar=super_admin/ingeniero/supervisor.

## Regla de oro de construcción
Construir **TODO dentro de una Solución** de Dataverse (publicador prefijo
`csen_`). El Plan Developer es solo dev/1-usuario, NO producción. Cuando la
empresa licencie, se exporta la solución como **managed** y se importa al entorno
productivo, sin retrabajo. Nunca construir en el entorno default ni fuera de la
solución.

## Método de trabajo
- **Propón antes de construir.** Para cada pieza nueva, describe el diseño y
  espera visto bueno antes de armarla.
- **Documenta en el repo.** Cada decisión de arquitectura → un ADR en `docs/adr/`.
  Las guías de construcción viven en `docs/dataverse/` y `docs/powerapps/`.
- **El repo es la fuente de verdad.** Mantén `SESSION.md` actualizado con el
  estado y el "próximo paso concreto" (acción ejecutable, sin ambigüedad).
- Commits pequeños y descriptivos. Nunca `push --force`.

## Cómo debes comportarte
- **Verifica antes de afirmar.** Si una guía menciona un campo/opción del modelo
  original, contrástalo con `docs/dataverse/01-tablas.md`, no de memoria.
- **Sé honesto sobre límites.** La UI de Power Platform cambia seguido; si no estás
  seguro de un paso, dilo y pide captura, no inventes clics.
- **Señala riesgos antes de que ocurran** (fugas de datos, permisos, costos
  premium inesperados, portabilidad a producción).
- No adules. Si algo de lo que se pide es peor que una alternativa, dilo.

## Qué NO hacer
- No tocar el repo `axon` original (Laravel).
- No introducir conectores premium sin avisar (validar costo).
- No reintroducir EAV, bifásico, SharePoint ni acceso anónimo (ya descartados).
- No construir fuera de la Solución.

## Estado y próximo paso
Diseño documentado en `docs/dataverse/` (00 guía de construcción, 01 tablas, 02
BPF+roles, 03 despliegue) y `docs/powerapps/05` (YAML Canvas). **Próximo paso:**
construir en make.powerapps.com siguiendo `docs/dataverse/00-construir-en-powerapps.md`
— Bloque 0 (entorno+auditoría) y Bloque 1 (crear la Solución `Axon Solicitudes`).

## Mapa de documentos
- `docs/adr/0001` — SharePoint (superseded). `docs/adr/0002` — pivote a Dataverse (vigente).
- `docs/dataverse/00` — guía paso a paso de construcción en make.powerapps.com.
- `docs/dataverse/01` — esquema de tablas (campos, tipos, choices, relación 1:N).
- `docs/dataverse/02` — máquina de estados (flujo real-time) + security roles.
- `docs/dataverse/03` — Plan Developer y despliegue a producción vía Solución.
- `docs/powerapps/05` — YAML `.pa.yaml` de la app Canvas de captura (Dataverse).
- `docs/powerapps/06` — YAML completo consolidado (App + 8 pantallas), probado en Studio real.
- `docs/powerapps/07` — referencia rápida de las colecciones `colOpc*` para `App.OnStart`.
- `docs/powerapps/04` — app de gestión interna.
- `docs/powerautomate/03` — flujo F-1 (notificaciones + acuse), adaptado a Dataverse.
