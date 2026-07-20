# Checklist de Onboarding v3.0 - Arquitectura Sándwich + Prevención Bug 57

**Ruta sugerida:** `docs/onboarding-clientes/checklist-onboarding-v3.md`
**Versión:** v3.0
**Fecha:** 2026-07-20
**Origen:** aprendizajes del debugging AnaMar + validación con 5 tenants en producción
**Aplicable a:** onboarding de las 12 empresas restantes de la Comunidad 4030

---

## 1. Contexto y objetivo

Este checklist consolida el patrón validado durante el onboarding de AnaMar (empresa_id=12) y
las lecciones aprendidas resolviendo los bugs 57 (NULL en Capa 3) y 58 (memoria contaminada).

**Meta:** que cualquier operador AVE pueda crear un tenant desde cero, activarlo en producción y
verlo funcionando por WhatsApp end-to-end en menos de 60 minutos, sin sorpresas.

---

## 2. Prerrequisitos

Antes de iniciar el checklist debes tener:

- Chatwoot inbox del tenant creado (WABA + Phone Number ID + webhook configurado)
- OpenAI API key del tenant (validada y con saldo)
- URL de imágenes/videos del producto ya cargadas en `servicios_media` (o listas para cargar)
- Plantilla del prompt V6 estilo Jhoana o V7 estilo Ana adaptada al tono del tenant
- Acceso a Postgres AVE, n8n, Chatwoot, WhatsApp de prueba

---

## 3. Paso 1 - Crear empresa en Postgres

```sql
-- Ejemplo genérico. Reemplazar valores entre <> por los reales del tenant.
INSERT INTO empresas (
  nombre, chatwoot_url, chatwoot_api_key, chatwoot_account_id,
  email, activo
) VALUES (
  '<Nombre Comercial>',
  'https://chatwoot.identechnology.co',
  '<chatwoot_api_key>',
  <chatwoot_account_id>,
  '<email_admin>',
  true
)
RETURNING id;
-- Guardar el id devuelto como <empresa_id> para los siguientes pasos.
```

---

## 4. Paso 2 - Crear `bot_config` con Capa 3 EXPLÍCITAMENTE inicializada

**Regla crítica derivada del bug 57:** `prompt_capa_reglas_fin` NUNCA debe quedar NULL. Siempre
inicializar con `''` (string vacío) aunque no vayas a usar Capa 3 al inicio.

```sql
INSERT INTO bot_config (
  empresa_id, openai_api_key, openai_key_status,
  modelo_bot, modelo_clasificador, temperatura_bot, max_tokens_bot,
  objetivo_bot, tono,
  prompt_clasificador,
  prompt_capa_cliente,
  prompt_capa_reglas_fin      -- CRÍTICO: siempre '' explícito, nunca NULL
) VALUES (
  <empresa_id>,
  '<openai_api_key>',
  'active',
  'gpt-4o-mini', 'gpt-4o-mini',
  0.7, 1000,
  '<objetivo comercial del bot>',
  '<tono de comunicación deseado>',
  '<prompt_clasificador estándar>',
  '<prompt_capa_cliente V7 con few-shots>',
  ''                            -- Nunca NULL, siempre ''
);
```

### Auditoría obligatoria post-INSERT

```sql
SELECT empresa_id,
       prompt_capa_cliente IS NULL     AS capa2_null,
       prompt_capa_reglas_fin IS NULL  AS capa3_null,
       LENGTH(prompt_capa_cliente)     AS capa2_chars,
       LENGTH(prompt_capa_reglas_fin)  AS capa3_chars
FROM bot_config
WHERE empresa_id = <empresa_id>;
```

**Ambos `_null` deben ser `false`.** Si alguno es `true` → detener onboarding y corregir con
`UPDATE bot_config SET ... = '' WHERE empresa_id = <empresa_id>`.

---

## 5. Paso 3 - Activar módulos según tipo de tenant

Usar `modulo_id` real (PK). Ver tabla en `docs/modulos/aclaracion-ids-modulos.md`.

### 5.1 Tenant dropshipping / e-commerce con pedidos (default para Comunidad 4030)

```sql
INSERT INTO empresa_modulos (empresa_id, modulo_id, activo) VALUES
  (<empresa_id>, 1, true),   -- NUCLEO
  (<empresa_id>, 3, true),   -- MEDIOS
  (<empresa_id>, 4, true),   -- PEDIDOS
  (<empresa_id>, 5, true);   -- GUARDRAILS
```

### 5.2 Tenant servicios con agendamiento

```sql
INSERT INTO empresa_modulos (empresa_id, modulo_id, activo) VALUES
  (<empresa_id>, 1, true),   -- NUCLEO
  (<empresa_id>, 2, true),   -- AGENDAMIENTO
  (<empresa_id>, 3, true),   -- MEDIOS
  (<empresa_id>, 5, true);   -- GUARDRAILS
```

### 5.3 Verificación de activación

```sql
SELECT em.empresa_id, m.nombre, em.modulo_id, m.orden, em.activo
FROM empresa_modulos em
JOIN modulos_bot m ON m.id = em.modulo_id
WHERE em.empresa_id = <empresa_id>
ORDER BY m.orden;
```

Deben aparecer 4 filas con `activo = true`.

---

## 6. Paso 4 - Cargar `servicios`, `info_negocio`, `servicios_media`

Aquí no hay cambios respecto a versiones anteriores del checklist. Verificar:

- Al menos 1 fila en `servicios` con `activo = true`
- Al menos 1 fila en `info_negocio` con contenido relevante para `info_negocio_pg`
- URLs de imágenes/videos cargadas en `servicios_media` y accesibles públicamente

---

## 7. Paso 5 - Configurar `etiquetas_pipeline`

```sql
INSERT INTO etiquetas_pipeline (empresa_id, nombre, chatwoot_prioridad, color, es_conversion, orden, activo) VALUES
  (<empresa_id>, 'frio',     'none',   '#FFB6C1', false, 1, true),
  (<empresa_id>, 'caliente', 'medium', '#0000FF', false, 2, true),
  (<empresa_id>, 'pedido',   'urgent', '#089308', true,  3, true);
```

---

## 8. Paso 6 - Generar prompt V7 con SOMI

Usar SOMI v3.2 para generar el `prompt_capa_cliente` siguiendo el estándar 4030 amplificado.
Elementos obligatorios verificables:

- Identidad clara del asistente (nombre + empresa)
- Reglas de comportamiento (max palabras, max preguntas, uso de emojis)
- Bloque completo de HERRAMIENTA MULTIMEDIA con reglas anti-markdown
- Mínimo 6 few-shots con **URLs reales** del tenant (no URLs genéricas ni placeholders)
- Ejemplo explícito de RESPUESTA INCORRECTA como contra-ejemplo
- Marcadores `[[SPLIT:2500]]`, `[[SPLIT:2000]]`, `[[SPLIT]]` según flujo
- Referencia a tools disponibles según módulos activos
- Recordatorio crítico final sobre formato de URLs

Longitud objetivo: 8000 - 12000 caracteres. Referencias:

- AnaMar V7: 10543 chars (funciona bien)
- tienda4030 V6 Jhoana: 16607 chars (funciona bien, más detalle)
- Uhane: 5308 chars (funciona bien, tenant simple)
- PC_Outlet: 14081 chars (funciona bien, tenant complejo)

---

## 9. Paso 7 - Aplicar `prompt_capa_cliente` con `$BODY$` + backup + verificación

```sql
-- Backup preventivo
CREATE TABLE backup_bot_config_<empresa>_v1_<yyyymmdd> AS
SELECT * FROM bot_config WHERE empresa_id = <empresa_id>;

-- UPDATE con $BODY$ (NO usar $$, revisar bug 50)
UPDATE bot_config
SET prompt_capa_cliente = $BODY$
<contenido completo del prompt generado por SOMI>
$BODY$,
    updated_at = NOW()
WHERE empresa_id = <empresa_id>;

-- Verificación obligatoria de longitud (nunca omitir)
SELECT empresa_id,
       LENGTH(prompt_capa_cliente) AS capa2_chars,
       LENGTH(prompt_capa_reglas_fin) AS capa3_chars,
       updated_at
FROM bot_config
WHERE empresa_id = <empresa_id>;
```

Si `capa2_chars` sale muy por debajo de lo esperado (< 6000) → el UPDATE probablemente falló por
sintaxis. Revisar delimitadores y reintentar.

---

## 10. Paso 8 - Sincronizar `system_prompt` como respaldo histórico (opcional)

Algunos flujos legacy usan el campo `system_prompt` como respaldo. Si aplica al tenant:

```sql
UPDATE bot_config
SET system_prompt = prompt_capa_cliente
WHERE empresa_id = <empresa_id>;
```

---

## 11. Paso 9 - Prueba end-to-end por WhatsApp

Antes de la primera prueba, si el tenant tuvo pruebas manuales previas, **limpiar la memoria**
para evitar contaminación (ver bug 58):

```sql
-- Verificar si hay memoria previa
SELECT COUNT(*) AS mensajes_previos
FROM n8n_chat_histories
WHERE session_id LIKE '<chatwoot_account_id>_%';

-- Si hay, backup y limpiar
CREATE TABLE backup_chat_memory_<empresa>_<yyyymmdd> AS
SELECT * FROM n8n_chat_histories
WHERE session_id LIKE '<chatwoot_account_id>_%';

DELETE FROM n8n_chat_histories
WHERE session_id LIKE '<chatwoot_account_id>_%';
```

### 3 escenarios de prueba obligatorios

1. **Escenario A - Interés directo en producto**
   - Enviar: *"me interesa la resina"* (o el producto principal del tenant)
   - Verificar: saludo + fotos/video como archivos reales + beneficios + pregunta de cierre
   - Verificar: NO aparece markdown (`###`, `1.`, `![texto](url)`, `"aquí tienes"`)

2. **Escenario B - Precio directo desde el primer mensaje**
   - Enviar: *"cuanto vale?"*
   - Verificar: saludo + beneficios + fotos + escalonado de precios + pregunta de cierre

3. **Escenario C - Saludo genérico**
   - Enviar: *"hola"*
   - Verificar: saludo + pregunta de ciudad, SIN llamar `buscar_media_servicio` todavía

---

## 12. Paso 10 - Auditoría final

Query maestra de auditoría post-onboarding:

```sql
SELECT
  bc.empresa_id,
  e.nombre AS empresa,
  bc.prompt_capa_reglas_fin IS NULL AS capa3_null,
  LENGTH(bc.prompt_capa_cliente)    AS capa2_chars,
  LENGTH(bc.prompt_capa_reglas_fin) AS capa3_chars,
  (SELECT COUNT(*) FROM empresa_modulos WHERE empresa_id = bc.empresa_id AND activo = true) AS modulos_activos,
  (SELECT STRING_AGG(m.nombre, ', ' ORDER BY m.orden)
   FROM empresa_modulos em JOIN modulos_bot m ON m.id = em.modulo_id
   WHERE em.empresa_id = bc.empresa_id AND em.activo = true) AS modulos,
  bc.modelo_bot,
  bc.openai_key_status,
  bc.updated_at
FROM bot_config bc
JOIN empresas e ON e.id = bc.empresa_id
WHERE bc.empresa_id = <empresa_id>;
```

### Criterios de aprobación

- [ ] `capa3_null = false`
- [ ] `capa2_chars` entre 6000 y 16000
- [ ] `modulos_activos >= 3`
- [ ] `openai_key_status = 'active'`
- [ ] `modelo_bot = 'gpt-4o-mini'` (default estándar)
- [ ] Los 3 escenarios de prueba funcionaron sin markdown residual
- [ ] Al menos 1 pedido de prueba registrado correctamente en tabla `pedidos` (si aplica)

---

## 13. Anti-checklist - errores comunes a evitar

Estos son los errores más frecuentes documentados en incidentes previos:

| Error | Bug asociado | Prevención |
|-------|--------------|------------|
| Dejar `prompt_capa_reglas_fin` como NULL | 57 | Siempre inicializar con `''` en el INSERT |
| No crear `empresa_modulos` para el nuevo tenant | (no numerado) | Ver Paso 3 arriba |
| Usar `modulo_id = 30` en vez de `modulo_id = 3` | Confusión histórica | Ver tabla en `aclaracion-ids-modulos.md` |
| No limpiar memoria después de refactor de prompt | 58 | Ver Paso 9 arriba |
| Usar `$$` en vez de `$BODY$` para UPDATE largo | 44/50 | Ver `guia-maestra-prompts-ave.md` |
| Omitir verificación de LENGTH post-UPDATE | 44/50 | Nunca omitir el SELECT posterior |
| Copiar queries hardcodeadas al forkear Appsmith | 49 | Revisar todos los widgets tras el fork |
| Meta-referencias a NUCLEO en Capa 2/3 | 46 | Redactar en imperativo directo sin auto-referencias |

---

## 14. Referencias cruzadas

- `docs/modulos/arquitectura-prompts-sandwich.md` (arquitectura de 3 capas)
- `docs/modulos/aclaracion-ids-modulos.md` (tabla canónica de IDs)
- `docs/troubleshooting/bugs-resueltos.md` (bugs 57 y 58 detallados)
- `docs/promps/guia-maestra-prompts-ave.md` (operativa segura con `$BODY$`, backup, LENGTH)
- `docs/modulos/panel-cliente-appsmith.md` (Panel Cliente self-service)

---

## 15. Estado de los 5 tenants tras aplicar este checklist

Ejemplo real de auditoría post-cierre de AnaMar (referencia):

| empresa_id | Empresa    | capa2 | capa3 | módulos | modelo      | estado       |
|------------|------------|-------|-------|---------|-------------|--------------|
| 1          | agencIA    | 6302  | 0     | 4       | gpt-4o-mini | producción   |
| 7          | Uhane SAS  | 5308  | 0     | 4       | gpt-4o-mini | producción   |
| 8          | PC_Outlet  | 14081 | 0     | 3       | gpt-4o-mini | producción   |
| 9          | tienda4030 | 16607 | 0     | 4       | gpt-4o-mini | producción   |
| 12         | AnaMar     | 10543 | 0     | 4       | gpt-4o-mini | producción   |

Los 5 pasan la auditoría del Paso 10. Este es el estado objetivo para los 12 tenants restantes
de la Comunidad 4030.
