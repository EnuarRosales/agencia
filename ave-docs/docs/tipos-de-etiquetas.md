# Tipos de Etiquetas en AVE

Documento de referencia transversal. AVE usa "etiquetas" (labels en Chatwoot) para varios propósitos distintos, que conviene no confundir. Esta distinción nació como hallazgo dentro del módulo [desactivar-bot.md](modulos/desactivar-bot.md) — confundir los dos primeros tipos fue la causa raíz de un bug serio (ver [Bug original](modulos/desactivar-bot.md#4-comportamiento-anómalo-detectado-bug-pre-existente)). Se documenta aquí, a nivel de plataforma, porque aplica a cualquier módulo futuro que toque etiquetas — no es exclusivo de `desactivar_bot`.

---

## Resumen rápido

| Tipo | ¿Quién la genera? | ¿Dónde se configura? | ¿Varía por empresa? | Comportamiento al sincronizar a un contacto/destino |
|---|---|---|---|---|
| **Funcional / calificación** | `AI_Agent_Clasificador` (IA) | Tabla `etiquetas_pipeline` | Sí | Reemplaza siempre — una sola a la vez |
| **Operativa (configurable)** | Tools específicas del agente, o reglas configuradas | Tabla `etiquetas_operativas` | Sí | Reemplaza — refleja el estado actual |
| **Fija del sistema** | Eventos determinísticos del backend (ej. agendamiento confirmado) | Hardcodeada en el query correspondiente | No | Reemplaza — refleja el estado actual |

Las tres categorías, pese a su origen distinto, comparten una regla común desde 2026-06-21: **todas reflejan el estado presente**, ninguna se acumula como historial permanente. Si algo deja de aplicar, se quita de donde se haya sincronizado.

---

## 1. Etiquetas funcionales / de calificación

Viven en la tabla **`etiquetas_pipeline`**. Son el resultado del `AI_Agent_Clasificador`, que analiza cada conversación y **siempre** devuelve exactamente una de ellas. Representan **en qué etapa de calificación está el lead**.

Ejemplo (agencIA): `exitoso`, `lead-caliente`, `lead-tibio`, `lead-frio`.

**Características:**
- Son **mutuamente excluyentes** — una conversación está en una sola en un momento dado.
- Las genera el clasificador automáticamente en cada turno — es una **inferencia de IA**, no un hecho determinístico (puede "dejarse escapar" igual que cualquier decisión del modelo).
- Controlan el [sistema de seguimiento](modulos/seguimiento.md) vía la columna `es_conversion` (mecanismo en revisión — ver [decisión futura de migración](modulos/desactivar-bot.md#6-decisión-futura-relacionada-migrar-es_conversion-a-etiquetas_operativas)).

---

## 2. Etiquetas operativas (configurables)

Se aplican desde **tools específicas** del agente o se detectan mediante reglas configurables, no desde el clasificador. Representan **una acción operativa que debe ocurrir**, no una etapa de calificación.

Ejemplo: `desactivar_bot` (tool del agente, y/o regla configurable en `etiquetas_operativas`).

**Características:**
- **No son excluyentes entre sí ni con las de calificación** — una conversación puede estar en `lead-caliente` (funcional) y tener `desactivar_bot` activo (operativa) al mismo tiempo.
- Viven en la tabla **`etiquetas_operativas`**, con columnas `empresa_id`, `etiqueta`, `accion`, `activo` — configurable desde Appsmith, sin tocar n8n. Ver [administración en Appsmith](modulos/desactivar-bot.md#52-administración-desde-appsmith).
- La columna `accion` permite que una misma tabla sirva para múltiples propósitos (`desactivar_bot` hoy; `desactivar_seguimiento` preparado para el futuro).
- Cada empresa decide qué etiquetas disparan qué acción — esto es lo que las hace **configurables**, a diferencia del tercer tipo.

---

## 3. Etiquetas fijas del sistema

Etiquetas que **no son configurables por el usuario ni dependen de ninguna tabla** — son eventos que el sistema aplica siempre, de forma fija, cuando ocurre algo concreto y verificable.

Ejemplo: `agendado`. Se aplica automáticamente cuando `registrar_cita_pg` confirma que una cita quedó guardada en la base de datos.

**Características:**
- No varían por empresa — el evento es el mismo para todas (aunque cada empresa puede o no usar el módulo de agendamiento).
- No se activan/desactivan desde Appsmith — están hardcodeadas directamente en el query que las detecta, porque representan un hecho, no una regla de negocio.
- A diferencia de las dos categorías anteriores (que dependen de una inferencia de IA o de configuración), estas son **determinísticas**: o el evento ocurrió, o no ocurrió.

**Implementación actual:** `agendado` es, por ahora, la única etiqueta de este tipo, detectada en `PG_get_pipeline_y_operativas_lead` con un `CASE WHEN 'agendado' = ANY(c.etiquetas)...`. Si en el futuro aparecen más, conviene generalizar el patrón en vez de repetir un `CASE` por cada una.

`compra-realizada` existía en el diseño original pero **no está en uso** — no hay confirmación de que el flujo de pedidos (`registrar_pedido`) aplique ninguna etiqueta al completarse una compra.

---

## Regla de diseño para clasificar una etiqueta nueva

Antes de decidir dónde vive una etiqueta nueva, hacerse estas preguntas en orden:

1. **¿La genera el clasificador de IA, analizando la conversación?** → Funcional, va en `etiquetas_pipeline`.
2. **¿Es una regla de negocio que el cliente podría querer cambiar, activar o desactivar?** → Operativa configurable, va en `etiquetas_operativas` con su `accion` correspondiente.
3. **¿Es un evento determinístico del backend, igual para todas las empresas, sin necesidad de configuración?** → Fija del sistema, hardcodeada en el query que corresponda.

**Nunca mezclar dos tipos en la misma tabla ni en la misma lógica de control** — esa mezcla fue exactamente la causa raíz del bug documentado en el módulo `desactivar-bot.md`.

---

## Dónde se sincronizan hoy

El único punto actual de sincronización de etiquetas hacia un contacto de Chatwoot es `POST_sincronizar_labels_contacto_cada_turno`, dentro del módulo `desactivar-bot.md`, que combina las tres categorías en una sola llamada, corriendo en cada turno. Ver [sección 5.1 de ese módulo](modulos/desactivar-bot.md#51-hallazgo-relacionado-resuelto-sincronización-del-pipeline-al-contacto) para el detalle técnico completo.

---

## Bitácora de cambios

| Fecha | Versión | Cambio | Autor |
|---|---|---|---|
| 2026-06-21 | 1.0 | Documento creado, extrayendo y generalizando la distinción de tipos de etiquetas que originalmente vivía solo dentro de `desactivar-bot.md`. | Enuar |
