---
name: claude-agent-sdk
description: Eigene KI-Agenten mit dem Claude Agent SDK (TypeScript `@anthropic-ai/claude-agent-sdk`, Python `claude-agent-sdk`) bauen – query(), Optionen für Tools/Permissions/MCP/Subagents/Hooks, Authentifizierung und Hosting. Verwenden, wenn Claude Codes Agent-Loop programmatisch in eine Anwendung, einen Service oder eine Pipeline eingebettet werden soll.
when_to_use: 'Auslöser: Agent SDK; claude_agent_sdk; query() mit Optionen; eigenen Agenten bauen; Claude Code als Bibliothek'
---

# Claude Agent SDK

## Zweck und Einsatzgebiet

Das Agent SDK stellt dieselben Tools, denselben Agent-Loop und dasselbe Kontextmanagement wie Claude Code als **Bibliothek für Python und TypeScript** bereit.
Dazu gehören eingebaute Tools, Hooks, Subagents, MCP, Permissions, Sessions sowie Skills und Plugins aus `.claude/`.

Abgrenzung:
- **Claude Code CLI:** interaktive Arbeit im Terminal.
- **Client SDK / Messages API:** direkte API-Aufrufe, den Tool-Loop schreibst du selbst.
- **Managed Agents:** Anthropic hostet den Agenten.
- **Andere Sprachen:** die CLI mit `claude -p --output-format json` als Subprozess aufrufen.

## Installation

Voraussetzungen: **Node.js 18.14+** oder **Python 3.10+** sowie ein API-Key aus der Claude Console.

```bash
# TypeScript
npm install @anthropic-ai/claude-agent-sdk
npm install --save-dev tsx

# Python mit uv
uv add claude-agent-sdk

# Python mit pip
python3 -m venv .venv && source .venv/bin/activate
pip install claude-agent-sdk
```

Beide SDKs bündeln eine native Claude-Code-Binary.
Ausnahmen: `npm ci --omit=optional` installiert keine Binary, und Python-Source-Distributionen (z. B. ARM64-Windows) bringen keine mit. In diesen Fällen Claude Code nativ installieren.

```bash
export ANTHROPIC_API_KEY=your-api-key     # Das SDK lädt KEINE .env-Dateien automatisch

# Alternativ ein Cloud-Provider (Credentials des Providers konfigurieren):
export CLAUDE_CODE_USE_BEDROCK=1          # Amazon Bedrock
export CLAUDE_CODE_USE_VERTEX=1           # Google Cloud Agent Platform
export CLAUDE_CODE_USE_FOUNDRY=1          # Microsoft Foundry
```

## Offizielle Dokumentation

- Überblick: https://code.claude.com/docs/en/agent-sdk/overview
- Quickstart: https://code.claude.com/docs/en/agent-sdk/quickstart
- TypeScript-Referenz: https://code.claude.com/docs/en/agent-sdk/typescript
- Python-Referenz: https://code.claude.com/docs/en/agent-sdk/python
- Permissions: https://code.claude.com/docs/en/agent-sdk/permissions
- Hooks: https://code.claude.com/docs/en/agent-sdk/hooks
- MCP: https://code.claude.com/docs/en/agent-sdk/mcp
- Subagents: https://code.claude.com/docs/en/agent-sdk/subagents
- Sessions: https://code.claude.com/docs/en/agent-sdk/sessions
- Hosting: https://code.claude.com/docs/en/agent-sdk/hosting
- Troubleshooting: https://code.claude.com/docs/en/agent-sdk/troubleshooting
- Repos und Changelogs: https://github.com/anthropics/claude-agent-sdk-typescript, https://github.com/anthropics/claude-agent-sdk-python
- Beispiel-Agenten: https://github.com/anthropics/claude-agent-sdk-demos

## Codebeispiele

### Python

```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, ResultMessage


async def main():
    async for message in query(
        prompt="Review utils.py for bugs that would cause crashes. Fix any issues you find.",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Edit", "Glob"],   # ohne Rückfrage erlaubt
            permission_mode="acceptEdits",            # Datei-Edits automatisch akzeptieren
            system_prompt="You are a senior Python developer. Follow PEP 8.",
        ),
    ):
        if isinstance(message, AssistantMessage):
            for block in message.content:
                if hasattr(block, "text"):
                    print(block.text)
                elif hasattr(block, "name"):
                    print(f"Tool: {block.name}")
        elif isinstance(message, ResultMessage):
            print(f"Done: {message.subtype}")


asyncio.run(main())
```

### TypeScript (ESM, `"type": "module"` in package.json)

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Review utils.py for bugs that would cause crashes. Fix any issues you find.",
  options: {
    allowedTools: ["Read", "Edit", "Glob"],
    permissionMode: "acceptEdits",
  },
})) {
  if (message.type === "assistant" && message.message?.content) {
    for (const block of message.message.content) {
      if ("text" in block) console.log(block.text);
      else if ("name" in block) console.log(`Tool: ${block.name}`);
    }
  } else if (message.type === "result") {
    console.log(`Done: ${message.subtype}`);
  }
}
```

```bash
npx tsx agent.ts     # bzw. uv run agent.py / python agent.py
```

### Tool-Profile

| Tools | Fähigkeit |
| --- | --- |
| `Read`, `Glob`, `Grep` | Nur lesen und analysieren |
| `Read`, `Edit`, `Glob` | Code analysieren und ändern |
| `Read`, `Edit`, `Bash`, `Glob`, `Grep` | Volle Automatisierung |

## Regeln für die Verwendung

1. **Authentifizierung mit API-Key oder Cloud-Provider.** Anthropic erlaubt Drittanbietern ohne vorherige Genehmigung nicht, claude.ai-Login oder -Rate-Limits in ihren Produkten anzubieten.
2. **Branding:** erlaubt sind z. B. „Claude Agent“ oder „{YourAgentName} Powered by Claude“. **Nicht** erlaubt sind „Claude Code“, „Claude Code Agent“ oder Claude-Code-Optik.
3. **Kleinste Tool-Menge** pro Agent wählen. `Bash` nur, wenn nötig.
4. **`permission_mode`/`permissionMode` bewusst setzen.** `bypassPermissions` nur in isolierten Containern.
5. **Den Stream filtern und das Ergebnis auswerten** (`ResultMessage.subtype`). Fehler nicht stillschweigend ignorieren.
6. **Kosten und Schleifen begrenzen** (z. B. Turn- und Budget-Optionen laut SDK-Referenz). Nur Optionen nutzen, die in der Referenz stehen.
7. Die Nutzung unterliegt den **Commercial Terms of Service** von Anthropic.

## Fehlerbehebung

| Symptom | Lösung |
| --- | --- |
| `Not logged in` / `Invalid API key` | `ANTHROPIC_API_KEY` in der Shell setzen, die den Agenten startet; `.env` wird nicht automatisch geladen (z. B. `dotenv` nutzen) |
| Top-level `await` schlägt fehl (TS) | `npm pkg set type=module` oder die Datei als `agent.mts` benennen |
| Binary nicht gefunden | Ohne `--omit=optional` neu installieren oder Claude Code nativ installieren und `pathToClaudeCodeExecutable` setzen |
| PowerShell blockiert `Activate.ps1` | `Set-ExecutionPolicy -Scope Process RemoteSigned` |
| Agent fragt nach und hängt | Benötigte Tools in `allowedTools` aufnehmen oder den Permission-Modus anpassen |
| Unerwartete Projekt-Settings wirken | Das SDK lädt `.claude/` und `~/.claude/` wie Claude Code; Arbeitsverzeichnis und Settings prüfen |

## Sicherheitsgrenzen

- **Der Agent handelt mit den Rechten des Prozesses.** In Produktion in einem Container oder einer Sandbox mit eingeschränktem Dateisystem und Netzwerk betreiben (siehe Hosting-Doku).
- In nicht-interaktiven Läufen **laden Projekt-MCP-Server ohne Rückfrage**. Fremde Repositories daher nicht ungeprüft als Arbeitsverzeichnis verwenden.
- **Nutzereingaben und Tool-Ergebnisse sind nicht vertrauenswürdig** (Prompt Injection). Kritische Aktionen über Hooks oder Permission-Callbacks absichern.
- **API-Keys** nur über Umgebungsvariablen oder einen Secret-Manager bereitstellen, nie im Code oder in Logs.
- Endnutzer-Daten nur gemäß den Datenschutz- und Aufbewahrungsrichtlinien verarbeiten.
