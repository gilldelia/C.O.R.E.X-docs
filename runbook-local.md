# COREX Runbook Local – Configuration, Secrets, Logs

## Setup checklist (local)
- SDK : .NET 10 (preview acceptable)
- Commandes : `dotnet --info` OK, PowerShell disponible
- Restore : `dotnet restore C.O.R.E.X.sln` (ou `./scripts/test.ps1`)
- Secrets : renseignés (Slack/Trello) via user-secrets ou env (voir ci-dessous)
- Logs : dossier `logs/` accessible en écriture
- Hot reload : `./scripts/run.ps1 -Dev -Watch` (optionnel)

## Configuration sources (precedence)
_Ordered from lowest to highest precedence (later sources override earlier ones)._
1. `appsettings.json` (required, checked in, no secrets)
2. `appsettings.{ENVIRONMENT}.json` (optional override, e.g. Development)
3. `appsettings.local.json` (optional, always loaded if present, regardless of environment; **note: lowercase 'local'** — casing matters on case-sensitive filesystems)
4. User secrets (`dotnet user-secrets`) scoped to `src/Corex.App/Corex.App.csproj`
5. Environment variables prefixed `COREX__` (highest precedence — overrides all file-based sources and user secrets)

## Required secrets (fail fast per service)
- Slack (enabled by default)
  - `ExternalServices:Slack:BotToken`
  - `ExternalServices:Slack:AppLevelToken` (Socket Mode)
  - `ExternalServices:Slack:SigningSecret`
- Trello (disabled by default; set `ExternalServices:Trello:Enabled=true` when a Trello feature is active)
  - `ExternalServices:Trello:ApiKey`
  - `ExternalServices:Trello:ApiToken`

Startup exits with code `2` if required configuration for any enabled service is missing or invalid (options validation failure).

### Environment variables (examples)
- `COREX__EXTERNALSERVICES__SLACK__BOTTOKEN`
- `COREX__EXTERNALSERVICES__SLACK__APPLEVELTOKEN`
- `COREX__EXTERNALSERVICES__SLACK__SIGNINGSECRET`
- `COREX__EXTERNALSERVICES__TRELLO__ENABLED=true`
- `COREX__EXTERNALSERVICES__TRELLO__APIKEY`
- `COREX__EXTERNALSERVICES__TRELLO__APITOKEN`
- `DOTNET_ENVIRONMENT=Development` (dev mode)

## Set secrets locally (recommended)
```bash
# from repo root
DOTNET_ENVIRONMENT=Development dotnet user-secrets init --project src/Corex.App/Corex.App.csproj
DOTNET_ENVIRONMENT=Development dotnet user-secrets set "ExternalServices:Slack:BotToken" "xoxb-your-token" --project src/Corex.App/Corex.App.csproj
DOTNET_ENVIRONMENT=Development dotnet user-secrets set "ExternalServices:Slack:SigningSecret" "signing-secret" --project src/Corex.App/Corex.App.csproj
DOTNET_ENVIRONMENT=Development dotnet user-secrets set "ExternalServices:Trello:ApiKey" "trello-key" --project src/Corex.App/Corex.App.csproj
DOTNET_ENVIRONMENT=Development dotnet user-secrets set "ExternalServices:Trello:ApiToken" "trello-token" --project src/Corex.App/Corex.App.csproj
```

## Using `.env` (optional)
- Copy `.env.example` to `.env` at repo root.
- Fill values for `COREX__EXTERNALSERVICES__*` keys.
- Note: Environment variables override user-secrets due to configuration source precedence.
- Export them to your shell before running. PowerShell example:
```powershell
Get-Content .env | ForEach-Object {
  if ($_ -and $_ -notmatch '^#') {
    $name,$value = $_ -split '=',2; [Environment]::SetEnvironmentVariable($name,$value)
  }
}
```

## Logging (Serilog)
- Structured logs with correlation in scope: `correlation_id`, `agent_id`, `event_id`, plus `environment`/`application`.
- Sinks:
  - Console (structured template)
  - File `logs/corex.log` (rolling daily)
- Minimum level is `Information` by default; Development environment overrides to `Debug` via `appsettings.Development.json`.

## Debug / diagnostic scenarios
- Voir en direct : `./scripts/run.ps1 -Dev` (console, niveau Debug) ou `-Watch` pour hot reload.
- Fichiers de logs : `logs/corex.log` (rolling daily). Supprimables si trop volumineux.
- Ports : application console, pas d’écoute HTTP dans l’état actuel (fakes Slack/Trello). Aucun port réservé.
- Exceptions globales : handlers AppDomain/TaskScheduler, exit codes (0 ok, 1 erreur, 2 config, 130 annulation). Vérifier la console ou le fichier de logs.
- Resilience I/O : retry + circuit breaker (Polly) enregistrés. Une indispo I/O ne stoppe pas le process, vérifier les warnings dans les logs.

## Run locally
```powershell
# Dev logs + hot reload
DOTNET_ENVIRONMENT=Local dotnet watch run --project src/Corex.App/Corex.App.csproj -c Debug

# Run once (debug)
DOTNET_ENVIRONMENT=Local dotnet run --project src/Corex.App/Corex.App.csproj -c Debug
```

## Verify
```powershell
./scripts/test.ps1
```
Ensures config validation and structured logging pipeline are healthy.

## Comment tester US3.1 (Slack Socket Mode → Runtime)
- Pré-requis :
  - Variables d'environnement Slack renseignées (tokens Socket Mode) via user-secrets ou env (`ExternalServices:Slack:*`). Aucun secret dans le repo.
  - Canal Slack de test (ex. `#corex-test`).
- Étapes manuelles (2-5 min) :
 1) Lancer le runtime local (`DOTNET_ENVIRONMENT=Local dotnet run --project src/Corex.App/Corex.App.csproj -c Debug`).
  2) Envoyer un message "ping 1" sur le canal de test.
  3) Vérifier les logs structurés :
     - `slack.message_received` (channel, ts, user, text)
     - `runtime.runid_assigned` (run_id dérivé de l'event Slack)
     - `runtime.message_enqueued` (run_id, event_id)
  4) Renvoyer "ping 1" : pas de dédup sur le contenu (le contenu seul ne déduplique pas).
  5) Si vous rejouez exactement le même event Slack (même `event_id`, ou même couple channel+ts si `event_id` absent), vérifier un `duplicate_runid_dropped` (dédup RunId avant traitement).
- Notes :
  - Le RunId est dérivé en priorité de `event_id` Slack (si présent), avec repli sur le couple `channel:ts`, de façon déterministe via `RunId.FromExternal`.
  - Aucune logique métier dans l'adapter ; il publie uniquement dans la boucle runtime.

## Lint / format
```powershell
# Check formatting/style (no changes)
./scripts/lint.ps1

# Apply formatting
./scripts/format.ps1
```
