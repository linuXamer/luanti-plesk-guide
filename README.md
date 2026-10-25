# Luanti mit VoxeLibre auf einem Plesk-Server (Docker Stacks)

> **Short version in English:** This guide is in German. It documents a working setup for a small, private Luanti server with the VoxeLibre game on a Plesk-managed VM, deployed through Plesk Docker Stacks. It runs fine on a tiny 4 vCPU / 8 GB RAM machine and keeps all game data out of the public web roots. Copy whatever is useful to you – no guarantees.

## Warum dieses Repo

Ich betreibe neben ein paar Websites auf einer kleinen VM (4 vCPU, 8 GB RAM) einen Luanti-Server mit VoxeLibre für eine kleine Gruppe. Es läuft, es macht Spaß, und es ist kein Hexenwerk – aber es gibt ein paar Stellen, an denen man sich sauber verhaken kann: falsche Benutzer-IDs, falscher Konfigurationspfad, Spieldaten im falschen Ordner. Diese Anleitung schreibt auf, was bei mir funktioniert, inklusive der Fehler, die ich vorher gemacht habe.

Wer etwas Ähnliches nachbauen will, kann sich hier alles abschauen. Keine Garantie, dass es überall genauso läuft – die getestete Umgebung steht unten.

## Für wen das interessant ist

- Plesk-Nutzer, die neben ihren Websites einen Spielserver betreiben wollen
- Einrichtungen, Medienpädagogik, Workshops: eine kostenlose gemeinsame Bauwelt
- Selfhosting-Einsteiger, die mehr wollen als eine nackte Compose-Datei

## Getestete Umgebung

- Kleine VM mit 4 vCPU und 8 GB RAM – für eine kleine Gruppe locker ausreichend
- Plesk mit Docker-Erweiterung und Stacks-Funktion
- Luanti über das LinuxServer-Image `lscr.io/linuxserver/luanti:latest` (Stand: Oktober 2026)
- VoxeLibre vom offiziellen Repository, Standardbranch (Stand: Oktober 2026)

Ich habe bei der Erstinstallation nicht auf gepinnte Versionen geachtet. Wer es langfristig reproduzierbar will, sollte einen konkreten Image-Tag und einen VoxeLibre-Release bzw. Commit festhalten und notieren (Befehle dazu weiter unten).

## Aufbau und Sicherheitsprinzip

Die Spieldaten liegen außerhalb aller öffentlichen Webseitenverzeichnisse:

```text
/var/www/vhosts/example.org/
├── httpdocs/                  # öffentliche Hauptwebsite
├── luanti.example.org/        # öffentliche Subdomain (optionale Info-Seite)
│   └── index.html
└── luanti-data/               # Spieldaten – niemals als Dokumentenstamm verwenden
    ├── games/
    │   └── voxelibre/
    ├── main-config/
    │   └── minetest.conf
    ├── worlds/
    └── mod_data/              # wird je nach Spiel automatisch angelegt
```

Der Container bindet ausschließlich `luanti-data` nach `/config/.minetest` – dort erwartet das LinuxServer-Image seine Konfiguration, Spiele und Welten ([docs.linuxserver.io](https://docs.linuxserver.io/images/docker-luanti/)).

Die optionale `index.html` unter `luanti.example.org` ist nur eine Informationsseite (Serveradresse, Port, Client-Download, Teilnahmehinweise). Keine Passwörter, privaten Konfigurationen oder Spielerdaten dort. Gespielt wird über den Luanti-Client, nicht im Browser.

## Voraussetzungen

- Linux-Server mit administrativem SSH-Zugang
- Plesk mit Docker-Erweiterung und Stacks-Funktion
- Domain (hier als Beispiel: `example.org`) und Subdomain `luanti.example.org`
- Installiertes `git` auf dem Server
- Luanti-Client auf den Rechnern aller Teilnehmenden

## Schritt 1: Subdomain anlegen (optional, nur für die Info-Seite)

In Plesk: **Websites & Domains → Subdomain hinzufügen**

```text
Subdomain:         luanti
Hauptdomain:       example.org
Dokumentenstamm:   luanti.example.org
```

Danach prüfen, dass der öffentliche Ordner tatsächlich unter `/var/www/vhosts/example.org/luanti.example.org` liegt. `luanti-data` darf niemals als Dokumentenstamm eingestellt werden.

## Schritt 2: DNS einstellen

Beim tatsächlich zuständigen DNS-Anbieter (in Plesk oder beim externen Provider):

```text
Typ:      A
Name:     luanti
Ziel:     öffentliche IPv4-Adresse des Spielservers
```

Ein vorhandener AAAA-Eintrag sollte nur auf eine IPv6-Adresse zeigen, über die der Server wirklich erreichbar ist. Der Client verbindet sich direkt mit **UDP-Port 30000** – ein HTTP-Reverse-Proxy wird für den Spielbetrieb nicht gebraucht.

## Schritt 3: Spieldaten per SSH vorbereiten

Als `root` anmelden. Zuerst prüfen, ob `luanti-data` bereits existiert – eine bestehende Installation nicht mit den Neueinrichtungsbefehlen überschreiben:

```bash
ls -ld /var/www/vhosts/example.org/luanti-data
```

Verzeichnisse anlegen und VoxeLibre klonen:

```bash
mkdir -p /var/www/vhosts/example.org/luanti-data/games
mkdir -p /var/www/vhosts/example.org/luanti-data/main-config

cd /var/www/vhosts/example.org/luanti-data/games
git clone https://git.minetest.land/VoxeLibre/VoxeLibre.git voxelibre
```

Der Ordnername `voxelibre` wird später als `--gameid voxelibre` verwendet.

Genutzten Commit anzeigen und für die eigene Doku notieren:

```bash
git -C /var/www/vhosts/example.org/luanti-data/games/voxelibre rev-parse HEAD
```

## Schritt 4: PUID und PGID ermitteln (der wichtigste Schritt)

Das LinuxServer-Image schreibt Dateien nicht als root, sondern als die über `PUID`/`PGID` angegebene Identität. Diese muss zum Besitzer der eingebundenen Dateien passen – sonst gibt es Schreibrechte-Fehler oder Dateien, die man später nicht mehr sauber verwalten kann ([docs.linuxserver.io](https://docs.linuxserver.io/images/docker-luanti/)).

Die eigenen Werte ermittelt man am Systembenutzer des Plesk-Abonnements:

```bash
stat -c 'UID=%u GID=%g' /var/www/vhosts/example.org/httpdocs
```

In dieser Anleitung sind `PUID=10000` und `PGID=1003` **nur Beispielwerte**. Auf dem eigenen Server neu ermitteln und überall ersetzen: im `chown`-Befehl und in den Umgebungsvariablen der Compose-Datei.

## Schritt 5: minetest.conf anlegen

Die Datei `minetest.conf.example` aus diesem Repo nach `/var/www/vhosts/example.org/luanti-data/main-config/minetest.conf` kopieren und die beiden Platzhalter ersetzen:

- `name`: exakter Spielername, der Adminrechte bekommen soll
- `default_password`: ein eigenes Gruppenpasswort für neue Konten

Wichtige Hinweise dazu:

- `disallow_empty_password` wird ohne abschließendes `s` geschrieben.
- `default_password` ist das Anfangspasswort für **neue** Konten, kein zusätzliches Passwort vor bestehenden Konten. Bestehende Konten behalten ihr eigenes Passwort, wenn der Wert später geändert wird.
- `server_announce = false` versteckt den Server nur aus der öffentlichen Serverliste – es ist **kein** Zugangsschutz.
- `max_users` ist nur eine Obergrenze, keine Aussage darüber, was die Hardware flüssig versorgen kann.
- Modul-Freigaben wie `secure.trusted_mods` nur nach ausdrücklicher Vorgabe des verwendeten Spiels setzen, nicht als „Optimierung".

## Schritt 6: Besitzer und Berechtigungen setzen

Mit den selbst ermittelten IDs (hier als Beispiel 10000:1003):

```bash
chown -R 10000:1003 /var/www/vhosts/example.org/luanti-data
chmod 600 /var/www/vhosts/example.org/luanti-data/main-config/minetest.conf

stat -c '%U:%G | UID=%u GID=%g | %n' \
  /var/www/vhosts/example.org/luanti-data \
  /var/www/vhosts/example.org/luanti-data/main-config/minetest.conf
```

Nicht den gesamten Webspace oder andere Websites mit `chown` anfassen – die IDs beziehen sich ausschließlich auf den Luanti-Datenordner.

## Schritt 7: Stack in Plesk anlegen

In Plesk: **Docker → Stacks → Stack hinzufügen**, dann den Inhalt von `compose.yaml` aus diesem Repo einfügen und anpassen:

- `PUID`/`PGID` auf die eigenen Werte setzen
- Volume-Pfad auf die eigene Domain ändern
- `--worldname` nach Geschmack

Danach den Stack bereitstellen. Für eine reproduzierbare Installation `latest` durch einen konkreten, getesteten Image-Tag ersetzen und den Tag notieren.

## Schritt 8: Netzwerkzugang prüfen

Der Spielserver braucht:

```text
Port:       30000
Protokoll:  UDP
Richtung:   eingehend
```

Falls eine Provider-Firewall aktiv ist, den Verkehr dort freigeben. Wichtig: Ein veröffentlichtes Docker-Portmapping kann bestimmte Host-Firewall-Regeln umgehen. Deshalb nicht allein aus der Plesk-Firewallanzeige ableiten, ob der Port von außen erreichbar ist – das ernsthaft von außen testen.

## Schritt 9: Container prüfen

Volume kontrollieren:

```bash
docker inspect luanti-server \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

Erwartet: `/var/www/vhosts/example.org/luanti-data -> /config/.minetest`

Benutzer-IDs kontrollieren:

```bash
docker inspect luanti-server \
  --format '{{range .Config.Env}}{{println .}}{{end}}' \
  | grep -E '^(PUID|PGID)='
```

Logs ansehen:

```bash
docker logs --tail 100 luanti-server
```

Fehler zu fehlenden Spielen, Schreibrechten oder Konfigurationsdateien zuerst beheben. Ein gestarteter Container allein bestätigt noch nicht, dass der Spielserver richtig arbeitet.

## Schritt 10: Erstmals als Administrator anmelden

Im Luanti-Client:

```text
Adresse:  luanti.example.org
Port:     30000
Name:     exakt der Name aus name = ...
Passwort: das gesetzte Gruppenpasswort
```

Nach dem Verbinden mit `/admin` und `/privs` kontrollieren, dass der Name Adminrechte hat. Direkt anschließend im Spielmenü ein eigenes, einzigartiges Admin-Passwort setzen – und erst danach den Gruppenzugang weitergeben.

## Schritt 11: Weitere Spieler einladen

Alle brauchen einen Luanti-Client auf ihrem Gerät. Zugangsdaten: Adresse, Port 30000, eindeutiger Spielername, Gruppenpasswort beim ersten Zugang.

VoxeLibre muss nicht zusätzlich auf den Clients installiert werden – die benötigten Medien werden beim Verbinden vom Server geladen, der erste Beitritt kann entsprechend länger dauern. Client- und Serverversion aufeinander abstimmen ([docs.linuxserver.io](https://docs.linuxserver.io/images/docker-luanti/)).

## Betrieb

- **Bei Nichtgebrauch:** Container über Plesk stoppen. Die Daten bleiben im eingebundenen Verzeichnis erhalten. Zum Weitermachen einfach wieder starten.
- **minetest.conf ändern:** Container stoppen → Datei bearbeiten → Container starten.
- **Compose ändern:** Bei Änderungen an PUID, PGID, Volumes oder anderen Parametern den Stack aktualisieren und den Container **neu erstellen** lassen. Ein einfacher Neustart übernimmt solche Änderungen nicht zuverlässig.
- **Updates:** Erst Backup ziehen, dann neues Image abrufen und Container neu erstellen. Nicht ungeprüft direkt vor einem Termin/Workshop updaten.

## Backup

Für eine einfache konsistente Sicherung den Container vorher stoppen. Als `root`:

```bash
BACKUP="/root/luanti-backup-$(date +%Y%m%d-%H%M%S).tar.gz"

tar -czpf "$BACKUP" \
  -C /var/www/vhosts/example.org \
  luanti-data

chmod 600 "$BACKUP"
```

Danach den Container wieder starten. Zusätzlich sichern: Compose-Datei, verwendeter Image-Tag, verwendeter VoxeLibre-Commit. Eine Kopie außerhalb des Servers aufbewahren – und nie in einem öffentlichen Webseitenordner speichern.

## Typische Fehler (alle selbst getestet)

- **PUID/PGID blind übernommen** → Schreibrechte-Chaos im Datenordner. Immer selbst ermitteln.
- **Datenordner als Dokumentenstamm der Subdomain eingestellt** → Spieldaten würden öffentlich ausgeliefert. `luanti-data` gehört zu keiner Website.
- **Konfiguration im falschen Pfad erwartet** → Container findet Spiel oder Konfiguration nicht. Das Image erwartet alles unter `/config/.minetest`.
- **`disallow_empty_password` mit „s" geschrieben** → Option wird stillschweigend ignoriert.
- **`server_announce = false` als Zugangsschutz missverstanden** → versteckt nur in der öffentlichen Serverliste.
- **Auf die Plesk-Firewallanzeige verlassen trotz Docker-Portmapping** → Docker kann Host-Regeln umgehen, Port von außen testen.

## Grenzen

- Das ist eine Praxisanleitung, kein Sicherheitsrezept. Wer den Zugang wirklich begrenzen will, muss Zugangsdaten vertraulich austauschen – versteckt ist nicht geschützt.
- `max_users` sagt nichts über die tatsächliche Leistungsfähigkeit der Hardware aus.
- `latest` und der VoxeLibre-Standardbranch können sich jederzeit ändern; getestet ist jeweils der Stand von Oktober 2026.

## Quellen

- LinuxServer Luanti-Image: https://docs.linuxserver.io/images/docker-luanti/
- VoxeLibre: https://git.minetest.land/VoxeLibre/VoxeLibre
- Luanti: https://www.luanti.org
- Plesk, Dokumentenstamm ändern: https://support.plesk.com/hc/en-us/articles/12377087631255

## Lizenz

Anleitungstext (README und weitere Dokumentation): **CC BY 4.0**.
Beispieldateien (`compose.yaml`, `minetest.conf.example`): **MIT**.
Details in der LICENSE-Datei.
