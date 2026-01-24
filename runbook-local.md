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
3. User secrets (`dotnet user-secrets`) scoped to `src/Corex.App/Corex.App.csproj`
4. Environment variables prefixed `COREX__` (overrides user secrets)

## Required secrets (fail fast at startup)
- `ExternalServices:Slack:BotToken`
- `ExternalServices:Slack:SigningSecret`
- `ExternalServices:Trello:ApiKey`
- `ExternalServices:Trello:ApiToken`

Startup exits with code `1` if required configuration (including these secrets) is missing or invalid (options validation failure).

### Environment variables (examples)
- `COREX__EXTERNALSERVICES__SLACK__BOTTOKEN`
- `COREX__EXTERNALSERVICES__SLACK__SIGNINGSECRET`
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
./scripts/run.ps1 -Dev -Watch

# Run once (debug)
./scripts/run.ps1 -Dev
```

## Verify
```powershell
./scripts/test.ps1
```
Ensures config validation and structured logging pipeline are healthy.

## Lint / format
```powershell
# Check formatting/style (no changes)
./scripts/lint.ps1

# Apply formatting
./scripts/format.ps1
```
