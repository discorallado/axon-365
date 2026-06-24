# Flujos de Power Automate

Reproducen las notificaciones y la máquina de estados del sistema original.
Todos usan conectores estándar (SharePoint, Office 365 Outlook) → sin costo
premium.

```
F-1  Al crear solicitud  → genera código + notifica equipo + acusa al solicitante
F-2  Al cambiar estado   → valida la transición + registra HistorialEstados
```

---

## F-1 — Nueva solicitud recibida

Equivale a `NewSubmissionReceived` + `SubmissionConfirmed` + generación de
`reference_code`.

**Disparador:** SharePoint — *Cuando se crea un elemento* en la lista `Solicitudes`.

**Pasos:**

1. **Generar código de referencia.**
   - Inicializar variable `RefCode`:
     ```
     SOL-@{toUpper(substring(replace(guid(), '-', ''), 0, 8))}
     ```
   - **Actualizar elemento** en `Solicitudes`: `Title = RefCode`.

2. **Notificar al equipo interno** (reemplaza la notificación a `super_admin` y
   `supervisor`).
   - **Enviar un correo (V2)** — Office 365 Outlook.
   - Para: el grupo/lista de gestores (o destinatarios fijos del equipo).
   - Asunto: `Nueva solicitud @{variables('RefCode')} — @{triggerOutputs()?['body/NombreProyecto']}`
   - Cuerpo: datos de contacto, proyecto, link al ítem en SharePoint.

3. **Acuse al solicitante** (reemplaza `SubmissionConfirmed`).
   - Condición: `ContactoEmail` no vacío.
   - **Enviar un correo (V2)** a `ContactoEmail`.
   - Asunto: `Recibimos tu solicitud @{variables('RefCode')}`
   - Cuerpo: confirmación + código de referencia para seguimiento.

> Opcional: en lugar de correo al equipo, publicar en un canal de **Teams** con
> el conector estándar de Teams.

---

## F-2 — Cambio de estado (máquina de estados)

Equivale a `SubmissionStateMachine` + escritura de `SubmissionStatusHistory`.
Hace cumplir las transiciones válidas y audita cada cambio.

**Transiciones permitidas** (del enum `SubmissionStatus` original):

```
nueva        → en_revision
en_revision  → cotizada
cotizada     → aprobada     (terminal)
cotizada     → rechazada    (terminal)
```

Cualquier otra transición es inválida y debe revertirse.

**Disparador:** SharePoint — *Cuando se modifica un elemento* en `Solicitudes`.

**Pasos:**

1. **Obtener valores anterior y nuevo del estado.**
   - El disparador da el valor actual (`Estado`). Para el anterior, usar los
     **valores de la versión previa**: activar el disparador con
     *trigger condition* sobre cambio de `Estado`, y guardar el estado anterior
     en una columna oculta `EstadoPrevio` que el propio flujo actualiza al final
     (patrón estándar para "valor anterior" en SharePoint).

2. **Validar la transición.**
   - **Condición** — comprobar que `(EstadoPrevio → Estado)` esté en la lista
     permitida. Expresión, p. ej.:
     ```
     or(
       and(equals(EstadoPrevio,'nueva'),       equals(Estado,'en_revision')),
       and(equals(EstadoPrevio,'en_revision'), equals(Estado,'cotizada')),
       and(equals(EstadoPrevio,'cotizada'),    equals(Estado,'aprobada')),
       and(equals(EstadoPrevio,'cotizada'),    equals(Estado,'rechazada'))
     )
     ```
   - **Si NO es válida:** revertir — **Actualizar elemento** dejando
     `Estado = EstadoPrevio` y, opcionalmente, notificar al usuario que la
     transición no está permitida. Terminar.

3. **Si es válida — registrar en `HistorialEstados`.**
   - **Crear elemento** en `HistorialEstados`:
     - `Solicitud` = ID del ítem disparador
     - `EstadoAnterior` = `EstadoPrevio`
     - `EstadoNuevo` = `Estado`
     - `CambiadoPor` = `@{triggerOutputs()?['body/Editor/Email']}`
     - `Comentario` = (columna de comentario de la solicitud, si la usas)
     - `FechaCambio` = `@{utcNow()}`

4. **Actualizar `EstadoPrevio`** = `Estado` (para el próximo cambio).

5. **Notificar** al solicitante y/o asignado del nuevo estado (opcional, según
   reglas de negocio: p. ej. avisar al cliente cuando pasa a `cotizada`).

> **Nota sobre concurrencia:** el patrón `EstadoPrevio` es la forma estándar de
> obtener el "valor anterior" en SharePoint, que no expone el valor previo en el
> disparador. Alternativa más robusta: usar el historial de versiones vía la API
> REST `/versions`, pero es más complejo y para este volumen no se justifica.

---

## Resumen de equivalencias

| Sistema original | Power Automate |
|---|---|
| `reference_code = 'SOL-'.Str::random(8)` | F-1 paso 1 (`guid()` recortado) |
| `NewSubmissionReceived` → super_admin/supervisor | F-1 paso 2 (correo/Teams al equipo) |
| `SubmissionConfirmed` → solicitante | F-1 paso 3 (correo al `ContactoEmail`) |
| `SubmissionStateMachine::ALLOWED_TRANSITIONS` | F-2 paso 2 (condición de validación) |
| `SubmissionStatusHistory::create(...)` | F-2 paso 3 (crear ítem en HistorialEstados) |
