# Modulo: Cal.com Self-Hosted - Instalacion + OAuth Google Calendar

**Proyecto:** AVE - agencIA
**Fase 1 (instalacion base):** 06 jul 2026 - COMPLETADA
**Fase 2 (OAuth Google Calendar):** 07 jul 2026 - COMPLETADA
**Version Cal.com:** v6.2.0-sh (imagen `calcom/cal.com:latest`)
**URL productiva:** https://citas.identechnology.co
**Estado:** OPERATIVO - integracion bidireccional con Google Calendar validada

---

## 1. Contexto y decision de arquitectura

Cal.com self-hosted reemplaza las tools de Google Calendar del flujo principal
de AVE. Se elige esta plataforma sobre alternativas por:

- **vs Calendly (pago):** Cal.com self-hosted es AGPLv3, USD 0/mes.
- **vs Cal.com Cloud (pago):** control total de datos, URL propia branded,
  sin limite de usuarios.
- **vs cal.diy (fork community):** cal.diy NO incluye Teams ni Organizations,
  imprescindibles para el modelo multi-tenant de AVE (agencIA, Uhane SAS,
  PC_Outlet, tienda4030). Se descarta.

**Decision de imagen:** oficial `calcom/cal.com:latest` desde DockerHub,
mantenida por el equipo de Cal.com desde nov-2025 (el repo `calcom/docker`
fue archivado; el Dockerfile y docker-compose viven ahora en el monorepo
`calcom/cal.com`).

**Multi-tenancy:** se usara la feature **Teams** (incluida en AGPLv3). Cada
empresa AVE sera un Team con sus propios event types, calendarios conectados,
webhooks y disponibilidad. Organizations (Enterprise) se evaluara solo si AVE
crece a mas de 50 clientes con necesidad de sub-dominios propios.

---

## 2. Prerequisitos VPS validados

VPS: Hostinger `srv1732020`, IP publica `2.25.172.227`.

| Recurso            | Valor                            | Estado |
|--------------------|----------------------------------|--------|
| Docker             | 29.5.3                           | OK     |
| Docker Compose     | v5.1.4 (plugin, sin guion)       | OK     |
| RAM total          | 15 Gi (11 Gi libres al arranque) | OK     |
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

## 4. Estructura de archivos - Fase 1

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

## 5. Comandos de arranque y verificacion

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

Los puertos NO se exponen al host; Traefik enruta desde
`citas.identechnology.co` a `calcom:3000` por la red interna
`traefik_network`.

---

## 6. Verificacion de acceso - Wizard inicial de Cal.com

URL: **https://citas.identechnology.co**

### Paso 1 - Usuario administrador
- Usuario: `enuar`
- Nombre Completo: `Enuar Emilio Rosales`
- Email: `enuaremiliorosales@gmail.com`
- Contrasena: >= 15 chars, mayus/minus, al menos 1 numero
  (generada con `openssl rand -base64 20 | tr -d '/+=' | head -c 22`)
- 2FA activado con Google Authenticator / Authy

### Paso 2 - Licencia
- Seleccion: **Licencia AGPLv3** (gratis)
- Accion: "Saltar paso" (equivale a aceptar AGPLv3 sin ingresar license
  key Enterprise)

### Paso 3 - Activar aplicaciones
- Todas las apps que requieren OAuth (Google Calendar, Google Meet, Zoom,
  Office 365) se dejan en **OFF**.
- La configuracion OAuth se realiza en Fase 2 (siguiente seccion).

### Verificacion Fase 1
- Panel admin accesible en `https://citas.identechnology.co/event-types`
- Pagina publica accesible en `https://citas.identechnology.co/enuar`
- SSL Let's Encrypt emitido automaticamente por Traefik
- 3 event types por defecto creados: `/enuar/30min`, `/enuar/15min`,
  `/enuar/secret` (este ultimo oculto)

---

## 7. Fase 2 - OAuth Google Calendar

**Objetivo:** que Cal.com pueda leer/escribir en el Google Calendar de
cualquier usuario/tenant. Cuando un cliente reserva, el evento se crea
automaticamente en el Google Calendar del propietario del calendar; y los
bloqueos manuales del calendar impiden que se reserven esos slots.

### 7.1 Google Cloud Console - Crear proyecto

1. URL: https://console.cloud.google.com
2. Loguearse con `enuaremiliorosales@gmail.com`
3. Selector de proyectos -> **NEW PROJECT**
4. Nombre: `ave-calcom-oauth`
5. **Sin activar billing** (Google Calendar API es gratis en el free tier
   hasta 1.000.000 requests/dia)

**Decision:** proyecto SEPARADO del existente `n8n-30-05-2026` por
separacion de responsabilidades, auditoria de accesos independiente, y
para que rotaciones de credenciales de un proyecto no afecten al otro.

### 7.2 Habilitar Google Calendar API

1. En el proyecto `ave-calcom-oauth`: menu -> **APIs y servicios ->
   Biblioteca**
2. Buscar: `Google Calendar API`
3. Click en la tarjeta -> **HABILITAR**
4. Verificar que el boton pase a "ADMINISTRAR" (estado: Habilitada)

### 7.3 Configurar Pantalla de consentimiento OAuth

Menu lateral: **Google Auth Platform -> Pantalla de consentimiento**

| Campo                | Valor                            |
|----------------------|----------------------------------|
| App name             | `AVE Citas`                      |
| User support email   | `enuaremiliorosales@gmail.com`   |
| Audience / User Type | **External**                     |
| Contact email        | `enuaremiliorosales@gmail.com`   |
| Estado publicacion   | **Testing** (NO publicar aun)    |

**Por que External y no Internal:** Internal requiere Google Workspace
organizacional. External permite que cualquier cuenta Google conecte
(limitada a usuarios de prueba en modo Testing).

### 7.4 Crear cliente OAuth 2.0

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

Ambos deben guardarse en **gestor de contrasenas** (Bitwarden / 1Password /
KeePassXC). El `client_secret` puede reobtenerse desde la vista de detalle
del cliente (Google no lo oculta despues de la creacion, a diferencia de
otros proveedores).

### 7.5 Agregar usuarios de prueba

Menu: **Google Auth Platform -> Publico -> Usuarios de prueba**

Emails iniciales (Fase 2):
- `enuaremiliorosales@gmail.com`
- `info@identechnology.co` (solo si tiene cuenta Google asociada)

Limite: **100 usuarios en modo Testing**. Cuando lleguen los tenants
reales (Fase 3), agregar sus emails Google aqui. Solo emails listados aqui
podran completar el flujo OAuth con la app en modo Testing.

### 7.6 Pegar credenciales en Cal.com

URL: **https://citas.identechnology.co/settings/admin/apps/calendar**

1. Buscar tarjeta **Google Calendar** -> click en icono lapiz
   (Editar claves)
2. Rellenar los 3 campos:

| Campo          | Valor                                                                    |
|----------------|--------------------------------------------------------------------------|
| `client_id`    | El generado en Google Cloud                                              |
| `client_secret`| El generado en Google Cloud                                              |
| `redirect_uris`| `https://citas.identechnology.co/api/integrations/googlecalendar/callback` |

3. Guardar
4. Activar el **toggle** de Google Calendar (queda en verde/ON)

### 7.7 Prueba OAuth end-to-end

1. Ir a **Ajustes -> Mi cuenta -> Calendarios** o URL directa
   `https://citas.identechnology.co/apps/installed/calendar`
2. En **App Store**, buscar Google Calendar -> **Install app**
3. Popup Google: loguearse con `enuaremiliorosales@gmail.com`
4. Aparece advertencia: **"Google no verificó esta app"**
   - Es esperado en modo Testing, NO es peligroso
   - Click **"Configuracion avanzada"** -> **"Ir a AVE Citas (no seguro)"**
5. Pantalla de permisos -> **Permitir**
6. Retorno automatico a Cal.com con Google Calendar conectado

**Verificaciones bidireccionales confirmadas en produccion (07 jul 2026):**

| Escenario                                    | Resultado |
|----------------------------------------------|-----------|
| Slot ocupado en Google Calendar              | Bloqueado automaticamente en Cal.com (OK) |
| Reserva creada en Cal.com                    | Evento aparece en Google Calendar (OK) |
| Cancelacion de cita en Cal.com               | Evento eliminado del Google Calendar (OK) |
| Reagendamiento en Cal.com                    | (Pendiente validar Fase 3)                |

---

## 8. Notas de seguridad

### 8.1 Secretos expuestos durante Fase 1 (rotar OBLIGATORIO)

Durante el proceso de instalacion Fase 1 se pegaron en el chat de
asistencia los siguientes secretos que deben considerarse
**comprometidos** y ser rotados:

- Password cuenta Hover (`enuar` -> panel hover.com)
- Password buzon `info@identechnology.co`

### 8.2 Password admin Cal.com

- Rotado a >= 15 caracteres el 07 jul 2026 (Bloque 1 de Fase 2)
- 2FA activado con Google Authenticator / Authy
- Codigos de recuperacion guardados en gestor seguro

### 8.3 client_secret OAuth Google

- Generado el 07 jul 2026 durante creacion del cliente OAuth
- Guardado en gestor seguro (NO descargado como JSON por politica de
  seguridad institucional del equipo)
- Puede rotarse en cualquier momento desde:
  Google Cloud Console -> Clientes -> AVE Citas - Cal.com -> Rotar secreto

### 8.4 Buenas practicas aplicadas

- `.env` con permisos `600` (solo root puede leer)
- Postgres de Cal.com aislado en red interna `calcom_internal`
- Cal.com no expone puertos al host (solo via Traefik)
- SSL obligatorio (HTTP redirige a HTTPS via Traefik)
- HSTS activo via Traefik
- Modo Testing en OAuth (control de que emails pueden conectar)

---

## 9. TODO abiertos al cierre de Fase 2

### Prioridad ALTA - Seguridad (pendientes de Fase 1)

- [ ] Rotar password buzon `info@identechnology.co` en panel Hover
- [ ] Actualizar `SMTP_PASSWORD` en `/opt/calcom/.env` con el nuevo password
- [ ] Reiniciar contenedor: `cd /opt/calcom && docker compose restart calcom`
- [ ] Verificar envio SMTP: reservar una cita de prueba y confirmar
      recepcion del correo
- [ ] Rotar password cuenta Hover (usuario `enuar`)
- [ ] Activar 2FA en cuenta Hover

### Prioridad MEDIA - Operativo

- [ ] Descartar los event types por defecto `Reunion de 15 min` y
      `Reunion de 30 min` si no se van a usar (o adaptarlos al estandar AVE)
- [ ] Configurar zona horaria y horario de disponibilidad estandar AVE
      (lun-vie 08:00-17:00 America/Bogota)
- [ ] Backup automatizado del volumen `calcom_pgdata` (integrar con backup
      script existente del VPS)
- [ ] Documentar procedimiento de restore desde backup

---

## 10. Roadmap Fase 3 - Multi-tenant + integracion AVE

**Objetivo:** configurar el modelo multi-tenant en Cal.com y conectarlo al
flujo principal de AVE (n8n + Postgres + Chatwoot).

### 10.1 Crear Teams (uno por empresa)

En Cal.com: **Equipos -> Crear equipo**

| Team slug     | Empresa      | empresa_id |
|---------------|--------------|------------|
| agencia       | agencIA      | 1          |
| uhane         | Uhane SAS    | 7          |
| pcoutlet      | PC_Outlet    | 8          |
| tienda4030    | tienda4030   | 9          |

Cada team tendra:
- URL propia: `https://citas.identechnology.co/<team>`
- Sus propios event types
- Sus propios miembros
- Su propio calendario Google conectado
- Su propio branding basico

### 10.2 Onboarding OAuth por tenant

Para cada tenant nuevo:
1. Obtener email Google del propietario del calendar (owner del team)
2. Google Cloud Console -> Publico -> Usuarios de prueba -> Agregar email
3. Owner del team se loguea en Cal.com, va a Ajustes -> Calendarios ->
   Install app (Google Calendar) -> autoriza con su cuenta
4. Verificar que aparece su calendar en Installed apps del team

### 10.3 Definir event types por team

Ejemplos por tipo de negocio:
- **agencIA:** Consulta estrategica 45min, Demo producto 30min, Discovery
  call 60min
- **Uhane SAS:** Sesion yoga privada 60min, Consulta wellness 30min
- **PC_Outlet:** Cotizacion equipo 20min, Soporte tecnico 30min
- **tienda4030:** Cita en tienda 15min

Configurar para cada uno:
- Duracion
- Buffer time (10min antes/despues)
- Minimo aviso previo (2h)
- Descripcion, ubicacion (Google Meet auto-generado o presencial)

### 10.4 Tabla `citas` en Postgres AVE

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
CREATE INDEX idx_citas_estado ON citas(estado);
```

### 10.5 Webhooks Cal.com -> n8n

En Cal.com -> **Ajustes -> Desarrollador -> Webhooks:**

- URL destino: `https://hn8n.identechnology.co/webhook/calcom-booking`
- Eventos suscritos:
  - `BOOKING_CREATED`
  - `BOOKING_RESCHEDULED`
  - `BOOKING_CANCELLED`
  - `MEETING_ENDED`
- Secret HMAC: generar en Cal.com y guardar en n8n para validar firma
- Formato: JSON POST

**Workflow n8n `calcom-booking-handler`:**

1. Nodo Webhook: recibe POST de Cal.com
2. Nodo Function: valida HMAC signature con el secret
3. Nodo Switch: identifica empresa por `team_slug` del payload
4. Nodo Postgres: inserta / actualiza registro en tabla `citas`
5. Nodo Postgres: actualiza `estado_lead` en tabla `leads` segun evento
   (BOOKING_CREATED -> `agendado`, CANCELLED -> `cancelado`, etc.)
6. Nodo HTTP Request: envia confirmacion por WhatsApp via Chatwoot API
7. Nodo Postgres: encola recordatorio 24h antes usando el sistema
   blindado con contador secuencial `recordatorios_enviados`

**Coherencia con el sistema existente:**
- Aprovechar el patron `isExecuted` de n8n 2.23.3 aplicado en
  workflows de openai-multitenant
- Refuerzo estructural en `update_pipeline` para prevenir keys mal
  formadas (ya aplicado en 43 bugs previos)
- Contador secuencial de recordatorios blindado (fix aplicado:
  quitar reset en flujo principal)

---

## 11. Roadmap Fase 4 (futuro) - OAuth adicional y publicacion

### 11.1 OAuth Zoom (opcional segun necesidad de tenants)

Cuando algun tenant requiera video-llamadas via Zoom (en lugar de Google
Meet por defecto):

1. URL: https://marketplace.zoom.us
2. Crear app tipo **OAuth (User-managed)**
3. Redirect URL:
   `https://citas.identechnology.co/api/integrations/zoom/callback`
4. Scopes: `meeting:write`, `meeting:read`
5. Pegar credenciales en Cal.com -> Apps -> Conferencing -> Zoom

### 11.2 OAuth Office 365 / Outlook (opcional para tenants MS365)

Solo si algun tenant corporativo lo requiere:

1. URL: https://portal.azure.com
2. Azure Active Directory -> App registrations -> New registration
3. Redirect URI:
   `https://citas.identechnology.co/api/integrations/office365calendar/callback`
4. API permissions: Calendars.ReadWrite (delegated)
5. Certificates & secrets -> New client secret

### 11.3 Publicacion oficial de la app OAuth (Google)

**Trigger:** cuando AVE supere ~50 clientes activos o requiera abrir el
sistema al publico general sin agregar cada email manualmente.

Requiere:
- Politica de privacidad publicada en dominio identechnology.co
- Terminos de servicio publicados
- Verificacion de propiedad del dominio (via Search Console)
- Justificacion escrita de por que se piden scopes sensibles de Calendar
- Proceso de revision de Google: 2-6 semanas

Beneficio: quitar la advertencia "Google no verifico esta app", eliminar
el limite de 100 usuarios de prueba, permitir onboarding self-service de
tenants sin intervencion manual en Google Cloud Console.

---

## 12. Comandos de mantenimiento

```bash
# Actualizar Cal.com a la version mas reciente
cd /opt/calcom
docker compose pull
docker compose down
docker compose up -d

# Ver logs del ultimo dia
docker compose logs --since 24h calcom

# Backup manual del volumen Postgres
docker exec calcom-postgres pg_dump -U calcom calcom > \
  /root/backups/calcom-$(date +%Y%m%d-%H%M%S).sql

# Restaurar (ejemplo, cuidado en produccion)
docker exec -i calcom-postgres psql -U calcom calcom < \
  /root/backups/calcom-YYYYMMDD-HHMMSS.sql

# Entrar a la BD interactivamente
docker exec -it calcom-postgres psql -U calcom -d calcom

# Ver uso de recursos del stack
docker stats calcom calcom-postgres

# Reiniciar solo el contenedor calcom (por ejemplo tras cambio en .env)
cd /opt/calcom && docker compose restart calcom

# Verificar que la app este saludable
curl -I https://citas.identechnology.co
# Esperar HTTP 200 o 307 (redirect a login)
```

---

## 13. Referencias

- Cal.com repo oficial: https://github.com/calcom/cal.com
- Imagen DockerHub oficial: https://hub.docker.com/r/calcom/cal.com
- Docs self-hosting: https://cal.com/docs/self-hosting
- Cal.com v5.9 (Docker oficial en monorepo):
  https://cal.com/blog/calcom-v5-9
- Traefik v2 docs (labels Docker):
  https://doc.traefik.io/traefik/v2.11/
- Let's Encrypt ACME challenge:
  https://letsencrypt.org/docs/challenge-types/
- Google Cloud Console: https://console.cloud.google.com
- Google Calendar API docs:
  https://developers.google.com/calendar/api/guides/overview
- OAuth 2.0 for Google APIs:
  https://developers.google.com/identity/protocols/oauth2

---

**Fin del documento. Fase 1 + Fase 2 COMPLETADAS.**
**Proximo paso: Fase 3 - Multi-tenant + integracion con AVE.**
