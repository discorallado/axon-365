# Máquina de estados y permisos en Dataverse

Reemplaza al flujo F-2 de SharePoint (la validación a mano con `EstadoPrevio` y
anti-bucle). En Dataverse el enforcement se reparte en tres piezas:

1. **Security roles** — quién puede leer/escribir cada tabla (acceso grueso).
2. **Flujo en tiempo real** (real-time / instant) sobre el cambio de `Estado` —
   valida la transición y la regla por rol; revierte si es inválida. Es el
   equivalente fiel al `SubmissionStateMachine` original.
3. **Business Process Flow (BPF)** — opcional, guía visual del avance. UX, no
   enforcement.

> **Ventaja Dataverse:** el disparador entrega el **valor anterior (pre-image)** de
> forma nativa. Ya no se necesita la columna `EstadoPrevio` ni el truco anti-bucle
> de SharePoint.

---

## 1. Transiciones (verificadas contra `SubmissionStateMachine`)

```
nueva        → en_revision, rechazada
en_revision  → cotizada, rechazada
cotizada     → aprobada, rechazada, en_revision
aprobada     → nueva   (reapertura, SOLO super_admin)
rechazada    → nueva   (reapertura, SOLO super_admin)
mismo → mismo: nunca
```

**Reglas por rol:**
- **Avanzar** (en_revision/cotizada/aprobada) → `super_admin`, `ingeniero`, `supervisor`.
- **Rechazar** → `super_admin`, `supervisor`.
- **Reabrir** (terminal → nueva) → solo `super_admin`.

---

## 2. Security roles (acceso por tabla)

Crear 5 roles en la solución, mapeando los roles del PMIS original. Privilegios
sobre `Solicitud` / `SolicitudTablero` / `HistorialEstado`:

| Rol | Solicitud | SolicitudTablero | HistorialEstado | Nivel de acceso |
|---|---|---|---|---|
| `super_admin` | C R W D | C R W D | C R | Organización |
| `supervisor` | R W | R W | C R | Organización |
| `ingeniero` | R W | R W | C R | Organización |
| `calidad` | R | R | R | Organización |
| `tecnico` | — | — | — | (sin acceso al módulo) |

(C=Create, R=Read, W=Write, D=Delete. El público no aplica: la captura la hacen
usuarios internos autenticados.)

> Los security roles dan el acceso **grueso** (quién entra y edita). La regla fina
> "quién puede mover a qué estado" NO se expresa con roles → va en el flujo (§3).
> Esto sí replica fielmente la matriz de policies, a diferencia de SharePoint.

---

## 3. Flujo en tiempo real — validación de transición + rol

**Tipo:** flujo de Dataverse *en tiempo real* (real-time), o clasic real-time
workflow, sobre **Actualizar** de `Solicitud`, filtrado a la columna `Estado`.
Síncrono, para poder **revertir antes de guardar** si es inválido.

**Lógica:**

```
ENTRADA: estadoAnterior (pre-image de Estado), estadoNuevo (Estado),
         usuario = modificador, roles = roles de seguridad del usuario

1. Si estadoAnterior == estadoNuevo  → no hacer nada (salir).

2. transicionValida =
     (anterior=nueva       Y nuevo ∈ {en_revision, rechazada}) O
     (anterior=en_revision Y nuevo ∈ {cotizada, rechazada})    O
     (anterior=cotizada    Y nuevo ∈ {aprobada, rechazada, en_revision}) O
     (anterior ∈ {aprobada, rechazada} Y nuevo = nueva)

   Si NO transicionValida → ERROR "Transición no permitida" (cancela el guardado).

3. Regla por rol:
     - nuevo = rechazada      → requiere rol ∈ {super_admin, supervisor}
     - reapertura (ant. terminal → nueva) → requiere rol = super_admin
     - resto (avanzar)        → requiere rol ∈ {super_admin, ingeniero, supervisor}
   Si no cumple → ERROR "No autorizado para esta transición" (cancela).

4. (Válida) Registrar en HistorialEstado:
     Solicitud, EstadoAnterior=anterior, EstadoNuevo=nuevo,
     CambiadoPor=usuario, Comentario, Fecha=ahora, Organizacion.

5. Si nuevo = aprobada → notificar a super_admin + ingeniero (Power Automate aparte
     o paso de correo). Reemplaza SubmissionApprovedNotification.
```

> Para **cancelar el guardado** en un flujo real-time se usa el resultado de error
> de la etapa (en classic workflow: *Stop Workflow* con estado *Canceled*; en
> flujo real-time de Power Automate: acción que arroje error de negocio). Así la
> transición inválida no persiste — equivalente al `abort(403)` original.

**Obtener los roles del usuario:** en Dataverse, consultar `systemuserroles` del
modificador, o usar la expresión de pertenencia a Team/rol. Es más directo que el
workaround de SharePoint (lista `RolesUsuarios`).

---

## 4. Business Process Flow (opcional, solo UX)

Un BPF da una barra de progreso guiada. Es **lineal por naturaleza**, así que
modela bien el camino feliz y mal las vueltas atrás/reapertura. Por eso el
enforcement real vive en §3, no en el BPF.

Etapas sugeridas del BPF (visual):
```
[Nueva] → [En revisión] → [Cotización] → [Resolución]
```
- En "Resolución", el campo `Estado` decide aprobada/rechazada.
- El retroceso `cotizada → en_revision` y la reapertura los permite el flujo §3
  aunque el BPF no los dibuje. Si el BPF estorba, omitirlo: no es obligatorio.

---

## 5. Comparación con el enfoque SharePoint (F-2)

| SharePoint F-2 | Dataverse |
|---|---|
| Columna `EstadoPrevio` para el valor anterior | Pre-image nativo |
| Trigger condition anti-bucle | No necesario (real-time sobre campo) |
| Rol vía lista `RolesUsuarios` | `systemuserroles` nativo |
| Revertir con "Actualizar elemento" | Cancelar el guardado (síncrono) |
| Lista `HistorialEstados` obligatoria | Auditoría nativa + tabla solo para comentario |
