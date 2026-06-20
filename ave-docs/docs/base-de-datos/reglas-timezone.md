# Reglas de Zona Horaria (Timezone)

Documento crítico. El manejo incorrecto de zonas horarias causó varios bugs difíciles de diagnosticar en AVE. Esta es la regla a seguir **siempre** al escribir queries que comparen o escriban timestamps.

---

## 1. El problema de fondo

La infraestructura corre en **UTC**:
- El sistema operativo del VPS está en `Etc/UTC`.
- El contenedor de n8n está en `Etc/UTC`.
- El contenedor de Postgres está en `Etc/UTC` (su reloj interno).

Pero **la sesión de Postgres** puede tener una zona horaria distinta:
- **DBeaver** (tu cliente manual): sesión en `America/Bogota`.
- **n8n** (la conexión del workflow): sesión en `Etc/UTC`.

Esto significa que `NOW()` devuelve valores distintos según quién ejecute el query:
- En DBeaver: `NOW()` → hora de Bogotá (correcta).
- En n8n: `NOW()` → hora UTC (5 horas adelantada respecto a Bogotá).

Verificación:
```sql
-- Ejecutado desde n8n devuelve:
SELECT NOW(), current_setting('TIMEZONE');
-- now: 2026-06-19T15:41:31Z  |  current_setting: Etc/UTC
```

---

## 2. La regla (según el tipo de columna)

El tratamiento depende de si la columna es `WITH` o `WITHOUT` time zone.

### Columnas `timestamp WITHOUT time zone`

Estas columnas **no guardan información de zona horaria** — almacenan el número tal cual llega. Si se escriben con un `NOW()` en UTC, quedan en UTC sin que nada lo indique.

**Regla:** al escribir o comparar, usar siempre:
```sql
NOW() AT TIME ZONE 'America/Bogota'
```

**Columnas afectadas en `conversaciones`:**
- `ultimo_mensaje_usuario_at`
- `created_at`
- `updated_at`

### Columnas `timestamp WITH time zone`

Estas columnas **sí manejan zona horaria** — Postgres las almacena internamente en UTC y las convierte automáticamente según la sesión que consulta.

**Regla:** al comparar, usar `NOW()` puro, **sin** `AT TIME ZONE`:
```sql
NOW()
```

**Columnas afectadas en `conversaciones`:**
- `ultimo_recordatorio_at`

⚠️ **Si se mezcla el tratamiento** (por ejemplo, usar `AT TIME ZONE` sobre una columna `WITH time zone`), se introduce un desfase y las comparaciones fallan silenciosamente.

---

## 3. Cómo verificar el tipo de una columna

```sql
SELECT column_name, data_type
FROM information_schema.columns
WHERE table_name = 'conversaciones'
  AND column_name IN ('ultimo_mensaje_usuario_at', 'ultimo_recordatorio_at', 'created_at', 'updated_at');
```

Resultado esperado:
- `timestamp without time zone` → usar `AT TIME ZONE 'America/Bogota'`
- `timestamp with time zone` → usar `NOW()` puro

---

## 4. Ejemplo correcto (del workflow de recordatorios)

```sql
CASE
  WHEN c.recordatorios_enviados = 0 THEN
    -- ultimo_mensaje_usuario_at es WITHOUT time zone → AT TIME ZONE
    c.ultimo_mensaje_usuario_at < (NOW() AT TIME ZONE 'America/Bogota') - INTERVAL '1 hour' * (...)
  ELSE
    -- ultimo_recordatorio_at es WITH time zone → NOW() puro
    c.ultimo_recordatorio_at < NOW() - INTERVAL '1 hour' * (...)
END
```

---

## 5. Corrección de datos ya escritos mal

Si una fila quedó con un timestamp en UTC cuando debía estar en Bogotá (desfase de ~5 horas), corregir restando el offset:

```sql
UPDATE conversaciones
SET ultimo_mensaje_usuario_at = ultimo_mensaje_usuario_at - INTERVAL '5 hours'
WHERE id = [ID];
```

(El fix de zona horaria en los queries solo aplica hacia adelante; los datos viejos no se corrigen solos.)

---

## 6. Síntoma típico del bug

Si ves que `NOW() - ultimo_mensaje_usuario_at` da un **valor negativo** (tiempo "en el futuro"), es señal de que el timestamp se guardó en una zona horaria distinta a la que usa la comparación. Revisar el tipo de columna y aplicar la regla correspondiente.

---

## 8. Verificación obligatoria tras aplicar el fix

Este bug **reapareció** después de haber sido corregido, porque el fix se perdió al reconstruir el workflow por otro motivo (ver [Bug 9](../troubleshooting/bugs-resueltos.md#bug-9-recurrencia-del-bug-de-timezone-tras-reconstrucción-del-workflow)).

**Regla:** después de aplicar este fix en cualquier nodo de n8n, verificar inmediatamente abriendo el nodo en el **editor en vivo** y confirmando que el texto del query contiene `AT TIME ZONE 'America/Bogota'` donde corresponde. No basta con haberlo aplicado una vez — cualquier reconstrucción o reimportación posterior del workflow puede revertirlo silenciosamente si se parte de una versión base desactualizada.

## 9. Auditoría de otras tablas (completada 2026-06-20)

Se auditaron `leads` y `citas` por el mismo patrón de bug. Resultado: **ambas tablas tenían el mismo problema** (`NOW()` sin `AT TIME ZONE`) en 5 nodos del workflow principal.

| Tabla | Columna | Tipo | Nodo afectado | Estado |
|---|---|---|---|---|
| leads | `ultima_interaccion` | without time zone | `PG_upsert_lead` | ✅ Corregido |
| leads | `updated_at` | without time zone | `PG_upsert_lead` | ✅ Corregido |
| leads | `updated_at` | without time zone | `PG_actualizar_estado_lead` | ✅ Corregido |
| leads | `updated_at` | without time zone | `actualizar_lead_datos` | ✅ Corregido |
| leads | `updated_at` | without time zone | `PG_actualizar_estado_lead_directo` | ✅ Corregido |
| citas | `updated_at` | without time zone | `cancelar_cita_pg` | ✅ Corregido |

**Nota:** `citas.fecha_inicio` y `citas.fecha_fin` (también `without time zone`) **no se corrigen con este fix** — se escriben directamente desde el razonamiento del agente IA (`$fromAI(...)::timestamp`), no desde `NOW()`. Su corrección depende de que el agente calcule bien la fecha, lo cual está ligado al hallazgo de la sección 10.

`eventos_lead` — no se encontró ningún nodo en `Bot_Agencia_final` que escriba en esta tabla. Pendiente confirmar si se usa desde otro proceso o si es una tabla sin implementar aún.

## 10. Hallazgo relacionado: posible bug en `{fecha_actual}` del system prompt

El `system_prompt` de cada empresa reemplaza el placeholder `{fecha_actual}` así:
```javascript
.replace('{fecha_actual}', $now.toFormat('dd/MM/yyyy'))
```

`$now` en n8n usa la zona horaria **global configurada en la instancia de n8n** (no la del Postgres). Si esa configuración global está en UTC (probable, dado que el resto de la infraestructura lo está), entre las **7pm y medianoche hora Bogotá** el agente recibiría la fecha del día **siguiente** como "hoy" — pudiendo calcular mal fechas relativas como "mañana" al agendar citas en ese rango horario.

**Estado:** detectado, sin confirmar ni corregir todavía. **Cómo verificar:** preguntarle al bot "¿qué fecha es hoy?" en horario nocturno (después de 7pm Bogotá) y comparar contra la fecha real.

**Fix propuesto (pendiente de validar):**
```javascript
.replace('{fecha_actual}', $now.setZone('America/Bogota').toFormat('dd/MM/yyyy'))
```

## 11. Pendiente

- ~~Revisar si el mismo problema afecta `leads`, `citas`, `eventos_lead`~~ → Completado, ver sección 9.
- Confirmar si `eventos_lead` se usa desde algún otro proceso.
- Investigar y corregir el posible bug de `$now` en `{fecha_actual}` (sección 10).
