# Módulo: Sincronización `desactivar_bot` (en progreso)

**Estado:** Diseño implementado y validado. Bug original (y su causa real: duplicación en dos caminos paralelos) resuelto y confirmado con prueba de regresión limpia. Quedan pendientes secundarios (ver sección 7).

---

## 1. Concepto clave: dos tipos de etiquetas en AVE

Antes de explicar el módulo, es necesario distinguir dos categorías de etiquetas que existen en el sistema y que **no deben mezclarse** — confundirlas fue la causa raíz del bug documentado en la sección 4.

### 1.1 Etiquetas funcionales / de calificación

Viven en la tabla **`etiquetas_pipeline`**. Son el resultado del `AI_Agent_Clasificador`, que analiza cada conversación y **siempre** devuelve exactamente una de ellas. Representan **en qué etapa de calificación está el lead**.

Ejemplo (agencIA):

| empresa_id | nombre | es_conversion |
|---|---|---|
| 1 | exitoso | true |
| 1 | lead-caliente | false |
| 1 | lead-tibio | false |
| 1 | lead-frio | false |

Características:
- Son **mutuamente excluyentes** — una conversación está en una sola en un momento dado.
- Las genera el clasificador automáticamente en cada turno.
- Controlan el [sistema de seguimiento](seguimiento.md) vía `es_conversion`.

### 1.2 Etiquetas operativas

Se aplican desde **tools específicas** del agente, no del clasificador. Representan **una acción operativa que ocurrió**, no una etapa de calificación.

Ejemplos: `agendado` (tool `etiqueta_agendado`), `desactivar_bot` (tool `desactivar_bot`).

Características:
- **No son excluyentes entre sí ni con las de calificación** — una conversación puede estar en `lead-caliente` (funcional) y tener `agendado` (operativa) al mismo tiempo.
- Cada una dispara una acción de sistema específica.
- Viven en tablas dedicadas, separadas de `etiquetas_pipeline` (ver tabla `etiquetas_operativas` en sección 5).

**Regla de diseño:** nunca mezclar ambos tipos en la misma tabla ni en la misma lógica de control. Cada tipo tiene su propia tabla de configuración y su propio propósito.

---

## 2. Objetivo de la funcionalidad

Cuando el agente de IA decide transferir una conversación a un humano, el sistema debe reflejar ese cambio en dos lugares:

1. **Chatwoot** (custom attribute `desactivar_bot`) — ya funciona vía la tool del agente.
2. **Postgres** (columna `desactivar_bot` en `conversaciones`) — fuente de verdad para procesos automatizados (ej. el workflow de recordatorios no debe enviar mensajes a una conversación que un humano está atendiendo).

**Problema adicional a resolver:** el agente puede, en ocasiones, omitir la ejecución de la tool `desactivar_bot` aunque el caso lo amerite ("dejarlo escapar"). Se necesita un mecanismo de respaldo (segundo filtro) que no dependa exclusivamente de que el agente ejecute la tool correctamente.

---

## 3. Diseño: doble filtro

**Filtro 1 — El agente.** Sigue usando la tool `desactivar_bot` según las reglas de su system prompt (queja, pedir humano, negociación, cuenta de alto valor).

**Filtro 2 — Nodo de seguridad, configurable por empresa.** En cada turno, revisa si alguna de las etiquetas actualmente aplicadas a la conversación coincide con una lista configurable de "etiquetas que deben forzar la desactivación", específica por empresa. Si hay coincidencia, sincroniza `desactivar_bot = true` en Postgres y Chatwoot — sin importar si la tool fue ejecutada o no por el agente.

Esto cubre el caso de "el agente lo dejó escapar", sin depender de una lista fija en el código.

---

## 4. Comportamiento anómalo detectado (bug, pre-existente)

Antes de este rediseño, el filtro 2 existía pero estaba mal construido — mezclaba etiquetas operativas y funcionales en una condición hardcodeada dentro del nodo `If_clasificar_lead`:

```javascript
['compra-realizada', 'agendado', 'desactivar_bot'].includes(e)
```

**Problema:** esto forzaba `desactivar_bot = true` también cuando la etiqueta presente era `agendado` — contradiciendo explícitamente el system prompt del agente, que indica: *"CRÍTICO: Después de agendar NO uses desactivar_bot. El agente permanece activo..."*.

**Origen:** se construyó como parche puntual para resolver el caso de "el agente no ejecuta la tool", pero se le agregaron etiquetas adicionales sin considerar el efecto colateral, y quedó con nombres de etiquetas fijos en el código — no configurable, no multi-tenant.

**Evidencia:** una conversación de agencIA que completó el agendamiento (etiqueta `agendado`) quedó con `desactivar_bot = true` en Postgres, pese a que el diseño dice que el bot debe seguir activo en ese caso.

### 4.1 Hallazgo crítico durante la implementación: existía un SEGUNDO camino duplicado

Al corregir `If_clasificar_lead` (sección 5) y volver a probar, **el bug seguía reapareciendo** pese a que ese nodo ya estaba arreglado y verificado. Una investigación exhaustiva — incluyendo barrido cronológico completo de ejecuciones en n8n, porque pruebas superficiales daban resultados contradictorios e inconcluyentes — reveló la causa real: existía un **segundo nodo independiente**, `If_forzar_desactivar_bot`, con exactamente el mismo patrón hardcodeado:

```javascript
['compra-realizada', 'agendado'].includes(e)
```

Este nodo era un camino **paralelo y completamente separado** que también terminaba en `PG_actualizar_estado_lead_directo → PG_actualizar_conversacion_directo` (el nodo que fuerza `desactivar_bot = true` de forma incondicional). Arreglar solo `If_clasificar_lead` cerró una puerta, pero esta segunda puerta hacia el mismo destino seguía abierta.

**Lección crítica:** cuando existen múltiples caminos paralelos hacia el mismo nodo de efecto (en este caso, dos `If` distintos que convergen en `PG_actualizar_estado_lead_directo`), corregir la condición de uno solo no es suficiente — hay que mapear **todos** los caminos de entrada a un nodo de efecto antes de dar un bug por resuelto. Una prueba que "a veces falla y a veces no" tras un fix aplicado correctamente es señal de que hay otra puerta de entrada sin identificar, no de que el fix esté mal.

**Resolución (Opción B, aprobada):** en vez de duplicar la lógica de `debe_desactivar` en ambos `If`, se eliminó por completo `If_forzar_desactivar_bot` junto con los nodos que solo él alimentaba (`desactivar_bot_auto`, `PG_sync_desactivar_bot_auto`), dejando un único camino posible hacia `PG_actualizar_estado_lead_directo`: vía `If_clasificar_lead`, controlado exclusivamente por `PG_check_desactivar_bot` y la tabla `etiquetas_operativas`.

```
Antes:
  actualizar_estado_lead → If_forzar_desactivar_bot
                              ├─ [true]  → desactivar_bot_auto → PG_sync_desactivar_bot_auto → PG_actualizar_estado_lead_directo → ...
                              └─ [false] → actualizar_etiqueta

Después:
  actualizar_estado_lead → actualizar_etiqueta
```

**Validado:** prueba de regresión completa desde cero (sin contaminación de pruebas anteriores) — conversación agendada con etiqueta `exitoso` aplicada por el clasificador, `desactivar_bot` quedó correctamente en `false`, el bot permaneció activo y respondió con normalidad tras la confirmación de la cita.

---

## 5. Solución: tabla configurable `etiquetas_operativas`

**Decisión de diseño (evolucionada durante la implementación):** en vez de crear una tabla dedicada únicamente a `desactivar_bot`, se optó por una tabla más general — `etiquetas_operativas` — con una columna `accion`, pensada para alojar también, en el futuro, la lógica de `desactivar_seguimiento` que hoy vive en `etiquetas_pipeline.es_conversion` (ver sección 7). Esto evita tener múltiples tablas haciendo variaciones del mismo patrón configurable.

```sql
CREATE TABLE etiquetas_operativas (
  id serial PRIMARY KEY,
  empresa_id int REFERENCES empresas(id),
  etiqueta varchar(100) NOT NULL,
  accion varchar(50) NOT NULL,  -- 'desactivar_bot' | 'desactivar_seguimiento' (futuro)
  activo boolean DEFAULT true,
  UNIQUE(empresa_id, etiqueta, accion)
);
```

**Configuración para agencIA** (corrige el bug — `agendado` y `compra-realizada` ya no aparecen; se agregó `lead-tibio` como prueba del mecanismo configurable, validada con éxito):

| empresa_id | etiqueta | accion | activo |
|---|---|---|---|
| 1 | desactivar_bot | desactivar_bot | true |
| 1 | lead-tibio | desactivar_bot | true *(prueba, validada — pendiente decidir si se deja permanente)* |

**Nodo nuevo — `PG_check_desactivar_bot`** (reemplaza la condición JavaScript hardcodeada de `If_clasificar_lead`):

```sql
SELECT EXISTS (
  SELECT 1 FROM etiquetas_operativas eo
  WHERE eo.empresa_id = (SELECT id FROM empresas WHERE chatwoot_account_id = {{ $('Contexto').item.json.account_id }} LIMIT 1)
    AND eo.accion = 'desactivar_bot'
    AND eo.activo = true
    AND eo.etiqueta = ANY(
      ARRAY[{{ ($('GET_etiquetas_actuales').item.json.payload || []).map(t => `'${t}'`).join(',') }}]::text[]
    )
) AS debe_desactivar;
```

Ubicado en la cadena: `get_estado_conversacion → GET_etiquetas_actuales → PG_check_desactivar_bot → If_clasificar_lead`. El `If_clasificar_lead` evalúa `{{ $json.debe_desactivar }}` (boolean) en vez del array hardcodeado.

**Nodo nuevo — `POST_sincronizar_desactivar_bot_chatwoot`** (sincroniza el resultado hacia Chatwoot, ya que `PG_actualizar_conversacion_directo` solo escribía en Postgres, dejando el custom attribute de Chatwoot desactualizado):

```json
{
  "custom_attributes": {
    "desactivar_bot": true,
    "en_seguimiento": false
  }
}
```

Conectado en paralelo a `GET_etiquetas_contacto` / `POST_sincronizar_etiquetas_contacto`, después de `PG_actualizar_conversacion_directo`.

⚠️ **Nota:** inicialmente se intentó incluir `estado_lead` en este body usando `{{ $('AI_Agent_Clasificador').item.json.output }}`, pero causó error (`Node 'AI_Agent_Clasificador' hasn't been executed`) — ese nodo no se ejecuta en todos los caminos que llegan hasta aquí. Se quitó esa referencia; el `estado_lead` se sincroniza en otros puntos del flujo donde sí está disponible.

### 5.1 Hallazgo relacionado (no resuelto, fuera de alcance de este módulo)

Durante la implementación se encontró que `POST_sincronizar_etiquetas_contacto` (un nodo preexistente, que sincroniza *labels* — no custom attributes — a nivel de *contacto*, no de conversación) **también** tiene el mismo patrón de etiquetas hardcodeadas: `['compra-realizada','agendado']`. No se tocó en esta sesión para no mezclar cambios en una misma prueba. Queda como hallazgo para una futura revisión, posiblemente aplicando la misma solución de `etiquetas_operativas`.

---

## 6. Decisión futura relacionada: migrar `es_conversion` a `etiquetas_operativas`

Discutido pero **no implementado todavía** (queda para un paso separado, según lo acordado). Resumen de la decisión tomada:

- `exitoso` (la etiqueta del clasificador que hoy dispara `desactivar_seguimiento` vía `etiquetas_pipeline.es_conversion`) se **eliminaría** como concepto — es una inferencia probabilística del clasificador (analiza los últimos 10 mensajes y "adivina" si hubo conversión), con el mismo riesgo de "dejarlo escapar" que ya vimos con la tool `desactivar_bot`.
- En su lugar, `etiquetas_operativas` se convertiría en la **única fuente de verdad** para ambas acciones (`desactivar_bot` y `desactivar_seguimiento`), usando etiquetas operativas reales y determinísticas (`agendado`, `compra-realizada`, etc.) en vez de inferencias del clasificador.
- **Antes de ejecutar esta migración**, hay que auditar que cada camino que hoy dispara `exitoso` (agendó videollamada, completó diagnóstico, o contrató un servicio — ver `prompt_clasificador` de agencIA) tenga su propia etiqueta operativa real conectada, para no perder cobertura. En particular, revisar si `registrar_pedido` aplica alguna etiqueta al completarse una compra.
- Para empresas futuras (ej. PC Outlet), esto simplifica el onboarding: en vez de inventar un `exitoso` artificial, se usa directamente la etiqueta operativa real del negocio (ej. `cliente-potencial`).

**Estado:** aprobado conceptualmente, pendiente de ejecución como paso separado.

---

## 7. Pendiente de esta implementación

- [x] Confirmar el nodo exacto que alimenta a `If_clasificar_lead` (`GET_etiquetas_actuales`) e insertar `PG_check_desactivar_bot` ahí.
- [x] Aplicar el cambio en n8n y verificar en el editor en vivo.
- [x] Probar caso 2 (mecanismo configurable): conversación clasificada como `lead-tibio` → forzó `desactivar_bot = true` correctamente (validado, conversación 1299).
- [x] Agregar sincronización a Chatwoot (`POST_sincronizar_desactivar_bot_chatwoot`), que faltaba en el camino `PG_actualizar_conversacion_directo`.
- [x] **Identificar y eliminar el segundo camino duplicado** (`If_forzar_desactivar_bot`) — ver sección 4.1.
- [x] Probar caso de regresión: conversación que agenda → `desactivar_bot` quedó correctamente en `false` (validado desde cero, conversación 1316/145, etiqueta `exitoso` aplicada sin disparar desactivación).
- [ ] Quitar la fila de prueba `lead-tibio` de `etiquetas_operativas` (sigue desactivada por la prueba de regresión) o reactivarla/decidir su uso definitivo.
- [ ] Documentar en Appsmith cómo administrar esta tabla (agregar panel similar al de `recordatorio_config`).
- [ ] **Desfase de un turno en `If_clasificar_lead`** (ver sección 7.1) — decidir si se corrige moviendo la verificación de posición en el flujo, o se deja documentado como limitación aceptada.
- [ ] Revisar el hallazgo relacionado en `POST_sincronizar_etiquetas_contacto` (sección 5.1) — mismo patrón de etiquetas hardcodeadas, en un nodo distinto.
- [ ] Ejecutar la migración de `es_conversion` hacia `etiquetas_operativas` (sección 6), una vez auditados todos los caminos de conversión.

### 7.1 Hallazgo: desfase de un turno en la verificación

La cadena `get_estado_conversacion → GET_etiquetas_actuales → PG_check_desactivar_bot → If_clasificar_lead` corre **antes** de que `AI_Agent_Clasificador` aplique la etiqueta del turno actual. Esto significa que `PG_check_desactivar_bot` siempre evalúa el estado de etiquetas **del turno anterior**, no el que se acaba de calcular en el turno presente.

**Evidencia:** en la primera prueba con `lead-caliente` (conversación 1287), el mecanismo no se disparó en el turno donde se aplicó la etiqueta por primera vez — se disparó en un turno posterior, una vez que la etiqueta ya estaba aplicada desde el turno anterior. Confirmado el mismo patrón con `lead-tibio` (conversación 1299).

**Impacto práctico:** el mecanismo funciona correctamente, pero con un retraso de un turno completo. En conversaciones reales (que normalmente tienen varios mensajes seguidos) esto no suele ser perceptible, pero es una limitación real a tener en cuenta.

**Decisión (2026-06-20):** se deja documentado como limitación conocida, sin corregir por ahora. Mover la verificación de posición en el flujo es un cambio más invasivo (cerca de la entrada al agente, zona sensible — ver [Bug 4](../troubleshooting/bugs-resueltos.md#bug-4-error-de-memoria-key-parameter-is-empty)) para un beneficio marginal en el comportamiento percibido por el cliente. Revisar si se vuelve prioritario más adelante.

---

## 8. Bitácora de cambios

| Fecha | Versión | Cambio | Autor |
|---|---|---|---|
| 2026-06-20 | 1.0 | **Bug resuelto y validado.** Hallazgo clave: existía un segundo camino duplicado (`If_forzar_desactivar_bot`) con el mismo patrón hardcodeado, no detectado en el diagnóstico inicial — explicaba por qué el bug reaparecía pese a haber corregido `If_clasificar_lead`. Eliminados `If_forzar_desactivar_bot`, `desactivar_bot_auto` y `PG_sync_desactivar_bot_auto`; unificado en un solo camino de control vía `PG_check_desactivar_bot`. Prueba de regresión limpia exitosa: conversación agendada con etiqueta `exitoso`, `desactivar_bot` permaneció en `false` correctamente. | Enuar |
| 2026-06-20 | 0.2 | Implementado y validado parcialmente: tabla `etiquetas_operativas` (con columna `accion`, pensada para alojar tambien la futura migracion de seguimiento), nodo `PG_check_desactivar_bot`, sincronizacion a Chatwoot agregada. Validado el mecanismo configurable con `lead-tibio`. Detectado y documentado el desfase de un turno. | Enuar |
| 2026-06-20 | 0.1 | Diagnóstico del bug (mezcla de etiquetas operativas y funcionales en `If_clasificar_lead`). Diseño aprobado del doble filtro con tabla configurable. | Enuar |
