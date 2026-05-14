# Procesos operacionales — Mapa de Beneficios Bancarios

## Infraestructura actual
| Recurso | Valor |
|---|---|
| Repositorio | https://github.com/ZackProyect/beneficios-mapa |
| URL producción | https://zackproyect.github.io/beneficios-mapa/ |
| Rama principal | `main` |
| Hosting | GitHub Pages (activado en Settings → Pages → main / root) |
| Archivos en repo | `index.html`, `manifest.json`, `icon-192.png`, `icon-512.png` |

---

## Proceso de despliegue

### Pasos para actualizar el sitio
1. Editar `C:\Users\crist\Downloads\beneficios-bci.html` (archivo de trabajo)
2. Copiar a la carpeta de deploy:
   ```powershell
   Copy-Item "C:\Users\crist\Downloads\beneficios-bci.html" "C:\Users\crist\Downloads\beneficios-mapa\index.html" -Force
   ```
3. Subir a GitHub vía API (requiere token con scope `repo`):
   ```powershell
   $token = "<TOKEN>"
   $headers = @{ Authorization = "Bearer $token"; Accept = "application/vnd.github+json"; "X-GitHub-Api-Version" = "2022-11-28" }
   $fileMeta = Invoke-RestMethod -Uri "https://api.github.com/repos/ZackProyect/beneficios-mapa/contents/index.html" -Headers $headers
   $b64 = [Convert]::ToBase64String([System.IO.File]::ReadAllBytes("C:\Users\crist\Downloads\beneficios-mapa\index.html"))
   $body = @{ message = "Descripción del cambio"; content = $b64; sha = $fileMeta.sha } | ConvertTo-Json -Depth 3
   Invoke-RestMethod -Uri "https://api.github.com/repos/ZackProyect/beneficios-mapa/contents/index.html" -Method PUT -Headers $headers -Body $body -ContentType "application/json"
   ```
4. GitHub Pages actualiza automáticamente en 1-2 minutos

### Verificar que el deploy esté activo
```powershell
$r = Invoke-WebRequest -Uri "https://zackproyect.github.io/beneficios-mapa/" -UseBasicParsing
Write-Output "HTTP $($r.StatusCode) — tamaño: $($r.Content.Length) bytes"
```

---

## Gestión de tokens GitHub
- Los tokens clásicos (`ghp_...`) tienen scope `repo` para leer y escribir archivos
- **IMPORTANTE**: Los tokens no deben quedar en archivos del repositorio
- Generar en: https://github.com/settings/tokens/new
- Configuración recomendada: expiración 7-30 días, scope solo `repo`
- Revocar tokens usados en: https://github.com/settings/tokens

### Tokens usados (revocar después de uso)
| Token (primeros 8 chars) | Uso | Estado |
|---|---|---|
| `ghp_OOiKb...` | Crear repo + subir archivos | Debe revocarse |

---

## Clave de API BCI
- **Header**: `ocp-apim-subscription-key`
- **Valor**: `fa981752762743668413b68821a43840`
- **Origen**: Extraído del tráfico de red de `bci.cl/beneficios`
- **Estado**: Activo (mayo 2026)
- **Riesgo**: Key pública embebida en el HTML. Si el banco la rota, hay que actualizarla.

---

## Monitoreo
Actualmente no hay monitoreo automatizado. Pendiente:
- [ ] Script PowerShell que verifica cada banco y alerta si la API falla
- [ ] Verificación semanal de que GitHub Pages responde HTTP 200
- [ ] Revisión mensual del cache de geocodificación (el prefijo v3 debe actualizarse si cambia la lógica)

---

## Cache de geocodificación
- **Prefijo localStorage**: `beneficios_osm_v3_`
- **TTL**: 24 horas
- **Invalidar manualmente**: Cambiar el prefijo a `v4_` en el código
- **¿Cuándo invalidar?**: Cuando cambia la lógica de filtrado de Chile o se detectan coordenadas incorrectas

---

## Proceso para agregar un nuevo banco
1. Obtener el endpoint de la API desde el Network tab del navegador
2. Verificar CORS (`access-control-allow-origin: *`)
3. Documentar estructura JSON en `4-tecnologia/arquitectura.md`
4. Implementar `fetchNuevoBanco()` y `mapNuevoBancoOffer()` en `index.html`
5. Agregar a `BANK_CONFIGS` con colores corporativos
6. Agregar tarjeta en pantalla de selección
7. Probar localmente con servidor HTTP (no file://)
8. Desplegar siguiendo el proceso anterior
