# Módulo servicios_media

**Fecha:** 2026-06-28  
**Repo:** `ave-api` — `https://github.com/EnuarRosales/ave-api`  
**Estado:** ✅ Completo (Fases 1, 2, 4 y 5)

---

## Contexto

Antes de este módulo, cada servicio tenía un único campo `imagen_url` en la tabla `servicios`. El objetivo es soportar múltiples archivos por producto (imágenes, videos, PDFs) con gestión completa desde Appsmith y entrega al usuario final en WhatsApp.

---

## Fase 1 — Base de datos

### Tabla `servicios_media`

```sql
CREATE TABLE servicios_media (
    id          SERIAL PRIMARY KEY,
    servicio_id INTEGER NOT NULL REFERENCES servicios(id) ON DELETE CASCADE,
    empresa_id  INTEGER NOT NULL REFERENCES empresas(id) ON DELETE CASCADE,
    tipo        VARCHAR(10) NOT NULL CHECK (tipo IN ('imagen', 'video', 'pdf')),
    url         TEXT NOT NULL,
    orden       INTEGER DEFAULT 1,
    descripcion VARCHAR(255),
    activo      BOOLEAN DEFAULT true,
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_servicios_media_servicio ON servicios_media(servicio_id);
CREATE INDEX idx_servicios_media_empresa  ON servicios_media(empresa_id);
```

**Notas:**
- `created_at` usa `NOW()` sin timezone — consistente con el resto de tablas de AVE
- `ON DELETE CASCADE` — si se elimina un servicio o empresa, se eliminan sus registros de la tabla
- El archivo físico en disco **no** se elimina automáticamente en cascade — solo cuando se llama al endpoint `DELETE /:id`

---

## Fase 2 — Endpoints API

**Base URL:** `https://api.identechnology.co/api/v1/media`

### Listar archivos de un servicio
```
GET /api/v1/media/:servicio_id
```
Devuelve array de archivos activos ordenados por `orden ASC`.

### Subir imagen
```
POST /api/v1/media/upload/imagen/:empresa_id/:servicio_id
Content-Type: multipart/form-data — Campo: archivo
```
Formatos: JPG, PNG, WEBP — Límite: 5MB

### Subir video
```
POST /api/v1/media/upload/video/:empresa_id/:servicio_id
Content-Type: multipart/form-data — Campo: archivo
```
Formatos: MP4, MOV, AVI — Límite: 100MB

### Subir PDF
```
POST /api/v1/media/upload/pdf/:empresa_id/:servicio_id
Content-Type: multipart/form-data — Campo: archivo
```
Formato: PDF — Límite: 10MB

### Actualizar metadata
```
PUT /api/v1/media/:id
```
Campos opcionales: `orden`, `descripcion`, `activo`

### Eliminar archivo
```
DELETE /api/v1/media/:id
```
Elimina el registro de la BD **y** el archivo físico del disco.

---

## Almacenamiento VPS

```
/var/www/ave/media/
└── {empresa_id}/
    ├── imagenes/
    ├── videos/
    └── pdfs/
```

**URL pública:** `https://api.identechnology.co/ave/media/{empresa_id}/{tipo}/{filename}`

Servido como static por Express:
```js
app.use('/ave/media', express.static('/var/www/ave/media'));
```

### Compatibilidad legacy
- `POST /api/v1/media/upload/:empresa_id/:servicio_id` — sube al directorio legacy
- `DELETE /api/v1/media/:empresa_id/:filename` — elimina del directorio legacy

---

## Estructura de archivos API

```
src/api/v1/media/
├── media.routes.js
├── media.controller.js
└── media.service.js
```

**Variables de entorno agregadas al `docker-compose.yml`:**
```env
DATABASE_URL=postgresql://postgres:****@n8n-postgres-1:5432/postgres
```

---

## Fase 4 — n8n

### Cambios en el flujo `Flujo_principal_agencIA`

| Nodo anterior | Nodo nuevo | Cambio |
|--------------|------------|--------|
| `buscar_imagen_servicio` | `buscar_media_servicio` | Query a `servicios_media` |
| `Code_detectar_imagen` | `Code_detectar_media` | Detecta URLs de imagen, video y PDF |
| `If_tiene_imagen` | `If_tiene_media` | Sin cambio funcional |
| `Descargar_imagen` | `Descargar_media` | Lee URL desde `SplitInBatches_media` |
| `Enviar_imagen_Chatwoot` | `Enviar_media_Chatwoot` | Envía cualquier tipo de archivo |
| — | `Code_split_archivos` | Convierte array de archivos en items separados |
| — | `SplitInBatches_media` | Loop — envía archivos uno por uno |

### Flujo de media

```
Code_detectar_media ──→ Responder_Mensaje (texto al cliente)
                    └─→ If_tiene_media
                              ├── true → Code_split_archivos → SplitInBatches_media
                              │                                   ├── loop → Descargar_media → Enviar_media_Chatwoot → SplitInBatches_media
                              │                                   └── done → get_estado_conversacion
                              └── false → get_estado_conversacion
```

### Query `buscar_media_servicio`

```sql
SELECT sm.id, sm.tipo, sm.url, sm.orden, sm.descripcion
FROM servicios_media sm
INNER JOIN servicios s ON s.id = sm.servicio_id
WHERE sm.empresa_id = {{ $('PG_get_empresa').item.json.id }}
AND s.empresa_id = {{ $('PG_get_empresa').item.json.id }}
AND s.activo = true
AND sm.activo = true
AND LOWER(s.nombre) ILIKE '%{{ $fromAI("servicio", "Nombre o parte del nombre del servicio", "string") }}%'
ORDER BY sm.tipo, sm.orden ASC
LIMIT 10;
```

### Code_detectar_media

```javascript
const output = $input.item.json.output || '';

const urlRegex = /(https?:\/\/[^\s]+\.(?:png|jpg|jpeg|webp|gif|mp4|mov|avi|pdf))/gi;
const matches = [...output.matchAll(urlRegex)].map(m => m[1]);

if (matches.length > 0) {
  let textoLimpio = output;
  matches.forEach(url => { textoLimpio = textoLimpio.replace(url, ''); });
  textoLimpio = textoLimpio.replace(/\n\n+/g, '\n').trim();

  const archivos = matches.map((url, i) => {
    const ext = url.split('.').pop().toLowerCase();
    let tipo = 'imagen';
    if (['mp4', 'mov', 'avi'].includes(ext)) tipo = 'video';
    if (ext === 'pdf') tipo = 'pdf';
    return { url, tipo, orden: i + 1 };
  });

  return {
    json: { output: textoLimpio, tiene_media: true, archivos, total: archivos.length }
  };
} else {
  return {
    json: { output, tiene_media: false, archivos: [], total: 0 }
  };
}
```

### Code_split_archivos

```javascript
const archivos = $input.first().json.archivos || [];

return archivos.map(archivo => ({
  json: { url: archivo.url, tipo: archivo.tipo, orden: archivo.orden }
}));
```

### Sección 9 agregada al prompt del agente

```
# 9. MANEJO DE MEDIOS

Cuando un cliente solicite ver fotos, imagenes, videos o informacion de un servicio,
usa buscar_media_servicio para obtener los archivos disponibles.

Reglas:
- Nunca listes los archivos como "Imagen 1, Imagen 2, Video".
- Incluye las URLs exactamente como las devuelve buscar_media_servicio en tu respuesta.
- No describes ni numeras los archivos. Solo incluye las URLs en el texto.
- Acompana las URLs con un mensaje natural y breve.

buscar_media_servicio → Cuando el cliente pida ver fotos, imagenes, videos o materiales de un servicio.
```

### Fix — `PG_actualizar_conversacion`

El nodo fallaba con "Multiple matches found" cuando el loop procesaba múltiples archivos. Causa: `.item` es ambiguo con múltiples items en contexto. Fix: cambiar a `.first()`.

```sql
-- Antes
$('GET_etiquetas_actuales').item.json.payload
$('Contexto').item.json.account_id
$('Contexto').item.json.conversation_id

-- Después
$('GET_etiquetas_actuales').first().json.payload
$('Contexto').first().json.account_id
$('Contexto').first().json.conversation_id
```

---

## Fase 5 — Appsmith

### Queries creadas (datasource AVE_API)

| Query | Método | URL |
|-------|--------|-----|
| `get_media_servicio` | GET | `/api/v1/media/{{tbl_servicios.selectedRow.id}}` |
| `upload_imagen_media` | POST | `/api/v1/media/upload/imagen/{{tbl_servicios.selectedRow.empresa_id}}/{{tbl_servicios.selectedRow.id}}` |
| `upload_video_media` | POST | `/api/v1/media/upload/video/{{tbl_servicios.selectedRow.empresa_id}}/{{tbl_servicios.selectedRow.id}}` |
| `upload_pdf_media` | POST | `/api/v1/media/upload/pdf/{{tbl_servicios.selectedRow.empresa_id}}/{{tbl_servicios.selectedRow.id}}` |
| `delete_media` | DELETE | `/api/v1/media/{{tbl_media.triggeredRow.id}}` |

**Nota:** Todas las queries de upload usan `MULTIPART_FORM_DATA` con key `archivo`, Type `File`, Data format `Binary`.

### Widgets en `container_media`

| Widget | Tipo | Configuración |
|--------|------|---------------|
| `txt_titulo_media` | Text | `{{"Medios de: " + tbl_servicios.selectedRow.nombre}}` |
| `tbl_media` | Table | Data: `{{get_media_servicio.data}}` |
| `fp_imagen` | FilePicker | onFilesSelected: `upload_imagen_media` → onSuccess: `get_media_servicio` |
| `fp_video` | FilePicker | onFilesSelected: `upload_video_media` → onSuccess: `get_media_servicio` |
| `fp_pdf` | FilePicker | onFilesSelected: `upload_pdf_media` → onSuccess: `get_media_servicio` |

### Columna Eliminar en `tbl_media`

- Type: Button — Text: `Eliminar` — Color: Danger
- onClick: `delete_media` → onSuccess: `get_media_servicio`

### Configuraciones adicionales

- `tbl_servicios` onRowSelected → `get_media_servicio.run()`
- Columnas ocultas en `tbl_media`: `id`, `servicio_id`, `empresa_id`, `activo`, `created_at`
- Botón legacy "Subir imagen del servicio" eliminado

---

## Comportamiento del eliminado

1. `DELETE /api/v1/media/:id` se ejecuta desde Appsmith
2. API borra el registro de `servicios_media` en Postgres
3. API borra el archivo físico del disco en `/var/www/ave/media/`
4. `get_media_servicio` se refresca automáticamente

El almacenamiento en VPS se libera inmediatamente al eliminar.

---

## Commits

```
feat: módulo servicios_media — endpoints GET/POST/PUT/DELETE + soporte imagen/video/pdf
```

## Archivos de respaldo

- `Flujo_principal_agencIA_v2.json` — flujo con todas las correcciones aplicadas
