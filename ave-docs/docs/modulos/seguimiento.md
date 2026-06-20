# Módulo: Sistema de Seguimiento (`en_seguimiento`)

Sistema genérico multi-empresa que controla qué conversaciones están sujetas a recordatorios automáticos. Reemplaza el criterio hardcodeado anterior (`agendado = false`, que solo servía para agencIA) por uno configurable que funciona para cualquier empresa.

---

## 1. Problema que resuelve

El workflow de recordatorios necesita saber **cuándo dejar de enviar mensajes de seguimiento** a una conversación. El criterio depende del negocio:

- **agencIA:** deja de seguir cuando el lead agenda (etiqueta `exitoso`).
- **PC Outlet:** deja de seguir cuando el lead pasa a `cliente-potencial` (transferido a asesor).
- **Futuras empresas:** cada una con su propia etiqueta de conversión.

La solución: una columna genérica `en_seguimiento_activo` que cada empresa apaga según su propia lógica, sin tocar el código del workflow.

---

## 2. Componentes

### 2.1 Columna en Postgres

```sql
ALTER TABLE conversaciones ADD COLUMN en_seguimiento_activo boolean DEFAULT true;
```

- `true` = seguimiento activo → el workflow de recordatorios SÍ procesa esta conversación.
- `false` = seguimiento apagado → el workflow la IGNORA (ya convirtió o un humano la atiende).
- El `DEFAULT true` aplica solo a filas nuevas; para filas existentes hay que hacer `UPDATE ... SET en_seguimiento_activo = true WHERE en_seguimiento_activo IS NULL`.

### 2.2 Custom attribute en Chatwoot

Atributo `en_seguimiento` (tipo checkbox/boolean, nivel conversación). Refleja visualmente el estado en el panel lateral de cada conversación. Se sincroniza con la columna de Postgres.

### 2.3 Mecanismo de conversión (tabla `etiquetas_pipeline`)

La columna `es_conversion` (boolean) en `etiquetas_pipeline` define qué etiquetas apagan el seguimiento. Ejemplo para agencIA:

| nombre | es_conversion |
|---|---|
| exitoso | **true** |
| lead-caliente | false |
| lead-tibio | false |
| lead-frio | false |

Puede haber más de una etiqueta con `es_conversion = true` por empresa.

---

## 3. Lógica de funcionamiento (3 momentos)

### Momento 1 — Inicialización (conversación nueva)

Cuando llega el primer mensaje, la conversación arranca con `en_seguimiento = true` en ambos sistemas.

- **Postgres:** automático por el `DEFAULT true` de la columna.
- **Chatwoot:** lo escribe el nodo `Chatwoot_inicializar_en_seguimiento`, que solo corre cuando `mensajes_count = 1` (señal de conversación nueva, no de UPDATE).

**Nodos involucrados:**
```
PG_get_empresa
  → (rama paralela) If_conversacion_nueva
      → [true] Chatwoot_inicializar_en_seguimiento
      → [false] (nada)
```

⚠️ **Importante:** `If_conversacion_nueva` cuelga de una rama **paralela** a la entrada del agente. NO debe alimentar al `AI_Agent_Principal` (ver [bug de memoria](../troubleshooting/bugs-resueltos.md#bug-4-error-de-memoria-key-parameter-is-empty)).

La condición usa referencia explícita al nodo:
```
{{ $('PG_upsert_conversacion').item.json.mensajes_count }} == 1
```

### Momento 2 — Mantenimiento (cada turno)

En cada mensaje, el nodo `actualizar_estado_lead` reescribe los custom attributes de Chatwoot. **Debe incluir TODOS los campos**, no solo `estado_lead`, porque la API de Chatwoot reemplaza el objeto completo (ver [bug de custom_attributes](../troubleshooting/bugs-resueltos.md#bug-3-custom-attributes-se-borran-en-cada-turno)).

Para esto, el nodo `PG_get_conversacion_actual` (nuevo) lee el estado real antes de reescribir:
```sql
SELECT en_seguimiento_activo, desactivar_bot
FROM conversaciones
WHERE empresa_id = (SELECT id FROM empresas WHERE chatwoot_account_id = {{ $('Contexto').item.json.account_id }} LIMIT 1)
  AND chatwoot_conversation_id = {{ $('Contexto').item.json.conversation_id }}
LIMIT 1;
```

Y `actualizar_estado_lead` manda el objeto completo:
```json
{
  "custom_attributes": {
    "estado_lead": "{{ $('AI_Agent_Clasificador').item.json.output }}",
    "en_seguimiento": {{ $('PG_get_conversacion_actual').item.json.en_seguimiento_activo }},
    "desactivar_bot": {{ $('PG_get_conversacion_actual').item.json.desactivar_bot }}
  }
}
```

### Momento 3 — Apagado (conversión)

Cuando el clasificador asigna una etiqueta con `es_conversion = true`, se apaga el seguimiento.

**Nodos involucrados:**
```
actualizar_etiqueta
  → PG_check_es_conversion
    → If_es_conversion
        → [true] Chatwoot_actualizar_en_seguimiento (Chatwoot)
        → [true] PG_actualizar_seguimiento (Postgres)
        → [false] (nada)
```

**`PG_check_es_conversion`** consulta si la etiqueta recién aplicada es de conversión:
```sql
SELECT COALESCE(
  (SELECT ep.es_conversion FROM etiquetas_pipeline ep
   WHERE ep.empresa_id = (SELECT id FROM empresas WHERE chatwoot_account_id = {{ $('Contexto').item.json.account_id }} LIMIT 1)
     AND ep.nombre = '{{ $('AI_Agent_Clasificador').item.json.output }}'
   LIMIT 1),
  false
) AS es_conversion;
```

Si devuelve `true`, `PG_actualizar_seguimiento` apaga la columna:
```sql
UPDATE conversaciones SET
  en_seguimiento_activo = false,
  updated_at = NOW() AT TIME ZONE 'America/Bogota'
WHERE empresa_id = (...) AND chatwoot_conversation_id = {{ $('Contexto').item.json.conversation_id }};
```

---

## 4. Sincronización con `desactivar_bot`

Cuando un humano toma el control (`desactivar_bot`), también se apaga el seguimiento — no tiene sentido seguir enviando recordatorios automáticos si hay una persona atendiendo.

- **`desactivar_bot_auto`** (rama automática): ya sincronizado vía `PG_sync_desactivar_bot_auto`, que pone `desactivar_bot = true` Y `en_seguimiento_activo = false` en Postgres.
- **Tool `desactivar_bot`** (invocada por el agente): ⚠️ **PENDIENTE** — esta tool solo escribe en Chatwoot, no sincroniza Postgres todavía. Las tools de un AI Agent no permiten encadenar nodos lineales. Solución recomendada (opción A): leer el custom attribute real de Chatwoot en cada turno y reflejarlo en Postgres. Ver troubleshooting.

---

## 5. Integración con recordatorios

El workflow de recordatorios filtra por este campo:
```sql
AND c.en_seguimiento_activo = true
```

Esto reemplazó al antiguo `AND c.agendado = false`. Resultado: una conversación convertida (`en_seguimiento_activo = false`) queda automáticamente excluida de los recordatorios, sin importar la empresa.

---

## 6. Validación (procedimiento)

**Validar que se mantiene activo turno a turno:**
1. Conversación nueva → verificar checkbox `en_seguimiento` marcado en Chatwoot.
2. Enviar 2-3 mensajes → debe seguir marcado.

**Validar que se apaga al convertir:**
1. Completar agendamiento (agencIA) hasta que el clasificador devuelva `exitoso`.
2. Verificar: `SELECT en_seguimiento_activo FROM conversaciones WHERE id = [ID]` → debe ser `false`.
3. Checkbox en Chatwoot debe desmarcarse.

**Validar exclusión de recordatorios:**
1. Conversación con `en_seguimiento_activo = false` → ejecutar workflow de recordatorios → NO debe aparecer en `PG_get_conversaciones_pendientes`.
2. Conversación con `en_seguimiento_activo = true` que cumpla tiempo → SÍ debe recibir recordatorio.

---

## 7. Bitácora de cambios

| Fecha | Versión | Cambio | Autor |
|---|---|---|---|
| 2026-06-19 | 1.0 | Implementación inicial del sistema genérico de seguimiento: columna `en_seguimiento_activo`, custom attribute en Chatwoot, apagado automático por `es_conversion`, inicialización en conversación nueva, fix de borrado de custom_attributes, sincronización de `desactivar_bot_auto`. | Enuar |
