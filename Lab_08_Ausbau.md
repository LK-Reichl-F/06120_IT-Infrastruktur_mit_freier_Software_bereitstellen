# Lab 08: Erweiterung – Weitere eigene Images auf Basis von debian:trixie-slim

![line](images/banner.png)

## Berufliche Aufgabenstellung

Die Vorbereitung auf die Zertifizierung nach ISO/IEC 27001 wirft eine weitere Frage auf, die bisher unbeantwortet blieb: Die Sensordaten, die künftig über die IoT-Pipeline gesammelt werden, existieren ausschließlich in der laufenden InfluxDB-Instanz – es gibt keinerlei dokumentierte Sicherung dieser Daten. Control **A.8.13 (Information Backup)** verlangt jedoch eine definierte, nachvollziehbare Backup-Routine mit klarem Zeitplan.

Ein Kollege aus dem Entwicklungsteam hat dafür bereits ein kleines Backup-Skript geschrieben und getestet – inklusive der nötigen Zeitsteuerung. Was fehlt, ist ein reproduzierbares, eigenständiges Docker-Image, das dieses Skript zuverlässig und nachvollziehbar betreibt, statt es manuell auf einem Server abzulegen. Genau das ist Ihre Aufgabe in dieser Übung: Sie übernehmen die fertigen Dateien und bauen daraus – mit dem in Lab 08 gelernten Muster – ein eigenes Image.

---

## Einführung

Diese Übung konzentriert sich bewusst **ausschließlich auf das Bauen eines eigenen Docker-Images** – nicht auf das Schreiben von Bash-Skripten oder auf Kenntnisse über `cron`/`crontab`. Das Backup-Skript sowie die Zeitsteuerung liegen bereits fertig geschrieben vor und werden Ihnen im Abschnitt [Bereitgestellte Dateien](#bereitgestellte-dateien) vollständig samt Erklärung zur Verfügung gestellt. Sie müssen diese Dateien nicht selbst verstehen oder verändern können – nur grob wissen, wofür sie da sind.

Ihre eigentliche Aufgabe ist es, mit dem in Lab 08 gelernten Muster (Betriebssystem-Image + `apt install`) ein eigenes **Dockerfile** zu schreiben, das diese vorgegebenen Dateien zu einem lauffähigen, eigenständigen Image zusammenfügt – inklusive der passenden Berechtigungen, Umgebungsvariablen und des Startbefehls.

## Lernziele

- Das apt-install-Muster aus Lab 08 (Betriebssystem-Image + `apt install`) selbstständig auf ein neues, eigenes Problem übertragen
- Vorgegebene Dateien (Skript, Konfiguration) mit `COPY` korrekt in ein eigenes Image einbinden und die passenden Zugriffsrechte setzen
- Ein Dockerfile eigenständig anhand einer Anforderungsliste entwerfen, statt einer Schritt-für-Schritt-Anleitung zu folgen
- Die eigene Lösung anhand klar definierter Abnahmekriterien selbst überprüfen
- Den Bezug zwischen einer technischen Lösung und einem ISO-27001-Control (A.8.13) herstellen

## Voraussetzungen

| Anforderung | Details |
|---|---|
| **Lab 08** | Vollständig durchgeführt – insbesondere das apt-install-Muster aus Schritt 2 und die Wegwerf-InfluxDB aus Schritt 4 |
| **Docker** | Installiert (aus Lab 05, Buildx aus Lab 08) |
| **Kenntnisse** | `FROM`/`RUN`/`COPY`/`ENV`/`CMD`, `docker build`/`run`/`network create` – **keine** Bash- oder cron-Kenntnisse nötig |

---

## Inhalt

- [Hintergrundwissen](#hintergrundwissen)
  - [Das Muster verallgemeinern](#das-muster-verallgemeinern)
  - [Was macht das Skript im Hintergrund? (kurz)](#was-macht-das-skript-im-hintergrund-kurz)
  - [ISO 27001: Control A.8.13 – Information Backup](#iso-27001-control-a813--information-backup)
- [Bereitgestellte Dateien](#bereitgestellte-dateien)
  - [backup.sh](#backupsh)
  - [entrypoint.sh](#entrypointsh)
  - [crontab.txt](#crontabtxt)
- [Übungsaufgabe](#übungsaufgabe)
  - [Ausgangslage](#ausgangslage)
  - [Anforderungen (Definition of Done)](#anforderungen-definition-of-done)
  - [Vorgehensvorschlag](#vorgehensvorschlag)
  - [Tipps, falls Sie nicht weiterkommen](#tipps-falls-sie-nicht-weiterkommen)
  - [Ihre Lösung testen](#ihre-lösung-testen)
- [Reflexion und weiterführende Fragen](#reflexion-und-weiterführende-fragen)
- [Wie geht es weiter?](#wie-geht-es-weiter)
- [Aufräumarbeiten](#aufräumarbeiten)
- [Autoren und Urheberrecht](#autoren-und-urheberrecht)

---

## Hintergrundwissen

### Das Muster verallgemeinern

Das Dockerfile-Skelett aus Lab 08 ist nicht an nginx gebunden – es funktioniert für praktisch jede Software, die als Debian-Paket verfügbar ist:

```dockerfile
FROM debian:trixie-slim

RUN apt-get update && \
    apt-get install -y --no-install-recommends <PAKET(E)> && \
    rm -rf /var/lib/apt/lists/*

# ... COPY eigener Dateien, ENV, CMD je nach Anwendungsfall
```

Nur `<PAKET(E)>` und das, was danach passiert, ändert sich mit dem Anwendungsfall:

| Ziel | apt-Paket(e) | Ergebnis |
|---|---|---|
| Statische Webseite | `nginx-light` | Portal v1 aus Lab 08 |
| Passwort-Hashes erzeugen | `apache2-utils` | Wie in Lab 06 zur Erstellung des bcrypt-Hashes für die nginx-Basic-Authentication (`htpasswd`) |
| MQTT-Testwerkzeug | `mosquitto-clients` | Vorgriff auf Lab 08: dort zum Testen des Mosquitto-Brokers (`mosquitto_pub`/`mosquitto_sub`) |
| Zeitgesteuerte Aufgaben | `cron`, `curl` | Container, der wiederkehrende Aufgaben nach Zeitplan ausführt – Gegenstand dieser Übung |

Genau dieses Skelett – Basis-Image wählen, benötigte Pakete per `apt install` einrichten, eigene Dateien per `COPY` einbinden – ist alles, was Sie in dieser Übung selbst schreiben müssen.

### Was macht das Skript im Hintergrund? (kurz)

Damit das Backup-Skript automatisch und regelmäßig läuft, ohne dass jemand es von Hand startet, kommt der Linux-Zeitplaner **cron** zum Einsatz. Für diese Übung müssen Sie cron **weder konfigurieren noch seine Syntax verstehen** – die passende Konfigurationsdatei liegt bereits fertig vor (siehe [Bereitgestellte Dateien](#bereitgestellte-dateien)).

Eine einzige Besonderheit ist trotzdem für den *Container*-Betrieb wichtig, nicht für cron selbst: cron muss im **Vordergrund** laufen, da ein Container beendet wird, sobald sein Hauptprozess (PID 1) endet – ein Prozess, der als Dienst im Hintergrund arbeitet (daemon), würde den Container sofort wieder stoppen lassen, genau wie bei nginx in Lab 08. Auch darum kümmert sich bereits eine vorbereitete Datei (`entrypoint.sh`) für Sie.

> **Hinweis:** cron-in-einem-Container ist eine pragmatische, weit verbreitete Lösung für einfache Zeitpläne – aber kein Selbstläufer. Produktivsysteme setzen dafür teils auf host-seitige Zeitsteuerung (`systemd`-Timer, der z. B. `docker exec` aufruft) oder dedizierte Scheduler-Container. Eine der Reflexionsfragen greift das auf – Sie müssen dafür aber keine cron-Syntax kennen.

### ISO 27001: Control A.8.13 – Information Backup

Control A.8.13 verlangt, dass Backups von Informationen, Software und Systemen gemäß einer vereinbarten Richtlinie erstellt, getestet und regelmäßig überprüft werden. Für die Aufgabenstellung in diesem Lab bedeutet das konkret: ein **definierter, automatisierter, protokollierter** Zeitplan – nicht ein gelegentliches, manuelles „Mal-eben-Sichern".

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

## Bereitgestellte Dateien

Die folgenden drei Dateien sind bereits fertig geschrieben und getestet. Legen Sie ein Projektverzeichnis an und speichern Sie alle drei darin ab – **unverändert**:

```bash
mkdir -p /srv/iot-backup
cd /srv/iot-backup
```

So spielen die drei Dateien beim Containerstart zusammen – Sie müssen dafür keine der Dateien selbst verstehen, nur den zeitlichen Ablauf:

```mermaid
sequenceDiagram
    participant Run as docker run (-e INFLUX_TOKEN=...)
    participant Entry as entrypoint.sh (PID 1)
    participant Cron as cron
    participant Backup as backup.sh
    participant Influx as InfluxDB
    participant Vol as /backup (Host-Verzeichnis)

    Run->>Entry: Container startet, setzt ENV-Variablen
    Entry->>Entry: sichert INFLUX_*-Variablen in /etc/backup-env
    Entry->>Cron: startet cron im Vordergrund
    loop alle 2 Minuten (crontab.txt)
        Cron->>Backup: startet backup.sh
        Backup->>Backup: liest /etc/backup-env
        Backup->>Influx: Flux-Abfrage per curl
        Influx-->>Backup: CSV-Antwort
        Backup->>Vol: komprimiert als iot-backup-&lt;Zeitstempel&gt;.csv.gz
    end
```

### backup.sh

```bash
nano backup.sh
```

```bash
#!/bin/bash
set -e

# Von entrypoint.sh beim Containerstart hinterlegte INFLUX_*-Variablen einlesen,
# da cron sie nicht automatisch zur Verfuegung stellt
source /etc/backup-env

ZEITSTEMPEL=$(date +%Y%m%d-%H%M%S)
ZIEL="/backup/iot-backup-${ZEITSTEMPEL}.csv.gz"

FLUX_QUERY="from(bucket: \"${INFLUX_BUCKET}\") |> range(start: -24h)"

curl -s -X POST "${INFLUX_URL}/api/v2/query?org=${INFLUX_ORG}" \
  -H "Authorization: Token ${INFLUX_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"query\": \"${FLUX_QUERY}\", \"dialect\": {\"annotations\": []}}" \
  | gzip > "${ZIEL}"

echo "$(date '+%Y-%m-%d %H:%M:%S') Backup geschrieben: ${ZIEL}" >> /var/log/backup.log
```

> **Wofür ist diese Datei da?**  
> Dieses Skript stellt dieselbe Flux-Abfrage und denselben HTTP-POST an InfluxDB wie das Portal aus Lab 08, Schritt 5.1 – nur dass die Antwort hier nicht in eine HTML-Tabelle gerendert, sondern komprimiert in eine mit Datum und Uhrzeit benannte Datei geschrieben wird. Sie müssen dieses Skript nicht verändern oder in allen Details verstehen – nur wissen, dass es zwei Programme benötigt, um zu funktionieren: `curl` (für die HTTP-Abfrage) und `gzip` (zum Komprimieren).

### entrypoint.sh

```bash
nano entrypoint.sh
```

```bash
#!/bin/bash
set -e

# cron startet Prozesse mit einer eigenen, minimalen Umgebung - die per
# "docker run -e" gesetzten Variablen kommen dort NICHT automatisch an.
# Deshalb werden sie hier einmalig beim Containerstart in eine Datei
# geschrieben, die backup.sh bei jedem Lauf selbst einliest.
printenv | grep -E '^INFLUX_' > /etc/backup-env

exec cron -f
```

> **Wofür ist diese Datei da?**  
> Dieses kurze Startskript löst zwei technische Probleme, bevor der eigentliche Zeitplan losläuft: Es rettet die beim `docker run` gesetzten `INFLUX_*`-Variablen in eine Datei, damit `backup.sh` später darauf zugreifen kann, und startet anschließend `cron` im Vordergrund (`-f`), damit der Container nicht sofort wieder beendet wird. Sie müssen dieses Skript nicht selbst schreiben können – nur wissen, dass es beim Containerstart ausgeführt werden muss.

### crontab.txt

```bash
nano crontab.txt
```

```
*/2 * * * * root /usr/local/bin/backup.sh >> /var/log/cron.log 2>&1

```

> **Wofür ist diese Datei da?**  
> Diese Zeile weist cron an, `backup.sh` alle 2 Minuten auszuführen (in einem Produktivsystem wäre ein selteneres Intervall üblich – 2 Minuten dienen hier nur dazu, das Ergebnis in dieser Übung zügig beobachten zu können). Sie müssen die genaue Schreibweise nicht verstehen können. Achten Sie beim Speichern nur darauf, die **Leerzeile am Dateiende** zu übernehmen – sie gehört zur Datei dazu.

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

## Übungsaufgabe

### Ausgangslage

Sie benötigen dieselbe Testumgebung wie in Lab 08, Schritt 4: ein Docker-Netzwerk und einen darin laufenden, mit Testdaten gefüllten InfluxDB-Container. Falls Sie diese Umgebung bereits abgebaut haben, legen Sie sie mit genau den dortigen Befehlen erneut an (Netzwerk `portal-test`, Container `influxdb`, Org `alp`, Bucket `iot`, dieselben drei Testpunkte im Measurement `umwelt`).

### Anforderungen (Definition of Done)

Schreiben Sie im Projektverzeichnis `/srv/iot-backup/` (neben den drei bereitgestellten Dateien) ein eigenes Dockerfile, das:

1. auf `debian:trixie-slim` aufbaut,
2. per `apt install` **ausschließlich** die Software installiert, die `backup.sh` und `entrypoint.sh` zum Laufen brauchen (siehe die Erklärungen oben – welche zwei Programme werden dort genannt? Welches dritte Werkzeug fehlt noch, damit die Zeitsteuerung selbst funktioniert?),
3. `backup.sh` und `entrypoint.sh` nach `/usr/local/bin/` kopiert und dort ausführbar macht,
4. `crontab.txt` nach `/etc/cron.d/iot-backup` kopiert und mit den Rechten `644` versieht,
5. das Verzeichnis `/backup` sowie leere Log-Dateien `/var/log/cron.log` und `/var/log/backup.log` anlegt,
6. per `ENV` dieselben Vorgabewerte wie beim Portal aus Lab 08 setzt (`INFLUX_URL`, `INFLUX_ORG`, `INFLUX_BUCKET`) – **ohne** `INFLUX_TOKEN`,
7. beim Containerstart `entrypoint.sh` ausführt.

Bauen Sie daraus ein Image `iot-backup:1.0` und starten Sie testweise einen Container: Das InfluxDB-Token wird – wie beim Portal in Lab 08 – ausschließlich zur Laufzeit per `-e` übergeben, niemals im Image gespeichert. Die Backup-Dateien landen über einen Bind Mount in einem Host-Verzeichnis.

### Vorgehensvorschlag

Kein verbindlicher Ablauf, aber eine mögliche Orientierung:

- Schreiben Sie zunächst nur den `FROM`- und `RUN apt-get install`-Teil und bauen Sie das Image testweise – so sehen Sie sofort, ob die Paketnamen stimmen, bevor Sie weiterarbeiten.
- Ergänzen Sie danach Schritt für Schritt die `COPY`-Zeilen, die Rechte-Anpassungen, `ENV` und zuletzt `CMD`.
- Denken Sie an ein Host-Verzeichnis für die Backup-Dateien, das Sie beim `docker run` per `-v` einbinden.

### Tipps, falls Sie nicht weiterkommen

> **Tipp 1 – Welche Pakete genau?** `backup.sh` braucht `curl` (siehe Erklärung oben). Für die Zeitsteuerung selbst brauchen Sie zusätzlich das Paket `cron`. `gzip`, das `backup.sh` ebenfalls verwendet, ist bereits Teil des Debian-Basissystems und muss nicht eigens installiert werden.

> **Tipp 2 – Rechte der crontab-Datei:** `crontab.txt` muss im Image unter `/etc/cron.d/iot-backup` liegen und die Zugriffsrechte `644` haben (nicht ausführbar!). Setzen Sie diese Rechte explizit mit einer eigenen `RUN chmod`-Zeile.

> **Tipp 3 – Skripte ausführbar machen:** Anders als `crontab.txt` müssen `backup.sh` und `entrypoint.sh` nach dem `COPY` erst mit `chmod +x` ausführbar gemacht werden.

> **Tipp 4 (optional) – Reihenfolge im Dockerfile:** Denken Sie an die Cache-Lektion aus Lab 08, Schritt 3: Wenn `RUN apt-get install` in einer eigenen, frühen Zeile steht und die `COPY`-Zeilen erst danach folgen, bleibt der langsamere Installations-Layer bei künftigen Änderungen an den Dateien im Cache erhalten.

### Ihre Lösung testen

```bash
docker ps                                       # iot-backup laeuft dauerhaft, kein Exited
ls -la /srv/backups                             # nach 2-4 Minuten sollten .csv.gz-Dateien erscheinen
zcat /srv/backups/iot-backup-*.csv.gz | head    # Inhalt der aeltesten Sicherung ansehen
docker exec iot-backup cat /var/log/backup.log  # Protokoll aller Laeufe
```

- Baut das Image ohne Fehler?
- Läuft der Container dauerhaft (`docker ps`, nicht `Exited`)?
- Erscheinen nach ein paar Minuten neue `.csv.gz`-Dateien im Host-Verzeichnis?
- Lässt sich eine Backup-Datei entpacken und enthält sie die erwarteten Messwerte?
- Zeigt das Log jeden Lauf nachvollziehbar an?

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

## Reflexion und weiterführende Fragen

- Wie würden Sie diesen Backup-Container in den Compose-Stack aus Lab 08 integrieren – welcher Dienst müsste dort ergänzt werden, und welche Volumes/Netzwerke bräuchte er?
- cron im Container gilt als pragmatische, aber diskutierte Lösung. Welche Alternativen gäbe es, denselben Zeitplan umzusetzen (Stichwort: Host-Cron, systemd-Timer, dedizierter Scheduler-Container)?
- Was müsste sich an Ihrer Lösung ändern, damit alte Backups automatisch nach einer definierten Aufbewahrungsfrist gelöscht werden?
- Welche weiteren Werkzeuge könnten Sie sich vorstellen, mit demselben Grundmuster (`FROM debian:trixie-slim` + `apt install`) als eigenes Image zu bauen?

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

## Wie geht es weiter?

Versuchen Sie zunächst, das Dockerfile selbstständig zu schreiben. Eine vollständige Musterlösung finden Sie in [`Lab_07_ausbau_musterloesung.md`](Lab_07_ausbau_musterloesung.md). Nutzen Sie sie, um Ihre eigene Lösung zu vergleichen oder falls Sie an einer Stelle nicht weiterkommen.

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

## Aufräumarbeiten

```bash
# Backup-Container und Image entfernen
docker stop iot-backup
docker rm iot-backup
docker rmi iot-backup:1.0

# Falls die Testumgebung nur für diese Übung neu angelegt wurde
docker stop influxdb
docker rm influxdb
docker network rm portal-test

# Eigene Projekt- und Backup-Verzeichnisse entfernen
rm -rf /srv/iot-backup
rm -rf /srv/backups
```

[↑ Zum Inhaltsverzeichnis](#inhalt)

## Autoren und Urheberrecht

- Erstellt von: Michael Lotter, Florian Reichl
- Datum: 07/2026
- Version: v1.0

![line](images/banner.png)
<p align="center">
<a href="Lab_07.md"><img src="images/previous.png" width="150px"></a>
<a href="Lab_08.md"><img src="images/next.png" width="150px"></a>
</p>
