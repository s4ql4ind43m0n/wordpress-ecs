FROM php:8.3-fpm

RUN apt-get update && apt-get install -y \
    libfreetype6-dev \
    libjpeg62-turbo-dev \
    libpng-dev \
    libzip-dev \
    unzip \
    curl \
    && docker-php-ext-configure gd \
        --with-freetype \
        --with-jpeg \
    && docker-php-ext-install \
        mysqli \
        pdo \
        pdo_mysql \
        gd \
        zip \
        opcache \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /var/www/html

COPY php.ini /usr/local/etc/php/conf.d/custom.ini

RUN chown -R www-data:www-data /var/www/html

EXPOSE 9000

CMD ["php-fpm", "-F"]
