# Módulo: Recordatorios Automáticos

Workflow de n8n que envía mensajes de seguimiento automáticos por WhatsApp cuando un lead deja de responder. Los mensajes son generados por IA con base en el historial real de cada conversación.

**Workflow n8n:** `AVE_Recordatorios_Automáticos`
**Tipo de disparo:** Cron (programado)
**Naturaleza:** Multi-tenant (procesa todas las empresas en cada ejecución)

---

## 1. Propósito y lógica de negocio

Cuando un lead escribe por WhatsApp y luego deja de responder, el sistema le envía recordatorios escalonados para reactivar la conversación. Cada empresa configura su propia secuencia de recordatorios (cuántos, con qué tiempo de espera y con qué mensaje base).

El sistema soporta **N recordatorios configurables** — no está limitado a un número fijo. La cantidad y timing se definen en la tabla `recordatorio_config` desde Appsmith.

---

## 2. Configuración (tabla `recordatorio_config`)

Cada fila es un paso de la secuencia de recordatorios para una empresa.

| Columna | Tipo | Descripción |
|---|---|---|
| `empresa_id` | int | Empresa a la que aplica |
| `orden` | int | Posición en la secuencia (1, 2, 3...) |
| `horas_despues` | numeric | Horas de espera. Acepta decimales: `0.2` = 12 min, `0.5` = 30 min |
| `mensaje` | text | Prompt base / guía para la IA (no es el texto literal enviado) |
| `activo` | boolean | Si el recordatorio está habilitado |

**Importante sobre el campo `mensaje`:** no se envía literal al cliente. Se usa como *instrucción/guía* para que la IA genere un mensaje contextualizado con el historial real de la conversación. Por eso dos recordatorios pueden sonar distintos aunque partan del mismo `mensaje` base. **Decisión de diseño confirmada (jun 2026): se mantiene la generación por IA.** Si en el futuro se quisiera envío literal, habría que agregar un campo `usar_ia` en `recordatorio_config` y bifurcar el flujo antes de `OpenAI_generar`.

Ejemplo de configuración para agencIA:

| orden | horas_despues | mensaje |
|---|---|---|
| 1 | 0.2 | Hola, queria asegurarme de que recibio la informacion... |
| 2 | 0.5 | Es un buen momento para dar el siguiente paso. Agendemos... |

---

## 3. Estructura del workflow (nodo por nodo)

El workflow usa un diseño de **loop lineal** que itera sobre empresas y, dentro de cada una, sobre las conversaciones pendientes. Esto reemplazó un diseño anterior de ramas fijas (rec1/rec2) que solo soportaba 2 recordatorios.

```
Cron_5min
  → PG_get_empresas
    → Loop_empresas (SplitInBatches)
      → PG_get_conversaciones_pendientes  [Always Output Data = true]
        → If_hay_pendientes
          ├─ TRUE  → Loop_conversaciones (SplitInBatches)
          │            → GET_historial  [Continue On Fail]
          │              → If_conversacion_existe
          │                  ├─ TRUE  → Code_historial → OpenAI_generar [Continue On Fail]
          │                  │            → Code_extract → Chatwoot_enviar [Continue On Fail]
          │                  │              → PG_marcar → (vuelve a Loop_conversaciones)
          │                  └─ FALSE → PG_marcar_fantasma → (vuelve a Loop_conversaciones)
          └─ FALSE → Loop_empresas (siguiente empresa)
```

### Detalle de cada nodo

**`Cron_5min`** — Disparador programado. Durante pruebas corre cada 5 min. En producción puede subirse a 15-30 min para no saturar la API de OpenAI. (Antes se llamaba `Cron_30min`; renombrado para que el nombre refleje el intervalo real — ver Bug 13 sobre nombres engañosos.)

**`PG_get_empresas`** — Trae todas las empresas activas con su configuración (incluido `chatwoot_api_key`, `chatwoot_account_id`, `openai_api_key`, y el array de `recordatorios` agregado por empresa vía `json_agg`). Usa INNER JOIN con `bot_config` y `recordatorio_config` (activa): una empresa sin recordatorio_config activa NO entra al loop. Query:

```sql
SELECT
  e.id AS empresa_id, e.nombre, e.chatwoot_url, e.chatwoot_api_key,
  e.chatwoot_account_id, bc.openai_api_key, bc.objetivo_bot, bc.tono, bc.modelo_bot,
  json_agg(
    json_build_object('orden', rc.orden, 'horas_despues', rc.horas_despues, 'mensaje', rc.mensaje)
    ORDER BY rc.orden
  ) AS recordatorios
FROM empresas e
JOIN bot_config bc ON bc.empresa_id = e.id
JOIN recordatorio_config rc ON rc.empresa_id = e.id AND rc.activo = true
WHERE e.activo = true
GROUP BY e.id, e.nombre, e.chatwoot_url, e.chatwoot_api_key,
         e.chatwoot_account_id, bc.openai_api_key, bc.objetivo_bot, bc.tono, bc.modelo_bot;
```

**`Loop_empresas`** — SplitInBatches. **Crítico:** la salida 0 es "done" (termina), la salida 1 es "loop" (procesa el item actual). El nodo siguiente (`PG_get_conversaciones_pendientes`) debe colgar de la **salida 1**, no la 0.

**`PG_get_conversaciones_pendientes`** — El corazón de la lógica. **Tiene "Always Output Data" activado** (Bug 18): emite un item aunque no haya filas, para que el loop de empresas no se detenga cuando una empresa no tiene pendientes. Trae las conversaciones de la empresa actual que cumplen TODAS estas condiciones:
- `estado = 'abierta'`
- `desactivar_bot = false`
- `en_seguimiento_activo = true` ← criterio genérico de exclusión (ver [módulo seguimiento](seguimiento.md))
- `ultimo_mensaje_usuario_at IS NOT NULL`
- `recordatorios_enviados < (cantidad total de recordatorios activos)`
- Y cumple la condición de tiempo según el paso:
  - Si `recordatorios_enviados = 0`: compara `ultimo_mensaje_usuario_at` contra el umbral del recordatorio `orden = 1`.
  - Si `recordatorios_enviados >= 1`: compara `ultimo_recordatorio_at` contra el umbral del recordatorio `orden = recordatorios_enviados + 1`.

```sql
SELECT c.id AS conversacion_id, c.chatwoot_conversation_id, c.lead_id,
  c.recordatorios_enviados, c.ultimo_recordatorio_at, c.ultimo_mensaje_usuario_at,
  l.nombre AS lead_nombre, l.telefono AS lead_telefono, l.estado_lead
FROM conversaciones c
JOIN leads l ON l.id = c.lead_id
WHERE c.empresa_id = {{ $('Loop_empresas').item.json.empresa_id }}
  AND c.estado = 'abierta'
  AND c.desactivar_bot = false
  AND c.en_seguimiento_activo = true
  AND c.ultimo_mensaje_usuario_at IS NOT NULL
  AND c.recordatorios_enviados < json_array_length(
        (SELECT json_agg(rc) FROM recordatorio_config rc
         WHERE rc.empresa_id = c.empresa_id AND rc.activo = true))
  AND (
    CASE
      WHEN c.recordatorios_enviados = 0 THEN
        c.ultimo_mensaje_usuario_at < (NOW() AT TIME ZONE 'America/Bogota') - INTERVAL '1 hour' *
        (SELECT rc.horas_despues FROM recordatorio_config rc
         WHERE rc.empresa_id = c.empresa_id AND rc.orden = 1 AND rc.activo = true)
      ELSE
        c.ultimo_recordatorio_at < (NOW() AT TIME ZONE 'America/Bogota') - INTERVAL '1 hour' *
        (SELECT rc.horas_despues FROM recordatorio_config rc
         WHERE rc.empresa_id = c.empresa_id
           AND rc.orden = c.recordatorios_enviados + 1 AND rc.activo = true)
    END
  );
```

⚠️ **Nota de zona horaria** (actualizada Bug 16): las cuatro columnas temporales de `conversaciones` (`ultimo_mensaje_usuario_at`, `ultimo_recordatorio_at`, `created_at`, `updated_at`) son `timestamp WITHOUT time zone` en hora Bogotá. TODAS las comparaciones y escrituras usan `(NOW() AT TIME ZONE 'America/Bogota')`. La columna `ultimo_recordatorio_at` se migró de `WITH time zone` a `WITHOUT time zone` para unificar el modelo y eliminar el desfase de 5h que sufría la rama ELSE. Ver [reglas de timezone](../base-de-datos/reglas-timezone.md) y Bug 16 en `bugs-resueltos.md`.

```sql
-- Migración aplicada (Bug 16):
ALTER TABLE conversaciones
  ALTER COLUMN ultimo_recordatorio_at TYPE timestamp without time zone
  USING ultimo_recordatorio_at AT TIME ZONE 'America/Bogota';
```

**`If_hay_pendientes`** — Verifica que el resultado anterior no esté vacío (`conversacion_id` notEmpty). **Ambas ramas conectadas** (Bug 17): TRUE → `Loop_conversaciones` (procesa); FALSE → `Loop_empresas` (siguiente empresa). Sin la rama FALSE conectada, una empresa sin pendientes detenía todo el loop antes de procesar las siguientes.

**`Loop_conversaciones`** — SplitInBatches sobre las conversaciones pendientes. Misma regla de salidas: 0 = done (vuelve a `Loop_empresas` por la siguiente empresa), 1 = loop (procesa la conversación actual hacia `GET_historial`).

**`GET_historial`** — HTTP GET a Chatwoot para traer los mensajes de la conversación. **Continue On Fail activado** (Bug 19): si la conversación ya no existe en Chatwoot (404), no rompe el lote. **Autenticación dinámica por empresa** (no credencial fija):
```
URL: https://chatwoot.identechnology.co/api/v1/accounts/{{ $('Loop_empresas').item.json.chatwoot_account_id }}/conversations/{{ $json.chatwoot_conversation_id }}/messages
Header: api_access_token = {{ $('Loop_empresas').item.json.chatwoot_api_key }}
```

**`If_conversacion_existe`** — (NUEVO, Bug 19 / Capa 2) Detecta conversaciones borradas en Chatwoot. Cuando `GET_historial` falla con 404 bajo Continue On Fail, n8n 2.23.3 reemplaza el item por un objeto `error` sin campo `payload`. Condición: `{{ $json.payload }}` **exists**.
- TRUE (existe payload) → `Code_historial` (flujo normal).
- FALSE (404, item con `error`) → `PG_marcar_fantasma`.

**`PG_marcar_fantasma`** — (NUEVO, Bug 19 / Capa 2) Desactiva en Postgres conversaciones que ya no existen en Chatwoot, para no reintentarlas en cada ciclo. Auto-sincronización BD↔Chatwoot.
```sql
UPDATE conversaciones SET
  en_seguimiento_activo = false,
  updated_at = NOW() AT TIME ZONE 'America/Bogota'
WHERE id = {{ $('Loop_conversaciones').item.json.conversacion_id }};
```
Se eligió `en_seguimiento_activo = false` (no `estado = 'cerrada'`) por ser acción quirúrgica: solo apaga recordatorios sin afectar otros procesos que leen `estado`.

**`Code_historial`** — Procesa el historial y arma el contexto. Determina qué recordatorio toca con `recordatorios.find(r => r.orden === numRec + 1)`, donde `numRec = recordatorios_enviados`. Esto es lo que hace al workflow genérico para N recordatorios. Robusto ante payload vacío (`$input.item.json.payload || []`).

**`OpenAI_generar`** — Llama a OpenAI con el `mensaje` base como guía + el historial, para generar el texto final del recordatorio. Usa `openai_api_key` dinámica de la empresa. **Continue On Fail activado** (Bug 19).

**`Code_extract`** — Extrae el texto generado de la respuesta de OpenAI.

**`Chatwoot_enviar`** — HTTP POST a Chatwoot para enviar el mensaje. Auth dinámica igual que `GET_historial`. `message_type: outgoing`. **Continue On Fail activado** (Bug 19).

**`PG_marcar`** — Actualiza la conversación: `recordatorios_enviados = recordatorios_enviados + 1`, `ultimo_recordatorio_at = NOW() AT TIME ZONE 'America/Bogota'`, `updated_at = NOW() AT TIME ZONE 'America/Bogota'`, e inserta un registro en la tabla `recordatorios` para historial.

---

## 4. Cómo se decide "cuál recordatorio sigue"

El único dato que determina el siguiente recordatorio es la columna **`recordatorios_enviados`** en `conversaciones` — un contador simple que sube de 1 en 1.

- Conversación nueva → `recordatorios_enviados = 0`.
- Tras enviar el recordatorio 1 → `PG_marcar` lo sube a `1`.
- Tras enviar el recordatorio 2 → sube a `2`.

No hay memoria de "cuál mensaje específico mandé", solo "cuántos he mandado". Por eso, si `PG_marcar` falla o algo resetea el contador, el sistema repite el recordatorio anterior.

**Reseteo del contador:** cuando el lead vuelve a escribir, `PG_actualizar_conversacion` (en el workflow principal del bot) pone `recordatorios_enviados = 0`. Esto es intencional: si el lead respondió, ya no está inactivo, y la secuencia debe reiniciarse desde cero si vuelve a quedar en silencio.

---

## 5. Cómo configurar para que se ejecute a los N minutos

**Dos cosas independientes a ajustar:**

1. **El tiempo de espera** (`horas_despues` en `recordatorio_config`, vía Appsmith). Para 5 minutos: `0.0833` (5/60). Para 12 min: `0.2`. Para 30 min: `0.5`.

2. **La frecuencia del cron** (nodo `Cron_5min` en n8n). El recordatorio solo se envía en el ciclo del cron *posterior* a cumplirse el umbral. Si el cron corre cada 30 min pero el umbral es 5 min, el envío real puede tardar hasta 30 min. Para pruebas precisas, mantener el cron en 5 min.

**En producción:** considerar volver el cron a 15-30 min para no saturar la API de OpenAI con ejecuciones innecesarias.

---

## 6. Manejo de conversaciones borradas en Chatwoot (Bug 19)

Si una conversación o contacto se borra/cierra en Chatwoot pero sigue activa en Postgres:

1. `GET_historial` intenta traer el historial → Chatwoot responde **404**.
2. **Continue On Fail** evita que el 404 detenga el lote (las demás conversaciones siguen).
3. `If_conversacion_existe` detecta la ausencia de `payload` → rama FALSE.
4. `PG_marcar_fantasma` pone `en_seguimiento_activo = false` → **nunca más se reintenta**.

Resultado: sin reintentos infinitos, sin logs sucios, sin tokens de OpenAI desperdiciados. La BD se mantiene sincronizada con Chatwoot automáticamente. Validado en producción con `chatwoot_conversation_id=100` (borrada en Chatwoot → auto-desactivada en BD con `success: true`).

---

## 7. Procedimiento de prueba desde cero

```sql
-- 1. Limpiar (respetando foreign keys: hijos antes que padres)
DELETE FROM recordatorios WHERE empresa_id = 1;
DELETE FROM citas WHERE lead_id IN (SELECT id FROM leads WHERE empresa_id = 1);
DELETE FROM eventos_lead WHERE lead_id IN (SELECT id FROM leads WHERE empresa_id = 1);
DELETE FROM pedidos WHERE lead_id IN (SELECT id FROM leads WHERE empresa_id = 1);
DELETE FROM conversaciones WHERE empresa_id = 1;
DELETE FROM leads WHERE empresa_id = 1;

-- 2. Confirmar en cero
SELECT
  (SELECT count(*) FROM leads WHERE empresa_id = 1) AS leads,
  (SELECT count(*) FROM conversaciones WHERE empresa_id = 1) AS conversaciones;
```

> Nota FK: solo `recordatorios` tiene FK a `conversaciones`. Las tablas `conversaciones`, `citas`, `recordatorios`, `eventos_lead` y `pedidos` tienen FK a `leads`. Borrar siempre hijos antes que padres. Para limpieza total de TODAS las empresas con reinicio de IDs: `TRUNCATE TABLE recordatorios, citas, eventos_lead, pedidos, conversaciones, leads RESTART IDENTITY CASCADE;`

3. Escribir "Hola" por WhatsApp (crea conversación con `en_seguimiento_activo = true`).
4. Verificar estado inicial:
```sql
SELECT id, en_seguimiento_activo, desactivar_bot, recordatorios_enviados, ultimo_mensaje_usuario_at
FROM conversaciones WHERE empresa_id = 1 AND estado = 'abierta' ORDER BY created_at DESC LIMIT 1;
```
5. Esperar el tiempo configurado, o simular para acelerar:
```sql
UPDATE conversaciones SET ultimo_mensaje_usuario_at = (NOW() AT TIME ZONE 'America/Bogota') - INTERVAL '1 hour' WHERE id = [ID];
```
6. Ejecutar el workflow completo con **"Execute Workflow"** (no "Execute step" — los SplitInBatches no funcionan aislados).
7. (Recomendado) Verificar que la conversación exista en Chatwoot antes de esperar el envío:
```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "api_access_token: [CHATWOOT_API_KEY]" \
  "https://chatwoot.identechnology.co/api/v1/accounts/[ACCOUNT_ID]/conversations/[CONV_ID]/messages"
```
Un `200` confirma que existe; un `404` significa fantasma (el workflow la auto-desactivará).

---

## 8. Bitácora de cambios

| Fecha | Versión | Cambio | Autor |
|---|---|---|---|
| 2026-06-22 | 2.1 | Fix timezone rama ELSE + migración `ultimo_recordatorio_at` a sin-zona (Bug 16). Fix cableado loop multi-empresa: rama FALSE de `If_hay_pendientes` vuelve a `Loop_empresas` (Bug 17). Always Output Data en pendientes (Bug 18). Manejo de conversaciones borradas en Chatwoot: Continue On Fail + auto-desactivación vía `If_conversacion_existe` / `PG_marcar_fantasma` (Bug 19). Cron renombrado a `Cron_5min`. | Enuar |
| 2026-06-19 | 2.0 | Rediseño de ramas fijas (rec1/rec2) a loop lineal que soporta N recordatorios. Auth de Chatwoot dinámica por empresa. Fix de timezone en comparaciones. Condición de exclusión cambiada a `en_seguimiento_activo`. | Enuar |
| (anterior) | 1.0 | Versión inicial con 2 recordatorios fijos y credencial Chatwoot fija. | Enuar |