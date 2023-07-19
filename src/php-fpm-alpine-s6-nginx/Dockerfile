ARG ALPINE_VERSION='3.18'
ARG PHP_VERSION='8.2'
ARG S6_OVERLAY_VERSION='v3.1.5.0'

ARG BASE_IMAGE=php:${PHP_VERSION}-fpm-alpine${ALPINE_VERSION}

FROM ${BASE_IMAGE} as s6_builder

ARG S6_OVERLAY_VERSION
ARG S6_DIR=/opt/s6/
ARG S6_SRC_URL="https://github.com/just-containers/s6-overlay/releases/download/${S6_OVERLAY_VERSION}"

RUN mkdir -p $S6_DIR; \
    # make system updates
    # apt-get update; \
    # install "xz-utils" and "wget"
    # apt-get install -yq $S6_SRC_DEP --no-install-recommends --no-install-suggests; \
    export SYS_ARCH=$(uname -m); \
    case "$SYS_ARCH" in \
        aarch64 ) export S6_ARCH='aarch64' ;; \
        arm64   ) export S6_ARCH='aarch64' ;; \
        armhf   ) export S6_ARCH='armhf'   ;; \
        arm*    ) export S6_ARCH='arm'     ;; \
        i4*     ) export S6_ARCH='i486'    ;; \
        i6*     ) export S6_ARCH='i686'    ;; \
        s390*   ) export S6_ARCH='s390x'   ;; \
        *       ) export S6_ARCH='x86_64'  ;; \
    esac; \
    untar (){ \
        echo "⏬ Downloading $1"; \
        wget --no-check-certificate -O- $1 | tar Jxp -C $S6_DIR; \
    }; \
    \
    # Download the latest v3.1.5.0/s6-overlay-noarch.tar.xz and tar it
    echo "⬇️ Downloading s6 overlay:${S6_ARCH}-${S6_VERSION} for ${SYS_ARCH}" \
        && untar ${S6_SRC_URL}/s6-overlay-noarch.tar.xz \
        && untar ${S6_SRC_URL}/s6-overlay-${S6_ARCH}.tar.xz

# main image
FROM ${BASE_IMAGE}
LABEL maintainer="@quadrubo"

ENV DEBIAN_FRONTEND="noninteractive" \
    S6_KEEP_ENV=1

COPY --from=s6_builder /opt/s6 /

ARG PHP_VERSION='8.2'
ARG ALPINE_PHP_VERSION='extensions'

ENV BUILD_PHP_VERSION=$PHP_VERSION \
    DEBIAN_FRONTEND=noninteractive \
    S6_BEHAVIOUR_IF_STAGE2_FAILS=2 \
    S6_CMD_WAIT_FOR_SERVICES_MAXTIME=0 \
    S6_VERBOSITY=1 \
    COMPOSER_ALLOW_SUPERUSER=1 \
    COMPOSER_HOME=/composer \
    COMPOSER_MAX_PARALLEL_HTTP=24 \
    WEBUSER_HOME="/var/www/html" \
    PUID=9999 \
    PGID=9999 \ 
    MSMTP_RELAY_SERVER_HOSTNAME="mailhog" \
    MSMTP_RELAY_SERVER_PORT="1025" \
    PHP_POOL_NAME="www" \
    PHP_PM_CONTROL=ondemand \
    PHP_PM_MAX_CHILDREN="20" \
    PHP_PM_START_SERVERS="2" \
    PHP_PM_MIN_SPARE_SERVERS="1" \
    PHP_PM_MAX_SPARE_SERVERS="3" \
    SSL_MODE="full"

ADD php-version-packages/${ALPINE_PHP_VERSION}.txt /tmp/php-packages.txt

RUN apk update \
    \
    # configure web user and group
    && addgroup -g $PGID webgroup \
    && adduser -D -s /bin/bash -h $WEBUSER_HOME -u $PUID -G webgroup webuser \
    \
    # install dependencies
    && apk add --no-cache \
        bash \
        curl \
        ca-certificates \
        nginx \
        fcgi \
    \
    # Installing php extensions
    && curl -sSLf \
        -o /usr/local/bin/install-php-extensions \
        https://github.com/mlocati/docker-php-extension-installer/releases/latest/download/install-php-extensions \ 
    && chmod +x /usr/local/bin/install-php-extensions \
    && install-php-extensions $(cat /tmp/php-packages.txt) \
    \
    # set symlink to version number for script management
    && ln -sf /usr/local/etc/php/${BUILD_PHP_VERSION}/ /usr/local/etc/php/current_version \
    \
    # ensure web permissions are correct
    && chown -R webuser:webgroup /var/www/html/ \
    \
    # install `php-fpm-healthcheck`
    && curl -o /usr/local/bin/php-fpm-healthcheck https://raw.githubusercontent.com/renatomefi/php-fpm-healthcheck/master/php-fpm-healthcheck \
    && chmod +x /usr/local/bin/php-fpm-healthcheck \
    \
    # cleanup
    && rm -rf /tmp/* /var/tmp/* usr/share/doc/* /var/www/html/* \
    && rm -f /etc/nginx/sites-enabled/default \
    && mkdir -p /etc/nginx/sites-enabled

# Copy over S6 configurations
COPY --chmod=755 etc/s6-overlay/ /etc/s6-overlay/

COPY etc/nginx/ /etc/nginx/

COPY usr/local/etc/php-fpm.d/ /usr/local/etc/php-fpm.d/

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

COPY --chmod=755 etc/s6-overlay/ /etc/s6-overlay/

WORKDIR /var/www/html

EXPOSE 80
EXPOSE 443

ENTRYPOINT [ "/init" ]