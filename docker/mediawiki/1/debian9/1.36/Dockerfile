#!/bin/bash
#
# Copyright (C) 2019  Google LLC
#
# This program is free software; you can redistribute it and/or
# modify it under the terms of the GNU General Public License
# as published by the Free Software Foundation; either version 2
# of the License, or (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA  02110-1301, USA.

FROM marketplace.gcr.io/google/php7-apache2:7.3

ENV MEDIAWIKI_MAJOR_VERSION 1.36
ENV MEDIAWIKI_VERSION 1.36.2
ENV MEDIAWIKI_SHA512 e4f18425dad547c547cac4926d5bcb2f996fcedc5df84b1a55773954c9c98ea0db075e65d5352ae92e2040b5154393c0dfb4d25e48a44272233008b61d29df78

ENV COMPOSER_VERSION 2.1.9
ENV COMPOSER_SHA256 4d00b70e146c17d663ad2f9a21ebb4c9d52b021b1ac15f648b4d371c04d648ba

ENV MEDIAWIKI_HOSTNAME localhost
ENV MEDIAWIKI_SITENAME my_first_wiki

ENV C2D_RELEASE 1.36.2

# System dependencies
RUN apt-get update; \
    apt-get install -y --no-install-recommends \
        gettext-base \
        git \
        librsvg2-bin \
        netcat \
        imagemagick \
        # Required for SyntaxHighlighting
        python3 \
        unzip \
        zip \
    && rm -rf /var/lib/apt/lists/*

# Install the PHP extensions we need
RUN savedAptMark="$(apt-mark showmanual)"; \
    \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        libicu-dev \
    ; \
    \
    docker-php-ext-install -j "$(nproc)" \
        intl \
        mbstring \
        mysqli \
        opcache \
    ; \
    \
    pecl install apcu-5.1.17; \
    docker-php-ext-enable \
        apcu \
    ; \
    \
    # reset apt-mark's "manual" list so that "purge --auto-remove" will remove all build dependencies
    apt-mark auto '.*' > /dev/null; \
    apt-mark manual $savedAptMark; \
    ldd "$(php -r 'echo ini_get("extension_dir");')"/*.so \
        | awk '/=>/ { print $3 }' \
        | sort -u \
        | xargs -r dpkg-query -S \
        | cut -d: -f1 \
        | sort -u \
        | xargs -rt apt-mark manual; \
    \
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
    rm -rf /var/lib/apt/lists/*

# Enable Short URLs
RUN a2enmod rewrite; \
    { \
        echo '<Directory /var/www/html>'; \
        echo '  RewriteEngine On'; \
        echo '  RewriteCond %{REQUEST_URI} !=/server-status'; \
        echo '  RewriteCond %{REQUEST_FILENAME} !-f'; \
        echo '  RewriteCond %{REQUEST_FILENAME} !-d'; \
        echo '  RewriteRule ^ %{DOCUMENT_ROOT}/index.php [L]'; \
        echo '</Directory>'; \
    } > "$APACHE_CONFDIR/conf-available/short-url.conf"; \
    a2enconf short-url

# set recommended PHP.ini settings
# see https://secure.php.net/manual/en/opcache.installation.php
RUN { \
        echo 'opcache.memory_consumption=128'; \
        echo 'opcache.interned_strings_buffer=8'; \
        echo 'opcache.max_accelerated_files=4000'; \
        echo 'opcache.revalidate_freq=60'; \
        echo 'opcache.fast_shutdown=1'; \
    } > /usr/local/etc/php/conf.d/opcache-recommended.ini

# Install composer binary
RUN set -e \
    && cd /usr/local/bin \
    && curl -o composer.phar "https://getcomposer.org/download/${COMPOSER_VERSION}/composer.phar" \
    && echo "${COMPOSER_SHA256} composer.phar" | sha256sum -c - \
    && chmod +x composer.phar

WORKDIR /mediawiki

# MediaWiki setup
RUN curl -fSL "https://releases.wikimedia.org/mediawiki/${MEDIAWIKI_MAJOR_VERSION}/mediawiki-${MEDIAWIKI_VERSION}.tar.gz" -o mediawiki.tar.gz; \
    echo "${MEDIAWIKI_SHA512} *mediawiki.tar.gz" | sha512sum -c -; \
    tar -x --strip-components=1 -f mediawiki.tar.gz; \
    rm mediawiki.tar.gz; \
    chown -R www-data:www-data extensions skins cache images \
    && mkdir -p /data

COPY docker-entrypoint.sh /docker-entrypoint.sh
COPY common_functions.sh /common_functions.sh
COPY custom_settings.php /mediawiki-config/custom_settings.php

WORKDIR /var/www/html

ENTRYPOINT [ "/docker-entrypoint.sh" ]
CMD ["apache2-foreground"]
