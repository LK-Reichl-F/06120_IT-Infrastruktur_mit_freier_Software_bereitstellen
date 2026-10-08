# Lab 08: Erweiterung – Beispiellösung

![line](images/banner.png)

> Diese Datei enthält eine Beispiellösung zur Übungsaufgabe aus [`Lab_08_ausbau.md`](Lab_08_ausbau.md). Es gibt nicht die eine richtige Lösung – vergleichen Sie Ihre eigene Umsetzung mit dieser, statt sie als einzig gültigen Weg zu betrachten. Die Dateien `backup.sh`, `entrypoint.sh` und `crontab.txt` sind identisch mit den in der Übungsaufgabe bereitgestellten Dateien – die eigentliche Lösung der Übung ist das **Dockerfile**.

## Inhalt

- [Projektstruktur](#projektstruktur)
- [Das Dockerfile](#das-dockerfile)
- [Bauen und testen](#bauen-und-testen)
- [Design-Entscheidungen im Überblick](#design-entscheidungen-im-überblick)
- [Autoren und Urheberrecht](#autoren-und-urheberrecht)

---

## Projektstruktur

```
/srv/iot-backup/
├── Dockerfile          ← das ist Ihre eigentliche Aufgabe
├── entrypoint.sh       ← bereitgestellt, unverändert
├── backup.sh           ← bereitgestellt, unverändert
└── crontab.txt         ← bereitgestellt, unverändert
```

`entrypoint.sh`, `backup.sh` und `crontab.txt` entsprechen exakt den Dateien aus dem Abschnitt „Bereitgestellte Dateien" in [`Lab_08_ausbau.md`](Lab_08_ausbau.md#bereitgestellte-dateien) – hier nicht erneut abgedruckt.

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

## Das Dockerfile

```bash
nano Dockerfile
```

```dockerfile
FROM debian:trixie-slim

RUN apt-get update && \
    apt-get install -y --no-install-recommends cron curl && \
    rm -rf /var/lib/apt/lists/*

COPY entrypoint.sh /usr/local/bin/entrypoint.sh
COPY backup.sh /usr/local/bin/backup.sh
RUN chmod +x /usr/local/bin/entrypoint.sh /usr/local/bin/backup.sh

COPY crontab.txt /etc/cron.d/iot-backup
RUN chmod 0644 /etc/cron.d/iot-backup

RUN mkdir -p /backup && \
    touch /var/log/cron.log /var/log/backup.log

ENV INFLUX_URL=http://influxdb:8086 \
    INFLUX_ORG=alp \
    INFLUX_BUCKET=iot

CMD ["/usr/local/bin/entrypoint.sh"]
```

| Zeile | Bedeutung |
|---|---|
| `apt-get install ... cron curl` | Nur diese beiden Pakete werden zusätzlich benötigt: `curl` für die HTTP-Abfrage in `backup.sh`, `cron` für die Zeitsteuerung. `gzip` ist Teil des Debian-Basissystems (Priorität `required`) und muss nicht eigens installiert werden – auch die `-slim`-Variante entfernt nur Dokumentation und ähnlichen Ballast, keine essenziellen Werkzeuge |
| `COPY entrypoint.sh/backup.sh ...` + `chmod +x` | Die beiden Skripte werden an einen üblichen Ort für eigene ausführbare Programme kopiert und explizit ausführbar gemacht – ein `COPY` allein überträgt keine Ausführungsrechte |
| `COPY crontab.txt /etc/cron.d/iot-backup` | Legt die Cron-Konfiguration am von Debian erwarteten Ort ab; der Dateiname selbst ist frei wählbar |
| `chmod 0644 /etc/cron.d/iot-backup` | cron verlangt für Dateien in `/etc/cron.d/` genau diese Berechtigung – nicht ausführbar, nur für den Eigentümer beschreibbar |
| `mkdir -p /backup` | Zielverzeichnis für die Backup-Dateien – wird beim `docker run` per Bind Mount mit einem Host-Verzeichnis verbunden |
| `ENV INFLUX_URL=... INFLUX_ORG=... INFLUX_BUCKET=...` | Dieselben Vorgabewerte wie beim Portal aus Lab 08 – nur `INFLUX_TOKEN` fehlt bewusst, da es niemals im Image stehen darf |
| `CMD [".../entrypoint.sh"]` | Startet den bereitgestellten Wrapper, der die Umgebungsvariablen sichert und anschließend cron im Vordergrund startet |

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

## Bauen und testen

Testumgebung wie in Lab 08, Schritt 4 (bei Bedarf neu anlegen):

```bash
docker network create portal-test 2>/dev/null; true
docker run -d --name influxdb --network portal-test \
  -e DOCKER_INFLUXDB_INIT_MODE=setup \
  -e DOCKER_INFLUXDB_INIT_USERNAME=admin \
  -e DOCKER_INFLUXDB_INIT_PASSWORD='<SICHERES-PASSWORT>' \
  -e DOCKER_INFLUXDB_INIT_ORG=alp \
  -e DOCKER_INFLUXDB_INIT_BUCKET=iot \
  -e DOCKER_INFLUXDB_INIT_ADMIN_TOKEN='<TOKEN>' \
  influxdb:2
```

Image bauen:

```bash
docker build -t iot-backup:1.0 .
```

Host-Verzeichnis für die Backups anlegen und Container starten:

```bash
mkdir -p /srv/backups
docker run -d --name iot-backup \
  --restart unless-stopped \
  --network portal-test \
  -e INFLUX_TOKEN='<TOKEN>' \
  -v /srv/backups:/backup \
  iot-backup:1.0
```

Nach ein bis zwei Minuten prüfen:

```bash
docker ps                              # iot-backup laeuft dauerhaft, kein Exited
ls -la /srv/backups                    # erste .csv.gz-Datei sollte erscheinen
zcat /srv/backups/iot-backup-*.csv.gz | head    # Inhalt der aeltesten Sicherung ansehen
docker exec iot-backup cat /var/log/backup.log  # Protokoll aller Laeufe
docker exec iot-backup cat /var/log/cron.log    # Ausgabe/Fehler von cron selbst
```

> **Tipp zur Fehlerbehebung:** Bleibt `/srv/backups` leer, prüfen Sie zuerst `docker exec iot-backup cat /var/log/cron.log`. Erscheinen dort Fehlermeldungen zu leeren Variablen, prüfen Sie `docker exec iot-backup cat /etc/backup-env` – ist die Datei leer oder fehlt sie, wurde `entrypoint.sh` beim Containerstart nicht wie vorgesehen ausgeführt (meist ein Hinweis auf ein falsches `CMD` oder fehlende Ausführungsrechte). Bleibt die Log-Datei ganz leer, prüfen Sie mit `docker exec iot-backup ls -la /etc/cron.d/`, ob `iot-backup` dort mit den Rechten `-rw-r--r--` liegt.

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

## Design-Entscheidungen im Überblick

- **`cron` statt einer Endlosschleife mit `sleep`:** Beide Wege wären technisch möglich. `cron` wurde hier bewusst gewählt, weil es das in der Praxis geläufigere Werkzeug für zeitgesteuerte Aufgaben ist und zusätzlich das apt-install-Muster auf ein neues Paket überträgt.
- **`entrypoint.sh` als Wrapper statt cron direkt als `CMD`:** Nur über diesen Umweg gelangen die per `docker run -e` gesetzten Variablen überhaupt zu `backup.sh` – ohne ihn bliebe `INFLUX_TOKEN` beim cron-Lauf leer.
- **Komprimierung mit `gzip` statt unkomprimiert:** Spart Speicherplatz auf Dauer und ist mit einem einzigen Pipe-Befehl umsetzbar – kein zusätzliches Paket nötig, da `gzip` bereits im Basis-Image enthalten ist.
- **Kein automatisches Löschen alter Backups:** Bewusst nicht Teil dieser Musterlösung, sondern als Reflexionsfrage offengelassen – ein produktives System bräuchte eine explizite Aufbewahrungsfrist (Retention Policy), die je nach Kontext unterschiedlich ausfallen kann.

[↑ Zum Inhaltsverzeichnis](#inhalt)

## Autoren und Urheberrecht

- Erstellt von: Michael Lotter, Florian Reichl
- Datum: 07/2026
- Version: v1.0

![line](images/banner.png)
<p align="center">
<a href="Lab_08_ausbau.md"><img src="images/previous.png" width="150px"></a>
<a href="Lab_08.md"><img src="images/next.png" width="150px"></a>
</p>
