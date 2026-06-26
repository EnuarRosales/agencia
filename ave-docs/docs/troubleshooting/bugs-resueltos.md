# Troubleshooting — Bugs Resueltos

Catálogo de bugs encontrados y resueltos en AVE, con su causa raíz y solución. Útil como referencia cuando aparezcan síntomas similares.

---

## Bug 1: Recordatorios nunca llegaban (401 Invalid Access Token)

**Síntoma:** El workflow de recordatorios se ejecutaba pero ningún mensaje llegaba. El nodo `GET_historial` fallaba con `401 - Invalid Access Token`.

**Causa raíz:** El nodo usaba una **credencial fija de Chatwoot** (Header Auth) con un token que se había vencido/rotado. Además, ese enfoque es estructuralmente incorrecto para multi-tenant: cada empresa tiene su propia cuenta de Chatwoot con su propio token.

**Solución:** Reemplazar la credencial fija por headers dinámicos por empresa:
```
sendHeaders: true
header: api_access_token = {{ $('Loop_empresas').item.json.chatwoot_api_key }}
```
Aplicar en todos los nodos HTTP que hablan con Chatwoot (`GET_historial`, `Chatwoot_enviar`).

**Lección:** En multi-tenant, nunca usar credenciales fijas para recursos que son por-empresa. Tomar el token dinámicamente del dato que ya trae cada empresa.

---

## Bug 2: SplitInBatches no avanzaba / "Source data is missing"

**Síntoma 1:** El loop se quedaba sin procesar, no avanzaba al siguiente nodo.
**Síntoma 2:** Al ejecutar el nodo aislado con "Execute step": `Source data is missing`.

**Causa raíz (síntoma 1):** Las salidas del `SplitInBatches` estaban invertidas. La salida 0 es "done" (cuando termina el loop) y la salida 1 es "loop" (item actual a procesar). El nodo siguiente estaba conectado a la salida 0 en vez de la 1.

**Causa raíz (síntoma 2):** Los nodos `SplitInBatches` **no se pueden ejecutar de forma aislada** con "Execute step" — necesitan el contexto del workflow completo corriendo desde el trigger, porque mantienen estado entre llamadas.

**Solución:**
- Conectar el procesamiento a la salida 1 (loop), dejar la salida 0 (done) vacía o hacia el cierre.
- Siempre probar con **"Execute Workflow"** completo, nunca "Execute step" sobre un SplitInBatches.

**Lección:** Los SplitInBatches tienen dos salidas con semántica específica (0=done, 1=loop) y requieren ejecución completa del workflow.

---

## Bug 3: Custom attributes se borran en cada turno

**Síntoma:** El custom attribute `en_seguimiento` (y `desactivar_bot`) se desmarcaba solo en cada mensaje, aunque ningún nodo de conversión se hubiera ejecutado.

**Causa raíz:** La API de Chatwoot en `POST /custom_attributes` **reemplaza el objeto completo**, no hace merge parcial. El nodo `actualizar_estado_lead` (que corre en cada turno) mandaba solo `{"estado_lead": "..."}`, borrando todos los demás custom attributes que no estuvieran en el payload.

**Solución:**
1. Agregar nodo `PG_get_conversacion_actual` que lee el estado real de `en_seguimiento_activo` y `desactivar_bot` antes de reescribir.
2. Hacer que `actualizar_estado_lead` mande el objeto **completo** con los 3 campos:
```json
{
  "custom_attributes": {
    "estado_lead": "{{ $('AI_Agent_Clasificador').item.json.output }}",
    "en_seguimiento": {{ $('PG_get_conversacion_actual').item.json.en_seguimiento_activo }},
    "desactivar_bot": {{ $('PG_get_conversacion_actual').item.json.desactivar_bot }}
  }
}
```
3. Aplicar el mismo principio en todos los nodos que escriben custom_attributes.

**Lección:** El endpoint `/custom_attributes` de Chatwoot es destructivo (reemplaza todo). Siempre enviar el objeto completo, leyendo primero el estado actual.

---

## Bug 4: Error de memoria "Key parameter is empty"

**Síntoma:** Error intermitente en el nodo `Postgres Chat Memory`: `Key parameter is empty`. El agente a veces respondía bien, a veces fallaba.

**Causa raíz:** Al insertar el nodo `If_conversacion_nueva` en el flujo, se conectó de forma que **sus DOS salidas alimentaban al `AI_Agent_Principal`**. Esto creaba una entrada ambigua al agente. Cuando el agente arranca y construye su contexto (`buildToolsAgentExecutionContext` → `getOptionalMemory`), la expresión del `sessionKey` (`{{ $('Contexto').item.json.session_id }}`) no se resolvía de forma consistente, dejando la clave vacía.

**Diagnóstico clave:** El webhook llegaba completo (con `conversation.id`), y el agente construía bien el prompt — el problema NO era falta de datos, sino la estructura del flujo con entrada ambigua al agente.

**Solución:** Restaurar la entrada **única y limpia** al agente:
```
PG_get_empresa → AI_Agent_Principal   (una sola conexión, directa)
```
Y mover la inicialización de `en_seguimiento` a una rama **paralela** que NO alimenta al agente:
```
PG_get_empresa → If_conversacion_nueva → Chatwoot_inicializar_en_seguimiento
```

**Lección:** Al insertar nodos en un flujo con un AI Agent, la entrada principal al agente debe ser una conexión única y limpia. La lógica adicional va en ramas paralelas, nunca convergiendo de vuelta a la entrada del agente.

**Nota:** Hubo un intento previo de fix que cambió el `sessionKey` sin verificar primero cómo se generaba `session_id` — eso causó otro error ("No prompt specified") y tuvo que revertirse. Siempre verificar el origen real de un dato antes de cambiarlo. (Ver Bug 27, donde finalmente se resolvió el `session_id` correctamente.)

---

## Bug 5: Desfase de zona horaria en comparaciones de tiempo

**Síntoma:** Recordatorios que no se enviaban pese a cumplir el tiempo, o `NOW() - ultimo_mensaje_usuario_at` daba valores negativos.

**Causa raíz:** La sesión de Postgres de n8n usa `Etc/UTC`, mientras DBeaver usa `America/Bogota`. Las columnas `timestamp WITHOUT time zone` se escribían/comparaban sin conversión, mezclando UTC con hora local.

**Solución:** Ver documento completo [reglas-timezone.md](../base-de-datos/reglas-timezone.md). Resumen:
- Columnas `WITHOUT time zone` → `NOW() AT TIME ZONE 'America/Bogota'`
- Columnas `WITH time zone` → `NOW()` puro

**Lección:** Distinguir siempre el tipo de columna (`WITH`/`WITHOUT` time zone) antes de comparar timestamps. No mezclar tratamientos.

---

## Bug 6: Conversaciones huérfanas (404 en GET_historial)

**Síntoma:** `GET_historial` fallaba con `404 - Resource could not be found`.

**Causa raíz:** La tabla `conversaciones` tenía registros con `chatwoot_conversation_id` que ya no existían en Chatwoot (conversaciones de prueba borradas directamente en Chatwoot sin reflejarse en la BD).

**Solución:** Limpiar o cerrar las conversaciones huérfanas:
```sql
UPDATE conversaciones SET estado = 'cerrada'
WHERE empresa_id = 1 AND chatwoot_conversation_id != [el que sí existe];
```

**Lección:** Borrar conversaciones de prueba en Chatwoot no las elimina de Postgres. Mantener sincronía o limpiar periódicamente los datos de prueba.

---

## Bug 7: Token de Google Calendar vencido (EAUTH)

**Síntoma:** El nodo `ver_disponibilidad` fallaba con error de autorización OAuth (`invalid_grant` / refresh token inválido).

**Causa raíz:** El refresh token de OAuth de Google Calendar se venció/revocó. Google revoca tokens si la app OAuth está en modo "Testing" (caducan a los 7 días) o por inactividad prolongada.

**Solución:**
1. En n8n → Credentials → credencial de Google Calendar → "Reconnect" / "Connect my account".
2. Completar el flujo OAuth de Google.
3. Si persiste: en Google Cloud Console, pasar la app OAuth de "Testing" a "Production" para tokens permanentes.

**Lección:** Los tokens OAuth de Google caducan en modo Testing. Para producción, la app debe estar publicada.

---

## Bug 8: Etiqueta no aparece en Chatwoot (carga infinita)

**Síntoma:** La página de Etiquetas en Chatwoot se quedaba cargando ("Obteniendo etiquetas") indefinidamente.

**Causa raíz:** Problema del navegador (sesión/cookie/caché corrupta), no del backend. En modo incógnito funcionaba.

**Solución:** Limpiar cookies del dominio de Chatwoot, o recargar con `Ctrl + Shift + R`, o cerrar y reabrir sesión.

**Lección:** Si algo falla solo en el navegador normal pero funciona en incógnito, es caché/cookies, no el sistema.

---

## Bug 9: Recurrencia del bug de timezone tras reconstrucción del workflow

**Síntoma:** Los recordatorios de una prueba nocturna llegaron ~5 horas tarde (esperados ~21:25 y ~21:55, llegaron a las 02:25 y 03:00) — el mismo síntoma del [Bug 5](#bug-5-desfase-de-zona-horaria-en-comparaciones-de-tiempo), que ya se había resuelto en una sesión anterior.

**Causa raíz:** Al reconstruir `Bot_Agencia_final` para resolver el [Bug 4 (memoria)](#bug-4-error-de-memoria-key-parameter-is-empty), los tres nodos que escriben timestamps (`PG_upsert_conversacion`, `PG_actualizar_conversacion`, `PG_actualizar_conversacion_directo`) volvieron a quedar con `NOW()` puro, sin `AT TIME ZONE 'America/Bogota'`. El fix de timezone se había aplicado y validado correctamente en una sesión previa, pero **no se confirmó que siguiera persistiendo** después de la siguiente ronda de cambios sobre el mismo workflow.

**Verificación que destapó el problema:** comparar directamente el query mostrado en el **editor de n8n en producción** (no un archivo exportado previamente) contra lo esperado. El archivo exportado que se tenía como referencia mostraba el fix correcto, pero lo que realmente corría en producción no lo tenía — confirmando que la fuente de verdad es siempre el editor en vivo, no un export anterior.

**Solución:**
1. Reaplicar `NOW()` → `NOW() AT TIME ZONE 'America/Bogota'` en los 3 nodos, verificando cada uno directamente en el editor de n8n.
2. Corregir los datos ya contaminados de la fila afectada:
```sql
UPDATE conversaciones
SET
  ultimo_mensaje_usuario_at = ultimo_mensaje_usuario_at - INTERVAL '5 hours',
  updated_at = updated_at - INTERVAL '5 hours'
WHERE id = [ID afectado];
```
3. Validar con un mensaje real nuevo que `ultimo_mensaje_usuario_at` quede a segundos de `NOW() AT TIME ZONE 'America/Bogota'`.

**Lección crítica:** Un fix aplicado y validado en una sesión puede perderse silenciosamente si el workflow se reconstruye o reimporta después por otro motivo (en este caso, arreglar un bug no relacionado). **Después de cualquier cambio en n8n, verificar inmediatamente contra el editor en vivo** — nunca asumir que un archivo exportado previamente sigue reflejando el estado real de producción. Ante cualquier bug que "ya se había resuelto antes", lo primero es confirmar en el editor en vivo si el fix sigue presente, antes de re-diagnosticar desde cero.

---

## Bug 10 (DESCARTADO tras verificación): `$now` en `{fecha_actual}` — sospecha de UTC

**Estado:** Investigado y **descartado**. `$now` en esta instancia de n8n está correctamente configurado en hora de Bogotá, de forma independiente al resto de la infraestructura.

**Hipótesis original:** `AI_Agent_Principal` arma el placeholder `{fecha_actual}` del system prompt con `$now.toFormat('dd/MM/yyyy')`. Como el sistema operativo del contenedor y la sesión de Postgres usada por n8n están en UTC, se sospechó que `$now` también podría estarlo, causando errores de un día completo en horario nocturno (7pm-medianoche Bogotá).

**Verificación realizada:** se agregó un nodo `Edit Fields` de prueba con `{{ $now.toFormat('dd/MM/yyyy HH:mm:ss') }}` y se comparó contra la hora real del reloj. Resultado: `$now` devolvió `10:48:50` cuando la hora real era `10:50` — coincide, sin desfase.

**Conclusión:** `$now` tiene su propia configuración de zona horaria a nivel de instancia de n8n (separada del SO del contenedor y de la sesión de Postgres), y en este caso está correctamente puesta en `America/Bogota`. No requiere ninguna corrección.

**Lección:** no asumir que todas las fuentes de tiempo de una plataforma comparten la misma configuración de zona horaria solo porque corren en la misma infraestructura. n8n, Postgres (vía sesión) y el sistema operativo pueden tener cada uno su propia configuración independiente — verificar cada una por separado antes de intervenir.

---

## Bug 11: `agendado` forzaba `desactivar_bot=true` vía un SEGUNDO camino no detectado

**Síntoma:** Tras corregir `If_clasificar_lead` (que tenía `['compra-realizada', 'agendado', 'desactivar_bot']` hardcodeado, forzando incorrectamente la desactivación al agendar), las pruebas seguían fallando de forma intermitente y aparentemente contradictoria — a veces el bug parecía corregido, a veces reaparecía, sin patrón claro a primera vista.

**Causa raíz:** Existía un **segundo nodo independiente**, `If_forzar_desactivar_bot`, con el mismo patrón hardcodeado (`['compra-realizada', 'agendado'].includes(e)`), en un camino **paralelo y separado** del flujo, que también convergía en el mismo nodo de efecto (`PG_actualizar_estado_lead_directo → PG_actualizar_conversacion_directo`, que fuerza `desactivar_bot = true` de forma incondicional). El diagnóstico inicial encontró y corrigió un solo camino, sin saber que existía una segunda puerta hacia el mismo destino.

**Cómo se destapó:** Un barrido cronológico completo de **todas** las ejecuciones de una conversación de prueba en n8n Executions (no solo la "última" ejecución asumida), revisando nodo por nodo cuál tomaba qué rama, hasta encontrar la ejecución exacta donde `If_forzar_desactivar_bot` tomó la rama verdadera.

**Solución:** Eliminar la duplicación en vez de parchear ambos `If` por separado — se borró `If_forzar_desactivar_bot` y los nodos que solo él alimentaba (`desactivar_bot_auto`, `PG_sync_desactivar_bot_auto`), dejando un único camino de control hacia el nodo de efecto, vía `If_clasificar_lead` + `PG_check_desactivar_bot` + tabla `etiquetas_operativas` (ver [módulo desactivar-bot](../modulos/desactivar-bot.md)).

**Lección crítica:** Cuando un nodo de efecto (uno que escribe en la base de datos o dispara una acción) puede ser alcanzado desde **más de un camino** en el flujo, corregir la condición de un solo camino de entrada no resuelve el problema — hay que mapear **todas** las puertas de entrada a ese nodo antes de dar un bug por cerrado. Si un fix aplicado correctamente sigue fallando de forma intermitente, es señal de que existe otra ruta no identificada hacia el mismo efecto, no de que el fix esté mal hecho. La forma confiable de encontrarla es el barrido cronológico completo de ejecuciones reales en n8n — no asumir cuál fue "la última" ejecución relevante sin verificarlo.

---

## Bug 12: Appsmith — `updatedRow` sin prefijo de tabla causa `affectedRows: 0` silencioso

**Síntoma:** Un query de actualización (`UPDATE`) en una tabla de Appsmith conectada a un widget Table, con `onSave`, se ejecutaba **sin errores** pero nunca persistía los cambios — `affectedRows` siempre devolvía `0`, sin importar qué fila o campo se editara.

**Causa raíz:** El query referenciaba la fila editada como `{{updatedRow.campo}}`, tratándolo como si fuera una variable global del contexto de Appsmith. En realidad, `updatedRow` (igual que `selectedRow`, `processedTableData`, etc.) es una **propiedad específica de cada widget de tabla** — debe referenciarse siempre con el nombre exacto del widget como prefijo: `{{nombreDelWidget.updatedRow.campo}}`. Sin el prefijo, la expresión no resuelve al objeto correcto y el `WHERE id = {{updatedRow.id}}` termina comparando contra un valor vacío/nulo, por lo que el `UPDATE` se ejecuta válidamente pero no encuentra ninguna fila que coincida.

**Cómo se destapó:** Comparación directa contra un query equivalente ya funcional en la misma aplicación (`update_etiqueta`, para una tabla distinta), que sí tenía el prefijo correcto (`{{tbl_etiquetas.updatedRow.id}}`).

**Solución:** Agregar el prefijo del widget en todas las referencias a `updatedRow` dentro del query:
```sql
UPDATE etiquetas_operativas SET
  etiqueta = '{{tbl_etiquetas_oper.updatedRow.etiqueta}}',
  ...
WHERE id = {{tbl_etiquetas_oper.updatedRow.id}};
```

**Lección:** En Appsmith, `affectedRows: 0` sin ningún error visible es la señal característica de este problema — el query es sintácticamente válido pero sus referencias a datos de la fila están mal resueltas. Cuando un query de update "no hace nada" sin fallar, **comparar contra un query equivalente que ya funcione en la misma app** es la forma más rápida de encontrar diferencias de sintaxis como esta, en vez de depurar desde cero.

---

## Bug 13: Condición de carrera por conectar una rama nueva al nodo con nombre engañoso

**Síntoma:** Una rama nueva del flujo (sincronización de etiquetas de pipeline al contacto, ver [módulo desactivar-bot](../modulos/desactivar-bot.md)) leía siempre la etiqueta del turno **anterior**, nunca la del turno actual — un mensaje de retraso constante.

**Causa raíz:** La rama nueva se conectó desde `actualizar_estado_lead`, asumiendo por su nombre que ese nodo actualiza el estado del lead en la base de datos. En realidad, `actualizar_estado_lead` es un nodo HTTP que **solo escribe el custom attribute en Chatwoot** — el `UPDATE` real de `leads.estado_lead` en Postgres ocurre en un nodo distinto y posterior en la cadena, `PG_actualizar_estado_lead` (nombre casi idéntico, fácil de confundir). Como la rama nueva arrancaba en paralelo al mismo tiempo que la cadena que llega hasta `PG_actualizar_estado_lead`, la lectura de Postgres ganaba la carrera contra la escritura real, trayendo siempre el valor desactualizado.

**Cómo se destapó:** Inspección directa del nodo `actualizar_estado_lead` en n8n (tipo de nodo y body real) en vez de confiar en el nombre — reveló que era un `httpRequest` hacia Chatwoot, no un nodo Postgres.

**Solución:** Mover el origen de la conexión nueva desde `actualizar_estado_lead` hacia `PG_actualizar_estado_lead` — el nodo que efectivamente termina de escribir el dato en Postgres.

**Lección crítica:** Cuando se conecta una rama nueva a un nodo existente basándose en su nombre, **verificar el tipo de nodo y su contenido real antes de asumir qué hace** — especialmente cuando existen dos nodos con nombres casi idénticos (`actualizar_estado_lead` vs `PG_actualizar_estado_lead`) que sugieren la misma función pero operan sobre sistemas distintos (Chatwoot vs Postgres). Esto refuerza la lección del Bug 11: los nombres de nodos importan, y nombres ambiguos o casi-duplicados son una fuente directa de bugs de timing difíciles de diagnosticar sin inspección directa.

---

## Bug 14: Casi se rompe `actualizar_etiqueta` por desconectar un nodo "solo de lectura" sin revisar referencias internas

**Síntoma:** Ninguno todavía — este bug se **evitó** antes de llegar a producción, gracias a una verificación previa. Se documenta como caso de estudio porque el error estuvo a punto de cometerse y la lección es igual de valiosa que un bug real.

**Lo que se intentó:** Al mover el mecanismo de verificación de `desactivar_bot` a un punto distinto del flujo (para eliminar el [desfase de un turno](../modulos/desactivar-bot.md)), el plan era desconectar `get_estado_conversacion → GET_etiquetas_actuales` de su posición original, asumiendo que esa cadena solo alimentaba la verificación que se estaba moviendo.

**Por qué casi fue un error grave:** Una búsqueda de **todas** las referencias `$('GET_etiquetas_actuales')` en el workflow (no solo sus conexiones de salida visibles) reveló que era usado por **otros nodos centrales del flujo principal** (`actualizar_etiqueta`, `PG_actualizar_conversacion`), ya validados y en producción. En n8n, un nodo ejecutado en cualquier punto de un flujo queda disponible para **cualquier nodo posterior en la misma ejecución**, sin importar la "rama" — así que `GET_etiquetas_actuales`, aunque corre al inicio del turno, su resultado se sigue usando más adelante. Desconectarlo habría roto ese flujo central.

**Cómo se evitó:** Antes de desconectar nada, se hizo una búsqueda de texto completa por el nombre del nodo en **todo** el JSON del workflow (no solo el grafo de conexiones), revelando todos sus usos reales.

**Lección crítica:** Las conexiones visibles en el canvas (el grafo de `connections`) **no son la única forma en que un nodo depende de otro** en n8n. Las expresiones `$('NombreDeNodo')` crean dependencias implícitas que no aparecen como líneas en el diagrama. Antes de mover o eliminar cualquier nodo, buscar **todas las referencias por nombre** en el JSON completo del workflow — no basta con revisar las conexiones de entrada/salida visibles.

---

## Bug 15: La tool del agente sincronizaba a Chatwoot pero un nodo posterior sobrescribía el valor con datos viejos de Postgres

**Síntoma:** El agente usaba la tool `desactivar_bot` correctamente (`true` se aplicaba en Chatwoot), pero segundos después, en el mismo turno, el valor volvía a `false` solo — visible en el log de Chatwoot como *"Automation System agregó desactivar_bot"* seguido de *"Automation System eliminó a desactivar_bot"*.

**Causa raíz:** La tool `desactivar_bot` del agente escribe **directo a Chatwoot** (custom attribute), pero **nunca actualiza Postgres** — no existía ningún mecanismo que sincronizara ese cambio hacia la base de datos. Más adelante en el mismo turno, el nodo `actualizar_estado_lead` reconstruye el custom attribute completo (necesario desde el [Bug 3](#bug-3-custom-attributes-se-borran-en-cada-turno), que exige mandar el objeto completo) usando como fuente el valor de `desactivar_bot` guardado en **Postgres** — que seguía en `false`, porque nunca se había sincronizado. Al reconstruir el objeto completo con ese dato viejo, sobrescribía el `true` que la tool acababa de poner instantes antes.

**Cómo se destapó:** Inspección directa de la ejecución real en n8n, turno por turno: se confirmó que `PG_get_conversacion_actual` (la fuente de datos de `actualizar_estado_lead`) devolvía `desactivar_bot: false`, mientras que el output de la tool, momentos antes, había sido `true`.

**Solución:** Insertar una lectura **fresca** del estado real en Chatwoot (`GET /conversations/{id}`, que captura cualquier cambio hecho por la tool en el mismo turno) justo antes de que `actualizar_estado_lead` reconstruya el custom attribute — usando ese valor fresco en vez del valor potencialmente desactualizado de Postgres. Se agregó además un nodo que sincroniza ese mismo valor fresco hacia Postgres, cerrando finalmente la sincronización tool↔base de datos que nunca había existido.

**Lección crítica:** Cuando un valor se escribe en un sistema externo (Chatwoot) sin sincronizarlo a la base de datos propia, cualquier nodo posterior que "reconstruya el estado completo" usando la base de datos como fuente de verdad **revertirá silenciosamente** ese cambio externo, sin ningún error visible — porque desde la perspectiva de ese nodo, simplemente está haciendo su trabajo con los datos que tiene. La fuente de verdad real, en estos casos, debe ser el sistema que **acaba de cambiar** (Chatwoot, recién escrito por la tool), no la base de datos que aún no se enteró del cambio.
---

# Bugs resueltos — Workflow AVE_Recordatorios_Automaticos (sesión 22 jun 2026)

> Cubre los bugs 16 a 19, todos en el workflow `AVE - Recordatorios Automaticos`.

---

## Bug 16 — Inconsistencia de timezone y tipo de columna

**Workflow:** `AVE - Recordatorios Automaticos`
**Nodos:** `PG_get_conversaciones_pendientes`, `PG_marcar`
**Tabla:** `conversaciones` (columna `ultimo_recordatorio_at`)

### Síntoma
El cálculo de tiempo del segundo recordatorio en adelante quedaba desfasado respecto al primero.

### Causa raíz (doble)
1. **Tipo de columna inconsistente:** `ultimo_recordatorio_at` era `timestamp WITH time zone`,
   mientras `ultimo_mensaje_usuario_at`, `created_at` y `updated_at` eran `timestamp WITHOUT time zone`.
   Dos sistemas de tiempo conviviendo en el mismo nodo.
2. **`NOW()` sin huso en la rama ELSE:** la rama del primer recordatorio usaba
   `(NOW() AT TIME ZONE 'America/Bogota')`, pero la rama ELSE (orden 2+) usaba `NOW()` puro.
   Mismo patrón que el Bug 9.

### Fix
Migración de la columna rebelde a hora Bogotá sin zona:
```sql
ALTER TABLE conversaciones
  ALTER COLUMN ultimo_recordatorio_at TYPE timestamp without time zone
  USING ultimo_recordatorio_at AT TIME ZONE 'America/Bogota';
```
`PG_get_conversaciones_pendientes` rama ELSE y `PG_marcar` → ambos usan
`NOW() AT TIME ZONE 'America/Bogota'`. Las cuatro columnas temporales quedan
`timestamp without time zone` en hora Bogotá.

### Regla derivada
La regla no es "siempre `AT TIME ZONE`", sino **coherencia entre lo que se escribe y lo que se compara**.
Verificar siempre el `data_type` antes de aplicar `AT TIME ZONE`:
- `timestamp without time zone` → `AT TIME ZONE 'America/Bogota'` correcto (modelo AVE).
- `timestamp with time zone` → aplicar `AT TIME ZONE` lo convierte a sin-zona desplazado −5h; reintroduce el bug.

```sql
SELECT column_name, data_type FROM information_schema.columns
WHERE table_name = 'conversaciones' AND column_name LIKE '%_at';
```

---

## Bug 17 — Loop de empresas moría en la primera sin pendientes

**Nodos:** `If_hay_pendientes`, `Loop_empresas`

### Síntoma
Solo se procesaba la primera empresa (agencIA). Uhane y PC_Outlet nunca recibían recordatorios.

### Causa raíz
La rama FALSE de `If_hay_pendientes` no estaba conectada a nada. Cuando una empresa no tenía
pendientes, el flujo se detenía ahí y nunca regresaba a `Loop_empresas` para procesar las siguientes.

### Fix
Conectar la rama FALSE de `If_hay_pendientes` de regreso a `Loop_empresas`:
- TRUE  → `Loop_conversaciones` (procesa)
- FALSE → `Loop_empresas` (siguiente empresa)

---

## Bug 18 — Nodo Postgres detenía el flujo al devolver vacío

**Nodo:** `PG_get_conversaciones_pendientes`

### Síntoma
El workflow se detenía silenciosamente cuando una empresa no tenía conversaciones pendientes.
n8n muestra: "n8n stops executing the workflow when a node has no output data."

### Causa raíz
Por defecto, un nodo sin filas de salida termina la ejecución de esa rama. Como agencIA
(primera empresa) no tenía pendientes, cortaba todo antes de llegar a `If_hay_pendientes`,
dejando inútil el fix del Bug 17.

### Fix
Activar **"Always Output Data"** (`alwaysOutputData: true`) en `PG_get_conversaciones_pendientes`.
Así emite un item (vacío) aunque no haya filas, y `If_hay_pendientes` puede evaluarlo y enrutar.

> Nota: los tres fixes (17, 18 y "Continue On Fail" del 19) son interdependientes. En workflows
> con loops anidados en n8n se necesitan los tres juntos, o cualquier caso borde detiene el lote.

---

## Bug 19 — Conversación borrada en Chatwoot tumbaba el lote (+ Capa 2: auto-desactivación)

**Nodos:** `GET_historial`, `OpenAI_generar`, `Chatwoot_enviar` (Capa 1);
`If_conversacion_existe`, `PG_marcar_fantasma` (Capa 2, nuevos)

### Síntoma
Al intentar enviar a una conversación borrada/cerrada en Chatwoot, la API devolvía 404 y la
ejecución se detenía, dejando sin procesar las conversaciones siguientes del lote.

### Causa raíz
Las conversaciones borradas en Chatwoot seguían marcadas como `estado = 'abierta'` y
`en_seguimiento_activo = true` en Postgres. El workflow las reintentaba en cada ciclo, y el 404
sin manejo rompía el lote.

### Fix — Capa 1 (red de seguridad)
"Continue On Fail" (`onError: continueRegularOutput`) en los tres nodos HTTP: `GET_historial`,
`OpenAI_generar`, `Chatwoot_enviar`. Un 404 puntual salta esa conversación sin tumbar el lote.

### Fix — Capa 2 (auto-sincronización)
Detectar la conversación inexistente y desactivarla en Postgres para no reintentarla nunca más:
- `GET_historial` → `If_conversacion_existe` (condición: `{{ $json.payload }}` **exists**)
  - TRUE (existe payload) → `Code_historial` → flujo normal
  - FALSE (404, sin payload, item con `error`) → `PG_marcar_fantasma` → `Loop_conversaciones`
- `PG_marcar_fantasma`:
```sql
UPDATE conversaciones SET
  en_seguimiento_activo = false,
  updated_at = NOW() AT TIME ZONE 'America/Bogota'
WHERE id = {{ $('Loop_conversaciones').item.json.conversacion_id }};
```

### Detalle técnico clave (n8n 2.23.3)
Cuando un nodo HTTP con `continueRegularOutput` falla, n8n **reemplaza el item por un objeto
`error`** (no deja el payload vacío). Por eso la condición "¿existe `payload`?" detecta
correctamente el fallo: el item fantasma no tiene `payload`, tiene `error`, y cae por FALSE.

> Nota: el comportamiento de `PG_marcar_fantasma` se refinó después en el Bug 25 — un 404
> transitorio ya no desactiva; solo se marca fantasma si `GET_estado_conversacion` confirma que
> la conversación realmente no existe.

### Por qué `en_seguimiento_activo = false` y no `estado = 'cerrada'`
Acción quirúrgica: solo apaga los recordatorios sin afectar otros procesos que leen `estado`
(flujo principal, dashboard, reportes). Que una conversación se borre en Chatwoot significa
"ya no puedo enviarle recordatorios", no necesariamente "negocio cerrado". Coherente con la
desactivación manual ya usada en AVE.

### Validación
Probado con conversación real borrada en Chatwoot (chatwoot_conversation_id=100): `GET_historial`
dio 404 → `If_conversacion_existe` la mandó por FALSE → `PG_marcar_fantasma` ejecutó con
`success: true` → confirmado en BD `en_seguimiento_activo = false`.

---

# Bugs y cambios — Flujo principal agencIA (sesión 22 jun 2026, parte 2)

> Cubre los bugs 20 y 21 y la migración de `es_conversion`, en `Flujo principal agencIA`.

---

## Migración: es_conversion → etiquetas_operativas (acción desactivar_seguimiento)

### Contexto / problema
El nodo `PG_check_es_conversion` leía la columna `es_conversion` de `etiquetas_pipeline`
para decidir si apagar el seguimiento automático (recordatorios). Esto mezclaba dos conceptos
distintos en una sola fuente:

- **`es_conversion`** (columna en `etiquetas_pipeline`): marca si se cumplió el objetivo del
  agente. Sirve para **reportes**. Existe solo en BD, NO en Chatwoot.
- **`desactivar_seguimiento`** (acción en `etiquetas_operativas`): comportamiento **operativo** —
  dejar de enviar recordatorios.

Son **ejes independientes**: una etiqueta puede contar como conversión para reportes y aun así
seguir recibiendo recordatorios (ej. PC Outlet con `cliente-potencial`), o viceversa.

### Solución
Se mantienen separados:
- `etiquetas_pipeline.es_conversion` → se queda. Solo para reportes. NO se toca.
- `etiquetas_operativas` con `accion = 'desactivar_seguimiento'` → controla el corte de
  recordatorios, configurable por etiqueta/empresa desde Appsmith.

El nodo `PG_check_es_conversion` se migró para leer de `etiquetas_operativas`, conservando el
alias de salida `es_conversion` (para no tocar `If_es_conversion`):

```sql
SELECT COALESCE(
  (SELECT true FROM etiquetas_operativas eo
   WHERE eo.empresa_id = (SELECT id FROM empresas WHERE chatwoot_account_id = {{ $('Contexto').item.json.account_id }} LIMIT 1)
     AND eo.accion = 'desactivar_seguimiento'
     AND eo.activo = true
     AND eo.etiqueta = '{{ $('AI_Agent_Clasificador').item.json.output }}'
   LIMIT 1),
  false
) AS es_conversion;
```

### Datos insertados (jun 2026)
```sql
INSERT INTO etiquetas_operativas (empresa_id, etiqueta, accion, activo)
VALUES
  (1, 'exitoso',          'desactivar_seguimiento', true),
  (7, 'compra-realizada', 'desactivar_seguimiento', true);
-- PC Outlet (8) NO se incluye: cliente-potencial cuenta como conversión para reportes
-- pero debe seguir recibiendo recordatorios.
```

---

## Bug 20 — en_seguimiento se apagaba pero nunca se reactivaba

**Workflow:** `Flujo principal agencIA`
**Nodos:** `If_es_conversion` y rama FALSE (nuevos: `Chatwoot_reactivar_en_seguimiento`, `PG_reactivar_seguimiento`)

### Síntoma
Tras marcar un lead como `exitoso` (que apaga `en_seguimiento`), si el lead se enfriaba a
`lead-frio`, el seguimiento **permanecía desactivado** — no se reactivaba pese a que la etiqueta
ya no era de conversión.

### Causa raíz
`If_es_conversion` tenía solo la rama TRUE conectada (apaga seguimiento en Chatwoot + Postgres).
La rama FALSE estaba vacía: el sistema solo sabía **apagar**, nunca **encender**. Lógica de una
sola dirección.

### Fix
Conectar la rama FALSE a dos nodos nuevos que reactivan en ambos lados:
- `Chatwoot_reactivar_en_seguimiento` → POST custom_attributes con `en_seguimiento: true`
- `PG_reactivar_seguimiento` → `UPDATE conversaciones SET en_seguimiento_activo = true ...`

Ahora `en_seguimiento` refleja el estado actual de la etiqueta en cada turno:
- Etiqueta con `accion = desactivar_seguimiento` → OFF.
- Cualquier otra etiqueta → ON.

### Nota operativa importante (override manual)
Como consecuencia del fix, `en_seguimiento` se **re-evalúa y reactiva en cada turno** según la
etiqueta actual. Esto significa que **apagar `en_seguimiento` manualmente NO persiste**: el
siguiente turno lo reactivará si la etiqueta no es de conversión.

**Forma correcta de congelar una conversación (intervención manual):** activar `desactivar_bot`.
Al desactivar el bot, el flujo principal deja de procesar esa conversación turno a turno, por lo
que tampoco vuelve a tocar `en_seguimiento`, y el estado se mantiene.

**Regla:** NO manipular `en_seguimiento` de forma aislada — es una variable que maneja la lógica
del programa (se reactiva según etiqueta cada turno). Para intervención manual, usar
`desactivar_bot` + opcionalmente quitar `en_seguimiento`.

---

## Bug 21 — La prioridad subía pero nunca bajaba

**Workflow:** `Flujo principal agencIA`
**Nodos:** `actualizar_prioridad`, nodo `If` (eliminado)

### Síntoma
Cuando un lead pasaba de una etiqueta de prioridad alta (ej. `exitoso` = urgent) a una de
prioridad baja (`lead-frio` = none), la prioridad en Chatwoot **se mantenía en el valor alto** —
no bajaba.

### Causa raíz
Un nodo `If` evaluaba si la prioridad de la etiqueta era `'none'`:
- TRUE (es none) → rama vacía (no hacía nada).
- FALSE (no es none) → `actualizar_prioridad`.

La intención era evitar mandar `priority: "none"` a Chatwoot (que da error). Pero el efecto
colateral era que las etiquetas con prioridad `none` (como `lead-frio`) nunca actualizaban la
prioridad, dejando el valor anterior.

### Hallazgo técnico (Chatwoot API 4.14)
- `PATCH /conversations/{id}` con `priority: null` → **200**, limpia la prioridad. ✅
- `PATCH /conversations/{id}` con `priority: "none"` (string) → error. ❌
- El valor `null` (JSON) es la forma correcta de quitar prioridad, no el string `"none"`.

### Fix
1. `actualizar_prioridad` ahora convierte `none`/vacío → `null` en el jsonBody:
```
"priority": {{ (() => { const p = (...).chatwoot_prioridad; return (!p || p === 'none') ? "null" : JSON.stringify(p); })() }}
```
2. Se eliminó el nodo `If` (quedó redundante: ya no hace falta filtrar `none`).
   `actualizar_etiqueta` conecta directo a `actualizar_prioridad`, que ahora corre siempre y
   refleja la prioridad real de la etiqueta actual (urgent/high/medium o limpia).

### Validación
Probado en vivo con conversación 161 (agencIA):
- `exitoso` → prioridad urgent, seguimiento OFF.
- Etiquetas intermedias → seguimiento ON (Bug 20).
- `lead-frio` → seguimiento ON + prioridad limpiada (Bug 21).
Los tres comportamientos correctos.

---

# Bugs resueltos — Validación desactivar_bot en recordatorios (sesión 23 jun 2026)

> Bugs 22 a 26. Workflows: `AVE - Recordatorios Automaticos` y `Flujo principal agencIA`.
> Contexto: al implementar que el workflow de recordatorios NO envíe a conversaciones con el bot
> desactivado (validando contra Chatwoot, no solo Postgres), se destaparon varios bugs encadenados
> que impedían que CUALQUIER recordatorio se enviara — uno tapaba al otro.

---

## Bug 22 — en_seguimiento nacía desincronizado en conversaciones nuevas

**Workflow:** `Flujo principal agencIA`
**Nodo:** `PG_upsert_conversacion`

### Síntoma
Conversaciones nuevas tenían `en_seguimiento = true` en Chatwoot pero `en_seguimiento_activo = false`
en Postgres. Como el filtro de recordatorios lee Postgres, esas conversaciones nunca eran elegibles
y no recibían recordatorios.

### Causa raíz
`Chatwoot_inicializar_en_seguimiento` ponía `en_seguimiento: true` en Chatwoot, pero el INSERT de
`PG_upsert_conversacion` NO incluía la columna `en_seguimiento_activo`, así que quedaba en su default
(`false`). Ambos lados nacían desincronizados.

### Fix
Agregar `en_seguimiento_activo = true` al INSERT (NO al `ON CONFLICT`, que se deja intacto para no
reactivar el seguimiento de leads que ya convirtieron cuando vuelven a escribir).

### Validación
Conversación nueva 165 nació con `en_seguimiento_activo = true` sin UPDATE manual.

---

## Bug 23 — If_hay_pendientes fallaba por validación de tipo estricta

**Workflow:** `AVE - Recordatorios Automaticos`
**Nodos:** `If_hay_pendientes` (y por extensión los tres nodos `If`)

### Síntoma
El nodo fallaba con: `Wrong type: '38' is a number but was expecting a string [condition 0, item 0]`.
El flujo se detenía ahí y NINGÚN recordatorio avanzaba. Este error estuvo oculto detrás de otros
síntomas durante todo el diagnóstico.

### Causa raíz
La condición usaba operador tipo `string` / `notEmpty` con `typeValidation: strict`, pero
`conversacion_id` llega como número (ej. 38). En modo strict, n8n rechaza la comparación.

### Fix
- `typeValidation: strict` → `loose` en los tres nodos If (`If_hay_pendientes`,
  `If_conversacion_existe`, `If_bot_activo`).
- `If_hay_pendientes`: operador `notEmpty` → `exists` (no depende del tipo; funciona con número o
  string, y da false para el item vacío que emite "Always Output Data").

---

## Bug 24 — GET_historial con referencia $json rota tras insertar nodos intermedios

**Workflow:** `AVE - Recordatorios Automaticos`
**Nodo:** `GET_historial`

### Síntoma
`GET_historial` daba 404 SIEMPRE en la ejecución del workflow, aunque el mismo endpoint daba 200 por
curl. Esto disparaba la lógica de "fantasma" y nada se enviaba. Fue la causa raíz real del
"no llega el recordatorio".

### Causa raíz
La URL usaba `{{ $json.chatwoot_conversation_id }}`. Funcionaba cuando `GET_historial` recibía directo
el item de `Loop_conversaciones`. Pero al insertar `GET_estado_conversacion` + `If_bot_activo` en
medio, el `$json` que llega a `GET_historial` pasó a ser la **respuesta de Chatwoot** (estructura
`meta`/`id`/`custom_attributes`), que NO tiene el campo `chatwoot_conversation_id`. La referencia
resolvía a vacío → URL malformada → 404.

### Fix
Referencia explícita al nodo correcto:
```
{{ $('Loop_conversaciones').item.json.chatwoot_conversation_id }}
```

### Lección
Refuerza Bug 13 y 14: al insertar nodos en medio de un flujo, las referencias `$json` (que apuntan al
nodo inmediatamente anterior) se rompen silenciosamente. Para datos que vienen de un nodo específico,
usar referencias explícitas `$('NombreNodo').item.json.campo`, no `$json`.

---

## Bug 25 — PG_marcar_fantasma desactivaba conversaciones válidas por 404 transitorio

**Workflow:** `AVE - Recordatorios Automaticos`
**Nodos:** `If_conversacion_existe`, nuevo `If_estado_existe`

### Síntoma
Un 404 puntual de `GET_historial` (incluso transitorio, por timing o conversación recién creada)
hacía que `If_conversacion_existe` mandara la conversación a `PG_marcar_fantasma`, desactivándola
(`en_seguimiento_activo = false`) permanentemente. Conversaciones válidas quedaban desactivadas por
un error temporal.

### Causa raíz
La detección de "fantasma" recaía en `GET_historial` (`/messages`), que puede dar 404 transitorio.
Un solo 404 no debería significar "conversación borrada para siempre".

### Fix
Mover la detección de fantasma real al endpoint `/conversations/{id}` (`GET_estado_conversacion`),
que es la fuente de verdad sobre existencia. Nuevo nodo `If_estado_existe`:
```
GET_estado_conversacion → If_estado_existe
   ├─ EXISTE (custom_attributes presente) → If_bot_activo → ... → GET_historial → If_conversacion_existe
   │                                                                  ├─ tiene payload → envía
   │                                                                  └─ 404 transitorio → vuelve al loop (NO desactiva)
   └─ NO EXISTE (404 real) → PG_marcar_fantasma
```
Solo se marca fantasma si la conversación realmente no existe. Un 404 de `/messages` con la
conversación existente salta el envío y reintenta en el próximo ciclo, sin desactivar.

---

## Bug 26 — Validación de desactivar_bot fresco en recordatorios (mejora doble capa)

**Workflow:** `AVE - Recordatorios Automaticos`
**Nodos nuevos:** `GET_estado_conversacion`, `If_bot_activo`, `PG_sync_desactivar_bot`

### Problema
Si un humano desactivaba el bot manualmente en Chatwoot, Postgres podía quedar desfasado
(`desactivar_bot = false`), y el recordatorio se enviaba pese al bot desactivado. Mismo patrón de
desfase que el Bug 15.

### Solución (doble capa)
1. **Filtro Postgres** (`PG_get_conversaciones_pendientes`): `AND c.desactivar_bot = false`. Primera
   barrera eficiente — si Postgres ya está sincronizado, la conversación ni entra al loop.
2. **Validación fresca** (`GET_estado_conversacion` + `If_bot_activo`): lee `desactivar_bot` real de
   Chatwoot. Si está desactivado → `PG_sync_desactivar_bot` actualiza Postgres a `true` y NO envía.
   Atrapa el caso de desfase y de paso lo corrige para el futuro.

### Validación (3 casos)
- Bot activo + en seguimiento (conv 163) → **envía** ✅
- Bot desactivado en Postgres (conv 164) → bloqueado por filtro inicial, no envía ✅
- Bot desactivado solo en Chatwoot, Postgres desfasado (conv 161) → GET fresco lo detecta, no envía
  + sincroniza Postgres ✅

### Nota
La validación fresca consume un GET extra a Chatwoot por conversación pendiente; como el filtro de
Postgres descarta la mayoría, el costo es bajo. Todos los nodos HTTP nuevos tienen "Continue On Fail".

---

# Bug 27 — Colisión de memoria entre empresas por session_id sin account_id (sesión 25 jun 2026)

**Workflow:** `Flujo principal agencIA`
**Nodo:** `Contexto` (Set) → consumido por `Postgres Chat Memory`
**Resuelve el riesgo #3 que estaba pendiente en la documentación de arquitectura.**

### Síntoma
El bot a veces respondía con la identidad de la empresa equivocada (el agente de Uhane respondía como
si fuera agencIA) y, en un caso, saludó a un usuario con el nombre de otra persona ("Idier Loiza") de
una conversación distinta. Comportamiento **intermitente**.

### Causa raíz
La memoria conversacional (tabla `n8n_chat_histories`) se indexa por `session_id`, generado en el
nodo `Contexto` **solo con el `conversation_id`** de Chatwoot:
```
session_id = {{ $('Webhook').item.json.body.conversation.id }}
```
Cada cuenta de Chatwoot numera sus conversaciones de forma independiente. La conversación 103 de
agencIA (account 1) y la 103 de Uhane (account 3) generaban el mismo `session_id = 103` →
**compartían memoria**. El agente leía el historial de la otra empresa y heredaba su contexto. El
carácter intermitente se explica porque solo colisiona cuando dos conversaciones de empresas
distintas coinciden en número.

### Confirmación en datos
```sql
SELECT DISTINCT session_id FROM n8n_chat_histories ORDER BY session_id;
```
Mostraba mayoría de IDs "pelados" (`100`, `101`, `103`...) sin prefijo de cuenta.

### Fix (dos partes)
En el nodo `Contexto`, campo `session_id`:
1. **Valor** → combinar cuenta + conversación:
   ```
   session_id = {{ $('Webhook').item.json.body.account.id }}_{{ $('Webhook').item.json.body.conversation.id }}
   ```
2. **Tipo de campo** → cambiar de **Number** a **String**.

> El punto 2 es crítico y es la razón por la que un intento anterior falló y tuvo que revertirse
> (relacionado con la nota del Bug 4): con el campo tipado como Number, n8n rechazaba `1_168` con el
> error `'session_id' expects a number but we got '1_168'`, que aguas abajo se manifestaba como
> "No prompt specified". Cambiar el tipo a String lo resuelve.

Resultado: agencIA → `1_168`, Uhane → `3_111`. Memorias aisladas por empresa.

### Limpieza de memoria colisionada
Como el sistema estaba **pre-producción**, se vació la tabla para eliminar historiales ya
colisionados y arrancar limpio:
```sql
TRUNCATE TABLE n8n_chat_histories;
```

### Validación
1. `SELECT DISTINCT session_id ...` → solo formatos `cuenta_conversacion` (`1_168`, `3_111`). ✅
2. Prueba funcional: nombre distinto en cada empresa, luego "¿cómo me llamo?" en cada una → cada
   empresa recuerda el nombre correcto, sin cruces. ✅

### Lección
El `session_id` de memoria en un sistema multi-tenant DEBE incluir el identificador de tenant
(`account_id`), no solo el `conversation_id`. Y en nodos Set de n8n, un campo con valor compuesto
(con separadores como guion bajo) debe tiparse como String, no Number.
