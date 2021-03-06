FROM php:7.3-apache

# Install php libs
RUN set -ex; \
    \
    aptMarkList="$(apt-mark showmanual)"; \
    \
    apt-get update; \
    apt-get install -y --no-install-recommends \
        libjpeg-dev \
        libpng-dev \
        libzip-dev \
        libxml2-dev \
        libc-client-dev \
        libkrb5-dev \
        libldap2-dev \
        libzmq3-dev \
        zlib1g-dev \
        git \
    ; \
    \
# Install php-zmq
    cd /home; \
    curl -fSL https://github.com/zeromq/libzmq/releases/download/v4.3.2/zeromq-4.3.2.tar.gz -o zeromq-4.3.2.tar.gz; \
    tar -xvzf zeromq-4.3.2.tar.gz; \
    rm zeromq-4.3.2.tar.gz; \
    cd /home/zeromq-4.3.2; \
    ./configure; \
    make; \
    make install; \
    ldconfig; \
    ldconfig -p | grep zmq; \
    cd /home; \
    git clone https://github.com/zeromq/php-zmq.git; \
    cd /home/php-zmq; \
    phpize && ./configure; \
    make; \
    make install; \
    \
# END Instalation php-zmq
    docker-php-ext-install pdo_mysql; \
    docker-php-ext-install zip; \
    docker-php-ext-configure gd --with-png-dir=/usr --with-jpeg-dir=/usr; \
    docker-php-ext-install gd; \
    docker-php-ext-configure imap --with-kerberos --with-imap-ssl; \
    docker-php-ext-install imap; \
    docker-php-ext-configure ldap --with-libdir=lib/x86_64-linux-gnu/; \
    docker-php-ext-install ldap; \
    docker-php-ext-install exif; \
    docker-php-ext-enable zmq; \
    \
# reset a list of apt-mark
    apt-mark auto '.*' > /dev/null; \
    apt-mark manual $aptMarkList; \
    ldd "$(php -r 'echo ini_get("extension_dir");')"/*.so \
        | awk '/=>/ { print $3 }' \
        | sort -u \
        | xargs -r dpkg-query -S \
        | cut -d: -f1 \
        | sort -u \
        | xargs -rt apt-mark manual; \
    \
    apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false

# Install required libs
RUN set -ex; \
    apt-get install -y --no-install-recommends \
        cron \
        unzip \
    ; \
    rm -rf /var/lib/apt/lists/*

# php.ini
RUN { \
	echo 'expose_php = Off'; \
	echo 'display_errors = Off'; \
	echo 'display_startup_errors = Off'; \
	echo 'log_errors = On'; \
	echo 'memory_limit=256M'; \
	echo 'max_execution_time=180'; \
	echo 'max_input_time=180'; \
	echo 'post_max_size=30M'; \
	echo 'upload_max_filesize=30M'; \
	echo 'date.timezone=UTC'; \
} > ${PHP_INI_DIR}/conf.d/espocrm.ini

# Set timezone
RUN echo "UTC" > /etc/timezone && dpkg-reconfigure --frontend noninteractive tzdata

RUN a2enmod rewrite;

ENV DOCUMENT_ROOT /var/www/html
ENV DEFAULT_OWNER www-data
ENV DEFAULT_GROUP www-data

ENV ESPOCRM_VERSION 5.9.3
ENV ESPOCRM_SHA256 b571d502de31a40d5e267c8658742f24ebc5c09245707306529b0d3f2e7c03bc

ENV ESPOCRM_UPGRADE_VERSION 5.8.5
ENV ESPOCRM_UPGRADE_URL https://www.espocrm.com/downloads/upgrades/EspoCRM-upgrade-5.8.5-to-5.9.3.zip
ENV ESPOCRM_UPGRADE_SHA256 188b1de2c5877737e2ecf78ed29c5e333fd6ee8108f44a400aca74b66d4b79c1

VOLUME ${DOCUMENT_ROOT}

RUN set -ex; \
    curl -fSL "https://www.espocrm.com/downloads/EspoCRM-5.9.3.zip" -o EspoCRM.zip; \
	echo "${ESPOCRM_SHA256} *EspoCRM.zip" | sha256sum -c -; \
    unzip -q EspoCRM.zip -d /usr/src; \
    mv "/usr/src/EspoCRM-5.9.3" /usr/src/espocrm; \
	rm EspoCRM.zip; \
    mkdir /usr/src/espocrm-upgrades; \
    chown -R ${DEFAULT_OWNER}:${DEFAULT_GROUP} /usr/src/espocrm /usr/src/espocrm-upgrades

COPY ./docker-entrypoint.sh /usr/local/bin/
ENTRYPOINT [ "docker-entrypoint.sh" ]

RUN (crontab -l -u ${DEFAULT_OWNER}; echo "* * * * * cd ${DOCUMENT_ROOT}; /usr/local/bin/php -f cron.php > /dev/null 2>&1") | crontab - -u ${DEFAULT_OWNER}

CMD ["sh", "-c", "cron && apache2-foreground"]
