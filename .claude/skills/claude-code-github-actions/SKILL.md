---
name: claude-code-github-actions
description: Claude Code in GitHub Actions mit anthropics/claude-code-action@v1 einrichten und härten (@claude-Mentions, Automations-Prompts, Skills/Plugins in CI, OIDC/Workload Identity, Bedrock/Vertex/Foundry). Verwenden, wenn ein Agent CI-Workflows mit Claude anlegen, absichern oder Fehler wie "Claude antwortet nicht" beheben soll.
when_to_use: 'Auslöser: @claude in GitHub; claude-code-action Workflow; Claude in CI; Code Review Workflow; Claude reagiert nicht auf @claude'
---

# Claude Code GitHub Actions

## Zweck und Einsatzgebiet

`anthropics/claude-code-action` führt Claude Code in GitHub-Workflows aus. Es gibt zwei Modi:

- **Interaktiv** (kein `prompt`): reagiert auf `@claude` in Issues, PR-Kommentaren und Reviews.
- **Automation** (mit `prompt`): läuft bei beliebigen Events, z. B. PR geöffnet oder Cron.

Typische Aufgaben: Issue → PR, Fehler aus Kommentaren fixen, Code-Review, Triage, tägliche Reports.

## Installation

**Schnell:** in Claude Code `/install-github-app` ausführen. Voraussetzungen sind Admin-Rechte auf dem Repo und eine installierte, angemeldete `gh` CLI (`gh auth login`).

**Manuell:**
1. Die Claude GitHub App installieren: https://github.com/apps/claude
2. Ein Repository-Secret anlegen:
   - `ANTHROPIC_API_KEY` (aus https://platform.claude.com) **oder**
   - `CLAUDE_CODE_OAUTH_TOKEN` (lokal erzeugt mit `claude setup-token`)
3. `examples/claude.yml` aus dem Action-Repo nach `.github/workflows/` kopieren.

## Offizielle Dokumentation

- GitHub Actions: https://code.claude.com/docs/en/github-actions
- Cloud-Provider (Bedrock, Vertex, Foundry): https://code.claude.com/docs/en/github-actions-cloud-providers
- Action-Repo: https://github.com/anthropics/claude-code-action
- Inputs-Referenz: https://github.com/anthropics/claude-code-action/blob/main/docs/usage.md#inputs
- Security: https://github.com/anthropics/claude-code-action/blob/main/docs/security.md
- Setup (Workload Identity, eigene App): https://github.com/anthropics/claude-code-action/blob/main/docs/setup.md
- Migration von `@beta`: https://github.com/anthropics/claude-code-action/blob/main/docs/migration-guide.md
- Beispiele: https://github.com/anthropics/claude-code-action/tree/main/examples

## Codebeispiele

### Auf @claude reagieren

```yaml
name: Claude Code
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
jobs:
  claude:
    if: contains(github.event.comment.body, '@claude')
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write
      actions: read
    steps:
      - uses: actions/checkout@v6
        with:
          fetch-depth: 1
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

### Skill aus dem Repo bei jedem PR ausführen

```yaml
on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]
jobs:
  review:
    runs-on: ubuntu-latest
    permissions: { contents: read, pull-requests: read, issues: read, id-token: write }
    steps:
      - uses: actions/checkout@v6          # nötig, damit .claude/skills/ vorhanden ist
        with: { fetch-depth: 1 }
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          prompt: "/mein-review-skill"
          claude_args: "--max-turns 10"
```

### Gehärtete Variante (wie im claude-code-Repo): ohne statischen Key, Auto-Modus

```yaml
    permissions:
      contents: read
      pull-requests: read
      issues: read
      id-token: write
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4, per SHA gepinnt
        with: { fetch-depth: 1 }
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_federation_rule_id: ${{ vars.ANTHROPIC_FEDERATION_RULE_ID }}
          anthropic_organization_id: ${{ vars.ANTHROPIC_ORGANIZATION_ID }}
          anthropic_service_account_id: ${{ vars.ANTHROPIC_SERVICE_ACCOUNT_ID }}
          anthropic_workspace_id: ${{ vars.ANTHROPIC_WORKSPACE_ID }}
          claude_args: "--permission-mode auto"
```

### Häufige Inputs

| Input | Zweck |
| --- | --- |
| `prompt` | Text oder Skill-Aufruf (`/skill`, `/plugin:skill`); ohne `prompt` gilt der interaktive Modus |
| `claude_args` | Beliebige CLI-Argumente (`--max-turns`, `--model`, `--allowedTools`, `--mcp-config`) |
| `anthropic_api_key` / `claude_code_oauth_token` | Authentifizierung |
| `github_token` | Eigener Token; ohne ihn authentifiziert die Action als Claude GitHub App |
| `plugin_marketplaces` / `plugins` | Plugins vor dem Lauf installieren |
| `settings` | Claude-Code-Settings als JSON oder Pfad |
| `trigger_phrase` | Standard ist `@claude` |
| `use_bedrock` / `use_vertex` / `use_foundry` | Cloud-Provider per OIDC nutzen |
| `allowed_bots` / `allowed_non_write_users` | Ausnahmen von den Prüfungen „menschlicher Akteur“ und „Schreibrecht“ |

## Regeln für die Verwendung

1. **Secrets nur als GitHub Secrets** referenzieren (`${{ secrets.… }}`), nie im YAML oder Repo.
2. **Minimale `permissions:`** pro Job. `id-token: write` ist für App- und OIDC-Auth nötig.
3. **Kosten und Laufzeit begrenzen:** `--max-turns`, `timeout-minutes` am Job und `concurrency`-Gruppen.
4. **Im Automation-Modus Tools gezielt freigeben** (`--allowedTools` oder `settings`). Ohne Freigabe hat Claude keinen Shell- oder API-Zugriff.
5. **Projektstandards in `CLAUDE.md`** ablegen und kurz halten, sie werden bei jedem Lauf gelesen.
6. **Third-Party-Actions per Commit-SHA pinnen** und Workflows im Review prüfen.
7. **Von `@beta` migrieren:**
   - `@v1` verwenden
   - `mode` entfernen
   - `direct_prompt` → `prompt`
   - Optionen wie `max_turns` in `claude_args` verschieben

## Fehlerbehebung

| Symptom | Lösung |
| --- | --- |
| Keine Reaktion auf `@claude` | App installiert? Workflows aktiv? Secret gesetzt? Exakt `@claude` als ganzes Wort? Hat der Nutzer Schreibrechte? |
| Bot löst keinen Lauf aus | Bots werden abgewiesen, außer sie stehen in `allowed_bots` (das gilt auch für Cron-Läufe, die einem Bot zugeordnet sind) |
| CI läuft nicht auf Claudes Commits | Commits mit dem Standard-`GITHUB_TOKEN` triggern keine Workflows; `github_token` weglassen (App-Auth) oder ein App-Token nutzen |
| Auth-Fehler | Key bzw. Token lokal mit `claude` testen; bei Cloud-Providern die OIDC-Konfiguration prüfen |
| PRs aus Forks werden nicht reviewt | GitHub gibt Secrets nicht an Fork-PRs weiter (gewolltes Verhalten) |
| Inline-Kommentare fehlen | `--allowedTools "mcp__github_inline_comment__create_inline_comment"` in `claude_args` setzen |

## Sicherheitsgrenzen

- **Inhalte aus Issues und PRs sind nicht vertrauenswürdig** (Prompt Injection). Nicht-Schreibberechtigte nur bewusst über `allowed_non_write_users` zulassen.
- **Keine Tokens oder Env-Variablen in Logs ausgeben.**
- **Auf `pull_request_target` mit Checkout von Fork-Code verzichten**, wenn Secrets im Job verfügbar sind.
- Die offizielle App hat ein breites Rechte-Set. Wo nur Contents, Issues und PRs nötig sind, eine eigene GitHub App verwenden.
- **Egress begrenzen**, z. B. wie im claude-code-Repo mit Egress-Firewall-Runner und Host-Allowlist (`.github/egress-firewall.yaml`, `mode: enforce`).
- Claude darf in CI **nicht selbst mergen oder Branch-Protection umgehen**. Änderungen vor dem Merge von Menschen reviewen lassen.
