# Lab 01: Ausbaustufe

![line](images/banner.png)

## Inhalt

- [Schritt 5: Reflexion und Erweiterungsaufgaben](#schritt-5-reflexion-und-erweiterungsaufgaben)
  - [5.1 Reflexionsfragen](#51-reflexionsfragen)
  - [5.2 Erweiterungsaufgabe A: Socket-Aktivierung](#52-erweiterungsaufgabe-a-socket-aktivierung)
  - [5.3 Erweiterungsaufgabe B: Firewall mit ufw konfigurieren](#53-erweiterungsaufgabe-b-firewall-mit-ufw-konfigurieren)
  - [5.4 Erweiterungsaufgabe C: Automatischen Neustart konfigurieren](#54-erweiterungsaufgabe-c-automatischen-neustart-konfigurieren)
  - [5.5 Erweiterungsaufgabe D: Logs mit journalctl analysieren](#55-erweiterungsaufgabe-d-logs-mit-journalctl-analysieren)
- [Rückblick und Zusammenfassung](#rückblick-und-zusammenfassung)
- [Aufräumarbeiten](#aufräumarbeiten)
- [Autoren und Urheberrecht](#autoren-und-urheberrecht)

---

### Schritt 5: Reflexion und Erweiterungsaufgaben

Nachdem Sie Cockpit lokal auf einem einzelnen Rechner betrieben haben – ohne weitere Absicherung, Redundanz oder Skalierungsmaßnahmen – ist es Zeit für eine strukturierte Reflexion.

#### 5.1 Reflexionsfragen

**Verfügbarkeit**
- Was passiert, wenn der Server, auf dem Cockpit läuft, ausfällt oder neu gestartet wird?
- Cockpit ist aktuell nur über `localhost` erreichbar. Was wäre notwendig, damit andere Geräte im Netzwerk darauf zugreifen können?
- Wie können Sie sicherstellen, dass Cockpit nach einem unerwarteten Absturz automatisch neu startet? *(Hinweis: `Restart=` in der Unit-Datei)*

**Sicherheit**
- Das Zertifikat ist selbstsigniert. Welche Risiken birgt das in einem produktiven Umfeld?
- Cockpit lauscht standardmäßig auf Port 9090. Welche Maßnahmen würden Sie ergreifen, um den Zugriff auf autorisierte Clients zu beschränken? *(Stichwort: Firewall, `ufw`)*

**Skalierbarkeit & Energieeffizienz**
- Cockpit läuft permanent im Hintergrund. Welche Alternative bietet systemd, um einen Dienst nur dann zu starten, wenn tatsächlich eine Verbindung eingeht? *(Stichwort: Socket-Aktivierung, `cockpit.socket`)*
- Wie verhält es sich, wenn nicht nur der Dienst Cockpit, sondern 50 weitere Dienste bereitgestellt werden sollen?

**Ausfallsicherheit**
- Was wäre notwendig, um Cockpit hochverfügbar zu betreiben (kein Single Point of Failure)?
- Welche Rolle spielen Backups der `/etc/cockpit/`-Konfiguration für die Wiederherstellung nach einem Systemausfall?

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

#### 5.2 Erweiterungsaufgabe A: Socket-Aktivierung

Deaktivieren Sie `cockpit.service` und aktivieren Sie stattdessen `cockpit.socket`, damit Cockpit erst bei einer eingehenden Verbindung gestartet wird:

```bash
sudo systemctl disable --now cockpit
sudo systemctl enable --now cockpit.socket
```

Beobachten Sie mit `journalctl -u cockpit -f`, **wann** der Dienst startet. Was stellen Sie fest? Welche Auswirkungen hat das auf den Ressourcenverbrauch?

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

#### 5.3 Erweiterungsaufgabe B: Firewall mit ufw konfigurieren

Installieren und konfigurieren Sie `ufw` (Uncomplicated Firewall), sodass Port 9090 nur aus dem lokalen Netzwerk erreichbar ist:

```bash
sudo apt install ufw -y
sudo ufw allow from 192.168.0.0/24 to any port 9090
sudo ufw enable
sudo ufw status verbose
```

| Segment | Bedeutung |
|---|---|
| `ufw allow` | Befehl: eine Erlaubnisregel hinzufügen |
| `from 192.168.0.0/24` | Quelle einschränken: nur dieses Subnetz darf zugreifen |
| `to any` | Ziel: jede lokale Adresse des Servers (nicht weiter eingeschränkt) |
| `port 9090` | Zielport: nur dieser Port ist von der Regel betroffen |

> **Tipp zur Fehlerbehebung:** Passen Sie das Subnetz `192.168.0.0/24` an Ihr tatsächliches lokales Netzwerk an. Prüfen Sie Ihre IP-Adresse mit `ip addr show`.

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

#### 5.4 Erweiterungsaufgabe C: Automatischen Neustart konfigurieren

Erstellen Sie eine systemd-**Drop-in-Datei**, die Cockpit bei einem Absturz automatisch neu startet:

```bash
sudo mkdir -p /etc/systemd/system/cockpit.service.d/
sudo nano /etc/systemd/system/cockpit.service.d/restart.conf
```
> `cockpit.service` definiert von Haus aus kein Neustartverhalten. Statt die Original-Datei zu kopieren und zu bearbeiten, legt man eine Drop-in-Datei `restart.conf` an, die nur den einen neuen Parameter Restart=on-failure enthält. systemd liest beim Start beides ein  und verhält sich so, als wäre der Parameter von Anfang an Teil der Unit-Datei gewesen.

Inhalt der Datei:

```ini
[Service]
Restart=on-failure
RestartSec=5s
```

Änderungen aktivieren:

```bash
sudo systemctl daemon-reload
sudo systemctl restart cockpit
```

Testen Sie den automatischen Neustart, indem Sie den Prozess gezielt beenden:

```bash
sudo kill -9 $(pgrep cockpit)   # $(pgrep cockpit): Command Substitution, liefert die Prozess-ID
sleep 6
sudo systemctl status cockpit
```

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

#### 5.5 Erweiterungsaufgabe D: Logs mit journalctl analysieren

```bash
journalctl -u cockpit --since '1 hour ago'
journalctl -u cockpit --no-pager | grep -i error   # Erzeugen: alle Logs | Filtern: nur Zeilen mit "error"
journalctl -b -u cockpit
```

Beantworten Sie: Welche Informationen finden Sie in den Logs? Wann und aus welchem Grund wurde Cockpit gestartet oder gestoppt?

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

## Rückblick und Zusammenfassung

### Was Sie erreicht haben

- Einen Dienst mit **`apt install`** installiert und den Status geprüft
- **systemd** verstanden: start, stop, restart, enable – und den Unterschied zwischen sofortigem Start und persistentem Autostart
- Konfigurationsänderungen unter **`/etc`** vorgenommen und den Dienst sicher neu gestartet
- **Unit-Dateien** analysiert und die Struktur von systemd-Diensten verstanden
- Die **Grenzen eines einfachen lokalen Setups** reflektiert und Ansätze zur Erweiterung erarbeitet

### Reflexion

– In welchen Szenarien ist ein dauerhaft laufender Dienst sinnvoll – und wann ist Socket-Aktivierung die bessere Wahl?  
– Wie würden Sie dieses Setup automatisiert auf mehreren Servern ausrollen?  


[↑ Zum Inhaltsverzeichnis](#inhalt)

---

## Aufräumarbeiten (nur nach Rücksprache mit Referenten)

Versetzen Sie Ihr System in den ursprünglichen Zustand:

```bash
# Dienst stoppen und deaktivieren
sudo systemctl disable --now cockpit cockpit.socket

# Paket vollständig entfernen
sudo apt purge cockpit -y
sudo apt autoremove -y

# Eigene Konfigurationsdateien löschen
sudo rm -rf /etc/cockpit/cockpit.conf /etc/cockpit/issue.cockpit
sudo rm -rf /etc/systemd/system/cockpit.service.d/
```

> **Tipp:** Mit `dpkg -l cockpit` können Sie nach dem Aufräumen prüfen, ob das Paket vollständig entfernt wurde. Der Status `rc` zeigt an, dass das Paket entfernt wurde, aber Konfigurationsdateien noch vorhanden sind (entfernt werden diese mit `purge`).

[↑ Zum Inhaltsverzeichnis](#inhalt)

---
## Autoren und Urheberrecht

- Erstellt von: Michael Lotter, Florian Reichl
- Datum: 2026
- Version: v1.0

![line](images/banner.png)
<p align="center">
<a href="Lab_01.md"><img src="images/previous.png" width="150px"></a>
<a href="Lab_02.md"><img src="images/next.png" width="150px"></a>
</p>