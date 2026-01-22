# COREX Architecture Skeleton (Epic 0)

## Solution layout
- `src/Corex.Core` : domaine (IDs, entités, ports). Aucun I/O externe.
- `src/Corex.Infrastructure` : composants techniques transverses (observabilité, clocks, policies) dépendant uniquement du Core.
- `src/Corex.Adapters` : implémentations I/O (fakes locales pour l’instant) dépendant du Core.
- `src/Corex.App` : point d’entrée console pour la boucle orchestrateur Ralph (squelette local-first).
- `tests/Corex.Core.Tests` : tests unitaires (xUnit) sur le domaine.

## Dépendances attendues
- `Corex.Core` : aucune dépendance externe.
- `Corex.Infrastructure` → `Corex.Core`.
- `Corex.Adapters` → `Corex.Core` (pas de logique métier, uniquement I/O / fakes).
- `Corex.App` → `Corex.Core`, `Corex.Infrastructure`, `Corex.Adapters`.
- Tests → projets ciblés uniquement.

## Commandes locales de référence
- Build : `dotnet build C.O.R.E.X.sln`
- Tests : `dotnet test C.O.R.E.X.sln`
- Run (squelette) : `dotnet run --project src/Corex.App/Corex.App.csproj`

## Notes Epic 0
- Local-first : aucune intégration Slack/Trello réelle (fakes dans `Adapters`).
- Observabilité : contexte de corrélation présent dans `Infrastructure` (IDs corrélation/causation/agent/run/event).
- Idempotence / traçabilité : à conserver dans chaque future action déclenchée par événement.
