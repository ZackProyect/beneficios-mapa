# Arquitectura técnica — Mapa de Beneficios Bancarios

## Stack tecnológico
| Capa | Tecnología | Razón |
|---|---|---|
| Frontend | HTML5 + CSS3 + JavaScript vanilla | Sin framework, máxima compatibilidad móvil, cero build |
| Mapa | Leaflet.js 1.9.4 | Open source, liviano, sin límites de uso |
| Clustering | Leaflet.MarkerCluster 1.5.3 | Agrupa marcadores automáticamente |
| Tipografía | Google Fonts (Nunito) | Legible en móvil |
| Geocodificación | Photon (Komoot) + Nominatim (OSM) | Gratuitos, sin API key |
| Cache | localStorage (24h TTL) | Evita geocodificaciones repetidas |
| Hosting | GitHub Pages | HTTPS gratuito, necesario para GPS en móvil |

## Archivos del proyecto
```
beneficios-mapa/
├── index.html      ← Aplicación completa (todo en un archivo)
├── manifest.json   ← PWA manifest (para instalación en móvil)
├── icon-192.png    ← Ícono de app 192x192
└── icon-512.png    ← Ícono de app 512x512
```

---

## APIs de bancos

### BCI
- **Endpoint**: `https://api.bciplus.cl/bff-loyalty-beneficios/v1/offers`
- **Auth**: Header `ocp-apim-subscription-key: fa981752762743668413b68821a43840`
- **CORS**: `access-control-allow-origin: *`
- **Paginación**: `?itemsPorPagina=100&pagina=N`
- **Campos clave**:
  - `nombre`, `subtitulo`, `categorias[].titulo`, `tags[].nombre`
  - `scheduling.dayRecurrence[]`, `scheduling.recurrenceLabel`
  - `imagenes.imagen1` (banner), `imagenes.imagen4` (logo de tienda → usado en marcadores)
  - `beneficio.discount.porcentajeDescuento`, `deal.discount.percentage`
  - `fechaTermino`, `link`, `modalidad`

### Santander
- **Endpoint**: `https://banco.santander.cl/beneficios/promociones.json`
- **Params**: `?per_page=9999&tags=home-disfrutadores&custom_fields=true&order_by=updated_at&desc=true&hash=151`
- **Auth**: Ninguna (CORS `*`, pero protegido por Akamai — requiere haber visitado bancosantander.cl antes)
- **Paginación**: Sin paginación (devuelve todo)
- **Campos clave**:
  - `title`, `tags[]`, `covers[]`, `latitude`, `longitude`, `location_street`
  - `custom_fields['Bajada externa'].value`, `custom_fields['Vigencia'].value`
  - `custom_fields['Sitio web beneficio'].value`

### Banco de Chile
- **Endpoint**: `https://sitiospublicos.bancochile.cl/api/content/spaces/personas/types/beneficios/entries`
- **Params**: `?page=N&per_page=100`
- **Auth**: Ninguna (CORS `*`)
- **Paginación**: `meta.total_pages` en la respuesta
- **Campos clave**:
  - `meta.name`, `meta.tags[]`, `meta.category` (formato: `beneficios/sabores/40-de-descuento-visa`)
  - `meta.unpublish_at` (fecha de expiración)
  - `fields.Titulo`, `fields.Logo.url`, `fields.Portada.url`
  - `fields['Tipo Beneficio']` (ej: "40%; dto.")
  - `fields.Sucursales` (HTML con formato: `<li>VACIO; Calle #123; Región; Ciudad; VACIO</li>`)
  - `fields.Extracto`, `fields.Descripcion` (HTML)
  - `fields.Vigencia`, `fields['Sitio web']`

---

## Pipeline de geocodificación

```
Por banco:
├── BCI / Santander
│   └── fetchSucursalesOSM(brand_name)
│       ├── 1. Photon (Komoot): ?q=nombre&bbox=-75.7,-55.9,-66.0,-17.5&osm_tag=...
│       │   └── Filtra: countrycode=cl + bounding box Chile + nombre match
│       └── 2. Nominatim (fallback): ?countrycodes=cl&q=nombre+Santiago
│
└── Banco de Chile
    └── parseSucursalesBCH(html) → [{street, city}]
        └── Por cada dirección:
            └── Nominatim: ?countrycodes=cl&q=calle, ciudad, Chile
                └── Rate limit: 1 request/segundo (bchDelay += 1100ms)

Cache:
- Prefijo: 'beneficios_osm_v3_'
- TTL: 24 horas
- Almacenamiento: localStorage
- Clave BCI/Santander: brand_name (o brand_name:bounds)
- Clave BancoChile: 'bch:street:city'
```

### Restricción geográfica Chile
- Photon: parámetro `bbox=-75.7,-55.9,-66.0,-17.5`
- Photon post-filter: `countrycode === 'cl'`
- Nominatim: `countrycodes=cl`
- `mergeSucursales()`: valida `inChile(lat, lng)` antes de agregar
- Chile bounds: `{n:-17.5, s:-55.9, w:-75.7, e:-66.0}`

---

## Modelo de datos interno (beneficio)

```javascript
{
  id: String,              // UUID único del beneficio
  nombre: String,          // Nombre del comercio
  brand_name: String,      // Nombre para agrupar sucursales en geocodificación
  subtitulo: String,       // Descripción corta
  icono: String,           // Emoji de categoría (fallback)
  categoria: String,       // Categoría principal
  categorias: [String],    // Lista de categorías
  tagNames: [String],      // Tags originales de la API
  descuento: String,       // "30%" o "2x1" o "CASHBACK"
  tipo: String,            // 'DISCOUNT' | 'CASHBACK' | 'PROMO' | 'BENEFIT'
  descripcion: String,     // Texto descriptivo
  dias: [String],          // ["Miércoles", "Sábado"]
  recurrenceLabel: String, // "Todos los días"
  modalidad: String,       // 'Presencial' | 'Online' | 'Presencial y Online' | 'General'
  vigencia: String|null,   // Fecha de expiración formateada
  imagen: String|null,     // URL imagen principal (banner)
  logo: String|null,       // URL logo del comercio (usado en marcadores del mapa)
  link: String|null,       // URL del sitio web
  sucursales: [{           // Ubicaciones geocodificadas
    lat: Number,
    lng: Number,
    nombre: String,
    direccion: String
  }],
  rawAddresses: [{         // Solo Banco de Chile: direcciones a geocodificar
    street: String,
    city: String
  }]
}
```

---

## Theming multi-banco
Usando CSS custom properties en `:root`:
```css
--primary       → color principal del banco
--primary-dark  → variante oscura (headers, gradientes)
--primary-light → variante clara (fondos de chips)
--badge-dcto-bg → color del badge de descuento
```
Se sobrescriben vía `document.documentElement.style.setProperty()` al seleccionar banco.

---

## Modelo de datos futuro (si se agrega backend)

```sql
-- Beneficios (sincronizados desde APIs de bancos)
CREATE TABLE beneficios (
  id          TEXT PRIMARY KEY,
  banco_id    TEXT,
  nombre      TEXT,
  categoria   TEXT,
  descuento   TEXT,
  modalidad   TEXT,
  vigencia    DATE,
  imagen_url  TEXT,
  logo_url    TEXT,
  link        TEXT,
  dias        JSONB,
  raw_data    JSONB,
  updated_at  TIMESTAMP
);

-- Sucursales geocodificadas
CREATE TABLE sucursales (
  id           SERIAL PRIMARY KEY,
  beneficio_id TEXT REFERENCES beneficios(id),
  nombre       TEXT,
  direccion    TEXT,
  lat          DECIMAL(9,6),
  lng          DECIMAL(9,6),
  geocodificado_en TIMESTAMP
);

-- Caché de geocodificación
CREATE TABLE geocache (
  query      TEXT PRIMARY KEY,
  lat        DECIMAL(9,6),
  lng        DECIMAL(9,6),
  fuente     TEXT,  -- 'photon' | 'nominatim'
  creado_en  TIMESTAMP
);
```
