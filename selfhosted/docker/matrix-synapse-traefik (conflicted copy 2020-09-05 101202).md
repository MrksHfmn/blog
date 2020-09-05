# Matrix Synapse mit Traefik unter Docker installieren

Ziel der Anleitung ist eine lauffähige Matrix Synapse Installation mit Traefik unter Docker.
Matrix Synapse läuft auf einem NAS mit einer dynamischen Adresse. Die Hauptdomain (example.de) ist einem anderen Server mit einer festen IP Adresse zugewiesen.
Damit das NAS von außen erreichbar ist, wird periodisch die aktuelle dynamische IP-Adresse bei DuckDNS hinterlegt.

Der Server wäre nun über diese Adresse (dynmatrix.duckdns.org) erreichbar, jedoch möchte ich meine Hauptdomain (example.de) wiederverwenden. Über einen CNAME DNS Eintrag wird dynmatrix.duckdns.org auf matrix.example.de umgeschlüsselt.

## Matrix Synapse
### DNS Einträge vorbereiten

Damit der Matrix Server im Internet überhaupt gefunden werden kann, wird ein DNS A Record benötigt.
```
matrix.example.de   3600  IN  A  123.12.12.1
```
Damit auch andere Matrix Server diesen Matrix Server finden, benötigen wir einen DNS SRV Record.
```
_dienst._proto.name        TTL   Klasse SRV  Priorität  Gewicht  Port  Ziel
_matrix._tcp.example.de    600   IN     SRV  0          10       443   matrix.example.de
```

### Federation konfigurieren
Normalerweise findet die Federation-Suche über Port 8448 statt, da ich aber keinen Port freigeben möchte, lässt sich das über einen Reverse-Proxy inkl. well-known Eintrag lösen.

#### Beispiel (nginx)
```nginx
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


