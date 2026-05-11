FROM docker.io/golang:1.25-bookworm AS CRON_BUILDER

ARG SUPERCRONIC_VERSION=v0.2.34

RUN apt-get update && \
    apt-get install -y git && \
    git clone https://github.com/aptible/supercronic.git && \
    cd supercronic && \
    git checkout $SUPERCRONIC_VERSION && \
    go build && \
    mv supercronic /usr/local/bin


# Build frontend assets with Node.js
FROM docker.io/node:22-bookworm AS FRONTEND_BUILDER

RUN corepack enable && corepack prepare pnpm@latest --activate

WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm run build


FROM docker.io/php:8.4-cli

ARG UNIT_VERSION=1.34.2

# Install Composer
COPY --from=docker.io/composer:2 /usr/bin/composer /usr/bin/composer

COPY --from=CRON_BUILDER /usr/local/bin/supercronic /usr/local/bin/supercronic

# Install system dependencies FIRST (before composer)
RUN apt-get update && \
    apt-get install -y supervisor ffmpeg curl gettext-base libfreetype6-dev libjpeg62-turbo-dev libpng-dev libwebp-dev libxpm-dev libpcre2-dev libicu-dev git unzip && \
    rm -rf /var/lib/apt/lists/* && \
    git clone https://github.com/nginx/unit.git && \
    cd unit && \
    git checkout $UNIT_VERSION && \
    ./configure --user=www-data --group=www-data && \
    ./configure php && \
    make && \
    make install && \
    cd .. && \
    rm -rf unit && \
    pecl install -o -f redis && \
    rm -rf /tmp/pear && \
    docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp --with-xpm && \
    docker-php-ext-install mysqli gd intl exif && \
    docker-php-ext-enable mysqli gd intl exif redis && \
    ln -s /dev/stdout /var/log/unit.log && \
    mkdir -p /usr/local/var/lib/unit /usr/local/var/run/unit /usr/local/var/log/unit

COPY docker/production/common/prepare_environment.sh /prepare_environment.sh
COPY docker/production/castopod/entrypoint.sh /entrypoint.sh
COPY docker/production/castopod/config.template.json /config.template.json
COPY docker/production/common/uploads.template.ini /uploads.template.ini
COPY docker/production/common/crontab.txt /crontab.txt
COPY docker/production/castopod/supervisord.conf /etc/supervisor/conf.d/supervisord.conf

# Copy application code
COPY . /var/www/castopod

# Copy built frontend assets from FRONTEND_BUILDER
COPY --from=FRONTEND_BUILDER /app/public/.vite /var/www/castopod/public/.vite
COPY --from=FRONTEND_BUILDER /app/public/assets /var/www/castopod/public/assets

# Install PHP dependencies
WORKDIR /var/www/castopod
RUN composer install --no-dev --optimize-autoloader --no-interaction

# Set permissions
RUN chmod 544 /entrypoint.sh && \
    chmod -R 750 /var/www/castopod && \
    chown -R root:www-data /var/www/castopod && \
    chown -R www-data:www-data /var/www/castopod/writable /var/www/castopod/public/media

VOLUME /var/www/castopod/public/media
EXPOSE 8000

ENTRYPOINT [ "sh", "-c" ]
CMD [ "/entrypoint.sh" ]
