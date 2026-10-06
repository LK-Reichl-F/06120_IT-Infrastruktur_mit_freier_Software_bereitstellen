# Lab 03: Linux-Server in der Cloud bereitstellen und sicher erreichbar machen

![line](images/banner.png)

## Einführung

Als Netzwerkadministrator werden Sie mit der Anforderung konfrontiert, interne Dienste künftig ortsunabhängig bereitzustellen. Mitarbeitende sollen von unterwegs oder aus dem Homeoffice auf Anwendungen zugreifen können – bisher war das nur im Firmennetz möglich. Eine eigene Hardware im Rechenzentrum zu betreiben ist aufwändig und kostenintensiv. Die Alternative: ein gemieteter Linux-Server bei einem Cloud-Anbieter, der rund um die Uhr online ist, eine feste öffentliche IP-Adresse besitzt und von Ihnen vollständig administriert wird.

Bevor auf diesem Server Dienste veröffentlicht werden können, müssen drei Dinge geklärt sein: Der Server muss unter einem stabilen, sprechenden Namen erreichbar sein – unabhängig davon, ob sich die IP-Adresse irgendwann ändert. Der administrative Zugriff per SSH muss abgesichert sein, denn ein öffentlich erreichbarer Server ist vom ersten Moment an automatisierten Angriffen ausgesetzt. Und der gesamte Aufbau muss nachvollziehbar und reproduzierbar dokumentiert sein.

Diese Übung deckt genau diese Grundlage ab: Sie beauftragen einen Server bei einem Cloud-Anbieter, registrieren einen Domänennamen, verknüpfen ihn per DNS-Eintrag mit der öffentlichen IP-Adresse und sichern den SSH-Zugriff mit einem Schlüsselpaar ab. Am Ende dieser Übung haben Sie eine sauber konfigurierte Basis, auf der in den nächsten Schritten Dienste sicher veröffentlicht werden können.

---

## Lernziele

- Einen **Linux-Server** bei einem Cloud-Anbieter beauftragen und verstehen, worauf dabei zu achten ist.
- Den **Beschaffungsprozess eines Domänennamens** kennen und nachvollziehen.
- Einen **DNS-A-Record** setzen, der eine Domain auf eine öffentliche IPv4-Adresse zeigt.
- Ein **SSH-Schlüsselpaar** generieren und den öffentlichen Schlüssel auf dem Server hinterlegen.
- Die **passwortbasierte SSH-Authentifizierung deaktivieren** und ausschließlich Key-basierte Authentifizierung zulassen.

---

## Voraussetzungen

- Ein Rechner mit Internetzugang (Windows, macOS oder Linux)
- Ein Terminal bzw. eine Shell:
  - macOS / Linux: das eingebaute Terminal
  - Windows: PowerShell oder Windows Terminal (OpenSSH ist ab Windows 10 vorinstalliert)
- Ein Account bei einem Cloud-Anbieter (z. B. Hetzner, DigitalOcean, IONOS, Netcup)
- Ein Account bei einem Domain-Registrar (z. B. INWX, Namecheap, Porkbun, united-domains)
- Ca. 10–15 Minuten Zeit für die DNS-Propagierung
- Für den Veranstaltungszeitraum ist für Sie eine VM mit öffentlicher IP und ssh-Zugang vorbereitet und Sie bekommen die Zugangsdaten. Gerne können Sie auch selbst eine VM bei einem Provider in Betrieb nehmen.

---

## Inhalt

- [Hintergrundwissen](#hintergrundwissen)
  - [Was ist SSH-Key-Authentifizierung?](#was-ist-ssh-key-authentifizierung)
- [Aufgaben](#aufgaben)
  - [Schritt 1: Server bereitstellen](#schritt-1-server-bereitstellen)
    - [1.1 Anbieter und Paket wählen](#11-anbieter-und-paket-wählen)
    - [1.2 Initialen Root-Zugriff einrichten](#12-initialen-root-zugriff-einrichten)
    - [1.3 Host-Key-Fingerprint verifizieren und erste Verbindung herstellen](#13-host-key-fingerprint-verifizieren-und-erste-verbindung-herstellen)
  - [Schritt 2: Domänenname beschaffen (theoretischer Überblick)](#schritt-2-domänenname-beschaffen-theoretischer-überblick)
    - [2.1 Registrar auswählen](#21-registrar-auswählen)
    - [2.2 Domain-Verfügbarkeit prüfen und registrieren](#22-domain-verfügbarkeit-prüfen-und-registrieren)
    - [2.3 DNS-Verwaltung verstehen](#23-dns-verwaltung-verstehen)
  - [Schritt 3: DNS-A-Record setzen](#schritt-3-dns-a-record-setzen)
    - [3.1 A-Record anlegen](#31-a-record-anlegen)
    - [3.2 DNS-Propagierung abwarten und prüfen](#32-dns-propagierung-abwarten-und-prüfen)
  - [Schritt 4: SSH-Schlüsselpaar generieren und hinterlegen](#schritt-4-ssh-schlüsselpaar-generieren-und-hinterlegen)
    - [4.1 Schlüsselpaar auf dem lokalen Rechner erzeugen](#41-schlüsselpaar-auf-dem-lokalen-rechner-erzeugen)
    - [4.2 Öffentlichen Schlüssel auf den Server übertragen](#42-öffentlichen-schlüssel-auf-den-server-übertragen)
    - [4.3 Key-basierte Verbindung testen](#43-key-basierte-verbindung-testen)
    - [4.4 Passwortauthentifizierung deaktivieren](#44-passwortauthentifizierung-deaktivieren)
    - [4.5 Abschließende Überprüfung](#45-abschließende-überprüfung)
- [Rückblick und Zusammenfassung](#rückblick-und-zusammenfassung)
- [Aufräumarbeiten](#aufräumarbeiten)
- [Autoren und Urheberrecht](#autoren-und-urheberrecht)

---

## Hintergrundwissen

### Was ist SSH-Key-Authentifizierung?

SSH unterstützt zwei Authentifizierungsverfahren: Passwort und Schlüsselpaar. Bei der Schlüsselpaar-Methode wird ein mathematisch zusammenhängendes Schlüsselpaar erzeugt – ein **privater Schlüssel** (verbleibt auf Ihrem Rechner) und ein **öffentlicher Schlüssel** (wird auf dem Server hinterlegt). Beim Login weist der Client kryptografisch nach, dass er im Besitz des privaten Schlüssels ist, indem er eine Signatur erzeugt, die der Server mit dem hinterlegten öffentlichen Schlüssel verifiziert. Der private Schlüssel selbst wird dabei nie übertragen. Da kein Passwort über das Netzwerk übertragen wird, ist diese Methode deutlich sicherer – insbesondere gegenüber Brute-Force-Angriffen.

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

## Aufgaben

### Schritt 1: Server bereitstellen

#### 1.1 Anbieter und Paket wählen

Loggen Sie sich in das Web-Dashboard Ihres Cloud-Anbieters ein und bestellen Sie einen neuen Linux-Server. Die genaue Bezeichnung variiert je nach Anbieter (z. B. „Server", „Droplet", „Instance" oder „vServer") – technisch erhalten Sie in allen Fällen einen vollständig administrierbaren Linux-Rechner mit öffentlicher IP-Adresse.

Wählen Sie folgende Eckdaten:

| Eigenschaft       | Empfehlung für diese Übung              |
|-------------------|-----------------------------------------|
| Betriebssystem    | Debian stable (aktuell Trixie), 64-Bit  |
| CPU / RAM         | 1 vCPU, 1–2 GB RAM (kleinstes Paket)    |
| Speicher          | 20 GB SSD                               |
| Standort          | EU-Rechenzentrum (z. B. Deutschland)    |
| Öffentliche IP    | IPv4 aktivieren                         |

#### 1.2 Initialen Root-Zugriff einrichten

Bei der Bestellung des Servers werden Sie nach einer initialen Zugriffsmethode gefragt. Wählen Sie vorerst **Passwort-Authentifizierung** oder einen temporären SSH-Key des Anbieters – dies wird in Schritt 4 durch einen eigenen Key ersetzt.

Notieren Sie sich:
- Die **öffentliche IPv4-Adresse** des Servers (wird nach der Bereitstellung im Dashboard angezeigt)
- Das **Root-Passwort** (falls vom Anbieter vergeben oder selbst gewählt)

#### 1.3 Host-Key-Fingerprint verifizieren und erste Verbindung herstellen

Beim ersten SSH-Verbindungsaufbau zu einem neuen Server zeigt der SSH-Client den **Fingerprint des öffentlichen Host-Keys** des Servers an und fragt, ob Sie ihm vertrauen:

```
The authenticity of host '49.12.34.56 (49.12.34.56)' can't be established.
ED25519 key fingerprint is SHA256:abc123XYZ.../beispiel+fingerprint=
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

> **⚠️ Kritischer Sicherheitshinweis: Nicht blind `yes` eingeben.**
> Wer hier ohne Prüfung bestätigt, akzeptiert möglicherweise den Schlüssel eines falschen Servers – etwa bei einem Man-in-the-Middle-Angriff. Der angezeigte Fingerprint muss mit dem tatsächlichen Fingerprint des Servers abgeglichen werden.

**Schritt 1: Fingerprint auf dem Server ermitteln**

Die meisten Cloud-Anbieter zeigen den Host-Key-Fingerprint im Dashboard direkt nach der Bereitstellung des Servers an (z. B. unter „Server Details", „Console" oder „Activity Log"). Notieren Sie sich diesen Wert.

Falls der Anbieter den Fingerprint nicht anzeigt, verbinden Sie sich über die **Web-Konsole des Anbieters** (VNC/KVM – diese Verbindung läuft nicht über das Netzwerk und ist daher vertrauenswürdig) und lesen Sie den Fingerprint direkt vom Server aus:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Beispielausgabe:
```
256 SHA256:abc123XYZ.../beispiel+fingerprint= root@server (ED25519)
```

**Schritt 2: Fingerprint im SSH-Dialog abgleichen**

Verbinden Sie sich nun per SSH von Ihrem lokalen Rechner:

```bash
ssh root@<IPv4-ADRESSE-DER-VM>
```

Vergleichen Sie den angezeigten Fingerprint **Zeichen für Zeichen** mit dem zuvor notierten Wert aus dem Dashboard oder der Web-Konsole.

- **Fingerprints stimmen überein** → Eingabe von `yes` ist sicher. Der Fingerprint wird in `~/.ssh/known_hosts` gespeichert und bei zukünftigen Verbindungen automatisch geprüft.
- **Fingerprints stimmen nicht überein** → Verbindung **sofort abbrechen** (`no` eingeben). Klären Sie die Ursache, bevor Sie fortfahren. Mögliche Gründe: falsche IP, Server wurde neu aufgesetzt, oder ein Angriff liegt vor.

Nach bestätigter Verbindung verlassen Sie den Server wieder:

```bash
exit
```

> **Tipp zur Fehlerbehebung:** Wenn die Verbindung abgelehnt wird, prüfen Sie, ob der Server vollständig gestartet ist (Status im Dashboard) und ob Port 22 in einer eventuell vorhandenen Anbieter-Firewall (häufig „Security Group" oder „Firewall" genannt) geöffnet ist.

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

### Schritt 2: Domänenname beschaffen (theoretischer Überblick)

> **Hinweis:** Dieser Schritt beschreibt den Prozess anbieterunabhängig. Falls Sie für die Übung eine eigene Domain nutzen, führen Sie die Schritte bei Ihrem Registrar entsprechend durch. Für reine Testzwecke ohne eigene Domain lesen Sie den Kasten am Ende dieses Schritts.

#### 2.1 Registrar auswählen

Ein **Domain-Registrar** ist ein Unternehmen, das zur Vergabe von Domänennamen akkreditiert ist. Bekannte Anbieter sind INWX, Namecheap, Porkbun oder united-domains. Die Preise variieren je nach Top-Level-Domain (TLD) – `.de`-Domains kosten typischerweise 0,50–1,50 € pro Monat, `.com`-Domains etwas mehr.

Kriterien bei der Wahl eines Registrars:

- Übersichtliche DNS-Verwaltung im Web-Interface
- Unterstützung von 2-Faktor-Authentifizierung für den Account
- Keine versteckten Verlängerungskosten
- Möglichkeit zum Domain-Transfer (kein Lock-in)

#### 2.2 Domain-Verfügbarkeit prüfen und registrieren

1. Rufen Sie die Website Ihres Registrars auf und suchen Sie nach Ihrem gewünschten Domänennamen.
2. Prüfen Sie die Verfügbarkeit. Falls der Name vergeben ist, schlägt das System alternative TLDs oder ähnliche Namen vor.
3. Legen Sie die Domain in den Warenkorb und schließen Sie die Registrierung ab. Sie müssen dabei Ihre Kontaktdaten (WHOIS-Daten) angeben – diese können in der Regel mit einer Datenschutz-Option (WHOIS-Privacy) verborgen werden.
4. Nach Abschluss der Zahlung ist die Domain innerhalb weniger Minuten aktiv.

#### 2.3 DNS-Verwaltung verstehen

Nach der Registrierung verwalten Sie die DNS-Einträge Ihrer Domain entweder direkt beim Registrar oder bei einem separaten DNS-Anbieter. Für diese Übung nutzen wir die DNS-Verwaltung des Registrars direkt.

Folgende Domainnamen werden im weiteren Verlauf benötigt:

| Verwendungszweck                    | Beispiel-Hostname |
|--------------------------------------|-------------------|
| SSH-Zugriff auf die VM (dieses Lab)  | `server.<IHRE-DOMAIN>` |
| Spätere Dienste (ab Lab 04)          | `cockpit.<IHRE-DOMAIN>`, `dienst-a.<IHRE-DOMAIN>`, `dienst-b.<IHRE-DOMAIN>` |

> **Keine eigene Domain zur Hand?**
> Für Test- und Übungszwecke bieten Dienste wie **nip.io** oder **sslip.io** eine kostenlose Alternative: Sie kodieren die IP-Adresse direkt im Hostnamen. Zum Beispiel löst `cockpit.49-12-34-56.nip.io` automatisch auf `49.12.34.56` auf – ohne DNS-Konfiguration. Diese Methode eignet sich für lokale Tests, nicht aber für produktive Umgebungen oder Let's Encrypt-Zertifikate.

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

### Schritt 3: DNS-A-Record setzen

Nun verknüpfen Sie den Domänennamen mit der öffentlichen IPv4-Adresse Ihrer VM.

#### 3.1 A-Record anlegen

1. Loggen Sie sich in das Verwaltungs-Interface Ihres Registrars ein und navigieren Sie zur **DNS-Verwaltung** Ihrer Domain.
2. Erstellen Sie einen neuen **A-Record** mit folgenden Werten:

| Feld       | Wert                          |
|------------|-------------------------------|
| Typ        | `A`                           |
| Name/Host  | `server` (für `server.<IHRE-DOMAIN>`) |
| Wert/Ziel  | `<IPv4-ADRESSE-DER-VM>`       |
| TTL        | `300` (5 Minuten, für schnelle Aktualisierungen) |

3. Speichern Sie den Eintrag.

> **Was ist TTL?**
> Die *Time to Live* gibt in Sekunden an, wie lange DNS-Resolver diesen Eintrag zwischenspeichern dürfen, bevor sie ihn erneut abfragen. Ein DNS-Resolver (meist beim Internetprovider oder bei öffentlichen Diensten wie Google DNS oder Cloudflare betrieben) ist der Server, der Namensauflösungen im Auftrag der Nutzer durchführt und Ergebnisse zwischenspeichert. Ein niedriger Wert (z. B. 300) ermöglicht schnelle Änderungen, erzeugt aber mehr Anfragen. Für stabile Produktivumgebungen sind Werte von 3600 (1 Stunde) üblich.

> **Exkurs: Wildcard statt Einzeleinträge**
>Statt für jeden zukünftigen Hostnamen einen eigenen A-Record anzulegen, könnten Sie auch einen einzigen Wildcard-Record setzen: Name *, Typ A, Wert <IPv4-Adresse>. Damit löst jede beliebige Subdomain von <IHRE-DOMAIN> automatisch auf diese IP auf – auch solche, die Sie nie angelegt haben. 

#### 3.2 DNS-Propagierung abwarten und prüfen

DNS-Änderungen werden nicht sofort weltweit sichtbar – sie müssen sich erst propagieren. Mit einem TTL von 300 Sekunden dauert das in der Regel 2–10 Minuten.

Prüfen Sie die Auflösung von Ihrem lokalen Rechner aus:

```bash
# macOS / Linux
dig <subdomain>.<IHRE-DOMAIN>

# Windows (PowerShell)
Resolve-DnsName <subdomain>.<IHRE-DOMAIN>
```

Das Ergebnis sollte die IP-Adresse Ihrer VM enthalten. Beispielausgabe bei `dig`:

```
;; ANSWER SECTION:
<subdomain>.<IHRE-DOMAIN>.  300  IN  A  49.12.34.56
```

Alternativ können Sie Online-Tools wie [dnschecker.org](https://dnschecker.org) nutzen, um die Propagierung weltweit zu überprüfen.

> **Tipp zur Fehlerbehebung:** Wenn `dig` keine Antwort liefert oder die falsche IP zeigt:
> - Warten Sie weitere 5 Minuten und versuchen Sie es erneut.
> - Prüfen Sie im DNS-Interface des Registrars, ob der Eintrag korrekt gespeichert wurde.
> - Leeren Sie ggf. den lokalen DNS-Cache: `sudo dscacheutil -flushcache` (macOS) bzw. `ipconfig /flushdns` (Windows).

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

### Schritt 4: SSH-Schlüsselpaar generieren und hinterlegen

Passwortbasierte SSH-Zugänge sind ein häufiges Angriffsziel. Automatisierte Bots scannen das Internet kontinuierlich nach offenen SSH-Ports und versuchen gängige Passwörter. In diesem Schritt ersetzen Sie die Passwortauthentifizierung durch ein kryptografisches Schlüsselpaar.

#### 4.1 Schlüsselpaar auf dem lokalen Rechner erzeugen

Öffnen Sie ein Terminal auf Ihrem **lokalen Rechner** (nicht auf der VM):

```bash
ssh-keygen -t ed25519 -C "laboruebung-vm"
# -t: Schlüsseltyp (ed25519) | -C: Kommentar zur Wiedererkennung des Schlüssels
```

Sie werden nach einem Speicherort und einer optionalen Passphrase gefragt:

```
Enter file in which to save the key (/home/IhrName/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

- **Speicherort:** Drücken Sie Enter, um den Standardpfad zu übernehmen.
- **Passphrase:** Empfohlen – eine Passphrase verschlüsselt den privaten Schlüssel zusätzlich. Selbst wenn die Schlüsseldatei gestohlen wird, ist sie ohne die Passphrase wertlos.

Nach der Eingabe werden zwei Dateien erstellt:

| Datei                        | Inhalt          | Verbleib          |
|------------------------------|-----------------|-------------------|
| `~/.ssh/id_ed25519`          | Privater Schlüssel | Nur auf Ihrem Rechner |
| `~/.ssh/id_ed25519.pub`      | Öffentlicher Schlüssel | Wird auf den Server kopiert |

> **Warum ed25519?**
> Ed25519 ist ein modernes Signaturverfahren auf Basis elliptischer Kurven. Es ist schneller, sicherer und erzeugt kürzere Schlüssel als das ältere RSA-Verfahren. Für neue Schlüssel ist ed25519 heute die empfohlene Wahl.

> **⚠️ Wichtig:** Der private Schlüssel (`id_ed25519`) verlässt niemals Ihren Rechner. Geben Sie ihn nicht weiter, laden Sie ihn nicht hoch und kopieren Sie ihn nicht auf Server.

#### 4.2 Öffentlichen Schlüssel auf den Server übertragen

Kopieren Sie den öffentlichen Schlüssel mit `ssh-copy-id` auf den Server. Dabei wird noch die Passwortauthentifizierung verwendet:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@server.<IHRE-DOMAIN>
```

Geben Sie das Root-Passwort des Servers ein, wenn Sie dazu aufgefordert werden. Der Befehl hängt den öffentlichen Schlüssel automatisch in die Datei `~/.ssh/authorized_keys` auf dem Server ein.

**Windows-Alternative** (falls `ssh-copy-id` nicht verfügbar ist):

```powershell
# Inhalt des öffentlichen Schlüssels anzeigen
Get-Content "$env:USERPROFILE\.ssh\id_ed25519.pub"
```

Kopieren Sie die Ausgabe. Verbinden Sie sich dann per SSH mit Passwort auf den Server und fügen Sie den Schlüssel manuell ein:

```bash
mkdir -p ~/.ssh
echo "HIER-DEN-KOPIERTEN-SCHLÜSSEL-EINFÜGEN" >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

#### 4.3 Key-basierte Verbindung testen

Testen Sie jetzt den Login mit dem Schlüssel – **ohne** Schließen der bestehenden Session (zum Zweck der Verfügbarkeit der ssh-Verbindung im Fehlerfall):

Öffnen Sie ein **neues** Terminalfenster und verbinden Sie sich:

```bash
ssh root@server.<IHRE-DOMAIN>
```

Falls eine Passphrase für den ssh-key vergeben wurde, werden Sie danach gefragt. Bei erfolgreicher Verbindung erscheint der Server-Prompt.

> **Tipp zur Fehlerbehebung:** Wenn der Login fehlschlägt:
> - Prüfen Sie die Berechtigungen auf der VM: `chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys`
> - Prüfen Sie, ob der richtige öffentliche Schlüssel in `~/.ssh/authorized_keys` eingetragen ist: `cat ~/.ssh/authorized_keys`
> - Aktivieren Sie ausführliche SSH-Ausgabe zur Fehlerdiagnose: `ssh -v root@server.<IHRE-DOMAIN>`

#### 4.4 Passwortauthentifizierung deaktivieren

Sobald der Key-basierte Login erfolgreich funktioniert, deaktivieren Sie die Passwortauthentifizierung auf dem Server. Führen Sie die folgenden Befehle **auf dem Server** aus:


##### Sicherheitskopie der SSH-Konfiguration anlegen

```bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

##### Passwortauthentifizierung deaktivieren

suche in der /etc/ssh/sshd_config die Zeile mit `PasswordAuthentication` und ersetze mit `PasswordAuthentication no`


##### Sicherstellen, dass nur autorisierte Keys akzeptiert werden

suche in der /etc/ssh/sshd_config die Zeile mit `PubkeyAuthentication` und ersetze mit `PubkeyAuthentication yes`


##### Laden Sie die SSH-Konfiguration neu:

```bash
systemctl reload sshd
```

#### 4.5 Abschließende Überprüfung

Öffnen Sie ein weiteres neues Terminalfenster und bestätigen Sie, dass ausschließlich der Key-basierte Login funktioniert:

```bash
# Dieser Befehl sollte erfolgreich sein (Key-Login)
ssh root@server.<IHRE-DOMAIN>

# Dieser Befehl sollte fehlschlagen (Passwort-Login)
ssh -o PubkeyAuthentication=no root@server.<IHRE-DOMAIN>
# -o: einzelne SSH-Option für diese Verbindung setzen, hier: Key-Login erzwingen ausschalten
```

Bei der zweiten Verbindung sollte die Fehlermeldung `Permission denied (publickey)` erscheinen – das bestätigt, dass Passwortauthentifizierung erfolgreich deaktiviert ist.

> **⚠️ Wichtig:** Trennen Sie Ihre bestehende SSH-Session erst, wenn Sie den Key-basierten Login erfolgreich getestet haben. Wenn Sie sich zu früh ausloggen und der Login mit Key nicht funktioniert, sperren Sie sich möglicherweise aus. Die meisten Cloud-Anbieter bieten in diesem Fall eine Web-Konsole (VNC/KVM) als Notfallzugang.

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

## Rückblick und Zusammenfassung

### Was Sie erreicht haben

- Einen **Linux-Server** in der Cloud bereitgestellt und verstanden, welche Parameter dabei relevant sind.
- Den **Beschaffungsprozess eines Domänennamens** nachvollzogen – von der Registrar-Wahl bis zur Registrierung.
- Einen **DNS-A-Record** angelegt und die Namensauflösung auf die Server-IP erfolgreich geprüft.
- Ein **ed25519-Schlüsselpaar** erzeugt, den öffentlichen Schlüssel auf dem Server hinterlegt und den Key-basierten SSH-Login eingerichtet.
- Die **Passwortauthentifizierung** für SSH deaktiviert und damit die Angriffsfläche des Servers deutlich reduziert.

### Reflexion

- Warum ist es sinnvoll, einen Domänennamen statt der IP-Adresse direkt zu verwenden – auch wenn sich die IP selten ändert?
- Welche Konsequenz hat eine zu hohe TTL, wenn Sie die Server-IP kurzfristig ändern müssen?
- Ist eine Passphrase auf dem privaten Schlüssel sinnvoll?

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

## Aufräumarbeiten (nur nach Rücksprache mit Referenten)

Wenn Sie den Server nur für diese Übung genutzt haben:

- Löschen Sie den Server im Dashboard Ihres Cloud-Anbieters, um weitere Kosten zu vermeiden.
- Wenn die Domain nicht weiter benötigt wird, kündigen Sie die Verlängerung beim Registrar (Domänennamen laufen bis zum Ende der bezahlten Laufzeit weiter).

[↑ Zum Inhaltsverzeichnis](#inhalt)

---

## Autoren und Urheberrecht

- Erstellt von: Michael Lotter, Florian Reichl
- Datum: 02/2026
- Version: v1.0

![line](images/banner.png)

<p align="center">
<a href="Lab_02.md"><img src="images/previous.png" width="150px"></a>
<a href="Lab_04.md"><img src="images/next.png" width="150px"></a>
</p>
