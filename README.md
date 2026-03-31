# GestorTareas — Documentación de Aprendizaje

Cree este proyecto web desarrollado con **Laravel Full-Stack + Docker**, con el fin de repasar y estudiar nuevas tecnologias.
Este archivo es un registro de todo lo aprendido durante la configuración del proyecto,
pensado para repasar conceptos sin necesidad de buscarlos de nuevo.

---

## Tabla de contenido

1. [El modelo mental de Docker](#1-el-modelo-mental-de-docker)
2. [Los archivos clave](#2-los-archivos-clave)
3. [La estructura de carpetas y por qué](#3-la-estructura-de-carpetas-y-por-qué)
4. [Los servicios del proyecto](#4-los-servicios-del-proyecto)
5. [Qué es PHP-FPM](#5-qué-es-php-fpm)
6. [El orden de creación del proyecto](#6-el-orden-de-creación-del-proyecto)
7. [Rutina diaria de trabajo](#7-rutina-diaria-de-trabajo)
8. [Comandos esenciales](#8-comandos-esenciales)
9. [Cómo debuguear errores](#9-cómo-debuguear-errores)
10. [Docker Desktop — cuándo usarlo](#10-docker-desktop--cuándo-usarlo)

---

## 1. El modelo mental de Docker

**Sin Docker — tu máquina ES el servidor:**
Instalas PHP, MySQL, Node, etc. directamente en tu PC. Si alguien más clona el proyecto, tiene que instalar todo igual que tú, en la misma versión, y configurarlo igual.

**Con Docker — los contenedores SON el servidor:**
Cada servicio vive en un contenedor aislado. Tu PC solo necesita Docker instalado. Cualquier persona que clone el repo levanta el proyecto exactamente igual con un solo comando.

```
Sin Docker:  tu máquina = servidor
Con Docker:  contenedores = servidor, tu máquina = solo ejecuta Docker
```

**La ventaja clave:** portabilidad. El proyecto funciona igual en tu PC, en la de un compañero, y en producción.

---

## 2. Los archivos clave

### `Dockerfile`
Define **cómo construir una imagen personalizada**. Es una receta: "parte de esta imagen base, instala estas dependencias, configura esto".

En este proyecto se usa para crear la imagen de PHP con todas las extensiones que necesita Laravel y con Composer incluido.

```dockerfile
FROM php:8.3-fpm          # imagen base oficial de PHP
RUN apt-get install ...   # instala dependencias del sistema
RUN pecl install redis    # instala extensión de Redis
COPY composer ...         # copia Composer desde su imagen oficial
USER www-data             # corre como www-data, no como root (seguridad + permisos)
```

> **Por qué `USER www-data`:** PHP-FPM corre como `www-data`. Si el contenedor arranca como `root`, los archivos que crea (logs, caché) quedan con dueño `root` y PHP-FPM no puede escribirlos — esto causa errores 500. Definir `USER www-data` desde el inicio evita ese problema.

---

### `docker-compose.yml`
El archivo más importante. Define **todos los servicios** que forman la aplicación y cómo se comunican entre sí.

```yaml
services:
  app:    # PHP-FPM + Laravel
  nginx:  # servidor web
  mysql:  # base de datos
  redis:  # cache y colas
  node:   # Vite para assets (solo perfil dev)
```

Los servicios se comunican entre sí por nombre. Por ejemplo, Laravel se conecta a MySQL usando `mysql` como host — Docker resuelve ese nombre internamente.

---

### `docker/nginx/default.conf`
Configuración del servidor web. La línea más importante:

```nginx
fastcgi_pass app:9000;
```

Le dice a Nginx: *"las peticiones PHP pásamelas al contenedor `app` en el puerto `9000`"*. Ese es el puerto donde escucha PHP-FPM.

---

### Volúmenes — cómo ves tus cambios en tiempo real

En el `docker-compose.yml` esta línea conecta tu carpeta local con el contenedor:

```yaml
volumes:
  - ./src:/var/www/src
```

Significa: todo lo que está en `./src` en tu PC aparece en `/var/www/src` dentro del contenedor. Editas en VS Code → el cambio se refleja instantáneamente en el contenedor. No necesitas reconstruir la imagen para ver cambios en el código.

---

## 3. La estructura de carpetas y por qué

```
proyecto/
├── docker/                   # config de infraestructura (separada del código)
│   ├── php/
│   │   └── Dockerfile
│   └── nginx/
│       └── default.conf
├── src/                      # código Laravel
├── docker-compose.yml
└── .env                      # credenciales para los contenedores
```

**Por qué `docker/`:** agrupa la config de infraestructura para no mezclarla con el código de la aplicación. Sin esta carpeta todo estaría en la raíz mezclado con Laravel.

**Por qué `src/` y no `app/`:** Laravel ya tiene una carpeta `app/` internamente (donde van modelos, controllers, etc.). Usar `app/` como carpeta contenedora crearía `app/app/` — confuso. `src/` viene de *source* (código fuente) y no colisiona con nada.

**Importante:** esta estructura no la impone Docker. Es una convención que la comunidad adoptó porque es la más ordenada. Tú la decides.

---

## 4. Los servicios del proyecto

| Contenedor | Imagen | Puerto | Qué hace |
|---|---|---|---|
| `gestor_app` | PHP 8.3-FPM custom | 9000 (interno) | Ejecuta Laravel |
| `gestor_nginx` | nginx:alpine | 80 → localhost | Servidor web |
| `gestor_mysql` | mysql:8.0 | 3306 | Base de datos |
| `gestor_redis` | redis:alpine | 6379 | Cache y colas |
| `gestor_node` | node:20-alpine | — | Vite / assets (perfil dev) |

**Flujo de una petición:**
```
Navegador → Nginx (puerto 80) → PHP-FPM (puerto 9000) → Laravel → respuesta
```

---

## 5. Qué es PHP-FPM

**PHP-FPM = PHP FastCGI Process Manager**

Nginx no puede ejecutar PHP por sí solo. Cuando recibe una petición a un archivo `.php`, se la pasa a PHP-FPM que sí puede ejecutarlo.

```
Sin PHP-FPM:  Nginx no sabe qué hacer con archivos PHP
Con PHP-FPM:  Nginx delega la ejecución a PHP-FPM → resultado HTML → respuesta
```

**La alternativa:** `php artisan serve` incluye todo en uno (servidor + PHP). Sirve para desarrollo rápido local, pero no está hecho para producción ni para arquitecturas con múltiples servicios. Con Docker usamos la arquitectura real.

---

## 6. El orden de creación del proyecto

```
1. Crear repo en GitHub
2. Clonar en tu PC (carpeta vacía)
3. Crear manualmente los archivos Docker:
   - docker-compose.yml
   - docker/php/Dockerfile
   - docker/nginx/default.conf
   - .env
4. docker compose up --detach --build
   (construye las imágenes y levanta los contenedores)
5. docker compose exec app composer create-project laravel/laravel .
   (crea Laravel dentro del contenedor — los archivos aparecen en src/ gracias a los volúmenes)
6. Configurar src/.env de Laravel:
   DB_CONNECTION=mysql
   DB_HOST=mysql
   DB_DATABASE=gestor_db
   DB_USERNAME=gestor
   DB_PASSWORD=secret
   CACHE_STORE=redis
   REDIS_HOST=redis
7. docker compose exec app php artisan migrate
8. Abrir localhost:80 → Laravel funcionando
```

> **Nota:** No necesitas PHP, Composer ni Node instalados en tu PC. El contenedor tiene todo eso.

---

## 7. Rutina diaria de trabajo

| Acción | Sin Docker | Con Docker |
|---|---|---|
| Levantar proyecto | `php artisan serve` + `npm run dev` | `docker compose up --detach` |
| Acceder en browser | `localhost:8000` | `localhost:80` (o solo `localhost`) |
| Assets / Vite | `npm run dev` | `npm run dev` (igual, o con el contenedor node) |
| Apagar | `Ctrl+C` | `docker compose down` |
| Editar código | Normal en VS Code | Igual — los volúmenes sincronizan automático |

**Lo que cambia en comandos de Laravel:**

```bash
# Antes
php artisan make:model Tarea -mc
composer require algún-paquete

# Ahora — prefijo docker compose exec app
docker compose exec app php artisan make:model Tarea -mc
docker compose exec app composer require algún-paquete
```

`docker compose exec app` significa: *"entra al contenedor `app` y ejecuta esto"*.

---

## 8. Comandos esenciales

```bash
# Levantar todos los contenedores en background
docker compose up --detach

# Levantar y reconstruir imágenes (cuando cambias el Dockerfile)
docker compose up --detach --build

# Apagar todos los contenedores
docker compose down

# Ver estado de los contenedores
docker compose ps

# Ver logs de un servicio
docker compose logs app
docker compose logs nginx

# Entrar a la terminal de un contenedor
docker compose exec app bash

# Comandos de Laravel
docker compose exec app php artisan migrate
docker compose exec app php artisan make:model Tarea -mc
docker compose exec app php artisan cache:clear
docker compose exec app composer require algún-paquete

# Levantar también el contenedor de Node (Vite)
docker compose --profile dev up --detach
```

---

## 9. Cómo debuguear errores

### El proceso mental

```
1. ¿Qué servicio falló?        → identifica quién tiene el log
2. Lee el log de ese servicio  → busca el error real, no el del browser
3. Si no hay log               → el problema es previo a la aplicación
4. Formula una hipótesis       → "creo que es X porque..."
5. Verifica directamente       → no adivines, confirma con un comando
6. Aplica la corrección mínima → no arregles lo que no está roto
```

### Para errores 500 en Laravel, siempre empieza aquí:

```bash
# 1. Log de Laravel (el más útil)
docker compose exec app bash -c "tail -50 /var/www/src/storage/logs/laravel.log"

# 2. Log de Nginx (qué peticiones llegaron y con qué código)
docker compose logs nginx

# 3. Log de PHP-FPM (errores de ejecución PHP)
docker compose logs app
```

### Caso real que ocurrió en este proyecto

**Síntoma:** Error 500 al abrir localhost, pantalla de Symfony Exception.

**Diagnóstico:**
- Log de Nginx: confirma el 500
- Log de Laravel: **no existe** → Laravel no puede ni escribir el log
- Si no puede escribir el log, el problema es de permisos del sistema de archivos

**Verificación:**
```bash
docker compose exec app bash -c "ls -la /var/www/src/storage/"
# resultado: dueño "root root" — PHP-FPM corre como www-data, no puede escribir
```

**Causa:** Composer instaló Laravel como `root`. PHP-FPM corre como `www-data`. Sin permiso de escritura en `storage/`, Laravel explota antes de poder hacer cualquier cosa.

**Solución permanente:** agregar `USER www-data` en el Dockerfile para que todo arranque con el usuario correcto desde el inicio.

> **Lección clave:** un error que no genera log propio es más grave que uno que sí lo genera. Si Laravel no puede escribir el log, el problema es de infraestructura, no de código.

---

## 10. Docker Desktop — cuándo usarlo

Docker Desktop es la UI visual de Docker. No reemplaza la terminal, la complementa.

| Úsalo para | No lo uses para |
|---|---|
| Ver si los contenedores están corriendo o caídos | Ejecutar comandos dentro de contenedores |
| Revisar logs visualmente | Construir imágenes |
| Detener o reiniciar un contenedor rápido | El trabajo diario del proyecto |
| Ver cuánto pesan las imágenes | |
| Administrar volúmenes | |

**Regla práctica:** Docker Desktop es tu panel de monitoreo. La terminal es donde trabajas.
