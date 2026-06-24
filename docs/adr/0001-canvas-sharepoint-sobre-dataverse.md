# ADR 0001 — Power Apps Canvas + SharePoint Lists para el módulo de solicitudes

- **Estado:** Aceptado
- **Fecha:** 2026-06-24
- **Contexto del repo:** migración del módulo de Solicitudes de Tableros desde
  Laravel/Filament (repo `axon`) a Microsoft 365.

## Contexto

El sistema original es un PMIS en Laravel 12 + Filament. La empresa decidió **no
costear el mantenimiento de una aplicación a medida** y operar sobre las
herramientas de Microsoft 365 que ya tiene licenciadas. El primer módulo a migrar
es el de solicitudes de tableros eléctricos.

El módulo original tiene un modelo relacional de tres niveles con auditoría y
adjuntos:

```
SubmissionRequest (1) ──< SubmissionItem (N, ~35 campos)
        ├──< Attachment (polimórfico)
        └──< SubmissionStatusHistory (auditoría de estados)
```

Características clave a preservar:
- **Patrón multi-tablero:** N tableros por solicitud, agregados dinámicamente.
- Lógica condicional (mostrar/ocultar campos según respuestas).
- Cálculo automático de corriente nominal.
- Máquina de estados auditada y gateada por rol (transiciones reales en
  `docs/powerautomate/03-flujos.md` F-2; incluye rechazo directo desde cualquier
  estado no terminal, retroceso `cotizada → en_revision`, y reapertura de
  terminales a `nueva` solo por `super_admin`).
- Adjuntos por solicitud y por tablero.

## Decisión

Construir sobre **Power Apps Canvas (captura y gestión) + SharePoint Lists
(datos) + Power Automate (automatización)**.

Razón decisiva de licenciamiento: una Power App Canvas que usa **solo conectores
estándar** (SharePoint, Outlook, Teams) está **incluida en la licencia M365** — no
requiere Power Apps premium ni Dataverse. Esto cumple el objetivo de costo cero y,
a la vez, conserva el patrón multi-tablero mediante una **galería editable**, que
es lo que ninguna alternativa de bajo costo lograba.

Condición habilitante: quienes llenan el formulario son **usuarios internos / con
cuenta Microsoft**, por lo que no se necesita acceso anónimo. (Confirmado con el
dueño del producto, 2026-06-24.)

### Decisiones menores asociadas

- **Choice múltiple nativo (opción A)** para `special_environment`,
  `required_protections`, `preferred_brands`, en lugar de listas hijas
  relacionales (opción B). Suficiente para el volumen; evita sobre-ingeniería.
- **`raw_data` (JSON snapshot) se descarta.** El versionado se confía al Historial
  de versiones nativo de SharePoint.
- **RichEditor → columnas de texto enriquecido** de SharePoint; se conserva el
  formato de `board_function` y `loads_to_feed`.
- **`reference_code`** lo genera Power Automate al crear la cabecera.

### Matriz de permisos (verificada contra `SubmissionRequestPolicy` y `SubmissionStateMachine`)

| Rol | Ver / Exportar | Avanzar estado | Rechazar | Reabrir (terminal→nueva) | Asignar | Borrar adjuntos |
|---|---|---|---|---|---|---|
| `super_admin` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `supervisor` | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ (si es el asignado) |
| `ingeniero` | ✅ | ✅ | ❌ | ❌ | ✅ | ✅ |
| `calidad` | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| `tecnico` | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

> Corrige una versión previa de esta matriz: `calidad` **no** cambia estados (solo
> ve/exporta) y reabrir es **solo** `super_admin`. SharePoint solo ofrece permisos
> de lista gruesos; replicar este detalle por rol/acción es una limitación abierta
> (ver Consecuencias y la decisión A-3 de la revisión).

## Alternativas descartadas

| Alternativa | Por qué se descartó |
|---|---|
| **Microsoft Forms** | No soporta lista dinámica de ítems → rompe el patrón multi-tablero. Sin lógica de cálculo. Solo texto plano (pierde RichEditor). Servía solo si se necesitara acceso 100% anónimo. |
| **Dataverse** | Máxima fidelidad (relaciones, auditoría nativa, Business Process Flow, RBAC granular), pero **requiere licencia premium** por usuario/app. Contradice el motivo de la migración (costo cero). Reconsiderar solo si la empresa adopta licencias premium. |
| **Power Pages** | Permitiría acceso anónimo + lista dinámica, pero es **licencia premium aparte** (~USD 200/mes base). Innecesario al ser usuarios internos. |
| **Excel** | Plano, sin relaciones, sin integridad referencial, sin adjuntos, problemas de concurrencia. Microsoft desaconseja Power Apps sobre Excel en producción. Perdía el patrón multi-tablero. |

## Consecuencias

**Positivas:**
- Costo de licencia cero (dentro de M365 existente).
- Sin mantenimiento de código propio.
- Patrón multi-tablero, lógica condicional y cálculo de corriente preservados.
- Auditoría vía Historial de versiones + lista `HistorialEstados`.

**Negativas / límites:**
- El solicitante debe iniciar sesión con cuenta MS (aceptable: son internos).
- SharePoint no soporta choices condicionales ni cálculos → se trasladan a la app
  (Power Fx). Si se editara fuera de la app (directo en la lista), esa lógica no
  aplica.
- RBAC a nivel de lista es más grueso que las policies de Laravel.
- La obtención del "estado anterior" en Power Automate requiere el patrón de
  columna `EstadoPrevio` (workaround estándar de SharePoint).

## Referencias

- Esquema de listas: `docs/sharepoint/01-listas-esquema.md`
- Guía Power Apps: `docs/powerapps/02-canvas-guia-construccion.md`
- Flujos: `docs/powerautomate/03-flujos.md`
- Sistema original: repo `axon`, `REQ-0001` y `PublicFormWizard`.
