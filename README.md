# docker-supervisor-php-laravel
Usando Docker para queue Redis e Banco PostGres - Ajuste fino para PHP LARAVEL


# Documentação de Configuração e Solução de Problemas do Laravel com Docker

## 1. Configuração do Docker Compose

Criamos um arquivo `docker-compose.yml` para definir os serviços necessários para a aplicação Laravel, incluindo PHP, PostgreSQL, Redis e Nginx.

```yaml
services:
  nginx:
    container_name: nginx-mail-queue-service
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "5050:80"
    volumes:
      - ./:/var/www
      - ./production/nginx/:/etc/nginx/conf.d/
    depends_on:
      - app
    networks:
      - laravel

  app:
    container_name: app-mail-queue-service
    image: mail-queue-service-image:latest
    build:
      context: ./production/php
      dockerfile: Dockerfile
      args:
        USER: '${DOCKER_USER:-sail}'
    restart: unless-stopped
    working_dir: /var/www/
    volumes:
      - ./:/var/www
      - ./production/php/php.ini:/etc/php/8.3/fpm/conf.d/99-php.ini
      - ./production/php/www.conf:/usr/local/etc/php-fpm.d/www.conf
    entrypoint: ["/bin/sh", "-c", "composer install --no-dev --optimize-autoloader --classmap-authoritative && php artisan migrate:fresh --seed --force && php artisan optimize && php-fpm -R"]
    depends_on:
      - pgsql
    networks:
      - laravel

  worker_email:
    container_name: db-queue-mail-queue-service
    image: mail-queue-service-image:latest
    working_dir: /var/www/
    volumes:
      - ./:/var/www
      - ./production/php/php.ini:/etc/php/8.3/cli/conf.d/99-php.ini
      - ./production/php/supervisord.ini:/etc/supervisor.d/supervisord.ini
    restart: unless-stopped
    command: ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisor.d/supervisord.ini"]
    depends_on:
      - redis
    networks:
      - laravel

  pgsql:
    container_name: pgsql
    image: 'postgres:alpine'
    ports:
      - '5432:5432'
    environment:
      POSTGRES_DB: your_database_name
      POSTGRES_USER: your_username
      POSTGRES_PASSWORD: your_password
    volumes:
      - './.docker/pgsql/dbdata:/var/lib/postgresql/data'
    networks:
      - laravel

  redis:
    container_name: redis-queue-service
    image: 'redis:alpine'
    restart: unless-stopped
    ports:
      - '6379:6379'
    networks:
      - laravel

networks:
  laravel:
    driver: bridge
```

## 2. Configuração do Dockerfile

Criamos um arquivo `Dockerfile` para definir a imagem do PHP necessária para a aplicação Laravel.

```dockerfile
FROM php:8.3.2-fpm-alpine

# Set your user name, ex: user=carlos
ARG USER
ARG uid=1000

ENV DEBIAN_FRONTEND noninteractive
ENV TZ=America/Sao_Paulo

RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# Install runtime dependencies
RUN apk update && \
    apk add --no-cache tzdata bash supervisor postgresql-dev freetype libpng libjpeg-turbo libwebp libxpm libxml2 libzip oniguruma

# Install build dependencies and PHP extensions
RUN apk add --no-cache --virtual .build-deps \
    autoconf \
    gcc \
    g++ \
    make \
    git \
    curl \
    freetype-dev \
    libpng-dev \
    libjpeg-turbo-dev \
    libwebp-dev \
    libxpm-dev \
    libxml2-dev \
    libzip-dev \
    oniguruma-dev \
    linux-headers \
    && docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp \
    && docker-php-ext-install -j$(nproc) pgsql pdo_pgsql mbstring exif pcntl bcmath gd sockets zip \
    && pecl install redis \
    && docker-php-ext-enable redis \
    && apk del .build-deps \
    && rm -rf /var/cache/apk/* /tmp/* /usr/share/man /usr/share/doc

RUN pear update-channels
RUN pecl update-channels

# Get latest Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Create system user to run Composer and Artisan Commands
RUN addgroup -g $uid $USER && adduser -D -u $uid -G $USER -h /home/$USER $USER \
    && mkdir -p /home/$USER/.composer \
    && chown -R $USER:$USER /home/$USER

# Ensure necessary directories exist and set permissions
RUN mkdir -p /var/www/storage/logs /var/www/bootstrap/cache \
    && chown -R www-data:www-data /var/www/storage /var/www/bootstrap/cache \
    && chmod -R 775 /var/www/storage /var/www/bootstrap/cache

# Set working directory
WORKDIR /var/www

# Ensure PHP-FPM runs as root to properly set user directives
CMD ["php-fpm", "-R"]
```
## 3. Ajustes de Permissões

Para garantir que o Laravel possa acessar e gravar arquivos em diretórios específicos, é necessário ajustar as permissões dos diretórios `storage` e `bootstrap/cache`.

```bash
# Ensure necessary directories exist and set permissions
RUN mkdir -p /var/www/storage/logs /var/www/bootstrap/cache \
    && chown -R www-data:www-data /var/www/storage /var/www/bootstrap/cache \
    && chmod -R 775 /var/www/storage /var/www/bootstrap/cache
```

## 4. Comandos Artisan

Rodamos o comando php artisan optimize dentro do contêiner para otimizar a aplicação:
  
  ```bash
  docker-compose exec app-mail-queue-service sh
php artisan optimize  
  ```

## 5. Verificação e Testes

Verificamos os logs e o status do Supervisor para garantir que os jobs estavam sendo processados corretamente:
  
  ```bash
  docker-compose exec db-queue-mail-queue-service sh
supervisorctl status
cat /var/www/storage/logs/horizon.log
cat /var/www/storage/logs/schedule.log
  ```

## 6. Atualização das Variáveis de Ambiente	

Atualizamos as variáveis de ambiente no arquivo `.env` para refletir as configurações do Docker Compose:
  
  ```bash
  DB_CONNECTION=pgsql
DB_HOST=pgsql
DB_PORT=5432
DB_DATABASE=your_database_name
DB_USERNAME=your_username
DB_PASSWORD=your_password

REDIS_HOST=redis
REDIS_PASSWORD=null
REDIS_PORT=6379
  ```

## 7. Configuração do Supervisor

Criamos um arquivo `supervisord.ini` para configurar o Supervisor para executar os jobs de fila do Laravel:

```ini
[supervisord]
nodaemon=true

[program:worker_email]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/artisan queue:work --queue=email --sleep=3 --tries=3
autostart=true
autorestart=true
numprocs=8
redirect_stderr=true
stdout_logfile=/var/www/storage/logs/worker_email.log
```

## 8. Configuração do Nginx

Criamos um arquivo `default.conf` para configurar o Nginx para servir a aplicação Laravel:

```nginx
server {
    listen 80;
    index index.php index.html;
    server_name localhost;
    root /var/www/public;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

## 9. Configuração do Laravel

Atualizamos o arquivo `config/queue.php` para definir as conexões de fila para Redis e PostgreSQL:

```php
'connections' => [
    'redis' => [
        'driver' => 'redis',
        'connection' => 'default',
        'queue' => env('REDIS_QUEUE', 'default'),
        'retry_after' => 90,
        'block_for' => null,
    ],

    'database' => [
        'driver' => 'database',
        'table' => 'jobs',
        'queue' => 'default',
        'retry_after' => 90,
    ],
],
```

## 10. Configuração do Horizon

Atualizamos o arquivo `config/horizon.php` para definir as conexões de fila para Redis e PostgreSQL:

```php
'environments' => [
    'production' => [
        'supervisor-1' => [
            'connection' => 'redis',
            'queue' => ['default', 'email'],
            'balance' => 'simple',
            'processes' => 8,
            'tries' => 3,
        ],
    ],
],
```


# Resumo Final

Com essas configurações e ajustes, conseguimos configurar e executar com sucesso uma aplicação Laravel com Docker, incluindo a execução de jobs de fila usando Redis e PostgreSQL, e a configuração do Supervisor para gerenciar os processos de fila.

1. Configuração do Docker Compose: Definimos os serviços necessários para a aplicação Laravel, incluindo PHP, PostgreSQL, Redis e Nginx.
2. Configuração do Dockerfile: Definimos a imagem do PHP necessária para a aplicação Laravel, incluindo as extensões e dependências necessárias.
3. Ajustes de Permissões: Ajustamos as permissões dos diretórios `storage` e `bootstrap/cache` para garantir que o Laravel possa acessar e gravar arquivos corretamente.
4. Comandos Artisan: Rodamos o comando `php artisan optimize` dentro do contêiner para otimizar a aplicação Laravel.
5. Verificação e Testes: Verificamos os logs e o status do Supervisor para garantir que os jobs estavam sendo processados corretamente.
6. Atualização das Variáveis de Ambiente: Atualizamos as variáveis de ambiente no arquivo `.env` para refletir as configurações do Docker Compose.
7. Configuração do Supervisor: Criamos um arquivo `supervisord.ini` para configurar o Supervisor para executar os jobs de fila do Laravel.
8. Configuração do Nginx: Criamos um arquivo `default.conf` para configurar o Nginx para servir a aplicação Laravel.
9. Configuração do Laravel: Atualizamos o arquivo `config/queue.php` para definir as conexões de fila para Redis e PostgreSQL.
10. Configuração do Horizon: Atualizamos o arquivo `config/horizon.php` para definir as conexões de fila para Redis e PostgreSQL.



Espero que essas informações sejam úteis para você e que você possa aplicá-las com sucesso em seus próprios projetos Laravel com Docker.

Obrigado por ler e boa sorte com seus projetos! 🚀