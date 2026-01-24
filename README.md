# llm

# COREX - LLM

## Objectifs
- Encadrer l’usage des LLM dans COREX (agents, orchestrateur, outils).
- Garantir fiabilité, traçabilité et testabilité sans dépendre d’APIs externes.
- Prévenir les boucles, doublons et réponses multiples sur un même événement.

## Périmètre
- Règles de prompting et de formatage.
- Gestion du contexte et de la mémoire.
- Patterns agentiques et usage des outils (MCP/externes).
- Stratégies d’évaluation et de non-régression.

## Conventions
- Toujours propager `correlation_id`, `causation_id`, `event_id`.
- Actions déclenchées par événement : idempotentes, traçables, dédupliquables.
- Logs structurés (JSON) avec les IDs ci-dessus et `agent_id`, `run_id`.
- Aucun « catch » silencieux ; les erreurs sont loguées avec contexte.

## Plan des documents
- `prompting.md` : règles de prompt.
- `context-and-memory.md` : gestion du contexte, résumés, RAG.
- `agentic-patterns.md` : patterns d’orchestration et de boucle.
- `tool-use-and-mcp.md` : appels d’outils, garde-fous, timeouts.
- `evals.md` : stratégies d’évaluation, datasets, seuils.




# Racine

# COREX

## Quickstart (5 minutes)
1. Prérequis : SDK .NET 10.0 en préversion (installer la dernière préversion depuis https://dotnet.microsoft.com/).
2. Restore : `dotnet restore C.O.R.E.X.sln` (ou `./scripts/test.ps1` fera le restore).
3. Run (dev, logs verbeux) : `./scripts/run.ps1 -Dev`.
4. Hot reload (dev) : `./scripts/run.ps1 -Dev -Watch`.
5. Tests : `./scripts/test.ps1`.
6. Format : `./scripts/format.ps1` (lint sans modifier : `./scripts/lint.ps1`).

Notes
- Secrets Slack/Trello requis pour un run complet ; fakes par défaut.
- Logs en dev au niveau `Debug` via `-Dev`.
- Sortie structurée : console + `logs/corex.log`.

## Dev loop
- Boucle rapide : `./scripts/run.ps1 -Dev -Watch`, puis `./scripts/test.ps1` avant commit.
- Lint check : `./scripts/lint.ps1` (échoue si format/style non conforme).

## Docs utiles
- `docs/runbook-local.md` : config/secrets/logs.
- `docs/epics/epic-0-foundations.md` : règles Epic 0.
- `.github/copilot-instructions.md` : invariants.




