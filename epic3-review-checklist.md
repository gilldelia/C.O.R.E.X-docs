# Epic 3 — Revue de fin & Checklist de clôture

**Date de la démo** : 2025-02-01  
**Validé par (PO)** : @gilldelia  
**Environnement**  : Local (DOTNET_ENVIRONMENT=Local)

---

## 1) Pré-requis PO — Configuration manuelle

Avant de lancer la démo, le PO (ou la personne qui valide) doit s'assurer que :

### Slack

| Élément | Comment | Vérifié |
|---------|---------|---------|
| App Slack créée avec Socket Mode activé | [api.slack.com/apps](https://api.slack.com/apps) → Features → Socket Mode | ✅ |
| Scopes bot : `chat:write`, `channels:history`, `app_mentions:read` | OAuth & Permissions → Bot Token Scopes | ✅ |
| App installée dans le workspace | Install App → Install to Workspace | ✅ |
| App ajoutée au canal de test (`#corex-test`) | Dans Slack : canal → Intégrations → Ajouter une app | ✅ |
| Tokens renseignés via user-secrets ou env | Voir section ci-dessous | ✅ |

### Secrets (au choix)

**Option A — user-secrets** (recommandé) :
```bash
dotnet user-secrets set "ExternalServices:Slack:BotToken" "xoxb-..." --project src/Corex.App/Corex.App.csproj
dotnet user-secrets set "ExternalServices:Slack:AppLevelToken" "xapp-..." --project src/Corex.App/Corex.App.csproj
dotnet user-secrets set "ExternalServices:Slack:SigningSecret" "..." --project src/Corex.App/Corex.App.csproj
```

**Option B — variables d'environnement** :
```powershell
$env:COREX__EXTERNALSERVICES__SLACK__BOTTOKEN = "xoxb-..."
$env:COREX__EXTERNALSERVICES__SLACK__APPLEVELTOKEN = "xapp-..."
$env:COREX__EXTERNALSERVICES__SLACK__SIGNINGSECRET = "..."
```

### Trello (désactivé par défaut)

Non requis pour l'Epic 3. Laisser `ExternalServices:Trello:Enabled=false` (par défaut dans `appsettings.json`).

---

## 2) Exécution de la démo

### Lancer le runtime
```powershell
DOTNET_ENVIRONMENT=Local dotnet run --project src/Corex.App/Corex.App.csproj -c Debug
```

### Suivre le runbook smoke tests
→ **`docs/epic3-smoke-tests.md`** (tous les scénarios listés)

---

## 3) Checklist de validation

### Epic 2 — Non-régression

| Critère | Vérifié |
|---------|---------|
| `./scripts/test.ps1` passe (tous les tests unitaires + intégration) | ✅ |
| Configuration validée au démarrage (log `Configuration validated successfully`) | ✅ |
| Exit code 2 si secret manquant (tester sans BotToken) | ✅ |
| Logs structurés avec `correlation_id`, `agent_id`, `event_id` dans le scope | ✅ |

### US3.1 — Réception Slack → Runtime

| Critère | Vérifié |
|---------|---------|
| Connexion Socket Mode réussie (log `slack.socket.connected`) | ✅ |
| Message reçu (log `runtime.event.received stage=Received outcome=Success`) | ✅ |
| RunId assigné de manière déterministe (même event → même RunId) | ✅ |

### US3.2 — Décision déterministe

| Critère | Vérifié |
|---------|---------|
| `help` → `runtime.event.decision decision=ShowHelp` | ✅ |
| `echo salut` → `runtime.event.decision decision=Echo` | ✅ |
| `bonjour` → `runtime.event.decision decision=NoAction` | ✅ |

### US3.3 — Action Slack (réponse)

| Critère | Vérifié |
|---------|---------|
| `help` → COREX poste le message d'aide sur Slack | ✅ |
| `echo hello` → COREX poste "hello" sur Slack | ✅ |
| `bonjour` → Aucun post Slack (log `outcome=Skipped`) | ✅ |
| Log `runtime.event.action stage=Action outcome=Success` présent | ✅ |

### US3.4 — Smoke tests

| Critère | Vérifié |
|---------|---------|
| `docs/epic3-smoke-tests.md` suivi en < 10 min | ✅ |
| Tous les scénarios documentés reproduits | ✅ |

### US3.5 — Contrat de logs

| Critère | Vérifié |
|---------|---------|
| Chaque log de flux contient `stage` (Received/Decision/Action) | ✅ |
| Chaque log contient `outcome` (Success/Dropped/Skipped/Failed) | ✅ |
| `run_id` présent dans le scope de tous les logs de flux | ✅ |
| Un scénario `echo hello` complet est reconstituable via logs seuls | ✅ |

### US3.6 — Gestion d'erreurs (pas de crash)

| Critère | Vérifié |
|---------|---------|
| `fail test` → log d'erreur avec `outcome=Failed` | ✅ |
| Runtime reste actif après `fail test` | ✅ |
| `echo ok` après `fail test` → réponse "ok" postée normalement | ✅ |

### Idempotence (optionnel)

| Critère | Vérifié |
|---------|---------|
| Doublon Slack (même RunId) → `outcome=Dropped reason=duplicate_runid` | ✅ |
| Pas de second post Slack sur doublon | ✅ |

---

## 4) Archivage des preuves

Après la démo, copier les logs dans `artifacts/epic3/` :

```powershell
New-Item -ItemType Directory -Force -Path artifacts/epic3
Copy-Item logs/corex*.log artifacts/epic3/
```

> Le dossier `artifacts/` est gitignored — les logs de démo ne seront pas commités.

---

## 5) Décision PO

| | |
|---|---|
| **Epic 3 validée** | ✅ Oui |
| **Commentaires** | Validation solo — tous les smoke tests passent, 127 tests CI green, logs conformes au contrat. |
| **Date** | 2025-02-01 |
| **Signature** | @gilldelia |

---

## Références

- Runbook local : `docs/runbook-local.md`
- Smoke tests : `docs/epic3-smoke-tests.md`
- Contrat de logs : `docs/log-contract.md`
- Configuration : `docs/configuration.md`
