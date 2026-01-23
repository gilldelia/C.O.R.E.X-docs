# COREX Runbook Local – Configuration & Secrets

## Configuration sources (precedence)
_Ordered from lowest to highest precedence (later sources override earlier ones)._
1. `appsettings.json` (required, checked in, no secrets)
2. `appsettings.{ENVIRONMENT}.json` (optional override, e.g. Development)
3. Environment variables prefixed `COREX__`
4. User secrets (`dotnet user-secrets`) scoped to `src/Corex.App/Corex.App.csproj`

## Required secrets (fail fast at startup)
- `ExternalServices:Slack:BotToken`
- `ExternalServices:Slack:SigningSecret`
- `ExternalServices:Trello:ApiKey`
- `ExternalServices:Trello:ApiToken`

If any is missing/empty, the app exits with code `1` and logs the missing keys.

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
- Fill values; they map to the same keys via the `COREX__` prefix.
- Export to your shell before running (PowerShell example):
```powershell
Get-Content .env | ForEach-Object {
  if ($_ -and $_ -notmatch '^#') {
    $name,$value = $_ -split '=',2; [Environment]::SetEnvironmentVariable($name,$value)
  }
}
```

## Run locally
```bash
DOTNET_ENVIRONMENT=Development dotnet run --project src/Corex.App/Corex.App.csproj
```

## Verify
```bash
dotnet test
```
This ensures config validation and tests are healthy after changes.
