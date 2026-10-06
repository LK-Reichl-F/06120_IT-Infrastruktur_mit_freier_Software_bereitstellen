# Lab 02: systemd Unit-Dateien erstellen und verstehen

![line](images/banner.png)

## Einführung

Ein Kollege hat ein kleines Bash-Skript geschrieben, das Netzwerkstatistiken ins Syslog schreibt. Das Skript liegt fertig auf dem Server – aber es startet nicht automatisch, läuft nicht dauerhaft und niemand weiß, ob es nach einem Neustart noch aktiv ist. Es gibt keine Unit-Datei.

Genau das ist eine typische Situation in der Praxis: Eigenentwicklungen, heruntergeladene Tools oder manuell installierte Dienste bringen keine Unit-Datei mit. Wer systemd nutzen will, muss sie selbst schreiben.

## Lernziele

- Die Struktur einer systemd Unit-Datei kennen und anwenden
- Einen eigenen Dienst ohne vorhandene Unit-Datei in systemd integrieren
- Den Unterschied zwischen einem fehlenden und einem fehlerhaften Dienst diagnostizieren
- Eine Unit-Datei schrittweise verbessern und das Ergebnis mit `journalctl` überprüfen

## Voraussetzungen

| Anforderung | Details |
|---|---|
| **Betriebssystem** | Linux Mint 21.x (Ubuntu-basiert) |
| **Benutzerrechte** | sudo-Berechtigung erforderlich |
| **Vorkenntnisse** | Grundlegende systemctl-Befehle (siehe Lab 01) |

---

## Inhalt

- [Aufgaben](#aufgaben)
  - [Schritt 1: Den Ausgangszustand verstehen – was fehlt ohne Unit-Datei?](#schritt-1-den-ausgangszustand-verstehen--was-fehlt-ohne-unit-datei)
  - [Schritt 2: Eine minimale Unit-Datei schreiben](#schritt-2-eine-minimale-unit-datei-schreiben)
  - [Schritt 3: Analyse der vorhanden Unit-Datei von Cockpit](#schritt-3-Die-Unit-datei-von-cockpit-analysieren)]
  - [Schritt 4: Die Unit-Datei schrittweise verbessern](#schritt-4-die-unit-datei-schrittweise-verbessern)
    - [4.1 Benutzerkontext festlegen](#41-benutzerkontext-festlegen)
    - [4.2 Neustartverhalten definieren](#42-neustartverhalten-definieren)
    - [4.3 Abhängigkeiten und Startreihenfolge festlegen](#43-abhängigkeiten-und-startreihenfolge-festlegen)
    - [4.4 Vollständige Unit-Datei](#44-vollständige-unit-datei)
  - [Schritt 5: Autostart aktivieren und Persistenz prüfen](#schritt-5-autostart-aktivieren-und-persistenz-prüfen)
  - [Schritt 6: Fehler in Unit-Dateien diagnostizieren](#schritt-6-fehler-in-unit-dateien-diagnostizieren)
- [Rückblick und Zusammenfassung](#rückblick-und-zusammenfassung)
- [Aufräumarbeiten](#aufräumarbeiten)
- [Autoren und Urheberrecht](#autoren-und-urheberrecht)

---

## Aufgaben

### Schritt 1: Den Ausgangszustand verstehen – was fehlt ohne Unit-Datei?

Bevor Sie eine Unit-Datei schreiben, erleben Sie zunächst, was systemd über einen Dienst ohne Unit-Datei weiß – nämlich nichts.
Bevor Sie das eigentliche Skript erstellen, ein Blick auf den grundlegenden Aufbau eines Bash-Skripts – am einfachsten möglichen Beispiel.

Ein Bash-Skript besteht aus drei wesentlichen Teilen:

```bash
#!/bin/bash          # Shebang: legt fest, welcher Interpreter die Datei ausführt
                     # Kommentare beginnen mit #

while true; do       # Endlosschleife – der Dienst soll dauerhaft laufen
    echo "$(date): Hallo vom Dienst" >> ~/dienste/test.log   # Ausgabe in Datei
    sleep 10         # 10 Sekunden warten, dann wiederholen
done
```

Die Zeile `>> ~/dienste/test.log` hängt jede neue Ausgabe ans Ende der Datei an (`>>` = anhängen, `>` = überschreiben). `$(date)` führt den Befehl `date` aus und fügt das Ergebnis direkt in den Text ein.

Legen Sie das Verzeichnis und ein einfaches Testskript an:

```bash
mkdir -p ~/dienste
nano ~/dienste/test.sh
```

Inhalt – passen Sie den Text frei an:

```bash
#!/bin/bash
while true; do
    echo "$(date): Dienst läuft" >> ~/dienste/test.log
    sleep 10
done
```

Skript ausführbar machen und kurz testen:

```bash
chmod +x ~/dienste/test.sh
~/dienste/test.sh &        # & startet das Skript im Hintergrund
cat ~/dienste/test.log     # sollte zwei Einträge zeigen
jobs                       # zeigt die laufenden Jobs an
kill %1                    # Hintergrundprozess (Job) wieder beenden, 
```

---

Nun das eigentliche Skript für den Dienst. Es erfüllt eine typische netzwerktechnische Aufgabe: Es erfasst regelmäßig den Zustand aktiver TCP-Verbindungen und schreibt die Zusammenfassung ins Syslog – nützlich etwa für die Fehlersuche bei unerwartet hoher Verbindungslast.

Dieses Skript speichern wir im Verzeichnis `/opt`. Dies ist nach dem [FHS](https://de.wikipedia.org/wiki/Filesystem_Hierarchy_Standard) ein Ort für zusätzliche Programme.

```bash
sudo mkdir -p /opt/netmon
sudo nano /opt/netmon/netmon.sh
```

Inhalt:

```bash
#!/bin/bash
while true; do
    echo "$(date): $(ss -s | grep 'TCP:')" | logger -t netmon
    sleep 30
done
```

> **Was macht dieses Skript?**  
> `ss -s` gibt eine Zusammenfassung aller Sockets aus. `grep 'TCP:'` filtert daraus die Zeile mit den TCP-Verbindungen. `logger -t netmon` schreibt die Ausgabe mit dem Tag `netmon` ins Syslog – von dort ist sie über `journalctl -t netmon` abrufbar.

> **Exkurs: Das wiederkehrende Muster von Bash-Befehlsketten**
>
> Die Zeile `echo "$(date): $(ss -s | grep 'TCP:')" | logger -t netmon` wirkt auf den ersten Blick wie ein einziger, komplizierter Befehl. Tatsächlich folgt sie einem Muster, das in den folgenden Übungen immer wieder auftaucht – einmal verstanden, lässt es sich auf jede ähnliche Zeile übertragen, auch wenn die konkreten Befehle wechseln:
>
> **[Text erzeugen] → (optional: filtern/umwandeln) → [irgendwo ablegen]**
>
> Zerlegt in seine Bestandteile:
>
> | Segment | Rolle |
> |---|---|
> | `echo "..."` | **Erzeuger** – gibt einen Text aus |
> | `$(date)` | **Command Substitution** – ein eigener Befehl in Klammern; seine Ausgabe wird an dieser Stelle in den Text eingesetzt |
> | `$(ss -s \| grep 'TCP:')` | ebenfalls Command Substitution – darin steckt wiederum eine eigene kleine Kette: `ss -s` **erzeugt** eine Socket-Übersicht, `grep 'TCP:'` **filtert** daraus nur die passende Zeile |
> | `\|` (Pipe) | Die Ausgabe des Befehls links wird zur Eingabe des Befehls rechts |
> | `logger -t netmon` | **Ablage** – schreibt die Eingabe mit dem Tag `netmon` ins Syslog |
>
> Typische Vertreter für die drei Rollen, denen Sie in späteren Übungen immer wieder begegnen werden:
>
> | Rolle | Typische Befehle |
> |---|---|
> | Text erzeugen | `echo`, `cat`, `curl` |
> | Filtern/Umwandeln | `grep`, `sed`, `gpg --dearmor` |
> | Ablegen | `tee`, `>`, `>>` |
>
> Sobald Sie in einer späteren Übung auf eine lange Befehlszeile mit `\|` oder `>` treffen, lohnt es sich, sie genau nach diesem Schema in ihre drei Rollen zu zerlegen, statt sie als unteilbares Ganzes zu lesen.

Skript ausführbar machen:

```bash
sudo chmod +x /opt/netmon/netmon.sh
```

Versuchen Sie nun, den Dienst über systemd zu starten – ohne dass eine Unit-Datei existiert:

```bash
sudo systemctl start netmon
```

> **Was passiert?**  
> systemd antwortet mit `Unit netmon.service could not be found`. Das ist der entscheidende Unterschied zu einem Dienst, der existiert aber gestoppt ist: Hier kennt systemd den Dienst schlicht nicht. Es gibt keine Unit-Datei, also gibt es aus systemds Sicht keinen Dienst.

Bestätigen Sie das:

```bash
sudo systemctl status netmon
sudo systemctl is-enabled netmon
```

Beide Befehle liefern Fehler – nicht weil der Dienst inaktiv ist, sondern weil er für systemd nicht existiert.

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

### Schritt 2: Eine minimale Unit-Datei schreiben

Es gibt drei Orte, an denen Angaben zum Start von Diensten gemacht werden können:

- Es gibt einen Standard-Ort, an dem die Unit-Dateien abgelegt werden, die vom Software-Entwickler geschrieben wurden und die beim installieren der Software auf dem Rechner abgespeichert werden:<br>`/usr/lib/systemd/system/` <br>Wichtig: Hier sollte man nichts ändern, weil beim nächsten Update die Änderungen überschrieben werden können.
- Eigene, __vollständige__ Unit-Dateien legt man in<br>`/etc/systemd/system/`<br>ab. Diese Änderungen bleiben auch nach einem Update erhalten.
- Möchte man nur Teile einer Unit-Datei anpassen oder ändern, dann kann man das in _Drop-in-Dateien_ Dateien machen. Hierfür gibt es folgende Namenskonventionen:
  - Nehmen wir an, die Unit-Datei für unseren Dienst heißt `mein-dienst.service`
  - Dann heißt das Verzeichnis, in dem man seine Drop-in-Dateien ablegt `/etc/systemd/system/mein-dienst.service.d/`
  - In diesem Verzeichnis kann man mehrere Dateien anlegen. Die in diesen Dateien gemachten Einstellungen überschreiben oder ergänzen die ansonsten bereits schon gültigen Einstellungen. Diese Dateien müssen mit der Endung `.conf` versehen sein und haben typischerweise `root` als Eigentümer und die Rechte 644 (siehe `chmod`). Beispiel:<br>`/etc/systemd/system/mein-dienst.service.d/meine-ergaenzung.conf`

Die offizielle Dokumentation zu systemd findet man auf [systemd.io](https://systemd.io/). Weitere Dokumentation zu diesem Thema findet man bei [Red Hat](https://docs.redhat.com/de/documentation/red_hat_enterprise_linux/10/html/using_systemd_unit_files_to_customize_and_optimize_your_system/working-with-systemd-unit-files).

Erstellen Sie die Unit-Datei:

```bash
sudo nano /etc/systemd/system/netmon.service
```

Minimaler Inhalt:

```ini
[Unit]
Description=Netzwerkmonitor – schreibt TCP-Statistiken ins Syslog

[Service]
ExecStart=/opt/netmon/netmon.sh

[Install]
WantedBy=multi-user.target
```

systemd über die neue Datei informieren:

```bash
sudo systemctl daemon-reload
```

> **Warum `daemon-reload`?**  
> systemd liest Unit-Dateien beim Start ein und hält sie im Speicher. Neue oder geänderte Dateien auf der Festplatte werden erst nach `daemon-reload` berücksichtigt. Ohne diesen Befehl würde systemd die neue Datei ignorieren.

Dienst starten und prüfen:

```bash
sudo systemctl start netmon
sudo systemctl status netmon
```

Prüfen Sie, ob das Skript tatsächlich Einträge ins Syslog schreibt:

```bash
journalctl -t netmon -f
```

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

### Schritt 3: Die Unit-Datei von Cockpit analysieren

Jeder mit apt installierte Dienst bringt eine Unit-Datei mit, die systemd beschreibt, wie der Dienst zu starten ist. Sehen Sie sich die Unit-Datei von Cockpit an:

```bash
systemctl cat cockpit.service
```

Die drei wichtigsten Abschnitte sind:

| Abschnitt | Bedeutung |
|---|---|
| `[Unit]` | Beschreibung und Abhängigkeiten (`Requires=`, `After=`) |
| `[Service]` | Startbefehl (`ExecStart=`), Neustartverhalten (`Restart=`) |
| `[Install]` | Autostart-Ziel (`WantedBy=multi-user.target`) |

Neben `.service`-Dateien kennt systemd weitere Unit-Typen, die in späteren Erweiterungsaufgaben relevant werden:

| Typ | Beispiel | Funktion |
|---|---|---|
| `.service` | `cockpit.service` | Hintergrundprozess / Daemon |
| `.socket` | `cockpit.socket` | Socket-Aktivierung: Dienst startet erst bei eingehender Verbindung |
| `.timer` | `apt-daily.timer` | Geplante Ausführung (Ersatz für cron) |
| `.target` | `multi-user.target` | Gruppeneinheit, vergleichbar mit einem Runlevel |

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

### Schritt 4: Die Unit-Datei schrittweise verbessern

Die minimale Unit-Datei funktioniert – ist aber noch nicht produktionstauglich. Erweitern Sie sie schrittweise.

#### 4.1 Benutzerkontext festlegen

Ein Dienst, der als `root` läuft, ist ein Sicherheitsrisiko. Legen Sie einen dedizierten Systembenutzer an und weisen Sie ihn dem Dienst zu:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin netmon
```

Unit-Datei öffnen und den `[Service]`-Abschnitt erweitern:

```bash
sudo nano /etc/systemd/system/netmon.service
```

```ini
[Service]
ExecStart=/opt/netmon/netmon.sh
User=netmon
Group=netmon
```

#### 4.2 Neustartverhalten definieren

```ini
[Service]
ExecStart=/opt/netmon/netmon.sh
User=netmon
Group=netmon
Restart=on-failure
RestartSec=10s
```

#### 4.3 Abhängigkeiten und Startreihenfolge festlegen

Der Dienst soll erst starten, nachdem das Netzwerk verfügbar ist:

```ini
[Unit]
Description=Netzwerkmonitor – schreibt TCP-Statistiken ins Syslog
After=network.target
```

#### 4.4 Vollständige Unit-Datei

Nach allen Ergänzungen sollte die Datei so aussehen:

```ini
[Unit]
Description=Netzwerkmonitor – schreibt TCP-Statistiken ins Syslog
After=network.target

[Service]
ExecStart=/opt/netmon/netmon.sh
User=netmon
Group=netmon
Restart=on-failure
RestartSec=10s

[Install]
WantedBy=multi-user.target
```

Änderungen aktivieren:

```bash
sudo systemctl daemon-reload
sudo systemctl restart netmon
sudo systemctl status netmon
```

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

### Schritt 5: Autostart aktivieren und Persistenz prüfen

```bash
sudo systemctl enable netmon
sudo reboot
```

Nach dem Neustart:

```bash
sudo systemctl status netmon
journalctl -t netmon --since '5 minutes ago'
```

Läuft der Dienst? Sind Einträge im Syslog vorhanden?

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

### Schritt 6: Fehler in Unit-Dateien diagnostizieren

Eine häufige Situation in der Praxis: Eine Unit-Datei existiert, aber der Dienst startet nicht. Üben Sie die Diagnose gezielt.

Bauen Sie absichtlich einen Fehler ein – ändern Sie den Pfad auf einen nicht existierenden:

```bash
sudo nano /etc/systemd/system/netmon.service
```

```ini
ExecStart=/opt/netmon/netmon-falsch.sh
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart netmon
```

Analysieren Sie den Fehler mit den richtigen Werkzeugen:

```bash
# Kurzstatus mit letzten Fehlermeldungen:
sudo systemctl status netmon

# Vollständiges Log des Dienstes:
journalctl -u netmon -n 30

# Nur Fehler seit dem letzten Boot:
journalctl -b -u netmon -p err
```

> **Was suchen Sie in der Ausgabe?**  
> `status=203/EXEC` bedeutet: Die angegebene Datei wurde nicht gefunden oder ist nicht ausführbar.  
> `status=1` bedeutet: Der Prozess ist mit einem Fehler beendet worden.  
> `Failed to start ...` kombiniert mit dem Exit-Code gibt in den meisten Fällen genug Information, um den Fehler zu lokalisieren.

Korrigieren Sie den Pfad, laden Sie neu und starten Sie den Dienst erneut.

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

## Rückblick und Zusammenfassung

### Was Sie erreicht haben

- Erlebt, was systemd über einen Dienst **ohne Unit-Datei** weiß – und was nicht
- Eine Unit-Datei von Grund auf geschrieben und schrittweise verbessert
- Sicherheitsaspekte (dedizierter Systembenutzer), Neustartverhalten und Abhängigkeiten konfiguriert
- Fehler in Unit-Dateien gezielt diagnostiziert

### Reflexion

– Wann ist eine Drop-in-Datei die bessere Wahl gegenüber einer vollständigen eigenen Unit-Datei?  
– Welche Konsequenz hat es, wenn `After=network.target` fehlt und das Skript Netzwerkverbindungen aufbaut?  

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

## Aufräumarbeiten (nur nach Rücksprache mit Referenten)

```bash
sudo systemctl disable --now netmon
sudo rm /etc/systemd/system/netmon.service
sudo systemctl daemon-reload
sudo rm -rf /opt/netmon
sudo userdel netmon
```

[↑ Zum Inhaltsverzeichnis](#inhalt)

## Autoren und Urheberrecht

- Erstellt von: Michael Lotter, Florian Reichl
- Datum: 2026
- Version: v1.0

![line](images/banner.png)
<p align="center">
<a href="Lab_01_ausbau.md"><img src="images/previous.png" width="150px"></a>
<a href="Lab_03.md"><img src="images/next.png" width="150px"></a>
</p>
