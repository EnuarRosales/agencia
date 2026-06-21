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

**Nota:** Hubo un intento previo de fix que cambió el `sessionKey` sin verificar primero cómo se generaba `session_id` — eso causó otro error ("No prompt specified") y tuvo que revertirse. Siempre verificar el origen real de un dato antes de cambiarlo.

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
