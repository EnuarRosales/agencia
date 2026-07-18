# Feature — Sistema [[SPLIT]] con delays dinámicos

**Versión:** 1.0
**Fecha:** 2026-07-18
**Autor:** Operador AVE
**Estado:** Implementado, validado y en producción
**Tenant piloto:** tienda4030 (empresa_id=9)

---

## 1. Contexto y motivación

En sesiones previas se implementó el sistema básico `[[SPLIT]]` que permite dividir la respuesta del bot en múltiples burbujas separadas en WhatsApp con delay fijo de 800ms entre ellas.

Sin embargo, cuando el flujo comercial combina **texto + multimedia** (fotos, videos), los dos flujos corren en paralelo en n8n y **la media suele llegar antes que el texto**. Esto causa desorden visual en WhatsApp, deteriorando la UX.

La solución arquitectónicamente correcta es permitir **delays personalizados por marcador**, controlables desde el prompt sin necesidad de modificar código.

## 2. Sintaxis del sistema

### 2.1 Marcadores soportados

| Marcador | Comportamiento |
|---|---|
| `[[SPLIT]]` | Delay default de 800ms entre burbujas |
| `[[SPLIT:2500]]` | Delay personalizado de 2500ms (2.5 segundos) |
| `[[SPLIT:1500]]` | Delay de 1.5 segundos |
| `[[SPLIT:N]]` | Delay de N milisegundos (rango válido: 300-6000) |

### 2.2 Límites de seguridad

- **Delay mínimo:** 300ms (evitar rate limit de WhatsApp)
- **Delay máximo:** 6000ms (evitar percepción de "bot congelado")
- **Segmentos máximos:** 6 por turno (7 burbujas totales)
- Valores fuera de rango se ajustan automáticamente

### 2.3 Reglas de sintaxis

- El marcador debe estar en su propia línea
- No abrir ni cerrar el mensaje con `[[SPLIT]]`
- El texto entre marcadores no puede estar vacío
- Retrocompatible: los prompts con `[[SPLIT]]` simple siguen funcionando

## 3. Arquitectura implementada

### 3.1 Componentes en n8n

**Nodo `Code_detectar_media`** (refactorizado):

- Normaliza line endings de Windows (`\r\n` → `\n`) para robustez
- Detecta ambos formatos de marcador con regex
- Extrae delays con función `parseDelay()`
- Aplica límite defensivo de 6 segmentos
- Retorna cada segmento con metadata `delay_antes_ms`

**Nodo `Code_enviar_burbujas`** (refactorizado):

- Lee `delay_antes_ms` de cada segmento
- Aplica el delay ANTES de enviar la burbuja al Chatwoot
- Retrocompatible: si no hay `delay_antes_ms`, usa 800ms default

**Flujo paralelo de media** (sin cambios):

- `If_tiene_media` → `Code_split_archivos` → `SplitInBatches_media` → `Descargar_media` → `Enviar_media_Chatwoot`
- Este flujo maneja las URLs como archivos binarios nativos

### 3.2 Diagrama del flujo

```
AI_Agent_Principal (output con [[SPLIT:xxxx]])
        ↓
Code_detectar_media (normaliza, parsea, segmenta)
        ├──→ If_es_multiple (TRUE) → Code_enviar_burbujas (respeta delays)
        │                                    ↓
        │                              Envía burbujas de texto secuencialmente
        │
        └──→ If_tiene_media (TRUE en paralelo) → SplitInBatches_media
                                                        ↓
                                                Enviar_media_Chatwoot (binario nativo)
```

## 4. Reglas de uso comercial

### 4.1 Cuándo USAR delays personalizados

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
Reloj (impacto emocional):    [[SPLIT:1500]] (rápido)
Resina (informativo):         [[SPLIT:2500]] (pausado)
Cierre de pedido:             [[SPLIT]] default
```

**Caso 3: Dar tiempo a leer contenido denso**

Después de un mensaje largo, dar tiempo para procesarlo:

```
"Aquí están todas las especificaciones técnicas:
- CPU: i7 12ª gen
- RAM: 16GB DDR5
- SSD: 512GB NVMe
[[SPLIT:3500]]
¿Cuál te llama más la atención?"
```

### 4.2 Cuándo NO usar

**NO usar en:**

- Saludos iniciales (enseña al cliente a chatear en burbujas)
- Preguntas simples de calificación
- Cierres transaccionales (cita confirmada, pedido registrado)
- Respuestas cortas conversacionales

**Delays a evitar:**

- Menor a 300ms: se percibe como bombardeo
- Mayor a 5000ms: cliente cree que el bot se colgó
- Delay uniforme muy largo: pierde ritmo natural

## 5. Casos de uso reales por tenant

### 5.1 tienda4030 — Resina Dental (implementado)

**Escenario:** cliente escribe "Hola quiero la resina"

```
"🤗 Que gusto atenderte! Soy Jhoana de tienda4030 😊
la encargada de ayudarte con la informacion.
[[SPLIT:2500]]
[URLs de fotos y video de la Resina]
[[SPLIT:2000]]
Beneficios de la Resina Dental Moldeable:
✅ Material dental moldeable de alta calidad
✅ Seguro y no toxico
✅ Reutilizable, dura 4-6 meses
🔥 Apariencia natural
[[SPLIT]]
Cuentanos, de que ciudad o municipio nos escribes?"
```

**Timing:**
- t=0s: Saludo llega primero
- t=2.5s: Empieza a llegar la media (2 fotos + video)
- t=5s: Beneficios llegan después de la media
- t=5.8s: Pregunta ciudad al final

**Resultado en WhatsApp:**
```
🔵 Saludo
🔵 Foto 1
🔵 Foto 2
🔵 Video
🔵 Beneficios
🔵 Pregunta ciudad
```

### 5.2 agencIA — Envío de link Cal.com (candidato futuro)

```
"Claro. Aqui tiene el link para agendar:
[[SPLIT:2000]]
https://citas.identechnology.co/team/agencia/consulta-estrategica-ave
[[SPLIT:1500]]
Recibira la confirmacion por correo con el enlace de Google Meet."
```

**Beneficio:** el link queda aislado con preview visual limpio, mejorando la conversión al agendamiento.

### 5.3 Uhane — Datos de pago (candidato futuro)

```
"Perfecto Enuar, aqui tienes los datos:
[[SPLIT:1800]]
Bancolombia Ahorros
Cuenta: 63400002911
Titular: Uhane SAS
NIT: 901.819.375-1
[[SPLIT:1500]]
Cuando confirmes el pago, avisame y coordino el despacho."
```

## 6. Bugs resueltos durante la implementación

### 6.1 Bug #52 — Doble espacio en nombres de servicios

**Detectado:** 2026-07-16

**Síntoma:** El LLM llamaba `buscar_media_servicio` con el nombre real del producto pero la búsqueda devolvía 0 resultados.

**Causa raíz:** En `servicios.nombre` había un doble espacio interno: `"Reloj  del Millonario"`. El `ILIKE` era literal y no matcheaba con el término limpio `"reloj del millonario"`.

**Solución:** UPDATE del nombre en BD con espacio único + normalización defensiva en el prompt para búsquedas más flexibles.

### 6.2 Bug #54 — LLM alucina URLs de multimedia

**Detectado:** 2026-07-16

**Síntoma:** El LLM devolvía URLs como `https://api.identechnology.co/ave/media/1/imagenes/resina1.jpg` cuando el tenant era empresa_id=9. Las URLs no existían en el servidor.

**Causa raíz:** Los ejemplos del módulo MEDIOS global usaban URLs literales con `/media/1/imagenes/archivo1.jpg`. El LLM las copiaba tal cual en lugar de llamar `buscar_media_servicio` para obtener las URLs reales.

**Solución aplicada:**

1. Módulo MEDIOS refactorizado: reemplazar URLs literales por placeholders `[URLs devueltas por buscar_media_servicio]`
2. Capa 2 del tenant reforzada con instrucción imperativa: "DEBES llamar buscar_media_servicio antes de generar tu respuesta"
3. Agregar recordatorio: "las URLs con /media/1/ son SOLO ejemplos, las reales tienen /media/9/"

### 6.3 Bug #55 — Line endings de Windows rompen detección

**Detectado:** 2026-07-16

**Síntoma:** El marcador `[[SPLIT:3000]]` funcionaba cuando estaba pegado inline al texto anterior (con caracteres residuales como `14[[SPLIT:3000]]a`), pero fallaba cuando estaba en su propia línea rodeado de saltos de línea.

**Causa raíz:** El texto llegaba a n8n con line endings de Windows (`\r\n`) en lugar de Unix (`\n`). El regex `\[\[SPLIT(?::(\w+))?\]\]` no capturaba correctamente el marcador cuando `\r` se colaba en el fragmento.

**Solución:** Agregar normalización al inicio de `Code_detectar_media`:

```javascript
output = output.replace(/\r\n/g, '\n').replace(/\r/g, '\n');
```

Con esto el formato queda robusto independiente del sistema operativo o cliente que genere el texto.

## 7. Aprendizajes arquitectónicos

### 7.1 Principio confirmado: ejemplos textuales literales

Durante la implementación del Bug #54 se confirmó de nuevo el principio arquitectónico clave:

> **El LLM copia ejemplos textuales del prompt LITERALMENTE, pero ignora o interpreta como opcional las reglas abstractas.**

Este principio guía todo el diseño de prompts en AVE. Aplicaciones concretas:

- Para forzar uso de una herramienta: incluir ejemplo textual del resultado esperado
- Para variables o URLs dinámicas: usar placeholders explícitos `[URL de la tool]`, nunca URLs literales
- Para reglas de sintaxis (como `[[SPLIT]]`): mostrar ejemplos con el marcador incluido

### 7.2 Timing en flujos paralelos

Cuando dos flujos corren en paralelo en n8n (texto vs media), el orden de llegada no es predecible:

- Textos son enviados con delays configurables (~800-3000ms)
- Media requiere descargar + upload binario (~1-4 segundos por archivo)

**Solución:** delegar el control de timing al prompt vía marcadores `[[SPLIT:xxxx]]`, permitiendo al operador AVE (o al cliente) coordinar el orden sin tocar código.

### 7.3 Robustez ante variaciones de input

Los sistemas que procesan output de LLMs deben ser robustos ante:

- **Line endings variables** (\r\n vs \n)
- **Espacios extras** al inicio/final de segmentos
- **Caracteres invisibles** que podrían romper regex

La normalización al inicio del procesamiento es una defensa barata y efectiva.

## 8. Configuración por tenant

### 8.1 Estado actual (2026-07-18)

| Tenant | ID | Modelo | [[SPLIT]] simple | [[SPLIT:xxxx]] custom | Estado |
|---|---|---|---|---|---|
| agencIA | 1 | gpt-4o-mini | Disponible | Disponible | Candidato para link Cal.com |
| Uhane SAS | 7 | gpt-4o-mini | Disponible | Disponible | Candidato para datos pago |
| PC_Outlet | 8 | gpt-4o-mini | Disponible | Disponible | Candidato para A/B/C |
| tienda4030 | 9 | gpt-4o-mini | En uso | En uso (Resina) | Producción |

### 8.2 Roadmap de replicación

- **agencIA**: aplicar `[[SPLIT:2000]]` en el envío del link de Cal.com para preview visual limpio
- **Uhane**: aplicar `[[SPLIT:1800]]` entre mensaje y datos de pago Bancolombia
- **PC_Outlet**: aplicar `[[SPLIT:2000]]` en presentación A/B/C con imágenes

## 9. Query de auditoría

Para verificar el uso del sistema en cualquier tenant:

```sql
SELECT
    empresa_id,
    LENGTH(prompt_capa_cliente) AS chars,
    (LENGTH(prompt_capa_cliente) - LENGTH(REPLACE(prompt_capa_cliente, '[[SPLIT]]', ''))) / LENGTH('[[SPLIT]]') AS splits_simples,
    (LENGTH(prompt_capa_cliente) - LENGTH(REPLACE(prompt_capa_cliente, '[[SPLIT:', ''))) / LENGTH('[[SPLIT:') AS splits_con_delay,
    updated_at
FROM bot_config
WHERE empresa_id IN (1, 7, 8, 9)
ORDER BY empresa_id;
```

## 10. Rollback plan

Si algún cambio genera problemas:

**Restaurar código de n8n:**
Importar el JSON del workflow previo desde el backup local.

**Restaurar prompt de tenant:**

```sql
UPDATE bot_config bc
SET prompt_capa_cliente = pre.prompt_capa_cliente
FROM bot_config_pre_camino_c_20260716 pre
WHERE bc.empresa_id = pre.empresa_id
  AND bc.empresa_id = 9;
```

**Restaurar NUCLEO:**

```sql
UPDATE modulos_bot mb
SET prompt_bloque = pre.prompt_bloque
FROM modulos_bot_pre_split_v2_20260716 pre
WHERE mb.codigo = pre.codigo
  AND mb.codigo = 'NUCLEO';
```

## 11. Referencias cruzadas

- `docs/troubleshooting/bugs-resueltos.md` — Bugs #52, #54, #55
- `docs/modulos/arquitectura-prompts-sandwich.md` v3.2 — Arquitectura actualizada
- `docs/promps/guia-maestra-prompts-ave.md` v2.3 — Guía con delays personalizados
- `docs/troubleshooting/queries-diagnostico.md` v1.2 — Queries de auditoría

---

**Fin del documento.**
