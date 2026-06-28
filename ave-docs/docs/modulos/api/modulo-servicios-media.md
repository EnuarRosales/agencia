# Módulo servicios_media

**Fecha:** 2026-06-28  
**Repo:** `ave-api` — `https://github.com/EnuarRosales/ave-api`  
**Estado:** ✅ Fases 1 y 2 completadas — Fases 4 y 5 pendientes

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
- `ON DELETE CASCADE` — si se elimina un servicio o empresa, se eliminan sus archivos de la tabla
- El archivo físico en disco **no** se elimina automáticamente en cascade — solo se elimina cuando se llama al endpoint `DELETE /:id`

---

## Fase 2 — Endpoints API

**Base URL:** `https://api.identechnology.co/api/v1/media`

### Listar archivos de un servicio
```
GET /api/v1/media/:servicio_id
```
Devuelve array de archivos activos ordenados por `orden ASC`.

**Response:**
```json
[
  {
    "id": 1,
    "servicio_id": 5,
    "empresa_id": 8,
    "tipo": "imagen",
    "url": "https://api.identechnology.co/ave/media/8/imagenes/5_1234567890.jpg",
    "orden": 1,
    "descripcion": null,
    "activo": true,
    "created_at": "2026-06-28T..."
  }
]
```

### Subir imagen
```
POST /api/v1/media/upload/imagen/:empresa_id/:servicio_id
Content-Type: multipart/form-data
Campo: archivo
```
- Formatos: JPG, PNG, WEBP
- Límite: 5MB

### Subir video
```
POST /api/v1/media/upload/video/:empresa_id/:servicio_id
Content-Type: multipart/form-data
Campo: archivo
```
- Formatos: MP4, MOV, AVI
- Límite: 100MB

### Subir PDF
```
POST /api/v1/media/upload/pdf/:empresa_id/:servicio_id
Content-Type: multipart/form-data
Campo: archivo
```
- Formato: PDF
- Límite: 10MB

### Actualizar metadata
```
PUT /api/v1/media/:id
Content-Type: application/json
```
```json
{
  "orden": 2,
  "descripcion": "Vista frontal del producto",
  "activo": true
}
```
Todos los campos son opcionales — solo actualiza los que se envíen.

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

Servido como static por Express en `src/app.js`:
```js
app.use('/ave/media', express.static('/var/www/ave/media'));
```

### Compatibilidad legacy
Las rutas antiguas de Appsmith siguen funcionando sin cambios:
- `POST /api/v1/media/upload/:empresa_id/:servicio_id` — sube imagen al directorio legacy `/var/www/ave/imagenes/servicios/`
- `DELETE /api/v1/media/:empresa_id/:filename` — elimina imagen del directorio legacy

---

## Estructura de archivos

```
src/api/v1/media/
├── media.routes.js      ← rutas nuevas + legacy
├── media.controller.js  ← listar, subirImagen, subirVideo, subirPdf, actualizar, eliminar + legacy
└── media.service.js     ← multer configs, buildMediaUrl, removeMediaFile + legacy
```

**Conexión a Postgres:** `src/config/database.js` — Pool con `DATABASE_URL` del environment.

---

## Variables de entorno

```env
DATABASE_URL=postgresql://postgres:****@n8n-postgres-1:5432/postgres
```
Agregada al `docker-compose.yml`. El contenedor `ave-api` y `n8n-postgres-1` comparten `traefik_network`, por eso se usa el nombre del contenedor como host.

---

## Commits

```
feat: módulo servicios_media — endpoints GET/POST/PUT/DELETE + soporte imagen/video/pdf
```

---

## Pendiente

### Fase 4 — n8n
- [ ] Renombrar tool `buscar_imagen_servicio` → `buscar_media_servicio`
- [ ] Query actualizado: leer de `servicios_media` en lugar de `servicios.imagen_url`
- [ ] Detectar `tipo` y enviar mensaje correcto a Chatwoot:
  - `imagen` → attachment tipo image (flujo actual)
  - `video` → attachment tipo video
  - `pdf` → attachment tipo document
- [ ] Soportar múltiples archivos por servicio (enviar todos los activos en orden)
- [ ] Actualizar prompt del agente para mencionar que puede compartir fotos, videos y PDFs

### Fase 5 — Appsmith
- [ ] Sección de medios en página Info Negocio
- [ ] Por producto: listar archivos con tipo, orden, descripción
- [ ] Botones: subir imagen / subir video / subir PDF
- [ ] Reordenar archivos (editar campo `orden`)
- [ ] Eliminar archivo individual
