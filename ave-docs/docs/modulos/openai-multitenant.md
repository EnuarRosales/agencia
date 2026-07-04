# Módulo: Multi-tenant OpenAI

> Cada empresa cliente de AVE usa **su propia API key de OpenAI**, aislada del resto. Este módulo describe cómo se logra ese aislamiento en n8n (agente principal, clasificador y transcripción Whisper), cómo se autogestiona el estado operativo de cada key, y cómo el panel Appsmith y el flujo n8n mantienen ese estado sincronizado.

## Objetivo

- Cada tenant paga su propio consumo de OpenAI (facturación directa contra su cuenta).
- Un problema con la key de un tenant (revocada, sin saldo, no configurada) no afecta a los demás.
- No hay duplicación de workflows por tenant.
- Rotación de key por parte del cliente sin necesidad de tocar el workflow.
- El estado operativo de cada key (`active`, `invalid`, `no_saldo`, `no_config`, `revoked`) se mantiene sincronizado automáticamente entre el flujo n8n y el panel admin, sin intervención manual.

## Arquitectura

Toda la lógica multi-tenant descansa en cinco piezas:

1. Un **credencial dinámico** de OpenAI en n8n (`OpenAI Dynamic (Multi-tenant)`) cuya API key se resuelve por expresión en tiempo de ejecución.
2. El nodo **`PG_get_empresa`** que trae la `openai_api_key` y el `openai_key_status` de la empresa desde `bot_config` en cada ejecución.
3. Un nodo Postgres separado (**`PG_get_openai_key_audio`**) que obtiene la key para la rama de transcripción de audio, donde `PG_get_empresa` aún no ha corrido.
4. Un mecanismo de **auto-actualización del estado** con dos entradas: el `PG_update_openai_key_status` cuando el agente falla en runtime, y `PG_update_status_desde_IF` cuando el `If_openai_key_valid` rechaza la key por estructura.
5. Una **query UPSERT en Appsmith** (`update_pipeline`) que al guardar una key nueva resetea `openai_key_status` a `active` y recalcula `openai_key_fingerprint`.

### Flujo de resolución de la key

Webhook (llega mensaje de cualquier tenant)
    v
Filtro_mensaje -> Contexto (extrae account_id)
    v
PG_upsert_lead -> PG_upsert_conversacion
    v
PG_get_empresa (SELECT bc.openai_api_key, bc.openai_key_status WHERE chatwoot_account_id = ...)
    v
If_openai_key_valid (valida estructura: sk-*, >=40 chars, status='active')
    +--(TRUE)--> AI_Agent_Principal <- credencial dinamico resuelve la key
    |
    +--(FALSE)-> [rama de notificacion + PG_update_status_desde_IF]

Para la rama de audio, la resolución es distinta porque la transcripción con Whisper ocurre **antes** de que corra `PG_get_empresa`:

Switch (rama AUDIO)
    v
PG_get_openai_key_audio (SELECT bc.openai_api_key WHERE chatwoot_account_id = ...)
    v
Descargar_Audio (produce binary)
    v
Transcribir (HTTP Request a Whisper, con Bearer dinamico)
    v
Filtro_mensaje -> Contexto -> ...

## Credencial dinámico en n8n

### Configuración

En **Credentials -> OpenAI account**, crear un credencial llamado `OpenAI Dynamic (Multi-tenant)`. En el campo **API Key**, activar el toggle **Expression** y poner:

    ={{ $('PG_get_empresa').item.json.openai_api_key }}

Este credencial se asigna a los nodos:
- `OpenAI Chat Model1` (modelo del `AI_Agent_Principal`).
- `OpenAI Chat Model` (modelo del `AI_Agent_Clasificador`).

### Nota importante

El botón **Test** del credencial **fallará** con "Credential is invalid" porque la expresión no puede resolverse fuera del contexto de una ejecución del workflow. Es esperado: la validación real ocurre cuando el workflow ejecuta con datos reales del webhook, no en el editor.

## Rama de audio: Whisper multi-tenant

Los nodos LangChain (`@n8n/n8n-nodes-langchain.openAi` para transcripción) no soportan el patrón de credencial dinámico de la misma manera, así que la rama de audio se refactorizó a un **HTTP Request directo** contra la API de Whisper.

### Nodo `Transcribir` (HTTP Request)

- **Method:** POST
- **URL:** `https://api.openai.com/v1/audio/transcriptions`
- **Headers:**
  - `Authorization: Bearer {{ $('PG_get_openai_key_audio').item.json.openai_api_key }}`
- **Body:** `multipart/form-data`
  - `file`: binary (inputDataFieldName: `data`)
  - `model`: `whisper-1`

### Nodo `PG_get_openai_key_audio`

Postgres executeQuery:

    SELECT bc.openai_api_key
    FROM bot_config bc
    JOIN empresas e ON e.id = bc.empresa_id
    WHERE e.chatwoot_account_id = {{ $('Webhook').item.json.body.account.id }}
      AND e.activo = true
    LIMIT 1;

### Orden crítico de nodos (ver Bug 29)

`PG_get_openai_key_audio` debe estar **antes** de `Descargar_Audio`, no entre él y `Transcribir`. Los nodos Postgres no propagan datos binarios, y meter uno entre el nodo que produce el binario y el nodo que lo consume corta silenciosamente la cadena binaria.

## Datos en `bot_config`

Columnas relevantes de `bot_config` para este módulo:

| Columna | Tipo | Propósito |
|---|---|---|
| `openai_api_key` | `varchar(500)` | Key OpenAI del tenant. Actualmente en texto plano. |
| `openai_key_status` | `varchar(20)` | Estado operativo: `active`, `invalid`, `no_saldo`, `revoked`, `no_config`. Default `active`. |
| `openai_key_fingerprint` | `varchar(8)` | Últimos 4 caracteres de la key. Cache para logs y UI sin exponer valor completo. La fuente de verdad para reportes es siempre `RIGHT(openai_api_key, 4)` (ver Bug 36). |
| `modelo_bot` | `varchar(50)` | Modelo usado por `AI_Agent_Principal`. |
| `modelo_clasificador` | `varchar(50)` | Modelo usado por `AI_Agent_Clasificador`. |

### Migración aplicada (03 jul 2026)

    ALTER TABLE bot_config
        ADD COLUMN IF NOT EXISTS openai_key_status VARCHAR(20) DEFAULT 'active',
        ADD COLUMN IF NOT EXISTS openai_key_fingerprint VARCHAR(8);

    UPDATE bot_config
    SET openai_key_fingerprint = RIGHT(openai_api_key, 4)
    WHERE openai_api_key IS NOT NULL
      AND openai_key_fingerprint IS NULL;

## Auto-gestión del estado `openai_key_status`

El campo `openai_key_status` en `bot_config` se mantiene sincronizado con la realidad operativa a través de **tres fuentes complementarias**. Ninguna requiere intervención manual del admin salvo casos excepcionales (marcar `revoked` por morosidad).

### Fuente 1 - Runtime del agente (OpenAI responde 401 o quota)

Cuando `AI_Agent_Principal` intenta usar la key y OpenAI la rechaza, se dispara:

AI_Agent_Principal (falla con error)
    v (salida error, gracias a On Error: Continue)
Code_analizar_error_openai (clasifica el error)
    v
PG_update_openai_key_status (UPDATE bot_config SET openai_key_status = 'invalid' o 'no_saldo')
    v
[rama de notificacion: nota, urgente, correos, log]

El nodo `Code_analizar_error_openai` analiza el mensaje de error y produce:
- `nuevo_status = 'invalid'` cuando detecta `Incorrect API key`, `Invalid API key` o `Authorization failed`.
- `nuevo_status = 'no_saldo'` cuando detecta `insufficient_quota`, `quota` o `billing`.
- `nuevo_status = 'no_config'` cuando detecta `You didn't provide` o `No API key`.
- `nuevo_status = null` para errores desconocidos (no se actualiza status).

El `PG_update_openai_key_status` incluye un guardián `AND debe_actualizar_bd = true` para no hacer UPDATE cuando el error no se pudo clasificar.

### Fuente 2 - Rama FALSE del IF de validación estructural (Bug 39)

Cuando `If_openai_key_valid` rechaza la key por estructura (NULL, sin `sk-`, longitud < 40) antes de llegar al agente, se dispara `PG_update_status_desde_IF` en paralelo a las notificaciones:

    UPDATE bot_config
    SET openai_key_status = CASE
          WHEN openai_api_key IS NULL OR LENGTH(openai_api_key) = 0 THEN 'no_config'
          WHEN openai_api_key NOT LIKE 'sk-%' THEN 'invalid'
          WHEN LENGTH(openai_api_key) < 40 THEN 'invalid'
          ELSE openai_key_status
        END,
        openai_key_fingerprint = CASE
          WHEN openai_api_key IS NULL OR LENGTH(openai_api_key) = 0 THEN NULL
          ELSE RIGHT(openai_api_key, 4)
        END,
        updated_at = NOW()
    WHERE empresa_id = {{ $('PG_get_empresa').item.json.id }};

Sin este nodo, keys estructuralmente inválidas quedaban en `active` indefinidamente (ver Bug 39).

### Fuente 3 - Reset automático al guardar desde Appsmith

Cuando el admin actualiza la key desde el panel Config Bot, la query `update_pipeline` (UPSERT sobre `bot_config`) fuerza:

    ON CONFLICT (empresa_id) DO UPDATE SET
      ...
      openai_api_key         = EXCLUDED.openai_api_key,
      openai_key_status      = 'active',
      openai_key_fingerprint = RIGHT(EXCLUDED.openai_api_key, 4),
      ...

Esto le da al cliente **una oportunidad limpia** cada vez que sube una key: se asume `active` de entrada, y si sigue mala, el flujo n8n la volverá a marcar como `invalid` en el próximo mensaje. La lógica de negocio no penaliza intentos pasados.

### Resumen visual de las 3 fuentes

   +---------------------------------------------------------+
   |  bot_config.openai_key_status  <- 3 fuentes actualizan  |
   +---------------------------------------------------------+
                     ^           ^           ^
                     |           |           |
             (a) PG_update    (b) PG_update  (c) update_pipeline
                 _openai_key      _status_       (Appsmith)
                 _status          desde_IF       reset a 'active'
                 (n8n runtime)    (n8n IF)       al guardar key

## Nota importante sobre expresiones en n8n 2.23.3

En n8n 2.23.3 y posteriores, referenciar un nodo que no ejecutó con `$('X').item?.json?.campo` **lanza error explícito** en vez de resolver a `undefined`. El patrón oficial compatible es:

    {{ $('NodoQuePuedeNoEjecutar').isExecuted
       ? $('NodoQuePuedeNoEjecutar').item.json.campo
       : fallback }}

Este patrón es especialmente relevante en los canales de notificación cuando un nodo puede recibir input desde dos rutas distintas (auto-update runtime vs. IF estructural). Ver Bug 38 para detalles y ejemplos.

## Verificación de aislamiento entre tenants

Test que se hizo en producción para confirmar que la resolución dinámica es real por tenant (no hay fallback silencioso a una key global):

1. Elegir un tenant real y **quitarle temporalmente la key en BD**:

       UPDATE bot_config SET openai_api_key = NULL WHERE empresa_id = <id>;

2. Enviar un mensaje por WhatsApp a ese tenant.
3. Resultado esperado: el nodo `OpenAI Chat Model1` devuelve el error de OpenAI:

       Authorization failed - please check your credentials
       Incorrect API key provided: ''

   La key vacía (`''`) confirma que la expresión resolvió correctamente al valor `NULL` del tenant, no a una key global fija.

Si en ese escenario el bot respondiera con éxito, habría un fallback silencioso a otra key y la resolución dinámica no estaría funcionando.

## Costo por tenant y monitoreo

El consumo se refleja directamente en el dashboard de OpenAI de cada cliente (`platform.openai.com/usage`), ya que la key usada es la del tenant. AVE **no** guarda actualmente contadores de tokens ni de costo por tenant en la BD - es una deuda técnica identificada. Cuando se implemente, la tabla candidata es `openai_usage_log` (empresa_id, conversation_id, modelo, operacion, prompt_tokens, completion_tokens, costo_estimado_usd, created_at).

## Referencias

- Sistema de notificación cuando la key falla: notificaciones-openai-key.md
- Bugs relacionados: 29 (Postgres corta binario), 30 ($json roto tras insertar nodo), 35 (snapshots inmutables), 36 (fingerprint desincronizado), 37 (input password Appsmith), 38 (isExecuted en n8n 2.23.3), 39 (auto-update en rama FALSE).