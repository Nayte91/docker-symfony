###> Dockerfile arguments
ARG NODE_VERSION
ARG COMPOSER_VERSION
ARG PHP_VERSION
ARG CADDY_VERSION
###< Dockerfile arguments

###> Backend
FROM composer:${COMPOSER_VERSION} AS backend__composer

FROM php:${PHP_VERSION}-fpm-alpine AS backend__php

RUN	apk update && \
#    docker-php-ext-install mysqli pdo pdo_mysql; docker-php-ext-enable pdo_mysql && \ # for MySQL
#    apk add --no-cache libpq postgresql-dev; docker-php-ext-install pdo_pgsql; apk del postgresql-dev && \ # for PostgreSQL
    apk add --no-cache libzip libzip-dev; docker-php-ext-install zip; apk del libzip-dev && \
    apk add --no-cache icu-libs icu-dev; docker-php-ext-install intl; apk del icu-dev && \
    apk add --no-cache --virtual .buildDeps build-base autoconf; pecl install apcu; docker-php-ext-enable apcu; pecl clear-cache; apk del .buildDeps

COPY ./system/backend/php-custom.ini /usr/local/etc/php/php.ini

COPY --from=backend__composer /usr/bin/composer /usr/bin/composer
WORKDIR /app

FROM backend__php AS backend__php--dev

RUN apk add --no-cache --virtual .buildDeps build-base autoconf; pecl install xdebug; docker-php-ext-enable xdebug; pecl clear-cache; apk del .buildDeps

FROM backend__php AS backend__php--prod

COPY . ./

RUN set -eux; \
    composer install --ignore-platform-reqs --no-interaction --prefer-dist; \
    composer dump-autoload --classmap-authoritative; \
    php bin/console secrets:generate-keys;

RUN chown -R www-data:www-data ./
###< Backend

###> Frontend
FROM node:${NODE_VERSION}-alpine AS frontend_npm--prod

WORKDIR /app

COPY ./package-lock.json ./package.json ./webpack.config.js ./
COPY --from=backend__php--prod /app/vendor/ ./vendor

RUN npm install

COPY ./assets/ ./assets/

RUN npm run build
###< Frontend

###> Server
FROM caddy:${CADDY_VERSION}-alpine AS server__caddy

WORKDIR /app

FROM server__caddy AS server__caddy--dev

FROM server__caddy AS server__caddy--prod

COPY --from=backend__php--prod /app/public/ ./public
COPY --from=frontend_npm--prod /app/public/build/ ./public/build
###< Server