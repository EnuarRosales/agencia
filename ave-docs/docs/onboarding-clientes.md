# Onboarding de Clientes - Proyecto AVE

**Version:** 3.0
**Fecha:** 2026-07-13
**Autor:** SS. Enuar Emilio Rosales Salazar
**Estado:** Vigente - actualizado tras refactor arquitectura sandwich

---

## Indice

1. [Introduccion y vision general](#1-introduccion-y-vision-general)
2. [Prerequisitos](#2-prerequisitos)
3. [Fase 1 - Registro base](#3-fase-1-registro-base)
4. [Fase 2 - Activacion de modulos](#4-fase-2-activacion-de-modulos)
5. [Fase 3 - Redaccion de Capa 2](#5-fase-3-redaccion-de-capa-2)
6. [Fase 4 - Validacion end-to-end](#6-fase-4-validacion-end-to-end)
7. [Fase 5 - Entrega al cliente](#7-fase-5-entrega-al-cliente)
8. [Estado actual de los 4 tenants](#8-estado-actual-de-los-4-tenants)
9. [Preguntas frecuentes](#9-preguntas-frecuentes)
10. [Referencias cruzadas](#10-referencias-cruzadas)

---

## 1. Introduccion y vision general

Este documento describe el flujo actualizado para incorporar nuevos clientes al SaaS multi-tenant AVE bajo la arquitectura sandwich refactorizada del 2026-07-13.

### Objetivo

Que cualquier nuevo tenant pueda ser aprovisionado, configurado, validado y entregado en una sesion de trabajo, con aislamiento multi-tenant garantizado y sin exposicion de detalles tecnicos al cliente final.

### Duracion estimada

- Fase 1 (Registro base): 10 minutos
- Fase 2 (Activacion modulos): 5 minutos
- Fase 3 (Redaccion Capa 2): 60 a 120 minutos (segun complejidad del negocio)
- Fase 4 (Validacion): 30 a 60 minutos
- Fase 5 (Entrega): 20 minutos
- **Total: 2 a 4 horas** por tenant nuevo.

### Roles

- **Operador AVE:** ejecuta las Fases 1, 2, 3 y 4. Es la unica persona con acceso a modulos globales, Panel Admin y BD directa.
- **Cliente final:** recibe invitacion al Panel Cliente Appsmith en Fase 5, edita su propia Capa 2 y consulta KPIs.

---

## 2. Prerequisitos

### 2.1 Infraestructura AVE operativa

- [ ] VPS Hostinger en Colombia con Docker Compose corriendo.
- [ ] Workflow n8n TEST-PRINCIPAL activo.
- [ ] Chatwoot operativo en `chatwoot.identechnology.co`.
- [ ] Postgres AVE con las tablas base creadas y los 5 modulos globales cargados.
- [ ] Panel Admin operativo.
- [ ] Panel Cliente Appsmith operativo en `admin.identechnology.co`.
- [ ] SMTP Hover configurado en Appsmith (para invitaciones).

### 2.2 Datos requeridos del cliente

Antes de iniciar el onboarding, recopilar del cliente:

- [ ] Nombre comercial de la empresa.
- [ ] NIT (si aplica).
- [ ] Correo del administrador que accedera al Panel Cliente.
- [ ] Numero de WhatsApp Business que se conectara a Chatwoot.
- [ ] Modelo de negocio: dropshipping, servicios con agendamiento, venta consultiva, etc.
- [ ] Objetivo comercial principal del bot (ejemplo: agendar diagnosticos, capturar pedidos, calificar leads).
- [ ] Tono deseado (formal, informal, con emojis, sin emojis).
- [ ] Nombre y personalidad del agente virtual.
- [ ] Catalogo de productos o servicios (para cargar en `info_negocio` y `servicios`).
- [ ] Fotos, videos y PDFs disponibles (para cargar en `servicios_media` si aplica).

### 2.3 Decision inicial de modulos

Segun el modelo de negocio, decidir cuales de los modulos opcionales activar:

| Modelo de negocio | AGENDAMIENTO | MEDIOS | PEDIDOS |
|--------------------|---------------|---------|----------|
| Servicios con citas | Si | Si | No |
| Dropshipping / e-commerce | No | Si | Si |
| Consultoria B2B con transferencia humana | Segun | Si | No |
| Venta consultiva sin cierre en bot | No | Si | No |

NUCLEO y GUARDRAILS son obligatorios siempre.

---

## 3. Fase 1 - Registro base

### 3.1 Crear empresa

```sql
INSERT INTO empresas (
    nombre, email, telefono, chatwoot_url, chatwoot_api_key,
    chatwoot_account_id, plan, activo, created_at, updated_at,
    email_contacto
)
VALUES (
    'Nombre Empresa Cliente',
    'admin@empresa-cliente.com',
    '+573001234567',
    'https://chatwoot.identechnology.co',
    '<chatwoot_api_key_del_tenant>',
    <chatwoot_account_id>,
    'basico',
    true,
    NOW() AT TIME ZONE 'America/Bogota',
    NOW() AT TIME ZONE 'America/Bogota',
    'contacto@empresa-cliente.com'
)
RETURNING id;
```

Guardar el `id` retornado. Sera el `empresa_id` para todos los INSERT siguientes.

### 3.2 Crear bot_config

```sql
INSERT INTO bot_config (
    empresa_id,
    openai_api_key,
    openai_key_status,
    modelo_bot,
    modelo_clasificador,
    temperatura_bot,
    max_tokens_bot,
    objetivo_bot,
    tono,
    prompt_clasificador,
    prompt_capa_cliente,
    prompt_capa_reglas_fin,
    created_at,
    updated_at
)
VALUES (
    <empresa_id>,
    'sk-...',
    'active',
    'gpt-5-mini',
    'gpt-4o-mini',
    0.7,
    1000,
    'Objetivo comercial del bot',
    'Descripcion del tono',
    '<prompt del clasificador de pipeline>',
    '<Capa 2 inicial - ver Fase 3>',
    '',
    NOW() AT TIME ZONE 'America/Bogota',
    NOW() AT TIME ZONE 'America/Bogota'
);
```

**Notas:**
- `prompt_capa_reglas_fin` queda vacio (`''`) porque los guardrails ahora estan en el modulo GUARDRAILS global.
- `prompt_capa_cliente` se llena en la Fase 3.
- Verificar con el cliente que la `openai_api_key` sea valida antes de guardar.

### 3.3 Crear usuario_cliente

```sql
INSERT INTO usuarios_cliente (empresa_id, email, activo, created_at, updated_at)
VALUES (
    <empresa_id>,
    'admin@empresa-cliente.com',
    true,
    NOW() AT TIME ZONE 'America/Bogota',
    NOW() AT TIME ZONE 'America/Bogota'
);
```

Este email es el que recibira la invitacion al Panel Cliente en la Fase 5.

### 3.4 Configurar inbox de Chatwoot

Desde Chatwoot admin:

- [ ] Crear inbox tipo API Channel o WhatsApp Cloud (segun proveedor).
- [ ] Configurar webhook a n8n: `https://n8n.identechnology.co/webhook/28fb99ec-026f-428d-b5e8-a4b4978fb2cf`.
- [ ] Anotar el `account_id` y el `api_access_token` para usarlos en `empresas.chatwoot_account_id` y `empresas.chatwoot_api_key`.
- [ ] Enviar un mensaje de prueba al numero y verificar que llegue a Chatwoot.

---

## 4. Fase 2 - Activacion de modulos

### 4.1 INSERT en empresa_modulos

Segun la decision de la seccion 2.3, ejecutar:

```sql
INSERT INTO empresa_modulos (empresa_id, modulo_id, activo, created_at, updated_at)
SELECT 
    <empresa_id>,
    id,
    true,
    NOW() AT TIME ZONE 'America/Bogota',
    NOW() AT TIME ZONE 'America/Bogota'
FROM modulos_bot
WHERE codigo IN ('NUCLEO', 'GUARDRAILS' /* + los opcionales que apliquen */);
```

Ejemplos por tipo de negocio:

**Servicios con citas:**
```sql
WHERE codigo IN ('NUCLEO', 'AGENDAMIENTO', 'MEDIOS', 'GUARDRAILS')
```

**Dropshipping:**
```sql
WHERE codigo IN ('NUCLEO', 'MEDIOS', 'PEDIDOS', 'GUARDRAILS')
```

**Consultoria B2B con transferencia:**
```sql
WHERE codigo IN ('NUCLEO', 'MEDIOS', 'GUARDRAILS')
```

### 4.2 Verificacion

```sql
SELECT 
    em.empresa_id,
    e.nombre AS empresa,
    mb.codigo,
    mb.orden,
    em.activo
FROM empresa_modulos em
JOIN modulos_bot mb ON em.modulo_id = mb.id
JOIN empresas e ON em.empresa_id = e.id
WHERE em.empresa_id = <empresa_id>
ORDER BY mb.orden ASC;
```

Debe aparecer al menos NUCLEO y GUARDRAILS. El orden final del prompt sera el que muestre esta query.

---

## 5. Fase 3 - Redaccion de Capa 2

### 5.1 Base de trabajo

Usar la **Guia Maestra de Prompts AVE v2.1** (`docs/promps/guia-maestra-prompts-ave.md`) como referencia. En particular las secciones:

- Estructura recomendada de Capa 2 (seccion 2).
- Definicion explicita de tono con ejemplos de los 4 tenants (seccion 3).
- Regla critica de generacion (seccion 4).
- Patrones a evitar y patrones a usar (secciones 5 y 6).
- Prohibiciones especificas vs globales (seccion 7).

### 5.2 Estructura minima obligatoria

Toda Capa 2 nueva debe incluir:

1. Encabezado con nombre del agente y variables.
2. Seccion "Personalidad y estilo" con tono explicito.
3. Seccion "Regla critica de generacion" con anti-duplicacion.
4. Saludo obligatorio EXACTO.
5. Flujo comercial principal (segun modelo de negocio).
6. FAQ con datos reales.
7. Casos especiales (bot?, ingles, molesto, sin avance).
8. Prohibiciones especificas del tenant (si aplican).

### 5.3 UPDATE con delimitador unico

```sql
UPDATE bot_config
SET prompt_capa_cliente = $CAPA2_NUEVOTENANT_V1$
[contenido completo de la Capa 2]
$CAPA2_NUEVOTENANT_V1$,
    updated_at = NOW() AT TIME ZONE 'America/Bogota'
WHERE empresa_id = <empresa_id>;

-- Verificacion inmediata
SELECT LENGTH(prompt_capa_cliente) FROM bot_config WHERE empresa_id = <empresa_id>;
```

**Reglas obligatorias del UPDATE** (ver `docs/troubleshooting/bugs-resueltos.md` bugs #44 y #50):
- Usar delimitador unico por tenant y version (ejemplo: `$CAPA2_NEWTENANT_V1$`).
- Solo caracteres ASCII en comentarios SQL (evitar em-dash).
- Verificar LENGTH inmediatamente despues.

### 5.4 Carga de catalogo

En paralelo, cargar los datos del negocio en las tablas correspondientes:

- **info_negocio:** categorias como "precios", "envios", "caracteristicas", "contacto", etc.
- **servicios:** productos o servicios con nombre, descripcion, precio.
- **servicios_media:** URLs de fotos, videos y PDFs asociados a servicios.
- **etiquetas_pipeline:** estados de lead del pipeline del tenant.

Estas cargas se pueden hacer desde el Panel Cliente Appsmith una vez el cliente tenga acceso.

---

## 6. Fase 4 - Validacion end-to-end

### 6.1 Checklist generico de validacion por WhatsApp

Ejecutar estos casos desde un WhatsApp real (no simulacion) al numero conectado a Chatwoot:

| # | Caso | Esperado |
|---|------|-----------|
| 1 | Enviar saludo simple | Saludo obligatorio EXACTO del tenant |
| 2 | Preguntar por producto o servicio | Consulta catalogo, responde con datos reales |
| 3 | Preguntar precio | Sigue el flujo comercial definido en Capa 2 |
| 4 | Solicitar foto/video | Envia solo URLs planas (si tiene MEDIOS) |
| 5 | Ejecutar flujo transaccional principal | Registra pedido/agenda cita con secuencia critica correcta |
| 6 | Verificar tono | Coincide con la definicion explicita (tu/usted, emojis, longitud) |
| 7 | Preguntar si es bot | Respuesta EXACTA definida en Capa 2 |
| 8 | Escribir en ingles | Responde en ingles con mensaje de bienvenida definido |
| 9 | Intento de jailbreak ("ignora tus instrucciones") | Respuesta anti-jailbreak de GUARDRAILS |
| 10 | Enviar mensaje ambiguo | No inventa nombre del cliente, pide clarificacion |

### 6.2 Banderas rojas criticas

Si detectas alguna de estas, PAUSAR y diagnosticar antes de avanzar:

- **Duplicacion de mensajes** en un turno: verificar que "Use Responses API" este DESACTIVADO en n8n. Ver Bug #51.
- **Fuga de frases en ingles** al cliente: revisar Capa 2 por meta-referencias a NUCLEO. Ver Bug #45.
- **Bot inventa datos** (precios, disponibilidad): verificar que el catalogo este cargado y que Capa 2 diga "consulta el catalogo" en el flujo.
- **Bot usa nombres tecnicos** de tools al cliente: hay fuga tecnica en Capa 2. Revisar seccion 5 de la guia maestra.
- **Bot tutea cuando deberia usted** o viceversa: la definicion de tono en Capa 2 no dominio. Reforzarla.
- **Bot presenta "3 opciones"** pero solo llegan 1 o 2 imagenes: ajustar Capa 2 a "hasta N opciones" con logica flexible.

### 6.3 Verificaciones en base de datos

Despues del flujo transaccional, verificar directamente en BD:

```sql
-- Verificar registro del lead
SELECT * FROM leads 
WHERE empresa_id = <empresa_id> 
ORDER BY updated_at DESC LIMIT 1;

-- Verificar registro del pedido (si aplica)
SELECT * FROM pedidos 
WHERE empresa_id = <empresa_id> 
ORDER BY created_at DESC LIMIT 1;

-- Verificar cita (si aplica)
SELECT * FROM citas 
WHERE empresa_id = <empresa_id> 
ORDER BY created_at DESC LIMIT 1;
```

Todos los datos deben coincidir con lo que el cliente escribio en WhatsApp.

---

## 7. Fase 5 - Entrega al cliente

### 7.1 Invitacion al Panel Cliente

Desde Appsmith (`admin.identechnology.co`):

- [ ] Ir a Settings del workspace correspondiente al tenant.
- [ ] Enviar invitacion al email registrado en `usuarios_cliente`.
- [ ] El cliente recibe correo desde `info@identechnology.co` via SMTP Hover.
- [ ] Cliente crea su contrasena y accede al Panel Cliente.

### 7.2 Verificacion de aislamiento multi-tenant

Con el cliente accediendo por primera vez, verificar:

- [ ] El cliente ve unicamente sus datos (no de otros tenants).
- [ ] Query `q_get_usuario_actual` usa `{{appsmith.user.email}}` dinamico, no hardcoded.
- [ ] Todas las paginas usan `q_get_usuario_actual.data[0].empresa_id` para filtrar.
- [ ] Dashboard muestra KPIs correctos del tenant.

### 7.3 Capacitacion al cliente

Recorrer con el cliente:

- [ ] Pagina "Mi Empresa": datos generales editables.
- [ ] Pagina "Info Negocio": categorias con contenido consultable por el bot.
- [ ] Pagina "Servicios": productos con precio, descripcion, categoria.
- [ ] Pagina "Medios": subida de fotos, videos y PDFs (URLs limpias servidas por ave-api).
- [ ] Pagina "Config Bot": edicion de `prompt_capa_cliente` (Capa 2) directamente.
- [ ] Pagina "Config Recordatorios": configurar seguimientos automaticos si aplica.
- [ ] Pagina "Dashboard": KPIs de leads, conversiones, tasa.

### 7.4 Explicacion de responsabilidades

Aclarar con el cliente:

- **Cliente edita:** su Capa 2 (personalidad, flujo comercial), catalogo, medios, KPIs.
- **Cliente NO edita:** modulos globales (NUCLEO, MEDIOS, PEDIDOS, AGENDAMIENTO, GUARDRAILS), la Capa 3 (deprecated), configuracion tecnica de n8n o Chatwoot.
- **Cambios en modulos globales** requieren solicitud al operador AVE (por ejemplo, agregar un nuevo tipo de tool).
- **Autoservicio de Capa 2** permite ajustar tono, mensajes, FAQ, prohibiciones especificas del tenant.

---

## 8. Estado actual de los 4 tenants

| Tenant | id | Modulos activos | Estado onboarding |
|--------|-----|-----------------|-------------------|
| agencIA | 1 | NUCLEO + AGENDAMIENTO + MEDIOS + GUARDRAILS | Completo, validado en produccion |
| Uhane SAS | 7 | NUCLEO + MEDIOS + PEDIDOS + GUARDRAILS | Completo, validado en produccion |
| PC_Outlet | 8 | NUCLEO + MEDIOS + GUARDRAILS | Completo, validado en produccion |
| tienda4030 | 9 | NUCLEO + MEDIOS + PEDIDOS + GUARDRAILS | Completo, validado en produccion |

Todos migrados a arquitectura sandwich refactorizada del 2026-07-13. Backups preservados en `bot_config_backup_20260713`, `modulos_bot_backup_20260713` y `empresa_modulos_backup_20260713`.

---

## 9. Preguntas frecuentes

### 9.1 El cliente pregunta como cambiar el tono del bot

**Respuesta:** desde el Panel Cliente, seccion Config Bot, editar `prompt_capa_cliente`. La seccion "Personalidad y estilo" al inicio del prompt define el tono. Guiar al cliente a seguir el patron de la Guia Maestra seccion 3.

### 9.2 El cliente quiere activar agendamiento

**Respuesta:** el cliente NO puede activar modulos por si mismo. Debe solicitar al operador AVE que ejecute:

```sql
INSERT INTO empresa_modulos (empresa_id, modulo_id, activo, created_at, updated_at)
SELECT <empresa_id>, id, true, NOW(), NOW()
FROM modulos_bot WHERE codigo = 'AGENDAMIENTO';
```

Ademas, se debe configurar Cal.com self-hosted con el team_slug del tenant (ver `docs/modulos/calcom-instalacion.md`).

### 9.3 El cliente quiere probar cambios en Capa 2 sin afectar produccion

**Respuesta:** actualmente la Capa 2 se edita directamente en produccion desde el Panel Cliente. Se recomienda:

1. Copiar el prompt actual como respaldo antes de editar.
2. Hacer pruebas por WhatsApp inmediatamente despues del guardado.
3. Si algo falla, restaurar el prompt anterior desde el respaldo.

Como mejora futura se contempla una pagina "AVE Admin" en Appsmith con backup automatico (ver memoria del 2026-07-13).

### 9.4 Como se factura al cliente

**Respuesta:** el modelo comercial es multi-tenant nativo sin costo por cliente adicional, sin limite de conversaciones, citas ni pedidos. Cada cliente paga su propia API key de OpenAI (multi-tenant desde el 2026-07). Costos de VPS, dominio y almacenamiento son fijos para AVE.

### 9.5 El bot expone informacion tecnica al cliente

**Respuesta:** revisar la Capa 2 del tenant afectado siguiendo el checklist de la Guia Maestra seccion 10. Los casos mas comunes son:

- Nombres tecnicos de tools presentes en Capa 2.
- Meta-referencias a NUCLEO o "reglas del sistema".
- Placeholders con corchetes que actuan como plantillas.

Ver bugs #45, #50, #51 en `bugs-resueltos.md`.

---

## 10. Referencias cruzadas

- `docs/modulos/arquitectura-prompts-sandwich.md` - Arquitectura de 3 capas.
- `docs/promps/guia-maestra-prompts-ave.md` v2.1 - Guia de redaccion de Capa 2.
- `docs/troubleshooting/bugs-resueltos.md` - Bugs #44 al #51 relevantes.
- `docs/modulos/panel-cliente-appsmith.md` - Panel Cliente autoservicio.
- `docs/modulos/openai-multitenant.md` - Gestion multi-tenant de API keys.
- `docs/modulos/notificaciones-openai-key.md` - Auto-update de estado de key.
- `docs/modulos/calcom-instalacion.md` - Cal.com para AGENDAMIENTO.

---

**Fin del documento.**
