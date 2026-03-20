---
name: vps-setup
description: Run initial NanoClaw setup on a fresh Hetzner VPS. Use when the user wants to install NanoClaw on a new server, provision a VPS, or set up a remote Linux instance from scratch.
---

# NanoClaw VPS Setup

Richtet NanoClaw auf einem frischen Hetzner-VPS ein. Läuft lokal — alle Remote-Befehle gehen per SSH auf den VPS. Nur pausieren wenn echte Nutzeraktion nötig ist (Hetzner-Console, Telegram-Auth).

## Schritt 0: Eckdaten abfragen

Frage mit `AskUserQuestion` ab:
- **VPS_IP** — öffentliche IPv4 des Hetzner-Servers (erst nach Schritt 1 bekannt, ggf. später)
- **GITHUB_USER** — GitHub-Username für den Fork-Clone
- **SSH_KEY_EXISTS** — existiert `~/.ssh/id_ed25519` bereits? (`ls ~/.ssh/id_ed25519.pub` lokal prüfen)

## Schritt 1: Server bei Hetzner anlegen (Nutzeraktion)

Empfehlung: **CX22** (2 vCPU, 4 GB RAM, ~4,35 €/mo). Bei 5+ parallelen Containern: **CX32**.

Nutzer muss in der Hetzner Cloud Console:
- Image: **Ubuntu 24.04**
- **IPv4 kaufen** (~0,50 €/mo) — ohne IPv4 kein GitHub/npm/Docker Hub
- SSH Key einfügen (aus Schritt 2)
- Firewall zuweisen (aus Schritt 3)

## Schritt 2: SSH-Key vorbereiten (lokal)

```bash
# Falls kein Key existiert:
ssh-keygen -t ed25519 -C "nanoclaw-vps"

# Public Key anzeigen — dieser kommt in Hetzner:
cat ~/.ssh/id_ed25519.pub
```

## Schritt 3: Hetzner Firewall

In Hetzner Console: **Firewalls → Firewall erstellen**, dann dem Server zuweisen.

Inbound: TCP Port 22 (`0.0.0.0/0, ::/0`) + ICMP (`0.0.0.0/0, ::/0`). Outbound: alles erlaubt.

## Schritt 4: Server-Grundkonfiguration (als root via SSH)

```bash
ssh root@VPS_IP "
  apt update && apt upgrade -y
  apt install -y git
  adduser --disabled-password --gecos '' nanoclaw
  mkdir -p /home/nanoclaw/.ssh
  cp ~/.ssh/authorized_keys /home/nanoclaw/.ssh/
  chown -R nanoclaw:nanoclaw /home/nanoclaw/.ssh
  chmod 700 /home/nanoclaw/.ssh && chmod 600 /home/nanoclaw/.ssh/authorized_keys
  curl -fsSL https://get.docker.com | sh
  usermod -aG docker nanoclaw
  loginctl enable-linger nanoclaw
"
```

Tailscale muss interaktiv eingerichtet werden (Browser-Auth):
```bash
ssh root@VPS_IP "curl -fsSL https://tailscale.com/install.sh | sh && tailscale up"
# Nutzer muss Link im Browser öffnen und autorisieren
```

## Schritt 5: Node.js & Claude Code (als nanoclaw)

```bash
ssh nanoclaw@VPS_IP "bash -l -c '
  curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
  source ~/.bashrc
  nvm install 22 && nvm use 22
  npm install -g @anthropic-ai/claude-code
'"
# Danach: ssh nanoclaw@VPS_IP "claude" — Login-Flow, erzeugt ~/.claude/.credentials.json
```

Claude-Login erfordert Nutzeraktion im Terminal.

## Schritt 6: Repo klonen und bauen

```bash
ssh nanoclaw@VPS_IP "bash -l -c '
  git clone https://github.com/GITHUB_USER/nanoclaw.git ~/nanoclaw
  cd ~/nanoclaw
  npm install
  npm run build
  ./container/build.sh
'"
```

## Schritt 7: .env anlegen und OAuth-Token setzen

```bash
# .env mit Basis-Werten anlegen (Nutzer nach fehlenden Tokens fragen):
ssh nanoclaw@VPS_IP "cat > ~/nanoclaw/.env << 'EOF'
ASSISTANT_NAME=Nanoclaw
TELEGRAM_BOT_TOKEN=<token>
BRAVE_API_KEY=<key>
DEFAULT_MODEL=claude-sonnet-4-6
EOF"

# OAuth-Token aus frischen Credentials extrahieren:
ssh nanoclaw@VPS_IP "python3 -c \"
import json
with open('/home/nanoclaw/.claude/.credentials.json') as f:
    d = json.load(f)
token = d['claudeAiOauth']['accessToken']
with open('/home/nanoclaw/nanoclaw/.env', 'a') as f:
    f.write(f'CLAUDE_CODE_OAUTH_TOKEN={token}\n')
print('Token gesetzt.')
\""
```

> **Langfristige Lösung:** `ANTHROPIC_API_KEY=sk-ant-api...` in `.env` — kein Token-Ablauf.

## Schritt 8: systemd User-Service

```bash
ssh nanoclaw@VPS_IP "bash -l -c '
  NODE_PATH=\$(which node)
  mkdir -p ~/.config/systemd/user ~/nanoclaw/logs ~/nanoclaw/store
  cat > ~/.config/systemd/user/nanoclaw.service << EOF
[Unit]
Description=NanoClaw Personal Assistant
After=network.target

[Service]
Type=simple
ExecStart=\${NODE_PATH} /home/nanoclaw/nanoclaw/dist/index.js
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
'"
```

## Schritt 9: Verifikation

```bash
ssh nanoclaw@VPS_IP "systemctl --user status nanoclaw --no-pager"
ssh nanoclaw@VPS_IP "tail -20 ~/nanoclaw/logs/nanoclaw.log"
```

Kein „Not logged in" im Log → Setup erfolgreich.

## Troubleshooting

**„Not logged in · Please run /login"** → OAuth-Token fehlt/abgelaufen. Schritt 7 (Token-Extraktion) wiederholen, dann `systemctl --user restart nanoclaw`.

**Container startet nicht (Image not found)** → `ssh nanoclaw@VPS_IP "cd ~/nanoclaw && ./container/build.sh"`

**groupCount: 0 nach DB-Migration** → DB liegt im Projekt-Root statt in `store/`. Korrigieren: `mv ~/nanoclaw/messages.db ~/nanoclaw/store/messages.db`

## Checkliste

- [ ] Hetzner CX22, Ubuntu 24.04, IPv4, SSH-Key, Firewall
- [ ] User `nanoclaw`, Docker, linger, Tailscale
- [ ] Node 22 via nvm, Claude Code eingeloggt
- [ ] Repo geklont, gebaut, Container gebaut
- [ ] `.env` vollständig inkl. `CLAUDE_CODE_OAUTH_TOKEN`
- [ ] systemd Service aktiv
- [ ] Log sauber, Testnachricht auf Telegram funktioniert
