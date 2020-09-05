# Matrix Synapse mit Traefik unter Docker installieren

Ziel der Anleitung ist eine lauffähige Matrix Synapse Installation mit Traefik unter Docker.
Matrix Synapse läuft auf einem NAS mit einer dynamischen Adresse. Die Hauptdomain (example.de) ist einem anderen Server mit einer festen IP Adresse zugewiesen.
Damit das NAS von außen erreichbar ist, wird periodisch die aktuelle dynamische IP-Adresse bei DuckDNS hinterlegt.

Der Server wäre nun über diese Adresse (dynmatrix.duckdns.org) erreichbar, jedoch möchte ich meine Hauptdomain (example.de) wiederverwenden. Über einen CNAME DNS Eintrag wird dynmatrix.duckdns.org auf matrix.example.de umgeschlüsselt.


## Traefik einrichten
### docker-compose.yaml
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
### traefik.yaml
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
### dynamic.yml
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
### dynamic.yml
Die acme.json Datei sollte inital angelegt werden mit:
```
touch acme.json && chmod 600 acme.json
```
### Traefik hochfahren
Nachdem alle Einstellungen ob angepasst sind, kann der Docker Container hochgefahren werden:
```
docker-compose pull && docker-compose up -d
```
Beim ersten Start wird versucht, alle nötigen Let's Encrypt Zertifikate zu holen. Das kann u.U. ein bisschen dauern.
Die Erneuerung der Zertikate funktioniert nun automatisch und wird durch Traefik getriggert.

## Matrix
### DNS Einträge vorbereiten

Damit der Matrix Server im Internet überhaupt gefunden wird, benötigt er einen DNS A Record.
```
matrix.example.de   3600  IN  A  123.12.12.1
```
Damit auch andere Matrix Server diesen Matrix Server finden, benötigen wir einen DNS SRV Record.
```
_dienst._proto.name        TTL   Klasse SRV  Priorität  Gewicht  Port  Ziel
_matrix._tcp.example.de    600   IN     SRV  0          10       443   matrix.example.de
```

### Federation konfigurieren
Normalerweise findet die Federation-Suche über Port 8448 statt, da ich aber keinen Port freigeben möchte, lässt sich das über einen Reverse-Proxy inkl. well-known Eintrag lösen. Da mein Matrix Synapse Server daheim läuft und mei

#### Beispiel (nginx)
```
server {
        listen 80 default_server;
        listen [IPv6]:80 default_server;

        server_name example.de www.example.de *.example.de;
        return 301 https://$host$request_uri;
}

server {
        listen 443 default_server http2 ssl;
        listen [IPv6]:443 ssl http2 default_server;

        server_name example.de www.example.de *.example.de;
        server_tokens off;

        root  /var/www/default/;

        access_log /var/log/nginx/default_access.log main;
        error_log  /var/log/nginx/default_error.log warn;


        location = /.well-known/matrix/server {
            access_log off;
            add_header Access-Control-Allow-Origin *;
            return 200 '{"m.server": "matrix.example.de:443"}';
        }
}
```


