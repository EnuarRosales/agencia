# Refactorización API — Routes / Controller / Service

**Fecha:** 2026-06-28  
**Repo:** `ave-api` — `https://github.com/EnuarRosales/ave-api`  
**Estado:** ✅ Completado y en producción

---

## Contexto

La API corría como un único archivo `index.js` monolítico (~480 líneas) sin separación de responsabilidades, sin versionado de rutas y sin estructura escalable. Se refactorizó completa a arquitectura Routes → Controller → Service con versionado `/api/v1/`.

---

## Estructura anterior

```
/opt/ave-api/
├── index.js          ← todo el código aquí
├── Dockerfile
├── docker-compose.yml
└── package.json
```

## Estructura nueva

```
/opt/ave-api/
├── src/
│   ├── app.js                          ← Express + middlewares + rutas
│   ├── server.js                       ← arranque del servidor
│   ├── config/
│   │   └── env.js                      ← variables de entorno centralizadas
│   ├── api/v1/
│   │   ├── index.js                    ← router raíz /api/v1/
│   │   ├── reportes/
│   │   │   ├── reportes.routes.js
│   │   │   ├── reportes.controller.js
│   │   │   └── reportes.service.js     ← lógica PDF (pdfkit) + email (nodemailer)
│   │   ├── dashboard/
│   │   │   ├── dashboard.routes.js
│   │   │   └── dashboard.controller.js ← HTML del dashboard con Chart.js
│   │   ├── media/
│   │   │   ├── media.routes.js
│   │   │   ├── media.controller.js
│   │   │   └── media.service.js        ← multer, fileFilter, buildUrl, remove
│   │   ├── empresas/                   ← reservado (vacío)
│   │   └── servicios/                  ← reservado (vacío)
│   ├── middlewares/                    ← reservado (vacío)
│   └── utils/                          ← reservado (vacío)
├── Dockerfile                          ← actualizado: COPY src/ + CMD src/server.js
├── docker-compose.yml                  ← agregado N8N_URL al environment
├── .gitignore
├── .env.example
└── package.json
```

---

## Cambios en rutas

| Módulo | Ruta anterior | Ruta nueva |
|--------|--------------|------------|
| Generar reporte PDF | `POST /reporte/generar` | `POST /api/v1/reportes/generar` |
| Enviar reporte email | `POST /reporte/enviar` | `POST /api/v1/reportes/enviar` |
| Dashboard HTML | `GET /dashboard` | `GET /api/v1/dashboard/` |
| Upload imagen | `POST /upload/servicio/:empresa_id/:servicio_id` | `POST /api/v1/media/upload/:empresa_id/:servicio_id` |
| Eliminar imagen | `DELETE /upload/servicio/:empresa_id/:filename` | `DELETE /api/v1/media/:empresa_id/:filename` |
| Health check | `GET /health` | `GET /health` (sin cambio) |

---

## Workflows n8n actualizados

Los siguientes workflows tenían URLs hardcodeadas a las rutas viejas y fueron actualizados:

| Workflow | Nodos modificados |
|----------|------------------|
| `AVE_Reporte_Manual` | `HTTP_generar_reporte`, `HTTP_enviar_reporte` |
| `AVE_Reportes_Automaticos` | `HTTP_generar_reporte`, `HTTP_enviar_reporte` |

Los workflows `Flujo_principal_agencIA`, `AVE_Recordatorios_Automaticos` y `AVE_Dashboard_Data` **no requirieron cambios** — no consumen la API directamente.

---

## Cambios en Docker

### Dockerfile
```dockerfile
# Antes
COPY index.js .
CMD ["node", "index.js"]

# Después
COPY src/ ./src/
CMD ["node", "src/server.js"]
```

### docker-compose.yml
```yaml
# Variable agregada
environment:
  - N8N_URL=https://hn8n.identechnology.co  # nueva
  - BASE_URL=https://api.identechnology.co
  - PORT=3001
```

---

## Variables de entorno

Archivo `.env.example` creado en la raíz del repo:

```env
PORT=3001
BASE_URL=https://api.identechnology.co
N8N_URL=https://hn8n.identechnology.co
```

El archivo `.env` real **no se versiona** (está en `.gitignore`). Las variables se inyectan vía `docker-compose.yml`.

---

## Verificación en producción

```bash
# Health check
curl https://api.identechnology.co/health
# → {"status":"ok","service":"ave-api"}

# Dashboard
curl -s -o /dev/null -w "%{http_code}" https://api.identechnology.co/api/v1/dashboard/
# → 200

# Reportes (sin body → 500 esperado por validación interna)
curl -s -o /dev/null -w "%{http_code}" -X POST https://api.identechnology.co/api/v1/reportes/generar
# → 500 (correcto — el service lanza "Faltan datos: empresa, kpis")
```

---

## Commits

```
chore: commit inicial — index.js monolítico antes de refactorización
feat: refactorización API — arquitectura Routes/Controller/Service + versionado /api/v1/
```

---

## Pendiente — Bloque 2

- [ ] Crear tabla `servicios_media` en PostgreSQL
- [ ] Implementar endpoints `GET/POST/PUT/DELETE /api/v1/media/:servicio_id`
- [ ] Soporte multi-archivo por producto (imagen, video, pdf)
- [ ] Directorio `/var/www/ave/media/{empresa_id}/imagenes|videos|pdfs/`
- [ ] Actualizar tool `buscar_imagen_servicio` → `buscar_media_servicio` en n8n
- [ ] Sección de medios en Appsmith (Info Negocio)
