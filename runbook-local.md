# COREX Runbook Local – Configuration, Secrets, Logs

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

## Run locally
```bash
DOTNET_ENVIRONMENT=Development dotnet run --project src/Corex.App/Corex.App.csproj
```

## Verify
```bash
dotnet test
```
Ensures config validation and structured logging pipeline are healthy.
