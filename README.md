# Docker Registry2 mit Tailscale Sidecar (Swarm Stack)

Private Docker Registry (`registry:2`), die ausschließlich über das Tailnet per HTTPS erreichbar ist.

## Aufbau

- `tailscale`: Tailscale im Userspace-Modus (kein `/dev/net/tun`, keine zusätzlichen Capabilities nötig). Terminiert HTTPS via `tailscale serve` und proxyt auf `http://registry:5000`.
- `registry`: `registry:2`, nur im Overlay-Netz `registry` erreichbar, kein Port am Host veröffentlicht.
- Die Serve-Config liegt in [tailscale/serve.json](tailscale/serve.json) und wird als Swarm-Config eingebunden. `${TS_CERT_DOMAIN}` ersetzt Tailscale selbst durch den FQDN des Nodes.
- Beide Services sind per Placement-Constraint an einen Node gebunden, weil die Volumes lokal sind.

## Voraussetzungen

- Docker Swarm (`docker swarm init`)
- In der Tailscale Admin-Console: MagicDNS und HTTPS-Zertifikate aktiviert
- Tailscale Auth Key (am besten tagged, z. B. `tag:registry`)

## Einrichtung

Node für die Registry markieren:

```bash
docker node update --label-add registry=true <node-name>
```

Umgebungsdatei anlegen:

```bash
cp .env.example .env
# TS_AUTHKEY eintragen
```

## Deployen

`docker stack deploy` liest `.env` nicht selbst:

```bash
set -a; source .env; set +a
docker stack deploy -c docker-compose.yml registry
```

Status und Logs:

```bash
docker stack ps registry
docker service logs -f registry_tailscale
```

Der Auth Key wird nur bei der Erstregistrierung gebraucht. Danach liegt der Node-State im Volume `tailscale-state`, das Ablaufen des Keys stört den laufenden Betrieb nicht. Bei einem neuen Key: `.env` anpassen und den Stack erneut deployen.

Ändert sich `tailscale/serve.json`, den Config-Namen in `docker-compose.yml` hochzählen (`registry_serve_v2`), da Swarm-Configs unveränderlich sind.

## Erreichbarkeit testen

```bash
curl -i https://<registry-fqdn>/v2/
```

Erwartet wird `200 OK` mit `{}`. Der FQDN lautet `registry2.<tailnet>.ts.net`.

## Images pushen und pullen

```bash
docker tag my-image:latest <registry-fqdn>/myapp:1.0.0
docker push <registry-fqdn>/myapp:1.0.0
docker pull <registry-fqdn>/myapp:1.0.0
```

## Authentifizierung

Die Registry hat keine eigene Auth, der Zugriff wird allein über Tailscale und die Tailnet-ACLs gesteuert. Beschränke den Zugriff auf Port 443 des Tags `tag:registry` auf die Geräte, die pushen oder pullen sollen. Ein `docker login` ist nicht nötig.

Löschen von Images ist aktiviert (`REGISTRY_STORAGE_DELETE_ENABLED`). Der Speicher wird erst nach einem Garbage-Collect freigegeben:

```bash
docker exec $(docker ps -q -f name=registry_registry) registry garbage-collect /etc/docker/registry/config.yml
```

## Entfernen

```bash
docker stack rm registry
```

Die Volumes `registry_registry-data` und `registry_tailscale-state` bleiben erhalten.
