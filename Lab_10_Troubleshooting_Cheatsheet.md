# Troubleshooting-Cheat-Sheet: Netzwerk & Telemetrie-Pipeline

Entstanden aus der Fehlersuche in Lab 10 (Router → Telegraf → InfluxDB). Die Befehle sind allgemein nützlich, nicht nur für dieses Lab. Grundidee der ganzen Fehlersuche: **von unten nach oben** – erst Transport (kommen Pakete an?), dann Anwendung (spricht sie das richtige Protokoll?), dann Daten (landen sie in der DB?).

---

## 1. Erreichbarkeit & offene Ports testen

### `telnet <host> <port>` – roher TCP-Verbindungstest
Telnet war ursprünglich ein Remote-Login (Port 23), der Client ist im Kern aber nur ein **TCP-Socket-Öffner**. Mit Portangabe verbindet er dorthin – ideal zum „Klopf mal an, ob da wer aufmacht".

```
telnet 94.130.25.101 57000
```

Drei Reaktionen, die man unterscheiden muss:

| Reaktion | Bedeutung |
|---|---|
| `Connected to … / Escape character is '^]'` | Handshake ok, **Dienst lauscht**, Weg frei. Raus: `Strg+]`, dann `quit`. |
| `Connection refused` | Weg frei, aber **kein Dienst** auf dem Port (Port zu / Dienst aus). |
| hängt / Timeout | **Paketfilter verwirft** (Firewall droppt statt abzulehnen) oder Host nicht erreichbar. |

„refused" vs. „hängt" ist die wichtigste Unterscheidung: **Dienst-Problem** vs. **Firewall-/Routing-Problem**.
Auf Cisco IOS-XE ist `telnet` an Bord (kein `nc`), daher dort das Mittel der Wahl für einen TCP-Test nach außen.

### `nc -vz <host> <port>` – die moderne Variante
```
nc -vz 94.130.25.101 57000     # ein Port
nc -vz 94.130.25.101 22 57000  # mehrere Ports nacheinander
```
- `-z` = nur Port testen, keine Daten senden
- `-v` = gesprächig (zeigt „succeeded"/„failed")

Macht im Kern dasselbe wie telnet, aber skriptfreundlicher. Vollzieht ebenfalls den echten TCP-Handshake – ist also auch im `tcpdump` als echte Verbindung sichtbar.

### `nmap -p <port> <host>` – das große Besteck
```
nmap -p 57000 94.130.25.101
nmap -p 22,3000,8086,57000 94.130.25.101
```
Für viele Ports auf einmal, mit Statusangabe `open` / `closed` / `filtered` (= gedroppt).

---

## 2. Öffentliche IP-Adresse ermitteln

Wichtig bei NAT: Die IP, mit der ein Gerät **nach außen** auftritt, ist nicht seine lokale IP.

```bash
curl -4 ifconfig.me          # erzwingt IPv4
curl -4 https://ipinfo.io/ip
curl -4 icanhazip.com
```

**Entscheidend: WO führt man das aus?**
- Auf dem **Server** ausgeführt → zeigt die öffentliche IP **des Servers** (Ziel des Streams).
- Auf dem **Laptop/im Home-Netz** ausgeführt → zeigt die öffentliche IP **deines Anschlusses** – und darüber geht per NAT auch der CML-Router raus. Das ist die Quelle, die in die Firewall-Regel muss.

Autoritativ bestimmen (falls unklar, über welche IP ein Gerät wirklich rauskommt): Port kurz für `Any` öffnen, am Server `tcpdump` mitlaufen lassen und die **Quell-IP** aus dem SYN-Paket ablesen.

---

## 3. Pakete mitschneiden: `tcpdump`

```bash
tcpdump -i any port 57000 -nn
```
- `-i any` = alle Interfaces (praktisch, aber kein Promiscuous-Mode – Warnung ignorieren)
- `port 57000` = Filter (Achtung: sieht nur diesen Port! Falscher Port im Config → man sieht nichts)
- `-nn` = keine DNS-/Port-Namensauflösung (schneller, eindeutiger)

**Den Handshake lesen** (Flags):
- `[S]` = SYN (Verbindungswunsch), `[S.]` = SYN/ACK (Antwort), `[.]` = ACK, `[P.]` = Daten (Push), `[F.]` = FIN (sauberes Zu), `[R]` = RST (hartes Zu)
- Vollständiger Aufbau: `[S]` → `[S.]` → `[.]`

**TCP-Fingerabdruck** – wer verbindet sich da? Aus den SYN-Optionen lässt sich der Gerätetyp erahnen:
- Cisco IOS: kleines `win`, `mss 536`, **keine** SACK/Timestamps/Window-Scaling
- moderner Linux: `mss 1412`, `sackOK`, `wscale`, `TS`

So haben wir in Lab 10 einen **Internet-Scanner** (der auf dem offenen Port anklopfte) vom **echten Router** unterschieden.

**Nur ICMP sehen** (Ping-Debugging): `tcpdump -i any icmp -nn`

---

## 4. Lauschende Ports am Host anzeigen: `ss`

```bash
ss -tlnp               # alle lauschenden TCP-Ports mit Prozess
ss -tlnp | grep 57000  # gezielt
ss -tn state established '( sport = :57000 )'  # bestehende Verbindungen auf dem Port
```
- `-t` TCP, `-l` listening, `-n` numerisch, `-p` Prozess, `-u` wäre UDP

**Stolperfalle:** In minimalen Containern (z. B. `telegraf:latest`) ist `ss` oft gar nicht installiert – dann liefert `docker compose exec telegraf ss …` nichts bzw. „not found", was **nicht** „Port zu" heißt. Besser vom **Host** prüfen oder `docker compose ps` nutzen (zeigt die Port-Mappings).

---

## 5. ICMP-Erreichbarkeit: `ping` / `traceroute`

```bash
ping <host>            # kommt man grundsätzlich hin? (nur ICMP!)
traceroute <host>      # welcher Hop schluckt die Pakete?
```

**Wichtige Lehre aus Lab 10:** `ping` geht ≠ Port offen. Eine Firewall kann ICMP erlauben, aber TCP auf einem bestimmten Port droppen. Ping bestätigt nur die grundsätzliche Route, **nicht** die Port-Erreichbarkeit – dafür braucht es `telnet`/`nc`.

---

## 6. Firewalls verstehen

- **Cloud-Firewall (z. B. Hetzner)** = „default deny" für eingehend: Was nicht in der Regelliste steht, wird **vor** der VM verworfen. Deshalb: ICMP kann durchgehen (eigene Regel), während TCP 57000 fehlt. Der Drop passiert am Netzwerkrand → `tcpdump` auf der VM sieht dann **gar nichts**.
- **Host-Firewall (`ufw`/`nftables`/`iptables`)** auf der VM:
  ```bash
  ufw status verbose
  nft list ruleset
  iptables -L -n -v
  ```
  Unterschied: Ein lokaler DROP würde im `tcpdump` auf `any` **trotzdem auftauchen** (tcpdump hängt vor netfilter). Sieht man das SYN im Dump nicht, sitzt der Block weiter vorne (Cloud-Firewall oder ausgehend beim Absender).

**Docker-Falle (wichtig bei Port 57000):** Ein Container, der mit `-p 57000:57000` (ohne IP-Präfix, also an `0.0.0.0`) veröffentlicht wird, läuft über die `DOCKER-USER`-/FORWARD-Chain – **ufw greift dort nicht**. Eine ufw-Regel für diesen Port bleibt wirkungslos. Die Einschränkung gehört deshalb in die **Cloud-Provider-Firewall** oder eine eigene `DOCKER-USER`-Regel.

**Diagnose-Trick:** Pakete kommen nicht an, aber ein Gerät mit gleicher Quell-IP (z. B. der Laptop) verbindet sich problemlos → der Block ist nicht die Firewall, sondern das sendende Gerät selbst (Config/Routing).

---

## 7. Docker-Compose-Stack prüfen

```bash
docker compose ps                     # laufen alle Container? welche Ports gemappt?
docker compose logs -f telegraf       # live mitlesen
docker compose logs --since 5m telegraf   # nur die letzten 5 Minuten
docker compose exec telegraf <cmd>    # Befehl IM Container ausführen
docker compose config --quiet         # YAML-Syntax prüfen
```

**`Up` vs. `Restarting`:** Hängt ein Container in `Restarting`, ist er nicht einsatzbereit – `docker compose ps` zeigt das. Leere `logs --since` bei einem laufenden Container heißt dagegen nur „nichts zu tun" (z. B. kein Client verbunden).

**localhost-Falle:** In einem Container ist `localhost` der **Container selbst**. Dienste im selben Docker-Netzwerk spricht man über den **Servicenamen** an (`influxdb:8086`, nicht `localhost:8086`).

---

## 8. InfluxDB abfragen

```bash
# landen Daten an? (pro Measurement)
docker compose exec influxdb influx query \
  'from(bucket:"telemetry") |> range(start:-2m) |> filter(fn: (r) => r._measurement == "cpu") |> limit(n:3)'

# welche Measurements gibt es überhaupt?
docker compose exec influxdb influx query \
  'import "influxdata/influxdb/schema" schema.measurements(bucket: "telemetry")'
```
Der zweite Befehl ist Gold wert, um Alias-Probleme zu erkennen: Steht statt `cpu`/`ifstats` ein langer YANG-Pfadname da, greift ein Alias nicht.

---

## 9. Cisco IOS-XE: Router-Troubleshooting

### 9a. Erreichbarkeit vom Router aus prüfen
Der Router ist in der Dial-out-Kette der **Absender** – oft muss man von dort aus testen, nicht nur vom Server.

```
ping 94.130.25.101                      ! ICMP-Erreichbarkeit zum Ziel
ping 94.130.25.101 source Gi1           ! gezielt aus einem bestimmten Interface
ping 94.130.25.101 repeat 100 size 1400 ! Dauerlast / MTU-Test (fürs Dashboard-"Probe aufs Exempel")
telnet 94.130.25.101 57000              ! roher TCP-Porttest nach außen (kein nc auf IOS)
traceroute 94.130.25.101                ! wo bleibt der Pfad hängen?
```
**Falle Quell-Interface/VRF:** Ein `ping` ohne `source` nutzt u. U. ein anderes Interface/die Global Table als der Telemetrie-Dialout. Geht der Ping, aber die Telemetrie nicht, lohnt der Blick, aus welchem Interface/VRF gestreamt wird (`source-address`/`source-vrf` in der Config).

### 9b. Interfaces & Routing
```
show ip interface brief                 ! Überblick: welche IFs up/up, welche IP?
show interfaces GigabitEthernet1        ! Details, Fehler, Counter (für ifstats-Gegenprobe)
show ip route                           ! gibt es eine (Default-)Route zum Ziel?
show ip route 94.130.25.101             ! welche Route greift konkret für dieses Ziel?
show cdp neighbors                       ! Nachbarschaft/Topologie in CML
```
`down/down` in CML ist meist ein nicht „gestartetes"/nicht verbundenes Interface im Lab, nicht zwingend ein Config-Fehler.

### 9c. NAT prüfen (wenn der Router selbst NAT macht)
```
show ip nat translations                ! aktive Übersetzungen (inside ↔ outside/global)
show ip nat statistics                  ! Treffer, konfigurierte Regeln
clear ip nat translation *              ! Übersetzungen zurücksetzen (Vorsicht: trennt Sessions)
```
Merke: Bei NAT gibt es **zwei** relevante Adressen – die Interface-IP (`source-address` der Subscription) und die Adresse **nach** NAT (die in die Server-Firewall muss).

### 9d. Telemetrie-Status
```
show telemetry internal connection          ! Peer, Port, Source, State (Connecting/Active)
show telemetry ietf subscription all        ! Übersicht aller Subscriptions
show telemetry ietf subscription <id> receiver   ! Receiver-Detail (Adresse, Port, Protokoll, State)
show telemetry ietf subscription <id>       ! Filter/Encoding/Sensor-Group
show run | section telemetry                ! komplette Telemetrie-Config im Original
```
- **State `Connecting`** = versucht zu verbinden, schafft es aber nicht (Ziel/Port/Firewall). **`Active`** = streamt.
- `valid` bei einer Subscription heißt nur „Config syntaktisch ok", **nicht** „verbunden".
- Zusammenfassungen trügen: Für Protokoll-/Port-Fehler immer die **Rohconfig** (`show run | section telemetry`) ansehen, nicht nur „valid/passt". Ein einziges falsches Keyword (`5700` statt `57000`, `protocol` falsch, ein `tls` zu viel) erklärt „TCP steht, aber keine Daten".

### 9e. Die wichtigste Falle: running- vs. startup-config
```
show running-config        ! aktuell aktive Config (flüchtig!)
show startup-config        ! wird beim Booten geladen
write memory               ! running → startup sichern  (Kurzform: wr mem / wr)
show archive               ! Config-Versionen (falls archive configure aktiv)
```
Jede Änderung ist zunächst **nur im running-config**. Ohne `write memory` ist sie nach Reload/Neustart **weg** – genau das hat in Lab 10 einen halben Tag Fehlersuche gekostet (Port-Korrektur war nicht gesichert, CML-Neustart lud den alten Stand).

### 9f. Mehr Einblick: debug & logging
```
terminal monitor           ! Log-/Debug-Ausgaben in die aktuelle (SSH-)Sitzung spiegeln
show logging               ! Puffer der letzten Meldungen
debug telemetry ...        ! gezielt Telemetrie-Debug (sparsam! Last!)
undebug all                ! alle Debugs wieder aus (Kurzform: u all)
```
Debug nur kurz und gezielt einschalten – auf produktiven Geräten kann es das Gerät ausbremsen.

---

## Merksätze

1. **Von unten nach oben:** Transport → Protokoll → Daten.
2. **Ping ≠ Port offen** – immer zusätzlich `telnet`/`nc` auf den konkreten Port.
3. **„refused" vs. „hängt"** trennt Dienst- von Firewall-Problem.
4. **tcpdump still trotz offener Regel** → Block sitzt vor der VM (Cloud-Firewall) oder beim Absender.
5. **`localhost` im Container** ≠ der Host – Servicenamen nutzen.
6. **`write memory`** nach jeder Router-Änderung.
7. **Bei NAT zwei Adressen** – Interface-IP vs. Adresse nach NAT; letztere gehört in die Firewall.
8. **Docker-Ports umgehen ufw** – Zugang über die Cloud-Firewall oder `DOCKER-USER` einschränken.
