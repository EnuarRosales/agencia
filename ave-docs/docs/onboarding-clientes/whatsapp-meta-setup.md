# Guía de Conexión WhatsApp Business API + Chatwoot — AVE
*Versión 2.0 — actualizada jun 2026 con lecciones de Uhane y PC Outlet*

---

## ⚠️ Orden crítico de configuración

El orden importa. Si se intenta registrar el número antes de crear el inbox en Chatwoot, el registro falla con error genérico sin explicación clara.

### Orden correcto (probado y validado):

1. Crear portfolio empresarial en Meta (si no existe)
2. Crear app bajo ese portfolio
3. Crear usuario del sistema en Business Manager
4. Asignar la app al usuario del sistema
5. Generar token permanente
6. Agregar método de pago en Meta
7. **Crear inbox en Chatwoot PRIMERO** ← crítico
8. Configurar webhook en Meta con URL y token de Chatwoot
9. **Activar campo `messages`** en Campos de webhook ← crítico
10. Registrar el número en Meta → Paso 2

---

## PASO 1 — Portfolio empresarial en Meta

`business.facebook.com` → Configuración → crear o verificar portfolio.

Cada cliente debe tener su propio portfolio. La app debe crearse **bajo el portfolio del cliente**, no bajo una cuenta personal.

---

## PASO 2 — Crear app en Meta for Developers

`developers.facebook.com` → Mis aplicaciones → Crear app

- Caso de uso: **"Conectarte con los clientes a través de WhatsApp"**
- **Portfolio:** seleccionar el portfolio del cliente ← paso crítico
- Nombre: `{cliente}-wa` (ej: `pcoutlet-wa`, `uhane-wa`)
- Correo: correo del cliente

---

## PASO 3 — Crear usuario del sistema

`business.facebook.com` → portfolio del cliente → Configuración → Usuarios → Usuarios del sistema → **+ Añadir**

- Nombre: `system-user`
- Rol: **Administrador**

---

## PASO 4 — Asignar la app al usuario del sistema

⚠️ Este paso es el que más confunde. El camino correcto es:

`Usuarios del sistema` → selecciona el usuario → clic en **tres puntos `...`** → **"Asignar activos"** → categoría **"Aplicaciones"** → selecciona la app del cliente → activa **"Administrar la aplicación" (Acceso total)** → clic en **"Asignar activos"**

❌ No usar el botón "Conectar activos" desde la app — ese solo muestra cuentas publicitarias.

---

## PASO 5 — Generar token permanente

`Usuarios del sistema` → selecciona el usuario → clic en **"Generar identificador"**

- Seleccionar la app del cliente
- Establecer caducidad: **Nunca**
- Permisos requeridos:
  - ✅ `whatsapp_business_messaging`
  - ✅ `whatsapp_business_management`

**⚠️ Copiar y guardar el token inmediatamente** — no se vuelve a mostrar completo.

Guardar en:
- `empresas.chatwoot_api_key` en la BD de AVE
- Lugar seguro del cliente

---

## PASO 6 — Agregar método de pago en Meta

`business.facebook.com` → portfolio del cliente → Facturación → **Cuentas de WhatsApp Business** → Añadir método de pago

Sin método de pago el registro del número falla con error genérico.

---

## PASO 7 — Crear inbox en Chatwoot ← ANTES de registrar el número

Chatwoot → cuenta del cliente → Ajustes → Entradas → Añadir bandeja de entrada → WhatsApp → **WhatsApp Cloud**

| Campo | Valor |
|---|---|
| Nombre | `{cliente}-produccion` |
| Número de teléfono | `+57XXXXXXXXXX` (con código de país, sin espacios) |
| ID de número de teléfono | Phone Number ID de Meta |
| ID de cuenta de negocio | WhatsApp Business Account ID de Meta |
| Clave de API | Token del sistema (paso 5) |

Al guardar, Chatwoot genera:
- **URL del webhook** → copiar
- **Token de verificación** → copiar

---

## PASO 8 — Configurar webhook en Meta

`developers.facebook.com` → app del cliente → Casos de uso → Personalizar → **Configuración** → sección Webhook

| Campo | Valor |
|---|---|
| URL de devolución de llamada | URL que dio Chatwoot |
| Identificador de verificación | Token que dio Chatwoot |

Clic en **"Verificar y guardar"**.

La URL tiene este formato:
```
https://chatwoot.identechnology.co/webhooks/whatsapp/+57XXXXXXXXXX
```

---

## PASO 9 — Activar campo `messages` ← EL MÁS OLVIDADO

En la misma pantalla de Configuración → sección **"Campos de webhook"** → buscar el campo **`messages`** → activar el toggle para suscribirse.

**Sin este paso los mensajes NO llegan a Chatwoot aunque el webhook esté verificado.**

Todos los campos vienen en "Suscripción cancelada" por defecto.

---

## PASO 10 — Registrar el número en Meta

`developers.facebook.com` → app del cliente → Paso 2. Configuración de producción → activar toggle **"Suscribirse a webhooks"** → clic en **"Registrarte"**

Si el número queda como "Registrado" → ✅ listo.

---

## PASO 11 — Webhook Chatwoot → n8n

⚠️ Este paso solo se hace UNA vez por cuenta de Chatwoot, no por cada cliente.

Chatwoot → cuenta del cliente → Ajustes → Integraciones → Webhooks → Añadir nuevo webhook

| Campo | Valor |
|---|---|
| URL | `https://hn8n.identechnology.co/webhook/chatwoot` |
| Eventos | ✅ Mensaje creado |

Si la cuenta ya tiene este webhook configurado (de un inbox anterior), no crear uno nuevo.

---

## PASO 12 — Registrar token en la BD de AVE

```sql
UPDATE empresas
SET chatwoot_api_key = '[token_sistema]'
WHERE id = [empresa_id];

SELECT id, nombre, chatwoot_account_id, chatwoot_url 
FROM empresas WHERE id = [empresa_id];
```

---

## Datos a anotar por cliente

| Campo | Dónde encontrarlo |
|---|---|
| Phone Number ID | developers → app → Configuración → Números de teléfono |
| WhatsApp Business Account ID | developers → app → Paso 2 → debajo del nombre de la cuenta |
| Token del sistema | Business Manager → Usuarios del sistema → Generar identificador |
| URL webhook Chatwoot | Chatwoot → inbox → Configuración → Token de verificación |
| Token verificación Chatwoot | Chatwoot → inbox → Configuración → Token de verificación |

---

## Errores frecuentes y soluciones

| Error | Causa | Solución |
|---|---|---|
| "Se ha producido un error durante el registro" | Cuenta de Facebook con restricciones activas | Revisar `facebook.com/accountquality` |
| "Se ha producido un error durante el registro" | Número con WhatsApp instalado | Desinstalar WhatsApp y eliminar cuenta primero |
| "Se ha producido un error durante el registro" | Sin método de pago en Meta | Agregar tarjeta en Facturación → Cuentas de WhatsApp Business |
| "No hay permisos disponibles" al generar token | App no asignada al usuario del sistema | BM → Usuarios del sistema → tres puntos → Asignar activos → Aplicaciones |
| "Provider config Invalid Credentials" en Chatwoot | Token personal en lugar de token del sistema | Generar token desde BM → Usuarios del sistema |
| "Phone number has already been taken" en Chatwoot | Inbox ya existe de sesión anterior | Usar el inbox existente, no crear uno nuevo |
| Mensajes no llegan a Chatwoot | Campo `messages` no suscrito en Meta | Activar `messages` en Campos de webhook |
| Mensajes llegan a Chatwoot pero bot no responde | Workflow n8n no está activo | Verificar que Bot_Agencia_final esté en Published |
| Mensajes llegan a Chatwoot pero bot no responde | Webhook Chatwoot → n8n no configurado | Ajustes → Integraciones → Webhooks → verificar URL de n8n |

---

## Clientes configurados

| Cliente | empresa_id | account_id | Número | Estado |
|---|---|---|---|---|
| agencIA | 1 | 1 | +57... | ✅ Activo |
| Uhane SAS | 7 | 3 | +573170997828 | ✅ Activo jun 2026 |
| PC Outlet | 8 | 4 | +573023371255 | ⏳ Pendiente token |
| Tienda Virtual 4030 | 9 | - | - | ⏳ Pendiente WhatsApp |
