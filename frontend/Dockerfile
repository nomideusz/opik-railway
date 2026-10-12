# Opik's nginx + React UI. The stock image has no server block (upstream
# compose mounts one), so this adds it, plus a script that writes the
# basic-auth credentials at boot.
FROM ghcr.io/comet-ml/opik/opik-frontend:2.2.96

USER root
RUN mkdir -p /etc/nginx/railway && chown 1001:1001 /etc/nginx/railway
COPY --chown=1001:1001 default.conf.template /etc/nginx/templates/default.conf.template
COPY --chmod=0755 railway-env.envsh /docker-entrypoint.d/16-railway-env.envsh
USER 1001:1001

# NGINX_ENTRYPOINT_LOCAL_RESOLVERS makes the stock entrypoint export
# /etc/resolv.conf as NGINX_LOCAL_RESOLVERS (IPv6 bracketed) for the template.
ENV NGINX_PORT=5173 \
    NGINX_ENTRYPOINT_LOCAL_RESOLVERS=1
