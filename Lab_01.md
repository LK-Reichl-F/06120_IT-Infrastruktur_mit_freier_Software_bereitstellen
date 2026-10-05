# Lab 01: Lokale Bereitstellung eines webbasierten Dienstes

![line](images/banner.png)

## Einführung

Als Netzwerkadministrator werden Sie damit beauftragt, auf einem Linux-Rechner im lokalen Netz eine webbasierte Verwaltungsoberfläche bereitzustellen – schnell, ohne zusätzliche Infrastruktur und mit möglichst geringem Aufwand. Die Wahl fällt auf **Cockpit**, ein von Red Hat entwickeltes Tool, das Systemressourcen, Dienste und Logs übersichtlich im Browser zugänglich macht.

Ihre Aufgabe: Cockpit installieren, den Dienst dauerhaft und zuverlässig betreiben und sicherstellen, dass er auch nach einem Serverneustart oder einer Konfigurationsänderung ohne manuelles Eingreifen wieder läuft.

## Lernziele

- Einen Dienst mit **`apt install`** installieren und den Installationsstatus prüfen
- Den Dienst mit **`systemctl`** starten, stoppen und dauerhaft aktivieren
- Das **Startverhalten** des Dienstes mit systemd dauerhaft sicherstellen (`enable`)
- **Konfigurationsänderungen** unter `/etc` vornehmen und den Dienst gezielt neu starten
- Die **Funktionsweise von systemd** und Unit-Dateien verstehen
- Grundlegende Fragen zu **Verfügbarkeit, Sicherheit, Skalierbarkeit und Ausfallsicherheit** reflektieren

## Voraussetzungen

| Anforderung | Details |
|---|---|
| **Betriebssystem** | Linux Mint 21.x (Cinnamon) |
| **Benutzerrechte** | sudo-Berechtigung erforderlich |
| **Internetzugang** | Für apt-Paketinstallation |
| **Browser** | Firefox oder Chromium (lokal installiert) |
| **Terminal** | GNOME Terminal, Konsole oder vergleichbar |

## Inhalt

- [Aufgaben](#aufgaben)
  - [Schritt 1: System aktualisieren und Cockpit installieren](#schritt-1-system-aktualisieren-und-cockpit-installieren)
    - [1.1 Software aktualisieren](#11-software-aktualisieren)
    - [1.2 Cockpit installieren](#12-cockpit-installieren)
    - [1.3 Installation überprüfen](#13-installation-überprüfen)
  - [Schritt 2: Den Cockpit-Dienst mit systemd verwalten](#schritt-2-den-cockpit-dienst-mit-systemd-verwalten)
    - [2.1 Aktuellen Dienststatus prüfen](#21-aktuellen-dienststatus-prüfen)
    - [2.2 Dienst starten](#22-dienst-starten)
    - [2.3 Dienst dauerhaft beim Systemstart aktivieren](#23-dienst-dauerhaft-beim-systemstart-aktivieren)
    - [2.4 Neustart simulieren und Persistenz prüfen](#24-neustart-simulieren-und-persistenz-prüfen)
    - [2.5 Cockpit im Browser aufrufen](#25-cockpit-im-browser-aufrufen)
  - [Schritt 3: Konfigurationsänderungen und sicherer Dienstneustart](#schritt-3-konfigurationsänderungen-und-sicherer-dienstneustart)
    - [3.1 Konfigurationsverzeichnis kennenlernen](#31-konfigurationsverzeichnis-kennenlernen)
    - [3.2 Neue Konfigurationsdatei anlegen](#32-neue-konfigurationsdatei-anlegen)
    - [3.3 Login-Banner erstellen](#33-login-banner-erstellen)
    - [3.4 Dienst nach Konfigurationsänderung neu starten](#34-dienst-nach-konfigurationsänderung-neu-starten)
    - [3.5 Ergebnis überprüfen](#35-ergebnis-überprüfen)
  - [Schritt 4: systemd – Unit-Datei analysieren und Befehle nachschlagen](#schritt-4-systemd--unit-datei-analysieren-und-befehle-nachschlagen)
    - [4.1 Die Unit-Datei von Cockpit analysieren](#41-die-unit-datei-von-cockpit-analysieren)
    - [4.2 Wichtige systemctl-Befehle im Überblick](#42-wichtige-systemctl-befehle-im-überblick)
- [Autoren und Urheberrecht](#autoren-und-urheberrecht)

## Aufgaben

### Schritt 1: System aktualisieren und Cockpit installieren

> #### Aufgaben als Administrator ausführen
>
> Ein normaler Linux-Benutzer darf viele Dinge nicht tun. Zum Beispiel darf ein normaler Benutzer keine Software installieren.
>
> Falls man doch Software installieren möchte, so wird man nur kurzzeitig Administrator, der bei Linux `root` heißt.
>
> Um einen einzelnen Befehl als Administrator auszuführen schreibt man einfach `sudo` vor den Befehl. Zum Beispiel `sudo apt install ein_programm`.
>
>Man kann auch das Benutzerkonto für längere Zeit zu `root` umschalten: Mit `sudo -s` ist man für die nächsten Befehle `root`. Solange, bis man diesen Zustand mit `exit` wieder beendet.
>
> Hinweise: 
> - Den Befehl `sudo` dürfen nur Benutzer verwenden, die zur Benutzergruppe `sudo` gehören. Alle anderen Benutzer erhalten eine Fehlermeldung.
> - Jedes mal, wenn man `sudo` benutzt, wird man nach dem eigenen Passwort gefragt.
> - Wird `sudo` innerhalb kurzer Zeit mehrmals aufgerufen, so muss man nur einmal das Passwort eingeben.

#### 1.1 Software aktualisieren

Bevor ein neues Paket installiert wird, synchronisieren Sie die lokale Paketdatenbank. Öffnen Sie ein Terminal und führen Sie aus:

```bash
sudo apt update
```

> **Was passiert hier?**  
> `apt update` lädt die aktuellen Paketlisten von den konfigurierten Quellen herunter. Es werden dabei **keine** Pakete installiert oder aktualisiert – nur die Metadaten (Paketname, Version, Abhängigkeiten) werden erneuert.

Mit der aktualisierten Paketdatenbank bringen wir die Software auf unserem Rechner auf den aktuellen Stand. Geben Sie im Terminal folgenden Befehl ein:

```bash
sudo apt upgrade
```

> **Was passiert hier?**  
> `apt upgrade` lädt die Pakete herunter, welche sich aktualisieren lassen. Im Anschluss werden diese Pakete installiert bzw. aktualisiert.



#### 1.2 Cockpit installieren

```bash
sudo apt install cockpit -y
```

Das `-y`-Flag bestätigt alle Rückfragen automatisch. apt löst Abhängigkeiten auf, lädt die Pakete herunter und legt dabei automatisch eine **systemd-Unit-Datei** für den Dienst an.

#### 1.3 Installation überprüfen

```bash
dpkg -l cockpit
```

> **Tipp:** Achten Sie auf den Status `ii` am Zeilenanfang – das bedeutet: **i**nstalliert und korrekt konfiguriert. Ein `rc` würde bedeuten, das Paket wurde entfernt, aber Konfigurationsdateien sind noch vorhanden.

> **Was ist `dpkg`?**
> `dpkg`
[↑ Zum Inhaltsverzeichnis](#inhalt)

---

### Schritt 2: Den Cockpit-Dienst mit systemd verwalten

> **Was ist systemd?**  
> **systemd** ist der Init-Prozess (PID 1) auf modernen Linux-Distributionen. Er ist verantwortlich für das Starten, Stoppen und Überwachen aller Dienste und wird direkt beim Hochfahren des Systems als erster Prozess gestartet. Dienste werden in sogenannten **Unit-Dateien** beschrieben, die systemd mitteilen, wie und wann ein Dienst gestartet werden soll. apt hat beim Installieren von Cockpit automatisch eine solche Unit-Datei angelegt – Sie müssen den Dienst nur noch aktivieren.

#### 2.1 Aktuellen Dienststatus prüfen

Nach der Installation ist Cockpit möglicherweise noch nicht aktiv. Prüfen Sie den Status:

```bash
sudo systemctl status cockpit
```

Relevante Ausgaben:
- `Active: active (running)` → Dienst läuft
- `Active: inactive (dead)` → Dienst ist gestoppt
- `Loaded: ... enabled` / `disabled` → Autostart-Verhalten beim Systemstart

#### 2.2 Dienst starten

```bash
sudo systemctl start cockpit
sudo systemctl status cockpit
```

#### 2.3 Dienst dauerhaft beim Systemstart aktivieren

Damit Cockpit nach einem Neustart automatisch startet, muss der Dienst **aktiviert** werden:

```bash
sudo systemctl enable --now cockpit
```

> **`enable` vs. `start`**  
> - `systemctl start` → startet den Dienst **sofort**, nur für die aktuelle Sitzung  
> - `systemctl enable` → aktiviert den **Autostart** beim nächsten Systemstart  
> - `systemctl enable --now` → kombiniert beides in einem Schritt

#### 2.4 Neustart simulieren und Persistenz prüfen

Starten Sie den Rechner neu:

```bash
sudo reboot
```

Prüfen Sie nach dem Neustart, ob Cockpit automatisch gestartet wurde:

```bash
sudo systemctl status cockpit
```

> **Tipp zur Fehlerbehebung:** Falls Cockpit nach dem Neustart nicht aktiv ist, prüfen Sie mit `sudo systemctl is-enabled cockpit`, ob `enable` korrekt ausgeführt wurde.

#### 2.5 Cockpit im Browser aufrufen

Öffnen Sie Ihren Browser und rufen Sie auf:

```
https://localhost:9090
```

Melden Sie sich mit Ihrem Linux-Benutzernamen und -Passwort an. Da das Zertifikat selbstsigniert ist, müssen Sie im Browser eine **Sicherheitsausnahme** bestätigen.

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

### Schritt 3: Konfigurationsänderungen und sicherer Dienstneustart

#### 3.1 Konfigurationsverzeichnis kennenlernen

Konfigurationsdateien von Cockpit liegen unter `/etc/cockpit/`. Erkunden Sie das Verzeichnis:

```bash
ls -la /etc/cockpit/
```

> **Hinweis:** Das Verzeichnis existiert, ist aber zunächst leer – Cockpit läuft nach der Installation mit seinen eingebauten Standardwerten, ohne dass eine eigene Konfigurationsdatei vorhanden ist. Eigene Einstellungen werden erst durch das Anlegen einer `cockpit.conf` aktiv.

#### 3.2 Neue Konfigurationsdatei anlegen

Die Datei `/etc/cockpit/cockpit.conf` existiert noch nicht und muss neu erstellt werden. Öffnen Sie nano mit dem gewünschten Dateipfad – nano legt die Datei beim Speichern automatisch an:

```bash
sudo nano /etc/cockpit/cockpit.conf
```

Fügen Sie folgenden Inhalt ein:

```ini
[WebService]
Origins = https://localhost:9090

[Session]
Banner = /etc/cockpit/issue.cockpit
```

Speichern Sie mit `Strg+O`, bestätigen Sie mit `Enter`, beenden Sie mit `Strg+X`.

#### 3.3 Login-Banner erstellen

```bash
echo 'Willkommen – nur für autorisierte Benutzer!' \
  | sudo tee /etc/cockpit/issue.cockpit
  # Erzeugen: Text ausgeben | Ablegen: in Datei schreiben + auf dem Bildschirm anzeigen
```
> Der angegebene Text 'Willkommen – nur für autorisierte Benutzer!' wird an zwei Stellen 
> geschrieben:
> - in die Datei issue.cockpit
> - auf stdout (Terminal)

#### 3.4 Dienst nach Konfigurationsänderung neu starten

```bash
sudo systemctl restart cockpit
```

> **Warum `restart` statt `stop` + `start`?**  
> `systemctl restart` führt Stopp und Start **atomar** in einem kontrollierten Schritt durch. Bei manuell hintereinander ausgeführten Befehlen `stop`/`start` entsteht ein kurzes Zeitfenster, in dem der Dienst nicht erreichbar ist. Zudem kann systemd bei Diensten mit `Type=notify` sicherstellen, dass der neue Prozess vollständig gestartet ist, bevor der alte beendet wird.

#### 3.5 Ergebnis überprüfen

```bash
sudo systemctl status cockpit

# Port-Erreichbarkeit prüfen:
ss -tlnp | grep 9090   # Erzeugen: aktive Listener anzeigen | Filtern: nur Zeile mit Port 9090
```

Rufen Sie `https://localhost:9090` erneut im Browser auf und prüfen Sie, ob das Banner beim Login erscheint.

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

### Schritt 4: Wichtige systemctl-Befehle im Überblick
Diese Befehle kann man als Administrator (root) eingeben, um Dienste zu verwalten.

```bash
sudo systemctl start cockpit       # Sofort starten
sudo systemctl stop cockpit        # Sofort stoppen
sudo systemctl restart cockpit     # Neu starten
sudo systemctl reload cockpit      # Konfiguration neu laden (ohne Prozessneustart)
sudo systemctl enable cockpit      # Autostart aktivieren
sudo systemctl disable cockpit     # Autostart deaktivieren
sudo systemctl status cockpit      # Status und letzte Logs
sudo systemctl is-active cockpit   # Gibt 'active' oder 'inactive' zurück
sudo systemctl is-enabled cockpit  # Gibt 'enabled' oder 'disabled' zurück

journalctl -u cockpit              # Alle Logs des Dienstes
journalctl -u cockpit -f           # Logs live verfolgen (follow)
journalctl -b -u cockpit           # Logs nur vom aktuellen Boot
```

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

## Autoren und Urheberrecht

- Erstellt von: Michael Lotter, Florian Reichl
- Datum: 2026
- Version: v1.0

![line](images/banner.png)
<p align="center">
<a href="README.md"><img src="images/previous.png" width="150px"></a>
<a href="Lab_01_ausbau.md"><img src="images/next.png" width="150px"></a>
</p>
