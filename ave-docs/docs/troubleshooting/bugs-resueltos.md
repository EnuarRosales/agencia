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
