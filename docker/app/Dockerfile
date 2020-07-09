FROM composer:latest AS composer
FROM spiralscout/roadrunner:1.8.1 AS roadrunner

FROM php:7.4-alpine

ENV \
    PHP_REDIS_VERSION="5.2.1" \
    COMPOSER_ALLOW_SUPERUSER="1" \
    COMPOSER_HOME="/tmp/composer" \
    OWN_SSL_CERT_DIR="/ssl-cert" \
    OWN_SSL_CERT_LIFETIME=1095 \
    PS1='\[\033[1;32m\]🐳  \[\033[1;36m\][\u@\h] \[\033[1;34m\]\w\[\033[0;35m\] \[\033[1;36m\]# \[\033[0m\]'

# persistent / runtime deps
ENV PHPIZE_DEPS \
    build-base \
    autoconf \
    libc-dev \
    pcre-dev \
    openssl \
    pkgconf \
    cmake \
    make \
    file \
    re2c \
    git \
    g++ \
    gcc \
    nano

# repmanent deps
ENV PERMANENT_DEPS \
    mysql-dev \
    gettext-dev \
    icu-dev \
    libintl \
    zlib-dev \
    libzip-dev \
    bzip2-dev \
    oniguruma-dev


COPY --from=composer /usr/bin/composer /usr/bin/composer
COPY --from=roadrunner /usr/bin/rr /usr/bin/rr

RUN set -xe \
    && apk add --no-cache ${PERMANENT_DEPS} \
    && apk add --no-cache --virtual .build-deps ${PHPIZE_DEPS} \
    # https://github.com/docker-library/php/issues/240
    && apk add --no-cache --repository http://dl-3.alpinelinux.org/alpine/edge/community gnu-libiconv \
    && docker-php-ext-configure mbstring --enable-mbstring \
    && docker-php-ext-configure opcache --enable-opcache \
    && docker-php-ext-configure mysqli --with-mysqli \
    && docker-php-ext-configure pdo_mysql --with-pdo-mysql \
    && docker-php-ext-configure bcmath --enable-bcmath \
    && docker-php-ext-configure pcntl --enable-pcntl \
    && docker-php-ext-configure intl --enable-intl \
    && docker-php-ext-install -j$(nproc) \
        zip \
        mysqli \
        pdo_mysql \
        mbstring \
        sockets \
        gettext \
        opcache \
        bcmath \
        pcntl \
        intl \
# To do:
# Install and enable igbinary
#    && pecl install igbinary \
    && git clone --branch ${PHP_REDIS_VERSION} https://github.com/phpredis/phpredis /tmp/phpredis \
       && cd /tmp/phpredis \
       && phpize  \
#       && ./configure --enable-redis-igbinary \
       && ./configure \
       && make  \
       && make install \
       && make test \
       && echo 'extension=redis.so' > /usr/local/etc/php/conf.d/redis.ini \
    && ( mkdir -p ${OWN_SSL_CERT_DIR} \
        && cd ${OWN_SSL_CERT_DIR} \
        && openssl genrsa -passout pass:x -out self-signed.key 2048 \
        && cp self-signed.key self-signed.key.orig \
        && openssl rsa -passin pass:x -in self-signed.key.orig -out self-signed.key \
        && openssl req -new -key self-signed.key -out cert.csr \
            -subj "/C=RU/ST=RU/L=Somewhere/O=SomeOrg/OU=IT Department/CN=example.com" \
        && openssl x509 -req -days ${OWN_SSL_CERT_LIFETIME} -in cert.csr -signkey self-signed.key -out self-signed.crt \
        && ls -lh ) \
    && apk del .build-deps \
    && rm -rf /scripts \
    && mkdir /scripts \
    && mkdir -p /scripts/aliases \
    && rm -rf /app /home/user ${COMPOSER_HOME} /var/cache/apk/* \
    && mkdir /app /home/user ${COMPOSER_HOME} \
    && composer global require 'hirak/prestissimo' --no-interaction --no-suggest --prefer-dist \
    && ln -s /usr/bin/composer /usr/bin/c



#COPY ./docker/app/etc/php/php.ini /usr/local/etc/php/php.ini
COPY ./docker/app/etc/php/opcache.ini /usr/local/etc/php/conf.d/opcache.ini
COPY ./docker/app/app-entrypoint.sh /app-entrypoint.sh
COPY ./docker/app/env_secrets_expand.sh /env_secrets_expand.sh

ENV PATH /scripts:/scripts/aliases:$PATH
COPY ./docker/app/aliases/* /scripts/aliases/

WORKDIR /app

COPY ./composer.* /app/
#COPY ./auth.* /app/

RUN set -xe \
    && composer install --no-interaction --no-ansi --no-suggest --prefer-dist  --no-autoloader --no-scripts \
    && composer install --no-dev --no-interaction --no-ansi --no-suggest --prefer-dist  --no-autoloader --no-scripts \
    && chmod -R 777 /home/user ${COMPOSER_HOME}

COPY . /app

RUN set -xe \
    && chmod +x /app-entrypoint.sh \
    && composer --version \
    && php -v \
    && php -m \
    && rr -h \
    && composer dump

EXPOSE 80
EXPOSE 443
VOLUME ["/app"]


# DO NOT OVERRIDE ENTRYPOINT IF YOU CAN AVOID IT! @see <https://github.com/docker/docker.github.io/issues/6142>
ENTRYPOINT ["/app-entrypoint.sh"]
CMD ["rr", "serve", "-c", "/app/.rr.yml"]
