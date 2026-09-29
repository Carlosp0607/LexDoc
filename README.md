# LexDoc

![Pruebas](https://github.com/Carlosp0607/lexdoc/actions/workflows/pruebas.yml/badge.svg)

Sistema de gestión y asignación de casos jurídicos para firmas de abogados. Control de acceso por roles, gestión documental y alertas automáticas por correo antes del vencimiento de un proceso.

Desarrollado bajo contrato de prestación de servicios para Turizo Lawyers Enterprise S.A. (enero 2024 – mayo 2025).

**Demo funcional:** [lexdoc.onrender.com](https://lexdoc.onrender.com)

> Esta demo es una recreación independiente del sistema, poblada únicamente con
> datos ficticios. No está conectada a la instancia de la firma ni contiene
> información real de casos, clientes o abogados.

---

## Qué resuelve

Una firma de abogados maneja decenas de procesos en paralelo, cada uno con documentos asociados y una fecha de vencimiento que no se puede pasar por alto. Sin un sistema, esa información vive en carpetas compartidas y en la memoria de quien asignó el caso.

LexDoc centraliza tres cosas:

- **Quién ve qué.** Cada abogado accede únicamente a los casos que le fueron asignados.
- **Dónde están los documentos.** Carga y descarga desde la aplicación, asociados al caso.
- **Cuándo vence.** El sistema revisa los vencimientos y envía un correo al abogado responsable antes de la fecha límite.

---

## Roles

La aplicación define tres perfiles. El rol se guarda en la tabla `usuarios` y determina a qué rutas puede entrar cada sesión.

| Rol | Permisos |
|---|---|
| **Superadmin** | Crea, edita y elimina usuarios. Ve todos los casos activos y el registro de auditoría. Gestiona su propio perfil y contraseña. |
| **Jefe** | Asigna casos a los abogados. Edita y elimina cualquier caso. Ve todos los procesos. |
| **Abogado** | Sube documentos, edita los casos asignados, cambia su estado y registra comentarios para el jefe. Solo ve lo propio. |

El aislamiento no es solo de interfaz: las consultas del panel de abogado filtran por `abogado_id`, así que una sesión de abogado no puede recuperar casos de otro ni cambiando la URL. Esa garantía está cubierta por pruebas automatizadas, ver la sección [Pruebas](#pruebas).

---

## Alertas de vencimiento

El módulo que más valor operativo aporta.

1. Cada hora, una tarea en segundo plano (APScheduler) revisa los casos activos que no están en estado `listo`.
2. Si `fecha_vencimiento` cae en los próximos 7 días y `alerta_enviada = 0`, envía un aviso de vencimiento próximo.
3. Si la fecha ya pasó y `alerta_vencido_enviada = 0`, envía un aviso de caso vencido.
4. Los correos salen por la API de Resend y cada registro queda marcado para no notificar dos veces. Cambiar la fecha del caso reactiva la alerta.

### Bandeja de correos (modo demo)

En la demostración los correos no se envían: se guardan en la tabla `correos_demo` y se muestran dentro de la aplicación, para que cualquier visitante pueda ver las alertas.

| Rol | Pantalla | Qué ve |
|---|---|---|
| Administrador | **Correos enviados** | Todos los correos, con destinatario, asunto y texto completo |
| Abogado | **Mis correos** | Solo las alertas de sus casos |

El administrador tiene dos botones:

- **Revisar vencimientos ahora**: ejecuta en el momento la revisión que el sistema hace sola cada hora.
- **Reiniciar alertas** (`/reset-alertas`): devuelve los flags a cero para poder generar las alertas de nuevo.

Los correos de recuperación de contraseña también llegan a la bandeja, así que el flujo de "¿Olvidaste tu contraseña?" se puede probar completo. Entre los casos de ejemplo hay uno próximo a vencer y uno vencido, para que aparezcan los dos tipos de alerta. El reinicio de cada 12 horas vacía la bandeja.

En modo real nada de esto existe: la pantalla y las rutas responden 404 y los correos salen por Resend.

### Plazos en pantalla

Los paneles de los tres roles usan los mismos umbrales que los correos:

| Plazo | Condición |
|---|---|
| Vencido | La fecha ya pasó |
| Urgente | 7 días o menos |
| Próximo | Entre 8 y 15 días |
| Al día | Más de 15 días |
| Cerrado | Caso en estado `listo`, sin importar la fecha. No cuenta como vencido ni urgente y no genera alertas |

Todas las fechas se calculan en hora de Colombia (`America/Bogota`), tanto en Python como en PostgreSQL, aunque el servidor corra en UTC.

### Documentos

**Ver** abre el PDF en el navegador; **Descargar** lo guarda en el equipo. Los archivos DOC y DOCX solo tienen la opción de descarga.

---

## Modo invitado

La demo expone tres cuentas de prueba, una por rol, accesibles desde `/invitado/<rol>` sin necesidad de credenciales.

Si la base de datos es nueva o quedó incompleta, la ruta crea la cuenta demo en el momento. Esto evita que el botón falle tras un redespliegue o una reinstalación de la base, que es lo que ocurre en el plan gratuito de Render.

---

## Modelo de datos

PostgreSQL. Siete tablas. Las dos principales, `usuarios` y `documentos`, se relacionan por `abogado_id`.

**usuarios**

| Columna | Tipo | Nota |
|---|---|---|
| `id` | SERIAL PK | |
| `nombre` | TEXT | |
| `email` | TEXT UNIQUE | Identificador de login |
| `password` | TEXT | Hash, no texto plano |
| `rol` | TEXT | `superadmin`, `jefe` o `abogado` |

**documentos**

| Columna | Tipo | Nota |
|---|---|---|
| `id` | SERIAL PK | |
| `titulo` | TEXT NOT NULL | |
| `cliente` | TEXT NOT NULL | |
| `archivo` | TEXT NOT NULL | Nombre del archivo almacenado |
| `fecha_vencimiento` | DATE NOT NULL | Base de las alertas |
| `notas` | TEXT | |
| `comentario_abogado` | TEXT | Seguimiento del responsable |
| `estado_caso` | TEXT | `pendiente` (defecto), `en_proceso`, `listo`, `requiere_revision` |
| `abogado_id` | INTEGER FK | Abogado asignado. `ON DELETE RESTRICT`: no se borra un abogado con casos |
| `asignado_por` | INTEGER | Quién hizo la asignación |
| `fecha_subida` | TIMESTAMP | |
| `fecha_actualizacion` | TIMESTAMP | |
| `alerta_enviada` | INTEGER | Aviso de 7 días ya enviado |
| `alerta_vencido_enviada` | INTEGER | Aviso de vencido ya enviado |
| `eliminado_en` | TIMESTAMP | Fecha de envío a papelera. `NULL` = activo |

**Tablas de soporte**

| Tabla | Uso |
|---|---|
| `archivos` | Contenido binario de los documentos (`BYTEA`) |
| `intentos_login` | Intentos fallidos, base del bloqueo de 15 minutos |
| `auditoria` | Quién hizo qué y cuándo |
| `tokens_recuperacion` | Enlaces de recuperación de contraseña (hash, vencimiento, uso único) |
| `correos_demo` | Bandeja de la demostración: correos guardados en vez de enviados |

Las tablas se crean con `CREATE TABLE IF NOT EXISTS` y las columnas nuevas se agregan con `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`. El arranque es idempotente: la aplicación puede reiniciarse sobre una base existente sin romper nada ni perder datos.

---

## Pruebas

54 pruebas con `pytest` (64 casos: dos de ellas se ejecutan con 6 valores cada una). Agrupadas:

| Grupo | Qué verifica |
|---|---|
| Acceso por rol | Sin sesión se redirige al login; un abogado no entra a rutas de jefe pero sí a su panel |
| Aislamiento de casos | Un abogado no abre, no lista y no descarga casos de otro abogado |
| Autenticación | Contraseña incorrecta, bloqueo tras 5 intentos, intentos registrados en la base, vencimiento de sesión |
| Usuarios | No se crea otro superadmin, el superadmin no se elimina, no se borra un abogado con casos |
| Casos | Fechas inválidas rechazadas, estado `listo` aceptado, papelera (eliminar, restaurar, borrado definitivo) |
| Archivos | Extensión falsa rechazada; fuera de demo se guarda y descarga el archivo real |
| Seguridad web | POST sin token CSRF rechazado, eliminar no acepta GET, cabecera CSP presente |
| Operación | Auditoría, recuperación de contraseña de un solo uso sin revelar si el correo existe, alertas próximas y vencidas, cambio de fecha reactiva la alerta |
| Reglas de negocio | `calcular_estado` con los umbrales vencido / urgente / próximo / ok |
| Modo demo | Solo se activa con `MODO_DEMO=1`; fuera de demo se retiran las cuentas demo; el reinicio no borra si hay archivos reales ni corre fuera de demo; reiniciar alertas solo existe en demo |
| Coherencia | Un caso listo aparece como cerrado; el comentario del abogado no se guarda como instrucción del jefe; Ver abre y Descargar descarga |
| Sesiones | Cerrar sesión solo por POST; usuario eliminado pierde la sesión; cambio de rol inmediato; cambiar o restablecer la contraseña cierra las otras sesiones; bloqueo por IP con correos distintos |
| Bandeja demo | En demo el correo se guarda y no llama a Resend; fuera de demo no se guarda; la revisión llena la bandeja; cada abogado ve solo sus correos; solo el administrador ejecuta la revisión; fuera de demo la bandeja no existe |
| Configuración | `APP_URL` obligatoria fuera de demo, correo del admin desde `ADMIN_EMAIL`, fechas en hora de Colombia en Python y PostgreSQL |

La central es `test_abogado_no_abre_el_caso_de_otro_abogado`: si alguien quita el `AND abogado_id = %s` de la consulta, falla antes de llegar a producción.

Requieren una base PostgreSQL accesible. Para levantar una desechable:

```bash
docker run -d --name pg-test \
  -e POSTGRES_PASSWORD=test \
  -e POSTGRES_DB=lexdoc_test \
  -p 5433:5432 \
  postgres:16-alpine
```

Ejecución:

```bash
DATABASE_URL="postgresql://postgres:test@localhost:5433/lexdoc_test" \
SECRET_KEY="clave-de-prueba" \
pytest -v
```

Las pruebas crean sus propios usuarios y casos, y los eliminan al terminar.

---

## Stack

| Componente | Tecnología |
|---|---|
| Lenguaje | Python |
| Framework | Flask |
| Base de datos | PostgreSQL (`psycopg2`) |
| Tareas programadas | APScheduler |
| Correo | Resend |
| Pruebas | pytest |
| Contenedores | Docker y Docker Compose |
| Servidor | Gunicorn |
| Despliegue | Render |

---

## Ejecución con Docker

La forma más rápida de levantar el proyecto completo, aplicación y base de datos, sin instalar Python ni PostgreSQL:

```bash
docker compose up
```

La aplicación queda en `http://localhost:5000`. El esquema y las cuentas de demostración se crean solos en el primer arranque.

Para detener y borrar los datos:

```bash
docker compose down -v
```

Los valores de conexión están fijos en `docker-compose.yml` a propósito: ese archivo es solo para desarrollo local y no toca las credenciales de producción. La aplicación corre con un único worker de Gunicorn, porque el planificador de alertas no debe duplicarse entre procesos.

## Ejecución manual

Si prefiere ejecutar sin contenedores.

```bash
git clone https://github.com/Carlosp0607/lexdoc.git
cd lexdoc
pip install -r requirements.txt
```

Variables de entorno requeridas:

```
DATABASE_URL=postgresql://usuario:clave@host:5432/basededatos
SECRET_KEY=cadena_aleatoria_para_las_sesiones
RESEND_API_KEY=clave_de_resend
RESEND_FROM="LexDoc <alertas@tu-dominio>"
ADMIN_PASSWORD=clave_del_administrador
ADMIN_EMAIL=admin@tu-dominio
MODO_DEMO=0
APP_URL=https://tu-dominio
```

`TRUST_PROXY=1` solo si la aplicación corre detrás de un proxy distinto de Render. En Render se activa sola.

`RESEND_FROM` es el remitente de los correos. Debe ser un dominio verificado en Resend; si no se define, se usa `onboarding@resend.dev`, que solo sirve para pruebas.

`APP_URL` es la dirección pública que va en los enlaces de recuperación de contraseña. Es **obligatoria fuera de demo**: sin ella la aplicación no arranca, para que los enlaces nunca apunten a otro dominio. En demo, si falta, se usa `https://lexdoc.onrender.com`.

`ADMIN_EMAIL` es el correo de la cuenta de administrador que se crea con `ADMIN_PASSWORD`. Debe ser un buzón de la firma: a él llega la recuperación de contraseña. La cuenta se crea una sola vez; cambiar después `ADMIN_EMAIL` crea un segundo administrador.

`MODO_DEMO` está **apagado por defecto**. Solo `MODO_DEMO=1` activa el acceso de invitado, los datos ficticios, el reinicio periódico de la base y las descargas como PDF de prueba. Sin la variable, o con cualquier otro valor, la aplicación corre en modo real: los archivos se guardan en PostgreSQL y se descargan tal cual.

Protecciones del modo real:

- Al arrancar fuera de demo se eliminan las tres cuentas demo (su clave es pública). Si alguna tiene casos asignados no se puede borrar y queda bloqueada con una clave aleatoria.
- El reinicio de la demo no se ejecuta fuera de demo, y aunque la instancia quede en demo por error, se cancela si la base contiene archivos reales.

Sin `RESEND_API_KEY` las alertas por correo no se programan. Sin `ADMIN_PASSWORD` no se crea la cuenta de administrador. Fuera de demo, `ADMIN_PASSWORD` sin `ADMIN_EMAIL` tampoco la crea y deja un error en el log.

```bash
python app.py
```

Las tablas se crean solas en el primer arranque.

---

## Estructura

```
app.py                  Rutas, lógica de negocio, esquema y alertas
wsgi.py                 Punto de entrada para Gunicorn y tareas programadas
Dockerfile              Imagen de la aplicación
docker-compose.yml      Aplicación + PostgreSQL para desarrollo local (modo demo)
pytest.ini              Configuración de pytest
requirements.txt        Dependencias con versión fija
templates/
  base.html             Esqueleto HTML común
  base_app.html         Menú lateral por rol y aviso de demo
  base_acceso.html      Pantallas sin sesión
  _componentes.html     Piezas reutilizables (plazos, estados, formularios)
  login.html, recuperar.html, restablecer.html
  correos.html          Bandeja de correos de la demostración
  superadmin/           Panel general, usuarios, auditoría, perfil y contraseña
  jefe/                 Casos, asignación, edición y papelera
  abogado/              Panel, carga y edición de casos propios
static/
  css/lexdoc.css        Estilos
  js/lexdoc.js          Único JavaScript de la aplicación (requisito de la CSP)
  favicon.svg
tests/
  test_lexdoc.py        Pruebas automatizadas
.github/workflows/
  pruebas.yml           Pruebas en cada push
  backup.yml            Copia de seguridad diaria cifrada
```

---

## Estado

En funcionamiento. Desplegado en Render con base de datos PostgreSQL gestionada.

La demo corre en el plan gratuito, donde la instancia entra en reposo tras un periodo de inactividad. La primera petición puede tardar cerca de 50 segundos en responder.

## Seguridad y operacion

- Contraseñas con hash, token CSRF en todos los formularios, cookie de sesion `HttpOnly`, `SameSite` y `Secure`.
- La sesion vence tras 30 minutos sin actividad.
- Cerrar sesión solo funciona por POST con token CSRF: un enlace o imagen de otro sitio no puede cerrar la sesión de nadie.
- La sesión se valida contra la base en cada petición: si el administrador elimina a un usuario, este sale en el siguiente clic; si le cambia el rol, el nuevo rol aplica de inmediato.
- Cambiar o restablecer la contraseña cierra las demás sesiones abiertas de ese usuario (columna `sesion_version`). Quien la cambia conserva la suya.
- Fechas y registros en hora de Colombia. Los registros anteriores a este cambio quedaron guardados en UTC (5 horas adelante).
- Bloqueo de 15 minutos tras 5 intentos fallidos con el mismo correo, o 20 desde la misma IP aunque cambie el correo (registrado en PostgreSQL). En Render la IP real se lee de `X-Forwarded-For`; fuera de Render solo se confía en esa cabecera con `TRUST_PROXY=1`.
- Politica CSP: solo se ejecuta JavaScript servido por la propia aplicacion (`static/js/lexdoc.js`).
- Archivos validados por extension y por su firma real; guardados en PostgreSQL.
- Registro de auditoria de cada accion (quien, que y cuando), visible para el superadmin.
- Los casos eliminados van a una papelera y se pueden restaurar.
- Recuperacion de contraseña por correo con enlace de un solo uso que vence en 60 minutos.
- Alertas por correo cuando faltan 7 dias o menos y cuando el caso ya vencio.

### Pruebas automaticas

Cada push ejecuta la suite de pytest contra PostgreSQL en GitHub Actions (`.github/workflows/pruebas.yml`).

### Copias de seguridad

`.github/workflows/backup.yml` genera cada dia un volcado de la base, comprimido y cifrado con AES-256, y lo guarda 30 dias como artefacto de GitHub Actions. Sin la clave el archivo no se puede leer.

Configuracion, en *Settings > Secrets and variables > Actions*:

| Secreto | Valor |
|---|---|
| `BACKUP_DATABASE_URL` | URL externa de la base en Render |
| `BACKUP_PASSPHRASE` | Clave para cifrar y descifrar |

Restaurar un backup descargado:

```bash
gpg --decrypt lexdoc-AAAAMMDD-HHMM.sql.gz.gpg | gunzip | psql "URL_DE_LA_BASE_DESTINO"
```
