# Onboarding de Nuevos Clientes + Git

Guía operativa para dar de alta una empresa nueva en AVE bajo la arquitectura sándwich de prompts multi-tenant, y para versionar el proyecto en Git de forma segura.

---

## Parte A — Onboarding de un cliente nuevo (arquitectura sándwich)

Checklist para agregar una empresa al sistema multi-tenant. Ejemplo referencia basado en tienda4030 (empresa_id=9), migrada el 12 jul 2026.

**Duración estimada:** 45-60 minutos para un tenant completo con datos, prompts y validación.

**Documentos relacionados:**
- `docs/modulos/arquitectura-prompts-sandwich.md` — arquitectura de 3 capas + módulos
- `docs/modulos/panel-cliente-appsmith.md` — setup del Panel Cliente
- `docs/troubleshooting/bugs-resueltos.md` bugs 44-49 — errores comunes a evitar

### A.1 Base de datos

1. Crear registro en `empresas` con `chatwoot_account_id` correspondiente. Anotar el `id` retornado (será el `empresa_id` en todos los pasos siguientes).

    ```sql
    INSERT INTO empresas (nombre, plan, email, telefono, chatwoot_account_id, activo)
    VALUES ('Nombre Comercial', 'basic', 'contacto@cliente.com', '3001234567', <CHATWOOT_ACCT_ID>, true)
    RETURNING id;
    ```

2. Crear `bot_config` con configuración base. **NO llenar `system_prompt` viejo**; se usarán `prompt_capa_cliente` y `prompt_capa_reglas_fin` en el paso A.4.

    ```sql
    INSERT INTO bot_config (
        empresa_id, 
        objetivo_bot, 
        tono, 
        modelo_bot, 
        modelo_clasificador,
        temperatura_bot, 
        max_tokens_bot, 
        openai_api_key,
        openai_key_status,
        prompt_clasificador,
        activo
    ) VALUES (
        <EMPRESA_ID>,
        'Placeholder objetivo - editar despues desde Panel Cliente',
        'Placeholder tono',
        'gpt-4o-mini',
        'gpt-4o-mini',
        0.7,
        1500,
        NULL,
        'no_config',
        'Clasifica leads como frio, tibio, caliente segun interes...',
        true
    );
    ```

3. Cargar `servicios` (productos/servicios del cliente).
4. Cargar `info_negocio` (categorías de información: precios, envios, caracteristicas, contacto, etc.).
5. Definir `etiquetas_pipeline` con el campo `es_conversion` marcado en la(s) etiqueta(s) de conversión.
6. Configurar `recordatorio_config` (orden, horas_despues, mensaje, activo).

### A.2 Chatwoot

1. Crear el inbox de WhatsApp Business API en la cuenta del cliente.
2. Generar el token de sistema (System User token) en Meta Business Manager.
3. Crear las etiquetas (labels) que coincidan con `etiquetas_pipeline`.
4. Confirmar que existen los custom attributes a nivel conversación: `estado_lead`, `desactivar_bot`, `en_seguimiento`.

### A.3 Módulos del bot en tablas de arquitectura sándwich

1. Decidir qué módulos activar según las tools que usará el bot:

    | Módulo | Cuándo activar |
    |---|---|
    | `NUCLEO` | **Siempre** (obligatorio para todos) |
    | `AGENDAMIENTO` | Si usa `ver_disponibilidad_calcom` y/o `agendar_cita_calcom` |
    | `MEDIOS` | Si usa `buscar_media_servicio` para enviar imágenes/videos/PDFs |
    | `PEDIDOS` | Si usa `registrar_pedido` para venta directa por WhatsApp |

2. Activar los módulos en `empresa_modulos`:

    ```sql
    -- NUCLEO siempre
    INSERT INTO empresa_modulos (empresa_id, modulo_id, activo)
    VALUES (<EMPRESA_ID>, (SELECT id FROM modulos_bot WHERE codigo = 'NUCLEO'), true)
    ON CONFLICT (empresa_id, modulo_id) DO UPDATE SET activo = true;
    
    -- Otros según aplique
    INSERT INTO empresa_modulos (empresa_id, modulo_id, activo)
    VALUES 
        (<EMPRESA_ID>, (SELECT id FROM modulos_bot WHERE codigo = 'MEDIOS'), true),
        (<EMPRESA_ID>, (SELECT id FROM modulos_bot WHERE codigo = 'PEDIDOS'), true)
    ON CONFLICT (empresa_id, modulo_id) DO UPDATE SET activo = true;
    ```

3. Verificar módulos activos:

    ```sql
    SELECT mb.codigo, mb.orden, em.activo
    FROM empresa_modulos em
    JOIN modulos_bot mb ON mb.id = em.modulo_id
    WHERE em.empresa_id = <EMPRESA_ID>
    ORDER BY mb.orden;
    ```

### A.4 Redactar y persistir Capa 2 (personalidad) y Capa 3 (guardrails)

Ver `docs/modulos/arquitectura-prompts-sandwich.md` sección 5 y 6 para la metodología completa. Aquí el resumen operativo:

1. **Redactar Capa 2 (`prompt_capa_cliente`)**: identidad del bot, saludo textual, tono, flujo comercial, argumentos de venta, preguntas frecuentes, casos especiales. Ver ejemplos de longitud típica: 4,000-15,000 chars según complejidad del negocio.

2. **Redactar Capa 3 (`prompt_capa_reglas_fin`)**: guardrails de seguridad. Longitud típica: 1,000-2,500 chars. **Reglas críticas de redacción**:
   - NO mencionar "NUCLEO", "reglas del sistema", "capas anteriores" (ver Bug 45)
   - Usar imperativo directo: `"NUNCA hagas X"`, `"SIEMPRE responde Y"`
   - Si el prompt total supera 10,000 chars, incluir regla anti-fuga como PRIMERA sección
   - Reglas anti-captura de nombre errónea si el bot captura nombre del cliente (ver Bug 46)

3. **Backup previo obligatorio**:

    ```sql
    CREATE TABLE IF NOT EXISTS backup_bot_config_<TENANT>_YYYYMMDD AS
    SELECT * FROM bot_config WHERE empresa_id = <EMPRESA_ID>;
    ```

4. **UPDATE con delimitador `$BODY$`** (nunca `$$` — ver Bug 44):

    ```sql
    UPDATE bot_config
    SET 
        prompt_capa_cliente = $BODY$<CONTENIDO_CAPA_2>$BODY$,
        prompt_capa_reglas_fin = $BODY$<CONTENIDO_CAPA_3>$BODY$,
        updated_at = NOW()
    WHERE empresa_id = <EMPRESA_ID>;
    ```

5. **Verificación obligatoria con LENGTH** (para detectar bug de delimitador):

    ```sql
    SELECT 
        LENGTH(prompt_capa_cliente) AS chars_capa_2,
        LENGTH(prompt_capa_reglas_fin) AS chars_capa_3,
        LEFT(prompt_capa_cliente, 100) AS inicio_capa_2,
        RIGHT(prompt_capa_cliente, 100) AS final_capa_2,
        LEFT(prompt_capa_reglas_fin, 100) AS inicio_capa_3,
        RIGHT(prompt_capa_reglas_fin, 100) AS final_capa_3
    FROM bot_config
    WHERE empresa_id = <EMPRESA_ID>;
    ```

    Si algún `chars_capa_*` es visiblemente menor a lo redactado (ej: 75 chars cuando se esperaba 1,500), hay bug de delimitador. Hacer rollback y reintentar.

### A.5 Configurar acceso al Panel Cliente Appsmith

1. Registrar el email del cliente en `usuarios_cliente`:

    ```sql
    INSERT INTO usuarios_cliente (email, empresa_id, nombre, rol, activo)
    VALUES ('cliente@empresa.com', <EMPRESA_ID>, 'Nombre Contacto', 'admin_cliente', true);
    ```

2. Verificar que la vista `v_usuario_empresa` devuelve UNA sola fila:

    ```sql
    SELECT * FROM v_usuario_empresa WHERE email = 'cliente@empresa.com';
    ```

    Si devuelve más de una fila, revisar Bug 47 en `bugs-resueltos.md`.

3. Crear workspace en Appsmith con el nombre del tenant (ej: `MI-EMPRESA`).
4. Forkear la app "Panel Cliente" plantilla desde `enuar's apps` al workspace nuevo.
5. **Auditar TODAS las queries del fork** buscando valores hardcodeados (ver Bugs 48 y 49):
   - Filtros literales tipo `WHERE empresa_id = 1`
   - Emails literales tipo `WHERE email = 'admin@algo.com'`
   - Reemplazar por expresiones dinámicas:
     - `WHERE empresa_id = {{q_get_usuario_actual.data[0].empresa_id}}`
     - `WHERE email = '{{appsmith.user.email}}'`
6. Invitar al cliente en el workspace con rol **App Viewer** (nunca Administrator ni Developer).

### A.6 n8n

1. Confirmar que el webhook de Chatwoot apunta al workflow principal.
2. Activar el workflow de recordatorios (es genérico, ya incluye la empresa nueva automáticamente al estar `activo = true`).

### A.7 Prueba end-to-end

1. Escribir por WhatsApp desde un número de prueba y validar:
    - Saludo del bot con la personalidad correcta del tenant.
    - **Sin fugas de razonamiento en inglés** (ver Bug 45).
    - **Sin extracción errónea de fragmentos como nombre** (ver Bug 46).
2. Simular flujo comercial completo del negocio (pregunta por precio, cotización, cierre, o transferencia según aplique).
3. Validar clasificación de etiquetas.
4. Validar apagado de `en_seguimiento` al convertir.
5. Validar recepción de recordatorios.
6. Desde el navegador del cliente (o incógnito), acceder a `admin.identechnology.co`:
    - Login con el email registrado.
    - Verificar que solo ve el workspace de su tenant.
    - Abrir el Panel Cliente y confirmar que todos los tabs muestran datos correctos (Dashboard, Mi Empresa, Info Negocio, Servicios, Config Bot, Config Recordatorios).

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
│   ├── infraestructura/
│   │   └── conexion-dbeaver.md
│   ├── modulos/
│   │   ├── calcom-instalacion.md
│   │   ├── desactivar-bot.md
│   │   ├── etiquetas-seguimiento-prioridad.md
│   │   ├── notificaciones-openai-key.md
│   │   ├── openai-multitenant.md
│   │   ├── panel-cliente-appsmith.md
│   │   ├── arquitectura-prompts-sandwich.md
│   │   ├── recordatorios.md
│   │   └── seguimiento.md
│   ├── onboarding-clientes/
│   │   ├── onboarding-clientes.md
│   │   └── whatsapp-meta-setup.md
│   ├── promps/
│   │   └── guia-maestra-prompts-ave.md
│   └── troubleshooting/
│       ├── bugs-resueltos.md
│       ├── onboarding-clientes.md
│       └── tipos-de-etiquetas.md
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

Ejemplos reales:
- `fix: en_seguimiento se borraba en cada turno por reemplazo de custom_attributes`
- `fix: error de memoria por entrada ambigua al AI Agent`
- `feat: sistema generico de seguimiento con en_seguimiento_activo`
- `fix: timezone en comparacion de ultimo_recordatorio_at (columna with time zone)`
- `feat: arquitectura sandwich de prompts con modulos_bot y empresa_modulos`
- `fix: v_usuario_empresa con UNION ALL para precedencia de usuarios_cliente`
- `docs: bugs 44-49 sobre delimitador $BODY$ y fugas de razonamiento del LLM`

---

## Parte C — Estado actual de los tenants

Al 12 de julio de 2026, los 4 tenants están migrados a la arquitectura sándwich:

| Tenant | empresa_id | Módulos activos | Capa 2 chars | Capa 3 chars | Estado |
|---|---|---|---|---|---|
| agencIA | 1 | NUCLEO + AGENDAMIENTO + MEDIOS | 5,489 | 2,058 | Producción |
| Uhane SAS | 7 | NUCLEO + MEDIOS + PEDIDOS | 4,176 | 1,178 | Producción, $11.4M COP en pedidos reales |
| PC_Outlet | 8 | NUCLEO + MEDIOS | 13,883 | 2,452 | Producción, modelo cierre humano |
| tienda4030 | 9 | NUCLEO + MEDIOS + PEDIDOS | 6,686 | 1,487 | Producción |

Backups preservados en tablas `backup_bot_config_<tenant>_20260712` y `system_prompt` viejo intacto en cada tenant como respaldo permanente.

---

## Parte D — Checklist final de verificación post-onboarding

Antes de considerar un tenant como "listo para producción", validar todos los ítems:

### D.1 Base de datos
- [ ] Registro en `empresas` con `chatwoot_account_id` correcto
- [ ] Registro en `bot_config` con `activo = true`
- [ ] `LENGTH(prompt_capa_cliente)` coincide con lo redactado
- [ ] `LENGTH(prompt_capa_reglas_fin)` coincide con lo redactado
- [ ] `LEFT()` y `RIGHT()` de las capas devuelven texto esperado (no SQL basura — Bug 44)
- [ ] Módulos correctos activos en `empresa_modulos`
- [ ] `servicios` y `info_negocio` cargados
- [ ] `etiquetas_pipeline` con `es_conversion` correcto
- [ ] `recordatorio_config` configurado y activo
- [ ] Backup de la sesión respaldado en tabla `backup_bot_config_<tenant>_YYYYMMDD`

### D.2 Chatwoot
- [ ] Inbox WhatsApp activo con token válido
- [ ] Labels creados y sincronizados con `etiquetas_pipeline`
- [ ] Custom attributes de conversación existentes

### D.3 Appsmith Panel Cliente
- [ ] Usuario del cliente registrado en `usuarios_cliente`
- [ ] `v_usuario_empresa` devuelve UNA sola fila para ese email (no duplicados — Bug 47)
- [ ] Workspace del tenant creado
- [ ] App forkeada al workspace nuevo
- [ ] Auditoría de queries completada (sin valores hardcodeados — Bugs 48-49)
- [ ] Cliente invitado con rol **App Viewer**
- [ ] Correo de invitación recibido por el cliente

### D.4 Prueba end-to-end
- [ ] Saludo del bot con personalidad correcta
- [ ] Consulta a `info_negocio_pg` antes de responder precios
- [ ] Sin fugas de razonamiento en inglés (Bug 45)
- [ ] Sin captura errónea de nombre del cliente (Bug 46)
- [ ] Flujo comercial completo funciona (venta / agendamiento / transferencia)
- [ ] Cliente accede a Panel Cliente y ve datos correctos en todos los tabs
- [ ] Sistema de recordatorios activo y enviando correctamente

Si todos los ítems están marcados, el tenant queda oficialmente en producción.
