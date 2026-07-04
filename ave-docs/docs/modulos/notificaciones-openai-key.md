# Módulo: Notificaciones de incidentes con la API key de OpenAI

> Cuando un tenant queda operativamente sin OpenAI (key vacía, inválida, sin saldo, revocada), el sistema debe detectarlo antes de invocar al agente, notificar a las tres partes involucradas (equipo humano vía nota privada + prioridad urgente en Chatwoot, administrador del sistema vía correo, cliente dueño del tenant vía correo), y actualizar el estado en BD para que el panel admin refleje la realidad. Todo con anti-flood en los canales de correo para no saturar bandejas.

## Objetivo

- Ningún prospecto queda "hablándole a la pared" cuando el bot no puede responder por falta de key.
- El equipo humano de atención sabe **por qué** el bot no respondió al entrar a cada conversación.
- El administrador del sistema se entera del incidente de forma proactiva, sin depender de reclamos del cliente.
- El cliente dueño del tenant recibe indicación de qué revisar (facturación, key rotada, key no cargada).
- Los correos no se disparan por cada mensaje del mismo tenant (anti-flood 30 min).
- El estado en `bot_config.openai_key_status` se actualiza en el mismo turno del incidente, tanto si el rechazo es estructural (formato de key) como en runtime (OpenAI dice 401/quota).

## Diseño del flujo

La detección se hace en un nodo `If_openai_key_valid` **justo después de `PG_get_empresa`** y **antes de invocar al agente**. Si la validación falla, se abre una rama de notificación en paralelo a la ruta normal del agente. Si la validación pasa, el flujo continúa al `AI_Agent_Principal` como siempre. En ambos escenarios de fallo (estructural o runtime), el estado en BD se actualiza automáticamente.

PG_get_empresa
   +-> If_openai_key_valid
   |      +-(TRUE)-> AI_Agent_Principal               [flujo normal]
   |      |             |
   |      |             +-(falla runtime)-> Code_analizar_error_openai
   |      |                                       v
   |      |                                 PG_update_openai_key_status
   |      |                                       v
   |      |                                 [dispara notificaciones]
   |      |
   |      +-(FALSE)-+-> Enviar_nota_privada_Chatwoot   [SIEMPRE, por conversacion]
   |                +-> Marcar_conversacion_urgente     [SIEMPRE, por conversacion]
   |                +-> PG_check_flood_30min
   |                |      +-> If_no_flood
   |                |             +-(sin flood)-+-> Enviar_email_admin -> PG_log_incidente
   |                |             |             +-> Enviar_email_cliente
   |                |             +-(con flood)-> (correos cortados)
   |                |
   |                +-> PG_update_status_desde_IF       [NUEVO - Bug 39]
   |
   +-> If_conversacion_nueva                            [conexion existente sin cambios]

### Distinción crítica: canales por-conversación vs. canales agregados

- **Por-conversación (sin anti-flood):** nota privada en Chatwoot + marcado urgente. Cero costo externo, información contextual por conversación, imprescindibles para trazabilidad operativa. Se disparan siempre. (Ver Bug 33.)
- **Agregados (con anti-flood 30 min):** correo admin + correo cliente. Notificaciones externas con costo/impacto en bandejas. Solo se disparan si no hubo notificación previa al mismo tenant en los últimos 30 minutos.
- **Estado en BD (sin anti-flood):** `PG_update_status_desde_IF` en la rama FALSE actualiza `openai_key_status` en cada turno independientemente del anti-flood - el estado siempre debe reflejar la realidad, no depender del ritmo de correos.

## Nodo `If_openai_key_valid`

Tres condiciones combinadas con `AND`:

| Condición | Operador | Valor esperado |
|---|---|---|
| `{{ $json.openai_api_key }}` | `notEmpty` | (no vacío) |
| `{{ $json.openai_api_key }}` | `startsWith` | `sk-` |
| `{{ $json.openai_key_status }}` | `equals` | `active` |

Cualquier falla en una de las tres tira la rama FALSE (incidente detectado).

Los cinco estados posibles de `bot_config.openai_key_status`:
- `active` - funcionando.
- `invalid` - rechazada por OpenAI o formato inválido.
- `no_saldo` - cuenta sin crédito.
- `revoked` - revocada manualmente por el admin (por morosidad, etc.).
- `no_config` - el cliente aún no subió su key.

## Query modificada de `PG_get_empresa`

Además de los campos originales, el SELECT devuelve cinco campos que este módulo necesita:

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

`destinatario_cliente` con `COALESCE(e.email, e.email_contacto)` cubre el caso de tenants que solo tienen uno de los dos correos poblado en la tabla `empresas`.

## Anti-flood (canales de correo)

### Nodo `PG_check_flood_30min`

    SELECT
      MAX(email_admin_enviado_at) AS ultimo_email_admin,
      COUNT(*) AS incidentes_recientes
    FROM openai_key_incidents
    WHERE empresa_id = {{ $('PG_get_empresa').item.json.id }}
      AND email_admin_enviado = true
      AND email_admin_enviado_at > NOW() - INTERVAL '30 minutes';

### Nodo `If_no_flood`

Condición: `{{ $json.incidentes_recientes }}` equals `0` (número).

- TRUE (0 incidentes recientes) -> dispara los dos correos y el `PG_log_incidente` en paralelo.
- FALSE (ya hubo notificación reciente) -> rama vacía, correos suprimidos.

La ventana anti-flood es **por-tenant, no por-conversación**: si el tenant tiene tres conversaciones distintas con el bot y las tres reciben mensajes dentro de la ventana, solo se envía un correo por tenant. Las notas privadas y el marcado urgente sí se aplican a cada conversación individual (ver Bug 33).

## Contenido de las notificaciones

### Nota privada en Chatwoot

Endpoint: `POST /api/v1/accounts/{id}/conversations/{id}/messages` con `"private": "true"`. Contenido dinámico según el `openai_key_status` del tenant:

    BOT DESACTIVADO AUTOMATICAMENTE

    Motivo: [segun openai_key_status]
    Detectado: [dd/MM/yyyy HH:mm]

    Acciones ejecutadas:
    - Correo enviado al administrador del sistema
    - Correo enviado al cliente ([destinatario_cliente])
    - Prioridad marcada como urgente

    Por favor, atienda esta conversacion manualmente hasta que el cliente resuelva el problema.

Mapeo de motivo:
- `no_config` -> "API key no configurada"
- `invalid` -> "API key inválida"
- `no_saldo` -> "Cuenta OpenAI sin saldo"
- otro -> "API key con problema"

### Correo al administrador

Destinatario: correo del admin del sistema (configurado en el nodo).
Asunto: `[agencIA] Cliente sin OpenAI key: [nombre_empresa]`
Cuerpo HTML: incluye empresa, ID, número de conversación afectada, timestamp, estado detectado, fingerprint de la key, y las acciones automáticas aplicadas.

El campo "Estado" del correo usa una expresión con patrón `isExecuted` + predicción del estado real, para evitar el problema de snapshots viejos (Bug 35) y la ausencia de `Code_analizar_error_openai` en la rama FALSE (Bug 38). Ver sección **Coherencia de estado** más abajo.

### Correo al cliente

Destinatario: `COALESCE(empresas.email, empresas.email_contacto)`.
Asunto: `Acción requerida: tu asistente de agencIA no está operativo`
Cuerpo HTML: explicación no-técnica del problema, tres posibles causas (key expirada, sin saldo, no cargada), pasos para verificar (link a `platform.openai.com/api-keys`) y aviso de que el equipo humano está atendiendo mientras se resuelve.

### Marcado urgente en Chatwoot

Endpoint: `PATCH /api/v1/accounts/{id}/conversations/{id}` con:

    {
      "priority": "urgent"
    }

Solo `priority`. NO se manda `custom_attributes` en este endpoint - Chatwoot los ignora silenciosamente (ver Bug 34). Además, por decisión de producto, no se fuerza `desactivar_bot: true` - cuando la key se restaure, el bot debe reanudar automáticamente al siguiente mensaje sin intervención manual.

## Coherencia de estado en los canales

Cuando el correo al admin y el log de incidentes leen el `openai_key_status` para reportar el motivo del incidente, hay tres complicaciones que aparecieron durante la implementación y que resolvimos con un patrón específico:

1. **Snapshot inmutable de `PG_get_empresa` (Bug 35)** - La referencia `$('PG_get_empresa').item.json.openai_key_status` devuelve el valor que la BD tenía al inicio del turno, no el que quedó después del auto-update.

2. **Nodo no ejecutado en rama alternativa (Bug 38)** - En el camino del auto-update, `Code_analizar_error_openai` sí ejecuta y tiene el `nuevo_status` correcto. En el camino de la rama FALSE del IF, `Code_analizar_error_openai` no ejecuta y n8n 2.23.3 rechaza la referencia con error.

3. **Estado aún no actualizado en BD (Bug 39)** - En la rama FALSE, `PG_update_status_desde_IF` corre en paralelo con las notificaciones, no antes. Si el correo lo lee del snapshot de `PG_get_empresa`, verá el valor viejo.

La solución adoptada es una **cascada de fallback con predicción del estado**, aplicada en el campo "Estado" del correo admin y en el campo `tipo_incidente` de `PG_log_incidente`:

    {{ $('Code_analizar_error_openai').isExecuted
       ? $('Code_analizar_error_openai').item.json.nuevo_status
       : (!$('PG_get_empresa').item.json.openai_api_key
            || $('PG_get_empresa').item.json.openai_api_key.length === 0
            ? 'no_config'
            : (!$('PG_get_empresa').item.json.openai_api_key.startsWith('sk-')
                  || $('PG_get_empresa').item.json.openai_api_key.length < 40
                  ? 'invalid'
                  : ($('PG_get_empresa').item.json.openai_key_status || 'desconocido'))) }}

Lo que hace la cascada:
1. Si vino por el camino del agente -> usa el `nuevo_status` que `Code_analizar_error_openai` calculó.
2. Si vino por el camino del IF -> replica la lógica de clasificación estructural en la propia expresión (predice el estado que `PG_update_status_desde_IF` va a poner en BD).
3. Fallback final: el status de BD tal cual, si no aplica ninguno de los anteriores.

De esta forma el correo, el log y la BD terminan diciendo lo mismo, sin depender del orden de ejecución de los nodos paralelos.

## Tabla `openai_key_incidents`

Fuente de verdad del historial de incidentes y motor del anti-flood.

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
| Password | App Password generada en `https://myaccount.google.com/apppasswords` |

Gmail personal limita a ~500 correos/día - suficiente para el volumen actual de incidentes esperado. Cuando se migre a correo corporativo, cambia solo el credencial, no la lógica del flujo.

## Sincronización con panel Appsmith

El panel `Config Bot` en Appsmith muestra el `openai_key_status` de cada tenant a través de un widget tipo badge con semáforo visual:

- `active` - verde (`#22c55e`)
- `invalid` - rojo (`#ef4444`)
- `no_saldo` - naranja (`#f59e0b`)
- `revoked` - gris (`#6b7280`)
- `no_config` - amarillo (`#eab308`)
- desconocido - gris claro (`#9ca3af`)

El badge se alimenta de `get_pipeline`, que ahora incluye `bc.openai_key_status` y `bc.openai_key_fingerprint` en su SELECT. Cada vez que el flujo n8n actualiza el status (por cualquiera de las tres fuentes), el badge en Appsmith refleja el cambio al recargar la query.

Cuando el admin edita la key desde el mismo panel, la query `update_pipeline` (UPSERT sobre `bot_config`) fuerza `openai_key_status = 'active'` y recalcula el fingerprint desde `EXCLUDED.openai_api_key`. Esto le da al cliente una oportunidad limpia sin depender de intervención manual del soporte.

## Comportamiento en restauración

Hay dos vías para restaurar un tenant tras un incidente. Ambas son compatibles con el sistema:

### Vía 1 - Cliente actualiza la key desde Appsmith (recomendado)

El cliente ingresa a su panel Config Bot, escribe la key nueva en el input y guarda. La query `update_pipeline` ejecuta:
1. Actualiza `openai_api_key` con la key nueva.
2. Fuerza `openai_key_status = 'active'`.
3. Recalcula `openai_key_fingerprint = RIGHT(openai_api_key, 4)`.

En el siguiente mensaje del prospecto:
1. `PG_get_empresa` trae la key nueva con status `active`.
2. `If_openai_key_valid` la deja pasar.
3. `AI_Agent_Principal` responde normal.

Si la key sigue mala (por ejemplo el cliente se equivocó), el auto-update de la Fuente 1 la volverá a marcar como `invalid` en el próximo turno.

### Vía 2 - Soporte actualiza la key directamente en BD

Cuando el soporte técnico corrige la key desde DBeaver (por ejemplo restaurando un caracter perdido, ver Bug 37), el UPDATE debe incluir explícitamente:

    UPDATE bot_config
    SET openai_api_key = '<key_nueva>',
        openai_key_status = 'active',
        openai_key_fingerprint = RIGHT('<key_nueva>', 4),
        updated_at = NOW()
    WHERE empresa_id = <id>;

Si solo se actualiza `openai_api_key` sin tocar `openai_key_status`, el flujo seguirá viendo el status viejo y bloqueará al tenant hasta que llegue un mensaje que dispare el auto-update.

**No es necesario limpiar `openai_key_incidents`** en ninguna de las dos vías - esa tabla es historial. El anti-flood se auto-desactiva porque en el nuevo turno la validación pasa y ni siquiera se llega al `PG_check_flood_30min`.

## Referencias

- Módulo base multi-tenant OpenAI: `openai-multitenant.md`
- Bugs relacionados: 31, 33, 34, 35, 36, 37, 38, 39.
- Comportamiento de `PATCH /conversations/{id}`: `../troubleshooting/bugs-resueltos.md`