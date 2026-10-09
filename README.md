# Django + React + PostgreSQL con Docker Compose

Guía de referencia para desarrollar, ejecutar, depurar y mantener este proyecto en macOS (Apple Silicon/M5).

> **Importante:** ejecuta los comandos de Docker Compose desde la raíz del repositorio, donde está `docker-compose.yml` o `compose.yaml`. Los ejemplos asumen que los servicios se llaman `backend`, `frontend` y `db`, y que la base de datos se llama `django_db` con el usuario `django_user`. Si tus nombres difieren, ajusta los comandos.
>
> **Seguridad:** no subas `.env`, claves secretas, contraseñas ni tokens a GitHub. Usa `.env.example` con valores ficticios. No ejecutes `docker compose down -v` si necesitas conservar la base de datos.

## Índice

- [1. Requisitos](#1-requisitos)
- [2. Estructura del proyecto](#2-estructura-del-proyecto)
- [3. Variables de entorno](#3-variables-de-entorno)
- [4. Comandos esenciales de Docker Compose](#4-comandos-esenciales-de-docker-compose)
- [5. Logs y diagnóstico](#5-logs-y-diagnóstico)
- [6. Django dentro de Docker](#6-django-dentro-de-docker)
- [7. Usuarios y autenticación JWT](#7-usuarios-y-autenticación-jwt)
- [8. PostgreSQL con `psql`](#8-postgresql-con-psql)
- [9. Backups y restauración](#9-backups-y-restauración)
- [10. React, Vite y pnpm](#10-react-vite-y-pnpm)
- [11. Pruebas HTTP con curl y Postman](#11-pruebas-http-con-curl-y-postman)
- [12. Problemas frecuentes](#12-problemas-frecuentes)
- [13. Git y GitHub](#13-git-y-github)
- [14. Flujo habitual de trabajo](#14-flujo-habitual-de-trabajo)

## 1. Requisitos

Comprueba que Docker Desktop está instalado y en ejecución:

```bash
docker --version
docker compose version
docker info
```

Verifica que la arquitectura de la máquina sea Apple Silicon:

```bash
uname -m
# Esperado en Apple Silicon: arm64
```

## 2. Estructura del proyecto

Ejemplo orientativo:

```text
.
├── backend/
│   ├── manage.py
│   ├── requirements.txt
│   ├── Dockerfile
│   └── .env
├── frontend/
│   ├── package.json
│   ├── pnpm-lock.yaml
│   ├── Dockerfile
│   └── src/
├── docker-compose.yml
├── .gitignore
└── README.md
```

## 3. Variables de entorno

Configuración local del backend en `backend/.env` (ejemplo; sustituye los valores por los de tu entorno):

```dotenv
SECRET_KEY=replace-with-a-long-random-secret
DEBUG=True

DB_NAME=django_db
DB_USER=django_user
DB_PASSWORD=replace-with-a-strong-password
DB_HOST=db
DB_PORT=5432
```

Notas:

- Dentro de Docker Compose, Django debe usar `DB_HOST=db` (el nombre del servicio), no `localhost`.
- `localhost` se utiliza desde el Mac para conectarse a un puerto publicado.
- Los valores `POSTGRES_DB`, `POSTGRES_USER` y `POSTGRES_PASSWORD` inicializan PostgreSQL **solo cuando el directorio de datos está vacío**. Cambiar esas variables después no modifica automáticamente la contraseña de un usuario ya creado.
- `SECRET_KEY` y `DB_PASSWORD` deben coincidir con los nombres que lee `settings.py`.
- Para producción, `DEBUG=False`, secretos fuertes y variables gestionadas de forma segura.

Comprueba que los archivos de entorno no se rastrean:

```bash
git status --short
git check-ignore backend/.env
```

## 4. Comandos esenciales de Docker Compose

### Estado y servicios

```bash
docker compose config                 # Valida y muestra la configuración combinada
docker compose ps                     # Estado de los servicios
docker compose ps -a                  # Incluye contenedores detenidos
docker compose images                 # Imágenes asociadas al proyecto
docker compose top                    # Procesos dentro de los servicios
```

### Iniciar y detener

```bash
docker compose up -d                  # Construye si hace falta e inicia en segundo plano
docker compose up                     # Inicia y muestra logs en la terminal
docker compose stop                   # Detiene contenedores sin eliminarlos
docker compose start                  # Inicia contenedores detenidos
docker compose down                   # Elimina contenedores y red; conserva volúmenes nombrados
docker compose restart                # Reinicia servicios
docker compose restart backend        # Reinicia solo Django
docker compose restart frontend       # Reinicia solo React
docker compose restart db             # Reinicia solo PostgreSQL
```

### Construir y recrear

```bash
docker compose build                  # Construye las imágenes
docker compose build --no-cache       # Reconstruye sin caché
docker compose build backend          # Construye solo Django
docker compose build frontend         # Construye solo React
docker compose up -d --build          # Construye e inicia
docker compose up -d --force-recreate backend
```

Después de cambiar `requirements.txt`, `package.json`, el lockfile o un Dockerfile, normalmente reconstruye el servicio correspondiente.

### Ejecutar comandos en un servicio

```bash
docker compose exec backend sh
docker compose exec frontend sh
docker compose exec db sh
```

Si un servicio no está ejecutándose, usa `docker compose run --rm SERVICIO COMANDO` cuando sea apropiado.

### Volúmenes y redes

```bash
docker volume ls
docker volume inspect NOMBRE_DEL_VOLUMEN
docker network ls
docker compose port backend 8000
docker compose port frontend 5173
docker compose port db 5432
```

**Precaución:** `docker compose down -v` elimina los volúmenes declarados por Compose y puede borrar la base de datos persistida. No lo uses para solucionar errores de autenticación o configuración.

## 5. Logs y diagnóstico

```bash
docker compose logs
docker compose logs --tail=100
docker compose logs -f
docker compose logs -f backend
docker compose logs -f frontend
docker compose logs -f db
docker compose logs --since=10m backend
```

Comprueba los servicios:

```bash
docker compose ps
docker compose exec backend python --version
docker compose exec backend python -m pip list
docker compose exec frontend node --version
docker compose exec frontend pnpm --version
docker compose exec db psql --version
```

Si un contenedor está en bucle de reinicio, consulta primero sus logs:

```bash
docker compose logs --tail=200 backend
docker compose logs --tail=200 frontend
docker compose logs --tail=200 db
```

## 6. Django dentro de Docker

Ejecuta estos comandos desde la raíz del repositorio.

### Comprobaciones y migraciones

```bash
docker compose exec backend python manage.py check
docker compose exec backend python manage.py showmigrations
docker compose exec backend python manage.py migrate
docker compose exec backend python manage.py makemigrations
docker compose exec backend python manage.py makemigrations NOMBRE_APP
docker compose exec backend python manage.py sqlmigrate NOMBRE_APP NUMERO_MIGRACION
```

Orden habitual cuando cambias modelos:

1. Modifica `models.py`.
2. Ejecuta `makemigrations`.
3. Revisa la migración generada.
4. Ejecuta `migrate`.
5. Comprueba el resultado con `showmigrations`.

No generes migraciones automáticamente en producción sin revisarlas.

### Shell, configuración y datos

```bash
docker compose exec backend python manage.py shell
docker compose exec backend python manage.py diffsettings
docker compose exec backend python manage.py dumpdata --indent 2
```

Para consultar el motor de base de datos configurado sin imprimir la contraseña:

```bash
docker compose exec backend python manage.py shell -c \
"from django.conf import settings; print({k: v for k, v in settings.DATABASES['default'].items() if k != 'PASSWORD'})"
```

### Archivos estáticos y servidor

```bash
docker compose exec backend python manage.py collectstatic --noinput
docker compose logs -f backend
```

En desarrollo, el servidor suele estar disponible en `http://localhost:8000`. `collectstatic` se usa cuando la configuración de archivos estáticos lo requiere; no suele ser necesario para el flujo diario de desarrollo.

## 7. Usuarios y autenticación JWT

### Listar usuarios desde Django

```bash
docker compose exec backend python manage.py shell
```

Dentro de la shell:

```python
from django.contrib.auth import get_user_model

User = get_user_model()
print("Total:", User.objects.count())

for user in User.objects.all():
    print({
        "id": user.pk,
        "username": user.get_username(),
        "email": user.email,
        "is_active": user.is_active,
        "is_staff": user.is_staff,
        "is_superuser": user.is_superuser,
    })
```

Sal de la shell con `exit()`.

Esta consulta funciona también si el proyecto utiliza un modelo de usuario personalizado. No imprime contraseñas.

### Crear o cambiar credenciales

```bash
docker compose exec backend python manage.py createsuperuser
docker compose exec backend python manage.py changepassword admin
```

Para que SimpleJWT autentique a un usuario, este debe existir, tener la contraseña correcta y estar activo. Si el proyecto usa un modelo de usuario personalizado, usa su campo identificador configurado.

### Endpoints de ejemplo

Confirma las rutas reales en los `urls.py` del proyecto. En este proyecto se han usado estas rutas:

```text
POST http://localhost:8000/api/auth/token/
GET  http://localhost:8000/api/auth/me/
GET  http://localhost:8000/api/personas/
```

La ruta de token puede variar según la configuración de URLs.

Ejemplo de body JSON para obtener tokens:

```json
{
  "username": "TU_USUARIO",
  "password": "TU_CONTRASEÑA"
}
```

Una respuesta exitosa de SimpleJWT normalmente contiene `access` y `refresh`. Para acceder a un endpoint protegido, envía:

```http
Authorization: Bearer TU_ACCESS_TOKEN
```

No pegues tokens reales en este README ni los subas al repositorio.

## 8. PostgreSQL con `psql`

Los ejemplos asumen el servicio `db`, la base `django_db` y el usuario `django_user`.

### Conectarse

```bash
docker compose exec db psql -U django_user -d django_db
```

Desde el Mac, si el puerto `5432` está publicado:

```bash
psql -h localhost -p 5432 -U django_user -d django_db
```

### Consultas útiles dentro de `psql`

```sql
\conninfo                 -- Información de la conexión
\l                        -- Listar bases de datos
\du                       -- Listar roles/usuarios
\dt                       -- Listar tablas del esquema actual
\dt *.*                   -- Listar tablas de todos los esquemas visibles
\d nombre_tabla           -- Describir una tabla
\dn                       -- Listar esquemas
SELECT current_user;
SELECT current_database();
SELECT version();
\q                        -- Salir
```

Consultar las tablas de Django:

```sql
SELECT tablename
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY tablename;
```

En la configuración estándar de Django, listar usuarios (si existe `auth_user`):

```sql
SELECT id, username, email, is_active, is_staff, is_superuser
FROM auth_user
ORDER BY id;
```

Si usas un modelo de usuario personalizado, consulta la tabla que corresponda o utiliza la shell de Django descrita anteriormente.

Consultar migraciones aplicadas:

```sql
SELECT app, name, applied
FROM django_migrations
ORDER BY applied DESC
LIMIT 20;
```

### Contraseña de un rol PostgreSQL

Conéctate usando un rol autorizado y ejecuta:

```sql
ALTER USER django_user WITH PASSWORD 'REEMPLAZA_CON_UNA_CONTRASEÑA_SEGURA';
```

Esto cambia la contraseña dentro de PostgreSQL. Actualiza también `DB_PASSWORD` en `backend/.env` y recrea el contenedor de backend:

```bash
docker compose up -d --force-recreate backend
docker compose exec backend python manage.py check
```

No pongas contraseñas reales en scripts compartidos, commits ni capturas.

### Verificar la conexión de Django con PostgreSQL

```bash
docker compose exec backend python manage.py check
docker compose exec backend python manage.py migrate
docker compose exec db psql -U django_user -d django_db -c \
"SELECT current_user, current_database();"
```

Dentro de Docker, Django debe conectarse a `DB_HOST=db`, no a `localhost`.

## 9. Backups y restauración

### Exportar la base de datos a un archivo SQL

Desde la raíz del repositorio:

```bash
mkdir -p backups
docker compose exec -T db pg_dump -U django_user -d django_db > backups/django_db_$(date +%Y%m%d_%H%M%S).sql
```

Comprueba que el archivo no está vacío:

```bash
ls -lh backups/
```

Los backups pueden contener datos personales. No los subas a un repositorio público. Añade `backups/` al `.gitignore`.

### Restaurar desde un archivo SQL

La restauración puede sobrescribir o duplicar datos según el contenido del archivo. Haz primero un backup de la base actual y confirma el destino.

```bash
cat backups/ARCHIVO.sql | docker compose exec -T db psql -U django_user -d django_db
```

Para un volcado en formato custom generado con `pg_dump -Fc`, utiliza `pg_restore` en lugar de `psql`.

## 10. React, Vite y pnpm

Los comandos de esta sección se ejecutan dentro del contenedor `frontend`, salvo que se indique lo contrario.

```bash
docker compose exec frontend node --version
docker compose exec frontend pnpm --version
docker compose exec frontend pnpm install
docker compose exec frontend pnpm dev
docker compose exec frontend pnpm build
docker compose exec frontend pnpm lint
```

Los scripts disponibles dependen de `frontend/package.json`; compruébalos con:

```bash
docker compose exec frontend pnpm run
```

Si cambias `package.json`, `pnpm-lock.yaml` o el Dockerfile:

```bash
docker compose build --no-cache frontend
docker compose up -d frontend
docker compose logs -f frontend
```

URL habitual de Vite: `http://localhost:5173`.

### Variables de entorno del frontend

Vite expone al código cliente las variables con prefijo `VITE_`, por ejemplo `VITE_API_URL`. No guardes secretos en variables `VITE_`: se incluyen en el bundle que recibe el navegador.

Si el frontend corre en el navegador del Mac, una URL típica de API es:

```dotenv
VITE_API_URL=http://localhost:8000/api
```

Si Axios ya agrega `/api`, evita duplicar el prefijo en la URL base o en las rutas. Después de cambiar variables de Vite, reinicia/reconstruye el frontend según cómo se inyecten en la imagen.

## 11. Pruebas HTTP con curl y Postman

### Comprobar que el backend responde

```bash
curl -i http://localhost:8000/
curl -i http://localhost:8000/api/personas/
```

Un `404` en `/` puede ser normal si no existe una ruta raíz. Un `401` en una API protegida puede ser normal si no se envía un token.

### Obtener JWT con curl

Ajusta la ruta al endpoint real del proyecto:

```bash
curl -i -X POST http://localhost:8000/api/auth/token/ \
  -H "Content-Type: application/json" \
  -d '{"username":"TU_USUARIO","password":"TU_CONTRASEÑA"}'
```

### Consultar el usuario autenticado

Sustituye `ACCESS_TOKEN` por el token de acceso recibido:

```bash
curl -i http://localhost:8000/api/auth/me/ \
  -H "Authorization: Bearer ACCESS_TOKEN"
```

### Consultar Personas con JWT

```bash
curl -i http://localhost:8000/api/personas/ \
  -H "Authorization: Bearer ACCESS_TOKEN"
```

### Interpretar respuestas frecuentes

| Código | Significado habitual | Qué comprobar |
|---|---|---|
| `200 OK` | Petición correcta | Datos devueltos |
| `201 Created` | Recurso creado | Respuesta y base de datos |
| `400 Bad Request` | Datos inválidos | JSON, validaciones y campos requeridos |
| `401 Unauthorized` | Falta token o credenciales no válidas | Login, usuario activo, contraseña, token y encabezado Bearer |
| `403 Forbidden` | Usuario autenticado sin permisos suficientes | Permisos DRF |
| `404 Not Found` | Ruta o recurso no encontrado | `urls.py`, ID y prefijos `/api` |
| `500 Internal Server Error` | Error del servidor | `docker compose logs --tail=200 backend` |

### Diagnóstico de un `401` en React

1. En Postman, verifica que el login devuelve `200` y tokens.
2. Prueba `/api/auth/me/` con `Authorization: Bearer ACCESS_TOKEN`.
3. En el navegador abre **DevTools → Network**.
4. Comprueba que el login guarda el `access` token.
5. Inspecciona la petición a `/api/auth/me/` y confirma que envía el encabezado `Authorization`.
6. Verifica que el frontend y el backend usan la misma URL base y que el interceptor de Axios lee la misma clave de almacenamiento que escribe el login.

## 12. Problemas frecuentes

### `role "postgres" does not exist`

El clúster puede haberse inicializado con otro usuario, por ejemplo `django_user`. Conéctate usando el rol configurado:

```bash
docker compose exec db psql -U django_user -d django_db
```

### `password authentication failed`

1. Confirma `DB_USER`, `DB_PASSWORD`, `DB_HOST` y `DB_NAME` en `backend/.env`.
2. Recuerda que `DB_HOST=db` dentro de Compose.
3. Comprueba la contraseña del rol dentro de PostgreSQL.
4. Si la cambias con `ALTER USER`, actualiza también el `.env`.
5. Recrea backend y revisa los logs.

### `No active account found with the given credentials`

Comprueba desde Django que el usuario existe, que la contraseña es correcta y que `is_active=True`. No significa necesariamente que PostgreSQL esté desconectado.

### `ModuleNotFoundError`

Comprueba que el paquete está declarado en `backend/requirements.txt` y reconstruye:

```bash
docker compose build --no-cache backend
docker compose up -d backend
```

### `pnpm: not found`

Comprueba que el Dockerfile de frontend habilita Corepack (`RUN corepack enable`) y reconstruye:

```bash
docker compose build --no-cache frontend
docker compose up -d frontend
```

### Un cambio no aparece

```bash
docker compose logs --tail=100 SERVICIO
docker compose up -d --build SERVICIO
```

Si el cambio es de código y existe un volumen montado, revisa también la configuración de Vite/Django y los logs antes de reconstruir.

### El backend no conecta con la base

```bash
docker compose ps
docker compose logs --tail=200 db
docker compose logs --tail=200 backend
docker compose exec backend python manage.py check
```

No elimines el volumen como primera medida: diagnostica primero la conexión, las credenciales y las migraciones.

## 13. Git y GitHub

### Estado y cambios

```bash
git status
git diff
git diff --staged
git log --oneline -10
```

### Agregar y confirmar cambios

```bash
git add README.md
git add .
git status
git commit -m "docs: document Docker Django and PostgreSQL commands"
git push
```

Revisa siempre `git status` antes del commit para asegurarte de no incluir `.env`, tokens, contraseñas, bases de datos ni backups.

### Ejemplo de `.gitignore`

Adapta este archivo a tu proyecto:

```gitignore
# Secrets
.env
.env.*
!.env.example
**/.env
**/.env.*
!**/.env.example

# Python / Django
__pycache__/
*.py[cod]
*.sqlite3
.pytest_cache/
.coverage
htmlcov/
staticfiles/

# Node / frontend
node_modules/
dist/
.vite/

# Backups and local data
backups/
*.dump
*.sql

# Editors and OS
.DS_Store
.vscode/
.idea/
```

Si un secreto ya fue subido a GitHub, quitarlo del último commit no garantiza que desaparezca del historial. Rota/revoca el secreto y limpia el historial si corresponde.

## 14. Flujo habitual de trabajo

### Iniciar el proyecto

```bash
docker compose up -d
docker compose ps
docker compose logs --tail=50 backend
docker compose logs --tail=50 frontend
```

### Trabajar en modelos de Django

```bash
docker compose exec backend python manage.py makemigrations
docker compose exec backend python manage.py migrate
docker compose exec backend python manage.py check
```

### Probar autenticación

1. Confirma que `docker compose ps` muestra los servicios activos.
2. Obtén un token mediante el endpoint configurado.
3. Prueba `/api/auth/me/` enviando el token Bearer.
4. Prueba `/api/personas/` con el mismo token.
5. Si Postman funciona y React no, inspecciona el token almacenado y el interceptor de Axios en DevTools.

### Finalizar el trabajo sin borrar la base

```bash
docker compose stop
```

O elimina los contenedores y la red conservando los volúmenes nombrados:

```bash
docker compose down
```

Para volver a trabajar:

```bash
docker compose up -d
```

---

## Referencia rápida

| Necesidad | Comando |
|---|---|
| Estado de contenedores | `docker compose ps` |
| Logs de Django | `docker compose logs -f backend` |
| Logs de React | `docker compose logs -f frontend` |
| Logs de PostgreSQL | `docker compose logs -f db` |
| Shell de Django | `docker compose exec backend python manage.py shell` |
| Comprobar Django | `docker compose exec backend python manage.py check` |
| Migrar la base | `docker compose exec backend python manage.py migrate` |
| Crear superusuario | `docker compose exec backend python manage.py createsuperuser` |
| Entrar a PostgreSQL | `docker compose exec db psql -U django_user -d django_db` |
| Listar tablas SQL | `\dt` |
| Listar roles SQL | `\du` |
| Reconstruir backend | `docker compose build --no-cache backend` |
| Reconstruir frontend | `docker compose build --no-cache frontend` |
| Detener conservando datos | `docker compose down` |
| **Evitar si necesitas conservar la base** | **`docker compose down -v`** |
