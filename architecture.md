# COREX Architecture (Post-Epic 2)

## Solution layout

| Projet | Responsabilité | Dépendances |
|--------|----------------|-------------|
| `src/Corex.Core` | Domaine (Agent, RunId, InboundEnvelope, ports) | Aucune externe |
| `src/Corex.Runtime` | Orchestrateur Ralph Loop, déduplication RunId, résilience | Core |
| `src/Corex.Infrastructure` | Observabilité, résilience (Polly), clock | Core |
| `src/Corex.Adapters` | Implémentations I/O (fakes Slack/LLM/Trello) | Core |
| `src/Corex.App` | Host applicatif, configuration, bootstrap | Core, Runtime, Infrastructure, Adapters |
| `tests/*` | Tests unitaires et intégration (xUnit) | Projets ciblés |

## Flux d'exécution (Ralph Loop)

```
[Inbound Source] → Event → [Channel bounded] → [ConsumeLoop] → [Dedup RunId] → [Handler]
       ↑                         ↓ (si plein)
   Reconnect              backpressure_drop (log)
   + backoff
```

## Invariants verrouillés (Epic 2)

| Invariant | Garantie | Composant |
|-----------|----------|----------|
| 1 RunId = 1 action | Dédup stricte | `InMemoryRunIdDeduplicator` |
| Single-consumer | Traitement séquentiel | `Channel<T>` SingleReader |
| Backpressure observable | Logs + compteurs | `LogDrop()` |
| Résilience Slack | Reconnect + backoff + degraded state | `RunInboundWithReconnectAsync` |
| Traçabilité | IDs dans tous les scopes | `run_id`, `correlation_id`, etc. |

## Déduplication : EventId vs RunId

| Concept | Localisation | Usage |
|---------|--------------|-------|
| `EventId` | Core (`InMemoryEventDeduplicator`) | Identifiant unique de message (dédup technique) |
| `RunId` | Runtime (`InMemoryRunIdDeduplicator`) | Unité de travail logique (dédup métier) |

**Règle** : plusieurs événements peuvent partager le même `RunId` (ex. retry Slack). Seul le premier déclenche une action.

## Dépendances

```
Corex.App
    ├── Corex.Runtime → Corex.Core
    ├── Corex.Infrastructure → Corex.Core
    └── Corex.Adapters → Corex.Core
```

- **Corex.Core** : aucun SDK externe (Slack/Trello/HTTP interdit).
- **Corex.Adapters** : I/O uniquement, aucune logique métier.
- **Corex.Runtime** : orchestration, dédup, résilience.

## Commandes locales

| Action | Commande |
|--------|----------|
| Build | `./scripts/build.ps1` |
| Tests | `./scripts/test.ps1` |
| Run (dev) | `./scripts/run.ps1 -Dev` |
| Run (watch) | `./scripts/run.ps1 -Dev -Watch` |
| Lint | `./scripts/lint.ps1` |
| Format | `./scripts/format.ps1` |

## État actuel (fin Epic 2)

- ✅ Runtime robuste avec dédup, backpressure, résilience
- ✅ 76 tests passants
- ✅ Invariants documentés et testés
- ⏳ Slack/Trello : fakes uniquement (intégration réelle = Epic suivant)
- ⏳ Persistance dédup : in-memory (cross-restart = idempotence handlers)

## Références

- Invariants : `docs/invariants.md`
- ADR Runtime : `docs/adr/adr-sprint1-runtime.md`
- ADR Backpressure : `docs/adr/adr-epic2-backpressure.md`
- Epic 0 : `docs/epics/epic-0-foundations.md`
- Epic 1 : `docs/epics/epic-1-foundations.md`
- Epic 2 : `docs/epics/epic-2-foundations.md`
