# Lab 06: Reverse Proxy mit nginx und nativem ACME-Modul für die ISO-27001-Zertifizierung

![line](images/banner.png)

## Berufliche Aufgabenstellung

Das Unternehmen bereitet sich auf die Zertifizierung nach ISO/IEC 27001 vor. Im Rahmen der Auditvorbereitung hat ein externer Pentest die bestehende Infrastruktur überprüft und eine Feststellung dokumentiert, die vor dem eigentlichen Zertifizierungsaudit behoben werden muss.

Die TLS-Zertifikate werden vollständig automatisiert über Caddy bezogen. Welcher Zertifikatsanbieter dabei genutzt wird und über welches Protokoll, ist zwar in Caddys allgemeiner Dokumentation nachzulesen – aber an keiner Stelle im tatsächlich eingesetzten Caddyfile selbst erkennbar. Für Control **A.8.24 (Use of Cryptography)** der ISO 27001:2022 verlangt der Auditor, dass eine solche Entscheidung im eigenen, geprüften Konfigurationsbestand nachvollziehbar dokumentiert ist – ein impliziter Software-Standard, der sich mit einem künftigen Update jederzeit ändern könnte, ohne dass sich an der Konfiguration etwas ändert, reicht dafür nicht aus.

Im Rahmen des Risikobehandlungsplans gibt die Leitungsebene die Umsetzung frei; das Betriebsteam wird als Maßnahmenverantwortlicher (Risk Owner) für die Feststellung benannt, mit dem Ziel, sie vor dem Zertifizierungsaudit zu beheben: Caddy wird durch nginx mit dem nativen ACME-Modul abgelöst, wodurch Zertifikatsanbieter und -protokoll explizit konfiguriert und damit auditierbar dokumentiert werden.

---

## Einführung

In Lab 04 und Lab 05 hat Caddy die Zertifikatsverwaltung vollständig automatisch übernommen – komfortabel, aber für ein Audit ein Problem: *Welcher* Anbieter und *welches* Protokoll verwendet werden, steht nirgends im Caddyfile selbst, sondern ergibt sich stillschweigend aus Caddys eingebautem Standardverhalten. In diesem Lab lösen Sie Caddy durch **nginx** ab und konfigurieren die Zertifikatsausstellung mit dem seit 2025 verfügbaren, nativen `ngx_http_acme_module` explizit selbst – dessen Konfigurationsmodell verlangt zwingend einen benannten Issuer, ein Zertifikat „ohne jede Angabe" gibt es hier nicht.

## Lernziele

- ACME als offenes Protokoll verstehen und einen Zertifikatsanbieter (Issuer) bewusst und explizit konfigurieren
- nginx aus dem offiziellen Repository inklusive des nativen ACME-Moduls installieren und einrichten
- Mehrere virtuelle Hosts manuell als nginx-Server-Blöcke konfigurieren und den Unterschied zur automatisierten Caddy-Lösung nachvollziehen
- Das ISO-27001-Control A.8.24 mit konkreten technischen Maßnahmen in Verbindung bringen

## Voraussetzungen

| Anforderung | Details |
|---|---|
| **Server** | Debian stable (aktuell Trixie), öffentliche IPv4-Adresse (aus Lab 03) |
| **Caddy** | Aktuell noch aktiv – wird in diesem Lab abgelöst |
| **Dienst A / Dienst B** | Python-Testserver auf 127.0.0.1:8001/8002 aktiv (aus Lab 04) – nach einem Server-Neustart neu starten: `cd /srv/dienst-a && python3 -m http.server 8001 &` (analog 8002) |
| **Docker** | Installiert und betriebsbereit (aus Lab 05), inklusive der Container `nginx-web` und `nodered` |
| **SSH-Zugriff** | Key-Authentifizierung |
| **DNS** | Bestehende A-Records aus Lab 03–05 (`cockpit`, `dienst-a`, `dienst-b`, `www`, `dashboard`) |

---

## Inhalt

- [Hintergrundwissen](#hintergrundwissen)
  - [ACME als offenes Protokoll](#acme-als-offenes-protokoll)
  - [Das native ACME-Modul von nginx](#das-native-acme-modul-von-nginx)
- [Aufgaben](#aufgaben)
  - [Schritt 1: Caddy ablösen](#schritt-1-caddy-ablösen)
  - [Schritt 2: nginx aus dem offiziellen Repository installieren](#schritt-2-nginx-aus-dem-offiziellen-repository-installieren)
  - [Schritt 3: ACME-Modul aktivieren und Issuer konfigurieren](#schritt-3-acme-modul-aktivieren-und-issuer-konfigurieren)
  - [Schritt 4: Cockpit als nginx-Server-Block migrieren](#schritt-4-cockpit-als-nginx-server-block-migrieren)
  - [Schritt 5: Die übrigen Dienste analog migrieren](#schritt-5-die-übrigen-dienste-analog-migrieren)
  - [Schritt 6: Ergebnis im Browser prüfen](#schritt-6-ergebnis-im-browser-prüfen)
- [Reflexion und weiterführende Fragen](#reflexion-und-weiterführende-fragen)
- [Rückblick und Zusammenfassung](#rückblick-und-zusammenfassung)
- [Aufräumarbeiten](#aufräumarbeiten)
- [Autoren und Urheberrecht](#autoren-und-urheberrecht)

---

## Hintergrundwissen

### ACME als offenes Protokoll

ACME (*Automated Certificate Management Environment*) ist ein offenes, standardisiertes Protokoll zur automatisierten Ausstellung und Erneuerung von TLS-Zertifikaten – ursprünglich von Let's Encrypt entwickelt, heute von zahlreichen Zertifizierungsstellen unterstützt. Caddy, Traefik, Certbot und nginx sind allesamt nur unterschiedliche **ACME-Clients**, die mit einem austauschbaren **Issuer** (der Zertifizierungsstelle) sprechen. Diese Trennung von Werkzeug und Anbieter existiert bei Caddy zwar genauso – nur eben unsichtbar: Automatisches HTTPS funktioniert dort komplett ohne jede Angabe von CA oder Protokoll, weshalb in der Praxis oft nie jemand eine explizite Entscheidung trifft oder festhält. Das nginx-ACME-Modul erzwingt diese Angabe strukturell, weil ohne einen benannten `acme_issuer` gar kein Zertifikat ausgestellt wird – genau das machen Sie sich in diesem Lab zunutze.

### Das native ACME-Modul von nginx

Seit August 2025 bietet nginx mit `ngx_http_acme_module` ein offizielles, in Rust implementiertes Modul, das ACMEv2 direkt in der nginx-Konfiguration abbildet – sowohl für nginx Open Source als auch für NGINX Plus. Statt eines impliziten Automatismus definieren Sie einen benannten `acme_issuer`-Block mit der Server-URI der Zertifizierungsstelle, ordnen ihn explizit einem virtuellen Host über `acme_certificate` zu und richten einen eigenen Listener für die HTTP-01-Challenge ein. Aktuell wird ausschließlich die HTTP-01-Challenge unterstützt; DNS-01 ist für künftige Versionen angekündigt.

> Hinweise:
> - _HTTP-01-Challenge_ bedeutet, dass der ACME Issuer dem ACME Client eine Datei schickt, die der Client auf seinem Webserver bereitstellt. Damit zeigt der Client, dass er die Kontrolle über den Webserver unter dieser Adresse hat.
> - _DNS-01_ ist so ähnlich, nur muss hier der Client einen neuen TXT-Eintrag auf dem Nameserver zur angefragten Domain vornehmen.

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

## Aufgaben

### Schritt 1: Caddy ablösen

Stoppen und deaktivieren Sie Caddy. Das Caddyfile bleibt als Referenz erhalten, wird aber nicht mehr verwendet.

```bash
systemctl stop caddy
systemctl disable caddy
```

Prüfen Sie, dass die Ports 80 und 443 frei sind:

```bash
ss -tlnp | grep -E ':80 |:443 '   # Erzeugen: aktive Listener anzeigen | Filtern: nur Port 80/443
```

Die Ausgabe sollte leer sein.

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

### Schritt 2: nginx aus dem offiziellen Repository installieren

Wir installieren nginx nicht aus den Debian-Standardquellen, sondern aus dem offiziellen nginx-Repository – nur dort ist das aktuelle ACME-Modul verfügbar.

Zunächste ein paar Programme, die zur Installation nötig sind:
```bash
apt install -y curl gnupg2 ca-certificates lsb-release
```

Dann das Laden der Schlüssel:

```bash
curl -fsSL https://nginx.org/keys/nginx_signing.key \    # Erzeugen: Schlüssel-Datei laden
  | gpg --dearmor | tee /usr/share/keyrings/nginx-archive-keyring.gpg > /dev/null
    # Filtern/Umwandeln: in Binärformat wandeln | Ablegen: in Datei schreiben (Bildschirmausgabe unterdrückt)

echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/mainline/debian $(lsb_release -cs) nginx" \
  | tee /etc/apt/sources.list.d/nginx.list
    # Erzeugen: Repository-Zeile (inkl. Command Substitution $(lsb_release -cs)) | Ablegen: in Datei schreiben + anzeigen
```

> Dieselbe Grundstruktur **[Text erzeugen] → (filtern/umwandeln) → [ablegen]** wie im Exkurs zu Lab 02 – nur mit anderen Befehlen (`curl`/`echo` statt `echo`, `gpg --dearmor` statt `grep`, `tee` statt `logger`).

> **Tipp zur Fehlerbehebung:** Bricht `apt update` anschließend mit einer Meldung wie „The repository ... does not have a Release file" ab, prüfen Sie zunächst den tatsächlichen Codenamen Ihres Systems:
> ```bash
> cat /etc/os-release
> ```
> `VERSION_CODENAME` sollte für Debian stable aktuell `trixie` liefern. Weicht der von `lsb_release -cs` gelieferte Wert davon ab (z. B. auf einem Debian-Derivat mit eigenem Codenamen), tragen Sie den korrekten Debian-Codenamen händisch in die Repository-Zeile ein:
> ```bash
> rm /etc/apt/sources.list.d/nginx.list
> echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/mainline/debian trixie nginx" \
>   | tee /etc/apt/sources.list.d/nginx.list
> apt update
> ```
> (gleiche Befehlsstruktur wie oben – nur der Codename fest statt per Command Substitution eingetragen)

Paketliste aktualisieren und nginx mitsamt ACME-Modul installieren:

```bash
apt update
apt install -y nginx nginx-module-acme
```

Prüfen Sie die Installation:

```bash
nginx -v
systemctl enable --now nginx
curl http://localhost
```

Die letzte Zeile sollte die Standard-Begrüßungsseite von nginx liefern.

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

### Schritt 3: ACME-Modul aktivieren und Issuer konfigurieren

Das Modul muss als Erstes im Hauptkontext von nginx geladen werden:

```bash
nano /etc/nginx/nginx.conf
```

Fügen Sie als erste Zeile der Datei ein:

```nginx
load_module modules/ngx_http_acme_module.so;
```

> **Warum ganz oben?**  
> `load_module` muss im Hauptkontext stehen, bevor `events {}` oder `http {}` beginnen – nginx lädt dynamische Module beim Start, nicht zur Laufzeit.

Legen Sie ein Verzeichnis für die Zertifikatsdaten an und übergeben Sie es dem nginx-Benutzer:

```bash
mkdir -p /var/lib/nginx/acme-letsencrypt
chown -R nginx:nginx /var/lib/nginx/acme-letsencrypt
```

Erstellen Sie die ACME-Konfiguration:

```bash
nano /etc/nginx/conf.d/acme.conf
```

```nginx
resolver 8.8.8.8 valid=300s;

acme_shared_zone zone=acme_shared:1M;

acme_issuer letsencrypt {
    uri https://acme-v02.api.letsencrypt.org/directory;
    contact mailto:admin@<IHRE-DOMAIN>;
    state_path /var/lib/nginx/acme-letsencrypt;
    accept_terms_of_service;
}
```

> **Was passiert hier?**  
> `resolver` wird benötigt, damit das Modul die Adresse der Zertifizierungsstelle außerhalb der normalen Anfragebearbeitung selbst auflösen kann. `acme_issuer` definiert explizit, welcher Anbieter über welches Protokoll angesprochen wird – genau die Nachvollziehbarkeit, die im Audit gefordert ist. `accept_terms_of_service` ist eine bewusste, im Klartext sichtbare Zustimmung zu den Nutzungsbedingungen von Let's Encrypt.

Richten Sie zusätzlich einen Listener für die HTTP-01-Challenge ein, der gleichzeitig unverschlüsselte Anfragen auf HTTPS umleitet:

```bash
nano /etc/nginx/conf.d/challenge.conf
```

```nginx
# Virtueller Host für unverschlüsselte HTTP-Anfragen (Port 80)
server {
    # Lauscht auf Port 80. "default_server" macht diesen Block zum
    # Auffangbecken für ALLE Anfragen auf Port 80, die keinem anderen
    # Server-Block (z. B. cockpit.conf, dienst-a.conf) über server_name
    # zugeordnet werden können.
    listen 80 default_server;

    # Gilt für jeden angefragten Pfad - "/" fängt alles ab, was nicht
    # durch eine spezifischere location-Regel abgedeckt ist.
    location / {
        # Antwortet mit HTTP-Statuscode 301 (dauerhafte Weiterleitung)
        # und leitet auf dieselbe Adresse, aber über HTTPS, um.
        # $host        = der angefragte Hostname, z. B. cockpit.<IHRE-DOMAIN>
        # $request_uri = der vollständige ursprüngliche Pfad inkl. Query-String
        # Beispiel: http://dienst-a.<IHRE-DOMAIN>/seite?x=1
        #        -> https://dienst-a.<IHRE-DOMAIN>/seite?x=1
        return 301 https://$host$request_uri;
    }
}
```

Da `load_module` nur beim vollständigen Start eingelesen wird, genügt hier kein einfaches Reload:

```bash
nginx -t
systemctl restart nginx
```

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

### Schritt 4: Cockpit als nginx-Server-Block migrieren

Erstellen Sie die Konfiguration für Cockpit:

```bash
nano /etc/nginx/conf.d/cockpit.conf
```

```nginx
server {
    listen 443 ssl;
    server_name cockpit.<IHRE-DOMAIN>;

    acme_certificate letsencrypt;
    ssl_certificate $acme_certificate;
    ssl_certificate_key $acme_certificate_key;
    ssl_certificate_cache max=2;

    location / {
        proxy_pass https://127.0.0.1:9090;
        proxy_ssl_verify off;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```
#### Diese Konfiguration im Detail

`listen 443 ssl;` lässt diesen Server-Block verschlüsselte Verbindungen auf Port 443 annehmen – das `ssl` weist nginx an, hier einen TLS-Handshake durchzuführen, bevor irgendein Inhalt ausgeliefert wird. 

`server_name cockpit.<IHRE-DOMAIN>;` legt fest, dass dieser Block nur für Anfragen mit genau diesem Hostnamen zuständig ist – anhand des SNI beim TLS-Handshake entscheidet nginx, welcher von mehreren Server-Blöcken auf Port 443 eine konkrete Anfrage bearbeitet.

`acme_certificate letsencrypt;` ist die zentrale Anweisung: Sie teilt dem ACME-Modul mit, für `cockpit.<IHRE-DOMAIN>` (der Name wird automatisch aus `server_name` übernommen) jederzeit ein gültiges Zertifikat über den zuvor definierten Issuer `letsencrypt` bereitzuhalten. Das Modul überwacht die Restlaufzeit selbst und stößt die Erneuerung rechtzeitig vor **Ablauf der Gültigkeit des Zertifikats** an, ohne dass Sie eingreifen müssen.

`ssl_certificate $acme_certificate;` und `ssl_certificate_key $acme_certificate_key;` verweisen – im Unterschied zu sonst üblichen, statischen Dateipfaden – auf **Variablen**, die das ACME-Modul bereitstellt. Bei einer Erneuerung tauscht das Modul den Inhalt dieser Variablen aus; neue Verbindungen erhalten sofort das aktuelle Zertifikat, ohne dass ein `nginx -t` oder `systemctl reload` nötig wäre. Das unterscheidet diese Lösung von einer klassischen, statischen Konfiguration mit Certbot (alter Lösungsansatz für ACME), bei der nach jeder Erneuerung ein Reload erforderlich ist. 

`ssl_certificate_cache max=2;` ist eine reine Performance-Einstellung: nginx hält bis zu zwei geparste Zertifikate im Cache vor, unter anderem für die kurze Übergangsphase zwischen altem und neu ausgestelltem Zertifikat.

Damit diese Automatik dauerhaft funktioniert, müssen drei Voraussetzungen erhalten bleiben: Port 80 muss über `ufw` weiterhin für die HTTP-01-Challenge erreichbar sein, die DNS-A-Records müssen weiterhin auf diesen Server zeigen, und das Verzeichnis `/var/lib/nginx/acme-letsencrypt` darf nicht gelöscht werden, da dort Konto-Schlüssel und Status liegen.

Im `location`-Block regelt `proxy_pass https://127.0.0.1:9090;` die Weiterleitung an Cockpit. 
`proxy_ssl_verify off;` ist das nginx-Äquivalent zu Caddys `tls_insecure_skip_verify` aus Lab 04 – Cockpit betreibt intern weiterhin ein selbst signiertes Zertifikat auf Port 9090, das nur für diese lokale Verbindung akzeptiert werden muss; unkritisch, da diese Verbindung nie den Server verlässt. 
`proxy_set_header X-Real-IP $remote_addr;` gibt die tatsächliche Client-IP an Cockpit weiter – ohne diese Zeile würde Cockpit in seinen eigenen Logs nur die IP von nginx selbst sehen, was der im Audit geforderten Nachvollziehbarkeit widerspräche. Das wäre bei einem Zugriff aus einem Heimnetz über den Heimrouter die öffentliche IP des Heimrouters.
Die drei Zeilen `proxy_http_version 1.1;`, `proxy_set_header Upgrade $http_upgrade;` und `proxy_set_header Connection "upgrade";` ermöglichen WebSocket-Verbindungen durch den Proxy hindurch, die Cockpit für Live-Updates (z. B. Terminal, Ressourcen-Graphen) benötigt.


Validieren und laden Sie die Konfiguration:

```bash
nginx -t
systemctl reload nginx
```

Beobachten Sie die Ausstellung des Zertifikats:

```bash
journalctl -u nginx -f
```

Rufen Sie anschließend `https://cockpit.<IHRE-DOMAIN>` im Browser auf und prüfen Sie im Zertifikat: Aussteller Let's Encrypt, keine Sicherheitswarnung.

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

### Schritt 5: Die übrigen Dienste analog migrieren

Wiederholen Sie das Muster für die restlichen vier Dienste.

**Dienst A** (`/etc/nginx/conf.d/dienst-a.conf`):

```nginx
server {
    listen 443 ssl;
    server_name dienst-a.<IHRE-DOMAIN>;

    acme_certificate letsencrypt;
    ssl_certificate $acme_certificate;
    ssl_certificate_key $acme_certificate_key;
    ssl_certificate_cache max=2;

    location / {
        proxy_pass http://127.0.0.1:8001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Dienst B** benötigt zusätzlich Basic Authentication. Erzeugen Sie zunächst eine Passwortdatei:

```bash
apt install -y apache2-utils
htpasswd -B -c /etc/nginx/.htpasswd admin   # -B: bcrypt-Hash, -c: Datei neu anlegen, admin: Benutzername
```

> **Vergleich mit Caddy:** Caddy hat den bcrypt-Hash mit `caddy hash-password` direkt ins Caddyfile geschrieben. nginx benötigt dafür eine separate Datei und das Werkzeug `htpasswd` aus den Apache-Hilfsprogrammen – derselbe Hash-Algorithmus (bcrypt, Option `-B`), aber ein anderer Aufbewahrungsort.

`/etc/nginx/conf.d/dienst-b.conf`:

```nginx
server {
    listen 443 ssl;
    server_name dienst-b.<IHRE-DOMAIN>;

    acme_certificate letsencrypt;
    ssl_certificate $acme_certificate;
    ssl_certificate_key $acme_certificate_key;
    ssl_certificate_cache max=2;

    auth_basic "Geschützter Bereich";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass http://127.0.0.1:8002;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Unternehmenswebseite** (`/etc/nginx/conf.d/www.conf`), Container aus Lab 05 auf Host-Port 8080 (nur lokal gebunden, siehe Lab 05):

```nginx
server {
    listen 443 ssl;
    server_name www.<IHRE-DOMAIN>;

    acme_certificate letsencrypt;
    ssl_certificate $acme_certificate;
    ssl_certificate_key $acme_certificate_key;
    ssl_certificate_cache max=2;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**Node-RED Dashboard** (`/etc/nginx/conf.d/dashboard.conf`), Container aus Lab 05 auf Host-Port 1880 (nur lokal gebunden, mit aktivierter `adminAuth`, siehe Lab 05):

```nginx
server {
    listen 443 ssl;
    server_name dashboard.<IHRE-DOMAIN>;

    acme_certificate letsencrypt;
    ssl_certificate $acme_certificate;
    ssl_certificate_key $acme_certificate_key;
    ssl_certificate_cache max=2;

    location / {
        proxy_pass http://127.0.0.1:1880;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Validieren und laden Sie nach jeder Datei neu:

```bash
nginx -t
systemctl reload nginx
```

Verschaffen Sie sich am Ende einen Überblick über alle angelegten Konfigurationsdateien:

```bash
ls -la /etc/nginx/conf.d/
```

| Datei | Zuständigkeit |
|---|---|
| `acme.conf` | ACME-Issuer und Shared Zone |
| `challenge.conf` | HTTP-01-Challenge und Redirect auf HTTPS |
| `cockpit.conf` | Cockpit-Weboberfläche |
| `dienst-a.conf` | Öffentlich erreichbarer Demo-Dienst |
| `dienst-b.conf` | Demo-Dienst mit Basic Authentication |
| `www.conf` | Unternehmenswebseite (Docker-Container) |
| `dashboard.conf` | Node-RED-Dashboard (Docker-Container) |

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

### Schritt 6: Ergebnis im Browser prüfen

Rufen Sie alle fünf Domains auf und prüfen Sie jeweils das Zertifikat (Aussteller: Let's Encrypt, keine Browserwarnung):

```
https://cockpit.<IHRE-DOMAIN>
https://dienst-a.<IHRE-DOMAIN>
https://dienst-b.<IHRE-DOMAIN>
https://www.<IHRE-DOMAIN>
https://dashboard.<IHRE-DOMAIN>
```

Vergewissern Sie sich, dass Caddy nicht mehr läuft:

```bash
systemctl status caddy
```

Der Dienst muss als `inactive (dead)` gemeldet sein.

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

## Reflexion und weiterführende Fragen

**Zur ISO-27001-Vorbereitung**

- Welche Pentest-Feststellung lässt sich anhand der nginx-Konfiguration jetzt konkret nachweisen – und was genau würde ein Auditor dafür sehen wollen?
- Vergleichen Sie den Aufwand für eine neue Subdomain bei Caddy (Lab 04) mit dem Aufwand bei nginx mit ACME-Modul – was hat sich an Übersicht und Kontrolle verändert, was an Komfort?
- Warum ist die explizite Konfiguration des ACME-Issuers aus Audit-Sicht wichtiger als die automatische Lösung von Caddy?

---

## Rückblick und Zusammenfassung

### Was Sie erreicht haben

- Caddy abgelöst und durch nginx mit dem nativen ACME-Modul ersetzt
- ACME als offenes Protokoll verstanden und einen Zertifikatsanbieter explizit und nachvollziehbar konfiguriert
- Fünf virtuelle Hosts manuell als nginx-Server-Blöcke eingerichtet, inklusive Basic Authentication für einen geschützten Dienst
- Die technische Maßnahme dem ISO-27001-Control A.8.24 zugeordnet

### Zentrale Befehle dieser Übung

| Befehl | Bedeutung |
|---|---|
| `load_module modules/ngx_http_acme_module.so;` | ACME-Modul beim Start von nginx laden |
| `acme_issuer name { uri ...; }` | Zertifizierungsstelle explizit definieren |
| `acme_certificate issuer;` | Zertifikat für den aktuellen Server-Block anfordern |
| `nginx -t` | Konfiguration auf Syntaxfehler prüfen |
| `systemctl reload nginx` | Konfiguration ohne Prozessneustart übernehmen |

---

## Aufräumarbeiten

Dieser Reverse Proxy ist das Arbeitsergebnis des Labs – es empfiehlt sich, ihn **weiterlaufen zu lassen**. In Lab 07 werden Sie damit weiterarbeiten.

## Autoren und Urheberrecht

- Erstellt von: Michael Lotter, Florian Reichl
- Datum: 07/2026
- Version: v1.0

![line](images/banner.png)
<p align="center">
<a href="Lab_05.md"><img src="images/previous.png" width="150px"></a>
<a href="Lab_07.md"><img src="images/next.png" width="150px"></a>
</p>
