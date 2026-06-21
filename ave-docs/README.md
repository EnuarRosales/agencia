# AVE — Agencia Virtual de Engagement

Plataforma multi-tenant de gestión de leads por WhatsApp con agentes de IA, desarrollada por **agencIA / identechnology.co**.

AVE permite que múltiples empresas (clientes) operen cada una su propio agente conversacional de IA sobre WhatsApp, con calificación automática de leads, agendamiento, seguimiento y reportería, todo desde una sola infraestructura compartida.

---

## Tabla de contenidos

- [Arquitectura](docs/arquitectura.md)
- [Esquema de base de datos](docs/base-de-datos/esquema.md)
- [Reglas de zona horaria](docs/base-de-datos/reglas-timezone.md)
- [Tipos de etiquetas en AVE](docs/tipos-de-etiquetas.md)
- **Módulos**
  - [Recordatorios automáticos](docs/modulos/recordatorios.md)
  - [Sistema de seguimiento (en_seguimiento)](docs/modulos/seguimiento.md)
  - [Sincronización desactivar_bot](docs/modulos/desactivar-bot.md)
- [Troubleshooting / bugs resueltos](docs/troubleshooting/bugs-resueltos.md)
- [Onboarding de nuevos clientes](docs/onboarding-clientes.md)
  - [Guía de integración WhatsApp Business API + Chatwoot](docs/onboarding-clientes/whatsapp-meta-setup.md)

---

## Stack tecnológico

| Componente | Tecnología | URL / Detalle |
|---|---|---|
| Infraestructura | VPS Ubuntu + Docker + Traefik | IP en UTC |
| Orquestación de flujos | n8n | `hn8n.identechnology.co` |
| Base de datos | PostgreSQL | — |
| CRM / Inbox | Chatwoot | `chatwoot.identechnology.co` |
| Panel de administración | Appsmith | `admin.identechnology.co` |
| API / microservicios | Node.js + Express | `api.identechnology.co` |
| Modelo de IA | OpenAI (GPT-4o-mini) | vía n8n |
| Calendario | Google Calendar | OAuth |

---

## Empresas activas (tenants)

| Empresa | empresa_id | chatwoot_account_id | Negocio | Agente |
|---|---|---|---|---|
| agencIA | 1 | 1 | Automatización / IA B2B | David |
| Iden / Uhanne SAS | 7 | 3 | Venta de vasos | — |
| PC Outlet Colombia | 8 | 4 | Laptops remanufacturadas | Andrés |

---

## Principio de diseño multi-tenant

Toda la lógica de la plataforma es **genérica por empresa**. La configuración específica de cada cliente (prompts, etiquetas, recordatorios, servicios) vive en la base de datos y se administra desde Appsmith, **sin necesidad de modificar los workflows de n8n** al agregar un cliente nuevo.

Los workflows leen la configuración de cada empresa en tiempo de ejecución usando el `chatwoot_account_id` que llega en el webhook, y la aplican dinámicamente. Esto significa que un mismo workflow sirve a todas las empresas.

---

## ⚠️ Seguridad — antes de versionar en Git

Los exports de workflows de n8n (`.json`) **contienen credenciales en texto plano** (`openai_api_key`, `chatwoot_api_key`, etc.). **Nunca** subir estos archivos sin limpiarlos primero.

Ver [guía de limpieza de workflows](docs/onboarding-clientes.md#limpieza-de-workflows-para-git) y el archivo `.gitignore` incluido.

---

## Convención de la bitácora de cambios

Cada módulo documentado incluye al final una sección **Bitácora de cambios** con el formato:

```
| Fecha | Versión | Cambio | Autor |
```

Para cambios en código/workflows, usar mensajes de commit descriptivos con prefijo:
- `feat:` nueva funcionalidad
- `fix:` corrección de bug
- `docs:` cambios solo de documentación
- `refactor:` reestructuración sin cambio de comportamiento
