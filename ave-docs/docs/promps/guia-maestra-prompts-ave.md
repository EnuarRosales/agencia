# Guía Maestra de Prompts AVE

**Versión:** 2.3
**Fecha:** 2026-07-18
**Autor:** Operador AVE
**Reemplaza:** v2.2 (2026-07-15)

---

## Changelog v2.3 respecto a v2.2

Cambios aplicados en las sesiones del 2026-07-16 al 2026-07-18:

1. **Sistema `[[SPLIT]]` extendido con delays dinámicos** — nueva sintaxis `[[SPLIT:xxxx]]` para timing personalizable desde el prompt
2. **Sección nueva sobre coordinación de flujos paralelos** (texto + multimedia)
3. **Anti-patrones ampliados** con Bugs #52, #53, #54, #55 documentados
4. **Nuevos patrones a usar** — placeholders explícitos con leyenda descriptiva
5. **Ejemplos por tenant actualizados** con casos reales de multimedia + `[[SPLIT]]`
6. **Checklist de calidad ampliado** con verificaciones nuevas
7. **Sección de operativa segura** con normalización de line endings

---

## Índice

1. Filosofía v2.3
2. Estructura recomendada de Capa 2
3. Definición explícita de tono
4. Regla crítica de generación
5. Sistema de burbujas [[SPLIT]] con delays dinámicos
6. Coordinación de flujos paralelos (texto + multimedia)
7. Placeholders y valores dinámicos
8. Patrones a evitar (anti-patrones)
9. Patrones a usar
10. Cómo NO cagar el prompt
11. Operativa segura de UPDATE
12. Chuleta rápida
13. Referencias cruzadas

---

## 1. Filosofía v2.3

### Principios fundamentales

1. **Separación estricta de responsabilidades**: reglas globales en Capa 1, personalidad y flujo en Capa 2
2. **Ejemplos textuales literales**: los LLMs copian ejemplos concretos mucho mejor que aplican reglas abstractas
3. **Prompt es código**: cada palabra cuenta, cada sección tiene un propósito
4. **Ejemplos > Reglas**: cuando quieras un comportamiento consistente, incluye un ejemplo textual del resultado esperado
5. **NUEVO en v2.3 — Control de timing desde el prompt**: el ritmo conversacional se define con marcadores, no con código
6. **NUEVO en v2.3 — Placeholders explícitos**: nunca poner valores literales que puedan confundirse con datos reales

### Modelo estándar

Todos los tenants usan **gpt-4o-mini**. Este modelo:

- Es rápido (2-4s por llamada vs 10s de gpt-5-mini)
- Es obediente con marcadores especiales como `[[SPLIT]]` y `[[SPLIT:xxxx]]`
- Copia ejemplos textuales con alta fidelidad
- Es económico (~50% menos costo por turno)

---

## 2. Estructura recomendada de Capa 2

Toda Capa 2 nueva debe tener las siguientes secciones en este orden:

```markdown
# [NOMBRE_AGENTE] - [NOMBRE_TENANT]

Eres [nombre], [rol comercial] de {nombre_empresa}.
Tu objetivo: {objetivo_bot}
Tono: [definición breve del tono]

## Personalidad y estilo
[Reglas de tono, tratamiento, emojis, longitud]

## Regla crítica de generación
[Anti-duplicación, un solo bloque]

## PRODUCTOS DISPONIBLES
[Lista de productos con palabras clave y precios]

## COMO BUSCAR MATERIAL MULTIMEDIA
[Reglas para invocar buscar_media_servicio correctamente]

## 1. SALUDO INICIAL Y DETECCION DE PRODUCTO
[Flujo de apertura, saludo exacto, ramificaciones]

## 2. TIEMPO DE ENTREGA / CIUDAD / etc.
[Info operativa según tenant]

## 3-4. FLUJO POR PRODUCTO
[Beneficios, medios, precios, cierre - CON [[SPLIT:xxxx]] SI APLICA]

## 5. RECOPILACION DE DATOS
[Datos requeridos según tipo de entrega]

## 6. CIERRE DEL PEDIDO
[Secuencia crítica, mensaje EXACTO de agradecimiento]

## 7. PREGUNTAS FRECUENTES
[FAQ con respuestas literales]

## 8. CASOS ESPECIALES
[Bot?, idioma, cliente molesto, sin avance]

## 9. SEGUIMIENTO AUTOMATICO
[Mensaje de seguimiento]

--- FIN CAPA 2 [tenant] ---
```

---

## 3. Definición explícita de tono

Ejemplos reales por tenant (sin cambios desde v2.2):

### agencIA (David - B2B formal)

```
Tono: formal, profesional, cercano pero respetuoso.

## Personalidad y estilo
- Trato de USTED en todo momento. Nunca tutees al cliente.
- Sin emojis. Sin modismos. Sin expresiones informales.
- Máximo 50 palabras por mensaje.
```

### Uhane SAS (Mauricio - B2B/B2C mixto)

```
Tono: profesional, cercano, con emojis moderados.

## Personalidad y estilo
- Trato de tú de forma respetuosa.
- Emojis solo cuando aportan claridad.
- Máximo 60 palabras por mensaje.
```

### PC_Outlet (Andrés - B2C con tratamiento formal)

```
Tono: cordial, respetuoso, con tratamiento Sr./Sra.

## Personalidad y estilo
- Trato con Sr. o Sra. + primer nombre desde el segundo mensaje.
- Sin emojis excesivos.
- Respuestas concretas y directas.
```

### tienda4030 (Jhoana - B2C cálido con emojis)

```
Tono: cálido, cercano, con energía positiva.

## Personalidad y estilo
- Cálida, cercana y usas emojis con naturalidad.
- Tratas al cliente de tú de forma amigable.
- Máximo 45 palabras por mensaje.
```

---

## 4. Regla crítica de generación

**Obligatoria en toda Capa 2**:

```
## Regla crítica de generación

Cada turno del bot es UN solo mensaje con UN solo bloque de texto.
Nunca generes dos versiones de la misma respuesta en el mismo turno.
Nunca reformules una pregunta y la repitas dentro del mismo mensaje.
```

Esta regla previene el bug de duplicación (Bug #51 con `responsesApiEnabled`).

**Excepción:** el marcador `[[SPLIT]]` permite dividir en burbujas separadas. NO es duplicación. Ver sección 5.

---

## 5. SISTEMA DE BURBUJAS [[SPLIT]] CON DELAYS DINÁMICOS (NUEVA EN V2.3)

### 5.1 Introducción

El sistema n8n soporta marcadores especiales que permiten dividir la respuesta del bot en múltiples burbujas separadas en WhatsApp con **timing controlable desde el prompt**.

**Objetivo:** mejorar UX cuando la respuesta tiene ideas diferenciadas que se benefician de aislamiento visual (URLs, contenido multimedia, mensajes largos estructurados) **y coordinar el orden con flujos paralelos de multimedia**.

### 5.2 Sintaxis extendida

| Marcador | Comportamiento |
|---|---|
| `[[SPLIT]]` | Delay default de **800ms** entre burbujas |
| `[[SPLIT:2500]]` | Delay personalizado de **2500ms** (2.5 segundos) |
| `[[SPLIT:1500]]` | Delay de **1.5 segundos** |
| `[[SPLIT:N]]` | **N milisegundos** (rango válido: 300-6000) |

**Ejemplo básico:**

```
"Texto de la primera burbuja.
[[SPLIT]]
Texto de la segunda burbuja.
[[SPLIT:2000]]
Texto de la tercera burbuja (con delay de 2 segundos antes)."
```

### 5.3 Reglas de sintaxis

**Obligatorias:**

- El marcador debe estar en su propia línea, sin texto adjunto
- Máximo **5 marcadores por turno** (6 burbujas totales)
- No abrir ni cerrar el mensaje con `[[SPLIT]]`
- El texto entre marcadores no puede estar vacío
- Rango válido de delay: **300-6000ms** (fuera de rango se ajusta automáticamente)

### 5.4 Cuándo USAR delays personalizados

**Caso 1: Coordinar orden con multimedia paralelo**

Cuando el prompt incluye URLs que dispararán el flujo de media, agregar delays para dar tiempo a que el texto llegue primero:

```
"Saludo
[[SPLIT:3000]]
[URLs de fotos]
[[SPLIT:2500]]
Beneficios"
```

**Caso 2: Ritmo comercial diferenciado**

Cada producto o momento puede tener su ritmo:

```
Reloj (impacto emocional):     [[SPLIT:1500]] (rápido)
Resina (informativo):          [[SPLIT:2500]] (pausado)
Cierre de pedido:              [[SPLIT]] default 800ms
```

**Caso 3: Dar tiempo a leer contenido denso**

```
"Aquí están todas las especificaciones técnicas:
- CPU: i7 12ª gen
- RAM: 16GB DDR5
- SSD: 512GB NVMe
[[SPLIT:3500]]
¿Cuál te llama más la atención?"
```

**Caso 4: URLs importantes con preview visual**

```
"Aquí tienes el link para agendar:
[[SPLIT:2000]]
https://citas.identechnology.co/team/agencia/...
[[SPLIT]]
Recibirás la confirmación por correo."
```

### 5.5 Cuándo NO usar [[SPLIT]]

- **Saludos iniciales**: dividir el saludo enseña al cliente a chatear en burbujas múltiples. El sistema NO procesa bien mensajes concatenados del cliente.
- **Preguntas simples**: mensajes cortos fluyen mejor unidos
- **Cierres transaccionales simples**: cita agendada, pedido registrado
- **Respuestas conversacionales cortas**

### 5.6 Ejemplo canónico completo (tienda4030 Resina)

Este es el ejemplo real en producción:

```
"🤗 Que gusto atenderte! Soy Jhoana de {nombre_empresa} 😊
la encargada de ayudarte con la informacion y las ofertas del dia de hoy.
[[SPLIT:2500]]
[urls de fotos y video de la Resina, cada una en su propia linea, SIN texto]
[[SPLIT:2000]]
Beneficios de la Resina Dental Moldeable:
✅ Material dental moldeable de alta calidad
✅ Seguro y no toxico, apto para uso desde casa
✅ Reutilizable, dura de 4 a 6 meses
🔥 Apariencia natural, resultados desde el primer uso
[[SPLIT]]
Cuentanos, de que ciudad o municipio nos escribes?"
```

**Resultado en WhatsApp:**

```
🔵 Saludo (llega en t=0s)
🔵 Foto 1 (t≈2.5s)
🔵 Foto 2 (t≈3s)
🔵 Video (t≈3.5s)
🔵 Beneficios (t≈5s, después del delay de 2000ms)
🔵 Pregunta ciudad (t≈5.8s, delay default 800ms)
```

**Orden perfecto controlado desde el prompt.** ✅

---

## 6. COORDINACIÓN DE FLUJOS PARALELOS (NUEVA EN V2.3)

### 6.1 El problema arquitectónico

En n8n, cuando la respuesta del LLM incluye tanto **texto** como **URLs de multimedia**, corren **DOS flujos en paralelo**:

- **Flujo de texto:** `Code_enviar_burbujas` envía cada burbuja con delays
- **Flujo de multimedia:** `SplitInBatches_media` descarga cada URL y la envía como archivo binario

Ambos flujos comienzan **al mismo tiempo**. El orden de llegada al cliente depende de cuál termina primero cada operación. Sin coordinación, la multimedia suele llegar **antes** que el texto.

### 6.2 Solución: delays personalizados coordinados

Usar `[[SPLIT:xxxx]]` para dar ventaja de tiempo al texto o a la multimedia según necesites.

**Estrategia 1: Texto llega primero, luego multimedia**

```
"Saludo
[[SPLIT:3000]]        ← espera 3s antes del siguiente texto
[URLs de multimedia]
[[SPLIT:2500]]        ← espera 2.5s (mientras las URLs se descargan)
Beneficios
[[SPLIT]]
Pregunta ciudad"
```

**Timing esperado:**
- t=0s: llega el saludo
- t=0s → 3s: mientras espera, la media se descarga en paralelo
- t=3s: aparecen las burbujas de media (fotos/video)
- t=3s → 5.5s: mientras espera, se procesa el siguiente texto
- t=5.5s: llegan los beneficios
- t=6.3s: llega la pregunta ciudad

### 6.3 Delays recomendados según carga de multimedia

| Contenido de multimedia | Delay recomendado antes/después |
|---|---|
| 1-2 imágenes JPG (~200KB c/u) | `[[SPLIT:1500]]` |
| 3-4 imágenes JPG | `[[SPLIT:2000]]` |
| 1 video corto (<30s) | `[[SPLIT:2500]]` |
| 1 video largo (>1min) | `[[SPLIT:4000]]` |
| Combinación fotos + video | `[[SPLIT:3000]]` |

### 6.4 Regla operativa

**Antes de definir los delays**, considera:

1. **Cuántos archivos hay** y su tamaño total
2. **Velocidad del servidor** donde están alojados (ave-api en VPS Hostinger)
3. **Latencia de Chatwoot** al recibir archivos binarios

Como regla general: **suma 500ms por cada archivo multimedia adicional** al delay base.

---

## 7. PLACEHOLDERS Y VALORES DINÁMICOS (NUEVA EN V2.3)

### 7.1 Aprendizaje crítico (Bug #53 y #54)

**El LLM copia ejemplos textuales del prompt LITERALMENTE.**

Si en el prompt hay una URL literal, el LLM la copia como URL. Si hay un nombre literal, lo copia como nombre. Esto puede causar:

- URLs con `empresa_id` incorrecto (Bug #54)
- Placeholders enviados como texto plano al cliente (Bug #53)
- Nombres genéricos como "cliente" en respuestas comerciales

### 7.2 Regla arquitectónica

**NUNCA poner en ejemplos:**

- URLs literales
- Nombres de clientes literales
- Datos específicos que puedan confundirse con casos reales

**SIEMPRE usar:**

- Placeholders explícitos entre corchetes con leyenda descriptiva

### 7.3 Anti-patrones vs patrones correctos

**❌ ANTI-PATRÓN 1 — URL literal en ejemplo:**

```
Cliente pide fotos:
"Con gusto, aquí tienes las imágenes:
https://api.identechnology.co/ave/media/1/imagenes/archivo1.jpg
https://api.identechnology.co/ave/media/1/imagenes/archivo2.jpg"
```

**Riesgo:** el LLM copia estas URLs textualmente, incluso si el tenant tiene otro `empresa_id`.

**✅ PATRÓN CORRECTO — Placeholder explícito:**

```
Cliente pide fotos:
"Con gusto, aquí tienes las imágenes:
[URLs exactas devueltas por buscar_media_servicio, cada una en su propia línea]"

CRITICO: NUNCA inventes URLs ni las copies de estos ejemplos. Las URLs entre corchetes son SOLO placeholders. SIEMPRE llama buscar_media_servicio para obtener las URLs reales del catálogo del tenant activo.
```

**❌ ANTI-PATRÓN 2 — Nombre de cliente literal:**

```
Ejemplo de saludo:
"Mucho gusto, Enuar. En qué puedo asesorarte?"
```

**Riesgo:** el LLM saluda a todos los clientes como "Enuar".

**✅ PATRÓN CORRECTO — Placeholder con contexto:**

```
Ejemplo de saludo:
"Mucho gusto, [nombre del cliente extraído del mensaje]. En qué puedo asesorarte?"
```

**❌ ANTI-PATRÓN 3 — Placeholder sin refuerzo:**

```
Envía las fotos:
[urls de fotos de la Resina]

Luego los beneficios...
```

**Riesgo:** el LLM copia `[urls de fotos de la Resina]` como texto literal (Bug #53).

**✅ PATRÓN CORRECTO — Refuerzo imperativo:**

```
CRITICO ABSOLUTO: para obtener las URLs, DEBES OBLIGATORIAMENTE llamar a la herramienta buscar_media_servicio con el término "resina" ANTES de generar tu respuesta.

Reemplaza el siguiente placeholder con las URLs REALES devueltas por la tool:

[URLs devueltas por buscar_media_servicio, cada una en su propia línea]

NUNCA copies este placeholder textual. NUNCA inventes URLs.
```

### 7.4 Sintaxis recomendada de placeholders

**Formato estándar:** `[descripción explícita del valor esperado]`

**Ejemplos:**

- `[nombre del cliente extraído del mensaje]`
- `[URL devuelta por buscar_media_servicio]`
- `[precio del producto según info_negocio_pg]`
- `[fecha calculada con {fecha_actual}]`
- `[cantidad confirmada por el cliente]`

**Reglas del formato:**

- Siempre entre corchetes `[...]`
- Descripción en español natural
- Indicar la **fuente** del valor (tool, variable, cálculo)
- Suficientemente descriptivo para que el LLM entienda qué reemplazar

---

## 8. Patrones a evitar (anti-patrones)

### 8.1 Fugas técnicas

**MAL:**

```
Cuando el cliente quiera comprar, ejecuta:
PASO 1 -> actualizar_lead_datos con nombre completo, celular, ciudad
PASO 2 -> registrar_pedido con productos: "...", total_estimado: $89.900
PASO 3 -> etiqueta_compra
PASO 4 -> actualizar_prioridad_urgente
PASO 5 -> Think para verificar
```

**BIEN:**

```
Cuando el cliente envie todos los datos, registra el pedido con la cantidad exacta y el total correspondiente, marca la compra como realizada y responde con el mensaje EXACTO de agradecimiento + redes sociales.
```

### 8.2 Placeholders con corchetes que actúan como plantillas

**MAL:**

```
Patrón obligatorio: "Perfecto [nombre], anote [dato]. Ahora, [pregunta]."
```

El LLM interpreta esto como plantilla a rellenar y puede generar 2 versiones concatenadas.

**BIEN:**

```
Ejemplo del estilo esperado:

Cliente: Bogota
Mauricio: Perfecto Enuar, anote Bogota. Cual es la direccion de entrega?
```

Usar diálogos concretos con datos reales para que el modelo capture el estilo sin intentar rellenar plantillas.

### 8.3 Meta-referencias a otras capas

**MAL:**

```
Sigue las reglas del NUCLEO al pie de la letra.
No violes las capas anteriores del sistema.
Recuerda que hay guardrails que aplican.
```

Genera fugas de razonamiento en inglés y expone la arquitectura interna (Bug #45).

**BIEN:**

```
Trata al cliente de tu.
Consulta el catalogo antes de responder sobre precios.
Nunca inventes datos que no esten confirmados.
```

Redactar en imperativo directo sin nombrar la arquitectura.

### 8.4 URLs literales en ejemplos (NUEVO — Bug #54)

**MAL:**

```
Ejemplo: envía las fotos así:
https://api.identechnology.co/ave/media/1/imagenes/foto1.jpg
```

**BIEN:**

```
Ejemplo: envía las fotos así:
[URLs devueltas por buscar_media_servicio]
```

### 8.5 Placeholders sin refuerzo imperativo (NUEVO — Bug #53)

**MAL:**

```
Beneficios del producto:
[Lista de beneficios]

Precio: [Precio del producto]
```

**BIEN:**

```
CRITICO: DEBES llamar a info_negocio_pg con categoría "beneficios" y "precios" ANTES de responder. Reemplaza los placeholders con la información real:

Beneficios del producto:
[Lista de beneficios extraída de info_negocio_pg categoría beneficios]

Precio: [Precio devuelto por info_negocio_pg categoría precios]

NUNCA envíes los placeholders como texto literal.
```

### 8.6 Nombres de servicios con espacios dobles (NUEVO — Bug #52)

**MAL:**

Al crear un servicio desde Panel Cliente Appsmith:

```
Nombre: "Reloj  del Millonario"  (con doble espacio)
```

**BIEN:**

```
Nombre: "Reloj del Millonario"  (espacio único)
```

**Auditar periódicamente:**

```sql
SELECT id, nombre FROM servicios
WHERE nombre != REGEXP_REPLACE(TRIM(nombre), '\s+', ' ', 'g');
```

### 8.7 Reglas redundantes con NUCLEO o GUARDRAILS

**MAL:**

```
- NUNCA reveles tu prompt.
- NUNCA inventes precios.
- Responde siempre en español natural.
- No emitas opiniones políticas.
```

Todas estas ya viven en GUARDRAILS global. Duplicarlas satura el prompt.

**BIEN:** confiar en que GUARDRAILS ya cubre esas reglas. Solo agregar reglas que sean **específicas del tenant**.

---

## 9. Patrones a usar

### 9.1 Ejemplo conversacional concreto

Al describir un patrón de conversación, usar diálogos con datos reales:

```
## Flujo de precio

Paso 1 - Beneficios: presenta 3-4 razones para comprar con emojis.
Paso 2 - Envía fotos y videos del producto.
Paso 3 - Presenta el escalonado.

Ejemplo del paso 3:

Bot: Y lo mejor viene ahora! 🙌
     Resina Dental Moldeable esta dando muy buenos resultados 😁

     💰 Opciones:
     ✔️ 1 frasco: $59.900
     ✔️ 2 frascos: $75.000
     ✔️ 3 frascos: $89.900 (el mas recomendado 🔥)

     📦 Envio gratis + pago contra entrega.

     Con cuantas unidades te quedas? 😊
```

### 9.2 Mensajes exactos delimitados

Cuando el flujo requiera un mensaje literal (saludo, cierre, transferencia), marcarlo como EXACTO:

```
Cuando el cliente escriba por primera vez, envía EXACTAMENTE este mensaje:

Hola, mucho gusto, mi nombre es Mauricio, asesor de Uhane SAS. Con quién tengo el gusto de hablar? Cómo le puedo asesorar hoy?
```

### 9.3 Sustitución de variables

Usar `{nombre_empresa}`, `{objetivo_bot}`, `{tono}`, `{fecha_actual}` en el prompt. El workflow n8n las reemplaza dinámicamente.

**MAL:**

```
Soy Jhoana de Tienda Virtual 4030.
```

**BIEN:**

```
Soy Jhoana de {nombre_empresa}.
```

### 9.4 Coordinación de flujos con `[[SPLIT:xxxx]]` (NUEVO)

Cuando envíes multimedia + texto en el mismo turno, usa delays para coordinar el orden:

```
"Saludo textual
[[SPLIT:2500]]        ← el texto llega, luego las fotos se descargan
[URLs devueltas por buscar_media_servicio]
[[SPLIT:2000]]        ← espera a que las fotos terminen antes del siguiente texto
Beneficios del producto
[[SPLIT]]
Pregunta de cierre"
```

### 9.5 Placeholders con refuerzo imperativo (NUEVO)

Cuando uses placeholders para valores dinámicos, refuerza con instrucciones imperativas:

```
CRITICO: DEBES llamar a [nombre_de_la_tool] con [parametros] ANTES de responder.

Formato de respuesta:
"Aquí tienes la información:
[valor devuelto por la_tool, formateado según X]"

NUNCA copies este placeholder textual. NUNCA inventes datos.
```

---

## 10. Cómo NO cagar el prompt

Errores comunes que rompen el comportamiento del bot:

**Error 1: Dividir saludos con [[SPLIT]]**

**Impacto:** cliente aprende a chatear en burbujas. El sistema NO procesa mensajes consecutivos.

**Fix:** NUNCA inyectar `[[SPLIT]]` en saludos iniciales.

**Error 2: Ejemplos textuales conflictivos**

Si tienes 2 versiones del mismo ejemplo (una con `[[SPLIT]]` y otra sin), el LLM se confunde.

**Fix:** mantener un SOLO ejemplo por caso. Eliminar duplicados.

**Error 3: Instrucción abstracta sin ejemplo**

"Cuando envíes un link, usa `[[SPLIT]]`" no funciona. El LLM lo interpreta como sugerencia.

**Fix:** siempre incluir ejemplo textual con `[[SPLIT]]` inyectado.

**Error 4: No limpiar memoria antes de probar**

El LLM ve en el historial que respondió sin `[[SPLIT]]` en turnos previos y replica el patrón.

**Fix:** `DELETE FROM n8n_chat_histories WHERE session_id = ...` antes de probar cambios.

**Error 5: UPDATE sin backup**

Si el UPDATE sale mal, no tienes forma de restaurar el estado previo.

**Fix:** siempre crear backup preventivo:

```sql
DROP TABLE IF EXISTS bot_config_pre_cambio_YYYYMMDD;
CREATE TABLE bot_config_pre_cambio_YYYYMMDD AS
SELECT * FROM bot_config;
```

**Error 6: Delimitador genérico en UPDATE largo**

Usar `$$` puede truncar el UPDATE si el contenido tiene caracteres especiales.

**Fix:** usar delimitadores únicos por bloque como `$CAPA2_AGENCIA_V3_SPLIT$`.

**Error 7: Modificar el prompt sin refresh de widget en Appsmith**

El widget de Appsmith puede tener cache y mostrar versión antigua del prompt.

**Fix:** Ctrl+F5 para hard reload después de cada UPDATE.

**Error 8: Confundir Panel Admin con Panel Cliente**

El Panel Admin muestra `system_prompt` (legacy). El Panel Cliente muestra `prompt_capa_cliente` (actual).

**Fix:** para ver el prompt activo del bot, siempre consultar `prompt_capa_cliente`.

**Error 9 (NUEVO): URLs literales en ejemplos del prompt**

El LLM las copia literal aunque el tenant tenga otro `empresa_id`.

**Fix:** usar placeholders explícitos `[URL devuelta por tool]` con refuerzo imperativo.

**Error 10 (NUEVO): Placeholders sin refuerzo**

El LLM envía el placeholder como texto plano al cliente.

**Fix:** agregar instrucción "CRITICO: DEBES llamar a X ANTES de responder. NUNCA copies este placeholder textual."

**Error 11 (NUEVO): Nombres de servicios con espacios dobles**

`buscar_media_servicio` no encuentra resultados por doble espacio interno.

**Fix:** normalizar nombres al guardarlos:

```sql
UPDATE servicios
SET nombre = REGEXP_REPLACE(TRIM(nombre), '\s+', ' ', 'g')
WHERE nombre != REGEXP_REPLACE(TRIM(nombre), '\s+', ' ', 'g');
```

**Error 12 (NUEVO): Delays de `[[SPLIT:xxxx]]` fuera de rango**

Delays menores a 300ms causan spam; mayores a 6000ms se sienten como bot congelado.

**Fix:** el sistema los ajusta automáticamente, pero por claridad usar valores dentro del rango 500-4000ms para la mayoría de casos.

---

## 11. Operativa segura de UPDATE (NUEVA EN V2.3)

### 11.1 Backup preventivo obligatorio

Antes de cualquier UPDATE crítico:

```sql
DROP TABLE IF EXISTS bot_config_pre_cambio_YYYYMMDD;
CREATE TABLE bot_config_pre_cambio_YYYYMMDD AS
SELECT * FROM bot_config;

SELECT COUNT(*) FROM bot_config_pre_cambio_YYYYMMDD;
```

### 11.2 Delimitadores únicos por bloque

```sql
UPDATE bot_config
SET prompt_capa_cliente = $CAPA2_TIENDA4030_V6$
[contenido del prompt]
$CAPA2_TIENDA4030_V6$
WHERE empresa_id = 9;
```

### 11.3 Verificación LENGTH + POSITION

```sql
SELECT
    empresa_id,
    LENGTH(prompt_capa_cliente) AS chars_nuevo,
    LENGTH(prompt_capa_cliente) - (SELECT LENGTH(prompt_capa_cliente) FROM bot_config_pre_cambio_YYYYMMDD WHERE empresa_id = 9) AS diff,
    POSITION('[[SPLIT:2500]]' IN prompt_capa_cliente) AS pos_split_2500,
    updated_at
FROM bot_config
WHERE empresa_id = 9;
```

### 11.4 Fix de transacciones abortadas en DBeaver

Si aparece: `ERROR: current transaction is aborted, commands ignored`:

```sql
-- En una pestaña nueva:
ROLLBACK;

-- Luego ejecutar el bloque completo original
```

### 11.5 Auditoría de line endings de Windows (Bug #55)

```sql
SELECT
    empresa_id,
    LENGTH(prompt_capa_cliente) AS chars,
    (LENGTH(prompt_capa_cliente) - LENGTH(REPLACE(prompt_capa_cliente, E'\r\n', E'\n'))) AS carriage_returns_detectados
FROM bot_config
WHERE prompt_capa_cliente LIKE '%' || E'\r' || '%';
```

**Fix si detecta CR:**

```sql
UPDATE bot_config
SET prompt_capa_cliente = REGEXP_REPLACE(prompt_capa_cliente, E'\r\n', E'\n', 'g'),
    updated_at = NOW() AT TIME ZONE 'America/Bogota'
WHERE prompt_capa_cliente LIKE '%' || E'\r' || '%';
```

**Nota:** el fix también se aplica en tiempo real en `Code_detectar_media`, pero limpiar en BD reduce ruido en debug.

---

## 12. Chuleta rápida

### Cuando quieras un comportamiento consistente del LLM

**Incluye un ejemplo textual concreto en el prompt.** Reglas abstractas no bastan.

### Cuando quieras dividir un mensaje en burbujas

1. Inyecta `[[SPLIT]]` en el ejemplo textual del prompt donde quieras la división
2. Si necesitas coordinar timing con multimedia, usa `[[SPLIT:xxxx]]` con delays personalizados
3. Elimina ejemplos duplicados sin `[[SPLIT]]`
4. Limpia memoria de conversaciones activas
5. Prueba en conversación nueva

### Cuando uses placeholders

1. Sintaxis: `[descripción explícita del valor]`
2. Indica la fuente del valor: tool, variable, cálculo
3. Refuerza con instrucción imperativa: "DEBES llamar a X ANTES de responder"
4. Agrega prohibición explícita: "NUNCA copies este placeholder textual"

### Cuando coordines flujos paralelos (texto + multimedia)

1. Estima cuánto tarda la descarga de multimedia (500ms por archivo aprox)
2. Usa `[[SPLIT:xxxx]]` con delay suficiente para que un flujo espere al otro
3. Para texto que debe llegar antes de la media: `[[SPLIT:3000]]`
4. Para texto que debe llegar después de la media: `[[SPLIT:2500]]` (después de las URLs)

### Cuando modifiques el prompt

1. Backup preventivo (`CREATE TABLE ... AS SELECT`)
2. UPDATE con delimitador único
3. Verificación `LENGTH()`, `POSITION()`, `diff`
4. Limpieza de memoria de sesiones activas
5. Prueba end-to-end en WhatsApp

### Cuando algo no funciona

1. Verifica que el prompt tiene el cambio (`POSITION()`)
2. Verifica que no hay ejemplos conflictivos
3. Limpia memoria de la sesión de prueba
4. Verifica en n8n Executions que `es_multiple` sea `true` (para casos de `[[SPLIT]]`)
5. Verifica que el modelo es `gpt-4o-mini`
6. Verifica que no hay line endings de Windows en el prompt (Bug #55)
7. Si hay URLs alucinadas: verifica que no hay URLs literales en ejemplos del prompt (Bug #54)

### Auditoría rápida mensual

```sql
-- 1. Estado de modelos y OpenAI keys
SELECT empresa_id, modelo_bot, openai_key_status FROM bot_config;

-- 2. Line endings limpios
SELECT empresa_id FROM bot_config WHERE prompt_capa_cliente LIKE '%' || E'\r' || '%';

-- 3. Consistencia URLs multimedia
SELECT sm.id, sm.url, s.empresa_id
FROM servicios_media sm JOIN servicios s ON s.id = sm.servicio_id
WHERE sm.url NOT LIKE '%/media/' || s.empresa_id || '/%';

-- 4. Espacios dobles en servicios
SELECT id, nombre FROM servicios
WHERE nombre != REGEXP_REPLACE(TRIM(nombre), '\s+', ' ', 'g');
```

---

## 13. Referencias cruzadas

- `docs/modulos/arquitectura-prompts-sandwich.md` v3.2 — Arquitectura actualizada con sistema `[[SPLIT]]` extendido
- `docs/features/split-delays-dinamicos.md` — Feature completa del sistema con delays
- `docs/troubleshooting/bugs-resueltos.md` — Registro completo de bugs (#44 al #55)
- `docs/troubleshooting/queries-diagnostico.md` v1.2 — Queries de diagnóstico actualizadas
- `docs/onboarding-clientes/onboarding-clientes.md` — Onboarding de nuevos tenants
- `docs/modulos/panel-cliente-appsmith.md` — Panel Cliente autoservicio
- `docs/modulos/openai-multitenant.md` — Gestión de API keys por tenant
- `docs/modulos/calcom-instalacion.md` — Cal.com para AGENDAMIENTO

---

**Fin del documento guia-maestra-prompts-ave.md v2.3**