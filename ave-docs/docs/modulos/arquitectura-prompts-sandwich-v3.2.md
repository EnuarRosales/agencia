# Arquitectura de Prompts Sándwich - AVE

**Versión:** 3.2
**Fecha:** 2026-07-18
**Autor:** Operador AVE
**Reemplaza:** v3.1 (2026-07-15)

---

## Changelog v3.2 respecto a v3.1

Cambios aplicados en las sesiones del 2026-07-16 al 2026-07-18:

1. **Sistema `[[SPLIT]]` extendido con delays dinámicos personalizables desde el prompt**. Sintaxis `[[SPLIT:xxxx]]` donde xxxx es milisegundos de delay (300-6000).
2. **Módulo NUCLEO ampliado** con documentación completa del sistema `[[SPLIT]]` y sus delays personalizados (accesible por los 4 tenants).
3. **Módulo MEDIOS refactorizado** para prevenir alucinación de URLs por el LLM (Bug #54).
4. **`Code_detectar_media` refactorizado** con normalización de line endings (Bug #55) y límite defensivo de segmentos.
5. **`Code_enviar_burbujas` refactorizado** para respetar delays dinámicos definidos en cada segmento.
6. **Bug #52** (doble espacio en nombres de servicios) documentado y resuelto en tienda4030.
7. **Feature multimedia integrada con `[[SPLIT]]`** validada en producción para tienda4030.

---

## 1. Resumen ejecutivo

La arquitectura sándwich v3.2 mantiene el modelo de 3 capas de v3.1 (Capa 1 modular global + Capa 2 personalidad del cliente + Capa 3 deprecated) y consolida el sistema `[[SPLIT]]` con delays dinámicos como una feature completa, escalable y arquitectónicamente correcta.

Cambios principales:

- El sistema `[[SPLIT:xxxx]]` permite control total del ritmo conversacional desde el prompt
- La documentación del sistema vive en NUCLEO (accesible a los 4 tenants)
- El módulo MEDIOS ya no expone URLs literales que el LLM podría copiar erróneamente
- La normalización de line endings protege el sistema ante inputs de diferentes plataformas
- Todos los tenants pueden implementar burbujas con delays personalizados sin tocar código

---

## 2. Modelo de 3 capas (sin cambios desde v3.0)

**Capa 1** (módulos globales): NUCLEO + AGENDAMIENTO + MEDIOS + PEDIDOS + GUARDRAILS
**Capa 2** (por tenant): `bot_config.prompt_capa_cliente`
**Capa 3**: deprecated (columna preservada, contenido vacío)

---

## 3. Módulos globales — Estado v3.2

| Módulo | Código | Orden | Versión | Cambios v3.2 |
|---|---|---|---|---|
| NUCLEO | NUCLEO | 10 | v3 | Ampliado con documentación completa de `[[SPLIT:xxxx]]` |
| AGENDAMIENTO | AGENDAMIENTO | 20 | v1 | Sin cambios |
| MEDIOS | MEDIOS | 30 | v4 | Refactorizado para prevenir alucinación de URLs |
| PEDIDOS | PEDIDOS | 40 | v3 | Sin cambios |
| GUARDRAILS | GUARDRAILS | 100 | v2 | Sin cambios |

---

## 4. Sistema `[[SPLIT]]` con delays dinámicos (feature clave v3.2)

### 4.1 Objetivo arquitectónico

Permitir al operador AVE (o al cliente vía Panel Cliente Appsmith) controlar el ritmo conversacional del bot sin modificar código, mediante marcadores en el prompt que definen cuándo y cuánto esperar entre burbujas.

### 4.2 Sintaxis extendida

```
[[SPLIT]]          → delay default de 800ms
[[SPLIT:2500]]     → delay personalizado de 2500ms
[[SPLIT:1500]]     → delay de 1500ms
[[SPLIT:N]]        → N milisegundos (rango 300-6000)
```

Valores fuera de rango se ajustan automáticamente al mínimo/máximo permitido.

### 4.3 Arquitectura de procesamiento

```
AI_Agent_Principal responde con [[SPLIT:xxxx]]
        ↓
Code_detectar_media (v3.0 con parseDelay)
   1. Normaliza \r\n → \n (fix Bug #55)
   2. Detecta marcadores con regex
   3. Extrae delay de cada marcador
   4. Aplica límite defensivo (max 6 segmentos)
   5. Retorna segmentos con metadata { texto, tiene_media, archivos, delay_antes_ms }
        ↓
        ├──→ If_es_multiple (TRUE) → Code_enviar_burbujas
        │        1. Itera cada segmento
        │        2. Aplica delay_antes_ms antes de enviar
        │        3. Envía texto a Chatwoot API
        │
        └──→ If_tiene_media (TRUE) → SplitInBatches_media
                 1. Descarga archivos como binario
                 2. Envía a Chatwoot como attachment
                 3. WhatsApp renderiza nativamente
```

**Flujos paralelos:** ambos flujos corren al mismo tiempo. El operador AVE controla el orden coordinando los delays en el prompt.

### 4.4 Reglas de uso (documentadas en NUCLEO global)

**USAR `[[SPLIT]]` cuando:**

- Envío de URLs importantes (agendamiento, pagos, catálogo)
- Envío de material multimedia + mensaje explicativo
- Mensaje largo estructurado con información diferenciada

**NO usar `[[SPLIT]]` cuando:**

- Saludos iniciales o respuestas cortas
- Preguntas simples de calificación
- Cierres transaccionales
- Respuestas conversacionales naturales

**Límites:**

- Máximo 5 marcadores por turno (6 burbujas totales)
- Delay mínimo: 300ms, máximo: 6000ms
- El sistema aplica límites automáticamente

### 4.5 Ejemplos canónicos por caso de uso

**Caso 1: Envío de link con contexto**

```
"Aquí tienes el link para agendar:
[[SPLIT:2000]]
https://citas.identechnology.co/team/agencia/...
[[SPLIT]]
Recibirás la confirmación por correo."
```

**Caso 2: Presentación de producto con multimedia**

```
"Que gusto atenderte, soy Jhoana
[[SPLIT:2500]]
[URLs de fotos y video de la Resina]
[[SPLIT:2000]]
Beneficios del producto:
✅ Beneficio 1
✅ Beneficio 2
[[SPLIT]]
Cuentanos, de que ciudad nos escribes?"
```

**Caso 3: Confirmación con datos estructurados**

```
"Perfecto, aquí tienes los datos de pago:
[[SPLIT:1800]]
Bancolombia Ahorros
Cuenta: 63400002911
Titular: Uhane SAS
[[SPLIT:1500]]
Cuando confirmes el pago, avísame y coordino el despacho."
```

---

## 5. Módulo MEDIOS v4 — Prevención de URLs alucinadas

### 5.1 Motivación del cambio

En v3.1, el módulo MEDIOS contenía ejemplos con URLs literales:

```
https://api.identechnology.co/ave/media/1/imagenes/archivo1.jpg
```

Estos ejemplos causaban que el LLM copiara las URLs textualmente (con `/media/1/`) en lugar de invocar `buscar_media_servicio` para obtener las URLs reales del tenant activo. **Bug #54.**

### 5.2 Solución arquitectónica

**Ejemplos con placeholders explícitos** en lugar de URLs literales:

```
Cliente pide fotos:
"Con gusto, aqui tiene las imagenes:
[URLs exactas devueltas por buscar_media_servicio, cada una en su propia linea]"

CRITICO: NUNCA inventes URLs ni las copies de estos ejemplos. Las URLs entre corchetes son SOLO placeholders. SIEMPRE llama buscar_media_servicio para obtener las URLs reales del catalogo del tenant activo.
```

### 5.3 Refuerzo en Capa 2 del tenant

Para casos críticos (como el envío inicial de fotos en tienda4030), agregar en la Capa 2 del tenant una instrucción imperativa adicional:

```
CRITICO ABSOLUTO: DEBES llamar a buscar_media_servicio con el termino del producto ANTES de generar tu respuesta. NUNCA inventes URLs ni las copies de ejemplos que veas en las reglas.
```

Esta redundancia (regla global en MEDIOS + refuerzo específico en Capa 2) protege contra los casos donde el LLM podría seguir el ejemplo literal.

---

## 6. Bugs resueltos en v3.2

### Bug #52 — Doble espacio en nombres de servicios (2026-07-16)

- **Síntoma:** `buscar_media_servicio` no encontraba servicios cuando el nombre tenía doble espacio interno
- **Causa:** `ILIKE` es literal, `"Reloj  del"` (doble espacio) no matchea con `"reloj del"` (uno)
- **Solución:** UPDATE del nombre en BD + validación defensiva

### Bug #54 — LLM alucina URLs de multimedia (2026-07-16)

- **Síntoma:** El LLM devolvía URLs con empresa_id incorrecto (`/media/1/`)
- **Causa:** El LLM copiaba literalmente las URLs del ejemplo en el módulo MEDIOS
- **Solución:** Placeholders explícitos en ejemplos + refuerzo en Capa 2

### Bug #55 — Line endings de Windows rompen detección (2026-07-16)

- **Síntoma:** `[[SPLIT:xxxx]]` funcionaba inline pero fallaba con formato limpio en líneas separadas
- **Causa:** `\r\n` de Windows interferían con el regex de detección
- **Solución:** Normalización `\r\n → \n` al inicio de `Code_detectar_media`

Detalles completos en `docs/troubleshooting/bugs-54-y-55-consolidados.md`.

---

## 7. Estado por tenant al cierre v3.2

| Tenant | ID | Modelo | Capa 2 [[SPLIT]] | Capa 2 [[SPLIT:xxxx]] | Producción |
|---|---|---|---|---|---|
| agencIA | 1 | gpt-4o-mini | Disponible | Disponible | Sin uso activo (candidato para link Cal.com) |
| Uhane SAS | 7 | gpt-4o-mini | Disponible | Disponible | Sin uso activo (candidato para datos pago) |
| PC_Outlet | 8 | gpt-4o-mini | Disponible | Disponible | Sin uso activo (candidato para A/B/C) |
| tienda4030 | 9 | gpt-4o-mini | En uso | En uso (Resina con `[[SPLIT:2500]]` y `[[SPLIT:2000]]`) | **Producción activa** |

**Todos los tenants tienen el sistema disponible arquitectónicamente**. Se aplicará caso a caso cuando el cliente lo solicite o el operador AVE lo determine.

---

## 8. Migración desde v3.1

Para replicar los cambios de v3.2 en otro entorno:

1. **Backup obligatorio:** `modulos_bot`, `bot_config`, workflow n8n
2. **Actualizar NUCLEO** con la sección extendida de `[[SPLIT]]` (~1300 chars agregados)
3. **Actualizar módulo MEDIOS** reemplazando URLs literales por placeholders
4. **Actualizar código de `Code_detectar_media`** con normalización + parseDelay + límite defensivo
5. **Actualizar código de `Code_enviar_burbujas`** para respetar `delay_antes_ms`
6. **Validar retrocompatibilidad**: prompts existentes con `[[SPLIT]]` simple deben seguir funcionando
7. **Aplicar `[[SPLIT:xxxx]]` selectivamente** en tenants según necesidad

---

## 9. Referencias cruzadas

- `docs/troubleshooting/bugs-54-y-55-consolidados.md` — Bugs #54 y #55 completos
- `docs/troubleshooting/bugs-resueltos.md` — Registro completo incluyendo #52
- `docs/features/split-delays-dinamicos.md` — Feature completa del sistema
- `docs/promps/guia-maestra-prompts-ave.md` v2.3 — Guía práctica actualizada
- `docs/troubleshooting/queries-diagnostico.md` v1.2 — Queries actualizadas

---

## 10. Backlog para v3.3+

**Candidatos futuros:**

- Modelo dinámico leyendo `bot_config.modelo_bot` desde el workflow (actualmente hardcoded)
- Typing indicator entre burbujas para mejorar percepción humana
- Delay dinámico proporcional al largo del texto anterior
- Panel Cliente Appsmith expone la configuración de `[[SPLIT]]` con dropdowns
- Módulo global `BURBUJAS` dedicado (separar reglas del NUCLEO en módulo específico)

**Bugs latentes conocidos:**

- Rate limiting de Meta si un tenant hace muchas burbujas por conversación en pico
- No hay buffer de agregación para mensajes rápidos del cliente

---

**Fin del documento arquitectura-prompts-sandwich.md v3.2**
