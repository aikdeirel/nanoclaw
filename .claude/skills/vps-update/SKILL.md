---
name: vps-update
description: Update a running NanoClaw VPS installation. Use when the user wants to sync data from their local machine to the VPS, pull new code, rebuild, or restart the service on a remote server.
---

# NanoClaw VPS Update

Aktualisiert eine laufende NanoClaw-Installation auf einem VPS. Läuft **lokal** — Remote-Befehle per SSH, Daten-Sync per rsync. Claude Code muss nicht auf dem VPS laufen.

Für Erstinstallation auf frischem VPS: `/vps-setup`.

## Schritt 0: Eckdaten abfragen

Frage mit `AskUserQuestion` ab:
- **VPS_IP** — öffentliche IPv4 des Hetzner-Servers
- **LOCAL_NANOCLAW** — lokaler Pfad zur NanoClaw-Installation (z.B. `~/nanoclaw`)

SSH-Zugriff sofort prüfen:
```bash
ssh -o ConnectTimeout=5 nanoclaw@VPS_IP "echo 'SSH OK'"
```
Falls Fehler → abbrechen und mit Nutzer klären.

## Schritt 1: Situation analysieren

Selbstständig prüfen und Ergebnis präsentieren — **vor** der Entscheidung was gemacht werden soll. Alle Checks parallel ausführen.

```bash
# Lokaler Code-Stand
cd LOCAL_NANOCLAW && git log --oneline -5

# VPS Code-Stand
ssh nanoclaw@VPS_IP "cd ~/nanoclaw && git log --oneline -5"

# Neue Commits auf origin/main (nur lokal prüfen, kein fetch auf VPS nötig)
cd LOCAL_NANOCLAW && git fetch origin && git log HEAD..origin/main --oneline

# Lokale DB
ls -lh LOCAL_NANOCLAW/store/messages.db

# VPS DB
ssh nanoclaw@VPS_IP "ls -lh ~/nanoclaw/store/messages.db 2>/dev/null || echo 'Keine DB auf VPS'"

# Lokale Gruppen
ls LOCAL_NANOCLAW/groups/ 2>/dev/null

# VPS Gruppen
ssh nanoclaw@VPS_IP "ls ~/nanoclaw/groups/ 2>/dev/null || echo 'Keine groups/ auf VPS'"

# Service-Status VPS
ssh nanoclaw@VPS_IP "systemctl --user is-active nanoclaw"

# Node/npm-Version lokal vs. VPS
node --version && npm --version
ssh nanoclaw@VPS_IP "
  export NVM_DIR=\$HOME/.nvm && source \$NVM_DIR/nvm.sh
  node --version && npm --version
"
```

Ergebnis als Tabelle zusammenfassen:

| | Lokal | VPS |
|---|---|---|
| Letzter Commit | … | … |
| Commits Unterschied | … hinter VPS / … voraus | — |
| DB-Größe + Zeitstempel | … | … |
| Gruppen | … | … |
| Service-Status | — | active / inactive |
| Node-Version | … | … |

**Warnungen aktiv einblenden wenn:**
- VPS-DB ist neuer als lokale DB → „⚠️ VPS-DB ist neuer — Daten-Sync würde Daten überschreiben!"
- Node-Versionen unterscheiden sich → „⚠️ Node-Version lokal/VPS verschieden — Build-Probleme möglich"

Dann fragen:

> Was soll gemacht werden?
> 1. **Code-Update** — Git pull, bauen, Service neu starten
> 2. **Daten-Sync** — .env, DB und groups/ lokal → VPS
> 3. **Beides** — Service stop → Code → Daten → Start (empfohlen)
> 4. **Nur Service neu starten**

## Schritt 2: Service stoppen

Bei allen Szenarien außer reinem Analyse-Run:

```bash
ssh nanoclaw@VPS_IP "systemctl --user stop nanoclaw && systemctl --user is-active nanoclaw || echo 'Service gestoppt'"
```

## Schritt 3: Code-Update (Szenarien 1 und 3)

```bash
ssh nanoclaw@VPS_IP "
  export NVM_DIR=\$HOME/.nvm && source \$NVM_DIR/nvm.sh
  cd ~/nanoclaw
  git pull origin main
  npm install
  npm run build
  echo Build OK
"
```

> **Hinweis:** `bash -l` lädt nvm nicht zuverlässig. Immer explizit `source $NVM_DIR/nvm.sh` verwenden.

Container nur neu bauen wenn sich Container-Dateien geändert haben:
```bash
# ORIG_HEAD zeigt auf den Stand vor dem Pull — korrekt auch bei mehreren gepullten Commits:
ssh nanoclaw@VPS_IP "cd ~/nanoclaw && git diff ORIG_HEAD HEAD --name-only | grep '^container/'"

# Falls ja — sauberer Rebuild (--no-cache allein reicht nicht wegen buildkit-Cache):
ssh nanoclaw@VPS_IP "docker builder prune -f && cd ~/nanoclaw && ./container/build.sh"
```

## Schritt 4: Daten-Sync (Szenarien 2 und 3)

### Was wird übertragen

| Was | Übertragen? | Grund |
|-----|------------|-------|
| `.env` | ✅ Ja | API-Keys, Bot-Tokens |
| `store/messages.db` | ✅ Ja (Zielpfad: `store/`!) | Gruppen, Nachrichten-History |
| `groups/*/CLAUDE.md` | ✅ Ja | Per-Gruppe Konfiguration |
| `sessions/` | ❌ Nein | Session-IDs an alte Instanz gebunden |
| `data/` | ❌ Nein | IPC-Dateien, nicht portierbar |
| `logs/` | ❌ Nein | Nicht nötig |
| `telegram_main/`, `telegram_swarm/` | ❌ Nein | Alte Session-Daten |

```bash
cd LOCAL_NANOCLAW

rsync -av .env nanoclaw@VPS_IP:~/nanoclaw/

# WICHTIG: Zielpfad store/ — nicht Projekt-Root!
rsync -av store/messages.db nanoclaw@VPS_IP:~/nanoclaw/store/

rsync -av --exclude='logs/' groups/ nanoclaw@VPS_IP:~/nanoclaw/groups/
```

> **Kritisch:** DB muss nach `~/nanoclaw/store/messages.db`. Im Projekt-Root landet sie?
> Service startet mit leerer DB → `groupCount: 0` → alle Telegram-Nachrichten werden verworfen.

### OAuth-Token neu setzen

Der übertragene Token aus `.env` ist auf dem VPS **ungültig**. Frischen Token extrahieren:

```bash
ssh nanoclaw@VPS_IP "python3 -c \"
import json
with open('/home/nanoclaw/.claude/.credentials.json') as f:
    d = json.load(f)
token = d['claudeAiOauth']['accessToken']
with open('/home/nanoclaw/nanoclaw/.env', 'r') as f:
    lines = [l for l in f.readlines() if not l.startswith('CLAUDE_CODE_OAUTH_TOKEN=')]
if lines and not lines[-1].endswith('\n'):
    lines[-1] += '\n'
lines.append(f'CLAUDE_CODE_OAUTH_TOKEN={token}\n')
with open('/home/nanoclaw/nanoclaw/.env', 'w') as f:
    f.writelines(lines)
print('Token gesetzt.')
\""
```

Falls `.credentials.json` auf dem VPS fehlt → erst `ssh nanoclaw@VPS_IP "claude"` und Login-Flow.

> **Dauerlösung:** `ANTHROPIC_API_KEY=sk-ant-api...` in `.env` → kein Token-Ablauf mehr.

## Schritt 5: Service starten und verifizieren

```bash
ssh nanoclaw@VPS_IP "systemctl --user start nanoclaw"

# 5 Sekunden warten, dann Log prüfen:
sleep 5
ssh nanoclaw@VPS_IP "tail -30 ~/nanoclaw/logs/nanoclaw.log"
ssh nanoclaw@VPS_IP "systemctl --user status nanoclaw --no-pager"
```

Gesunder Start: `groupCount: N` mit N > 0, kein „Not logged in".

## Troubleshooting

**groupCount: 0** → DB liegt falsch (Projekt-Root statt `store/`):
```bash
ssh nanoclaw@VPS_IP "
  systemctl --user stop nanoclaw
  mv ~/nanoclaw/messages.db ~/nanoclaw/store/messages.db 2>/dev/null || true
  systemctl --user start nanoclaw
"
```

**„Not logged in · Please run /login"** → Token abgelaufen (läuft nach ~8h ab). Schritt 4 (Token-Extraktion) wiederholen + `systemctl --user restart nanoclaw`.

**„No conversation found with session ID: <uuid>"** → Stale Sessions aus alter Instanz:
```bash
ssh nanoclaw@VPS_IP "
  systemctl --user stop nanoclaw
  node -e \"const DB=require('better-sqlite3');const db=new DB('/home/nanoclaw/nanoclaw/store/messages.db');db.prepare('DELETE FROM sessions').run();db.close();console.log('Done')\"
  systemctl --user start nanoclaw
"
```

**Container startet nicht** → `ssh nanoclaw@VPS_IP "cd ~/nanoclaw && ./container/build.sh"`

**Build schlägt fehl** → `ssh nanoclaw@VPS_IP "cd ~/nanoclaw && rm -rf node_modules && npm install && npm run build"`

## Checkliste

- [ ] SSH-Zugriff geprüft
- [ ] Situation analysiert (Code-Stand, DB-Stand)
- [ ] Service gestoppt
- [ ] (Code) git pull + npm install + build erfolgreich
- [ ] (Code) Container neu gebaut falls container/ geändert
- [ ] (Daten) .env übertragen
- [ ] (Daten) store/messages.db nach `store/` übertragen
- [ ] (Daten) groups/ übertragen (ohne logs/)
- [ ] (Daten) OAuth-Token neu gesetzt
- [ ] Service gestartet, groupCount > 0, kein Auth-Fehler
- [ ] Testnachricht auf Telegram funktioniert
