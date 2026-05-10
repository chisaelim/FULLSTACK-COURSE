# Project — Project Initialization

## Table of Contents

- [Project Structure](#project-structure)
- [File Contents](#file-contents)
    - [`compose.yaml`](#composeyaml)
    - [`docker/laravel/Dockerfile`](#dockerlaraveldockerfile)
    - [`docker/laravel/entrypoint.development.sh`](#dockerlaravelentrypointdevelopmentsh)
    - [`docker/vuejs/Dockerfile`](#dockervuejsdockerfile)
    - [`docker/vuejs/entrypoint.development.sh`](#dockervuejsentrypointdevelopmentsh)
- [Environment Variables](#environment-variables)
- [Ports](#ports)
- [Troubleshooting](#troubleshooting)

---

## Project Structure

```
Project/
├── compose.yaml                        # Docker Compose configuration
├── .gitattributes                      # Enforces LF line endings and case sensitivity
├── .gitignore                          # Ignores OS/editor junk files
├── .editorconfig                       # Enforces consistent formatting across editors
├── .dockerignore                       # Specifies files to ignore in Docker builds
├── docker/
│   ├── laravel/
│   │   ├── Dockerfile                  # PHP 8.4-cli + Composer 2.9 image
│   │   └── entrypoint.development.sh  # Entrypoint script for laravel-service
│   └── vuejs/
│       ├── Dockerfile                  # Node 24.12.0 (Alpine) image
│       └── entrypoint.development.sh  # Entrypoint script for vuejs-service
├── laravel-app/                        # Laravel source code (mounted into laravel-container)
└── vuejs-app/                          # Vue.js source code (mounted into vuejs-container)
```

---

## File Contents

Copy each block below and paste it into the corresponding file.

---

### `compose.yaml`

```yaml
services:
    laravel-service:
        container_name: laravel-container
        build:
            context: .
            dockerfile: docker/laravel/Dockerfile
        working_dir: /var/www/html
        volumes:
            - ./laravel-app:/var/www/html
            - ./docker/laravel/entrypoint.development.sh:/usr/local/bin/entrypoint.development.sh
        ports:
            - "8000:8000"
        depends_on:
            - mysql-service
        command: ["bash", "/usr/local/bin/entrypoint.development.sh"]

    vuejs-service:
        container_name: vuejs-container
        build:
            context: .
            dockerfile: docker/vuejs/Dockerfile
        working_dir: /app
        volumes:
            - ./vuejs-app:/app
            - ./docker/vuejs/entrypoint.development.sh:/usr/local/bin/entrypoint.development.sh
        ports:
            - "5173:5173"
        depends_on:
            - laravel-service
        command: ["sh", "/usr/local/bin/entrypoint.development.sh"]

    mysql-service:
        image: mysql:8.0
        container_name: mysql-container
        environment:
            MYSQL_DATABASE: my_database_system
            MYSQL_USER: my_database_user
            MYSQL_PASSWORD: my_database_password
            MYSQL_ROOT_PASSWORD: root
            TZ: UTC
        volumes:
            - mysql_data:/var/lib/mysql

    phpmyadmin:
        image: phpmyadmin:5.2.2
        container_name: phpmyadmin-container
        depends_on:
            - mysql-service
        environment:
            UPLOAD_LIMIT: 50M
            PMA_HOST: mysql-service
            PMA_PORT: 3306
            PMA_USER: root
            PMA_PASSWORD: root
        ports:
            - "9000:80"
volumes:
    mysql_data:
```

---

### `docker/laravel/Dockerfile`

```dockerfile
FROM php:8.4-cli

WORKDIR /var/www/html

RUN apt-get update && apt-get install -y \
    git \
    unzip \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    libzip-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    default-mysql-client \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install pdo_mysql mbstring bcmath gd zip pcntl \
    && rm -rf /var/lib/apt/lists/*

COPY docker/laravel/*.sh /usr/local/bin/

COPY --from=composer:2.9 /usr/bin/composer /usr/bin/composer

COPY laravel-app /var/www/html

RUN chown -R www-data:www-data /var/www/html/bootstrap/cache \
 && chown -R www-data:www-data /var/www/html/storage \
 && chmod +x /usr/local/bin/*.sh

RUN composer install

EXPOSE 8000
```

---

### `docker/laravel/entrypoint.development.sh`

```bash
#!/bin/bash
set -e

# composer install
# wait $!
php artisan key:generate
wait $!
php artisan migrate
wait $!
php artisan serve --host=0.0.0.0 --port=8000
```

---

### `docker/vuejs/Dockerfile`

```dockerfile
FROM node:24.12.0-alpine

WORKDIR /app

COPY docker/vuejs/*.sh /usr/local/bin/

COPY vuejs-app /app

RUN chmod +x /usr/local/bin/*.sh

EXPOSE 5173
```

---

### `docker/vuejs/entrypoint.development.sh`

```sh
#!/bin/sh
set -e

# npm install
# wait $!
npm run dev -- --host=0.0.0.0 --port=5173
```

---

### `laravel-app/.env.example`

```dotenv
APP_NAME=Laravel
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost

APP_LOCALE=en
APP_FALLBACK_LOCALE=en
APP_FAKER_LOCALE=en_US

APP_MAINTENANCE_DRIVER=file
# APP_MAINTENANCE_STORE=database

# PHP_CLI_SERVER_WORKERS=4

BCRYPT_ROUNDS=12

LOG_CHANNEL=stack
LOG_STACK=single
LOG_DEPRECATIONS_CHANNEL=null
LOG_LEVEL=debug

DB_CONNECTION=mysql
DB_HOST=mysql-service
DB_PORT=3306
DB_DATABASE=my_database_system
DB_USERNAME=my_database_user
DB_PASSWORD=my_database_password

SESSION_DRIVER=database
SESSION_LIFETIME=120
SESSION_ENCRYPT=false
SESSION_PATH=/
SESSION_DOMAIN=null

BROADCAST_CONNECTION=log
FILESYSTEM_DISK=local
QUEUE_CONNECTION=database

CACHE_STORE=database
# CACHE_PREFIX=

MEMCACHED_HOST=127.0.0.1

REDIS_CLIENT=phpredis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_MAILER=log
MAIL_SCHEME=null
MAIL_HOST=127.0.0.1
MAIL_PORT=2525
MAIL_USERNAME=null
MAIL_PASSWORD=null
MAIL_FROM_ADDRESS="hello@example.com"
MAIL_FROM_NAME="${APP_NAME}"

AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=
AWS_USE_PATH_STYLE_ENDPOINT=false

VITE_APP_NAME="${APP_NAME}"
```

## Environment Variables

Laravel's database connection is configured in `laravel-app/.env.example`. Copy it to `laravel-app/.env` if it doesn't exist — the entrypoint runs `php artisan key:generate` which handles the app key automatically.

| Variable        | Value                  | Purpose                                                                  |
| --------------- | ---------------------- | ------------------------------------------------------------------------ |
| `DB_CONNECTION` | `mysql`                | Laravel uses the MySQL driver                                            |
| `DB_HOST`       | `mysql-service`        | Docker service name — resolved by Docker's internal DNS, not `localhost` |
| `DB_PORT`       | `3306`                 | MySQL default port                                                       |
| `DB_DATABASE`   | `my_database_system`   | Database name                                                            |
| `DB_USERNAME`   | `my_database_user`     | Application user                                                         |
| `DB_PASSWORD`   | `my_database_password` | Password for `my_database_user`                                          |

> `DB_HOST` must be `mysql-service` (the service name in `compose.yaml`), not `127.0.0.1`. Each container is isolated — `localhost` inside the Laravel container refers to itself, not the MySQL container.

---

## Ports

| Service    | Container port | Host port            | URL                                 |
| ---------- | -------------- | -------------------- | ----------------------------------- |
| Laravel    | 8000           | 8000                 | http://localhost:8000               |
| Vue.js     | 5173           | 5173                 | http://localhost:5173               |
| MySQL      | 3306           | none (internal only) | accessible only to other containers |
| phpMyAdmin | 80             | 9000                 | http://localhost:9000               |

---

## Troubleshooting

### Laravel fails on first `docker compose up` with a migration error

MySQL takes a few seconds to initialise on the very first run. Run `docker compose up` again — MySQL will be ready on the second start.

### `php artisan migrate` fails with "Access denied"

The credentials in `laravel-app/.env` must exactly match `MYSQL_USER`, `MYSQL_PASSWORD`, and `MYSQL_DATABASE` in the `mysql-service` service in `compose.yaml`.

### Vue.js shows a blank page or HMR does not work

The `--host=0.0.0.0` flag is required for Vite to be reachable outside the container. Check `docker/vuejs/entrypoint.development.sh`.

### Port already in use

Another process is using 8000, 5173, 3306, or 9000. Stop that process or change the host-side port (left number) in the `ports` mapping in `compose.yaml`.

### Data is lost after `docker compose down`

`docker compose down` keeps the `mysql_data` named volume — your data is safe.  
`docker compose down -v` **deletes the volume and all data**. Only use it for a clean reset.
