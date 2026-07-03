# Módulo: Notificaciones de incidentes con la API key de OpenAI

> Cuando un tenant queda operativamente sin OpenAI (key vacía, inválida, sin saldo, revocada), el sistema debe detectarlo antes de invocar al agente y notificar a las tres partes involucradas: el equipo humano de atención (nota privada en Chatwoot + prioridad urgente), el administrador del sistema (correo) y el cliente dueño del tenant (correo). Todo con anti-flood en los canales de correo para no saturar bandejas.

## Objetivo

- Ningún prospecto queda "hablándole a la pared" cuando el bot no puede responder por falta de key.
- El equipo humano de atención sabe **por qué** el bot no respondió al entrar a cada conversación.
- El administrador del sistema se entera del incidente de forma proactiva, sin depender de reclamos del cliente.
- El cliente dueño del tenant recibe indicación de qué revisar (facturación, key rotada, key no cargada).
- Los correos no se disparan por cada mensaje del mismo tenant (anti-flood 30 min).

## Diseño del flujo

La detección se hace en un nodo `If_openai_key_valid` **justo después de `PG_get_empresa`** y **antes de invocar al agente**. Si la validación falla, se abre una rama de notificación en paralelo a la ruta normal del agente. Si la validación pasa, el flujo continúa al `AI_Agent_Principal` como siempre.

```
PG_get_empresa
   ├─→ If_openai_key_valid
   │      ├─(TRUE)──→ AI_Agent_Principal              [flujo normal]
   │      │
   │      └─(FALSE)─┬─→ Enviar_nota_privada_Chatwoot  [SIEMPRE, por conversación]
   │                ├─→ Marcar_conversacion_urgente    [SIEMPRE, por conversación]
   │                └─→ PG_check_flood_30min
   │                       └─→ If_no_flood
   │                              ├─(sin flood)─┬─→ Enviar_email_admin → PG_log_incidente
   │                              │             └─→ Enviar_email_cliente
   │                              └─(con flood)─→ ⛔ (correos cortados)
   │
   └─→ If_conversacion_nueva                            [conexión existente sin cambios]
```

### Distinción crítica: canales por-conversación vs. canales agregados

- **Por-conversación (sin anti-flood):** nota privada en Chatwoot + marcado urgente. Cero costo externo, información contextual por conversación, imprescindibles para trazabilidad operativa. Se disparan siempre. (Ver Bug 33.)
- **Agregados (con anti-flood 30 min):** correo admin + correo cliente. Notificaciones externas con costo/impacto en bandejas. Solo se disparan si no hubo notificación previa al mismo tenant en los últimos 30 minutos.

## Nodo `If_openai_key_valid`

Tres condiciones combinadas con `AND`:

| Condición | Operador | Valor esperado |
|---|---|---|
| `{{ $json.openai_api_key }}` | `notEmpty` | (no vacío) |
| `{{ $json.openai_api_key }}` | `startsWith` | `sk-` |
| `{{ $json.openai_key_status }}` | `equals` | `active` |

Cualquier falla en una de las tres tira la rama FALSE (incidente detectado).

Los cinco estados posibles de `bot_config.openai_key_status`:
- `active` — funcionando.
- `invalid` — rechazada por OpenAI.
- `no_saldo` — cuenta sin crédito.
- `revoked` — revocada manualmente por el admin.
- `no_config` — el cliente aún no subió su key.

## Query modificada de `PG_get_empresa`

Además de los campos originales, el SELECT devuelve cinco campos que este módulo necesita:

```sql
SELECT
  e.id,
  e.nombre,
  e.chatwoot_url,
  e.chatwoot_api_key,
  e.chatwoot_account_id,
  e.email AS empresa_email,
  e.email_contacto AS empresa_email_contacto,
  COALESCE(e.email, e.email_contacto) AS destinatario_cliente,
  bc.openai_api_key,
  bc.openai_key_status,
  bc.openai_key_fingerprint,
  ...
```

`destinatario_cliente` con `COALESCE(e.email, e.email_contacto)` cubre el caso de tenants que solo tienen uno de los dos correos poblado en la tabla `empresas`.

## Anti-flood (canales de correo)

### Nodo `PG_check_flood_30min`

```sql
SELECT
  MAX(email_admin_enviado_at) AS ultimo_email_admin,
  COUNT(*) AS incidentes_recientes
FROM openai_key_incidents
WHERE empresa_id = {{ $('PG_get_empresa').item.json.id }}
  AND email_admin_enviado = true
  AND email_admin_enviado_at > NOW() - INTERVAL '30 minutes';
```

### Nodo `If_no_flood`

Condición: `{{ $json.incidentes_recientes }}` equals `0` (número).

- TRUE (0 incidentes recientes) → dispara los dos correos y el `PG_log_incidente` en paralelo.
- FALSE (ya hubo notificación reciente) → rama vacía, correos suprimidos.

La ventana anti-flood es **por-tenant, no por-conversación**: si el tenant tiene tres conversaciones distintas con el bot y las tres reciben mensajes dentro de la ventana, solo se envía un correo por tenant. Las notas privadas y el marcado urgente sí se aplican a cada conversación individual (ver Bug 33).

## Contenido de las notificaciones

### Nota privada en Chatwoot

Endpoint: `POST /api/v1/accounts/{id}/conversations/{id}/messages` con `"private": "true"`. Contenido dinámico según el `openai_key_status` del tenant:

```
🤖 ⚠️ BOT DESACTIVADO AUTOMÁTICAMENTE

Motivo: [según openai_key_status]
Detectado: [dd/MM/yyyy HH:mm]

Acciones ejecutadas:
✔ Correo enviado al administrador del sistema
✔ Correo enviado al cliente ([destinatario_cliente])
✔ Prioridad marcada como urgente

Por favor, atienda esta conversación manualmente hasta que el cliente resuelva el problema.
```

Mapeo de motivo:
- `no_config` → "API key no configurada"
- `invalid` → "API key inválida"
- `no_saldo` → "Cuenta OpenAI sin saldo"
- otro → "API key con problema"

### Correo al administrador

Destinatario: correo del admin del sistema (configurado en el nodo).
Asunto: `⚠️ [agencIA] Cliente sin OpenAI key: [nombre_empresa]`
Cuerpo HTML: incluye empresa, ID, número de conversación afectada, timestamp, estado detectado, fingerprint de la key, y las acciones automáticas aplicadas.

### Correo al cliente

Destinatario: `COALESCE(empresas.email, empresas.email_contacto)`.
Asunto: `Acción requerida: tu asistente de agencIA no está operativo`
Cuerpo HTML: explicación no-técnica del problema, tres posibles causas (key expirada, sin saldo, no cargada), pasos para verificar (link a `platform.openai.com/api-keys`) y aviso de que el equipo humano está atendiendo mientras se resuelve.

### Marcado urgente en Chatwoot

Endpoint: `PATCH /api/v1/accounts/{id}/conversations/{id}` con:
```json
{
  "priority": "urgent"
}
```

Solo `priority`. NO se manda `custom_attributes` en este endpoint — Chatwoot los ignora silenciosamente (ver Bug 34). Además, por decisión de producto, no se fuerza `desactivar_bot: true` — cuando la key se restaure, el bot debe reanudar automáticamente al siguiente mensaje sin intervención manual.

## Tabla `openai_key_incidents`

Fuente de verdad del historial de incidentes y motor del anti-flood.

```sql
CREATE TABLE openai_key_incidents (
    id                          BIGSERIAL PRIMARY KEY,
    empresa_id                  INTEGER NOT NULL REFERENCES empresas(id) ON DELETE CASCADE,
    conversation_id             BIGINT,
    tipo_incidente              VARCHAR(50) NOT NULL,
    motivo                      TEXT,
    key_fingerprint             VARCHAR(8),
    email_admin_enviado         BOOLEAN DEFAULT false,
    email_admin_enviado_at      TIMESTAMPTZ,
    email_cliente_enviado       BOOLEAN DEFAULT false,
    email_cliente_enviado_at    TIMESTAMPTZ,
    email_cliente_destino       VARCHAR(255),
    nota_chatwoot_enviada       BOOLEAN DEFAULT false,
    nota_chatwoot_enviada_at    TIMESTAMPTZ,
    created_at                  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_incidents_empresa_fecha
    ON openai_key_incidents(empresa_id, created_at DESC);

CREATE INDEX idx_incidents_email_admin_at
    ON openai_key_incidents(empresa_id, email_admin_enviado_at DESC)
    WHERE email_admin_enviado = true;
```

El índice parcial (`WHERE email_admin_enviado = true`) es el que hace eficiente la query del anti-flood.

## Configuración del credencial SMTP

Los nodos `Enviar_email_admin` y `Enviar_email_cliente` usan un credencial de tipo **SMTP** (no OAuth2). Configuración de arranque con Gmail:

| Campo | Valor |
|---|---|
| Credential Name | `Gmail SMTP - agencIA` |
| Host | `smtp.gmail.com` |
| Port | `465` |
| SSL/TLS | ON |
| User | correo Gmail del admin |
| Password | App Password generada en https://myaccount.google.com/apppasswords |

Gmail personal limita a ~500 correos/día — suficiente para el volumen actual de incidentes esperado. Cuando se migre a correo corporativo, cambia solo el credencial, no la lógica del flujo.

## Comportamiento en restauración

Cuando el cliente resuelve el problema (repone saldo, sube key nueva, activa la key) y en BD se cambia:
```sql
UPDATE bot_config SET openai_api_key = '<nueva>', openai_key_status = 'active' WHERE empresa_id = <id>;
```
El siguiente mensaje entrante del tenant:
1. Pasa por `If_openai_key_valid` — ahora las tres condiciones se cumplen.
2. Rama TRUE → va directo al `AI_Agent_Principal`.
3. El bot responde normal, sin intervención manual.

**No es necesario limpiar `openai_key_incidents`** — esa tabla es historial. El anti-flood se auto-desactiva porque en el nuevo turno la validación pasa y ni siquiera se llega al `PG_check_flood_30min`.

## Referencias

- Módulo base multi-tenant OpenAI: openai-multitenant.md
- Bugs relacionados: 31 (`=` en JSON body), 33 (anti-flood granular), 34 (custom_attributes en PATCH).
- Comportamiento de `PATCH /conversations/{id}`: ../troubleshooting/bugs-resueltos.md
