# Versions
# https://hub.docker.com/r/serversideup/php/tags?name=8.3-fpm-nginx-alpine
ARG SERVERSIDEUP_PHP_VERSION=8.2-fpm-nginx-v2.2.1
# https://github.com/minio/mc/releases
ARG MINIO_VERSION=RELEASE.2024-11-05T11-29-45Z
# https://github.com/cloudflare/cloudflared/releases
ARG CLOUDFLARED_VERSION=2024.11.0
# https://www.postgresql.org/support/versioning/ - Can not updated automatically so keep it at 15
ARG POSTGRES_VERSION=15

FROM minio/mc:${MINIO_VERSION} AS minio-client

FROM serversideup/php:${SERVERSIDEUP_PHP_VERSION}

ARG TARGETPLATFORM
ARG CLOUDFLARED_VERSION
ARG MINIO_VERSION
ARG POSTGRES_VERSION

# Use build arguments for caching
ARG BUILDTIME_DEPS="dirmngr ca-certificates software-properties-common gnupg gnupg2 apt-transport-https curl"
ARG RUNTIME_DEPS="postgresql-client-$POSTGRES_VERSION php8.2-pgsql openssh-client git git-lfs jq lsof"

# Install dependencies
RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update && \
    apt-get install -y $BUILDTIME_DEPS && \
    curl -fSsL https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor | tee /usr/share/keyrings/postgresql.gpg > /dev/null && \
    echo deb [arch=amd64,arm64,ppc64el signed-by=/usr/share/keyrings/postgresql.gpg] http://apt.postgresql.org/pub/repos/apt/ jammy-pgdg main | tee -a /etc/apt/sources.list.d/postgresql.list && \
    apt-get update && \
    apt-get install -y $RUNTIME_DEPS && \
    apt-get -y autoremove && apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* /usr/share/doc/*

COPY --chmod=755 docker/dev/etc/s6-overlay/ /etc/s6-overlay/

COPY docker/dev/nginx.conf /etc/nginx/conf.d/custom.conf

RUN echo "alias ll='ls -al'" >>/etc/bash.bashrc && \
    echo "alias a='php artisan'" >>/etc/bash.bashrc

RUN mkdir -p /usr/local/bin

RUN --mount=type=cache,target=/root/.cache \
    /bin/bash -c "if [[ ${TARGETPLATFORM} == 'linux/amd64' ]]; then \
    echo 'amd64' && \
    curl -sSL https://github.com/cloudflare/cloudflared/releases/download/${CLOUDFLARED_VERSION}/cloudflared-linux-amd64 -o /usr/local/bin/cloudflared && chmod +x /usr/local/bin/cloudflared \
    ;fi"

RUN --mount=type=cache,target=/root/.cache \
    /bin/bash -c "if [[ ${TARGETPLATFORM} == 'linux/arm64' ]]; then \
    echo 'arm64' && \
    curl -L https://github.com/cloudflare/cloudflared/releases/download/${CLOUDFLARED_VERSION}/cloudflared-linux-arm64 -o /usr/local/bin/cloudflared && chmod +x /usr/local/bin/cloudflared \
    ;fi"

COPY --from=minio-client /usr/bin/mc /usr/bin/mc
RUN chmod +x /usr/bin/mc

RUN { \
    echo 'upload_max_filesize=256M'; \
    echo 'post_max_size=256M'; \
  } > /etc/php/current_version/cli/conf.d/upload-limits.ini
