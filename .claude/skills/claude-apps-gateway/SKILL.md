---
name: claude-apps-gateway
description: Den selbst gehosteten Claude apps gateway (`claude gateway --config gateway.yaml`) vor Amazon Bedrock, Claude Platform on AWS, Google Cloud Agent Platform, Microsoft Foundry oder der Anthropic API betreiben – SSO/OIDC, Postgres, Gruppen-Policies, Deployment auf AWS (ECS/EKS) und GCP (Cloud Run) mit Terraform. Verwenden, wenn ein Infrastruktur-Agent Gateway-Konfiguration, Deployment oder Client-Anbindung bearbeitet.
when_to_use: 'Auslöser: Claude apps gateway; gateway.yaml; Claude Code über Bedrock mit SSO; Gateway Terraform AWS/GCP; /login Cloud gateway'
---

# Claude apps gateway

## Zweck und Einsatzgebiet

Der Claude apps gateway ist ein **selbst gehosteter Dienst** zwischen den Claude-Code-Clients der Entwickler und dem Modell-Provider. Er:

- meldet Entwickler per **Unternehmens-IdP (OIDC/SSO)** an, sodass keine API-Keys auf Laptops liegen,
- hält die Upstream-Credentials (Bedrock, Claude Platform on AWS, Google Cloud Agent Platform, Microsoft Foundry, Anthropic API) mit Failover,
- setzt **Modellzugriff und Managed Settings pro IdP-Gruppe** serverseitig durch,
- leitet Telemetrie (OTLP) an den eigenen Observability-Stack weiter.

Er ist im `claude`-Binary enthalten. Einsatz: Data-Residency-Anforderungen und Inferenz über den eigenen Cloud-Account.

## Installation

**Voraussetzungen**
- Claude Code **v2.1.195 oder neuer** auf dem Server und auf jedem Entwicklerrechner (`claude update`)
- PostgreSQL 11+ (am besten aktuell, mit `?sslmode=require`)
- Ein Modell-Upstream
- HTTPS-Erreichbarkeit
- Ein Hostname, der nur auf **private Adressen** auflöst

```bash
claude gateway --config gateway.yaml
```

Der Start bricht ab (fail-closed), wenn Config, Postgres, OIDC-Discovery oder ein Upstream-Client fehlerhaft sind.
Erfolgreicher Start: `claude gateway listening on http://0.0.0.0:8080`.

**Referenz-Deployments im claude-code-Repo** (`examples/gateway/`):
- `aws/`: ECS Fargate oder EKS, RDS PostgreSQL, Secrets Manager, IAM-Rolle für Bedrock, Okta als IdP. Enthält `setup.sh`, `Dockerfile`, `gateway.yaml.example` und `terraform/` (zwei Apply-Durchläufe, siehe dortiges README).
- `gcp/`: Cloud Run oder GKE, Cloud SQL for PostgreSQL, Secret Manager, Service-Account-Auth zur Agent Platform, Google Workspace als IdP. Dieselben Dateien wie bei AWS, `setup.sh` arbeitet mit `gcloud`.

## Offizielle Dokumentation

- Überblick und Quickstart: https://code.claude.com/docs/en/claude-apps-gateway
- Konfigurationsreferenz: https://code.claude.com/docs/en/claude-apps-gateway-config
- Deployment und Threat Model: https://code.claude.com/docs/en/claude-apps-gateway-deploy
- Walkthrough AWS: https://code.claude.com/docs/en/claude-apps-gateway-on-aws
- Walkthrough GCP: https://code.claude.com/docs/en/claude-apps-gateway-on-gcp
- Spend Limits: https://code.claude.com/docs/en/claude-apps-gateway-spend-limits
- Referenz-Artefakte: https://github.com/anthropics/claude-code/tree/main/examples/gateway
- AWS-Samples: https://github.com/aws-samples/anthropic-on-aws/tree/main/claude-apps-gateway

## Codebeispiele

### Auszug aus `gateway.yaml` (Struktur aus `examples/gateway/aws/gateway.yaml.example`)

```yaml
listen:
  host: 0.0.0.0
  port: 8080
  # Pflicht: feste externe Origin, damit nichts vom Host-Header abgeleitet wird
  public_url: https://claude-gateway.internal.example.com
  # Nur die Subnetze des Load Balancers, damit X-Forwarded-For vertraut wird
  trusted_proxies: [10.0.1.0/24, 10.0.2.0/24]

# Secrets per Expansion statt im Klartext:
#   ${ENV_VAR}     liest eine Umgebungsvariable
#   ${file:/pfad}  liest eine gemountete Datei
```

Die übrigen Pflichtabschnitte (IdP/OIDC, Postgres, Upstreams, Modelle) und alle optionalen Schlüssel stehen in der **Konfigurationsreferenz**. Sie nicht aus dem Gedächtnis ergänzen, sondern aus `gateway.yaml.example` und der Referenz übernehmen.

### AWS-Referenz nutzen

```bash
cd examples/gateway/aws
cp gateway.yaml.example gateway.yaml      # alle REPLACE_ME ersetzen; setup.sh baut sonst nicht
./setup.sh                                # Walkthrough per aws CLI
# oder: terraform/ laut terraform/README.md (Two-Pass-Apply)
```

### Entwicklerrechner anbinden (Managed Settings)

```json
{
  "forceLoginMethod": "gateway",
  "forceLoginGatewayUrl": "https://claude-gateway.internal.example.com",
  "parentSettingsBehavior": "merge"
}
```

Danach in Claude Code `/login` ausführen, im Screen **Cloud gateway** Enter drücken und die Anmeldung im Browser abschließen.
Ohne Managed Settings können `forceLoginMethod` und `forceLoginGatewayUrl` ab v2.1.295 auch in `~/.claude/settings.json` stehen.

## Regeln für die Verwendung

1. **Secrets nie in `gateway.yaml` eintragen.** `${ENV_VAR}` oder `${file:…}` verwenden und die Werte aus Secrets Manager bzw. Secret Manager injizieren. Die ausgefüllte `gateway.yaml` ist im Beispiel gitignored.
2. **Upstream-Credentials über Workload-Identitäten** (IRSA, ECS-Task-Rolle, Instance Profile, GCP-Service-Account) bereitstellen, nicht als statische Keys.
3. **Bedrock-IAM:** `bedrock:InvokeModel` und `bedrock:InvokeModelWithResponseStream` auf die `inference-profile/us.anthropic.*`- und `foundation-model/anthropic.*`-ARNs. Das einmalige Use-Case-Formular im Bedrock-Konsolen-Katalog muss eingereicht sein.
4. **`public_url` setzen** und TLS per `listen.tls` oder per TLS-terminierendem Ingress bereitstellen.
5. **`trusted_proxies` eng halten** und die Subnetze nicht mit nicht vertrauenswürdigen Workloads teilen.
6. **Gateway-Sessions holen keine server-managed Settings aus claude.ai.** Policies im `managed`-Block des Gateways pflegen, und dort `forceLoginMethod`/`forceLoginOrgUUID` weglassen.
7. **Postgres sichern**, wenn Spend Limits aktiv sind (sie speichern dauerhafte Spend-, Audit- und Identitätsdaten).

## Fehlerbehebung

| Symptom | Lösung |
| --- | --- |
| Gateway beendet sich beim Start | Die letzte stderr-Zeile nennt Config, Postgres, OIDC oder Upstream; den genannten Teil korrigieren |
| `/login` lehnt den Gateway ab | Hostname löst auf eine öffentliche IP auf; private Adresse nutzen oder eigene Public-Blöcke in `gatewayInternalNetworks` erklären (ab v2.1.268) |
| `/login` scheitert hinter einem Unternehmens-Proxy | Gateway-Host in `NO_PROXY` aufnehmen |
| `http://` wird abgelehnt | Nur Loopback-Hosts dürfen `http` nutzen; sonst HTTPS |
| Sessions beenden sich direkt nach dem Login | `forceLoginMethod`/`forceLoginOrgUUID` stehen im `managed`-Block des Gateways und müssen entfernt werden |
| `setup.sh` baut nicht | In `gateway.yaml` stehen noch `REPLACE_ME`-Platzhalter |
| Modell abgelehnt | Die IdP-Gruppe hat keine Freigabe für dieses Modell (serverseitige RBAC) |

## Sicherheitsgrenzen

- Der Gateway ist **sicherheitskritische Infrastruktur**. Er hält die Upstream-Credentials und setzt Policies durch. Änderungen nur per Review und IaC.
- **Nur intern erreichbar betreiben** (private Adressen, interner Load Balancer, Ingress auf das Unternehmensnetz beschränkt).
- **X-Forwarded-For-Spoofing:** Breite `trusted_proxies` erlauben gefälschte Client-IPs in Audit-Logs und Rate-Limits.
- Den **TLS-Fingerprint** für Entwickler veröffentlichen, damit sie ihn beim ersten `/login` vergleichen können.
- Deployments und `terraform apply` nur mit **ausdrücklicher Freigabe** durch einen Menschen ausführen, nie eigenmächtig durch einen Agenten.
- Die Beispiel-Artefakte sind ein **Arbeitsbeispiel, keine supportete Produktionsumgebung** und müssen an die eigene Umgebung angepasst werden.
