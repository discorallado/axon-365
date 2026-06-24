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
     SOL-@{toUpper(substring(replace(guid(), '-', ''), 0, 12))}
     ```
   - 12 hex (no 8) para acercarse a la entropía del original `Str::random(8)`.
   - **Verificar unicidad:** antes de asignar, **Obtener elementos** filtrando
     `Title eq 'RefCode'`; si existe, regenerar. (El original tampoco fuerza
     unicidad a nivel BD, pero aquí el riesgo de colisión es mayor.)
   - **Actualizar elemento** en `Solicitudes`: `Title = RefCode`.

2. **Inicializar `EstadoPrevio`** = `nueva` (respaldo, por si la app no lo seteó).
   Necesario para que F-2 evalúe correctamente la primera transición.

3. **Notificar al equipo interno** (reemplaza la notificación a `super_admin` y
   `supervisor`).
   - **Enviar un correo (V2)** — Office 365 Outlook.
   - Para: el grupo/lista de gestores (o destinatarios fijos del equipo).
   - Asunto: `Nueva solicitud @{variables('RefCode')} — @{triggerOutputs()?['body/NombreProyecto']}`
   - Cuerpo: datos de contacto, proyecto, link al ítem en SharePoint.

4. **Acuse al solicitante** (reemplaza `SubmissionConfirmed`).
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

**Transiciones permitidas** (verificadas contra `SubmissionStateMachine::ALLOWED_TRANSITIONS`
del sistema original, no inventar):

```
nueva        → en_revision, rechazada
en_revision  → cotizada, rechazada
cotizada     → aprobada, rechazada, en_revision   ← puede volver a revisión
aprobada     → nueva   (reapertura, SOLO super_admin)
rechazada    → nueva   (reapertura, SOLO super_admin)
```

- El mismo estado → mismo estado **nunca** se permite.
- `aprobada` y `rechazada` son terminales: solo se "reabren" a `nueva`.
- **Reglas por rol** (del original): rechazar → `super_admin`/`supervisor`;
  avanzar (cualquier otra) → `super_admin`/`ingeniero`/`supervisor`; reabrir →
  solo `super_admin`.
  **Decisión A-3 (resuelta):** la regla dura se aplica **aquí, en F-2** (servidor),
  igual que el `SubmissionStateMachine` original; la app Canvas solo espeja las
  reglas para UX. Se gatean únicamente las acciones sensibles —**rechazar** y
  **reabrir**—; el resto se cubre con el permiso de edición de la lista. El rol del
  editor se resuelve por **grupo de seguridad de Azure AD por rol** (o una lista
  `RolesUsuarios` si no se administra en AAD). Ver paso 2-bis.

Cualquier otra transición es inválida y debe revertirse.

**Disparador:** SharePoint — *Cuando se modifica un elemento* en `Solicitudes`.

**⚠️ Trigger condition obligatoria (evita auto-bucle).** F-2 modifica
`EstadoPrevio` al final, y F-1 modifica `Title`; ambos re-dispararían F-2. Limitar
el disparo a cambios reales de `Estado`:

```
@not(equals(triggerOutputs()?['body/Estado/Value'], triggerOutputs()?['body/EstadoPrevio/Value']))
```

**Pasos:**

1. **Obtener valores anterior y nuevo del estado.**
   - El disparador da el valor actual (`Estado`). El anterior se lee de la columna
     oculta `EstadoPrevio`, que el flujo actualiza al final (patrón estándar para
     "valor anterior" en SharePoint, que no expone la versión previa en el disparador).
   - `EstadoPrevio` se inicializa a `nueva` al crear (app + F-1), de modo que la
     primera transición se evalúe bien.

2. **Validar la transición.**
   - **Condición** — comprobar que `(EstadoPrevio → Estado)` esté permitida:
     ```
     or(
       and(equals(EstadoPrevio,'nueva'),       or(equals(Estado,'en_revision'), equals(Estado,'rechazada'))),
       and(equals(EstadoPrevio,'en_revision'), or(equals(Estado,'cotizada'),    equals(Estado,'rechazada'))),
       and(equals(EstadoPrevio,'cotizada'),    or(equals(Estado,'aprobada'),     equals(Estado,'rechazada'), equals(Estado,'en_revision'))),
       and(or(equals(EstadoPrevio,'aprobada'), equals(EstadoPrevio,'rechazada')), equals(Estado,'nueva'))
     )
     ```
   - **Si NO es válida:** revertir — **Actualizar elemento** dejando
     `Estado = EstadoPrevio` y notificar al usuario que la transición no está
     permitida. Terminar.

2-bis. **Validar el rol del editor (decisión A-3).** Solo para acciones sensibles:
   - Obtener el rol del editor: comprobar pertenencia del
     `@{triggerOutputs()?['body/Editor/Email']}` a un grupo de seguridad de Azure
     AD (acción *Comprobar pertenencia a grupo* / `Office 365 Groups`), o `LookUp`
     en la lista `RolesUsuarios`.
   - **Si `Estado = rechazada`** y el rol ∉ {`super_admin`,`supervisor`} → revertir
     a `EstadoPrevio`, notificar "No autorizado para rechazar". Terminar.
   - **Si la transición es reapertura** (`EstadoPrevio` terminal → `nueva`) y el rol
     ≠ `super_admin` → revertir, notificar "Solo super_admin puede reabrir". Terminar.
   - El resto de transiciones (avanzar) no requieren chequeo extra de rol aquí.

3. **Si es válida — registrar en `HistorialEstados`.**
   - **Crear elemento** en `HistorialEstados`:
     - `Solicitud` = ID del ítem disparador
     - `Organizacion` = `Organizacion` del ítem (multi-tenant-ready)
     - `EstadoAnterior` = `EstadoPrevio`
     - `EstadoNuevo` = `Estado`
     - `CambiadoPor` = `@{triggerOutputs()?['body/Editor/Email']}`
     - `Comentario` = (columna de comentario de la solicitud, si la usas)
     - `FechaCambio` = `@{utcNow()}`

4. **Actualizar `EstadoPrevio`** = `Estado` (para el próximo cambio).

5. **Notificación de aprobación** (reproduce `SubmissionApprovedNotification`).
   - Condición: `Estado = aprobada`.
   - Notificar a los usuarios **`super_admin` + `ingeniero`** activos (no al
     solicitante). En M365: correo a un grupo/lista de distribución de esos roles.

6. **Notificación de avance al solicitante/asignado** (opcional, según negocio:
   p. ej. avisar al cliente cuando pasa a `cotizada`).

> **Nota sobre concurrencia:** el patrón `EstadoPrevio` es la forma estándar de
> obtener el "valor anterior" en SharePoint, que no expone el valor previo en el
> disparador. Alternativa más robusta: usar el historial de versiones vía la API
> REST `/versions`, pero es más complejo y para este volumen no se justifica.

---

## Resumen de equivalencias

| Sistema original | Power Automate |
|---|---|
| `reference_code = 'SOL-'.Str::random(8)` | F-1 paso 1 (`guid()` recortado a 12 hex + chequeo de unicidad) |
| `NewSubmissionReceived` → super_admin/supervisor | F-1 paso 3 (correo/Teams al equipo) |
| `SubmissionConfirmed` → solicitante | F-1 paso 4 (correo al `ContactoEmail`) |
| `SubmissionStateMachine::ALLOWED_TRANSITIONS` | F-2 paso 2 (condición de validación, incl. rechazo directo, retroceso y reapertura) |
| `SubmissionStatusHistory::create(...)` | F-2 paso 3 (crear ítem en HistorialEstados) |
| `SubmissionApprovedNotification` → super_admin/ingeniero | F-2 paso 5 |
