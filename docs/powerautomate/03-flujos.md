> ⚠️ **Adaptado a Dataverse ([ADR 0002](../adr/0002-pivote-a-dataverse.md)).**
> Solo queda aquí **F-1 (notificaciones + acuse)**, ahora disparado por *fila
> creada* en la tabla Dataverse `Solicitud`; el código de referencia lo da una
> columna **Autonumber** (`reference_code`), ya no `guid()`. **F-2 (máquina de
> estados) se reemplazó por el flujo real-time de transiciones + security roles**
> — ver [dataverse/02-bpf-y-roles.md](../dataverse/02-bpf-y-roles.md). Con
> Dataverse desaparecen los workarounds `EstadoPrevio`/anti-bucle y la lista
> `HistorialEstados` (la audita Dataverse de forma nativa).

# Flujos de Power Automate

Reproduce las notificaciones del sistema original. Usa conectores estándar
(Dataverse, Office 365 Outlook) → sin costo premium.

```
F-1  Al crear solicitud  → notifica al equipo + acusa al solicitante
```

La validación de transiciones de estado y la auditoría **ya no viven aquí**:
las cubre el flujo real-time + security roles de
[dataverse/02-bpf-y-roles.md](../dataverse/02-bpf-y-roles.md).

---

## F-1 — Nueva solicitud recibida

Equivale a `NewSubmissionReceived` + `SubmissionConfirmed`. El
`reference_code` ya no lo genera el flujo: lo asigna la columna **Autonumber**
de Dataverse al crear la fila.

**Disparador:** Dataverse — *Cuando se agrega una fila* a la tabla `Solicitud`.

**Pasos:**

1. **Notificar al equipo interno** (reemplaza la notificación a `super_admin` y
   `supervisor`).
   - **Enviar un correo (V2)** — Office 365 Outlook.
   - Para: el grupo/lista de gestores (o destinatarios fijos del equipo).
   - Asunto: `Nueva solicitud @{triggerOutputs()?['body/cr_referencecode']} — @{triggerOutputs()?['body/cr_nombreproyecto']}`
   - Cuerpo: datos de contacto, proyecto, link al registro en la app Model-driven.

2. **Acuse al solicitante** (reemplaza `SubmissionConfirmed`).
   - Condición: `ContactoEmail` no vacío.
   - **Enviar un correo (V2)** a `ContactoEmail`.
   - Asunto: `Recibimos tu solicitud @{triggerOutputs()?['body/cr_referencecode']}`
   - Cuerpo: confirmación + código de referencia para seguimiento.

> Opcional: en lugar de correo al equipo, publicar en un canal de **Teams** con
> el conector estándar de Teams.

> Los nombres lógicos de columna (`cr_referencecode`, `cr_nombreproyecto`, …)
> dependen del prefijo del publisher de tu Solución — ajústalos a los reales de
> [dataverse/01-tablas.md](../dataverse/01-tablas.md).

---

## Resumen de equivalencias

| Sistema original | Power Automate / Dataverse |
|---|---|
| `reference_code = 'SOL-'.Str::random(8)` | Columna **Autonumber** `reference_code` (Dataverse, nativa) |
| `NewSubmissionReceived` → super_admin/supervisor | F-1 paso 1 (correo/Teams al equipo) |
| `SubmissionConfirmed` → solicitante | F-1 paso 2 (correo al `ContactoEmail`) |
| `SubmissionStateMachine::ALLOWED_TRANSITIONS` | Flujo real-time + security roles ([dataverse/02](../dataverse/02-bpf-y-roles.md)) |
| `SubmissionStatusHistory::create(...)` | Auditoría nativa de Dataverse |
| `SubmissionApprovedNotification` → super_admin/ingeniero | Flujo real-time ([dataverse/02](../dataverse/02-bpf-y-roles.md)) |
