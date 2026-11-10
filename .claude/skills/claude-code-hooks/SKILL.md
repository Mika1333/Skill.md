---
name: claude-code-hooks
description: Konfigurieren und Debuggen von Claude-Code-Hooks (PreToolUse, PostToolUse, UserPromptSubmit, SessionStart, Stop u.a.) in settings.json oder Plugins, inklusive Matcher, Exit-Codes und JSON-Ausgabe. Verwenden, wenn ein Agent Abläufe automatisieren, Tool-Aufrufe validieren oder blockieren oder Hook-Fehler analysieren soll.
when_to_use: 'Auslöser: Hook einrichten; nach jedem Edit formatieren; Befehl vor Ausführung blockieren; Hook feuert nicht'
---

# Claude Code Hooks

## Zweck und Einsatzgebiet

Hooks sind Handler, die Claude Code an festen Punkten im Lebenszyklus ausführt, zum Beispiel vor oder nach einem Tool-Aufruf, beim Absenden eines Prompts oder beim Start einer Session.
Sie laufen **deterministisch**, unabhängig davon, ob das Modell „daran denkt“.

Typische Einsätze:
- Formatter oder Linter nach `Edit`/`Write`
- Shell-Befehle vor der Ausführung prüfen und blockieren
- Kontext beim Session-Start nachladen
- Benachrichtigungen versenden

## Installation

Hooks werden in Settings-Dateien unter dem Schlüssel `hooks` konfiguriert. Es gibt nichts zu installieren:

- `~/.claude/settings.json` (Benutzer), `.claude/settings.json` (Projekt, committet), `.claude/settings.local.json` (lokal)
- Managed Settings (Organisation)
- In Plugins: `hooks/hooks.json`
- In Skill- und Subagent-Frontmatter (`hooks`)

Skripte z. B. unter `.claude/hooks/` ablegen, ausführbar machen (`chmod +x`) und per `${CLAUDE_PROJECT_DIR}` referenzieren.

## Offizielle Dokumentation

- Hooks-Referenz: https://code.claude.com/docs/en/hooks
- Hooks-Leitfaden mit Beispielen: https://code.claude.com/docs/en/hooks-guide
- Hooks debuggen: https://code.claude.com/docs/en/hooks#debug-hooks
- Hooks im Agent SDK: https://code.claude.com/docs/en/agent-sdk/hooks
- Beispiel-Hook im Repo: https://github.com/anthropics/claude-code/blob/main/examples/hooks/bash_command_validator_example.py

## Codebeispiele

### Struktur: Event → Matcher-Gruppe → Handler

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 \"$CLAUDE_PROJECT_DIR\"/.claude/hooks/validate_bash.py",
            "timeout": 30
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write" }
        ]
      }
    ]
  }
}
```

### Validator (nach dem offiziellen Beispiel im Repo)

```python
#!/usr/bin/env python3
import json, re, sys

RULES = [(r"^grep\b(?!.*\|)", "Use 'rg' (ripgrep) instead of 'grep'")]

data = json.load(sys.stdin)                      # Event-JSON kommt über stdin
if data.get("tool_name") != "Bash":
    sys.exit(0)
cmd = data.get("tool_input", {}).get("command", "")
issues = [msg for pat, msg in RULES if re.search(pat, cmd)]
if issues:
    print("\n".join(issues), file=sys.stderr)
    sys.exit(2)                                  # Exit 2 blockiert; stderr geht an Claude
sys.exit(0)
```

### Entscheidung per JSON (PreToolUse)

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Destructive command blocked by hook"
  }
}
```

`permissionDecision` erlaubt `allow`, `deny`, `ask` oder `defer`.

### Handler gezielt filtern mit `if` (Permission-Syntax)

```json
{ "type": "command", "if": "Bash(rm *)", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/block-rm.sh" }
```

### Referenz

- **Events (Auswahl):** `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PermissionRequest`, `PostToolUse`, `PostToolUseFailure`, `Notification`, `SubagentStart`, `SubagentStop`, `Stop`, `PreCompact`, `SessionEnd`, `ConfigChange`, `FileChanged`. Die vollständige Liste steht in der Referenz.
- **Handler-Typen:** `command`, `http`, `mcp_tool`, `prompt`, `agent` (experimentell).
- **Matcher:**
  - `"*"`, `""` oder kein Matcher: alle
  - Nur Buchstaben, Ziffern, `_`, `-`, Leerzeichen, `,` und `|`: exakter Name bzw. Liste, z. B. `Edit|Write`
  - Alles andere: JavaScript-Regex, z. B. `mcp__memory__.*`
- **Exit-Codes:**
  - `0`: Erfolg; JSON auf stdout wird ausgewertet
  - `2`: blockierender Fehler; stderr ist die Begründung
  - Andere Codes, **auch `1`**: nicht blockierend, die Aktion läuft weiter
- **Variablen:** `CLAUDE_PROJECT_DIR`, in Plugins `CLAUDE_PLUGIN_ROOT` und `CLAUDE_PLUGIN_DATA`.

## Regeln für die Verwendung

1. **Zum Durchsetzen `exit 2` verwenden, nicht `exit 1`.** Code 1 ist laut Doku nicht blockierend.
2. **Harte Erlaube- und Verbotsregeln gehören ins Permission-System** (`permissions.deny`). Der `if`-Filter ist laut Doku nur „best-effort“.
3. **Timeouts bei `PreToolUse` lassen den Aufruf durch** (fail open). Für echte Gates `"onFailure": "block"` setzen (ab v2.1.295).
4. **stdout sauber halten:** genau ein JSON-Objekt oder reiner Text. Gemischte Ausgabe kann das Ergebnis still verändern.
5. **Pfade mit `${CLAUDE_PROJECT_DIR}` angeben.** Bei Platzhaltern mit Sonderzeichen die Exec-Form (`args`) nutzen, sie läuft ohne Shell.
6. **Hooks schnell halten.** Lange Arbeit mit `async` erledigen; Ausgaben über 10.000 Zeichen werden gekürzt.
7. **Nicht doppelt konfigurieren.** Ein Hook in den Settings und zusätzlich in einem Plugin läuft zweimal.

## Fehlerbehebung

| Symptom | Lösung |
| --- | --- |
| Hook feuert nicht | Event-Name exakt prüfen (Groß-/Kleinschreibung); Matcher `mcp__memory` ohne `.*` trifft kein Tool; `/status` und `claude doctor` auf verworfene Einträge prüfen |
| Hook läuft, blockiert aber nicht | Exit-Code 1 statt 2 verwendet oder Timeout (fail open) |
| „command not found“ / Skript fehlt | Absoluten Pfad über `${CLAUDE_PROJECT_DIR}` nutzen, `chmod +x`; ein fehlendes Skript erzeugt nur einen Hinweis im Transkript |
| Hooks im Projekt wirken nicht | Workspace-Trust noch nicht bestätigt, oder die Organisation setzt `allowManagedHooksOnly` |
| Unklare Ursache | `claude --debug` starten; das Debug-Log zeigt Treffer, Exit-Codes und Ausgaben |

## Sicherheitsgrenzen

- **Hooks laufen mit den Rechten des Nutzers** und führen beliebige Befehle aus. Fremde Hooks aus Repos oder Plugins vor der Aktivierung lesen.
- Eingaben aus dem Event-JSON (Dateipfade, Befehle) **nie ungeprüft in Shell-Strings einsetzen** (Injection). Werte quoten oder `args` nutzen.
- **HTTP-Hooks:** Ziel-URLs per `allowedHttpHookUrls` begrenzen; nur die in `allowedEnvVars` gelisteten Variablen werden in Header eingesetzt.
- **Keine Secrets** in Hook-Ausgaben schreiben: `additionalContext`, `systemMessage` und stdout landen im Kontext von Claude.
- Ein Hook ist **keine Sicherheitsgrenze gegen einen böswilligen Nutzer**. Für Organisationen Managed Settings mit `allowManagedHooksOnly` verwenden.
