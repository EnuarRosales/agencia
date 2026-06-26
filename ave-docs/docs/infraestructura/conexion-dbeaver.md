# Conexión a la base de datos AVE con DBeaver (túnel SSH)

> `docs/infraestructura/conexion-dbeaver.md`
> Cómo conectar DBeaver desde un equipo local a la base de datos de producción de AVE en el VPS,
> de forma segura mediante túnel SSH (sin exponer Postgres a internet).

---

## Contexto

La base de datos de AVE corre en un contenedor Docker (`n8n-postgres-1`) dentro del VPS Hostinger.
Postgres **no está expuesto** directamente a internet, por lo que la conexión desde un equipo local
se hace a través de un **túnel SSH**: DBeaver se conecta por SSH al VPS y, a través de ese túnel,
alcanza Postgres como si fuera local.

> Hay dos contenedores Postgres en el VPS: `n8n-postgres-1` (base de AVE: conversaciones, leads,
> recordatorios, etc.) y `chatwoot-postgres-1` (base interna de Chatwoot). **El de AVE es
> `n8n-postgres-1`.**

## Datos de conexión

### Pestaña SSH (túnel hacia el VPS)
| Campo | Valor |
|---|---|
| Host/IP | `2.25.172.227` (IP pública del VPS) |
| Port | `22` |
| User Name | `root` |
| Authentication Method | Password |
| Password | (contraseña de root del VPS) |
| Save credentials | ✅ |

### Pestaña General (Postgres a través del túnel)
| Campo | Valor |
|---|---|
| Host | `localhost` (porque vía el túnel, Postgres se ve como local) |
| Port | `5432` |
| Database | `postgres` |
| Username | `postgres` |
| Password | (contraseña de Postgres) |

> **Importante:** la contraseña de la pestaña SSH (root del VPS) y la de la pestaña General (Postgres)
> son **independientes**, aunque puedan configurarse iguales. No confundirlas: el túnel usa root del
> VPS; la base usa el usuario de Postgres.

## Cómo obtener las credenciales (desde el VPS)

Las credenciales de Postgres están en las variables de entorno del contenedor:
```bash
docker exec n8n-postgres-1 env | grep POSTGRES
```
Si `POSTGRES_USER` y `POSTGRES_DB` no aparecen, usan el default `postgres` (confirmable porque
`psql -U postgres` funciona y el owner de las tablas es `postgres`).

Para verificar que es la base correcta de AVE:
```bash
docker exec n8n-postgres-1 psql -U postgres -c "\dt"
```
Debe listar: `conversaciones`, `leads`, `recordatorios`, `empresas`, `bot_config`,
`recordatorio_config`, `etiquetas_pipeline`, `etiquetas_operativas`, etc.

## Verificar / establecer la contraseña de root del VPS

La contraseña de root NO es la misma que la de Postgres y no siempre queda registrada.

Para comprobar si una contraseña candidata es la de root (verificación real contra el hash):
```bash
echo "CANDIDATA" | python3 -c "import crypt,sys,spwd; h=spwd.getspnam('root').sp_pwdp; p=sys.stdin.readline().strip(); print('CORRECTA' if crypt.crypt(p,h)==h else 'INCORRECTA')"
```
> No usar `su root` para verificar: si ya estás logueado como root, no pide contraseña y da un
> falso positivo.

Para establecer/resetear la contraseña de root (si se accede al VPS por panel web/Google y no se
conoce la de SSH):
```bash
passwd root
```
> Si se cambia, actualizar el Password en la pestaña SSH de DBeaver en **todos** los equipos que
> conecten, porque es la misma para todos.

## Pasos de conexión en DBeaver

1. Nueva conexión → PostgreSQL.
2. Pestaña **General**: llenar Host `localhost`, Port `5432`, Database `postgres`, usuario y contraseña de Postgres.
3. Pestaña **SSH**: activar túnel, llenar IP del VPS, puerto 22, usuario root, contraseña de root.
4. **Test tunnel configuration** → debe confirmar el SSH.
5. **Probar conexión** → debe confirmar Postgres.
6. Aceptar.

## Troubleshooting

- **`SSH password authentication failed / Exhausted available authentication methods`**: la
  contraseña de la pestaña SSH es incorrecta o se confundió con la de Postgres. Verificar/resetear
  con `passwd root`.
- **El túnel conecta pero Postgres no**: revisar que Host sea `localhost` (no la IP) y que usuario/
  base de la pestaña General sean los de Postgres.
- **Una IP/host de contenedor no funciona desde el PC**: los nombres de contenedor Docker
  (`n8n-postgres-1`) solo resuelven dentro del VPS, no desde el equipo local. Por eso vía túnel se
  usa `localhost`.

## Nota de seguridad

Mantener la contraseña de root del VPS distinta a la de Postgres y a cualquier otra credencial.
Compartir el mismo password para acceso root y base de datos de producción es un riesgo: si una se
expone, ambas quedan comprometidas. Rotar credenciales antes de salir a producción.
