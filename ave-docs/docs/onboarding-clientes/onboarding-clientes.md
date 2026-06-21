# Onboarding de Nuevos Clientes + Git

Guía operativa para dar de alta una empresa nueva en AVE y para versionar el proyecto en Git de forma segura.

---

## Parte A — Onboarding de un cliente nuevo

Checklist para agregar una empresa al sistema multi-tenant. Ejemplo basado en PC Outlet (empresa_id=8).

### A.1 Base de datos

1. Crear registro en `empresas` con `chatwoot_account_id` correspondiente.
2. Crear `bot_config` con: `system_prompt`, `objetivo_bot`, `tono`, `prompt_clasificador`, `openai_api_key`, `modelo_bot`.
3. Cargar `servicios` (productos/servicios del cliente).
4. Cargar `info_negocio` (categorías de información: ubicación, pagos, envíos, garantía, etc.).
5. Definir `etiquetas_pipeline` con el campo `es_conversion` marcado en la(s) etiqueta(s) de conversión.
6. Configurar `recordatorio_config` (orden, horas_despues, mensaje, activo).

### A.2 Chatwoot

1. Crear el inbox de WhatsApp Business API en la cuenta del cliente y generar el token de sistema (System User token) en Meta Business Manager — ver la guía paso a paso completa, con errores frecuentes y checklist: [Guía de integración WhatsApp Business API + Chatwoot](onboarding-clientes/whatsapp-meta-setup.md).
2. Crear las etiquetas (labels) que coincidan con `etiquetas_pipeline`.
3. Confirmar que existen los custom attributes a nivel conversación: `estado_lead`, `desactivar_bot`, `en_seguimiento`.

### A.3 Appsmith

1. Pegar `system_prompt`, `objetivo_bot`, `tono`, `prompt_clasificador`.
2. Verificar que la columna `tono` sea tipo `text` (no `varchar`) — textos largos no caben en varchar.
3. Cargar imágenes de productos (`imagen_url` en `servicios`).

### A.4 n8n

1. Confirmar que el webhook de Chatwoot apunta al workflow principal.
2. Activar el workflow de recordatorios (es genérico, ya incluye la empresa nueva automáticamente al estar `activo = true`).

### A.5 Prueba end-to-end

1. Escribir por WhatsApp y validar saludo del agente.
2. Validar clasificación de etiquetas.
3. Validar apagado de `en_seguimiento` al convertir.
4. Validar recepción de recordatorios.

---

## Parte B — Versionar AVE en Git de forma segura

### B.1 Estructura recomendada del repositorio

```
ave-plataforma/
├── README.md
├── .gitignore
├── docs/
│   ├── arquitectura.md
│   ├── base-de-datos/
│   │   ├── esquema.md
│   │   └── reglas-timezone.md
│   ├── modulos/
│   │   ├── recordatorios.md
│   │   └── seguimiento.md
│   ├── troubleshooting/
│   │   └── bugs-resueltos.md
│   └── onboarding-clientes.md
├── workflows/
│   ├── Bot_Agencia_final.clean.json      ← SIN credenciales
│   └── AVE_Recordatorios.clean.json      ← SIN credenciales
└── sql/
    └── migraciones/
        ├── 001_normalizar_empresas.sql
        ├── 002_en_seguimiento_activo.sql
        └── ...
```

### B.2 ⚠️ Limpieza de workflows para Git

Los exports de n8n contienen **credenciales en texto plano**. Antes de versionar cualquier `.json`:

**Datos sensibles que aparecen en los JSON:**
- `openai_api_key` (en `PG_get_empresa`, `OpenAI_generar`, etc.)
- `chatwoot_api_key`
- IDs de credenciales de n8n

**Procedimiento de limpieza:**

Opción 1 — Reemplazar valores por placeholders antes de commitear:
```bash
# Ejemplo con sed (ajustar patrones segun tu JSON)
sed -E 's/"sk-proj-[A-Za-z0-9_-]+"/"REDACTED_OPENAI_KEY"/g' Bot_Agencia_final.json > Bot_Agencia_final.clean.json
```

Opción 2 — Los tokens reales viven en las **credenciales de n8n** (no en el JSON del nodo), así que al exportar, n8n normalmente solo deja el ID de credencial, no el secreto. Pero los `api_key` que están como **datos en queries SQL** (ej. el SELECT de `PG_get_empresa` trae `chatwoot_api_key` de la BD) NO son secretos en el JSON — son nombres de columna. El riesgo real está en valores hardcodeados dentro de expresiones o headers.

**Regla de oro:** antes de cada commit, hacer `git diff` y buscar visualmente cualquier `sk-`, `Bearer`, token largo, o password. El `.gitignore` incluido bloquea patrones comunes, pero la revisión manual es la última línea de defensa.

### B.3 Flujo de trabajo Git sugerido

```bash
# Primera vez
cd ave-plataforma
git init
git add .
git commit -m "docs: estructura inicial de documentacion AVE"
git remote add origin git@github.com:tu-usuario/ave-plataforma.git
git push -u origin main

# Cada cambio posterior
git add docs/modulos/recordatorios.md
git commit -m "docs: agregar troubleshooting de timezone en recordatorios"
git push
```

### B.4 Convención de mensajes de commit

| Prefijo | Uso |
|---|---|
| `feat:` | Nueva funcionalidad o módulo |
| `fix:` | Corrección de un bug |
| `docs:` | Solo cambios de documentación |
| `refactor:` | Reestructuración sin cambiar comportamiento |
| `chore:` | Mantenimiento, limpieza, configuración |

Ejemplos reales de esta sesión:
- `fix: en_seguimiento se borraba en cada turno por reemplazo de custom_attributes`
- `fix: error de memoria por entrada ambigua al AI Agent`
- `feat: sistema generico de seguimiento con en_seguimiento_activo`
- `fix: timezone en comparacion de ultimo_recordatorio_at (columna with time zone)`

---

## Parte C — Pendientes documentados

| Pendiente | Prioridad | Detalle |
|---|---|---|
| Sincronizar tool `desactivar_bot` con Postgres | Media | La tool del agente solo escribe Chatwoot. Opción A: leer custom attribute real cada turno y reflejar en Postgres. |
| Auditar timezone en otras tablas | Media | Revisar `leads`, `citas`, `eventos_lead` por uso de `NOW()` sin `AT TIME ZONE`. |
| Aislamiento multi-tenant de `session_id` | Baja | El `session_id` de memoria se basa solo en `conversation_id`; podría colisionar entre empresas. Verificar generación antes de cambiar. |
| Completar onboarding PC Outlet | Alta | Inbox WhatsApp, token, prompts en Appsmith, imágenes, prueba e2e. |
