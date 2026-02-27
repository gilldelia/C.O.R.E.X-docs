# Epic 3 — Smoke Tests Fonctionnels

**Objectif** : Valider US3.1 à US3.6 en < 10 minutes sur un environnement local propre.

---

## Pré-requis (une seule fois)

1. **SDK** : .NET 10 preview (`dotnet --version` → `10.x`)
2. **Secrets Slack** configurés (user-secrets ou `appsettings.local.json`) :
   - `ExternalServices:Slack:BotToken` (scope `chat:write`)
   - `ExternalServices:Slack:AppLevelToken` (Socket Mode)
   - `ExternalServices:Slack:SigningSecret`
3. **App Slack** jointe au canal de test (ex. `#corex-test`)
4. **Repo cloné** et restauré : `dotnet restore C.O.R.E.X.sln`

---

## Lancement du runtime

```powershell
# Depuis la racine du repo
DOTNET_ENVIRONMENT=Local dotnet run --project src/Corex.App/Corex.App.csproj -c Debug
```

Attendu dans la console :
```
runtime.bootstrap.start environment=Local slack_enabled=True
runtime.loop.starting inbound=SlackSocketInboundEventSource
slack.socket.open_url.request
slack.socket.open_url.ok
slack.socket.connecting url=wss://…
slack.socket.connected
slack.socket.hello received
```

---

## Scénario 1 — US3.1 : Réception Slack → Runtime

| Action | Attendu (logs) |
|--------|----------------|
| Envoyer `ping 1` sur `#corex-test` | `runtime.event.received stage=Received outcome=Success channel=… text_preview=ping 1` |
| | `Agent event processing for bootstrap-agent` (niveau Debug) |

✅ **US3.1 validée** si ces logs apparaissent.

---

## Scénario 2 — US3.2 : Décision déterministe

| Message envoyé | Attendu (`runtime.event.decision`) |
|----------------|---------------------------------------|
| `help` | `decision=ShowHelp decision_reason=prefix=help` |
| `echo salut` | `decision=Echo decision_reason=prefix=echo` |
| `bonjour` | `decision=NoAction decision_reason=default` |

✅ **US3.2 validée** si chaque message produit la décision attendue.

---

## Scénario 3 — US3.3 : Action Slack (réponse)

| Message envoyé | Attendu (logs + Slack) |
|----------------|------------------------|
| `help` | Log `runtime.event.action stage=Action outcome=Success action=ShowHelp` |
| | **Slack** : COREX poste "COREX help: commands -> help, echo \<text\>." |
| `echo hello` | Log `runtime.event.action stage=Action outcome=Success action=Echo` |
| | **Slack** : COREX poste "hello" |
| `bonjour` | Log `runtime.event.action stage=Action outcome=Skipped action=NoAction reason=no_action_required` |
| | **Pas de post Slack** |

✅ **US3.3 validée** si les réponses Slack correspondent et les logs sont présents.

---

## Scénario 4 (optionnel) — Idempotence / Doublon technique

Pour simuler un doublon Slack (même `event_id`/`ts`) :

1. **Commenter temporairement l'ACK** dans `SlackSocketClient.cs` :
   ```csharp
   // await AckAsync(socket, envelopeId, cancellationToken);
   ```
2. Rebuild/run.
3. Envoyer `echo test` une seule fois.
4. Slack renvoie le message (retry_attempt > 0).

| Attendu |
|---------|
| 1er passage : `runtime.event.action outcome=Success` |
| 2e passage : `runtime.event.received outcome=Dropped reason=duplicate_runid` (pas de second post) |

5. **Remettre l'ACK** après le test.

✅ **Idempotence validée** si un seul post Slack malgré le retry.

---

## Résumé des validations

| US | Critère | ✅ / ❌ |
|----|---------|--------|
| US3.1 | Log `runtime.event.received stage=Received outcome=Success` | |
| US3.2 | `runtime.event.decision` correct pour help/echo/autre | |
| US3.3 | `runtime.event.action outcome=Success` + réponse Slack visible | |
| US3.5 | Chaque log de flux contient `stage` + `outcome` ; `run_id` dans le scope | |
| US3.6 | `fail test` → erreur loggée, runtime survit, `echo ok` fonctionne après | |
| (opt) | Doublon → `outcome=Dropped reason=duplicate_runid`, pas de double post | |

---

## Temps estimé

| Étape | Durée |
|-------|-------|
| Lancement runtime | 30s |
| Scénario 1 (US3.1) | 1 min |
| Scénario 2 (US3.2) | 2 min |
| Scénario 3 (US3.3) | 2 min |
| Scénario 5 (US3.5 — vérif logs) | 1 min |
| Scénario 6 (US3.6 — fail + echo) | 2 min |
| Scénario 4 (optionnel — doublon) | 3 min |
| **Total** | **< 10 min** |

---

## Troubleshooting

| Problème | Solution |
|----------|----------|
| `slack.socket.open_url` échoue | Vérifier `AppLevelToken` (Socket Mode activé sur l'app Slack) |
| `slack.message_posted` absent | Vérifier `BotToken` (scope `chat:write`) et app jointe au canal |
| `runtime.event.action stage=Action outcome=Skipped` | Le channel n'est pas remonté ; vérifier le payload Slack |
| `outcome=Dropped reason=duplicate_runid` inattendu | RunId identique (même event_id ou channel:ts) ; comportement normal si retry Slack |

---

## Scénario 5 — US3.5 : Contrat de logs

Après avoir envoyé `echo hello` (scénario 3), vérifier dans les logs :

| Étape | Log attendu | Champs |
|-------|-------------|--------|
| Received | `runtime.event.received` | `stage=Received`, `outcome=Success`, `run_id`, `event_id` |
| Decision | `runtime.event.decision` | `stage=Decision`, `outcome=Success`, `decision=Echo` |
| Action | `runtime.event.action` | `stage=Action`, `outcome=Success`, `action=Echo` |

Vérifier que le `run_id` est **constant** sur les 3 lignes (présent dans le scope Serilog : suffixe `corr=... agent=... event=...`).

✅ **US3.5 validée** si les 3 stages sont présents avec le même run_id.

---

## Scénario 6 — US3.6 : Gestion d'erreurs (pas de crash)

| Action | Attendu |
|--------|---------|
| Envoyer `fail test` | Log `runtime.event.decision decision=Fail` |
| | Log `runtime.event.error stage=Action outcome=Failed reason=InvalidOperationException` |
| | **Runtime reste actif** (pas de crash) |
| Envoyer `echo ok` juste après | Log `runtime.event.action stage=Action outcome=Success action=Echo` |
| | **Slack** : COREX poste "ok" |

✅ **US3.6 validée** si le runtime survit au `fail` et traite normalement l'`echo` suivant.

---

## Archivage des preuves

Après la démo, archiver les logs pour traçabilité :

```powershell
New-Item -ItemType Directory -Force -Path artifacts/epic3
Copy-Item logs/corex*.log artifacts/epic3/
```

> `artifacts/` est gitignored — les logs ne seront pas commités.

Pour une preuve plus complète, copier aussi la sortie console dans un fichier :
```powershell
# Relancer le runtime avec tee :
DOTNET_ENVIRONMENT=Local dotnet run --project src/Corex.App/Corex.App.csproj -c Debug 2>&1 | Tee-Object -FilePath artifacts/epic3/console-output.log
```

---

## Script PowerShell (scripts/smoke-epic3.ps1)

Voir le script réel dans le dépôt : `scripts/smoke-epic3.ps1`.

Le script utilise `scripts/run.ps1` avec l'environnement Local.

Usage :
```powershell
./scripts/smoke-epic3.ps1        # run once
./scripts/smoke-epic3.ps1 -Watch # hot reload
```
