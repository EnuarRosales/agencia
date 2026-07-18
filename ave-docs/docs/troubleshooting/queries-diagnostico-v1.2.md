# Queries de Diagnóstico AVE

**Versión:** 1.2
**Fecha:** 2026-07-18
**Autor:** Operador AVE
**Reemplaza:** v1.1 (2026-07-15)

---

## Changelog v1.2 respecto a v1.1

- **Nueva sección: Auditoría de `[[SPLIT:xxxx]]` con delays dinámicos** — queries para verificar dónde están los marcadores con delays personalizados
- **Nueva query: Detección de servicios con URLs inconsistentes** (Bug #54) — verifica que las URLs de `servicios_media` coincidan con `empresa_id` del servicio
- **Nueva query: Auditoría de line endings** (Bug #55) — detecta si algún prompt tiene `\r\n` que podrían romper regex
- **Nueva query: Consumo de tokens estimado** con las nuevas documentaciones
- **Casos de uso actualizados** con nuevos escenarios

---

## Índice

1. Auditoría de `[[SPLIT]]` con delays
2. Consistencia de URLs multimedia (Bug #54)
3. Diagnóstico de servicios (Bug #52)
4. Memoria de n8n_chat_histories
5. Estado de OpenAI y modelos
6. Verificación de prompts
7. Análisis de conversaciones
8. Casos de uso consolidados

---

## 1. Auditoría de `[[SPLIT]]` con delays dinámicos

### 1.1 Contar marcadores por tenant (simples vs con delay)

**Propósito:** Ver cuántos marcadores de cada tipo hay por tenant.

```sql
SELECT
    empresa_id,
    LENGTH(prompt_capa_cliente) AS chars,
    (LENGTH(prompt_capa_cliente) - LENGTH(REPLACE(prompt_capa_cliente, '[[SPLIT]]', ''))) / LENGTH('[[SPLIT]]') AS splits_simples,
    (LENGTH(prompt_capa_cliente) - LENGTH(REPLACE(prompt_capa_cliente, '[[SPLIT:', ''))) / LENGTH('[[SPLIT:') AS splits_con_delay,
    modelo_bot,
    updated_at
FROM bot_config
WHERE empresa_id IN (1, 7, 8, 9)
ORDER BY empresa_id;
```

**Output esperado (2026-07-18):**

| empresa_id | chars | splits_simples | splits_con_delay | modelo_bot |
|---|---|---|---|---|
| 1 | ~5.500 | 0 | 0 | gpt-4o-mini |
| 7 | ~5.380 | 0 | 0 | gpt-4o-mini |
| 8 | ~14.081 | 0 | 0 | gpt-4o-mini |
| 9 | ~16.607 | 4 | 2 | gpt-4o-mini |

### 1.2 Ver contexto de los `[[SPLIT:xxxx]]` con delays

**Propósito:** Extraer los fragmentos donde están los marcadores con delay personalizado.

```sql
SELECT
    empresa_id,
    SUBSTRING(prompt_capa_cliente,
              POSITION('[[SPLIT:' IN prompt_capa_cliente) - 150,
              400) AS contexto_primer_split_delay
FROM bot_config
WHERE empresa_id = 9;
```

### 1.3 Listar los valores de delay usados

**Propósito:** Ver qué valores de delay están en uso en el prompt.

```sql
SELECT DISTINCT
    empresa_id,
    (regexp_matches(prompt_capa_cliente, '\[\[SPLIT:(\d+)\]\]', 'g'))[1] AS delay_valor
FROM bot_config
WHERE empresa_id = 9
ORDER BY delay_valor::int;
```

### 1.4 Verificar límite de 5 marcadores por turno

**Propósito:** Auditar que ningún ejemplo textual en el prompt exceda el límite defensivo.

```sql
-- Contar splits por bloque de ejemplo (aproximado)
WITH ejemplos AS (
  SELECT
    empresa_id,
    unnest(regexp_split_to_array(prompt_capa_cliente, 'Formato EXACTO')) AS bloque
  FROM bot_config
  WHERE empresa_id = 9
)
SELECT
  empresa_id,
  (LENGTH(bloque) - LENGTH(REPLACE(bloque, '[[SPLIT', ''))) / LENGTH('[[SPLIT') AS splits_en_bloque,
  LEFT(bloque, 100) AS inicio_bloque
FROM ejemplos
WHERE (LENGTH(bloque) - LENGTH(REPLACE(bloque, '[[SPLIT', ''))) / LENGTH('[[SPLIT') > 0
ORDER BY splits_en_bloque DESC;
```

**Bandera roja:** cualquier bloque con más de 5 splits.

---

## 2. Consistencia de URLs multimedia (Bug #54)

### 2.1 Detectar URLs con empresa_id incorrecto

**Propósito:** Verificar que las URLs en `servicios_media` coincidan con el `empresa_id` del servicio padre.

```sql
SELECT
    sm.id,
    s.empresa_id AS empresa_correcta,
    sm.tipo,
    sm.url,
    SUBSTRING(sm.url FROM '/media/(\d+)/') AS empresa_en_url,
    CASE
        WHEN SUBSTRING(sm.url FROM '/media/(\d+)/')::int = s.empresa_id THEN 'OK'
        ELSE 'ALERTA: empresa_id en URL no coincide con empresa_id del servicio'
    END AS estado
FROM servicios_media sm
JOIN servicios s ON s.id = sm.servicio_id
WHERE sm.activo = true
ORDER BY estado, sm.id;
```

### 2.2 Verificar simulación de búsqueda para todos los tenants

**Propósito:** Confirmar que un término natural devuelve los archivos correctos.

```sql
-- Simular buscar_media_servicio con término "reloj" en tienda4030
SELECT
    'reloj' AS termino,
    COUNT(*) AS encontrados,
    string_agg(DISTINCT sm.url, E'\n') AS urls_devueltas
FROM servicios s
JOIN servicios_media sm ON sm.servicio_id = s.id
WHERE s.empresa_id = 9
  AND s.activo = true
  AND sm.activo = true
  AND LOWER(s.nombre) ILIKE '%reloj%';
```

### 2.3 Detectar URLs alucinadas en logs

**Propósito:** Si tienes logs de `media_enviado_log`, detectar URLs que no existen.

```sql
SELECT
    m.id,
    m.empresa_id,
    m.servicio_nombre,
    m.url,
    CASE
        WHEN sm.id IS NULL THEN 'ALERTA: URL enviada pero no existe en servicios_media'
        ELSE 'OK'
    END AS estado
FROM media_enviado_log m
LEFT JOIN servicios_media sm ON sm.url = m.url AND sm.empresa_id = m.empresa_id
WHERE m.created_at > NOW() - INTERVAL '7 days'
ORDER BY estado DESC, m.created_at DESC
LIMIT 50;
```

---

## 3. Diagnóstico de servicios (Bug #52)

### 3.1 Detectar servicios con espacios dobles en nombre

```sql
SELECT
    id,
    empresa_id,
    nombre,
    LENGTH(nombre) AS chars_nombre,
    LENGTH(REGEXP_REPLACE(nombre, '\s+', ' ', 'g')) AS chars_normalizado,
    CASE
        WHEN LENGTH(nombre) != LENGTH(REGEXP_REPLACE(nombre, '\s+', ' ', 'g'))
        THEN 'ALERTA: contiene espacios multiples'
        ELSE 'OK'
    END AS estado
FROM servicios
WHERE activo = true
ORDER BY estado DESC, id;
```

### 3.2 Fix masivo de espacios dobles

```sql
UPDATE servicios
SET nombre = REGEXP_REPLACE(nombre, '\s+', ' ', 'g'),
    updated_at = NOW() AT TIME ZONE 'America/Bogota'
WHERE nombre != REGEXP_REPLACE(nombre, '\s+', ' ', 'g');
```

---

## 4. Memoria de n8n_chat_histories

### 4.1 Ver sesiones activas por tenant

```sql
SELECT
    session_id,
    COUNT(*) AS turnos,
    MAX(id) AS ultimo_id
FROM n8n_chat_histories
WHERE session_id LIKE '5_%'
GROUP BY session_id
ORDER BY ultimo_id DESC
LIMIT 10;
```

### 4.2 Limpiar memoria de todas las sesiones de un tenant

```sql
DELETE FROM n8n_chat_histories
WHERE session_id LIKE '5_%';
```

### 4.3 Detectar sesiones con historial largo (posible saturación)

```sql
SELECT
    session_id,
    COUNT(*) AS turnos_totales,
    MIN(id) AS primer_id,
    MAX(id) AS ultimo_id
FROM n8n_chat_histories
GROUP BY session_id
HAVING COUNT(*) > 30
ORDER BY turnos_totales DESC
LIMIT 10;
```

---

## 5. Estado de OpenAI y modelos

### 5.1 Verificar modelo actual por tenant

```sql
SELECT
    empresa_id,
    modelo_bot,
    modelo_clasificador,
    openai_key_status,
    updated_at
FROM bot_config
WHERE empresa_id IN (1, 7, 8, 9)
ORDER BY empresa_id;
```

**Estado esperado v3.2:** todos con `modelo_bot = 'gpt-4o-mini'`.

### 5.2 Ver historial de cambios de estado de OpenAI key

```sql
SELECT
    empresa_id,
    tipo_incidente,
    motivo,
    email_admin_enviado_at,
    key_fingerprint
FROM openai_key_incidents
WHERE empresa_id = 9
ORDER BY email_admin_enviado_at DESC
LIMIT 10;
```

---

## 6. Verificación de prompts

### 6.1 Tamaño del prompt total por tenant

```sql
SELECT
    empresa_id,
    LENGTH(prompt_capa_cliente) AS chars_capa2,
    LENGTH(system_prompt) AS chars_system_prompt_legacy,
    updated_at
FROM bot_config
WHERE empresa_id IN (1, 7, 8, 9)
ORDER BY empresa_id;
```

### 6.2 Auditoría de line endings en prompts (Bug #55)

**Propósito:** Detectar si algún prompt tiene `\r\n` que podrían causar problemas.

```sql
SELECT
    empresa_id,
    LENGTH(prompt_capa_cliente) AS chars,
    (LENGTH(prompt_capa_cliente) - LENGTH(REPLACE(prompt_capa_cliente, E'\r\n', E'\n'))) AS carriage_returns_detectados,
    CASE
        WHEN prompt_capa_cliente LIKE '%' || E'\r\n' || '%'
        THEN 'ALERTA: contiene line endings de Windows'
        ELSE 'OK: solo line endings Unix'
    END AS estado
FROM bot_config
WHERE empresa_id IN (1, 7, 8, 9);
```

**Fix si detecta CR:**

```sql
UPDATE bot_config
SET prompt_capa_cliente = REGEXP_REPLACE(prompt_capa_cliente, E'\r\n', E'\n', 'g'),
    updated_at = NOW() AT TIME ZONE 'America/Bogota'
WHERE prompt_capa_cliente LIKE '%' || E'\r' || '%';
```

**Nota:** el fix también se aplica en tiempo real en `Code_detectar_media`, pero limpiar en BD reduce el ruido en debug.

### 6.3 Extraer sección específica del prompt

```sql
SELECT
    empresa_id,
    SUBSTRING(prompt_capa_cliente,
              POSITION('SISTEMA DE BURBUJAS' IN prompt_capa_cliente),
              1000) AS seccion_split
FROM bot_config
WHERE empresa_id = 9;
```

### 6.4 Auditoría del módulo NUCLEO

```sql
SELECT
    codigo,
    LENGTH(prompt_bloque) AS chars,
    POSITION('SINTAXIS DISPONIBLE' IN prompt_bloque) AS pos_sintaxis_split,
    POSITION('LIMITE DE BURBUJAS' IN prompt_bloque) AS pos_limite_burbujas,
    updated_at
FROM modulos_bot
WHERE codigo = 'NUCLEO';
```

**Estado esperado (2026-07-18):** ambas posiciones > 0.

---

## 7. Análisis de conversaciones

### 7.1 Conversaciones recientes de un tenant

```sql
SELECT
    c.id,
    c.chatwoot_conversation_id,
    l.nombre,
    l.telefono,
    l.estado_lead,
    c.updated_at
FROM conversaciones c
JOIN leads l ON c.lead_id = l.id
WHERE c.empresa_id = 9
  AND c.updated_at > NOW() - INTERVAL '24 hours'
ORDER BY c.updated_at DESC
LIMIT 20;
```

### 7.2 Pedidos registrados en las últimas 24 horas

```sql
SELECT
    p.id,
    p.empresa_id,
    l.nombre,
    p.productos,
    p.total_estimado,
    p.created_at
FROM pedidos p
JOIN leads l ON p.lead_id = l.id
WHERE p.created_at > CURRENT_DATE
ORDER BY p.created_at DESC;
```

### 7.3 Media enviada por tenant (auditoría de flujo con [[SPLIT]])

```sql
SELECT
    m.empresa_id,
    m.servicio_nombre,
    m.tipo,
    COUNT(*) AS envios,
    MAX(m.created_at) AS ultimo_envio
FROM media_enviado_log m
WHERE m.created_at > NOW() - INTERVAL '7 days'
  AND m.empresa_id = 9
GROUP BY m.empresa_id, m.servicio_nombre, m.tipo
ORDER BY envios DESC;
```

---

## 8. Casos de uso consolidados

### Cuando el `[[SPLIT]]` con delays no funciona

1. Ejecuta **1.1** para confirmar que hay marcadores en el prompt
2. Ejecuta **1.2** para ver el contexto exacto
3. Ejecuta **6.2** para descartar line endings de Windows (Bug #55)
4. Revisar en n8n Executions el output de `Code_detectar_media`
5. Verificar que el nodo tenga el modo `Run Once for All Items`

### Cuando el bot envía URLs alucinadas

1. Ejecuta **2.1** para ver si hay URLs con empresa_id inconsistente
2. Ejecuta **2.2** para simular la búsqueda con términos genéricos
3. Ejecuta **2.3** para revisar logs de envíos
4. Verificar que el prompt use placeholders `[URLs de tool]` en lugar de URLs literales
5. Reforzar en Capa 2 la instrucción "DEBES llamar buscar_media_servicio"

### Cuando el bot dice "no tiene fotos" del producto

1. Ejecuta **3.1** para detectar espacios dobles (Bug #52)
2. Simular búsqueda con la Query **2.2** con el término natural
3. Si tiene doble espacio, aplicar el fix con REGEXP_REPLACE de **3.2**

### Antes de aplicar cambios grandes al NUCLEO

1. Ejecuta **6.4** para ver estado actual del NUCLEO
2. Crea backup preventivo: `CREATE TABLE modulos_bot_pre_cambio_YYYYMMDD AS SELECT * FROM modulos_bot`
3. Aplica el cambio
4. Ejecuta **6.4** de nuevo para verificar el diff
5. Prueba con un mensaje real en cualquier tenant

### Auditoría mensual completa

1. Ejecuta **5.1** — estado de modelos y OpenAI keys
2. Ejecuta **6.2** — line endings limpios
3. Ejecuta **2.1** — consistencia de URLs multimedia
4. Ejecuta **3.1** — espacios dobles en servicios
5. Ejecuta **7.1** — actividad conversacional
6. Ejecuta **7.3** — envíos de multimedia
7. Documentar hallazgos en `docs/troubleshooting/`

---

## Referencias cruzadas

- `docs/troubleshooting/bugs-resueltos.md` — Registro completo de bugs
- `docs/troubleshooting/bugs-54-y-55-consolidados.md` — Bugs #54 y #55
- `docs/features/split-delays-dinamicos.md` — Feature completa
- `docs/modulos/arquitectura-prompts-sandwich.md` v3.2
- `docs/promps/guia-maestra-prompts-ave.md` v2.3

---

**Fin del documento queries-diagnostico.md v1.2**
