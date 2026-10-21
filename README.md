# DM DNS

DNS-Server mit Werbe- und Tracking-Blocking für das eigene Netz, ähnlich Pi-hole und
AdGuard Home. DM DNS läuft als einzelner Container, lässt sich über eine Weboberfläche
(Deutsch und Englisch) bedienen und auf Wunsch als Cluster mit virtueller IP betreiben.

- Filterlisten (Hagezi vorausgewählt, eigene Listen im Adblock-, Hosts- und Domain-Format)
- Allow- und Blocklist, lokale DNS-Einträge, Clients und Gruppen mit eigenen Regeln
- Weiterleitung per UDP, DNS-over-TLS oder DNS-over-HTTPS, optional rekursiv mit Unbound
- Query-Log, Statistik und Ereignisse
- Cluster: eine Konfiguration für alle Nodes, Hochverfügbarkeit per Keepalived (VIP)
- Update-Hinweis in der Oberfläche, sobald ein neues Image bereitsteht

Der Quellcode ist nicht öffentlich. Dieses Repository enthält nur diese Beschreibung; das
Image liegt öffentlich unter `ghcr.io/apps-dm/dm-dns` (nur `linux/amd64`).

## Schnellstart mit Docker Compose

Voraussetzungen: Linux-Host (VM oder LXC) mit Docker und Compose v2, Port 53 frei.

```sh
sudo mkdir -p /opt/dm-dns && cd /opt/dm-dns
# dmdns läuft im Container als UID/GID 10001
sudo install -d -o 10001 -g 10001 -m 0750 data
```

`/opt/dm-dns/docker-compose.yml`:

```yaml
name: dm-dns
services:
  dmdns:
    image: ghcr.io/apps-dm/dm-dns:latest
    pull_policy: always
    network_mode: host # DNS auf Port 53 und echte Client-IPs
    user: "0:0" # startet als root und wechselt sofort auf UID 10001
    restart: unless-stopped
    read_only: true
    cap_drop: [ALL]
    cap_add: [NET_BIND_SERVICE, SETUID, SETGID]
    security_opt: ["no-new-privileges:true"]
    tmpfs:
      - /tmp:rw,noexec,nosuid,nodev,size=32m
      - /run:rw,noexec,nosuid,nodev,size=8m,uid=10001,gid=10001,mode=0750
    volumes:
      - ./data:/data
    mem_limit: 512m
    healthcheck:
      test: ["CMD", "dmdns", "health"]
      interval: 15s
      timeout: 5s
      start_period: 30s
      retries: 3
```

Starten:

```sh
docker compose up -d
docker compose logs -f   # zeigt den SHA-256-Fingerabdruck des Zertifikats
```

Dann `https://<server>:8443` öffnen. Die Weboberfläche nutzt ein selbstsigniertes
Zertifikat; den Fingerabdruck in der Browserwarnung mit dem aus dem Log vergleichen. Der
Setup-Assistent führt durch die Ersteinrichtung.

Belegt der Stub-Resolver von `systemd-resolved` Port 53 (`sudo ss -lunp 'sport = :53'`),
ihn vor dem Start abschalten:

```sh
sudo mkdir -p /etc/systemd/resolved.conf.d
printf '[Resolve]\nDNSStubListener=no\n' | sudo tee /etc/systemd/resolved.conf.d/dm-dns.conf
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved
```

## Cluster und Hochverfügbarkeit

Mehrere Nodes teilen sich eine Konfiguration: auf dem ersten Node unter **Cluster** den
Cluster initialisieren, „Node hinzufügen“ erzeugt einen Join-Token für den nächsten Node.
Port 8853/TCP muss zwischen den Nodes erreichbar sein.

Für eine virtuelle IP (Keepalived) auf **jedem** Node `NET_ADMIN` und `NET_RAW` unter
`cap_add` ergänzen, alle Nodes ins selbe Layer-2-Netz stellen und VRRP (IP-Protokoll 112)
zwischen ihnen erlauben. Danach die Seite **Hochverfügbarkeit** einrichten.

## Update

```sh
cd /opt/dm-dns
docker compose pull && docker compose up -d
```

Im Cluster nacheinander aktualisieren: erst die Secondaries, dann den Primary. Jeweils
warten, bis `docker compose ps` „healthy“ zeigt. Neben `latest` gibt es jede Version als
eigenen Tag (z. B. `ghcr.io/apps-dm/dm-dns:0.2.101`), um gezielt zu wechseln.

DM DNS prüft selbst stündlich, ob ein neueres Image bereitsteht, und zeigt dann unten
links neben der Version „Update: v…“. Die Prüfung lässt sich unter **Einstellungen →
Updates** abschalten.

---

## English summary

DM DNS is a self-hosted DNS server with ad and tracker blocking (similar to Pi-hole and
AdGuard Home), a German/English web UI, optional recursive resolving with Unbound, and
clustering with a Keepalived virtual IP. The source code is not public; this repository
only holds this description. The image is `ghcr.io/apps-dm/dm-dns:latest` (linux/amd64).
Use the Compose file above, open `https://<server>:8443` and follow the setup wizard.
Update with `docker compose pull && docker compose up -d` (in a cluster: secondaries
first, then the primary).
