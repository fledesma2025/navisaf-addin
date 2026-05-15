# CLAUDE.md

Este archivo proporciona orientación a Claude Code (claude.ai/code) al trabajar con el código de este repositorio.

## Qué es este proyecto

Un **Add-In de Geotab** (plataforma de telemática de flotas) llamado "Analizador de Viajes" para la cuenta `fleet_colombia` de Navisaf. Es una **aplicación de un solo archivo** — el add-in completo vive en `analizador-viajes.html` (HTML + CSS + JavaScript, ~1600 líneas, sin sistema de build).

No hay `package.json`, ni bundler, ni suite de pruebas, ni paso de compilación. Desarrollar significa editar el archivo HTML directamente.

## Contexto de despliegue

Este archivo corre dentro de la plataforma web de Geotab como un Add-In embebido. Geotab inyecta un global `geotab` y carga el archivo en un iframe. El punto de entrada es el registro en la línea 280:

```js
geotab.addin['transito-geocercas'] = function() {
  return {
    initialize: function(api, state, cb) { ... },
    focus: function() {},
    blur: function() {}
  };
};
```

El objeto `api` (almacenado como `geotabApi`) es la única forma de obtener datos — expone `geotabApi.call(typeName, search, successCb, errorCb)` contra la base de datos de Geotab. Tipos clave utilizados:
- `Zone` — geocercas; tiene `.id`, `.name`, `.points[]` (polígono con `.x`/`.y` para lng/lat) y `.radius` (para zonas circulares)
- `Device` — vehículos; tiene `.id`, `.name`
- `LogRecord` — registros de posición GPS; tiene `.device.id`, `.dateTime`, `.latitude`, `.longitude`, `.speed` (en km/h)

## Flujo de datos principal

1. **Inicio** (`cargarDatosIniciales`): obtiene todas las Zones (hasta 5000) y Devices (hasta 2000); llena los desplegables de filtros.

2. **Consulta** (al hacer clic en "Consultar"): obtiene `LogRecord` para el rango de fechas (hasta 100.000 registros), opcionalmente filtrado por vehículo. Agrupa registros por vehículo y ordena los puntos de cada uno por `dateTime`.

3. **Detección de viajes** (`detectarViajes`): máquina de estados por vehículo (`buscando_salida` → `buscando_llegada`). Un viaje comienza cuando el vehículo sale de la zona origen y termina cuando entra a la zona destino. Usa:
   - Zonas circulares: distancia `haversineM` ≤ `zona.radius`
   - Zonas poligonales: ray-casting (`raycast`) — **importante**: los puntos de zona usan `.x` para longitud y `.y` para latitud (convención de Geotab, al revés del orden habitual lat/lng).

4. **Métricas** (`calcMetricas`): calcula por viaje la distancia (suma de Haversine), velocidad máx/promedio y duración. Clasificación de estado:
   - `velMaxKmh > 100` → `'velocidad'`
   - `duracionMin > 600` (>10 h) → `'demorado'`
   - en caso contrario → `'a tiempo'`

5. **Mapa** (`dibujarMapa`): Leaflet.js cargado dinámicamente desde el CDN de unpkg en el primer uso. Usa `window._lmap` como instancia singleton del mapa y `window._lmap_layers` para las capas del viaje activo. El color del trazado indica el estado: rojo (velocidad), naranja (demorado), azul oscuro (normal). Los puntos de exceso de velocidad (>100 km/h) reciben marcadores circulares rojos individuales.

6. **Ruta óptima** (`calcularRutaOptima`): llama a la API pública demo de OSRM (`router.project-osrm.org`) con el primer y último punto GPS del viaje. El resultado se cachea en `f.rutaOptima` y se dibuja como una polilínea verde punteada.

7. **Exportación**: CSV con BOM UTF-8 (para Excel) generado en el cliente mediante `Blob` + `URL.createObjectURL`.

## Quirk estructural crítico

El archivo contiene **múltiples sobreescrituras apiladas** de las mismas funciones (`mostrarLayout`, `setCargando`, `renderTabla`, `renderKPIs`, `seleccionarViaje`, `dibujarMapa`, `mostrarError`, `dbg`, etc.). Están apiladas desde la línea ~727 en cuatro bloques sucesivos. **La última definición es la que aplica.** Las implementaciones canónicas y activas son las últimas (aproximadamente líneas 1303–1599). Al modificar cualquiera de estas funciones, editar solo la última ocurrencia y considerar consolidar o eliminar las versiones anteriores inactivas.

## Constantes de marca/estilo

- Azul marino principal: `#0d3c6e`
- Verde acento: `#00a884`
- Rojo exceso velocidad: `#d62c2c`
- Naranja demorado: `#d68a00`

## Cómo probar localmente

Abrir `analizador-viajes.html` directamente en un navegador para inspeccionar HTML/CSS. El JavaScript no funcionará fuera de Geotab (el objeto `geotabApi` no existe), pero se puede simular para desarrollo de UI:

```js
// Stub mínimo para desarrollo local
window.geotab = { addin: {} };
// Luego llamar initialize manualmente con un objeto api falso
```

Para probar con datos reales, el archivo debe cargarse a través del sistema de Add-In de Geotab (ya sea el Marketplace de Geotab o una configuración de Add-In personalizada apuntando a la URL donde esté alojado el archivo).
