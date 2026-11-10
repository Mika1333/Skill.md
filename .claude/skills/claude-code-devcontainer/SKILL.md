---
name: claude-code-devcontainer
description: Claude Code in Dev Containers betreiben (Dev Container Feature, Referenz-Container mit Egress-Firewall, persistente Auth-Volumes, Org-Policy im Image, unbeaufsichtigte Läufe). Verwenden, wenn ein Agent eine isolierte, reproduzierbare Umgebung für Claude Code aufsetzen oder Container-Probleme (Login, Firewall, Persistenz) lösen soll.
when_to_use: 'Auslöser: devcontainer mit Claude; Claude im Container; Firewall für Claude; Login nach Rebuild weg; dangerously-skip-permissions im Container'
---

# Claude Code im Dev Container

## Zweck und Einsatzgebiet

Ein Dev Container ist eine identische, isolierte Docker-Umgebung für das ganze Team.
Befehle, die Claude ausführt, laufen im Container. Dateiänderungen erscheinen über den Bind-Mount im lokalen Repo.

Einsatz:
- Reproduzierbare Toolchains
- Organisations-Policy im Image
- Netzwerk-Egress begrenzen
- Unbeaufsichtigte Läufe in einer abgeschotteten Umgebung

## Installation

### Variante A: Dev Container Feature (empfohlen für eigene Container)

```json
{
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu",
  "features": {
    "ghcr.io/anthropics/devcontainer-features/claude-code:1.0": {}
  }
}
```

Danach **Dev Containers: Rebuild Container** ausführen, im Container-Terminal `claude` starten und anmelden.
Der Tag `:1.0` pinnt nur das Install-Skript. Installiert wird die neueste Claude-Code-Version, die sich danach selbst aktualisiert.

### Variante B: Referenz-Container aus dem claude-code-Repo

Das Repo `anthropics/claude-code` klonen, in VS Code öffnen und **Reopen in Container** wählen. Der Referenz-Container besteht aus:

- `.devcontainer/devcontainer.json`: Volumes, `runArgs` (`NET_ADMIN`, `NET_RAW`), Extensions, `containerEnv`
- `.devcontainer/Dockerfile`: Basis-Image, Tools, Claude-Code-Installation
- `.devcontainer/init-firewall.sh`: begrenzt ausgehenden Traffic auf erlaubte Ziele (läuft als `postStartCommand`)

## Offizielle Dokumentation

- Dev Containers: https://code.claude.com/docs/en/devcontainer
- Feature-Quelle: https://github.com/anthropics/devcontainer-features/tree/main/src/claude-code
- Referenz-Konfiguration: https://github.com/anthropics/claude-code/tree/main/.devcontainer
- Netzwerkanforderungen: https://code.claude.com/docs/en/network-config
- Sandbox-Umgebungen im Vergleich: https://code.claude.com/docs/en/sandbox-environments
- Dev-Container-Spezifikation: https://containers.dev/

## Codebeispiele

### Auth und Settings über Rebuilds hinweg behalten

```json
"mounts": [
  "source=claude-code-config-${devcontainerId},target=/home/node/.claude,type=volume"
],
"containerEnv": {
  "CLAUDE_CONFIG_DIR": "/home/node/.claude"
}
```

`/home/node` an den Home-Pfad des `remoteUser` anpassen.

### Organisations-Policy ins Image backen

```dockerfile
RUN mkdir -p /etc/claude-code
COPY managed-settings.json /etc/claude-code/managed-settings.json
```

### Telemetrie und Auto-Update abschalten, Version pinnen

```json
"containerEnv": {
  "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1",
  "DISABLE_AUTOUPDATER": "1"
}
```

```dockerfile
# Statt des Features, wenn eine feste Version nötig ist:
RUN npm install -g @anthropic-ai/claude-code@X.Y.Z
```

### Unbeaufsichtigt (nur als Nicht-root und mit Egress-Begrenzung)

```bash
claude --dangerously-skip-permissions
```

## Regeln für die Verwendung

1. **`remoteUser` auf ein Nicht-root-Konto setzen.** Die CLI lehnt `--dangerously-skip-permissions` als root ab.
2. **`CLAUDE_CONFIG_DIR` auf den Volume-Pfad setzen**, sonst geht `~/.claude.json` bei jedem Rebuild verloren.
3. **Policy im Dockerfile ist umgehbar** für alle mit Schreibrecht aufs Repo. Verbindliche Policy über server-managed Settings oder MDM verteilen.
4. **MCP-Server** projektweit in `.mcp.json` definieren, Binaries im Dockerfile installieren und Remote-Domains auf die Firewall-Allowlist setzen.
5. **Auto-Modus vor Bypass:** Für weniger Rückfragen zuerst `--permission-mode auto` prüfen.
6. **Credentials für Cloud-Provider** per `containerEnv`, Codespaces-Secret oder Workload Identity übergeben, nicht als Datei-Mount vom Host.

## Fehlerbehebung

| Symptom | Lösung |
| --- | --- |
| Setup bzw. Login nach jedem Rebuild | Volume auf `~/.claude` mounten **und** `CLAUDE_CONFIG_DIR` auf denselben Pfad setzen |
| `Failed to install Node.js and npm` | `"ghcr.io/devcontainers/features/node:1": {}` vor dem Claude-Feature einfügen |
| OAuth-Login: kein Browser oder `localhost`-Fehler | Die angezeigte Login-URL manuell im Browser öffnen und den Code einfügen |
| Firewall-Skript schlägt fehl | `NET_ADMIN`/`NET_RAW` in `runArgs` prüfen; das Skript braucht beim Start Zugriff auf `api.github.com/meta` |
| Remote Control funktioniert nicht | `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` deaktiviert die nötige Feature-Flag-Abfrage |
| Codespaces verliert Auth | `ANTHROPIC_API_KEY` oder `CLAUDE_CODE_OAUTH_TOKEN` als Codespaces-Secret hinterlegen |

## Sicherheitsgrenzen

- Laut offizieller Doku ist **kein System immun**. Mit `--dangerously-skip-permissions` kann ein bösartiges Projekt alles exfiltrieren, was im Container erreichbar ist, **inklusive der Claude-Credentials in `~/.claude`**.
- Container mit Bypass **nur mit vertrauenswürdigen Repositories** verwenden und die Aktivitäten überwachen.
- **Keine Host-Secrets mounten** (`~/.ssh`, Cloud-Credential-Dateien). Repo-gebundene, kurzlebige Tokens bevorzugen.
- Der Bind-Mount macht **jede Dateiänderung im Workspace auf dem Host sichtbar**. Der Container schützt das Repo selbst nicht.
- Firewall-Capabilities (`NET_ADMIN`) nur für das Firewall-Skript verwenden und nicht für andere Zwecke erweitern.
