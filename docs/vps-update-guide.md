# NanoClaw VPS Update Guide

> Nicht committen — enthält persönliche IP-Adressen und Pfade.

Dieses Dokument beschreibt, wie eine **bereits laufende NanoClaw-VPS-Installation** aktualisiert wird.
Es deckt drei Szenarien ab: Code-Update, Daten-Sync vom lokalen System, oder beides kombiniert.

**Ausführungskontext:** Dieser Guide wird lokal ausgeführt. Der Agent nutzt SSH, um den VPS
fernzusteuern, und rsync, um Daten zu übertragen. Claude Code muss nicht auf dem VPS laufen.

Für eine Erstinstallation auf einem frischen VPS: siehe `vps-setup-guide.md`.

---

## Schritt 0: Eckdaten abfragen

Hole folgende Informationen vom Nutzer ein:

| Variable | Bedeutung | Beispiel |
|----------|-----------|---------|
| `VPS_IP` | Öffentliche IPv4 des Hetzner-Servers | `1.2.3.4` |
| `LOCAL_NANOCLAW` | Lokaler Pfad zur NanoClaw-Installation | `~/nanoclaw` |

Prüfe dann, ob SSH-Zugriff funktioniert:

```bash
ssh -o ConnectTimeout=5 nanoclaw@VPS_IP "echo 'SSH OK'"
```

Falls das fehlschlägt: SSH-Key prüfen oder VPS-IP falsch → abbrechen und mit Nutzer klären.

---

## Schritt 1: Situation analysieren

Führe folgende Checks durch und präsentiere dem Nutzer das Ergebnis, bevor du weitermachst:

### Code-Stand prüfen (lokal vs. VPS)

```bash
# Lokaler HEAD
cd LOCAL_NANOCLAW && git log --oneline -5

# VPS HEAD
ssh nanoclaw@VPS_IP "cd ~/nanoclaw && git log --oneline -5"

# Commits auf origin/main seit dem lokalen Stand
cd LOCAL_NANOCLAW && git fetch origin && git log HEAD..origin/main --oneline
```

### Daten-Stand prüfen

```bash
# Lokale DB-Größe und letzter Schreibzeitpunkt
ls -lh LOCAL_NANOCLAW/store/messages.db

# VPS DB-Größe
ssh nanoclaw@VPS_IP "ls -lh ~/nanoclaw/store/messages.db 2>/dev/null || echo 'Keine DB auf VPS'"

# Lokale .env (nur prüfen ob vorhanden, nicht anzeigen)
ls -l LOCAL_NANOCLAW/.env

# Lokale groups/ — Anzahl Gruppen
ls LOCAL_NANOCLAW/groups/ 2>/dev/null | wc -l
```

### Ergebnis präsentieren und fragen

Zeige dem Nutzer eine Zusammenfassung:
- Gibt es neue Commits auf origin/main?
- Ist die lokale DB neuer als die auf dem VPS?
- Welche groups/ existieren lokal?

Frage dann:

> Was soll gemacht werden?
> 1. **Code-Update** — neuen Code aus Git pullen, bauen, Service neu starten
> 2. **Daten-Sync** — .env, Datenbank und groups/ vom lokalen System auf VPS übertragen
> 3. **Beides** — Code-Update + Daten-Sync (empfohlene Reihenfolge: Service stop → Code → Daten → Start)
> 4. **Nur Service neu starten** — kein Pull, kein Sync

---

## Schritt 2: Service auf dem VPS stoppen

Bei Szenarien 1, 2, 3 und 4: Service erst stoppen, bevor Änderungen vorgenommen werden.

```bash
ssh nanoclaw@VPS_IP "systemctl --user stop nanoclaw && echo 'Service gestoppt'"
```

Bestätigen, dass der Service nicht mehr läuft:

```bash
ssh nanoclaw@VPS_IP "systemctl --user is-active nanoclaw || echo 'Service ist inaktiv'"
```

---

## Schritt 3 (Code-Update): Code aktualisieren und bauen

Nur bei Szenarien 1 und 3.

```bash
# Auf dem VPS via SSH:
ssh nanoclaw@VPS_IP "
  cd ~/nanoclaw
  git pull origin main
  npm install
  npm run build
  echo 'Build abgeschlossen'
"
```

> **Hinweis Container-Build-Cache:** Das buildkit-Cache ist aggressiv. `--no-cache` allein
> invalidiert COPY-Schritte nicht — der Builder-Volume behält alte Dateien. Für einen sauberen
> Rebuild erst den Builder prunen:

```bash
# Nur nötig wenn Container-Änderungen deployed werden:
ssh nanoclaw@VPS_IP "
  cd ~/nanoclaw
  docker builder prune -f
  ./container/build.sh
  echo 'Container gebaut'
"
```

Falls kein Container-Rebuild nötig (nur JS-Änderungen im Host-Prozess), den letzten Block weglassen.

---

## Schritt 4 (Daten-Sync): Daten vom lokalen System auf VPS übertragen

Nur bei Szenarien 2 und 3.

### Was wird übertragen — und was nicht

| Was | Übertragen? | Warum |
|-----|------------|-------|
| `.env` | ✅ Ja | API-Keys, Bot-Tokens |
| `store/messages.db` | ✅ Ja (mit Zielpfad!) | Registrierte Gruppen, Nachrichten-History |
| `groups/*/CLAUDE.md` | ✅ Ja | Per-Gruppe Konfiguration und Erinnerungen |
| `sessions/` | ❌ Nein | Session-IDs sind an die alte Instanz gebunden |
| `data/` | ❌ Nein | IPC-Dateien, nicht portierbar |
| `logs/` | ❌ Nein | Alte Logs, nicht nötig |
| `telegram_main/`, `telegram_swarm/` | ❌ Nein | Alte Session-Daten |

### Übertragung durchführen (lokal ausführen)

```bash
cd LOCAL_NANOCLAW

# .env übertragen
rsync -av .env nanoclaw@VPS_IP:~/nanoclaw/

# Datenbank — WICHTIG: Zielpfad store/ angeben, nicht Projekt-Root
rsync -av store/messages.db nanoclaw@VPS_IP:~/nanoclaw/store/

# Gruppen-Konfiguration (CLAUDE.md pro Gruppe, ohne Logs)
rsync -av --exclude='logs/' groups/ nanoclaw@VPS_IP:~/nanoclaw/groups/
```

> **Kritisch:** Die DB muss nach `~/nanoclaw/store/messages.db` — nicht nach `~/nanoclaw/messages.db`.
> Landet sie im Projekt-Root, legt der Service beim Start eine leere DB in `store/` an und findet
> `groupCount: 0` — alle eingehenden Telegram-Nachrichten werden dann still verworfen.

### OAuth-Token neu setzen

Der Token in der übertragenen `.env` stammt vom lokalen System und ist auf dem VPS **ungültig**.
Auf dem VPS einen frischen Token aus den dortigen Claude-Credentials extrahieren:

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
print('Token geschrieben.')
\""
```

> **Falls noch kein Login auf dem VPS erfolgt ist** (`.credentials.json` fehlt):
> ```bash
> ssh nanoclaw@VPS_IP "claude"
> ```
> Login-Flow durchführen, dann Token-Extraktion von oben wiederholen.

> **Langfristige Lösung:** `ANTHROPIC_API_KEY=sk-ant-api...` in `.env` eintragen —
> dann kein Token-Ablauf mehr nötig.

---

## Schritt 5: Service starten und verifizieren

```bash
ssh nanoclaw@VPS_IP "systemctl --user start nanoclaw && echo 'Service gestartet'"
```

Nach ca. 5 Sekunden Logs prüfen:

```bash
ssh nanoclaw@VPS_IP "tail -30 ~/nanoclaw/logs/nanoclaw.log"
```

**Erwartete Zeichen für gesunden Start:**
- `groupCount: N` mit N > 0 (falls DB übertragen wurde)
- Kein „Not logged in" oder „Auth error"
- Service-Status aktiv:

```bash
ssh nanoclaw@VPS_IP "systemctl --user status nanoclaw --no-pager"
```

---

## Troubleshooting

### „groupCount: 0" beim Start

DB liegt falsch. Service stoppen, DB in `store/` legen, neu starten:

```bash
ssh nanoclaw@VPS_IP "
  systemctl --user stop nanoclaw
  mv ~/nanoclaw/messages.db ~/nanoclaw/store/messages.db 2>/dev/null || true
  systemctl --user start nanoclaw
"
```

### Agent antwortet „Not logged in · Please run /login"

`CLAUDE_CODE_OAUTH_TOKEN` fehlt oder ist abgelaufen. Token neu extrahieren (Schritt 4,
Abschnitt „OAuth-Token neu setzen") und Service neu starten:

```bash
ssh nanoclaw@VPS_IP "systemctl --user restart nanoclaw"
```

Der Token läuft nach ca. 8 Stunden ab. Dauerlösung: `ANTHROPIC_API_KEY` in `.env` setzen.

### „No conversation found with session ID: <uuid>"

Stale Session-IDs aus alter Installation in der DB. Sessions löschen:

```bash
ssh nanoclaw@VPS_IP "
  systemctl --user stop nanoclaw
  node -e \"
const DB = require('better-sqlite3');
const db = new DB('/home/nanoclaw/nanoclaw/store/messages.db');
db.prepare('DELETE FROM sessions').run();
db.close();
console.log('Sessions cleared');
\"
  systemctl --user start nanoclaw
"
```

Alternativ via MCP SQL-Tool in Claude Code: `DELETE FROM sessions;`

### Container startet nicht (Image not found)

```bash
ssh nanoclaw@VPS_IP "cd ~/nanoclaw && ./container/build.sh"
```

Bei hartnäckigen Cache-Problemen (COPY-Schritte werden nicht neu ausgeführt):

```bash
ssh nanoclaw@VPS_IP "
  docker builder prune -f
  cd ~/nanoclaw && ./container/build.sh
"
```

### Build schlägt fehl (npm-Fehler)

```bash
ssh nanoclaw@VPS_IP "cd ~/nanoclaw && rm -rf node_modules && npm install && npm run build"
```

---

## Checkliste Update

- [ ] SSH-Zugriff auf VPS funktioniert
- [ ] Situation analysiert (Code-Stand, DB-Stand)
- [ ] Service auf VPS gestoppt
- [ ] (bei Code-Update) `git pull && npm install && npm run build` erfolgreich
- [ ] (bei Code-Update mit Container-Änderungen) Container neu gebaut
- [ ] (bei Daten-Sync) `.env` per rsync übertragen
- [ ] (bei Daten-Sync) `store/messages.db` per rsync nach `store/` übertragen (nicht Root!)
- [ ] (bei Daten-Sync) `groups/` per rsync übertragen (ohne `logs/`)
- [ ] (bei Daten-Sync) OAuth-Token in `.env` neu gesetzt
- [ ] Service gestartet
- [ ] `groupCount` im Log > 0 (falls Gruppen vorhanden)
- [ ] Kein Auth-Fehler im Log
- [ ] Testnachricht auf Telegram funktioniert
