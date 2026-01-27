# Configuration locale & secrets (Epic 0)

## Fichiers
- `src/Corex.App/appsettings.json` : base (aucun secret).
- `src/Corex.App/appsettings.Development.json` : overrides locaux non sensibles (activé si `DOTNET_ENVIRONMENT=Development`).
- `src/Corex.App/appsettings.local.json` : overrides/secrets locaux (toujours chargés si présents, quelle que soit l’environnement; **note: lowercase 'local'**), préférer user-secrets/env pour les secrets car ils ont une priorité plus élevée.
- `.env.example` : exemple d’env vars (copier vers `.env`, ne pas committer).
- User secrets activés sur `Corex.App` (`UserSecretsId` dans le csproj).

## Variables requises (fail-fast quand activé)
- Slack (activé par défaut) :
  - `ExternalServices:Slack:BotToken` (env `COREX__EXTERNALSERVICES__SLACK__BOTTOKEN` ou user-secrets)
  - `ExternalServices:Slack:AppLevelToken` (env `COREX__EXTERNALSERVICES__SLACK__APPLEVELTOKEN` ou user-secrets) — requis pour Socket Mode
  - `ExternalServices:Slack:SigningSecret` (env `COREX__EXTERNALSERVICES__SLACK__SIGNINGSECRET` ou user-secrets)
- Trello (désactivé par défaut, à activer pour les US qui l'utilisent) :
  - `ExternalServices:Trello:ApiKey` (env `COREX__EXTERNALSERVICES__TRELLO__APIKEY` ou user-secrets)
  - `ExternalServices:Trello:ApiToken` (env `COREX__EXTERNALSERVICES__TRELLO__APITOKEN` ou user-secrets)
  - `ExternalServices:Trello:Enabled=true` pour déclencher la validation

## Chargement (ordre)
1. `appsettings.json` (obligatoire)
2. `appsettings.{Environment}.json` (optionnel)
3. `appsettings.local.json` (optionnel, toujours chargé si présent)
4. User secrets (optionnel, recommandé en local)
5. Variables d'environnement (préfixe double underscore pour sections — priorité la plus élevée)

## Validation
Au démarrage, la config est validée. Les services externes ne sont contrôlés que s'ils sont marqués `Enabled=true` (Slack est activé par défaut, Trello ne l'est pas). Si une valeur manque pour un service activé, le process s’arrête avec une liste d’erreurs explicites.

## Commandes utiles
- Build : `dotnet build C.O.R.E.X.sln`
- Tests : `dotnet test C.O.R.E.X.sln`
- Run local : `dotnet run --project src/Corex.App/Corex.App.csproj`
- Définir un secret : `dotnet user-secrets set "ExternalServices:Slack:BotToken" "value" --project src/Corex.App/Corex.App.csproj`
