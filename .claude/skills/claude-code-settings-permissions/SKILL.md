---
name: claude-code-settings-permissions
description: Claude-Code-Settings (settings.json-Hierarchie, Vorrang, Merge von Listen) und Permissions (allow/ask/deny-Regeln, Permission-Modi, Bash-Sandbox) konfigurieren und debuggen. Verwenden, wenn ein Agent Rechte freigeben oder sperren, Team-Settings teilen oder klären soll, warum eine Einstellung nicht greift.
when_to_use: 'Auslöser: Befehl ohne Rückfrage erlauben; .env sperren; settings.json; Permission-Regel greift nicht; Sandbox aktivieren'
---

# Claude Code Settings & Permissions

## Zweck und Einsatzgebiet

Settings steuern das Verhalten von Claude Code: Modell, Rechte, Hooks, Plugins, Umgebungsvariablen und Sandbox.
Permissions legen fest, was Claude **ohne Rückfrage** darf, was **nachfragt** und was **verboten** ist.

## Installation

Es gibt nichts zu installieren. Die Dateien und ihr Geltungsbereich:

| Scope | Datei | Wirkung |
| --- | --- | --- |
| Benutzer | `~/.claude/settings.json` | Für dich, in allen Projekten |
| Projekt (geteilt) | `.claude/settings.json` | Für alle im Repo, wenn committet |
| Projekt (lokal) | `.claude/settings.local.json` | Nur für dich, nur in diesem Projekt (nicht committen) |
| Managed | `managed-settings.json`, MDM, server-managed | Organisation, nicht überschreibbar |

**Vorrang**, höchster zuerst: Managed → Kommandozeile (`--settings`) → Projekt lokal → Projekt geteilt → Benutzer.
Listen wie `permissions.allow` werden über die Ebenen **zusammengeführt**, nicht ersetzt.

## Offizielle Dokumentation

- Settings und Vorrang: https://code.claude.com/docs/en/settings
- Alle Schlüssel: https://code.claude.com/docs/en/settings-reference
- Beispieldateien: https://code.claude.com/docs/en/settings-example
- Permissions: https://code.claude.com/docs/en/permissions
- Permission-Modi: https://code.claude.com/docs/en/permission-modes
- Sandboxing: https://code.claude.com/docs/en/sandboxing
- Umgebungsvariablen: https://code.claude.com/docs/en/env-vars
- Konfiguration debuggen: https://code.claude.com/docs/en/debug-your-config
- JSON-Schema: https://json.schemastore.org/claude-code-settings.json
- Beispiele im Repo: https://github.com/anthropics/claude-code/tree/main/examples/settings

## Codebeispiele

### Team-Datei `.claude/settings.json`

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "allow": [
      "Bash(npm run lint)",
      "Bash(npm run test *)",
      "WebFetch(domain:docs.python.org)"
    ],
    "ask": [
      "Bash(git push *)"
    ],
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Bash(curl *)"
    ]
  }
}
```

### Regel-Syntax (Auszug)

| Regel | Trifft |
| --- | --- |
| `Bash` oder `Bash(*)` | Alle Bash-Befehle. Als deny wird das Tool ganz aus dem Kontext entfernt |
| `Bash(npm run build)` | Genau diesen Befehl |
| `Bash(npm run *)` | `npm run build`, `npm run test --watch`, `npm run` |
| `Bash(git log *)` | Nur `git log …`. `Bash(git *)` erlaubt dagegen jeden git-Befehl |
| `Read(./.env)` | Lesen von `.env` im aktuellen Verzeichnis |
| `WebFetch(domain:example.com)` | Fetches auf example.com |
| `mcp__puppeteer__*` | Alle Tools des MCP-Servers `puppeteer` (bei allow nur mit festem Server-Präfix) |
| `Agent(model:opus)` | Subagent-Aufrufe, die das Opus-Modell anfordern |

**Auswertung:** erst deny, dann ask, dann allow. Der erste Treffer gewinnt.
Ein allow kann kein Loch in ein deny schneiden.

### Permission-Modi

`default` (in der UI „Manual“), `acceptEdits`, `plan`, `auto`, `dontAsk`, `bypassPermissions`.
Den Standard per `permissions.defaultMode` setzen.
`auto` und `bypassPermissions` greifen **nicht** aus Projekt- oder lokalen Settings, sondern nur aus Benutzer- oder Managed Settings bzw. per `--permission-mode`.

### Bash-Sandbox (nach `examples/settings/settings-bash-sandbox.json`)

```json
{
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": false,
    "allowUnsandboxedCommands": false,
    "network": { "allowedDomains": [] }
  }
}
```

Die Sandbox gilt **nur für das Bash-Tool**, nicht für Read, Write, WebFetch, MCP oder Hooks.

### Nur für eine Session

```bash
claude --settings '{"model": "claude-opus-5-5"}'
claude --settings ./ci-settings.json
```

## Regeln für die Verwendung

1. **Team-Regeln** gehören in `.claude/settings.json`, **persönliche Ausnahmen** in `.claude/settings.local.json`.
2. **Gültiges JSON:** keine Kommentare, keine trailing commas. Den `$schema`-Eintrag für die Editor-Validierung setzen.
3. **Den `*` hinter den Unterbefehl setzen**: `Bash(git log *)` statt `Bash(git *)`.
4. **Netzwerkzugriff nicht über Bash-Muster einschränken.** Muster wie `Bash(curl http://github.com/ *)` sind fragil. Besser `curl`/`wget` verbieten, `WebFetch(domain:…)` erlauben und die Sandbox-Netzwerk-Allowlist nutzen.
5. **Wrapper wie `npx`, `docker exec`, `devbox run` nicht pauschal erlauben.** Sie führen beliebige Befehle aus, also nur konkrete Befehle freigeben (z. B. `Bash(devbox run npm test)`).
6. **Nach jeder Änderung prüfen:** `/status` zeigt die geladenen Quellen, `claude doctor` die verworfenen Einträge.

## Fehlerbehebung

| Symptom | Ursache / Lösung |
| --- | --- |
| Wert wird ignoriert | Eine höhere Ebene setzt denselben Schlüssel oder eine Env-Var überschreibt ihn (z. B. `ANTHROPIC_MODEL` vor `model`) |
| Team-Regel wirkt beim Kollegen nicht | Workspace-Trust fehlt: `allow`-Regeln, `additionalDirectories` und die meisten `env`-Werte greifen erst nach dem Trust (`deny`/`ask` sofort) |
| Trotz „Yes, don't ask again“ kommt die Rückfrage | Eine `ask`-Regel aus Projekt- oder Managed Settings sticht die lokale `allow`-Regel |
| Startwarnung „wildcard before the rest of the command“ | allow-Regel mit `*` vor dem Unterbefehl, umschreiben |
| `mcp__…(…)`-Regel übersprungen | MCP-Regeln mit Klammern sind in Settings ungültig; dafür `--disallowedTools` verwenden |
| Settings Error | JSON reparieren, Details mit `claude doctor` |

## Sicherheitsgrenzen

- **`bypassPermissions`** nur in isolierten Containern oder VMs. In Organisationen per `permissions.disableBypassPermissionsMode: "disable"` sperren.
- **Deny-Regeln für Bash greifen nicht** bei Aufrufen über absolute Pfade (`/bin/rm`) oder `sh -c`. Wo die Grenze halten muss, die Sandbox einsetzen.
- **Secrets schützen:** `.env`, Schlüsseldateien und Cloud-Credentials per `Read(...)`-deny sperren.
- Ein Agent darf **keine Rechte eigenmächtig erweitern** (z. B. `allow`-Regeln in geteilte Settings schreiben), ohne dass der Mensch zustimmt.
- Managed Settings sind die verbindliche Policy und dürfen nicht umgangen werden.
