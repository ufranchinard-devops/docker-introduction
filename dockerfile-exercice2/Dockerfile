# ---- Etape 1 : build du site avec Vite ----
FROM node:26-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# ---- Etape 2 : image finale minimale, sans Node ----
FROM nginxinc/nginx-unprivileged:1.30-alpine
USER root
RUN apk upgrade --no-cache \
 && apk del --no-cache nginx-module-geoip nginx-module-image-filter nginx-module-njs nginx-module-xslt curl
COPY --from=build /app/dist /usr/share/nginx/html
USER 101
EXPOSE 8080