# Plan Developer y despliegue a producción

El desarrollo se hace en un entorno **Power Apps Plan Developer** (Dataverse
completo, gratis, individual). Es ideal para construir y validar, pero tiene
límites que definen **cómo** construir desde el día uno.

## Límites del Plan Developer

- **No es para producción.** Es un entorno de desarrollo/prueba.
- **Un solo usuario.** Solo el propietario usa las apps; el equipo NO puede operar
  ahí. La validación con usuarios reales requiere ya el entorno licenciado.
- Capacidad y disponibilidad acotadas (suficiente para construir y demostrar).

## Regla de oro: construir TODO dentro de una Solución

Para que "pasar a la licencia de la empresa" sea **sin retrabajo**, todo se crea
dentro de una **Solución** (Solution) de Dataverse, no en el entorno default:

- Una solución `Axon Solicitudes` (con su *publisher* y prefijo, p. ej. `csen_`).
- Dentro: las tablas (`Solicitud`, `SolicitudTablero`, `HistorialEstado`), Choices,
  la app Canvas de captura, la app Model-driven, el flujo real-time de estados, el
  BPF, los flujos de Power Automate y los security roles.

## Camino a producción (cuando la empresa licencie)

```
Entorno Developer (dev)                    Entorno Producción (licenciado)
  Solución no administrada      ──export──►   Solución administrada (import)
  (se construye y edita aquí)                 (se despliega aquí, solo lectura)
```

1. Empresa adquiere licencias (Power Apps por usuario o por app) y crea el entorno
   **Producción** con Dataverse.
2. **Exportar** la solución desde Developer como **managed**.
3. **Importar** la solución managed en Producción.
4. Configurar conexiones, asignar **security roles** a los usuarios reales, cargar
   datos si aplica.
5. (Recomendado) Mantener un entorno de **pruebas/UAT** intermedio si el volumen lo
   amerita.

> **Qué NO hacer:** construir en el entorno *default* o fuera de una solución. Mover
> eso a producción después implica recrear a mano. La solución es el contenedor
> portable; es la diferencia entre "exportar/importar" y "rehacer".

## Datos vs. esquema

La solución lleva el **esquema** (tablas, apps, flujos, roles), NO los **datos**.
Las solicitudes capturadas en Developer son de prueba; en producción se parte
limpio (o se migran datos reales con una herramienta de importación si los hubiera).
