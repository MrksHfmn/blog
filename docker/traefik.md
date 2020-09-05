# Traefik 
## docker-compose.yaml
```
version: '3.8'

services:
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - traefik
    ports:
      - 80:80
      - 443:443
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /opt/docker/traefik/config/acme.json:/acme.json
      - /opt/docker/traefik/config/traefik.yml:/traefik.yml:ro
      - /opt/docker/traefik/config/dynamic.yml:/dynamic.yml:ro
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.entrypoints=http"
      - "traefik.http.routers.traefik.rule=Host(`traefik.example.de`)" # <-- Edit
      - "traefik.http.middlewares.traefik-auth.basicauth.users=user:password123" # <-- Edit
      - "traefik.http.middlewares.traefik-https-redirect.redirectscheme.scheme=https"
      - "traefik.http.routers.traefik.middlewares=traefik-https-redirect"
      - "traefik.http.routers.traefik-secure.entrypoints=https"
      - "traefik.http.routers.traefik-secure.rule=Host(`traefik.example.de`)" # <-- Edit
      - "traefik.http.routers.traefik-secure.middlewares=traefik-auth"
      - "traefik.http.routers.traefik-secure.tls=true"
      - "traefik.http.routers.traefik-secure.tls.certresolver=http"
      - "traefik.http.routers.traefik-secure.service=api@internal"

networks:
  traefik:
    external: true
```
## traefik.yaml
```
api:
  dashboard: true

entryPoints:
  http:
    address: ":80"
  https:
    address: ":443"

log:
  level: DEBUG

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
  file:
    filename: /dynamic.yml

certificatesResolvers:
  http:
    acme:
      email: user@example.de # <-- Edit
      storage: acme.json
      httpChallenge:
        entrypoint: http
```
## dynamic.yml
```
tls:
  options:
    default:
      minVersion: VersionTLS12

      cipherSuites:
        - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
        - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
        - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
        - TLS_AES_128_GCM_SHA256
        - TLS_AES_256_GCM_SHA384
        - TLS_CHACHA20_POLY1305_SHA256

  curvePreferences:
    - CurveP521
    - CurveP384

  sniStrict: true

http:
  middlewares:
    secHeaders:
      headers:
        browserXssFilter: true
        contentTypeNosniff: true
        frameDeny: true
        sslRedirect: true
        stsIncludeSubdomains: true
        stsPreload: true
        stsSeconds: 31536000
        customFrameOptionsValue: "SAMEORIGIN"

    https-redirect:
      redirectScheme:
        scheme: https
```
## acme.json
Die acme.json Datei sollte inital angelegt werden mit:
```
touch acme.json && chmod 600 acme.json
```
## Traefik hochfahren
Nachdem alle Einstellungen ob angepasst sind, kann der Docker Container hochgefahren werden:
```
docker-compose pull && docker-compose up -d
```
Beim ersten Start wird versucht, alle nötigen Let's Encrypt Zertifikate zu holen. Das kann u.U. ein bisschen dauern.
Die Erneuerung der Zertikate funktioniert nun automatisch und wird durch Traefik getriggert.
