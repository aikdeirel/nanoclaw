# NanoClaw VPS Setup Guide

> Nicht committen — enthält persönliche IP-Adressen und Pfade.

Dieses Dokument richtet NanoClaw auf einem frischen Hetzner-VPS ein (kein laufendes NanoClaw vorhanden).
Für Updates oder Daten-Sync einer bereits laufenden VPS-Installation: siehe `vps-update-guide.md`.

---

## Schritt 0: Eckdaten abfragen

Bevor du beginnst, hole folgende Informationen vom Nutzer ein:

| Variable | Bedeutung | Beispiel |
|----------|-----------|---------|
| `VPS_IP` | Öffentliche IPv4 des Hetzner-Servers | `1.2.3.4` |
| `GITHUB_USER` | GitHub-Username für den Fork-Clone | `aikdeirel` |
| `SSH_KEY_EXISTS` | Existiert bereits ein Ed25519-Key unter `~/.ssh/id_ed25519`? | ja/nein |

---

## Schritt 1: Server bei Hetzner anlegen

**Empfehlung: CX22** (ca. 4,35 €/mo) — 2 vCPU, 4 GB RAM, 40 GB SSD.
Bei 5+ parallelen Agent-Containern: **CX32** (8 GB RAM).

Beim Erstellen in der Hetzner Cloud Console:
- Image: **Ubuntu 24.04**
- SSH Key: öffentlichen Key einfügen (aus Schritt 2)
- **IPv4 kaufen** (~0,50 €/mo) — ohne IPv4 sind GitHub, npm, Docker Hub nicht erreichbar
- IPv6 ist gratis, wird automatisch verwendet

---

## Schritt 2: SSH-Key vorbereiten (lokal)

Falls `SSH_KEY_EXISTS = nein`:

```bash
ssh-keygen -t ed25519 -C "nanoclaw-vps"
# Erzeugt ~/.ssh/id_ed25519 (privat) und ~/.ssh/id_ed25519.pub (öffentlich)
cat ~/.ssh/id_ed25519.pub
# Diesen Wert beim Erstellen des Servers in Hetzner einfügen
```

Falls der Key schon existiert:

```bash
cat ~/.ssh/id_ed25519.pub
# Diesen Wert beim Erstellen des Servers in Hetzner einfügen
```

---

## Schritt 3: Hetzner Firewall einrichten

In der Hetzner Cloud Console: **Firewalls → Firewall erstellen**

Inbound-Regeln:

| Protokoll | Port | Quelle | Zweck |
|-----------|------|--------|-------|
| TCP | 22 | `0.0.0.0/0, ::/0` | SSH |
| ICMP | — | `0.0.0.0/0, ::/0` | Ping |

Ausgehende Regeln: keine (alles erlaubt). Firewall dem Server zuweisen.

---

## Schritt 4: Server-Grundkonfiguration (als root via SSH)

```bash
ssh root@VPS_IP

apt update && apt upgrade -y
apt install -y git

# User anlegen
adduser nanoclaw

# SSH-Key für nanoclaw-User übernehmen
mkdir -p /home/nanoclaw/.ssh
cp ~/.ssh/authorized_keys /home/nanoclaw/.ssh/
chown -R nanoclaw:nanoclaw /home/nanoclaw/.ssh
chmod 700 /home/nanoclaw/.ssh && chmod 600 /home/nanoclaw/.ssh/authorized_keys

# Docker installieren
curl -fsSL https://get.docker.com | sh
usermod -aG docker nanoclaw

# Tailscale installieren und aktivieren
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up  # Link im Browser öffnen und autorisieren
tailscale ip  # Tailscale-IP merken

# Service beim Boot ohne Login starten
loginctl enable-linger nanoclaw
```

---

## Schritt 5: Node.js & Claude Code (als nanoclaw-User)

```bash
ssh nanoclaw@VPS_IP

# nvm + Node 22
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install 22 && nvm use 22

# Claude Code CLI
npm install -g @anthropic-ai/claude-code

# Einmalig einloggen — erzeugt ~/.claude/.credentials.json
claude
```

---

## Schritt 6: NanoClaw klonen und bauen

```bash
# Weiterhin als nanoclaw auf dem VPS:
git clone https://github.com/GITHUB_USER/nanoclaw.git ~/nanoclaw
cd ~/nanoclaw

npm install
npm run build
./container/build.sh
```

---

## Schritt 7: .env anlegen

```bash
cat > ~/nanoclaw/.env << 'EOF'
ASSISTANT_NAME=Nanoclaw
TELEGRAM_BOT_TOKEN=<token>
TELEGRAM_BOT_POOL=<token1>,<token2>,...
BRAVE_API_KEY=<key>
DEFAULT_MODEL=claude-sonnet-4-6
EOF
```

OAuth-Token aus den frischen Claude-Credentials extrahieren und anhängen:

```bash
python3 -c "
import json
with open('/home/nanoclaw/.claude/.credentials.json') as f:
    d = json.load(f)
token = d['claudeAiOauth']['accessToken']
print(f'CLAUDE_CODE_OAUTH_TOKEN={token}')
" >> ~/nanoclaw/.env
```

> **Warum?** NanoClaw startet einen HTTP-Proxy, der API-Calls der Agent-Container authentifiziert.
> Der Proxy liest `CLAUDE_CODE_OAUTH_TOKEN` aus `.env`. Ohne diesen Token antwortet der Agent
> mit „Not logged in · Please run /login".
>
> **Langfristige Lösung:** Einen Anthropic API-Key in `.env` als `ANTHROPIC_API_KEY=sk-ant-api...`
> eintragen — dann läuft der Credential-Proxy im API-Key-Modus ohne Token-Ablauf.

---

## Schritt 8: systemd User-Service einrichten

```bash
mkdir -p ~/.config/systemd/user ~/nanoclaw/logs ~/nanoclaw/store

# node-Pfad ermitteln
which node  # z.B. /home/nanoclaw/.nvm/versions/node/v22.22.1/bin/node
# Den ermittelten Pfad in ExecStart unten eintragen:

cat > ~/.config/systemd/user/nanoclaw.service << 'EOF'
[Unit]
Description=NanoClaw Personal Assistant
After=network.target

[Service]
Type=simple
ExecStart=/home/nanoclaw/.nvm/versions/node/v22.22.1/bin/node /home/nanoclaw/nanoclaw/dist/index.js
WorkingDirectory=/home/nanoclaw/nanoclaw
EnvironmentFile=/home/nanoclaw/nanoclaw/.env
Restart=always
RestartSec=5
KillMode=process
Environment=HOME=/home/nanoclaw
Environment=PATH=/usr/local/bin:/usr/bin:/bin:/home/nanoclaw/.local/bin
StandardOutput=append:/home/nanoclaw/nanoclaw/logs/nanoclaw.log
StandardError=append:/home/nanoclaw/nanoclaw/logs/nanoclaw.error.log

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable nanoclaw
systemctl --user start nanoclaw
systemctl --user status nanoclaw
```

---

## Schritt 9: Verifikation

```bash
# groupCount im Log prüfen — muss beim ersten Start > 0 sein (wenn DB schon befüllt),
# oder 0 wenn frische Installation (Gruppen müssen erst registriert werden)
tail -20 ~/nanoclaw/logs/nanoclaw.log
```

---

## Schritt 10: Gruppen registrieren

Schreib dem Bot `/chatid` im jeweiligen Telegram-Chat, dann die Chat-ID über das IPC-System
registrieren oder via NanoClaw-Claude-Code-Session direkt in der DB eintragen.

---

## Troubleshooting

### Agent antwortet „Not logged in · Please run /login"
→ `CLAUDE_CODE_OAUTH_TOKEN` fehlt oder ist abgelaufen. Token neu extrahieren (Schritt 7) und
Service neu starten:
```bash
ssh nanoclaw@VPS_IP "systemctl --user restart nanoclaw"
```

### Container startet nicht (Image not found)
```bash
ssh nanoclaw@VPS_IP "cd ~/nanoclaw && ./container/build.sh"
```

### „groupCount: 0" nach Daten-Migration
→ DB liegt falsch. Muss nach `~/nanoclaw/store/messages.db` (nicht Projekt-Root):
```bash
ssh nanoclaw@VPS_IP "
  systemctl --user stop nanoclaw
  mv ~/nanoclaw/messages.db ~/nanoclaw/store/messages.db 2>/dev/null || true
  systemctl --user start nanoclaw
"
```

---

## Checkliste

- [ ] Infrastruktur: Hetzner-Server CX22, Ubuntu 24.04, IPv4, SSH-Key hinterlegt
- [ ] Firewall: TCP 22 + ICMP inbound
- [ ] Server-Setup: User `nanoclaw`, Docker, Tailscale, `loginctl enable-linger`
- [ ] Node.js 22 via nvm, Claude Code installiert und eingeloggt
- [ ] Repo geklont, `npm install && npm run build` erfolgreich
- [ ] Container gebaut (`./container/build.sh`)
- [ ] `.env` vollständig (inkl. `CLAUDE_CODE_OAUTH_TOKEN`)
- [ ] systemd Service aktiv (`systemctl --user status nanoclaw`)
- [ ] Service-Log sieht sauber aus (kein Auth-Fehler)
- [ ] Testnachricht auf Telegram funktioniert
