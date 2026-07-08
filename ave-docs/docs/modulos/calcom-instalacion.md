# Modulo: Cal.com Self-Hosted - Instalacion + OAuth + Multi-tenant

**Proyecto:** AVE - agencIA
**Fase 1 (instalacion base):** 06 jul 2026 - COMPLETADA
**Fase 2 (OAuth Google Calendar):** 07 jul 2026 - COMPLETADA
**Fase 3 (Multi-tenant + integracion AVE):** 07 jul 2026 - COMPLETADA
**Version Cal.com:** v6.2.0-sh (imagen `calcom/cal.com:latest`)
**URL productiva:** https://citas.identechnology.co
**Team en produccion:** agencIA (empresa_id=1) -> https://citas.identechnology.co/team/agencia
**Estado:** OPERATIVO END-TO-END - Cal.com -> n8n -> Postgres -> Chatwoot validado

---

## 1. Contexto y decision de arquitectura

Cal.com self-hosted reemplaza las tools de Google Calendar del flujo principal
de AVE. Se elige esta plataforma sobre alternativas por:

- **vs Calendly (pago):** Cal.com self-hosted es AGPLv3, USD 0/mes.
- **vs Cal.com Cloud (pago):** control total de datos, URL propia branded,
  sin limite de usuarios.
- **vs Cal.diy (fork community):** Cal.diy NO incluye Teams ni Organizations,
  imprescindibles para el modelo multi-tenant de AVE. Se descarta.

**IMPORTANTE - Cambio de licenciamiento en Cal.com v6.4 (abril 2026):**
Cal.com Inc. movio la edicion commercial a **closed source** y relanzo el
codigo abierto como **Cal.diy bajo MIT**. La edicion commercial requiere
license key Enterprise para features avanzadas.

**Sin embargo**, en la version en la que se instalo (v6.2.0-sh) la feature
**Teams sigue funcional para creacion de event types y URLs publicas**. El
unico bloqueo cosmetico es la gestion de Members ("commercial feature"),
que **NO impacta el caso de AVE** porque cada team tiene un unico owner
(el propietario del tenant conecta su propio Google Calendar).

**Roadmap si futuras versiones bloquean Teams completo:** migrar a users
como tenants (URL cambia de `/team/<slug>/<event>` a `/<username>/<event>`).
Arquitecturalmente equivalente.

**Multi-tenancy:** feature **Teams** (AGPLv3 hasta v6.2). Cada empresa
AVE es un Team con:
- URL propia branded
- Sus propios event types
- Su propio Google Calendar conectado
- Sus propios webhooks
- Miembros del tenant como hosts

---

## 2. Prerequisitos VPS validados

VPS: Hostinger `srv1732020`, IP publica `2.25.172.227`.

| Recurso            | Valor                            | Estado |
|--------------------|----------------------------------|--------|
| Docker             | 29.5.3                           | OK     |
| Docker Compose     | v5.1.4 (plugin, sin guion)       | OK     |
| RAM total          | 15 Gi                            | OK     |
| Disco raiz         | 193 Gi (177 Gi libres)           | OK     |
| Traefik            | v2.11 en 80/443                  | OK     |
| Red Docker         | `traefik_network` (bridge)       | OK     |
| Cert resolver ACME | `mytlschallenge` (Let's Encrypt) | OK     |
| Email ACME         | enuaremiliorosales@gmail.com     | OK     |

**Contenedores que coexisten con Cal.com en el mismo VPS:**

- `n8n-n8n-1`, `n8n-postgres-1` (n8n 2.23.3 + pgvector/pg16)
- `chatwoot-chatwoot-1`, `chatwoot-chatwoot-worker-1`,
  `chatwoot-postgres-1`, `chatwoot-redis-1`
- `ave-api` (backend AVE)
- `appsmith-appsmith-1` (panel admin)
- `traefik-traefik-1` (reverse proxy)
- **NUEVOS (Fase 1):** `calcom`, `calcom-postgres`

---

## 3. DNS y estructura de archivos

### 3.1 DNS

Panel Hover -> zona DNS `identechnology.co`:

```
Tipo:   A
Nombre: citas
Valor:  2.25.172.227
TTL:    300
```

Verificacion:
```
dig +short citas.identechnology.co
# -> 2.25.172.227
```

### 3.2 `/opt/calcom/docker-compose.yml`

**Nota:** este es el compose **con el fix SMTP aplicado** (Section 12.2).
`EMAIL_SERVER_HOST` corregido a `smtp.hostedemail.com`.

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
      # --- SMTP (Hover) - FIX aplicado Section 12.2 ---
      EMAIL_FROM: "info@identechnology.co"
      EMAIL_FROM_NAME: "AVE agencIA"
      EMAIL_SERVER_HOST: "smtp.hostedemail.com"
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

### 3.3 `/opt/calcom/.env` (TEMPLATE - valores redactados)

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

## 4. Comandos de arranque

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

**Salida esperada:**

```
NAME              STATUS                        PORTS
calcom            Up 8 seconds (healthy)        3000/tcp
calcom-postgres   Up 30 seconds (healthy)       5432/tcp
```

Los puertos NO se exponen al host; Traefik enruta desde
`citas.identechnology.co` a `calcom:3000` por la red interna
`traefik_network`.

---

## 5. Wizard inicial de Cal.com

URL: **https://citas.identechnology.co**

### Paso 1 - Usuario administrador
- Usuario: `enuar`
- Nombre Completo: `Enuar Emilio Rosales`
- Email: `enuaremiliorosales@gmail.com`
- Contrasena: >= 15 chars, generada con
  `openssl rand -base64 20 | tr -d '/+=' | head -c 22`
- 2FA activado con Google Authenticator / Authy
  (Ajustes -> Seguridad -> Two-Factor Authentication)
- Codigos de recuperacion guardados en gestor seguro

### Paso 2 - Licencia
- Seleccion: **Licencia AGPLv3** (gratis)
- Accion: "Saltar paso" (equivale a aceptar AGPLv3 sin ingresar
  license key Enterprise)

### Paso 3 - Activar aplicaciones
- Todas las apps que requieren OAuth (Google Calendar, Google Meet,
  Zoom, Office 365) se dejan en **OFF**.
- La configuracion OAuth se realiza en Fase 2.

### Verificacion Fase 1
- Panel admin accesible en `https://citas.identechnology.co/event-types`
- Pagina publica accesible en `https://citas.identechnology.co/enuar`
- SSL Let's Encrypt emitido automaticamente por Traefik
- 3 event types por defecto creados: `/enuar/30min`, `/enuar/15min`,
  `/enuar/secret` (este ultimo oculto)

---

## 6. Fase 2 - OAuth Google Calendar

### 6.1 Google Cloud Console - Crear proyecto

1. URL: https://console.cloud.google.com
2. Loguearse con `enuaremiliorosales@gmail.com`
3. Selector de proyectos -> **NEW PROJECT**
4. Nombre: `ave-calcom-oauth`
5. **Sin activar billing** (Google Calendar API es gratis en el free tier
   hasta 1.000.000 requests/dia)

**Decision:** proyecto SEPARADO del existente `n8n-30-05-2026` por
separacion de responsabilidades, auditoria de accesos independiente, y
para que rotaciones de credenciales de un proyecto no afecten al otro.

### 6.2 Habilitar Google Calendar API

1. En el proyecto `ave-calcom-oauth`: menu -> **APIs y servicios ->
   Biblioteca**
2. Buscar: `Google Calendar API`
3. Click en la tarjeta -> **HABILITAR**

### 6.3 Configurar Pantalla de consentimiento OAuth

Menu lateral: **Google Auth Platform -> Pantalla de consentimiento**

| Campo                | Valor                            |
|----------------------|----------------------------------|
| App name             | `AVE Citas`                      |
| User support email   | `enuaremiliorosales@gmail.com`   |
| Audience / User Type | **External**                     |
| Contact email        | `enuaremiliorosales@gmail.com`   |
| Estado publicacion   | **Testing** (NO publicar aun)    |

### 6.4 Crear cliente OAuth 2.0

Menu: **Google Auth Platform -> Clientes -> Crear cliente**

| Campo                              | Valor                             |
|------------------------------------|-----------------------------------|
| Tipo de aplicacion                 | Aplicacion web                    |
| Nombre                             | `AVE Citas - Cal.com`             |
| Origenes JavaScript autorizados    | `https://citas.identechnology.co` |

**URIs de redireccionamiento autorizadas (2 URIs):**

```
https://citas.identechnology.co/api/integrations/googlecalendar/callback
https://citas.identechnology.co/api/integrations/googlevideo/callback
```

Al crear se muestran:
- **ID de cliente** (formato: `<numero>-<hash>.apps.googleusercontent.com`)
- **Secreto del cliente** (formato: `GOCSPX-<hash>`)

Ambos se guardan en **gestor de contrasenas** (Bitwarden / 1Password /
KeePassXC). **NO se descarga el JSON** por politica de seguridad
institucional del equipo AVE.

### 6.5 Agregar usuarios de prueba

Menu: **Google Auth Platform -> Publico -> Usuarios de prueba**

Emails iniciales (Fase 2):
- `enuaremiliorosales@gmail.com`
- `info@identechnology.co` (solo si tiene cuenta Google asociada)

**Limite: 100 usuarios en modo Testing.**

### 6.6 Pegar credenciales en Cal.com

URL: **https://citas.identechnology.co/settings/admin/apps/calendar**

1. Buscar tarjeta **Google Calendar** -> click en icono lapiz
   (Editar claves)
2. Rellenar los 3 campos:

| Campo          | Valor                                                                    |
|----------------|--------------------------------------------------------------------------|
| `client_id`    | El generado en Google Cloud                                              |
| `client_secret`| El generado en Google Cloud                                              |
| `redirect_uris`| `https://citas.identechnology.co/api/integrations/googlecalendar/callback` |

3. Guardar y activar el **toggle** de Google Calendar (queda en verde/ON)

### 6.7 Prueba OAuth end-to-end

1. Ir a **Ajustes -> Mi cuenta -> Calendarios** o URL directa
   `https://citas.identechnology.co/apps/installed/calendar`
2. En **App Store**, buscar Google Calendar -> **Install app**
3. Popup Google: loguearse con `enuaremiliorosales@gmail.com`
4. Aparece advertencia: **"Google no verifico esta app"**
   - Es esperado en modo Testing, NO es peligroso
   - Click **"Configuracion avanzada"** -> **"Ir a AVE Citas (no seguro)"**
5. Pantalla de permisos -> **Permitir**
6. Retorno automatico a Cal.com con Google Calendar conectado

**Verificaciones bidireccionales confirmadas en produccion:**

| Escenario                                    | Resultado |
|----------------------------------------------|-----------|
| Slot ocupado en Google Calendar              | Bloqueado en Cal.com (OK) |
| Reserva creada en Cal.com                    | Evento en Google Calendar (OK) |
| Cancelacion en Cal.com                       | Evento eliminado en GCal (OK) |

---

## 7. Fase 2b - Google Meet como app de conferencing

Cal.com por defecto usa **Cal Video** (Daily.co embebido) que requiere
`DAILY_API_KEY`. Como alternativa mas simple se instala **Google Meet**,
que reutiliza el OAuth del proyecto `ave-calcom-oauth` sin necesidad de
credenciales adicionales.

### Pasos

1. URL directa: `https://citas.identechnology.co/apps/categories/conferencing`
2. Buscar tarjeta **Google Meet** -> click **Install**
3. Autorizar si pide OAuth (usa mismo `enuaremiliorosales@gmail.com`)
4. **Set as default** (opcional pero recomendado)
5. En cada Event Type -> Setup -> Location -> seleccionar **Google Meet**
6. Guardar

**Beneficio:** en cada reserva se genera un link real tipo
`https://meet.google.com/xxx-xxxx-xxx` automaticamente y aparece en el
email de confirmacion + en el evento de Google Calendar.

---

## 8. Fase 3 - Team multi-tenant agencIA

### 8.1 Crear Team

En Cal.com: **Teams -> +New team**

| Campo                     | Valor        |
|---------------------------|--------------|
| Nombre visible            | `agencIA`    |
| Slug URL                  | `agencia`    |
| URL publica               | `https://citas.identechnology.co/team/agencia` |
| Owner                     | enuar (unico miembro por ahora) |

**Nota sobre members:** el panel "Members" muestra un warning
"commercial feature" en v6.2.0-sh, pero como agencIA tiene un unico
owner (el propio `enuar`), esto no afecta operacion.

### 8.2 Event Type `Consulta estrategica AVE`

En **agencIA -> Event types -> +New event type**

| Campo                | Valor                                    |
|----------------------|------------------------------------------|
| Title                | `Consulta estrategica AVE`               |
| Slug (auto-generado) | `consulta-estrategica-ave`               |
| Duracion             | 30 minutos                               |
| Assignment           | **Collective**                           |
| Fixed host           | Enuar Emilio Rosales                     |
| Location             | Google Meet                              |
| Toggle activo        | ON                                       |

**URL publica final:**
`https://citas.identechnology.co/team/agencia/consulta-estrategica-ave`

---

## 9. Fase 3 - Ampliacion de schema en Postgres AVE

Todas las ALTER TABLE se ejecutaron via DBeaver contra la BD `postgres`
en `n8n-postgres-1`. Tabla `citas` ya existia con 11 columnas, se ampliaron
para soportar Cal.com sin romper compatibilidad con integraciones previas.

### 9.1 Ampliar tabla `empresas`

```sql
ALTER TABLE public.empresas
ADD COLUMN calcom_team_slug VARCHAR(50) UNIQUE;

COMMENT ON COLUMN public.empresas.calcom_team_slug IS
'Slug del team en Cal.com self-hosted. Ejemplo: agencia, uhane, pcoutlet,
tienda4030. Usado por webhooks para identificar tenant. UNIQUE = un slug
solo puede mapear a una empresa.';

-- Mapeo inicial
UPDATE public.empresas
SET calcom_team_slug = 'agencia'
WHERE id = 1;
```

### 9.2 Ampliar tabla `citas` (6 columnas nuevas)

```sql
ALTER TABLE public.citas ADD COLUMN calcom_booking_id INTEGER UNIQUE;
COMMENT ON COLUMN public.citas.calcom_booking_id IS
'ID numerico del booking en Cal.com (Booking.id). UNIQUE para idempotencia de webhooks.';

ALTER TABLE public.citas ADD COLUMN calcom_uid VARCHAR(100) UNIQUE;
COMMENT ON COLUMN public.citas.calcom_uid IS
'UID unico del booking usado en URLs publicas de Cal.com (Booking.uid).';

ALTER TABLE public.citas ADD COLUMN calcom_team_slug VARCHAR(50);
COMMENT ON COLUMN public.citas.calcom_team_slug IS
'Slug del team Cal.com. Redundante con empresas.calcom_team_slug via JOIN,
pero util para auditoria y queries rapidas.';

ALTER TABLE public.citas ADD COLUMN event_type_slug VARCHAR(100);
COMMENT ON COLUMN public.citas.event_type_slug IS
'Slug del event type reservado en Cal.com. Ej: consulta-estrategica-ave.';

ALTER TABLE public.citas ADD COLUMN meeting_url TEXT;
COMMENT ON COLUMN public.citas.meeting_url IS
'URL de videollamada generada automaticamente por Cal.com (Google Meet, Zoom, Cal Video).';

ALTER TABLE public.citas ADD COLUMN calcom_payload JSONB;
COMMENT ON COLUMN public.citas.calcom_payload IS
'Payload JSON completo del ultimo webhook Cal.com recibido. Para auditoria, debug y re-procesamiento.';
```

### 9.3 Indices adicionales

```sql
CREATE INDEX IF NOT EXISTS idx_citas_calcom_team_slug
ON public.citas (calcom_team_slug);

CREATE INDEX IF NOT EXISTS idx_citas_lead_id
ON public.citas (lead_id);

CREATE INDEX IF NOT EXISTS idx_citas_estado
ON public.citas (estado);
```

### 9.4 Verificaciones - integridad ya existente

**FKs ya existentes en `citas`:**

| Constraint                | Columna       | Referencia        | ON DELETE  |
|---------------------------|---------------|-------------------|------------|
| citas_empresa_id_fkey     | empresa_id    | empresas(id)      | CASCADE    |
| citas_lead_id_fkey        | lead_id       | leads(id)         | SET NULL   |
| citas_servicio_id_fkey    | servicio_id   | servicios(id)     | SET NULL   |

**Trigger ya existente:**

```
trg_citas_updated_at  BEFORE UPDATE  EXECUTE FUNCTION set_updated_at()
```

### 9.5 Estado final tabla `citas`

- **17 columnas totales** (11 originales + 6 nuevas Cal.com)
- **8 indices** (PK + 2 UNIQUE + 5 idx)
- **3 FKs** con politicas correctas
- **1 trigger** de updated_at automatico
- **Sin migracion de datos** (tabla estaba vacia al momento del ALTER)

---

## 10. Fase 3 - Workflow n8n `calcom-booking-handler`

Archivo importado: **`calcom-booking-handler.json`** (adjunto al modulo).

Contiene **21 nodos funcionales + 5 sticky notes** documentales.

- **URL webhook:** `https://hn8n.identechnology.co/webhook/calcom-booking`
- **Credencial Postgres:** reutilizada `VkbhLUz1eTi4XP1f` nombre
  `Postgres account` (misma que TEST-PRINCIPAL, AVE_Recordatorios, etc.)
- **Metodo HTTP:** POST con `rawBody: true` (para HMAC exacto)

### Diseno logico

```
1.  Webhook_calcom              (POST + rawBody)
2.  Validate_HMAC               (Code JS con crypto + timingSafeEqual)
3.  If_HMAC_valid
        FALSE -> Respond_401
        TRUE  -> continua
4.  Extract_data                (Set con 13 campos del payload)
5.  PG_get_empresa_by_team      (match por calcom_team_slug)
6.  If_empresa_encontrada
        FALSE -> Respond_404_tenant
        TRUE  -> continua
7.  PG_find_or_create_lead      (CTE atomica: match email OR telefono,
                                 si no INSERT con origen='calcom')
8.  Switch_trigger_event        (4 rutas)
        -> BOOKING_CREATED     -> 9.  PG_insert_cita
        -> BOOKING_RESCHEDULED -> 10. PG_update_cita_reschedule
        -> BOOKING_CANCELLED   -> 11. PG_update_cita_cancel
        -> MEETING_ENDED       -> 12. PG_update_cita_ended
13. PG_update_lead_estado       (CASE agendado/cancelado/atendido)
14. Format_wa_message           (Code JS con Intl.DateTimeFormat America/Bogota)
15. If_debe_enviar_wa
16. PG_find_active_conversation (busca conversacion abierta en Chatwoot)
17. If_has_conversation
        FALSE -> Respond_200
        TRUE  -> 18. Chatwoot_send_wa (POST HTTP con api_access_token del tenant)
19. Respond_200                 (JSON success)
```

### Campos extraidos por `Extract_data`

`trigger_event`, `booking_id`, `booking_uid`, `team_slug`,
`event_type_slug`, `fecha_inicio`, `fecha_fin`, `attendee_email`,
`attendee_name`, `attendee_phone`, `meeting_url`, `duracion_min`,
`raw_payload`.

### Match del lead

CTE con `WITH existing AS (...), inserted AS (...) SELECT ...`:
1. Buscar por `email` con `empresa_id` correcto
2. Fallback por `telefono` si no hay match
3. Si no existe -> `INSERT` con `origen='calcom'`, `estado_lead='agendado'`

Sin race conditions gracias a la atomicidad CTE.

---

## 11. Fase 3 - Configurar webhook en Cal.com

URL: `https://citas.identechnology.co/settings/developer/webhooks`

Al crear webhook Cal.com pregunta ambito -> **seleccionar team agencIA**
(no personal Enuar), para que el payload traiga `team.slug='agencia'`
que espera el nodo `PG_get_empresa_by_team`.

### Configuracion aplicada

| Campo             | Valor                                                       |
|-------------------|-------------------------------------------------------------|
| Subscriber URL    | `https://hn8n.identechnology.co/webhook/calcom-booking`     |
| Enable webhook    | ON                                                          |
| Event triggers    | Booking created, Booking rescheduled, Booking canceled, Meeting ended |
| Secret            | 64 chars hex (generado con `openssl rand -hex 32`)          |
| Payload template  | Default                                                     |

**Secret HMAC:** guardado en:
1. Cal.com (campo Secret del webhook)
2. Bloc de notas local temporal para paso 12.1 fix crypto
3. Hardcoded en nodo `Validate_HMAC` de n8n (feature Variables es
   Enterprise, no disponible en community edition)

### Ping test

**Resultado:** `passed` - HTTP 200. HMAC validado, workflow procesa sin
error el evento PING (aunque no matchea Switch, cae limpio a Respond_200).

---

## 12. Fixes aplicados durante la sesion

### 12.1 Fix `crypto` module bloqueado en n8n 2.23.3

**Sintoma:** primera ejecucion del workflow crasheo con:
```
Module 'crypto' is disallowed [line 1]
```

**Causa:** n8n 2.23.3 introdujo **nueva task-runner architecture** que
aplica sandbox restrictivo a Code nodes por defecto, bloqueando modulos
builtins de Node.

**Fix:** agregar variable de entorno en `/opt/n8n/.env`

```bash
# Backup
cp /opt/n8n/.env /opt/n8n/.env.bak-$(date +%Y%m%d-%H%M%S)

# Agregar variable
echo "" >> /opt/n8n/.env
echo "# Habilitar modulo crypto en Code nodes (necesario para HMAC calcom-booking-handler)" >> /opt/n8n/.env
echo "NODE_FUNCTION_ALLOW_BUILTIN=crypto" >> /opt/n8n/.env

# Reiniciar n8n
cd /opt/n8n && docker compose up -d n8n
```

**Verificacion:** re-ejecutar Ping test desde Cal.com -> nodo
`Validate_HMAC` ejecuta sin error (verde). Comparativa antes/despues:

| Momento    | Duracion | Estado      | Nodo fallado          |
|------------|----------|-------------|-----------------------|
| Antes fix  | 22ms     | Error       | Validate_HMAC (crypto)|
| Despues fix| 96ms     | Succeeded   | Ninguno               |

### 12.2 Fix SMTP hostname/certificate mismatch

**Sintoma:** emails de confirmacion no llegaban al asistente. En
`docker logs calcom`:

```
Hostname/IP does not match certificate's altnames:
Host: mail.hover.com.cust.hostedemail.com. is not in the cert's altnames:
DNS:*.hostedemail.com, DNS:hostedemail.com

SEND_BROKEN_INTEGRATION_ERROR
SEND_BOOKING_CONFIRMATION_ERROR
```

**Causa:** el wildcard `*.hostedemail.com` solo cubre 2 niveles de
subdomain (`algo.hostedemail.com`) pero el hostname
`mail.hover.com.cust.hostedemail.com` tiene 3 niveles. Node.js aplica
strict TLS por defecto y rechaza.

**Fix:** cambiar host SMTP a `smtp.hostedemail.com` (mismo servidor
`216.40.42.128`, cubierto por el wildcard).

Verificacion previa:
```bash
getent hosts smtp.hostedemail.com
# -> 216.40.42.128  mail.hostedemail.com smtp.hostedemail.com

nc -zv smtp.hostedemail.com 465
# -> Connection to smtp.hostedemail.com (216.40.42.128) 465 port [tcp/submissions] succeeded!
```

Aplicacion del fix:
```bash
# Backup
cp /opt/calcom/docker-compose.yml /opt/calcom/docker-compose.yml.bak-$(date +%Y%m%d-%H%M%S)

# Reemplazar host SMTP
sed -i 's|EMAIL_SERVER_HOST: "mail.hover.com.cust.hostedemail.com"|EMAIL_SERVER_HOST: "smtp.hostedemail.com"|' /opt/calcom/docker-compose.yml

# Validar y aplicar
cd /opt/calcom
docker compose config --quiet && echo "OK"
docker compose up -d calcom
```

**Verificacion post-fix:**
```bash
docker exec calcom env | grep EMAIL_SERVER_HOST
# -> EMAIL_SERVER_HOST=smtp.hostedemail.com

# Reservar y ver logs
docker logs calcom --since 3m 2>&1 | grep -iE "smtp|email|send"
# -> sendEmail from: AVE agencIA <info@identechnology.co> to: enuaremiliorosales@gmail.com
# -> SIN mensajes SEND_..._ERROR
```

Emails llegan a Gmail inbox (no spam).

### 12.3 Fix Cal Video / instalar Google Meet

**Sintoma:** primer email de reserva mostraba:
```
There was a problem adding a video link. We could not add the
dailyvideo meeting link to your scheduled event.
```

En Google Calendar aparecia location literal `integrations:daily`.

**Causa:** location default del event type era **Cal Video** (Daily.co
embebido) que requiere `DAILY_API_KEY` en el `.env`, no configurado.

**Fix:** instalar Google Meet desde App Store (reutiliza OAuth ya
existente sin necesidad de nueva credencial):

1. URL: `https://citas.identechnology.co/apps/categories/conferencing`
2. Google Meet -> **Install app**
3. Event Type "Consulta estrategica AVE" -> Setup -> Location -> **Google Meet**
4. Save

**Verificacion:** nueva reserva genera link real
`https://meet.google.com/jek-fore-jty` (ejemplo real de la sesion).
Apps status en email:
- Google Calendar OK
- Google Meet OK

---

## 13. Notas de seguridad

- **Password admin Cal.com:** >=15 chars + 2FA activo (rotado en Fase 2
  bloque 1, tras banner naranja de Cal.com al terminar Fase 1).
- **OAuth client_secret:** guardado en gestor de contrasenas seguro. NO
  descargado como JSON por politica de seguridad institucional del equipo
  AVE (bloqueada la descarga de JSON en Windows corporativo).
- **HMAC secret 64 chars hex:** almacenado como constante dentro del nodo
  `Validate_HMAC`. Feature de Variables de n8n es Enterprise, no
  disponible en community self-hosted. Rotable en cualquier momento
  editando el nodo + regenerando el secret en el webhook Cal.com.
- **Postgres calcom-postgres:** aislado en red interna `calcom_internal`,
  no expone puerto al host.
- **/opt/calcom/.env:** permisos `600` (solo root puede leer).
- **Modo Testing en OAuth Google Cloud:** limita a 100 emails predefinidos.
  Buen control de superficie hasta que AVE escale.

---

## 14. TODO pendientes al cierre de Fase 3

### Prioridad ALTA - Seguridad (heredados de Fases 1 y 2)

- [ ] Rotar password buzon `info@identechnology.co` en panel Hover
- [ ] Actualizar `SMTP_PASSWORD` en `/opt/calcom/.env` con nuevo password
- [ ] Reiniciar: `cd /opt/calcom && docker compose restart calcom`
- [ ] Verificar envio SMTP post-rotacion con reserva de prueba
- [ ] Rotar password cuenta Hover (usuario `enuar`)
- [ ] Activar 2FA en cuenta Hover
- [ ] Rotar password Postgres n8n (expuesto en
      `/opt/n8n/docker-compose.yml` y `/opt/n8n/.env`)

### Prioridad MEDIA - Warnings no bloqueantes Cal.com (detectados en logs)

- [ ] Crear Availability Schedule explicito para user `enuar` (elimina
      warning `No schedules found for user` - TRPCError repetido en logs)
- [ ] Agregar variable `ALLOWED_HOSTNAMES=citas.identechnology.co` al
      `docker-compose.yml` (elimina warning
      `Match of WEBAPP_URL with ALLOWED_HOSTNAMES failed`)
- [ ] Descartar event types por defecto del user personal `enuar`
      (Reunion de 15 min, 30 min, Secreta) si no se van a usar

### Prioridad MEDIA - Operativo

- [ ] Backup automatizado del volumen `calcom_pgdata` (integrar con
      backup script existente del VPS)
- [ ] Configurar recordatorio WhatsApp desde n8n aprovechando el sistema
      de recordatorios blindado existente (contador secuencial
      `recordatorios_enviados`) para citas de Cal.com. Actualmente Cal.com
      envia recordatorios por EMAIL pero no por WhatsApp.
- [ ] Cuando AVE supere 100 usuarios de prueba OAuth: iniciar proceso de
      publicacion/verificacion oficial en Google Cloud Console (2-6 semanas).

---

## 15. Roadmap Fase 4 - Onboarding tenants adicionales

Cuando algun cliente (Uhane SAS, PC_Outlet, tienda4030 o nuevo)
requiera agendamiento, seguir estos 6 pasos:

### Paso 1: Google Cloud Console

Mientras la app OAuth siga en modo Testing (limite 100 emails):
- Agregar email Google del owner del tenant en:
  `Google Auth Platform -> Publico -> Usuarios de prueba`

### Paso 2: Cal.com - Crear team

- **Teams -> +New team**
- Nombre visible: `<Nombre Cliente>`
- Slug: consistente y URL-safe (`uhane`, `pcoutlet`, `tienda4030`)
- Agregar owner del cliente como member

### Paso 3: Postgres - Mapeo

```sql
UPDATE public.empresas
SET calcom_team_slug = '<slug>'
WHERE id = <empresa_id>;
```

### Paso 4: Cal.com - Webhook por team

- **Settings -> Developer -> Webhooks -> +New webhook**
- Seleccionar team correspondiente
- Subscriber URL: `https://hn8n.identechnology.co/webhook/calcom-booking`
  (misma que agencIA)
- 4 triggers estandar: `Booking created`, `Booking rescheduled`,
  `Booking canceled`, `Meeting ended`
- Secret: **usar EL MISMO secret HMAC** ya configurado en n8n
  `Validate_HMAC` (para no romper la validacion)
- Payload template: default

### Paso 5: Cal.com - Event types del team

Segun necesidad del cliente:
- Consulta 30min, Demo 45min, Discovery call 60min, etc.
- Location: **Google Meet**
- Toggle activo
- El owner del team conecta su propio Google Calendar via Install app
  (usa OAuth Testing con email autorizado en Paso 1)

### Paso 6: Test end-to-end

1. Reserva en `https://citas.identechnology.co/team/<slug>/<event-type-slug>`
2. Verificar ejecucion en n8n Executions (workflow `calcom-booking-handler`)
3. Verificar cita en tabla `citas` con `calcom_team_slug='<slug>'`
4. Verificar lead con `origen='calcom'` asociado a `empresa_id` correcto

---

## 16. Referencias + Comandos de mantenimiento

### 16.1 Comandos de mantenimiento

```bash
# Ver estado del stack
cd /opt/calcom && docker compose ps

# Ver logs recientes
docker compose logs --tail=100 calcom

# Ver logs SMTP filtrados (util para debug de envio de correos)
docker logs calcom --since 30m 2>&1 | grep -iE "smtp|email|mail"

# Backup manual del volumen Postgres
docker exec calcom-postgres pg_dump -U calcom calcom \
  > /root/backups/calcom-$(date +%Y%m%d-%H%M%S).sql

# Restaurar (cuidado en produccion)
docker exec -i calcom-postgres psql -U calcom calcom \
  < /root/backups/calcom-YYYYMMDD-HHMMSS.sql

# Entrar a la BD interactivamente
docker exec -it calcom-postgres psql -U calcom -d calcom

# Reiniciar solo Cal.com (tras cambios en .env o compose)
cd /opt/calcom && docker compose restart calcom

# Reiniciar n8n (tras cambios en .env)
cd /opt/n8n && docker compose up -d n8n

# Verificar variables inyectadas al contenedor Cal.com
docker exec calcom env | grep EMAIL_SERVER_HOST

# Actualizar Cal.com a la version mas reciente
cd /opt/calcom
docker compose pull
docker compose down
docker compose up -d
```

### 16.2 Queries SQL frecuentes

**Ver citas del tenant agencIA (ultimas 20):**
```sql
SELECT id, calcom_booking_id, event_type_slug, fecha_inicio,
       estado, meeting_url, created_at
FROM public.citas
WHERE calcom_team_slug = 'agencia'
ORDER BY created_at DESC
LIMIT 20;
```

**Ver leads creados por Cal.com:**
```sql
SELECT id, nombre, email, telefono, estado_lead, ultima_interaccion
FROM public.leads
WHERE origen = 'calcom'
ORDER BY updated_at DESC
LIMIT 20;
```

**Contar citas por estado y tenant:**
```sql
SELECT calcom_team_slug, estado, COUNT(*) AS total
FROM public.citas
GROUP BY calcom_team_slug, estado
ORDER BY calcom_team_slug, estado;
```

### 16.3 Ver ejecuciones del workflow n8n

- URL: https://hn8n.identechnology.co
- Workflow: `calcom-booking-handler` -> pestana **Executions**

### 16.4 Referencias

- Cal.com repo oficial: https://github.com/calcom/cal.com
- Imagen DockerHub oficial: https://hub.docker.com/r/calcom/cal.com
- Docs self-hosting: https://cal.com/docs/self-hosting
- Cal.com v6.4 blog (cambio de licencia):
  https://cal.com/blog/calcom-v6-4
- Cal.diy community edition MIT (sin Teams): https://www.cal.diy/
- Traefik v2.11 docs (labels Docker):
  https://doc.traefik.io/traefik/v2.11/
- Let's Encrypt ACME challenge:
  https://letsencrypt.org/docs/challenge-types/
- Google Cloud Console: https://console.cloud.google.com
- Google Calendar API:
  https://developers.google.com/calendar/api/guides/overview
- OAuth 2.0 for Google APIs:
  https://developers.google.com/identity/protocols/oauth2
- n8n task runner NODE_FUNCTION_ALLOW_BUILTIN:
  https://docs.n8n.io/hosting/configuration/environment-variables/deployment/

---

**Fin del modulo. Fase 1 + Fase 2 + Fase 3 COMPLETADAS.**

**Siguiente sesion:** Fase 4 - Onboarding tenants segun demanda de clientes.
