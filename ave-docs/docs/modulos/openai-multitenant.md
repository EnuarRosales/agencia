# Módulo: Multi-tenant OpenAI

> Cada empresa cliente de AVE usa **su propia API key de OpenAI**, aislada del resto. Este módulo describe cómo se logra ese aislamiento en n8n, incluyendo el agente principal, el clasificador y la transcripción de audio (Whisper), sin duplicar workflows por tenant.

## Objetivo

- Cada tenant paga su propio consumo de OpenAI (facturación directa contra su cuenta).
- Un problema con la key de un tenant (revocada, sin saldo, no configurada) no afecta a los demás.
- No hay duplicación de workflows por tenant.
- Rotación de key por parte del cliente sin necesidad de tocar el workflow.

## Arquitectura

Toda la lógica multi-tenant descansa en tres piezas:

1. Un **credencial dinámico** de OpenAI en n8n (`OpenAI Dynamic (Multi-tenant)`) cuya API key se resuelve por expresión en tiempo de ejecución.
2. El nodo **`PG_get_empresa`** que trae la `openai_api_key` de la empresa desde `bot_config` en cada ejecución.
3. Un nodo Postgres separado (**`PG_get_openai_key_audio`**) que obtiene la key para la rama de transcripción de audio, donde `PG_get_empresa` aún no ha corrido.

### Flujo de resolución de la key

```
Webhook (llega mensaje de cualquier tenant)
    ↓
Filtro_mensaje → Contexto (extrae account_id)
    ↓
PG_upsert_lead → PG_upsert_conversacion
    ↓
PG_get_empresa (SELECT bc.openai_api_key WHERE chatwoot_account_id = ...)
    ↓
AI_Agent_Principal  ← el credencial dinámico resuelve la key desde este contexto
```

Para la rama de audio, la resolución es distinta porque la transcripción con Whisper ocurre **antes** de que corra `PG_get_empresa`:

```
Switch (rama AUDIO)
    ↓
PG_get_openai_key_audio (SELECT bc.openai_api_key WHERE chatwoot_account_id = ...)
    ↓
Descargar_Audio (produce binary)
    ↓
Transcribir (HTTP Request a Whisper, con Bearer dinámico)
    ↓
Filtro_mensaje → Contexto → ...
```

## Credencial dinámico en n8n

### Configuración

En **Credentials → OpenAI account**, crear un credencial llamado `OpenAI Dynamic (Multi-tenant)`. En el campo **API Key**, activar el toggle **Expression** y poner:

```
={{ $('PG_get_empresa').item.json.openai_api_key }}
```

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
```sql
SELECT bc.openai_api_key
FROM bot_config bc
JOIN empresas e ON e.id = bc.empresa_id
WHERE e.chatwoot_account_id = {{ $('Webhook').item.json.body.account.id }}
  AND e.activo = true
LIMIT 1;
```

### Orden crítico de nodos (ver Bug 29)

`PG_get_openai_key_audio` debe estar **antes** de `Descargar_Audio`, no entre él y `Transcribir`. Los nodos Postgres no propagan datos binarios, y meter uno entre el nodo que produce el binario y el nodo que lo consume corta silenciosamente la cadena binaria.

## Datos en `bot_config`

Columnas relevantes de `bot_config` para este módulo:

| Columna | Tipo | Propósito |
|---|---|---|
| `openai_api_key` | `varchar(500)` | Key OpenAI del tenant. Actualmente en texto plano. |
| `openai_key_status` | `varchar(20)` | Estado operativo: `active` \| `invalid` \| `no_saldo` \| `revoked` \| `no_config`. Default `active`. |
| `openai_key_fingerprint` | `varchar(8)` | Últimos 4 caracteres de la key. Para logs y UI sin exponer valor completo. |
| `modelo_bot` | `varchar(50)` | Modelo usado por `AI_Agent_Principal`. |
| `modelo_clasificador` | `varchar(50)` | Modelo usado por `AI_Agent_Clasificador`. |

### Migración aplicada (03 jul 2026)

```sql
ALTER TABLE bot_config
    ADD COLUMN IF NOT EXISTS openai_key_status VARCHAR(20) DEFAULT 'active',
    ADD COLUMN IF NOT EXISTS openai_key_fingerprint VARCHAR(8);

UPDATE bot_config
SET openai_key_fingerprint = RIGHT(openai_api_key, 4)
WHERE openai_api_key IS NOT NULL
  AND openai_key_fingerprint IS NULL;
```

## Verificación de aislamiento entre tenants

Test que se hizo en producción para confirmar que la resolución dinámica es real por tenant (no hay fallback silencioso a una key global):

1. Elegir un tenant real y **quitarle temporalmente la key en BD**:
   ```sql
   UPDATE bot_config SET openai_api_key = NULL WHERE empresa_id = <id>;
   ```
2. Enviar un mensaje por WhatsApp a ese tenant.
3. Resultado esperado: el nodo `OpenAI Chat Model1` devuelve el error de OpenAI:
   ```
   Authorization failed - please check your credentials
   Incorrect API key provided: ''
   ```
   La key vacía (`''`) confirma que la expresión resolvió correctamente al valor `NULL` del tenant, no a una key global fija.

Si en ese escenario el bot respondiera con éxito, habría un fallback silencioso a otra key y la resolución dinámica no estaría funcionando.

## Costo por tenant y monitoreo

El consumo se refleja directamente en el dashboard de OpenAI de cada cliente (`platform.openai.com/usage`), ya que la key usada es la del tenant. AVE **no** guarda actualmente contadores de tokens ni de costo por tenant en la BD — es una deuda técnica identificada. Cuando se implemente, la tabla candidata es `openai_usage_log` (empresa_id, conversation_id, modelo, operacion, prompt_tokens, completion_tokens, costo_estimado_usd, created_at).

## Referencias

- Sistema de notificación cuando la key falla: [notificaciones-openai-key.md](notificaciones-openai-key.md)
- Bugs relacionados: 29 (Postgres corta binario), 30 ($json roto tras insertar nodo).
