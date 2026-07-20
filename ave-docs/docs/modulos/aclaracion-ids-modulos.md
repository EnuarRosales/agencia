# Aclaración de IDs de módulos - `modulo_id` vs `orden`

**Ruta sugerida:** `docs/modulos/aclaracion-ids-modulos.md`
**Versión:** v1.0
**Fecha:** 2026-07-20
**Autor:** Enuar Emilio Rosales Salazar
**Motivo:** documentación previa mezclaba PK (`modulo_id`) con `orden`, generando confusión operativa

---

## 1. Contexto

Durante el debugging de AnaMar del 2026-07-20 se identificó que en sesiones y documentos previos se
venían utilizando los valores **10, 30, 40, 100** para referirse a los módulos globales del sistema
AVE. Esos valores corresponden al campo `orden` de la tabla `modulos_bot`, **no al `id` (PK)**.

El campo `orden` se usa exclusivamente en el `STRING_AGG` de `PG_get_empresa` para concatenar los
bloques de prompt en el orden correcto (NUCLEO primero, GUARDRAILS al final). Cualquier operación
DML o de JOIN sobre `empresa_modulos` debe usar el `modulo_id` real (PK).

Este documento es la referencia canónica.

---

## 2. Tabla de referencia canónica

| Módulo     | Nombre completo                                | `modulo_id` (PK) | `orden` |
|------------|------------------------------------------------|------------------|---------|
| NUCLEO     | Nucleo base del agente IA                      | **1**            | 10      |
| AGENDAMIENTO | Agendamiento de citas con Cal.com            | **2**            | 20      |
| MEDIOS     | Envio de fotos, videos y materiales            | **3**            | 30      |
| PEDIDOS    | Recepcion de pedidos (dropshipping/e-commerce) | **4**            | 40      |
| GUARDRAILS | Reglas de seguridad universales                | **5**            | 100     |

**Regla operativa:**
- Para `UPDATE`, `INSERT`, `DELETE` y `JOIN` sobre `empresa_modulos` → usar los valores de `modulo_id`.
- Para consultas ordenadas dentro de `PG_get_empresa` → el `ORDER BY m.orden` sigue haciendo su trabajo internamente.

---

## 3. Query canónica de PG_get_empresa (extracto)

Así es como el nodo `PG_get_empresa` en n8n arma `prompt_modulos_activos`:

```sql
COALESCE(
  (SELECT STRING_AGG(m.prompt_bloque, E'\n\n' ORDER BY m.orden)
   FROM empresa_modulos em
   JOIN modulos_bot m ON m.id = em.modulo_id
   WHERE em.empresa_id = e.id
     AND em.activo = true
     AND m.activo = true),
  ''
) AS prompt_modulos_activos
```

- El `JOIN m.id = em.modulo_id` compara PKs (valores 1, 2, 3, 4, 5).
- El `ORDER BY m.orden` ordena los bloques por posición lógica (10, 20, 30, 40, 100).
- El `COALESCE` protege contra el caso de empresa sin módulos activos (devuelve `''` en vez de `NULL`).

---

## 4. Queries de referencia rápida (usar `modulo_id` real)

### 4.1 Ver módulos activos de una empresa

```sql
SELECT em.empresa_id, m.nombre, em.modulo_id, m.orden, em.activo
FROM empresa_modulos em
JOIN modulos_bot m ON m.id = em.modulo_id
WHERE em.empresa_id = <id>
ORDER BY m.orden;
```

### 4.2 Activar un módulo específico (ejemplo: MEDIOS en AnaMar)

```sql
UPDATE empresa_modulos
SET activo = true
WHERE empresa_id = 12
  AND modulo_id = 3;   -- MEDIOS: PK real es 3, no 30
```

### 4.3 Insertar los 4 módulos estándar para un nuevo tenant

```sql
INSERT INTO empresa_modulos (empresa_id, modulo_id, activo) VALUES
  (<nuevo_id>, 1, true),   -- NUCLEO
  (<nuevo_id>, 3, true),   -- MEDIOS
  (<nuevo_id>, 4, true),   -- PEDIDOS
  (<nuevo_id>, 5, true);   -- GUARDRAILS
```

Para tenants tipo agencIA (con agendamiento):

```sql
INSERT INTO empresa_modulos (empresa_id, modulo_id, activo) VALUES
  (<nuevo_id>, 1, true),   -- NUCLEO
  (<nuevo_id>, 2, true),   -- AGENDAMIENTO
  (<nuevo_id>, 3, true),   -- MEDIOS
  (<nuevo_id>, 5, true);   -- GUARDRAILS
```

### 4.4 Auditoría de configuración modular por empresa

```sql
SELECT
  e.id AS empresa_id,
  e.nombre,
  COUNT(*) FILTER (WHERE em.activo = true) AS modulos_activos,
  STRING_AGG(m.nombre, ', ' ORDER BY m.orden) FILTER (WHERE em.activo = true) AS modulos
FROM empresas e
LEFT JOIN empresa_modulos em ON em.empresa_id = e.id
LEFT JOIN modulos_bot m ON m.id = em.modulo_id
WHERE e.activo = true
GROUP BY e.id, e.nombre
ORDER BY e.id;
```

---

## 5. Configuraciones estándar por tipo de tenant (referencia)

| Tenant tipo                          | Módulos activos (PK)         |
|--------------------------------------|------------------------------|
| Dropshipping / e-commerce con pedido | 1 (NUCLEO), 3 (MEDIOS), 4 (PEDIDOS), 5 (GUARDRAILS) |
| Servicios con agendamiento           | 1 (NUCLEO), 2 (AGENDAMIENTO), 3 (MEDIOS), 5 (GUARDRAILS) |
| E-commerce con transferencia humana  | 1 (NUCLEO), 3 (MEDIOS), 5 (GUARDRAILS) |
| Servicios sin multimedia             | 1 (NUCLEO), 5 (GUARDRAILS) |

Estados actuales de los 5 tenants en producción (2026-07-20):

| empresa_id | Nombre     | Tipo                              | Módulos activos             |
|------------|------------|-----------------------------------|-----------------------------|
| 1          | agencIA    | Servicios con agendamiento        | 1, 2, 3, 5                  |
| 7          | Uhane SAS  | E-commerce con pedidos            | 1, 3, 4, 5                  |
| 8          | PC_Outlet  | E-commerce con transferencia hum. | 1, 3, 5                     |
| 9          | tienda4030 | Dropshipping                      | 1, 3, 4, 5                  |
| 12         | AnaMar     | Dropshipping                      | 1, 3, 4, 5                  |

---

## 6. Nota histórica

Documentos anteriores mencionan indistintamente:

- `modulo_id = 30` en referencias a MEDIOS
- `modulo_id = 40` en referencias a PEDIDOS
- `modulo_id = 100` en referencias a GUARDRAILS

Esos valores están **incorrectos** en términos de PK. Corresponden al `orden` interno.

Al revisar sesiones o notas viejas, siempre traducir mentalmente:

```
10 → 1 (NUCLEO)
20 → 2 (AGENDAMIENTO)
30 → 3 (MEDIOS)
40 → 4 (PEDIDOS)
100 → 5 (GUARDRAILS)
```

Este documento reemplaza cualquier referencia previa a los IDs y debe considerarse la fuente
canónica de verdad.

---

## 7. Referencias cruzadas

- `docs/modulos/arquitectura-prompts-sandwich.md` (referencia arquitectónica)
- `docs/troubleshooting/bugs-resueltos.md` (bug 57 documenta el descubrimiento)
- `docs/onboarding-clientes/onboarding-clientes.md` (usa esta tabla en el paso de activación de módulos)
