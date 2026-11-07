# Gateway: one public origin for It's a Plan's web app and api (see Caddyfile).
#
# The official Caddy image, pinned by tag and digest, plus this template's Caddyfile.
# GATEWAY_CADDY_IMAGE is a service variable; Railway passes service variables to Docker builds as
# build arguments, so an upgrade is a variable change. The image's user, entrypoint and
# command are kept: caddy run --config /etc/caddy/Caddyfile --adapter caddyfile.
ARG GATEWAY_CADDY_IMAGE=caddy:2.11.9-alpine@sha256:0bbb055bda6793c999bde78b1b5ce3e89f829fb031af2cf50056ea1246559fee
FROM ${GATEWAY_CADDY_IMAGE}

# /etc/caddy exists in the base image, so the copy keeps its directory permissions.
COPY Caddyfile /etc/caddy/Caddyfile

# Fail the build rather than the deploy on a Caddyfile error. The addresses here are
# placeholders; the service variables supply the real ones at runtime.
RUN PORT=8080 \
    ITSAPLAN_API_UPSTREAM=api.railway.internal:3000 \
    ITSAPLAN_WEB_UPSTREAM=web.railway.internal:3001 \
    caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
