# Modulo: Panel Cliente Appsmith Multi-tenant

**Proyecto:** AVE - agencIA
**Fase 1 (fork y setup):** 12 jul 2026 - COMPLETADA
**Fase 2 (SMTP Hover + workspace):** 12 jul 2026 - COMPLETADA
**Fase 3 (vista con precedencia):** 13 jul 2026 - COMPLETADA
**Version Appsmith:** Community Edition v1.9.x self-hosted
**URL productiva:** https://admin.identechnology.co
**Workspaces:**
- `enuar's apps` (admin, privado del operador AVE)
- `SOMI-4030` (workspace de tienda4030)
- Futuros workspaces por tenant
**Estado:** OPERATIVO - cada tenant accede a su Panel Cliente con aislamiento multi-tenant

---

## 1. Contexto y decision de arquitectura

El Panel Cliente Appsmith es la interfaz de auto-gestion que AVE entrega a cada tenant para que administre su bot IA de WhatsApp sin necesidad de intervencion del equipo AVE. Antes de este panel, cualquier ajuste al bot (cambio de tono, actualizacion de servicios, edicion de recordatorios, rotacion de API key OpenAI) requeria acceso directo a la base de datos y coordinacion con el operador AVE.

Se elige esta arquitectura sobre alternativas por:

- **vs desarrollo custom en React/Next.js:** Appsmith self-hosted es AGPLv3, USD 0/mes, con templating declarativo y datasources nativos a PostgreSQL sin necesidad de mantener backend intermedio.
- **vs Retool Cloud:** self-hosted preserva datos en Colombia (compliance), URL propia branded y sin limite de tenants.
- **vs Panel Admin extendido:** el Panel Admin es interno del operador AVE con vista sobre todos los tenants. Exponer el mismo panel a clientes rompe el aislamiento multi-tenant y expone informacion sensible de otros clientes. Se descarta.

**Arquitectura multi-tenant elegida:** un solo Appsmith server, un workspace por tenant, y una vista PostgreSQL `v_usuario_empresa` como fuente de verdad para el mapeo usuario-empresa. Cada usuario logueado ve unicamente los datos de su empresa gracias a filtros dinamicos en todas las queries.

---

## 2. Prerequisitos infraestructura

VPS: Hostinger `srv1732020`, IP publica `2.25.172.227`.

| Recurso                | Valor                                       | Estado |
|------------------------|---------------------------------------------|--------|
| Appsmith               | Community Edition self-hosted               | OK     |
| Contenedor Docker      | `appsmith-appsmith-1`                       | OK     |
| Ruta configuracion     | `/opt/appsmith/appsmith-stacks/configuration/docker.env` | OK |
| Traefik SSL            | Let's Encrypt `mytlschallenge` resolver     | OK     |
| DNS                    | `admin.identechnology.co` -> 2.25.172.227   | OK     |
| SMTP                   | Hover `smtp.hostedemail.com:587` STARTTLS   | OK     |
| Postgres datasource    | `n8n-postgres-1` (BD `postgres`)            | OK     |

---

## 3. DNS y estructura de archivos

### 3.1 DNS

Panel Hover -> zona DNS `identechnology.co`:

```
Tipo:   A
Nombre: admin
Valor:  2.25.172.227
TTL:    300
```

Verificacion:
```
dig +short admin.identechnology.co
# -> 2.25.172.227
```

### 3.2 `/opt/appsmith/appsmith-stacks/configuration/docker.env`

**Variables criticas para el correcto funcionamiento:**

```env
# URL canonica del panel para invitaciones y reset password
APPSMITH_BASE_URL=https://admin.identechnology.co

# SMTP Hover para envio de invitaciones y correos internos
APPSMITH_MAIL_ENABLED=true
APPSMITH_MAIL_HOST=smtp.hostedemail.com
APPSMITH_MAIL_PORT=587
APPSMITH_MAIL_SMTP_TLS_ENABLED=true
APPSMITH_MAIL_SMTP_AUTH=true
APPSMITH_MAIL_USERNAME=info@identechnology.co
APPSMITH_MAIL_PASSWORD=<REDACTED_HOVER_MAILBOX_PASSWORD>
APPSMITH_MAIL_FROM=info@identechnology.co
APPSMITH_REPLY_TO=info@identechnology.co
```

Permisos: `chmod 600` (solo root puede leer).

**Notas importantes:**
- La documentacion oficial de Appsmith solo reconoce `APPSMITH_MAIL_HOST`, `APPSMITH_MAIL_PORT`, `APPSMITH_MAIL_SMTP_TLS_ENABLED`, `APPSMITH_MAIL_SMTP_AUTH`, `APPSMITH_MAIL_USERNAME`, `APPSMITH_MAIL_PASSWORD`, `APPSMITH_MAIL_FROM`, `APPSMITH_REPLY_TO`. Las variables `APPSMITH_MAIL_SMTP_STARTTLS_ENABLE` y `APPSMITH_MAIL_SMTP_SSL_ENABLE` son INVENTADAS y no funcionan en Appsmith Community Edition.
- El campo `APPSMITH_MAIL_FROM` debe ser un email valido sin nombre visible entre comillas. Appsmith CE valida como email puro. Si se pone `AVE agencIA <info@identechnology.co>` genera error de validacion.
- Cuando se configura SMTP en `docker.env`, las variables tienen precedencia sobre la configuracion UI en Settings > Email. Esto significa que la UI queda como referencia visual pero no puede sobrescribir el archivo.

---

## 4. Aplicar cambios al contenedor

Despues de editar `docker.env`, el comando `docker compose restart` NO relee el archivo. Es necesario forzar recreacion del contenedor:

```bash
cd /opt/appsmith
docker compose up -d --force-recreate

# Espera 30-60 segundos para que Appsmith arranque

docker exec appsmith-appsmith-1 env | grep APPSMITH_MAIL
docker exec appsmith-appsmith-1 env | grep APPSMITH_BASE_URL
```

Las variables deben aparecer listadas correctamente. Si no, revisar sintaxis del `docker.env` (especialmente caracteres especiales en passwords).

---

## 5. Estructura de tablas en PostgreSQL AVE

### 5.1 Tabla `usuarios_cliente`

```sql
CREATE TABLE public.usuarios_cliente (
  id            SERIAL PRIMARY KEY,
  email         VARCHAR(255) NOT NULL,
  empresa_id    INTEGER NOT NULL REFERENCES empresas(id),
  nombre        VARCHAR(200),
  rol           VARCHAR(50) DEFAULT 'admin_cliente',
  activo        BOOLEAN DEFAULT true,
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  updated_at    TIMESTAMPTZ DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_usuarios_cliente_email
  ON usuarios_cliente(email) WHERE activo = true;
```

Es la fuente principal de mapeo email-empresa. Un mismo email solo puede estar activo en una empresa a la vez.

### 5.2 Vista `v_usuario_empresa`

Fuente unica que Appsmith consulta desde `q_get_usuario_actual`. Combina dos fuentes con precedencia explicita:

```sql
CREATE VIEW public.v_usuario_empresa AS
-- Fuente 1: usuarios_cliente tiene PRECEDENCIA total
SELECT 
    uc.email::character varying AS email,
    uc.empresa_id,
    uc.nombre::character varying AS usuario_nombre,
    uc.rol::character varying AS rol,
    e.nombre::character varying AS empresa_nombre
FROM public.usuarios_cliente uc
JOIN public.empresas e ON e.id = uc.empresa_id
WHERE uc.activo = true

UNION ALL

-- Fuente 2: empresas.email SOLO si el email NO está en usuarios_cliente
SELECT 
    e.email::character varying AS email,
    e.id AS empresa_id,
    ('Contacto principal de ' || e.nombre::text)::character varying AS usuario_nombre,
    'admin_cliente'::character varying AS rol,
    e.nombre::character varying AS empresa_nombre
FROM public.empresas e
WHERE e.activo = true
  AND e.email IS NOT NULL
  AND e.email::text <> ''::text
  AND e.email::text NOT IN (
      SELECT uc.email::text 
      FROM public.usuarios_cliente uc
      WHERE uc.activo = true
  );
```

**Regla de precedencia:** si un email esta registrado en `usuarios_cliente` como activo, ese registro es la fuente autoritativa. La columna `empresas.email` solo da acceso automatico cuando el email NO existe en `usuarios_cliente`.

**Este diseño es el resultado del Bug 47** documentado en `troubleshooting/bugs-resueltos.md`. La version original usaba `UNION` sin filtro `NOT IN`, y cuando un email estaba en ambas fuentes con distintos `empresa_id`, la vista devolvia 2 filas y Appsmith elegia una de forma no determinística.

### 5.3 Tabla `empresas`

Ya existente en el proyecto AVE. Se usa la columna `email` como fuente secundaria de mapeo. Un email de contacto de una empresa da acceso automatico al Panel Cliente de esa empresa siempre y cuando el email no este previamente registrado en `usuarios_cliente`.

---

## 6. Workspaces en Appsmith

### 6.1 Workspace administrador (`enuar's apps`)

Uso exclusivo del operador AVE. Contiene:
- `AVE - Panel Admin` (interno, con vista global de todos los tenants)
- Otras apps administrativas

**No se comparte con clientes.** Los clientes solo tienen acceso a los workspaces por tenant.

### 6.2 Workspace por tenant

Cada tenant productivo tiene su propio workspace en Appsmith:
- Workspace `SOMI-4030` para tienda4030
- Workspaces futuros para agencIA, Uhane SAS, PC_Outlet segun necesidad comercial

Dentro del workspace del tenant:
- Una sola app "Panel Cliente" (forkeada desde una plantilla base)
- Usuarios invitados con rol **App Viewer** (permisos minimos)

### 6.3 Roles en Appsmith Community Edition

Los roles disponibles son:

- **Administrator** - Puede editar apps, invitar usuarios, eliminar apps. Solo para el operador AVE.
- **Developer** - Puede editar apps e invitar. No usar con clientes.
- **App Viewer** - Solo visualiza apps del workspace, no puede editar. Rol correcto para clientes.
- **Custom role** - Requiere Business Plan de pago. No disponible en CE.

**Regla de asignacion:** todo cliente invitado a su workspace debe tener rol **App Viewer** para prevenir modificaciones accidentales a la app o invitaciones cruzadas a otros clientes.

---

## 7. Estructura del Panel Cliente

Cada Panel Cliente tiene las siguientes paginas:

### 7.1 Dashboard

KPIs y graficas del tenant:
- Leads nuevos, activos, con recordatorios enviados
- Conversiones y tasa de conversion
- Distribucion por prioridad (bajo, medio, alto, urgente)
- Distribucion por etiqueta (frio, tibio, caliente, exitoso)
- Filtros de rango de fechas

**Queries:** `get_leads_dashboard`, `get_etiquetas_pipeline_dashboard`, `get_prioridad_dashboard`

### 7.2 Mi Empresa

Datos generales de la empresa:
- Nombre, plan, email de contacto, telefono
- Estado activo/inactivo (readonly)
- Formulario de edicion

**Queries:** `get_empresa`, `update_empresa`

### 7.3 Info Negocio

Contenido conversacional que el bot usa como fuente de verdad:
- Tabla de categorias (precios, envios, caracteristicas, contacto, etc.)
- Servicios/productos con precio y descripcion
- Media asociada a cada servicio (imagenes, videos, PDFs)
- Botones de upload por tipo de media

**Queries:** `get_info_negocio`, `get_servicios`, `get_media_servicio`, `upload_imagen_media`, `upload_video_media`, `upload_pdf_media`

**Datasource adicional:** `AVE_API` (backend Node.js custom para uploads binarios)

### 7.4 Config Bot

Configuracion tecnica y comercial del bot IA:
- Objetivo del bot, tono, personalidad
- Modelo LLM (gpt-4o-mini, gpt-5-mini, etc.)
- Temperatura y max tokens
- API key OpenAI (con estado, fingerprint, boton probar)
- System prompt (Capa 2 editable)
- Prompt clasificador
- Tabla de etiquetas del pipeline (colores, prioridad Chatwoot)
- Tabla de etiquetas operativas (agendamiento, compra, urgente)

**Queries:** `get_pipeline`, `update_pipeline`, `get_etiquetas`, `insert_etiqueta`, `update_etiqueta`, `delete_etiqueta`

**Nota importante sobre Capa 2:** el widget del "System Prompt" muestra `bot_config.prompt_capa_cliente` (Capa 2 editable), NO `bot_config.system_prompt` viejo. Ver documento `arquitectura-prompts-sandwich.md` para detalle de la arquitectura de 3 capas.

### 7.5 Config Recordatorios

Configuracion de mensajes automaticos de seguimiento:
- Recordatorio a las X horas post-conversacion
- Recordatorio de cita (agendamiento activo)
- Toggle de activacion global

**Queries:** `get_recordatorios_config`, `update_recordatorios_config`

---

## 8. Query base multi-tenant `q_get_usuario_actual`

Es la query raiz del mecanismo multi-tenant. Todas las demas queries dependen de su resultado. Se ejecuta on-page-load de forma automatica.

```sql
SELECT 
    email,
    empresa_id,
    usuario_nombre,
    rol,
    empresa_nombre
FROM public.v_usuario_empresa
WHERE email = '{{appsmith.user.email}}'
LIMIT 1;
```

**Expresion dinamica critica:** `{{appsmith.user.email}}` resuelve al email del usuario logueado en Appsmith. Es la unica fuente valida. **NUNCA reemplazarlo por un email literal**, aun temporalmente para debug. Ver Bug 49.

Todas las queries que filtran por tenant deben usar:

```sql
WHERE empresa_id = {{q_get_usuario_actual.data[0].empresa_id}}
```

**Ningun filtro por tenant debe usar un valor literal** como `WHERE empresa_id = 1`. Ver Bug 48.

---

## 9. Fixes tecnicos aplicados durante el desarrollo

### 9.1 Fix APPSMITH_BASE_URL

**Sintoma:** al invitar un usuario nuevo, Appsmith registraba el usuario correctamente pero NO enviaba el correo de invitacion. Log del contenedor mostraba error `Cannot generate token-bearing email flow, canonical instance URL not configured`.

**Causa:** ausencia de `APPSMITH_BASE_URL` en `docker.env`. Appsmith no puede generar links de invitacion sin conocer la URL canonica del panel.

**Fix:** agregar variable en `docker.env` y recrear contenedor:

```bash
echo "APPSMITH_BASE_URL=https://admin.identechnology.co" >> /opt/appsmith/appsmith-stacks/configuration/docker.env
cd /opt/appsmith && docker compose up -d --force-recreate
```

### 9.2 Fix SMTP Hover con puerto 587 STARTTLS

**Sintoma inicial:** al configurar SMTP con `smtp.hostedemail.com:465` (puerto SSL directo), Appsmith rechazaba con "Bad request: Failed to connect to the SMTP server".

**Causa:** Appsmith Community Edition tiene limitaciones con SSL directo en puerto 465. El motor SMTP interno prefiere STARTTLS con puerto 587.

**Fix:** cambiar puerto a 587 y confirmar que el TLS toggle esta ON (Appsmith interpreta ON como STARTTLS en puerto 587).

Verificacion de que Hover acepta puerto 587 con STARTTLS:

```bash
docker exec appsmith-appsmith-1 curl -v \
  --url "smtp://smtp.hostedemail.com:587" \
  --mail-from "info@identechnology.co" \
  --mail-rcpt "TU_CORREO_DE_PRUEBA@gmail.com" \
  --user "info@identechnology.co:CLAVE_DEL_BUZON" \
  --ssl-reqd
```

Salida esperada: `Authentication successful` y `250 CHUNKING`. El `502 VRFY command is disabled` que aparece al final es ruido normal de Hover y no impide el envio real.

### 9.3 Fix APPSMITH_MAIL_FROM formato

**Sintoma:** al configurar `APPSMITH_MAIL_FROM="AVE agencIA <info@identechnology.co>"` con nombre y email en formato RFC 5322, el UI Settings > Email rechazaba con "Validation Failure: fromEmail must be a well-formed email address".

**Causa:** Appsmith Community Edition solo acepta el email puro en el campo From, sin nombre visible entre chevrones. La feature "From name" requiere Business Plan.

**Fix:** cambiar el valor a solo el email puro:

```env
APPSMITH_MAIL_FROM=info@identechnology.co
```

Los correos enviados apareceran con "info@identechnology.co" como remitente sin nombre visible amigable. Es aceptable para invitaciones y reset password.

### 9.4 Fix queries hardcodeadas post-fork

Ver Bugs 48 y 49 en `bugs-resueltos.md`.

Al forkear una app Appsmith para crear un nuevo tenant, ejecutar auditoria manual de todas las queries buscando:
- Filtros literales tipo `WHERE empresa_id = 1`
- Emails literales tipo `WHERE email = 'admin@algo.com'`
- Referencias a widgets del app original que ya no existen

Reemplazar por:
- `{{q_get_usuario_actual.data[0].empresa_id}}` para filtros por empresa
- `{{appsmith.user.email}}` para filtros por email logueado

---

## 10. Onboarding de un nuevo tenant al Panel Cliente

Este es el procedimiento estandar cuando AVE captura un cliente nuevo y le entrega el Panel Cliente. Duracion aproximada: 15-20 minutos.

### 10.1 Preparar el registro de la empresa en PostgreSQL

Con acceso a DBeaver o Panel Admin, insertar la nueva empresa con sus datos basicos:

```sql
INSERT INTO empresas (nombre, plan, email, telefono, activo)
VALUES (
    'Nombre Comercial del Cliente',
    'basic',
    'contacto@cliente.com',
    '3001234567',
    true
) RETURNING id;
```

Anotar el `id` retornado, se usara como `empresa_id` en los siguientes pasos.

### 10.2 Insertar configuracion base en `bot_config`

```sql
INSERT INTO bot_config (
    empresa_id, 
    objetivo_bot, 
    tono, 
    modelo_bot, 
    modelo_clasificador,
    temperatura_bot, 
    max_tokens_bot, 
    openai_api_key,
    openai_key_status, 
    activo
) VALUES (
    <ID_EMPRESA>,
    'Placeholder de objetivo - editar despues desde Panel Cliente',
    'Placeholder de tono',
    'gpt-4o-mini',
    'gpt-4o-mini',
    0.7,
    1500,
    NULL,
    'no_config',
    true
);
```

### 10.3 Registrar el email de acceso en `usuarios_cliente`

Este es el paso critico que asegura el aislamiento multi-tenant.

```sql
INSERT INTO usuarios_cliente (
    email, 
    empresa_id, 
    nombre, 
    rol, 
    activo
) VALUES (
    'cliente@empresa.com',
    <ID_EMPRESA>,
    'Nombre del Contacto',
    'admin_cliente',
    true
);
```

Verificacion inmediata:

```sql
SELECT * FROM v_usuario_empresa WHERE email = 'cliente@empresa.com';
```

Debe devolver UNA sola fila con el `empresa_id` correcto.

### 10.4 Crear workspace y forkear app en Appsmith

Con acceso de administrador Appsmith:

1. Crear un workspace nuevo con el nombre del tenant (ej: `MI-EMPRESA`)
2. Ir al workspace `enuar's apps` y localizar la app "Panel Cliente" plantilla
3. Forkear la app hacia el workspace nuevo del tenant
4. Renombrar la app forkeada a "Panel Cliente" a secas
5. Auditar TODAS las queries buscando valores hardcodeados (ver 9.4)
6. Configurar datasources si es necesario (PostgreSQL AVE + AVE_API deberian heredarse del fork)

### 10.5 Invitar al usuario cliente

Dentro del workspace del tenant:

1. Ir a Settings > Members
2. Invite user
3. Ingresar email registrado en `usuarios_cliente`
4. Rol: **App Viewer**
5. Enviar invitacion

El cliente recibira un correo desde `info@identechnology.co` con link para crear su contraseña. El link redirige a `admin.identechnology.co` donde el cliente crea password y accede automaticamente al workspace de su tenant, sin ver otros workspaces ni el Panel Admin.

### 10.6 Verificacion end-to-end

Desde el navegador del cliente (o simulando con sesion incognito):

1. Acceder a `admin.identechnology.co`
2. Login con el email registrado
3. Verificar que solo ve el workspace de su tenant
4. Abrir el Panel Cliente
5. Verificar que Dashboard muestra sus datos, Config Bot muestra su configuracion, Mi Empresa muestra su empresa, etc.

Si algun tab muestra datos incorrectos, revisar en Appsmith que la query correspondiente use `{{q_get_usuario_actual.data[0].empresa_id}}` dinamicamente. Ver Bug 48.

---

## 11. Panel Cliente para pruebas con multiples tenants

En desarrollo, el operador AVE puede necesitar simular acceso a distintos Panel Cliente sin crear correos nuevos por cada tenant. Hay dos estrategias validas:

### Opcion A: Cambiar mapeo temporal en `usuarios_cliente`

```sql
-- Para simular acceso a tienda4030 desde tu email
UPDATE usuarios_cliente 
SET empresa_id = 9
WHERE email = 'operador@ave.com';

-- Al terminar la prueba, restaurar
UPDATE usuarios_cliente 
SET empresa_id = 1
WHERE email = 'operador@ave.com';
```

Simple y rapido para pruebas puntuales. **No usar en produccion.**

### Opcion B: Aliases de Gmail con `+`

Gmail acepta el patron `email+algo@gmail.com` como alias del mismo buzon. Registrar aliases distintos en Appsmith:

- `operador+agencia@gmail.com` -> mapear a empresa_id=1 (agencIA)
- `operador+uhane@gmail.com` -> mapear a empresa_id=7 (Uhane SAS)
- `operador+pcoutlet@gmail.com` -> mapear a empresa_id=8 (PC_Outlet)
- `operador+tienda@gmail.com` -> mapear a empresa_id=9 (tienda4030)

Todos los correos de invitacion llegan al mismo buzon real `operador@gmail.com` pero para Appsmith y para `usuarios_cliente` son emails distintos. Permite mantener sesiones abiertas en distintos navegadores/incognito sin interferencia. Mas limpio para debug prolongado.

---

## 12. Backup y rollback

### 12.1 Backup de la vista v_usuario_empresa

Antes de cualquier cambio a la vista:

```sql
CREATE OR REPLACE VIEW public.v_usuario_empresa_backup_YYYYMMDD AS
SELECT * FROM public.v_usuario_empresa;
```

Rollback en caso de error:

```sql
DROP VIEW IF EXISTS public.v_usuario_empresa CASCADE;
CREATE VIEW public.v_usuario_empresa AS 
SELECT * FROM public.v_usuario_empresa_backup_YYYYMMDD;
```

### 12.2 Backup de configuracion Appsmith

El estado de las apps Appsmith se persiste en `/opt/appsmith/appsmith-stacks/data/`. Hacer snapshot completo del stack antes de cambios mayores:

```bash
docker compose down
tar -czf /root/backups/appsmith-stacks-$(date +%Y%m%d).tar.gz /opt/appsmith/appsmith-stacks/
docker compose up -d
```

### 12.3 Backup del docker.env

```bash
cp /opt/appsmith/appsmith-stacks/configuration/docker.env \
   /opt/appsmith/appsmith-stacks/configuration/docker.env.bak-$(date +%Y%m%d)
```

---

## 13. Comandos de mantenimiento

```bash
# Ver estado del contenedor Appsmith
docker ps | grep appsmith

# Ver logs recientes
docker logs --tail=100 appsmith-appsmith-1

# Ver logs de correos SMTP
docker logs appsmith-appsmith-1 2>&1 | grep -iE "smtp|email|mail"

# Ver variables de entorno actuales del contenedor
docker exec appsmith-appsmith-1 env | grep APPSMITH

# Reiniciar solo Appsmith
cd /opt/appsmith && docker compose restart

# Aplicar cambios a docker.env (forzar recreacion)
cd /opt/appsmith && docker compose up -d --force-recreate

# Ver espacio en disco de los volumenes de Appsmith
du -sh /opt/appsmith/appsmith-stacks/
```

---

## 14. Queries SQL frecuentes de administracion

**Ver todos los usuarios_cliente y su mapeo:**
```sql
SELECT id, email, empresa_id, nombre, rol, activo, created_at
FROM usuarios_cliente
ORDER BY empresa_id, id;
```

**Ver la resolucion actual de la vista para un email:**
```sql
SELECT * FROM v_usuario_empresa WHERE email = '<EMAIL_A_VERIFICAR>';
```

**Ver todos los emails activos que tienen acceso al panel:**
```sql
SELECT email, empresa_id, empresa_nombre 
FROM v_usuario_empresa 
ORDER BY empresa_id;
```

**Deshabilitar temporalmente acceso de un usuario:**
```sql
UPDATE usuarios_cliente SET activo = false WHERE email = '<EMAIL>';
```

**Auditar duplicados potenciales en la vista:**
```sql
SELECT email, COUNT(*) 
FROM v_usuario_empresa 
GROUP BY email 
HAVING COUNT(*) > 1;
```

Debe devolver 0 filas. Si aparece algun duplicado, revisar Bug 47.

---

## 15. Referencias

- Appsmith self-hosted docs: https://docs.appsmith.com/
- Appsmith Email configuration: https://docs.appsmith.com/getting-started/setup/instance-configuration/email
- Documento arquitectura de prompts: `docs/modulos/arquitectura-prompts-sandwich.md`
- Documento de bugs resueltos: `docs/troubleshooting/bugs-resueltos.md`
- Documento OpenAI multi-tenant: `docs/modulos/openai-multitenant.md`
- Documento Cal.com self-hosted: `docs/modulos/calcom-instalacion.md`