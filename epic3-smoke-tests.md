# Epic 3 — Smoke Tests Fonctionnels

**Objectif** : Valider US3.1, US3.2, US3.3 en < 10 minutes sur un environnement local propre.

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
| Envoyer `ping 1` sur `#corex-test` | `slack.message_received channel=… ts=… user=… text=ping 1` |
| | `runtime.runid_assigned run_id=… external_id=…` |
| | `runtime.message_enqueued run_id=… event_id=…` |
| | `Agent event received for bootstrap-agent` |

✅ **US3.1 validée** si ces 4 logs apparaissent.

---

## Scénario 2 — US3.2 : Décision déterministe

| Message envoyé | Attendu (`runtime.decision_computed`) |
|----------------|---------------------------------------|
| `help` | `decision=ShowHelp reason=prefix=help` |
| `echo salut` | `decision=Echo reason=prefix=echo` |
| `bonjour` | `decision=NoAction reason=default` |

✅ **US3.2 validée** si chaque message produit la décision attendue.

---

## Scénario 3 — US3.3 : Action Slack (réponse)

| Message envoyé | Attendu (logs + Slack) |
|----------------|------------------------|
| `help` | Log `runtime.action_selected action=ShowHelp` |
| | Log `slack.message_posted run_id=… channel=… ts=…` |
| | **Slack** : COREX poste "COREX help: commands -> help, echo <text>." |
| `echo hello` | Log `runtime.action_selected action=Echo` |
| | Log `slack.message_posted …` |
| | **Slack** : COREX poste "hello" |
| `bonjour` | Log `runtime.action_selected action=NoAction` |
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
| 1er passage : `slack.message_posted` |
| 2e passage : `duplicate_runid_dropped` (pas de second `slack.message_posted`) |

5. **Remettre l'ACK** après le test.

✅ **Idempotence validée** si un seul post Slack malgré le retry.

---

## Résumé des validations

| US | Critère | ✅ / ❌ |
|----|---------|--------|
| US3.1 | Logs `slack.message_received`, `runtime.runid_assigned`, `runtime.message_enqueued` | |
| US3.2 | `runtime.decision_computed` correct pour help/echo/autre | |
| US3.3 | `runtime.action_selected` + `slack.message_posted` + réponse Slack visible | |
| (opt) | Doublon → `duplicate_runid_dropped`, pas de double post | |

---

## Temps estimé

| Étape | Durée |
|-------|-------|
| Lancement runtime | 30s |
| Scénario 1 (US3.1) | 1 min |
| Scénario 2 (US3.2) | 2 min |
| Scénario 3 (US3.3) | 3 min |
| Scénario 4 (optionnel) | 3 min |
| **Total** | **< 10 min** |

---

## Troubleshooting

| Problème | Solution |
|----------|----------|
| `slack.socket.open_url` échoue | Vérifier `AppLevelToken` (Socket Mode activé sur l'app Slack) |
| `slack.message_posted` absent | Vérifier `BotToken` (scope `chat:write`) et app jointe au canal |
| `runtime.action_skipped channel_missing` | Le channel n'est pas remonté ; vérifier le payload Slack |
| `duplicate_runid_dropped` inattendu | RunId identique (même event_id ou channel:ts) ; comportement normal si retry Slack |

---

## Script PowerShell (scripts/smoke-epic3.ps1)

Voir le script réel dans le dépôt : `scripts/smoke-epic3.ps1`.

Le script utilise `scripts/run.ps1` avec l'environnement Local.

Usage :
```powershell
./scripts/smoke-epic3.ps1        # run once
./scripts/smoke-epic3.ps1 -Watch # hot reload
```
