# Módulo: Cal.com Self-Hosted - Instalación Fase 1

**Proyecto:** AVE - agencIA
**Fecha:** 06 jul 2026
**Sesión:** Instalación base Cal.com self-hosted
**Estado:** FASE 1 COMPLETADA
**Versión Cal.com:** v6.2.0-sh (imagen `calcom/cal.com:latest`)
**URL productiva:** https://citas.identechnology.co

---

## 1. Contexto y decisión de arquitectura

Cal.com self-hosted reemplaza las tools de Google Calendar del flujo principal
de AVE. Se elige esta plataforma sobre alternativas por:

- **vs Calendly (pago):** Cal.com self-hosted es AGPLv3, USD 0/mes.
- **vs Cal.com Cloud (pago):** control total de datos, URL propia branded, sin
  límite de usuarios.
- **vs cal.diy (fork community):** cal.diy NO incluye Teams ni Organizations,
  imprescindibles para el modelo multi-tenant de AVE (agencIA, Uhane SAS,
  PC_Outlet, tienda4030). Se descarta.

**Decisión:** imagen oficial `calcom/cal.com:latest` desde DockerHub, mantenida
por el equipo de Cal.com desde nov-2025 (repo `calcom/docker` fue archivado).

**Multi-tenancy:** se usará la feature **Teams** (incluida en AGPLv3). Cada
empresa AVE será un Team con sus propios event types, calendarios conectados,
webhooks y disponibilidad.

---

## 2. Prerequisitos VPS validados

VPS: Hostinger `srv1732020`, IP pública `2.25.172.227`.

| Recurso            | Valor                          | Estado |
|--------------------|--------------------------------|--------|
| Docker             | 29.5.3                         | OK     |
| Docker Compose     | v5.1.4 (plugin, sin guion)     | OK     |
| RAM total          | 15 Gi (11 Gi libres)           | OK     |
| Disco raiz         | 193 Gi (177 Gi libres)         | OK     |
| Traefik            | v2.11 en 80/443                | OK     |
| Red Docker         | `traefik_network` (bridge)     | OK     |
| Cert resolver ACME | `mytlschallenge` (Let's Encrypt) | OK   |
| Email ACME         | enuaremiliorosales@gmail.com   | OK     |

**Contenedores ya operativos que coexisten con Cal.com:**
- `n8n-n8n-1`, `n8n-postgres-1` (n8n 2.23.3 + pgvector/pg16)
- `chatwoot-chatwoot-1`, `chatwoot-chatwoot-worker-1`, `chatwoot-postgres-1`,
  `chatwoot-redis-1`
- `ave-api` (backend AVE)
- `appsmith-appsmith-1` (panel admin)
- `traefik-traefik-1` (reverse proxy)

---

## 3. DNS configurado

Panel Hover -> zona DNS `identechnology.co`:

```
Tipo:   A
Nombre: citas
Valor:  2.25.172.227
TTL:    300
```

Verificacion desde el VPS:
```
dig +short citas.identechnology.co
# -> 2.25.172.227
```

---

## 4. Estructura de archivos

Directorio base: `/opt/calcom/`

### 4.1 `/opt/calcom/docker-compose.yml`

```yaml
services:
  calcom-postgres:
    image: postgres:16-alpine
    container_name: calcom-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: calcom
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: calcom
      TZ: America/Bogota
    volumes:
      - calcom_pgdata:/var/lib/postgresql/data
    networks:
      - calcom_internal
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U calcom -d calcom"]
      interval: 10s
      timeout: 5s
      retries: 5

  calcom:
    image: calcom/cal.com:latest
    container_name: calcom
    restart: unless-stopped
    depends_on:
      calcom-postgres:
        condition: service_healthy
    environment:
      # --- Core ---
      NEXTAUTH_SECRET: ${NEXTAUTH_SECRET}
      CALENDSO_ENCRYPTION_KEY: ${CALENDSO_ENCRYPTION_KEY}
      DATABASE_URL: postgresql://calcom:${POSTGRES_PASSWORD}@calcom-postgres:5432/calcom
      DATABASE_DIRECT_URL: postgresql://calcom:${POSTGRES_PASSWORD}@calcom-postgres:5432/calcom
      # --- URLs ---
      NEXT_PUBLIC_WEBAPP_URL: https://citas.identechnology.co
      NEXTAUTH_URL: https://citas.identechnology.co/api/auth
      NEXT_PUBLIC_WEBSITE_URL: https://citas.identechnology.co
      # --- Licenciamiento / consentimientos ---
      NEXT_PUBLIC_LICENSE_CONSENT: "agree"
      CALCOM_TELEMETRY_DISABLED: "1"
      NEXT_PUBLIC_TELEMETRY_KEY: ""
      # --- Branding ---
      NEXT_PUBLIC_APP_NAME: "AVE Citas"
      # --- SMTP (Hover) ---
      EMAIL_FROM: "info@identechnology.co"
      EMAIL_FROM_NAME: "AVE agencIA"
      EMAIL_SERVER_HOST: "mail.hover.com.cust.hostedemail.com"
      EMAIL_SERVER_PORT: "465"
      EMAIL_SERVER_USER: "info@identechnology.co"
      EMAIL_SERVER_PASSWORD: ${SMTP_PASSWORD}
      # --- Zona horaria ---
      TZ: America/Bogota
    networks:
      - calcom_internal
      - traefik_network
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=traefik_network"
      - "traefik.http.middlewares.calcom-headers.headers.customRequestHeaders.X-Forwarded-Proto=https"
      - "traefik.http.routers.calcom.entrypoints=websecure"
      - "traefik.http.routers.calcom.middlewares=calcom-headers"
      - "traefik.http.routers.calcom.rule=Host(`citas.identechnology.co`)"
      - "traefik.http.routers.calcom.tls=true"
      - "traefik.http.routers.calcom.tls.certresolver=mytlschallenge"
      - "traefik.http.services.calcom.loadbalancer.server.port=3000"

networks:
  calcom_internal:
    driver: bridge
  traefik_network:
    external: true

volumes:
  calcom_pgdata:
    driver: local
```

### 4.2 `/opt/calcom/.env` (TEMPLATE - valores redactados)

Los valores reales estan en el VPS. Permisos `chmod 600`.

```env
# ==========================================
# Cal.com self-hosted - AVE agencIA
# VPS Hostinger - srv1732020
# ==========================================

# Secretos core (generados con openssl rand -base64)
NEXTAUTH_SECRET=<REDACTED_OPENSSL_RAND_BASE64_32>
CALENDSO_ENCRYPTION_KEY=<REDACTED_OPENSSL_RAND_BASE64_24>

# Postgres dedicado Cal.com
POSTGRES_PASSWORD=<REDACTED_OPENSSL_RAND_BASE64_24_TRIMMED>

# SMTP Hover (buzon info@identechnology.co)
SMTP_PASSWORD=<REDACTED_HOVER_MAILBOX_PASSWORD>
```

**Comandos usados para generar los secretos:**
```bash
openssl rand -base64 32                                    # NEXTAUTH_SECRET
openssl rand -base64 24                                    # CALENDSO_ENCRYPTION_KEY
openssl rand -base64 24 | tr -d '/+=' | cut -c1-32         # POSTGRES_PASSWORD
```

---

## 5. Comandos de arranque

```bash
cd /opt/calcom

# Validar compose antes de levantar
docker compose config --quiet && echo "OK compose valido"

# Levantar stack (primera vez descarga ~800 MB)
docker compose up -d

# Verificar estado
docker compose ps

# Ver logs en vivo
docker compose logs -f calcom
```

**Salida esperada de `docker compose ps` al primer arranque:**

```
NAME              STATUS                        PORTS
calcom            Up About a minute (healthy)   3000/tcp
calcom-postgres   Up About a minute (healthy)   5432/tcp
```

Los puertos NO se exponen al host; Traefik enruta desde `citas.identechnology.co`
a `calcom:3000` por la red interna `traefik_network`.

---

## 6. Verificacion de acceso

URL: **https://citas.identechnology.co**

**Wizard inicial de Cal.com (3 pasos):**

1. **Paso 1 - Usuario administrador**
   - Usuario: `enuar`
   - Nombre Completo: `Enuar Emilio Rosales`
   - Email: `enuaremiliorosales@gmail.com`
   - Contrasena: >= 15 chars, mayus/minus, al menos 1 numero
     (ver TODO seccion 8: rotar a valor mas fuerte)

2. **Paso 2 - Licencia**
   - Seleccion: **Licencia AGPLv3** (gratis)
   - Accion: "Saltar paso" (equivale a aceptar AGPLv3 sin ingresar
     license key Enterprise)

3. **Paso 3 - Activar aplicaciones**
   - Todas las apps que requieren OAuth (Google Calendar, Google Meet,
     Zoom, Office 365) se dejan en **OFF** por ahora.
   - Se configuran en Fase 2 (requieren credenciales OAuth de Google Cloud
     Console, Zoom Marketplace, Azure AD).

**Verificacion final:**
- Panel admin accesible en `https://citas.identechnology.co/event-types`
- Pagina publica accesible en `https://citas.identechnology.co/enuar`
- SSL Let's Encrypt emitido automaticamente por Traefik
- 3 event types por defecto creados: `/enuar/30min`, `/enuar/15min`,
  `/enuar/secret` (este ultimo oculto)

---

## 7. Notas de seguridad

**Secretos expuestos durante la sesion (rotar OBLIGATORIO):**

Durante el proceso de instalacion, se pegaron en el chat de asistencia los
siguientes secretos que deben considerarse **comprometidos** y ser rotados:

- Password cuenta Hover (`enuar` -> panel hover.com)
- Password buzon `info@identechnology.co`

**Adicional:** el password admin inicial de Cal.com NO cumplio los 15
caracteres minimos requeridos por la propia app (aparece banner naranja en
el panel).

Todos los items estan en la seccion 8 (TODO).

---

## 8. TODO al cierre de Fase 1

**Prioridad ALTA - Seguridad:**

- [ ] Cambiar password admin Cal.com (>= 15 chars, mayus/minus, numeros)
      -> Panel Cal.com -> Ajustes -> Seguridad -> Change password
- [ ] Activar 2FA en Cal.com
      -> Ajustes -> Security -> Two-Factor Authentication (Google
      Authenticator / Authy)
- [ ] Rotar password buzon `info@identechnology.co` en panel Hover
      -> Hover -> Emails -> info@identechnology.co -> Change password
- [ ] Actualizar `SMTP_PASSWORD` en `/opt/calcom/.env` con el nuevo password
      del buzon
- [ ] Reiniciar contenedor: `cd /opt/calcom && docker compose restart calcom`
- [ ] Verificar envio SMTP: enviar booking de prueba y confirmar correo
- [ ] Rotar password cuenta Hover (usuario `enuar`)
- [ ] Activar 2FA en cuenta Hover

**Prioridad MEDIA - Operativo:**

- [ ] Descartar los event types por defecto `Reunion de 15 min` y
      `Reunion de 30 min` si no se van a usar (o adaptarlos al estandar AVE)
- [ ] Configurar zona horaria y horario de disponibilidad estandar AVE
      (lun-vie 08:00-17:00 America/Bogota)
- [ ] Backup automatizado del volumen `calcom_pgdata` (integrar con backup
      script existente del VPS)

---

## 9. Roadmap Fase 2 - OAuth integraciones

**Objetivo:** habilitar la conexion de calendarios y videollamadas externos
para los tenants.

### 9.1 Google Calendar + Google Meet

1. Ir a https://console.cloud.google.com
2. Crear proyecto: `ave-calcom-oauth`
3. Habilitar APIs:
   - Google Calendar API
   - Google Meet API (via Calendar API)
4. Configurar pantalla de consentimiento OAuth:
   - Tipo: **External** (Testing mode inicial)
   - App name: `AVE Citas`
   - Support email: `info@identechnology.co`
   - Authorized domains: `identechnology.co`
   - Scopes: `.../auth/calendar`, `.../auth/calendar.events`
5. Crear OAuth 2.0 Client ID:
   - Application type: **Web application**
   - Authorized redirect URIs:
     - `https://citas.identechnology.co/api/integrations/googlecalendar/callback`
     - `https://citas.identechnology.co/api/integrations/googlemeet/callback`
6. Copiar `client_id` y `client_secret`
7. En Cal.com: Aplicaciones -> Calendario -> Google Calendar -> Editar claves
   -> pegar `client_id`, `client_secret`, `redirect_uris` -> Guardar

### 9.2 Zoom

1. Ir a https://marketplace.zoom.us
2. Crear app: **OAuth** (User-managed)
3. Redirect URL: `https://citas.identechnology.co/api/integrations/zoom/callback`
4. Scopes: `meeting:write`, `meeting:read`
5. En Cal.com: Aplicaciones -> Conferencias -> Zoom -> Editar claves

### 9.3 Office 365 / Outlook Calendar (opcional)

Solo si algun tenant corporativo lo requiere.
1. Ir a https://portal.azure.com -> Azure Active Directory -> App registrations
2. New registration -> Redirect URI:
   `https://citas.identechnology.co/api/integrations/office365calendar/callback`

---

## 10. Roadmap Fase 3 - Teams multi-tenant + integracion AVE

**Objetivo:** configurar el modelo multi-tenant y conectar Cal.com al flujo
principal de AVE (n8n + Postgres + Chatwoot).

### 10.1 Crear Teams

En Cal.com -> Equipos -> Crear equipo. Uno por empresa:

- `agencia` (empresa_id = 1)
- `uhane` (empresa_id = 7)
- `pcoutlet` (empresa_id = 8)
- `tienda4030` (empresa_id = 9)

Cada team tendra:
- URL propia: `citas.identechnology.co/<team>`
- Sus propios event types
- Sus propios miembros
- Su propio calendario conectado

### 10.2 Conectar Google Calendar por tenant

Cada tenant conecta su Google Calendar corporativa desde el panel de su
team (requiere Fase 2 completa).

### 10.3 Webhooks Cal.com -> n8n

En Cal.com -> Ajustes -> Desarrollador -> Webhooks:

- URL: `https://hn8n.identechnology.co/webhook/calcom-booking`
- Eventos:
  - `BOOKING_CREATED`
  - `BOOKING_RESCHEDULED`
  - `BOOKING_CANCELLED`
  - `MEETING_ENDED`
- Secret: generar y guardar en n8n para validar HMAC

### 10.4 Workflow n8n para procesar bookings

Crear workflow `calcom-booking-handler` que:
1. Reciba webhook
2. Valide HMAC signature
3. Identifique empresa (por team_slug o event_type)
4. Inserte/actualice registro en tabla `citas` de Postgres AVE
5. Actualice `estado_lead` en tabla `leads`
6. Envie confirmacion por WhatsApp via Chatwoot API
7. Encole recordatorio (24h antes) usando el sistema existente de
   recordatorios blindado

### 10.5 Tabla `citas` en Postgres AVE

Schema propuesto (a validar en sesion Fase 3):

```sql
CREATE TABLE citas (
    id SERIAL PRIMARY KEY,
    calcom_booking_id VARCHAR(255) UNIQUE NOT NULL,
    empresa_id INT NOT NULL REFERENCES empresas(id),
    lead_id INT REFERENCES leads(id),
    event_type_slug VARCHAR(100),
    fecha_cita TIMESTAMPTZ NOT NULL,
    duracion_min INT,
    estado VARCHAR(20) DEFAULT 'confirmada',
        -- confirmada, reagendada, cancelada, completada, no_show
    meeting_url TEXT,
    calcom_payload JSONB,
    recordatorios_enviados INT DEFAULT 0,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_citas_empresa ON citas(empresa_id);
CREATE INDEX idx_citas_lead ON citas(lead_id);
CREATE INDEX idx_citas_fecha ON citas(fecha_cita);
```

---

## 11. Referencias

- Cal.com repo oficial: https://github.com/calcom/cal.com
- Imagen DockerHub oficial: https://hub.docker.com/r/calcom/cal.com
- Docs self-hosting: https://cal.com/docs/self-hosting
- Cal.com v5.9 (Docker oficial en monorepo):
  https://cal.com/blog/calcom-v5-9
- Traefik v2 docs (labels Docker): https://doc.traefik.io/traefik/v2.11/
- Let's Encrypt ACME challenge: https://letsencrypt.org/docs/challenge-types/

---

## 12. Comandos de mantenimiento

```bash
# Actualizar Cal.com a version mas reciente
cd /opt/calcom
docker compose pull
docker compose down
docker compose up -d

# Ver logs de un rango
docker compose logs --since 1h calcom

# Backup manual del volumen Postgres
docker exec calcom-postgres pg_dump -U calcom calcom > \
  /root/backups/calcom-$(date +%Y%m%d-%H%M%S).sql

# Restaurar (ejemplo, cuidado en produccion)
docker exec -i calcom-postgres psql -U calcom calcom < \
  /root/backups/calcom-YYYYMMDD-HHMMSS.sql

# Entrar a la BD
docker exec -it calcom-postgres psql -U calcom -d calcom

# Ver uso de recursos del stack
docker stats calcom calcom-postgres
```

---

**Fin del modulo Fase 1.**
