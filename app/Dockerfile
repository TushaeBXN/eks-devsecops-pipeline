FROM nginx:1.27-alpine

# Patch all Alpine packages to their latest available versions at build time.
# This closes known CVEs in base image packages (openssl, libxml2, expat, etc.)
# without floating to an unpinned "latest" tag.
RUN apk update && apk upgrade --no-cache

COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
