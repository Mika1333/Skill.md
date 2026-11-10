---
name: claude-code-managed-settings
description: Organisationsweite Claude-Code-Policies verteilen (managed-settings.json, managed-settings.d/, managed-mcp.json, MDM via Jamf/Iru/Intune/Group Policy, macOS-Plist, Windows-Registry, server-managed Settings) und prüfen. Verwenden, wenn ein Admin-Agent Sicherheitsrichtlinien für alle Entwicklerrechner ausrollen oder klären soll, warum eine Policy nicht greift.
when_to_use: 'Auslöser: managed settings; MDM Claude Code; Policy für alle Entwickler; bypassPermissions organisationsweit sperren; HIPAA settings'
---

# Claude Code Managed Settings (Enterprise/MDM)

## Zweck und Einsatzgebiet

Managed Settings sind die Settings, die eine Organisation auf alle Entwicklerrechner verteilt.
Sie stehen **über allen anderen Ebenen** (User, Projekt, lokal, `--settings`) und können von Nutzern nicht überschrieben werden.
Ausnahme: einige sicherheitsrelevante Schlüssel, bei denen ein strengerer Wert von unten gilt.

Einsatz:
- Bypass-Modus sperren
- Erlaubte Modelle festlegen
- Permission-Regeln und Hooks zentral vorgeben
- Marketplaces und MCP-Server einschränken
- Sandbox erzwingen

## Installation

### Dateibasiert

| OS | Pfad |
| --- | --- |
| macOS | `/Library/Application Support/ClaudeCode/managed-settings.json` |
| Linux und WSL | `/etc/claude-code/managed-settings.json` |
| Windows | `C:\Program Files\ClaudeCode\managed-settings.json` |

Im selben Verzeichnis sind außerdem möglich:
- `managed-settings.d/*.json`: Teil-Policies pro Team, die zusammengeführt werden
- `managed-mcp.json`: vorgegebene MCP-Server

Der Legacy-Pfad `C:\ProgramData\ClaudeCode\…` wird **nicht** gelesen.

### MDM / OS-Policy

- **macOS:** Konfigurationsprofil mit der Preference-Domain `com.anthropic.claudecode`. Gleiche Schlüssel wie in der JSON-Datei, verschachtelte Werte als Dictionaries.
- **Windows:** JSON als `REG_SZ`/`REG_EXPAND_SZ`-Wert `Settings` unter `HKLM\SOFTWARE\Policies\ClaudeCode`.
- **HKCU** (`HKCU\SOFTWARE\Policies\ClaudeCode`) ist nur ein Fallback und vom Nutzer beschreibbar.
- MDM-Policies werden beim Start gelesen und alle 30 Minuten auf Änderungen geprüft.

### Server-managed

Über die claude.ai-Admin-Konsole oder einen Claude apps gateway. Nur diese Variante erreicht auch Cloud-Sessions.

Vorlagen für Jamf, Iru (Kandji), Intune und Group Policy liegen unter `examples/mdm/` im claude-code-Repo.

## Offizielle Dokumentation

- Managed Settings verteilen: https://code.claude.com/docs/en/managed-settings
- Server-managed Settings: https://code.claude.com/docs/en/server-managed-settings
- Admin-Setup: https://code.claude.com/docs/en/admin-setup
- Settings-Referenz: https://code.claude.com/docs/en/settings-reference
- Managed MCP: https://code.claude.com/docs/en/managed-mcp
- Plugins für Organisationen: https://code.claude.com/docs/en/plugins/org
- HIPAA-Setup: https://code.claude.com/docs/en/hipaa-setup
- Beispiele: https://github.com/anthropics/claude-code/tree/main/examples/mdm und https://github.com/anthropics/claude-code/tree/main/examples/settings

## Codebeispiele

### Minimal (wie `examples/mdm/managed-settings.json`)

```json
{
  "permissions": {
    "disableBypassPermissionsMode": "disable"
  }
}
```

### Strikt (nach `examples/settings/settings-strict.json`)

```json
{
  "permissions": {
    "disableBypassPermissionsMode": "disable",
    "ask": ["Bash"],
    "deny": ["WebSearch", "WebFetch"]
  },
  "allowManagedPermissionRulesOnly": true,
  "allowManagedHooksOnly": true,
  "strictKnownMarketplaces": []
}
```

| Schlüssel | Wirkung |
| --- | --- |
| `permissions.disableBypassPermissionsMode` / `disableAutoMode` | `"disable"` sperrt den jeweiligen Modus |
| `allowManagedPermissionRulesOnly` | Ignoriert allow-, ask- und deny-Regeln aus User- und Projekt-Settings |
| `allowManagedHooksOnly` | Nur Hooks aus Managed Settings laufen |
| `strictKnownMarketplaces` | `[]` blockiert Plugin-Marketplaces |
| `availableModels` / `deniedModels` | Modell-Allowlist bzw. -Denylist (`model` allein ist nur ein Default) |
| `sandbox.enabled` + `allowUnsandboxedCommands: false` | Bash muss in der Sandbox laufen |

### Windows per Skript (Intune Platform Script, siehe `examples/mdm/windows/Set-ClaudeCodePolicy.ps1`)

```powershell
New-Item -ItemType Directory -Force -Path "C:\Program Files\ClaudeCode" | Out-Null
Set-Content -Path "C:\Program Files\ClaudeCode\managed-settings.json" `
  -Value '{"permissions":{"disableBypassPermissionsMode":"disable"}}' -Encoding UTF8
```

### Prüfen

```text
/status         →  Setting sources: Enterprise managed settings (file | plist | HKLM | HKCU | remote …)
```

```bash
claude doctor   # listet verworfene und ungültige Einträge
```

## Regeln für die Verwendung

1. **Erst auf einem Rechner testen** (lokal als `managed-settings.json`), mit `/status` verifizieren, dann ausrollen.
2. **Gültiges JSON ist kritisch:** Ein vorhandenes, aber nicht parsebares Managed-Dokument (Datei, Drop-in, Plist, HKLM) **verhindert den Start** von Claude Code.
3. **Eine Quelle bevorzugen.** Standardmäßig gilt „first-wins“: Die ranghöchste Quelle mit Policy-Schlüsseln wird genutzt, die anderen werden ignoriert. `managedSourcesBehavior: "merge"` nur bewusst einsetzen.
4. **Mehrere Teams** über `managed-settings.d/` statt über eine gemeinsame Datei arbeiten lassen.
5. **Beispiele aus dem Repo sind Community-Vorlagen.** Für die Korrektheit ist die eigene Organisation verantwortlich.
6. **Bei `.mobileconfig`** die Platzhalter `PayloadUUID` und `PayloadOrganization` ersetzen (`uuidgen`).

## Fehlerbehebung

| Symptom | Lösung |
| --- | --- |
| Policy greift nicht | `/status` prüfen: Welche Quelle wurde gewählt und welche übersprungen? Eventuell gewinnt eine höhere Quelle (first-wins) |
| Claude Code startet nicht mehr | Managed-Dokument nicht parsebar; die Fehlermeldung nennt die Quelle. JSON reparieren |
| Änderung kommt nicht an | MDM- und Remote-Quellen werden zeitversetzt geladen; Session neu starten |
| HKCU-Wert wird ignoriert | HKCU gilt nur, wenn kein Admin-Dokument (HKLM, Datei, MDM) vorhanden ist |
| Symlink-Warnung in `/status` | Den Link auf ein Ziel innerhalb des Systemverzeichnisses zeigen lassen |
| Cloud-Sessions ignorieren die Policy | Datei- und MDM-Policies erreichen Cloud-Sessions nicht; server-managed Settings nutzen |

## Sicherheitsgrenzen

- **Lokale Administratoren** können dateibasierte Policies ändern. Für Durchsetzung MDM mit regelmäßigem Redeploy oder server-managed Settings verwenden.
- **Die Sandbox gilt nur für Bash**, nicht für Read/Write/WebFetch/MCP/Hooks. Diese Tools zusätzlich über Permission-Regeln einschränken.
- **Keine Secrets in Managed Settings** (`env`-Block): Die Werte liegen im Klartext auf jedem Rechner.
- Ein Agent darf Managed Settings **nur im Auftrag und mit Freigabe** eines Administrators ändern. Policies weder lockern noch umgehen.
- Bei HIPAA-Organisationen die dedizierten Vorlagen (`settings-hipaa.json`, `managed-mcp-hipaa.json`) und `README-hipaa.md` befolgen.
