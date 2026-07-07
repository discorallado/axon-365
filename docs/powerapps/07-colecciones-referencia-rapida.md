# Referencia rápida — Collections (Colecciones) para crear a mano en make.powerapps.com

Listado de todas las 18 colecciones que necesita la app Canvas de captura (`Axon - Captura de Solicitudes`). Cada una alimenta un `ModernDropdown` o `ModernCombobox` en el formulario del tablero.

## Formato de creación

En `App` → `OnStart` → pega cada línea como:
```
ClearCollect(nombreColeccion; Table({Value:"código"; Label:"Etiqueta exacta"}; {Value:"..."; Label:"..."}; ...));;
```

El `Value` es lo que se guarda en Dataverse (la opción elegida); el `Label` es lo que ve el usuario en el dropdown.

---

## Las 18 colecciones

### 1. colOpcIngenieriaPor
```
{Value:"csenergy"; Label:"CSEnergy la provee"}
{Value:"cliente"; Label:"Por parte del cliente"}
{Value:"conjunta"; Label:"Conjunta (CSEnergy + Cliente)"}
```

### 2. colOpcTipoEntrega
```
{Value:"tablero"; Label:"Tablero Eléctrico"}
{Value:"sala"; Label:"Sala Eléctrica"}
{Value:"producto"; Label:"Producto Eléctrico"}
```

### 3. colOpcInstalacionNuevaReemplazo
```
{Value:"nueva"; Label:"Instalación nueva"}
{Value:"reemplazo"; Label:"Reemplazo de existente"}
```

### 4. colOpcTipoTablero
```
{Value:"fuerza"; Label:"Fuerza/Potencia"}
{Value:"alumbrado"; Label:"Alumbrado/Distribución BT"}
{Value:"control"; Label:"Control/Automatización"}
{Value:"transfer"; Label:"Transferencia (ATS/MTS)"}
{Value:"sincronizacion"; Label:"Sincronización de Generadores"}
{Value:"remoto"; Label:"Distribución Remoto"}
{Value:"pfcs"; Label:"Factor de Potencia"}
{Value:"medicion"; Label:"Medición/Centro de Carga"}
{Value:"variadores"; Label:"Variadores de Frecuencia"}
{Value:"arrancadores"; Label:"Arrancadores Suaves"}
{Value:"ups"; Label:"UPS/Respaldo"}
{Value:"otro"; Label:"Otro"}
```

### 5. colOpcUbicacion
```
{Value:"interior"; Label:"Interior"}
{Value:"exterior"; Label:"Exterior"}
```

### 6. colOpcAmbienteEspecial
```
{Value:"marino"; Label:"Ambiente marino"}
{Value:"minero"; Label:"Ambiente minero"}
{Value:"humedo"; Label:"Húmedo"}
{Value:"corrosivo"; Label:"Corrosivo"}
{Value:"polvoriento"; Label:"Polvoriento"}
{Value:"explosivo"; Label:"Atmósfera explosiva"}
{Value:"otro"; Label:"Otro"}
```

### 7. colOpcTipoMontaje
```
{Value:"autosoportado"; Label:"Autosoportado"}
{Value:"mural"; Label:"Mural"}
{Value:"rack_19"; Label:"Rack 19"}
{Value:"pedestal"; Label:"Pedestal"}
{Value:"otro"; Label:"Otro"}
```

### 8. colOpcTension
```
{Value:"220"; Label:"220 V"}
{Value:"380"; Label:"380 V"}
{Value:"400"; Label:"400 V"}
{Value:"440"; Label:"440 V"}
{Value:"480"; Label:"480 V"}
{Value:"690"; Label:"690 V"}
{Value:"1000"; Label:"1000 V"}
{Value:"otro"; Label:"Otro"}
```

### 9. colOpcSistema
```
{Value:"trifasico"; Label:"Trifásico"}
{Value:"monofasico"; Label:"Monofásico"}
{Value:"dc"; Label:"Corriente continua (DC)"}
{Value:"otro"; Label:"Otro"}
```

### 10. colOpcUnidadPotencia
```
{Value:"kW"; Label:"kW"}
{Value:"kVA"; Label:"kVA"}
```

### 11. colOpcFrecuencia
```
{Value:"50"; Label:"50 Hz"}
{Value:"60"; Label:"60 Hz"}
{Value:"otro"; Label:"Otra"}
```

### 12. colOpcProteccionesRequeridas
```
{Value:"interruptor_automatico"; Label:"Interruptor automático"}
{Value:"diferencial"; Label:"Diferencial"}
{Value:"fusible"; Label:"Fusible"}
{Value:"relevo_sobrecarga"; Label:"Relevo de sobrecarga"}
{Value:"relevo_falla_tierra"; Label:"Relevo de falla a tierra"}
{Value:"proteccion_tension"; Label:"Protección de tensión"}
{Value:"proteccion_corriente"; Label:"Protección de corriente"}
{Value:"descargador_tension"; Label:"Descargador de tensión"}
{Value:"otro"; Label:"Otro"}
```

### 13. colOpcMarcasPreferidas
```
{Value:"schneider"; Label:"Schneider Electric"}
{Value:"siemens"; Label:"Siemens"}
{Value:"abb"; Label:"ABB"}
{Value:"legrand"; Label:"Legrand"}
{Value:"eaton"; Label:"Eaton"}
{Value:"chint"; Label:"Chint"}
{Value:"hager"; Label:"Hager"}
{Value:"weidmuller"; Label:"Weidmüller"}
{Value:"phoenix"; Label:"Phoenix Contact"}
{Value:"otro"; Label:"Otro"}
```

### 14. colOpcMaterialGabinete
```
{Value:"acero_pintado"; Label:"Acero pintado"}
{Value:"acero_galvanizado"; Label:"Acero galvanizado"}
{Value:"acero_inoxidable"; Label:"Acero inoxidable"}
{Value:"acero_inox_316"; Label:"Acero inoxidable 316"}
{Value:"fibra_vidrio"; Label:"Fibra de vidrio"}
{Value:"poliester"; Label:"Poliéster"}
{Value:"aluminio"; Label:"Aluminio"}
```

### 15. colOpcColorGabinete
```
{Value:"7035"; Label:"RAL 7035 (gris claro)"}
{Value:"7016"; Label:"RAL 7016 (gris antracita)"}
{Value:"9016"; Label:"RAL 9016 (blanco tráfico)"}
{Value:"9005"; Label:"RAL 9005 (negro)"}
{Value:"5010"; Label:"RAL 5010 (azul)"}
{Value:"6005"; Label:"RAL 6005 (verde)"}
{Value:"otro"; Label:"Otro"}
```

### 16. colOpcTipoVentilacion
```
{Value:"natural"; Label:"Natural"}
{Value:"forzada"; Label:"Forzada (ventilador)"}
{Value:"sellado"; Label:"Sellado (IP alto)"}
{Value:"climatizado"; Label:"Climatizado (aire acondicionado)"}
```

### 17. colOpcExpansionFutura
```
{Value:"no"; Label:"Sin expansión"}
{Value:"10"; Label:"10%"}
{Value:"20"; Label:"20%"}
{Value:"30"; Label:"30%"}
{Value:"otro"; Label:"Otro"}
```

### 18. colOpcSiNo
Para toggle/booleanos:
```
{Value:"true"; Label:"Sí"}
{Value:"false"; Label:"No"}
```

---

## Cómo usarlo en App.OnStart

Copia el bloque completo de abajo, pégalo en el `OnStart` de la app (arriba del YAML del 06):

```powerapps
Set(varEditIndex; Blank());;
ClearCollect(colTableros; []);;
ClearCollect(colOpcIngenieriaPor; Table(
  {Value:"csenergy"; Label:"CSEnergy la provee"};
  {Value:"cliente"; Label:"Por parte del cliente"};
  {Value:"conjunta"; Label:"Conjunta (CSEnergy + Cliente)"}
));;
ClearCollect(colOpcTipoEntrega; Table(
  {Value:"tablero"; Label:"Tablero Eléctrico"};
  {Value:"sala"; Label:"Sala Eléctrica"};
  {Value:"producto"; Label:"Producto Eléctrico"}
));;
ClearCollect(colOpcInstalacionNuevaReemplazo; Table(
  {Value:"nueva"; Label:"Instalación nueva"};
  {Value:"reemplazo"; Label:"Reemplazo de existente"}
));;
ClearCollect(colOpcTipoTablero; Table(
  {Value:"fuerza"; Label:"Fuerza/Potencia"};
  {Value:"alumbrado"; Label:"Alumbrado/Distribución BT"};
  {Value:"control"; Label:"Control/Automatización"};
  {Value:"transfer"; Label:"Transferencia (ATS/MTS)"};
  {Value:"sincronizacion"; Label:"Sincronización de Generadores"};
  {Value:"remoto"; Label:"Distribución Remoto"};
  {Value:"pfcs"; Label:"Factor de Potencia"};
  {Value:"medicion"; Label:"Medición/Centro de Carga"};
  {Value:"variadores"; Label:"Variadores de Frecuencia"};
  {Value:"arrancadores"; Label:"Arrancadores Suaves"};
  {Value:"ups"; Label:"UPS/Respaldo"};
  {Value:"otro"; Label:"Otro"}
));;
ClearCollect(colOpcUbicacion; Table(
  {Value:"interior"; Label:"Interior"};
  {Value:"exterior"; Label:"Exterior"}
));;
ClearCollect(colOpcAmbienteEspecial; Table(
  {Value:"marino"; Label:"Ambiente marino"};
  {Value:"minero"; Label:"Ambiente minero"};
  {Value:"humedo"; Label:"Húmedo"};
  {Value:"corrosivo"; Label:"Corrosivo"};
  {Value:"polvoriento"; Label:"Polvoriento"};
  {Value:"explosivo"; Label:"Atmósfera explosiva"};
  {Value:"otro"; Label:"Otro"}
));;
ClearCollect(colOpcTipoMontaje; Table(
  {Value:"autosoportado"; Label:"Autosoportado"};
  {Value:"mural"; Label:"Mural"};
  {Value:"rack_19"; Label:"Rack 19"};
  {Value:"pedestal"; Label:"Pedestal"};
  {Value:"otro"; Label:"Otro"}
));;
ClearCollect(colOpcTension; Table(
  {Value:"220"; Label:"220 V"};
  {Value:"380"; Label:"380 V"};
  {Value:"400"; Label:"400 V"};
  {Value:"440"; Label:"440 V"};
  {Value:"480"; Label:"480 V"};
  {Value:"690"; Label:"690 V"};
  {Value:"1000"; Label:"1000 V"};
  {Value:"otro"; Label:"Otro"}
));;
ClearCollect(colOpcSistema; Table(
  {Value:"trifasico"; Label:"Trifásico"};
  {Value:"monofasico"; Label:"Monofásico"};
  {Value:"dc"; Label:"Corriente continua (DC)"};
  {Value:"otro"; Label:"Otro"}
));;
ClearCollect(colOpcUnidadPotencia; Table(
  {Value:"kW"; Label:"kW"};
  {Value:"kVA"; Label:"kVA"}
));;
ClearCollect(colOpcFrecuencia; Table(
  {Value:"50"; Label:"50 Hz"};
  {Value:"60"; Label:"60 Hz"};
  {Value:"otro"; Label:"Otra"}
));;
ClearCollect(colOpcProteccionesRequeridas; Table(
  {Value:"interruptor_automatico"; Label:"Interruptor automático"};
  {Value:"diferencial"; Label:"Diferencial"};
  {Value:"fusible"; Label:"Fusible"};
  {Value:"relevo_sobrecarga"; Label:"Relevo de sobrecarga"};
  {Value:"relevo_falla_tierra"; Label:"Relevo de falla a tierra"};
  {Value:"proteccion_tension"; Label:"Protección de tensión"};
  {Value:"proteccion_corriente"; Label:"Protección de corriente"};
  {Value:"descargador_tension"; Label:"Descargador de tensión"};
  {Value:"otro"; Label:"Otro"}
));;
ClearCollect(colOpcMarcasPreferidas; Table(
  {Value:"schneider"; Label:"Schneider Electric"};
  {Value:"siemens"; Label:"Siemens"};
  {Value:"abb"; Label:"ABB"};
  {Value:"legrand"; Label:"Legrand"};
  {Value:"eaton"; Label:"Eaton"};
  {Value:"chint"; Label:"Chint"};
  {Value:"hager"; Label:"Hager"};
  {Value:"weidmuller"; Label:"Weidmüller"};
  {Value:"phoenix"; Label:"Phoenix Contact"};
  {Value:"otro"; Label:"Otro"}
));;
ClearCollect(colOpcMaterialGabinete; Table(
  {Value:"acero_pintado"; Label:"Acero pintado"};
  {Value:"acero_galvanizado"; Label:"Acero galvanizado"};
  {Value:"acero_inoxidable"; Label:"Acero inoxidable"};
  {Value:"acero_inox_316"; Label:"Acero inoxidable 316"};
  {Value:"fibra_vidrio"; Label:"Fibra de vidrio"};
  {Value:"poliester"; Label:"Poliéster"};
  {Value:"aluminio"; Label:"Aluminio"}
));;
ClearCollect(colOpcColorGabinete; Table(
  {Value:"7035"; Label:"RAL 7035 (gris claro)"};
  {Value:"7016"; Label:"RAL 7016 (gris antracita)"};
  {Value:"9016"; Label:"RAL 9016 (blanco tráfico)"};
  {Value:"9005"; Label:"RAL 9005 (negro)"};
  {Value:"5010"; Label:"RAL 5010 (azul)"};
  {Value:"6005"; Label:"RAL 6005 (verde)"};
  {Value:"otro"; Label:"Otro"}
));;
ClearCollect(colOpcTipoVentilacion; Table(
  {Value:"natural"; Label:"Natural"};
  {Value:"forzada"; Label:"Forzada (ventilador)"};
  {Value:"sellado"; Label:"Sellado (IP alto)"};
  {Value:"climatizado"; Label:"Climatizado (aire acondicionado)"}
));;
ClearCollect(colOpcExpansionFutura; Table(
  {Value:"no"; Label:"Sin expansión"};
  {Value:"10"; Label:"10%"};
  {Value:"20"; Label:"20%"};
  {Value:"30"; Label:"30%"};
  {Value:"otro"; Label:"Otro"}
));;
ClearCollect(colOpcSiNo; Table(
  {Value:"true"; Label:"Sí"};
  {Value:"false"; Label:"No"}
));;
```

---

## Notas

- Las 18 colecciones **deben existir antes** de que la app cargue por primera vez, 
  de lo contrario los dropdowns mostrarán "Cargando..." indefinidamente.
- El bloque de `OnStart` se ejecuta una sola vez al abrir la app, así que es 
  el lugar perfecto para inicializar colecciones.
- Cada colección está separada por `;;` (doble punto y coma — separador de 
  instrucciones en Power Fx).
- Si necesitas agregar/quitar una opción más adelante, edita el `OnStart` 
  directamente en Studio (no necesitas tocar el YAML).

---

## Referencias

- **Construcción clic-por-clic:** [05-construir-canvas-captura.md](../dataverse/05-construir-canvas-captura.md)
- **YAML completo:** [06-yaml-completo-para-pegar.md](06-yaml-completo-para-pegar.md)
- **Fórmulas de captura:** [05-canvas-yaml-captura.md](05-canvas-yaml-captura.md)
