# Docker Registry2 mit Tailscale Sidecar (Swarm Stack)

Private Docker Registry (`registry:2`), die ausschließlich über das Tailnet per HTTPS erreichbar ist.

## Aufbau

- `tailscale`: Tailscale im Userspace-Modus (kein `/dev/net/tun`, keine zusätzlichen Capabilities nötig). Terminiert HTTPS via `tailscale serve` und proxyt auf `http://registry:5000`.
- `registry`: `registry:2`, nur im Overlay-Netz `registry` erreichbar, kein Port am Host veröffentlicht.
- Die Serve-Config erzeugt der Entrypoint beim Start nach `/tmp/serve.json`. Der Stack braucht dadurch keine Begleitdateien und lässt sich auch direkt im Portainer-Editor deployen. `${TS_CERT_DOMAIN}` ersetzt Tailscale selbst durch den FQDN des Nodes.
- Beide Services sind per Placement-Constraint an einen Node gebunden, weil die Volumes lokal sind.

## Voraussetzungen

- Docker Swarm (`docker swarm init`)
- In der Tailscale Admin-Console: MagicDNS und HTTPS-Zertifikate aktiviert
- Docker Secret `ts_authkey` mit einem Tailscale Auth Key (am besten tagged)

## Einrichtung

Node für die Registry markieren:

```bash
docker node update --label-add registry=true <node-name>
```

## Deployen

```bash
docker stack deploy -c docker-compose.yml registry2
```

Status und Logs:

```bash
docker stack ps registry2
docker service logs -f registry2_tailscale
```

Der Auth Key wird nur bei der Erstregistrierung gebraucht. Danach liegt der Node-State im Volume `tailscale-state`, das Ablaufen des Keys stört den laufenden Betrieb nicht. Der Key kommt aus dem externen Docker Secret `ts_authkey`; ein Entrypoint exportiert ihn als `TS_AUTHKEY`, weil Tailscale ihn nur aus der Umgebung liest. Eine Rotation des Secrets wird dadurch ohne Änderung am Stack übernommen.

## Erreichbarkeit testen

```bash
curl -i https://registry2.tail5a2ccd.ts.net/v2/
```

Erwartet wird `200 OK` mit `{}`. Der Name setzt sich aus `TS_HOSTNAME` (`registry2`) und dem Tailnet-Namen zusammen. Wird er in der Admin-Console geändert oder gibt es einen Namenskonflikt (`registry2-1`), muss er überall unten angepasst werden.

## Registry nutzen

Voraussetzung auf jedem Client (Build-Rechner und Swarm-Nodes): Das Gerät ist im Tailnet und löst Tailnet-Namen auf (Tailscale-DNS, auf Linux-Servern `sudo tailscale set --accept-dns=true`). Das Zertifikat ist ein gültiges Let's-Encrypt-Zertifikat, `insecure-registries` ist nicht nötig.

Im Folgenden steht `REG=registry2.tail5a2ccd.ts.net`.

### Image bauen und pushen

```bash
REG=registry2.tail5a2ccd.ts.net

docker build -t $REG/myapp:1.0.0 .
docker push $REG/myapp:1.0.0
```

Ein bereits vorhandenes lokales Image taggen und pushen:

```bash
docker tag my-local-image:latest $REG/myapp:latest
docker push $REG/myapp:latest
```

Die Architektur muss zum Ziel passen. Wird auf einem Mac (arm64) gebaut und auf einem x86-Node gestartet, entweder die Plattform angeben oder ein Multi-Arch-Image pushen:

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t $REG/myapp:1.0.0 --push .
```

### Image pullen

```bash
docker pull $REG/myapp:1.0.0
```

### Im Stack verwenden

Das Image wird vom Docker-Daemon des Nodes gezogen, nicht vom Container. Der Node braucht also Tailscale und Tailscale-DNS. Da die Registry keine eigene Auth hat, ist `--with-registry-auth` nicht nötig.

```yaml
services:
  app:
    image: registry2.tail5a2ccd.ts.net/myapp:1.0.0
```

Beim Deploy feste Versionen statt `latest` verwenden, sonst bleibt unklar, welcher Stand läuft.

### Inhalt anzeigen

```bash
curl -s https://$REG/v2/_catalog
curl -s https://$REG/v2/myapp/tags/list
```

### Image löschen

Löschen geht über den Digest und gibt den Speicher erst nach einem Garbage-Collect frei (siehe unten):

```bash
DIGEST=$(curl -sI -H 'Accept: application/vnd.docker.distribution.manifest.v2+json' \
  https://$REG/v2/myapp/manifests/1.0.0 | awk -F': ' 'tolower($1)=="docker-content-digest"{print $2}' | tr -d '\r')
curl -X DELETE https://$REG/v2/myapp/manifests/$DIGEST
```

Bei Multi-Arch-Images (Manifest-Liste) den `Accept`-Header `application/vnd.oci.image.index.v1+json` bzw. `application/vnd.docker.distribution.manifest.list.v2+json` verwenden.

## Authentifizierung

Die Registry hat keine eigene Auth, der Zugriff wird allein über Tailscale und die Tailnet-ACLs gesteuert. Beschränke den Zugriff auf Port 443 des Registry-Nodes (Tag des Auth Keys) auf die Geräte, die pushen oder pullen sollen. Ein `docker login` ist nicht nötig.

Löschen von Images ist aktiviert (`REGISTRY_STORAGE_DELETE_ENABLED`). Der Speicher wird erst nach einem Garbage-Collect freigegeben:

```bash
docker exec $(docker ps -q -f name=registry2_registry) registry garbage-collect /etc/docker/registry/config.yml
```

## Entfernen

```bash
docker stack rm registry2
```

Die Volumes `registry2_registry-data` und `registry2_tailscale-state` bleiben erhalten.
