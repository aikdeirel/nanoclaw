# NanoClaw VPS Setup Guide

> Nicht committen — enthält persönliche IP-Adressen und Pfade.

Zwei Szenarien: **Ersteinrichtung** (neuer VPS, kein laufendes Nanoclaw) und **Migration** (bestehende Installation auf Laptop/anderem Server umziehen).

---

## Teil A: Ersteinrichtung

### A1. Server bei Hetzner

**Empfehlung: CX22** (ca. 4,35 €/mo) — 2 vCPU, 4 GB RAM, 40 GB SSD.
Bei 5+ parallelen Agent-Containern: **CX32** (8 GB RAM).

Beim Erstellen:
- Image: **Ubuntu 24.04**
- SSH Key: öffentlichen Key einfügen
- **IPv4 kaufen** (~0,50 €/mo) — ohne IPv4 sind GitHub, npm, Docker Hub nicht erreichbar
- IPv6 ist gratis, wird automatisch verwendet

### A2. SSH-Key vorbereiten (Laptop)

```bash
ssh-keygen -t ed25519 -C "nanoclaw-vps"
# ~/.ssh/id_ed25519 (privat) und ~/.ssh/id_ed25519.pub (öffentlich)
```

### A3. Hetzner Firewall

In der Hetzner Cloud Console: **Firewalls → Firewall erstellen**

Inbound-Regeln:

| Protokoll | Port | Quelle | Zweck |
|-----------|------|--------|-------|
| TCP | 22 | `0.0.0.0/0, ::/0` | SSH |
| ICMP | — | `0.0.0.0/0, ::/0` | Ping |

Ausgehende Regeln: keine (alles erlaubt). Firewall dem Server zuweisen.

### A4. Server einrichten (als root)

```bash
ssh root@<VPS-IP>

apt update && apt upgrade -y
apt install -y git

# User anlegen
adduser nanoclaw
usermod -aG docker nanoclaw  # nach Docker-Installation

# SSH-Key übertragen
mkdir -p /home/nanoclaw/.ssh
cp ~/.ssh/authorized_keys /home/nanoclaw/.ssh/
chown -R nanoclaw:nanoclaw /home/nanoclaw/.ssh
chmod 700 /home/nanoclaw/.ssh && chmod 600 /home/nanoclaw/.ssh/authorized_keys

# Docker installieren
curl -fsSL https://get.docker.com | sh
usermod -aG docker nanoclaw

# Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up  # Link im Browser öffnen und autorisieren
tailscale ip  # Tailscale-IP merken

# Service beim Boot ohne Login starten
loginctl enable-linger nanoclaw
```

### A5. Node.js & Claude Code (als nanoclaw)

```bash
ssh nanoclaw@<VPS-IP>

# nvm + Node 22
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install 22 && nvm use 22

# Claude Code CLI
npm install -g @anthropic-ai/claude-code

# Einmalig einloggen — erzeugt ~/.claude/.credentials.json
claude
```

### A6. NanoClaw klonen und bauen

```bash
git clone https://github.com/<username>/nanoclaw.git ~/nanoclaw
cd ~/nanoclaw

npm install
npm run build
./container/build.sh
```

### A7. .env anlegen

```bash
cat > ~/nanoclaw/.env << 'EOF'
ASSISTANT_NAME=Nanoclaw
TELEGRAM_BOT_TOKEN=<token>
TELEGRAM_BOT_POOL=<token1>,<token2>,...
BRAVE_API_KEY=<key>
DEFAULT_MODEL=claude-sonnet-4-6
EOF
```

OAuth-Token hinzufügen (aus den soeben angelegten Claude-Credentials):

```bash
python3 -c "
import json
with open('/home/nanoclaw/.claude/.credentials.json') as f:
    d = json.load(f)
token = d['claudeAiOauth']['accessToken']
print(f'CLAUDE_CODE_OAUTH_TOKEN={token}')
" >> ~/nanoclaw/.env
```

> **Warum?** Nanoclaw startet einen HTTP-Proxy der API-Calls der Agent-Container authentifiziert.
> Der Proxy liest `CLAUDE_CODE_OAUTH_TOKEN` aus `.env`. Ohne diesen Token antwortet der Agent
> mit „Not logged in · Please run /login".

### A8. systemd Service

```bash
mkdir -p ~/.config/systemd/user ~/nanoclaw/logs ~/nanoclaw/store

# node-Pfad ermitteln
which node  # z.B. /home/nanoclaw/.nvm/versions/node/v22.22.1/bin/node

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

### A9. Gruppen registrieren

Schreib dem Bot `/chatid` im jeweiligen Telegram-Chat, dann über das IPC-System registrieren
(oder via NanoClaw-Claude-Code-Session direkt in der DB eintragen).

---

## Teil B: Migration (bestehende Installation umziehen)

### Was wird übertragen — und was nicht

| Was | Übertragen? | Warum |
|-----|------------|-------|
| `.env` | ✅ Ja | API-Keys, Bot-Tokens |
| `store/messages.db` | ✅ Ja (mit Zielpfad!) | Registrierte Gruppen, Nachrichten-History |
| `groups/*/CLAUDE.md` | ✅ Ja | Per-Gruppe Konfiguration und Erinnerungen |
| `sessions/` | ❌ Nein | Session-IDs sind an die alte Instanz gebunden |
| `data/` | ❌ Nein | IPC-Dateien der alten Instanz, nicht portierbar |
| `logs/` | ❌ Nein | Alte Logs, nicht nötig |
| `telegram_main/`, `telegram_swarm/` | ❌ Nein | Alte Session-Daten |

### B1. Schritte A1–A7 durchführen

Infrastruktur, Node, Claude Code, Repo klonen, bauen — alles wie bei der Ersteinrichtung.
**Aber: `.env` noch nicht anlegen** — die kommt per rsync vom alten System.

### B2. Daten übertragen (vom alten System aus)

```bash
# Auf dem ALTEN System (Laptop/Server) ausführen:
cd ~/nanoclaw  # oder Pfad zur lokalen Nanoclaw-Installation

# .env übertragen
rsync -av .env nanoclaw@<VPS-IP>:~/nanoclaw/

# Datenbank — WICHTIG: Zielpfad store/ angeben, nicht Projekt-Root
rsync -av store/messages.db nanoclaw@<VPS-IP>:~/nanoclaw/store/

# Gruppen-Konfiguration (CLAUDE.md pro Gruppe, ohne Logs)
rsync -av --exclude='logs/' groups/ nanoclaw@<VPS-IP>:~/nanoclaw/groups/
```

> **Kritisch:** Die DB muss nach `~/nanoclaw/store/messages.db` — nicht nach `~/nanoclaw/messages.db`.
> Landet sie im Projekt-Root, legt der Service beim Start eine leere DB in `store/` an und findet
> `groupCount: 0` — alle eingehenden Telegram-Nachrichten werden dann still verworfen.

### B3. OAuth-Token nach Migration setzen

Der Token in `.env` stammt vom alten System und ist **abgelaufen oder ungültig** auf dem neuen.
Auf dem **VPS** neu setzen:

```bash
# Zuerst Claude Code einloggen (falls noch nicht in A5 gemacht):
claude  # Login-Flow durchführen

# Token aus den frischen Credentials extrahieren und in .env eintragen:
python3 -c "
import json
with open('/home/nanoclaw/.claude/.credentials.json') as f:
    d = json.load(f)
token = d['claudeAiOauth']['accessToken']

with open('/home/nanoclaw/nanoclaw/.env', 'r') as f:
    lines = [l for l in f.readlines() if not l.startswith('CLAUDE_CODE_OAUTH_TOKEN=')]
# Sicherstellen dass .env mit Newline endet
if lines and not lines[-1].endswith('\n'):
    lines[-1] += '\n'
lines.append(f'CLAUDE_CODE_OAUTH_TOKEN={token}\n')
with open('/home/nanoclaw/nanoclaw/.env', 'w') as f:
    f.writelines(lines)
print('Token geschrieben.')
"
```

### B4. Service starten und prüfen

```bash
systemctl --user daemon-reload
systemctl --user enable nanoclaw
systemctl --user start nanoclaw

# Prüfen: groupCount muss > 0 sein
grep "groupCount" ~/nanoclaw/logs/nanoclaw.log | tail -3
# Erwartet: groupCount: 2 (oder mehr)
```

Wenn `groupCount: 0`: DB ist im falschen Pfad (siehe B2).

### B5. Token-Ablauf

Der OAuth-Token läuft nach ~8 Stunden ab. Wenn der Agent später wieder mit „Not logged in"
antwortet, Token neu extrahieren (Schritt B3) und Service neustarten:

```bash
systemctl --user restart nanoclaw
```

**Langfristige Lösung:** Einen Anthropic API-Key in `.env` als `ANTHROPIC_API_KEY=sk-ant-api...`
eintragen — dann läuft der Credential-Proxy im API-Key-Modus ohne Token-Ablauf.

---

## Troubleshooting

### „groupCount: 0" beim Start
→ DB liegt falsch. Service stoppen, DB in `store/` legen, neu starten:
```bash
systemctl --user stop nanoclaw
cp ~/nanoclaw/messages.db ~/nanoclaw/store/messages.db  # falls im Root gelandet
systemctl --user start nanoclaw
```

### Agent antwortet „Not logged in · Please run /login"
→ `CLAUDE_CODE_OAUTH_TOKEN` fehlt oder ist abgelaufen. Schritt B3 wiederholen.

### „No conversation found with session ID: <uuid>"
→ Stale Session-ID aus alter Installation in der DB. Service stoppen, löschen, neu starten:
```bash
systemctl --user stop nanoclaw
node -e "
const DB = require('better-sqlite3');
const db = new DB('/home/nanoclaw/nanoclaw/store/messages.db');
db.prepare('DELETE FROM sessions').run();
db.close();
console.log('Sessions cleared');
"
systemctl --user start nanoclaw
```
Oder via MCP SQL-Tool in Claude Code: `DELETE FROM sessions;`

### Container startet nicht (Image not found)
```bash
cd ~/nanoclaw && ./container/build.sh
```

---

## Checkliste Migration

- [ ] Infrastruktur (A1–A4): User, Docker, Tailscale, linger
- [ ] Node.js 22 via nvm (A5)
- [ ] Claude Code installiert und eingeloggt (`claude`)
- [ ] Repo geklont, `npm install && npm run build`
- [ ] Container gebaut (`./container/build.sh`)
- [ ] `.env` per rsync übertragen
- [ ] `store/messages.db` per rsync nach `store/` übertragen (nicht Root!)
- [ ] `groups/` per rsync übertragen (ohne `logs/`)
- [ ] OAuth-Token in `.env` neu gesetzt (Schritt B3)
- [ ] systemd Service eingerichtet und gestartet
- [ ] `groupCount` im Log > 0 bestätigt
- [ ] Testnachricht auf Telegram funktioniert
