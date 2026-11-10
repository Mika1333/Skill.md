---
name: claude-code-plugins
description: Claude-Code-Plugins und Plugin-Marketplaces installieren, erstellen, validieren und verteilen (plugin.json, marketplace.json, /plugin, claude plugin, --plugin-dir). Verwenden, wenn ein Agent Skills, Subagents, Hooks und MCP-Server als ein Paket bündeln, ein Plugin testen oder einen Team-Marketplace aufsetzen soll.
when_to_use: 'Auslöser: Plugin erstellen; Plugin installieren; Marketplace hinzufügen; plugin.json; Plugin lädt nicht'
---

# Claude Code Plugins & Marketplaces

## Zweck und Einsatzgebiet

Ein Plugin ist ein Verzeichnis mit Skills, Subagents, Hooks, MCP-Servern (und weiteren Komponenten) plus Manifest `.claude-plugin/plugin.json`. Es wird als **eine Einheit** installiert.
Ein Marketplace ist ein Katalog (`.claude-plugin/marketplace.json`), der Plugins und ihre Quellen auflistet.

Plugins lohnen sich, wenn Setups **geteilt, in vielen Projekten genutzt oder versioniert** werden sollen. Für ein einzelnes Projekt reichen standalone `.claude/`-Dateien.

## Installation

```text
/plugin                                     # Manager: Discover, Installed, Marketplaces, Errors
/plugin marketplace add <owner/repo | git-URL | pfad>
/plugin install <plugin>@<marketplace>
/reload-plugins                             # Änderungen auf Disk neu laden
```

```bash
claude plugin install code-review@claude-plugins-official
claude plugin list
claude plugin disable <plugin>@<marketplace>
claude plugin validate ./mein-plugin        # Manifest und Frontmatter prüfen (--strict: Warnungen = Fehler)
claude --plugin-dir ./mein-plugin           # nur für diese Session laden, ohne Installation
claude plugin init mein-tool                # Gerüst unter ~/.claude/skills/ erzeugen
```

Install-Scopes:
- **user:** du, in allen Projekten
- **project:** alle im Repo, über die committete `.claude/settings.json`
- **local:** du, nur in diesem Repo

## Offizielle Dokumentation

- Überblick: https://code.claude.com/docs/en/plugins
- Plugin erstellen: https://code.claude.com/docs/en/plugins/create
- Komponenten: https://code.claude.com/docs/en/plugins/components
- Manifest-Referenz: https://code.claude.com/docs/en/plugins/manifest-reference
- Installieren und verwalten: https://code.claude.com/docs/en/plugins/install
- Marketplace erstellen: https://code.claude.com/docs/en/plugins/create-marketplace
- Sicherheit und Vertrauen: https://code.claude.com/docs/en/plugins/security
- Plugins für Organisationen: https://code.claude.com/docs/en/plugins/org
- Fehlerbehebung: https://code.claude.com/docs/en/plugins/troubleshooting
- Evals für Plugins: https://code.claude.com/docs/en/plugin-evals
- Offizielle Plugins: https://github.com/anthropics/claude-plugins-official

## Codebeispiele

### Layout

```
mein-plugin/
├── .claude-plugin/
│   └── plugin.json          # NUR das Manifest liegt hier
├── skills/
│   └── review/SKILL.md      # → /mein-plugin:review
├── agents/
│   └── reviewer.md          # → Subagent mein-plugin:reviewer
├── hooks/
│   └── hooks.json           # gleiches Format wie "hooks" in settings.json
├── .mcp.json                # MCP-Server des Plugins
└── README.md
```

### `plugin.json`

```json
{
  "name": "mein-plugin",
  "description": "Review- und Release-Werkzeuge für unser Team",
  "version": "1.0.0",
  "author": { "name": "Plattform-Team" }
}
```

`name` ist Pflicht (ohne Leerzeichen) und wird zum Präfix aller Skills und Agents.

### Hooks und MCP im Plugin mit `${CLAUDE_PLUGIN_ROOT}`

```json
{
  "mcpServers": {
    "db-tools": {
      "command": "${CLAUDE_PLUGIN_ROOT}/servers/db-server",
      "args": ["--config", "${CLAUDE_PLUGIN_ROOT}/config.json"]
    }
  }
}
```

### Team-Marketplace `.claude-plugin/marketplace.json` (Struktur wie im claude-code-Repo)

```json
{
  "$schema": "https://json.schemastore.org/claude-code-marketplace.json",
  "name": "team-tools",
  "owner": { "name": "Plattform-Team" },
  "plugins": [
    {
      "name": "mein-plugin",
      "description": "Review- und Release-Werkzeuge",
      "source": {
        "source": "git-subdir",
        "url": "https://github.com/acme/claude-plugins.git",
        "path": "plugins/mein-plugin",
        "ref": "main"
      }
    }
  ]
}
```

### Plugins in GitHub Actions

```yaml
- uses: anthropics/claude-code-action@v1
  with:
    plugin_marketplaces: "https://github.com/anthropics/claude-plugins-official.git"
    plugins: "code-review@claude-plugins-official"
    prompt: "/code-review:code-review"
```

## Regeln für die Verwendung

1. **Nur `plugin.json` gehört in `.claude-plugin/`.** `skills/`, `agents/`, `hooks/` und `.mcp.json` liegen im Plugin-Root.
2. **Vor jedem Test und Release** `claude plugin validate <pfad>` ausführen.
3. **Entwickeln mit `--plugin-dir`**, Änderungen mit `/reload-plugins` laden.
4. **Bei der Migration aus `.claude/`** die Originale danach entfernen. Hooks haben kein Präfix und laufen sonst doppelt.
5. **Kontextkosten bedenken.** Jeder automatisch aufrufbare Skill bzw. Agent belegt in jedem Zug Kontext; ungenutzte Plugins deaktivieren.
6. **Versionieren:** `version` setzen, wenn Nutzer bis zum nächsten Release auf einer Version bleiben sollen.

## Fehlerbehebung

| Symptom | Lösung |
| --- | --- |
| Skills des Plugins fehlen | `skills/` liegt in `.claude-plugin/` oder ein Manifest-Pfad zeigt auf eine Datei; in den Root verschieben, `/reload-plugins` |
| `<component> path not found` (Errors-Tab) | Pfad im Manifest korrigieren oder Verzeichnis anlegen |
| `--plugin-dir` auf Marketplace-Root lädt nichts | Auf den Ordner eines einzelnen Plugins zeigen oder den Marketplace hinzufügen |
| `userConfig`-Dialog erscheint nicht | `/plugin configure <plugin>` in der Session ausführen |
| Plugin im Projekt aktiviert, aber nicht geladen | Jede Person muss es lokal installieren; Workspace-Trust prüfen |
| Plugin in einer Cloud-Session fehlt | Cloud-Sessions laden keine lokalen Plugins |

## Sicherheitsgrenzen

- **Jedes Plugin läuft mit den Rechten des Nutzers**, auch aus offiziellen Marketplaces. Hooks, MCP-Server und Skripte vor der Installation prüfen.
- Die Namen offizieller Marketplaces (`claude-plugins-official`, `claude-code-plugins`) werden nur aus `github.com/anthropics/`-Repos akzeptiert. Alle anderen Marketplaces sind Drittanbieter.
- `--plugin-url` nur auf Archive richten, die du kontrollierst.
- Organisationen können Marketplaces per Managed Settings erlauben, blockieren oder Plugins erzwingen (`strictKnownMarketplaces`). Das nicht umgehen.
- Keine Secrets in `plugin.json`, `hooks.json` oder `.mcp.json`. Variablen oder `userConfig` verwenden.
