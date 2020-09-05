# Matrix Synapse mit Traefik unter Docker installieren

Ziel der Anleitung ist eine lauffähige Matrix Synapse Installation mit Traefik unter Docker.
Matrix Synapse läuft auf einem NAS mit einer dynamischen Adresse. Die Hauptdomain (example.de) ist einem anderen Server mit einer festen IP Adresse zugewiesen.
Damit das NAS von außen erreichbar ist, wird periodisch die aktuelle dynamische IP-Adresse bei DuckDNS hinterlegt.

Der Server wäre nun über diese Adresse (dynmatrix.duckdns.org) erreichbar, jedoch möchte ich meine Hauptdomain (example.de) wiederverwenden. Über einen CNAME DNS Eintrag wird dynmatrix.duckdns.org auf matrix.example.de umgeschlüsselt.
