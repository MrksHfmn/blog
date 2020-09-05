# Dynamische DNS Adressen über Duck DNS

Über [Duck DNS](https://www.duckdns.org/) lassen sich dynamische DNS Adressen einer statischen DNS Adresse (z.B. test123.duckdns.org) zuweisen. Dabei ist die Subdomain frei wählbar, soweit noch verfügbar.

Ein Update ist einfach durchzuführen:
```sh
echo url="https://www.duckdns.org/update?domains=test123&token=1234353466&ip=" | curl -k -o /tmp/duck.log -K -
```

Da man aber in der Regel nicht auf eine *.duckdns.org Adresse zugreifen möchte, kann per CNAME diese Subdomain auf eine eigene Subdomain umgeleitet werden, die auf eine eigene bezahlte Domain verweist (z.B. test123.example.de).


Diese lässt sich beim Webhoster eintragen:
```
CNAME test123.example.de test123.duckdns.org. 3600
```

Ein Abfrage mit dig bringt:

```
dig test123.example.de

; <<>> DiG 9.16.6-Debian <<>> test123.example.de
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12127
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;test123.example.de.             IN      A

;; ANSWER SECTION:
test123.example.de.      599     IN      CNAME   test123.duckdns.org.
test123.duckdns.org.     60      IN      A       12.123.123.123         <--- IP Adresse des heimischen NAS

;; Query time: 200 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Sa Sep 05 10:24:47 CEST 2020
;; MSG SIZE  rcvd: 97
```