# Módulo: Etiquetas, Seguimiento y Prioridad (Flujo principal)

Documenta cómo el workflow `Flujo principal agencIA` gestiona, en cada turno, tres efectos
ligados a la etiqueta funcional que asigna el clasificador: la etiqueta misma, el estado de
seguimiento (`en_seguimiento`) y la prioridad de la conversación en Chatwoot.

**Workflow n8n:** `Flujo principal agencIA`
**Versión:** post-sesión 22 jun 2026 (parte 2)

---

## 1. Los tres conceptos (no confundir)

| Concepto | Dónde vive | Para qué sirve | ¿En Chatwoot? |
|---|---|---|---|
| `es_conversion` | columna en `etiquetas_pipeline` | Reportes: marca objetivo del agente cumplido | NO (solo BD) |
| `desactivar_seguimiento` | acción en `etiquetas_operativas` | Operación: corta recordatorios | vía custom attr `en_seguimiento` |
| `chatwoot_prioridad` | columna en `etiquetas_pipeline` | Prioridad visual de la conversación | sí (campo priority) |

**Punto clave:** `es_conversion` (reportes) y `desactivar_seguimiento` (operación) son ejes
**independientes**. Una etiqueta puede ser conversión para reportes y aun así seguir recibiendo
recordatorios, o viceversa.

---

## 2. Flujo por turno (rama post-clasificador)

En cada turno, tras `AI_Agent_Clasificador` y `actualizar_etiqueta`, el flujo se ramifica a tres
efectos paralelos:

```
actualizar_etiqueta
  ├→ PG_actualizar_estado_lead       (persiste estado del lead)
  ├→ actualizar_prioridad            (refleja chatwoot_prioridad de la etiqueta actual)
  └→ PG_check_es_conversion          (consulta etiquetas_operativas: ¿desactivar seguimiento?)
       └→ If_es_conversion
            ├─ TRUE  → Chatwoot_actualizar_en_seguimiento + PG_actualizar_seguimiento  (OFF)
            └─ FALSE → Chatwoot_reactivar_en_seguimiento + PG_reactivar_seguimiento    (ON)
```

---

## 3. Seguimiento (`en_seguimiento`) — comportamiento

`PG_check_es_conversion` consulta `etiquetas_operativas`: si la etiqueta actual tiene
`accion = 'desactivar_seguimiento'` y `activo = true`, devuelve `es_conversion = true`.

- **TRUE** → `en_seguimiento` se apaga (false) en Chatwoot Y Postgres.
- **FALSE** → `en_seguimiento` se enciende (true) en Chatwoot Y Postgres.

Resultado: `en_seguimiento` **refleja el estado actual de la etiqueta en cada turno**. Si un lead
se enfría (deja de estar en una etiqueta de conversión), el seguimiento se reactiva
automáticamente.

### ⚠️ Override manual de seguimiento
Como `en_seguimiento` se re-evalúa cada turno, **apagarlo manualmente NO persiste**: el siguiente
turno lo reactivará si la etiqueta no es de conversión.

**Forma correcta de congelar una conversación manualmente:** activar `desactivar_bot`. Con el bot
desactivado, el flujo principal deja de procesar esa conversación turno a turno y no vuelve a
tocar `en_seguimiento`, manteniendo el estado.

**Regla:** no manipular `en_seguimiento` de forma aislada — lo maneja la lógica del programa. Para
intervención manual usar `desactivar_bot` (+ opcionalmente quitar `en_seguimiento`).

---

## 4. Prioridad — comportamiento

`actualizar_prioridad` toma `chatwoot_prioridad` de la etiqueta actual y lo escribe en Chatwoot
vía `PATCH /conversations/{id}`. Refleja el estado actual: sube o baja según la etiqueta.

### Manejo del valor `none`
La API de Chatwoot (4.14) NO acepta `priority: "none"` (string) → da error. Para **quitar** la
prioridad hay que mandar `priority: null` (JSON null).

`actualizar_prioridad` convierte automáticamente `none`/vacío → `null`:
- Etiqueta con prioridad `urgent`/`high`/`medium` → manda ese valor.
- Etiqueta con prioridad `none` (ej. `lead-frio`) → manda `null` (limpia la prioridad).

Tabla de prioridades por etiqueta (agencIA, en `etiquetas_pipeline`):

| etiqueta | chatwoot_prioridad |
|---|---|
| exitoso | urgent |
| lead-caliente | high |
| lead-tibio | medium |
| lead-frio | none → null |

---

## 5. Nodo `actualizar_prioridad_urgente` (tool del agente)
Tool que el agente puede invocar para forzar `priority: urgent` en casos especiales (independiente
del flujo automático por etiqueta). Usa el mismo endpoint PATCH. Nota: si el agente fuerza urgent
y luego la etiqueta cambia a una de menor prioridad, `actualizar_prioridad` la corregirá en el
siguiente turno (refleja la etiqueta actual).

---

## 6. Bitácora

| Fecha | Cambio |
|---|---|
| 2026-06-22 | Migración `es_conversion` → `etiquetas_operativas` (separar reportes de operación). Bug 20: reactivación de `en_seguimiento` en rama FALSE. Bug 21: prioridad maneja `none`→null y baja correctamente; eliminado nodo `If` redundante. |
