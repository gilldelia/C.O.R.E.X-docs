# Configuration locale & secrets (Epic 0)

## Fichiers
- `src/Corex.App/appsettings.json` : base (aucun secret).
- `src/Corex.App/appsettings.Development.json` : overrides locaux non sensibles.
- `.env.example` : exemple d’env vars (copier vers `.env`, ne pas committer).
- User secrets activés sur `Corex.App` (`UserSecretsId` dans le csproj).

## Variables requises (fail-fast)
- `Slack:BotToken` (env `COREX__EXTERNALSERVICES__SLACK__BOTTOKEN` ou user-secrets)
- `Slack:SigningSecret` (env `COREX__EXTERNALSERVICES__SLACK__SIGNINGSECRET` ou user-secrets)
- `Trello:ApiKey` (env `COREX__EXTERNALSERVICES__TRELLO__APIKEY` ou user-secrets)
- `Trello:ApiToken` (env `COREX__EXTERNALSERVICES__TRELLO__APITOKEN` ou user-secrets)

## Chargement (ordre)
1. `appsettings.json` (obligatoire)
2. `appsettings.{Environment}.json` (optionnel)
3. Variables d’environnement (préfixe double underscore pour sections)
4. User secrets (optionnel, recommandé en local)

## Validation
Au démarrage, la config est validée ; si une valeur manque, le process s’arrête avec une liste d’erreurs explicites.

## Commandes utiles
- Build : `dotnet build C.O.R.E.X.sln`
- Tests : `dotnet test C.O.R.E.X.sln`
- Run local : `dotnet run --project src/Corex.App/Corex.App.csproj`
- Définir un secret : `dotnet user-secrets set "Slack:BotToken" "value" --project src/Corex.App/Corex.App.csproj`
