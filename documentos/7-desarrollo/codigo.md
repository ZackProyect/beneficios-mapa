# Documentación de código — Mapa de Beneficios Bancarios

## Archivo principal
`beneficios-mapa/index.html` — aplicación completa en un solo archivo (~1300 líneas)

---

## Estructura del archivo

```
index.html
├── <head>
│   ├── Meta tags (viewport, PWA, theme-color)
│   ├── Link manifest.json
│   ├── CSS Leaflet + MarkerCluster (CDN)
│   ├── Google Fonts (Nunito)
│   └── <style> CSS inline (~430 líneas)
│       ├── Variables CSS (:root → --primary, --primary-dark, etc.)
│       ├── Pantalla selección de banco (#bank-select-screen)
│       ├── Loading screen
│       ├── Mapa (#map)
│       ├── Header flotante (.mobile-header)
│       ├── Buscador flotante (.search-float)
│       ├── Panel de filtros (.filter-sheet)
│       ├── Panel de marcador (#markerPanel)
│       ├── Botón GPS (#btnMyLocation)
│       ├── Toast (#toast)
│       └── Carrusel de tarjetas (#cardList)
│
├── <body> HTML (~120 líneas)
│   ├── #bank-select-screen (3 tarjetas: BCI, Santander, Banco de Chile)
│   ├── #loading-screen
│   ├── #map
│   ├── .mobile-header (logo + btn cambiar banco + btn filtros)
│   ├── .search-float (input buscador + btn GPS pin)
│   ├── .area-btn (buscar en esta área)
│   ├── .addr-float (búsqueda de dirección)
│   ├── .filter-overlay + .filter-sheet (panel filtros)
│   ├── #markerPanel (panel deslizante de detalle)
│   ├── #btnMyLocation
│   ├── #toast
│   ├── #errBanner
│   ├── #carouselToggle
│   └── #cardList
│
└── <script> JS inline (~870 líneas)
    ├── Constantes y estado global
    ├── BANK_CONFIGS
    ├── Helpers (escAttr, stripHtml, getBadgeLabel, getCategoryEmoji, etc.)
    ├── Adaptadores de bancos (fetch + map por banco)
    ├── Selección de banco y theming
    ├── Inicialización del mapa (Leaflet)
    ├── Panel de marcador (showMarkerPanel, closeMarkerPanel)
    ├── Icono de marcador (createIcon)
    ├── Geocodificación OSM (fetchSucursalesOSM, fetchSucursalesNominatim)
    ├── Búsqueda de dirección (searchAddress, selectAddress)
    ├── Geolocalización usuario (showUserDot, requestMyLocation)
    ├── Filtros y render (filterBeneficios, render)
    └── initApp (carga principal + geocodificación en background)
```

---

## Variables globales de estado

```javascript
let activeBank      = null;    // Configuración del banco seleccionado
let beneficios      = [];      // Array de todos los beneficios cargados
let map             = null;    // Instancia del mapa Leaflet
let markerCluster   = null;    // Grupo de marcadores con clustering
let activeFilters   = { buscar:'', categoria:'', dias:[], modalidad:[], tipo:[], distancia:null };
let selectedId      = null;    // ID del beneficio seleccionado (click en tarjeta)
let userLat         = null;    // Latitud del usuario
let userLng         = null;    // Longitud del usuario
let userDotMarker   = null;    // Marcador de posición del usuario en el mapa
let sucCache        = {};      // Cache en memoria de geocodificaciones
```

---

## Funciones principales

### Configuración de bancos
```javascript
BANK_CONFIGS = { bci, santander, bancochile }
// Cada entrada: name, primary, primaryDark, primaryLight, badgeDcto, apiHeaders, fetchAll, logoHtml
```

### Adaptadores de bancos
| Función | Descripción |
|---|---|
| `fetchBCI(onProgress)` | Fetch paginado de BCI, paralelo desde página 2 |
| `mapBCIOffer(o)` | Mapea objeto de API BCI al modelo interno |
| `fetchSantander(onProgress)` | Fetch único sin paginación |
| `mapSantanderOffer(p)` | Mapea objeto Santander al modelo interno |
| `fetchBancoChile(onProgress)` | Fetch paginado de Banco de Chile |
| `mapBancoChileOffer(o)` | Mapea objeto Banco de Chile, parsea Sucursales HTML |
| `parseSucursalesBCH(html)` | Extrae `[{street, city}]` de HTML de sucursales |

### Theming
```javascript
selectBank(bankId)   // Aplica tema + oculta selección + inicia carga
applyTheme(bank)     // Setea CSS custom properties en :root
cambiarBanco()       // Vuelve a pantalla de selección, limpia estado
```

### Mapa y marcadores
```javascript
createIcon(b)          // Crea L.divIcon con logo o emoji fallback
showMarkerPanel(b, s)  // Abre panel inferior con datos del comercio
closeMarkerPanel()     // Cierra el panel
showUserDot(lat, lng)  // Agrega/actualiza marcador de posición del usuario
requestMyLocation()    // Solicita GPS con feedback toast
```

### Geocodificación
```javascript
fetchSucursalesOSM(brandName, bounds)        // Photon → Nominatim fallback
fetchSucursalesNominatim(brandName, bounds)  // Nominatim directo con countrycodes=cl
mergeSucursales(ben, locs)                   // Agrega locs a ben.sucursales sin duplicar
inChile(lat, lng)                            // Valida que coordenadas estén en Chile
loadCache(bankId) / saveCache(bankId)        // localStorage con TTL 24h
```

### Filtros y render
```javascript
filterBeneficios()   // Aplica activeFilters sobre beneficios[], retorna array filtrado
render()             // Actualiza #cardList + markers en markerCluster
```

### initApp(bankId)
Flujo principal de carga:
1. Muestra loading screen
2. Llama a `fetchAll(onProgress)` del banco
3. Filtra beneficios Online
4. Actualiza header con fecha y conteo
5. Muestra UI principal
6. Llama a `render()`
7. Inicia geocodificación en background:
   - BancoChile: geocodifica por dirección exacta (1 req/seg con delay)
   - BCI/Santander: geocodifica por brand_name con deduplicación

---

## Constantes importantes

```javascript
CHILE_BOUNDS = { n:-17.5, s:-55.9, w:-75.7, e:-66.0 }
CACHE_KEY_PREFIX = 'beneficios_osm_v3_'   // Cambiar a v4_ para invalidar cache
CACHE_TTL = 24 * 60 * 60 * 1000           // 24 horas en ms

// Días BCI
DAYS_MAP = { MON:'Lunes', TUE:'Martes', WED:'Miércoles', ... }

// Categorías y días Santander
SANT_CAT_MAP = { 'cat-sabores':'Restaurantes', 'cat-descuentos':'Descuentos', ... }
SANT_DAY_MAP = { 'todos-los-dias':'Todos los días', 'miercoles':'Miércoles', ... }

// Categorías y días Banco de Chile
BCH_CAT_MAP = { 'sabores':'Restaurantes', 'descuentos':'Descuentos', ... }
BCH_DAY_MAP = { 'todos-los-dias':'Todos los días', 'miercoles':'Miércoles', ... }
```

---

## Reglas de negocio implementadas en código

1. **Sin Online**: `filterBeneficios()` descarta siempre `b.modalidad === 'Online'`
2. **Solo Chile**: `inChile()` en `mergeSucursales()` + `countrycodes=cl` en Nominatim + `countrycode=cl` en Photon + `bbox` en Photon
3. **Logo en marcadores**: `createIcon()` usa `b.logo || b.imagen` (logo = imagen4 para BCI)
4. **Panel persistente**: marcadores usan `marker.on('click')` en vez de `bindPopup()` para evitar que el re-render destruya el popup
5. **Rate limit Nominatim**: geocodificación BancoChile usa `bchDelay += 1100ms` por request

---

## Cómo hacer un cambio de código

### 1. Editar
- Archivo de trabajo: `C:\Users\crist\Downloads\beneficios-bci.html`
- Nunca editar directamente `beneficios-mapa/index.html`

### 2. Probar localmente
```powershell
# Iniciar servidor HTTP en puerto 8080
$listener = [System.Net.HttpListener]::new()
$listener.Prefixes.Add("http://localhost:8080/")
$listener.Start()
# Abrir http://localhost:8080/beneficios-bci.html en el navegador
```

### 3. Copiar a deploy
```powershell
Copy-Item "C:\Users\crist\Downloads\beneficios-bci.html" `
          "C:\Users\crist\Downloads\beneficios-mapa\index.html" -Force
```

### 4. Subir a GitHub y verificar
```powershell
# Ver procesos.md de 5-operaciones para el script completo de deploy
# Verificar después:
$r = Invoke-WebRequest -Uri "https://zackproyect.github.io/beneficios-mapa/" -UseBasicParsing
Write-Output "HTTP $($r.StatusCode)"
```

---

## Bugs conocidos y pendientes

| Bug/Mejora | Descripción | Prioridad |
|---|---|---|
| Rate limit BancoChile | Si hay muchas sucursales, la geocodificación puede tardar varios minutos | Media |
| Cache BancoChile | Las direcciones geocodificadas de BancoChile se cachean con clave `bch:street:city` pero no se invalidan si cambia la oferta | Baja |
| Token GitHub expuesto | El token usado en sesión de setup quedó en el historial del chat | Revocar en github.com/settings/tokens |
| API key BCI en HTML | La key `fa981752762743668413b68821a43840` está en el código fuente público | Riesgo aceptado por ahora |
