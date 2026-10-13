# Docker Registry2 mit Tailscale Sidecar

Diese Konfiguration stellt eine private Docker Registry (`registry:2`) bereit, die ausschließlich über dein Tailscale-Netzwerk via HTTPS erreichbar ist.

## Voraussetzungen

- Docker + Docker Compose
- Tailscale Auth Key
- Tailnet-FQDN für den Node (z. B. `registry2.example-tailnet.ts.net`)

## Konfiguration

1. Beispiel-Umgebungsdatei kopieren:

   ```bash
   cp .env.example .env
   ```

2. Werte in `.env` setzen:

   - `TS_AUTHKEY`
   - `TS_CERT_DOMAIN`

## Starten

```bash
docker compose up -d
```

## Erreichbarkeit testen

```bash
curl -I https://<TS_CERT_DOMAIN>/v2/
```

Bei Erfolg sollte ein `401 Unauthorized` oder `200`-ähnlicher Registry-Response zurückkommen (beides zeigt in der Regel, dass der Endpoint erreichbar ist).

## Docker Login

```bash
docker login https://<TS_CERT_DOMAIN>
```

Hinweis: Wenn keine zusätzliche Registry-Basic-Auth aktiv ist, ist der Benutzername technisch oft nicht relevant; einige Docker-Clients verlangen trotzdem Eingabe.

## Images pushen

Image bauen und taggen:

```bash
docker build -t <TS_CERT_DOMAIN>/myapp:1.0.0 .
```

Push:

```bash
docker push <TS_CERT_DOMAIN>/myapp:1.0.0
```

## Images pullen

```bash
docker pull <TS_CERT_DOMAIN>/myapp:1.0.0
```

## Bereits vorhandenes lokales Image umtaggen

```bash
docker tag my-local-image:latest <TS_CERT_DOMAIN>/myapp:latest
docker push <TS_CERT_DOMAIN>/myapp:latest
```

## Hinweis zur Authentifizierung

Wenn der Zugriff auf die Registry ausschließlich über Tailscale erfolgt und dein Tailnet per ACLs sauber eingeschränkt ist, reicht in vielen Setups die Tailscale-Authentifizierung bereits aus.

## Automatische Rotation der Tailscale Auth Keys

Da Auth Keys maximal 90 Tage gültig sind, ist eine automatische Rotation sinnvoll.

### Architektur der Rotation

- Das Script [`scripts/rotate-ts-authkey.sh`](/home/raffael/Projekte/registry/scripts/rotate-ts-authkey.sh) verwendet einen Tailscale OAuth Client.
- Es holt damit ein kurzlebiges API-Access-Token (ca. 1h).
- Es erzeugt dann per Tailscale API einen frischen Auth Key.
- Der neue Key wird in [`.env`](/home/raffael/Projekte/registry/.env) als `TS_AUTHKEY` ersetzt.
- Danach wird der `tailscale`-Service via `docker compose up -d tailscale` neu gestartet.

### 1) Rotations-Konfiguration anlegen

```bash
cp .rotation.env.example .rotation.env
chmod 600 .rotation.env
```

Werte in [`.rotation.env`](/home/raffael/Projekte/registry/.rotation.env) setzen:

- `TS_TAILNET` (z. B. `example.com`)
- `TS_OAUTH_CLIENT_ID`
- `TS_OAUTH_CLIENT_SECRET`

Die übrigen Werte können in der Regel so bleiben (`TS_KEY_EXPIRY_SECONDS=2592000` = 30 Tage).

### 2) Script ausführbar machen

```bash
chmod +x scripts/rotate-ts-authkey.sh
```

### 3) Einmaliger manueller Test

```bash
./scripts/rotate-ts-authkey.sh
```

### 4) systemd Automation aktivieren

Die Unit-Dateien liegen in [systemd/](/home/raffael/Projekte/registry/systemd/):

- [tailscale-authkey-rotate.service](/home/raffael/Projekte/registry/systemd/tailscale-authkey-rotate.service)
- [tailscale-authkey-rotate.timer](/home/raffael/Projekte/registry/systemd/tailscale-authkey-rotate.timer)

Installieren und aktivieren:

```bash
sudo cp systemd/tailscale-authkey-rotate.service /etc/systemd/system/
sudo cp systemd/tailscale-authkey-rotate.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now tailscale-authkey-rotate.timer
```

Status prüfen:

```bash
systemctl list-timers | grep tailscale-authkey-rotate
systemctl status tailscale-authkey-rotate.timer
```

### Benötigte OAuth-Berechtigung

Der OAuth Client sollte mindestens die Berechtigung besitzen, Auth Keys zu erzeugen (`auth_keys` Write). Wenn du Tags setzt (z. B. `tag:registry`), muss der OAuth Client diese Tags ebenfalls vergeben dürfen.
