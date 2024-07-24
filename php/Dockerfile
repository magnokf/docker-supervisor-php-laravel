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

# Copy existing application directory contents
#COPY . /var/www

# Ensure necessary directories exist and set permissions
RUN mkdir -p /var/www/storage/logs /var/www/bootstrap/cache \
    && chown -R www-data:www-data /var/www/storage /var/www/bootstrap/cache \
    && chmod -R 775 /var/www/storage /var/www/bootstrap/cache

# Set working directory
WORKDIR /var/www

# Ensure PHP-FPM runs as root to properly set user directives
CMD ["php-fpm", "-R"]
