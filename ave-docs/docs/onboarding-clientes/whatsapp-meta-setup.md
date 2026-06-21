# Guía de Integración: WhatsApp Business API + Chatwoot

Proceso completo desde cero para crear una app de WhatsApp Business y conectarla con Chatwoot, incluyendo los errores más frecuentes encontrados durante la implementación y sus soluciones. **Aplica para cada cliente nuevo que se incorpore al sistema AVE.**

| Versión | Fecha | Plataforma |
|---|---|---|
| 1.0 | Junio 2026 | Chatwoot 4.14 |

---

## Prerrequisitos

- Cuenta de Facebook activa con verificación en 2 pasos.
- Acceso a Meta Business Suite (`business.facebook.com`).
- Acceso a Meta for Developers (`developers.facebook.com`).
- Chatwoot instalado y funcionando.
- Número de teléfono disponible (sin WhatsApp instalado).

⚠️ **Sobre el número de teléfono:** el número que uses **no puede tener WhatsApp instalado actualmente**. Si lo tiene, debes desregistrarlo primero desde la app antes de continuar. Se recomienda usar una SIM nueva o un número virtual dedicado.

---

## Paso 1 — Crear página de Facebook

Meta requiere una página de Facebook activa para asociarla a la cuenta de WhatsApp Business.

| # | Instrucción |
|---|---|
| 1 | Ve a `facebook.com` e inicia sesión con tu cuenta. |
| 2 | En el menú izquierdo busca "Páginas" → "Crear nueva página". |
| 3 | Ingresa el nombre del negocio o cliente. |
| 4 | Selecciona la categoría más cercana al tipo de negocio. |
| 5 | Clic en "Crear" y completa la información básica. |

---

## Paso 2 — Crear app en Meta for Developers

| # | Instrucción |
|---|---|
| 1 | Ve a `developers.facebook.com` e inicia sesión. |
| 2 | Clic en "Mis apps" → "Crear app". |
| 3 | En "Casos de uso" selecciona "Conectarte con los clientes a través de WhatsApp" y clic en Siguiente. |
| 4 | En "Negocio" selecciona el portfolio comercial existente o crea uno nuevo con "Crear un portfolio comercial". |
| 5 | Si creas un portfolio nuevo: en el popup selecciona "Verificar más tarde" (no es necesario para pruebas). |
| 6 | En "Requisitos" clic en Siguiente (normalmente no hay requisitos pendientes). |
| 7 | Revisa el resumen y clic en "Crear app". |

⚠️ **Si creas un portfolio comercial nuevo:** aparecerán opciones para verificar el negocio. Selecciona "Verificar más tarde" — puedes verificarlo después. La verificación es necesaria solo para enviar mensajes a usuarios fuera de tu lista de prueba.

---

## Paso 3 — Configurar WhatsApp en la app

### 3.1 Activar caso de uso WhatsApp

| # | Instrucción |
|---|---|
| 1 | Dentro de la app, ve a "Casos de uso" en el menú izquierdo. |
| 2 | Clic en "Personalizar" del caso "Conectar en WhatsApp". |
| 3 | Selecciona el portfolio comercial del cliente y clic en "Continuar". |
| 4 | Selecciona "Integrar con la API" como tipo de integración. |

### 3.2 Registrar número de teléfono

| # | Instrucción |
|---|---|
| 1 | Ve a "Paso 2: Configuración de producción" en el menú izquierdo. |
| 2 | Expande "Registra tu número de teléfono de WhatsApp". |
| 3 | Activa el toggle "Suscribir webhooks" — debe quedar en **azul/verde**. |
| 4 | Clic en "Registrar" al lado del número. |
| 5 | Anota los datos que aparecen: **Phone Number ID** e **WhatsApp Business Account ID**. |

📌 **Guarda estos datos — los necesitarás en Chatwoot:**

| Campo | Valor |
|---|---|
| Phone Number ID | Aparece al lado del número registrado. |
| WhatsApp Business Account ID | Aparece en el encabezado de la sección. |
| Número de teléfono | El número que registraste (ej. `+57 322 2752524`). |

---

## Paso 4 — Generar token de usuario del sistema

⚠️ **Por qué no usar el token permanente de Developers:** el token que genera Meta for Developers está vinculado a tu usuario personal de Facebook. Chatwoot requiere un **token de usuario del sistema** (System User Token). Si usas el token personal obtendrás el error *"Provider config Invalid Credentials"*. Siempre usa el token de usuario del sistema para integraciones con Chatwoot.

### 4.1 Crear usuario del sistema

| # | Instrucción |
|---|---|
| 1 | Ve a `business.facebook.com`. |
| 2 | Selecciona tu portfolio comercial. |
| 3 | Ve a Configuración → Usuarios → Usuarios del sistema. |
| 4 | Si no hay usuarios, clic en "+ Agregar" y crea uno con rol **Administrador**. |
| 5 | Si ya existe un usuario tipo "Employee", puedes usarlo. |

### 4.2 Asignar cuenta de WhatsApp al usuario

| # | Instrucción |
|---|---|
| 1 | Selecciona el usuario del sistema en la lista. |
| 2 | Clic en "Administrar activos" o los tres puntos "..." → editar. |
| 3 | En "Cuentas de WhatsApp" selecciona la cuenta del negocio. |
| 4 | En permisos selecciona "Control total" → "Todo". |
| 5 | Clic en "Asignar activos". |

### 4.3 Generar token permanente

| # | Instrucción |
|---|---|
| 1 | Con el usuario seleccionado, clic en "Generar token". |
| 2 | Selecciona la app del cliente. |
| 3 | En caducidad selecciona **"Nunca"**. |
| 4 | En permisos activa: `whatsapp_business_messaging` y `whatsapp_business_management`. |
| 5 | Clic en "Generar token". |
| 6 | **Copia y guarda el token** — no volverá a mostrarse completo. |

🔒 **Guarda el token de inmediato:** el token solo se muestra una vez de forma completa. Guárdalo en un lugar seguro como un gestor de contraseñas. Si lo pierdes deberás revocar el token y generar uno nuevo.

---

## Paso 5 — Crear inbox en Chatwoot

| # | Instrucción |
|---|---|
| 1 | Ve a Chatwoot → Ajustes → Entradas → Añadir bandeja de entrada. |
| 2 | Selecciona "WhatsApp" como canal. |
| 3 | Selecciona "WhatsApp Cloud" como proveedor. |
| 4 | Llena los campos con los datos de Meta (ver tabla abajo). |
| 5 | Clic en "Crear canal de WhatsApp". |
| 6 | Chatwoot mostrará la URL del webhook — **cópiala**. |

📌 **Datos requeridos para el inbox:**

| Campo | Valor |
|---|---|
| Nombre | Nombre descriptivo (ej. `iden-produccion`). |
| Número de teléfono | Formato internacional, ej. `+57 322 2752524`. |
| ID de número de teléfono | Phone Number ID de Meta for Developers. |
| ID de cuenta de negocio | WhatsApp Business Account ID de Meta. |
| Clave de API | Token del usuario del sistema (Paso 4.3). |

---

## Paso 6 — Configurar webhook en Meta

| # | Instrucción |
|---|---|
| 1 | Ve a Meta for Developers → tu app → Casos de uso → Personalizar WhatsApp. |
| 2 | Ve a "Paso 2: Configuración de producción". |
| 3 | Expande "Configurar webhooks". |
| 4 | En "URL de devolución de llamada" pega la URL que te dio Chatwoot. |
| 5 | En "Token de verificación" pega el token que te dio Chatwoot. |
| 6 | Clic en "Verificar y guardar". |
| 7 | Si es exitoso, Meta redirige a "Permisos y funciones". |

La URL del webhook de Chatwoot tiene este formato:
```
https://chatwoot.tudominio.co/webhooks/whatsapp/+NUMERODETELEFONO
```

⚠️ **Error frecuente — webhook no verifica:** si la verificación falla, revisa que la URL de Chatwoot sea accesible desde internet, que el token de verificación sea exactamente el que muestra Chatwoot, y que Chatwoot esté corriendo sin errores.

---

## Paso 7 — Configurar webhook en Chatwoot para n8n

| # | Instrucción |
|---|---|
| 1 | Ve a Chatwoot → Ajustes → Integraciones → Webhooks. |
| 2 | Clic en "Añadir nuevo webhook". |
| 3 | URL: `https://hn8n.tudominio.co/webhook/chatwoot`. |
| 4 | Eventos: marca **"Mensaje creado"**. |
| 5 | Clic en "Guardar". |

Con esto, cada mensaje que llegue al inbox de WhatsApp se enviará automáticamente a n8n para que el bot lo procese.

---

## Errores frecuentes y soluciones

### Error: "Provider config Invalid Credentials"

**Causa:** se usó el token permanente de Meta for Developers en lugar del token de usuario del sistema. El token personal de Facebook no tiene los permisos correctos para Chatwoot.

**Solución:** generar y usar el token de usuario del sistema (ver Paso 4).

### Error: El número no tiene WhatsApp

**Causa:** el número está registrado en WhatsApp Business API pero no en la app de WhatsApp normal. Esto es **normal** — la Cloud API no funciona como una app de WhatsApp convencional.

**Solución:**
- Para recibir mensajes de prueba, usa el código QR que aparece en Meta for Developers → Paso 2 → "Prueba tu número registrado".
- Escanea el QR con tu celular para abrir WhatsApp y enviar un mensaje al número registrado.
- También puedes agregar números de prueba en la sección correspondiente de Meta for Developers.

### Error: Webhook no verifica en Meta

**Causa:** la URL de Chatwoot no es accesible desde internet, o el token no coincide.

**Solución:**
- Verifica que Chatwoot esté corriendo: `curl https://chatwoot.tudominio.co/auth/sign_in`.
- Copia exactamente la URL y el token que muestra Chatwoot al crear el inbox.
- Asegúrate de no tener espacios extra al pegar los valores.

### Error: Mensajes no llegan a Chatwoot

**Causa:** el webhook de n8n no está activo, o el webhook de Chatwoot hacia n8n no está configurado.

**Solución:**
- Verifica que el workflow de n8n esté en estado **"Published"** (no draft).
- Revisa Chatwoot → Integraciones → Webhooks que la URL de n8n esté correcta.
- En Meta for Developers verifica que "Suscribir webhooks" esté activo (toggle azul).
- Revisa los logs de n8n en Executions para ver si hay errores.

---

## Checklist de verificación

Usa esta lista para confirmar que todo está correctamente configurado.

**Meta for Developers**
- [ ] App creada con caso de uso WhatsApp.
- [ ] Portfolio comercial asociado.
- [ ] Número de teléfono registrado (estado: "Registrado" en verde).
- [ ] "Suscribir webhooks" activado (toggle azul).
- [ ] Webhook URL de Chatwoot configurado y verificado.

**Business Manager**
- [ ] Usuario del sistema creado con rol Administrador.
- [ ] Cuenta de WhatsApp asignada al usuario con Control total.
- [ ] Token permanente generado con permisos `whatsapp_business_messaging` y `whatsapp_business_management`.
- [ ] Token guardado en lugar seguro.

**Chatwoot**
- [ ] Inbox WhatsApp Cloud creado con token del sistema.
- [ ] Phone Number ID correcto.
- [ ] Business Account ID correcto.
- [ ] Webhook de n8n configurado en Integraciones → Webhooks.

**n8n**
- [ ] Workflow `Bot_Agencia_final` en estado Published.
- [ ] Webhook de Chatwoot apuntando a la URL correcta de n8n.
- [ ] Prueba exitosa: mensaje enviado y respondido por el bot.

---

## Datos a registrar por cada cliente

Al completar el proceso para un nuevo cliente, registra estos datos en la base de datos (tabla `empresas`):

| Campo | Valor |
|---|---|
| `chatwoot_account_id` | ID de la cuenta en Chatwoot (se ve en la URL). |
| `chatwoot_api_key` | Token del usuario del sistema de Meta. |
| `chatwoot_url` | URL de Chatwoot (ej. `https://chatwoot.tudominio.co`). |
| Phone Number ID | ID del número en Meta for Developers. |
| WhatsApp Business Account ID | ID de la cuenta de negocio en Meta. |
| Número de teléfono | Formato `+57XXXXXXXXXX`. |

---

*AVE — identechnology.co · Junio 2026*
