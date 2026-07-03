# SESSION.md — Estado de la migración M365

> Estado de trabajo entre sesiones. Para continuar desde casa (claude.ai u otra
> máquina), pega este archivo + el README al inicio de la conversación.

---

## Última actualización
2026-07-01 (Migración a controles Modern de Power Apps en el wizard Canvas)

## Qué es esto
Migración del **Módulo de Solicitudes de Tableros Eléctricos** desde el PMIS
Laravel/Filament (repo `axon`) a Microsoft 365. Motivo: la empresa no quiere
costear mantenimiento de app a medida; usa M365 que ya paga. El repo `axon`
original queda congelado como proyecto personal y NO se toca.

## Arquitectura aprobada — PIVOTE A DATAVERSE (2026-06-25, ADR 0002 supersede 0001)
- **Datos:** **Dataverse** — tablas `Solicitud`, `SolicitudTablero` (relación 1:N).
  Ya NO SharePoint.
- **Captura:** Power Apps Canvas reapuntada a Dataverse (YAML en `docs/powerapps/05`),
  construida con **controles Modern** (Fluent) — `ModernButton`,
  `ModernTextInput`, `ModernNumberInput`, `ModernDropdown`, `ModernCombobox`,
  `ModernToggle`, `ModernDatePicker`, `ModernText`, `ModernTabList`.
  `Group`/`Gallery` siguen clásicos (no tienen versión Modern).
  **Arquitectura final de pantallas** (mockup del usuario, confirmado):
  `scrContactoProyecto` / `scrTableroForm` / `scrDocumentacion` navegadas por
  un `ModernTabList` compartido (salto directo, sin bloqueo) + botón
  **Siguiente** por pantalla (valida y avanza en orden) + `scrConfirmacion`
  fuera del flujo de pestañas. `scrTableroForm` es maestro-detalle en una
  sola pantalla (galería izquierda + formulario completo derecha,
  `docs/powerapps/02`), con Editar/Eliminar **arriba de la galería** actuando
  sobre `galTableros.Selected` (no un botón por fila). Construcción completa
  en `docs/dataverse/05` (app, pestañas, pantallas 1/3/confirmación) +
  `docs/powerapps/02` (pantalla 2, referenciada desde dataverse/05).
- **Estados:** Business Process Flow + security roles (nativos) — reemplazan F-2.
- **Automatización:** Power Automate F-1 (notificaciones + acuse).
- **Gestión:** app Model-driven.
- **Confirmado: Dataverse completo vía Plan Developer.** Sirve para construir/validar.
  ⚠️ Developer = dev, 1 usuario, NO producción. Para pasar a la licencia de la empresa
  sin retrabajo: construir TODO dentro de una **Solución** y luego exportar managed →
  importar en el entorno productivo. Detalle en `docs/dataverse/03-...md`.
- Decisiones que se conservan: usuarios internos con cuenta MS; `bifasico` eliminado
  (B-1); modelo ancho (no EAV). Dataverse resuelve A-3/A-4 (roles), M-2 (umbral 5000),
  M-3 (Autonumber), HistorialEstados (auditoría nativa), EstadoPrevio (lo hace el BPF).

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
  de adjuntos, umbral 5000.
- Decisiones de la revisión **resueltas** (2026-06-24): A-3 → reglas de rol en F-2
  (gatea rechazar/reabrir, rol vía grupo AAD o lista RolesUsuarios); M-1 → adjuntos
  nativos planos + convención de nombres; B-1 → eliminado `bifasico` en M365.

### Pivote a Dataverse — hecho esta sesión (2026-06-25)
- `docs/adr/0002-pivote-a-dataverse.md` creado (supersede 0001).
- `docs/powerapps/05-canvas-yaml-captura.md` — YAML `.pa.yaml` de la pantalla de
  captura, enlazado a Dataverse (Patch a tablas, Choice `'Col (Tabla)'.Opcion`,
  lookup por registro).
- Banners de "superseded/actualizar a Dataverse" en los 4 docs de SharePoint/Canvas.
- README y SESSION actualizados al nuevo rumbo.

### Pendiente ⏳
- [x] Confirmado Dataverse completo (Plan Developer). ✅
- [x] Esquema reescrito: `docs/dataverse/01-tablas.md`. ✅
- [x] Estados reescritos: `docs/dataverse/02-bpf-y-roles.md` (flujo real-time de
      transiciones + matriz de security roles + BPF opcional). ✅
- [x] Plan Developer y despliegue: `docs/dataverse/03-plan-developer-y-despliegue.md`. ✅
- [x] Guía mecánica Solución/tablas/roles: `docs/dataverse/00-construir-en-powerapps.md`. ✅
- [x] Gestión interna como app Model-driven documentada: `docs/dataverse/04-app-model-driven.md`
      (Business Rules de validación, vistas, formularios, subgrid, app, dashboard). ✅
- [x] YAML completo de las ~37 columnas del tablero (`docs/powerapps/05-canvas-yaml-captura.md`):
      3 pantallas nuevas (Identificación / Ubicación-ambiente-montaje / Eléctrico /
      Diseño constructivo), `Guardar tablero en la colección` con los 36 campos
      agrupados en un `With(...)`, y el `Envío` con el mapeo completo Switch/ForAll
      de cada Choice (simple y multi-select) a `'Columna (Tabla)'.'Opción'`. ✅
- [x] **Arquitectura final del wizard Canvas (esta sesión, según mockup del usuario):**
      4 pantallas — `scrContactoProyecto`, `scrTableroForm` (maestro-detalle),
      `scrDocumentacion`, `scrConfirmacion` — con **barra de pestañas
      compartida** (Bloque 14b de `docs/dataverse/05`, resalta con
      `App.ActiveScreen`, copiar/pegar igual en las 3 pantallas de formulario)
      + botón **Siguiente** por pantalla que valida antes de avanzar.
      `docs/dataverse/05-construir-canvas-captura.md` reescrito para esta
      arquitectura (ya no describe el wizard de 4 pasos por tablero — esa
      alternativa quedó descartada). ✅
- [x] `docs/powerapps/02-canvas-guia-construccion.md` reescrito para encajar:
      nombres de pantalla actualizados (`scrTableroForm`, `scrContactoProyecto`,
      `scrDocumentacion`), **Editar/Eliminar movidos arriba de la galería**
      (actúan sobre `galTableros.Selected`, no un botón por fila — deshabilitados
      si no hay selección), botón **Siguiente** en la pantalla de tableros que
      valida ≥1 tablero antes de pasar a Documentación. Sigue con: patrón
      `varEditIndex` + `Reset()` de los 35 controles, `Default` de cada campo,
      validación de obligatorios, guardar con dos bugs corregidos (`Orden` no se
      recalcula al editar; campos "Otro" ocultos se limpian a `Blank()` — este
      fix también se aplicó en `docs/powerapps/05-canvas-yaml-captura.md`),
      eliminar con confirmación, indicador de modo nuevo/editando, y la sección
      "Qué más podrías necesitar" (mejoras opcionales + riesgo de colTableros
      solo en memoria). ✅
- [x] **Migración a controles Modern (esta sesión):** los 3 docs activos del wizard
      Canvas (`dataverse/05`, `powerapps/02`, `powerapps/05`) reescritos para usar
      `Modern<Control>@1.0.0` en vez de los controles clásicos. Verificado contra
      Microsoft Learn antes de escribir (no por memoria). Cambios de fondo, no solo
      de nombre: campos numéricos → `ModernNumberInput` (`.Value` ya numérico, `Min`/
      `Max` nativos reemplazan `Clamp`); `Toggle.Value`→`Toggle.Checked`; `Default`
      de `ModernDropdown` ahora necesita el record completo → se resuelve con
      `LookUp` contra colecciones `colOpc*` (nuevas, en `App.OnStart`, ~17, una por
      Choice) en vez de los `Switch` gigantes de antes; barra de pestañas armada a
      mano reemplazada por el control dedicado `ModernTabList`. `Group`/`Gallery`
      quedan clásicos (sin versión Modern). `docs/powerapps/04-app-gestion-interna.md`
      (superseded, SharePoint, fuera del build activo) no se tocó — avisar si se
      quiere modernizar también. ✅
- [ ] Construir de verdad en make.powerapps.com: Solución → tablas → choices → roles
      (`00`) → Business Rules → vistas → formularios → app (`04`) → Canvas de captura
      con pestañas + maestro-detalle, controles Modern
      (`dataverse/05` + `powerapps/02` + `powerapps/05`) → flujo real-time de estados
      → Power Automate F-1.
- [x] Repo en GitHub: `discorallado/axon-365` (privado). ✅

## Próximo paso concreto
Toda la documentación de construcción está lista. Construcción real en
**make.powerapps.com**, siguiendo en orden:
1. [docs/dataverse/00-construir-en-powerapps.md](docs/dataverse/00-construir-en-powerapps.md) —
   Solución, choices, tablas, security roles (Bloques 0-6).
2. [docs/dataverse/04-app-model-driven.md](docs/dataverse/04-app-model-driven.md) —
   Business Rules de validación, vistas, formularios, subgrid, app Model-driven,
   dashboard (Bloques 7-13).
3. [docs/dataverse/05-construir-canvas-captura.md](docs/dataverse/05-construir-canvas-captura.md) —
   crear la app y las 4 pantallas, la barra de pestañas compartida (Bloque 14b),
   Pantalla 1 (Bloque 15), y para la **Pantalla 2 (tableros)** saltar al Bloque
   16 que remite a
   [docs/powerapps/02-canvas-guia-construccion.md](docs/powerapps/02-canvas-guia-construccion.md)
   §3 completo (maestro-detalle). Seguir con Pantalla 3 (Bloque 17) y
   confirmación (Bloque 18). Fórmulas exactas de cada campo en
   [docs/powerapps/05-canvas-yaml-captura.md](docs/powerapps/05-canvas-yaml-captura.md).
   **Importante:** las etiquetas de Choice citadas en los `Switch` deben coincidir
   exactamente con las que se creen en el paso 1 — verificar antes de conectar a producción.
4. Flujo en tiempo real de estados ([docs/dataverse/02](docs/dataverse/02-bpf-y-roles.md) §3)
   y Power Automate F-1.

El usuario construye clicando en el navegador siguiendo las guías; Claude no
opera make.powerapps.com directamente (decisión: documentación en vez de
navegación en vivo).

## Notas de logística
- Continuable en claude.ai pegando este SESSION.md + README. Construcción 100% en navegador.
- Repo en GitHub: https://github.com/discorallado/axon-365 (privado). Remote `origin` configurado.
- Ruta local: `\\wsl.localhost\ddev\home\ubuntu\axon-365` (rama `main`).
