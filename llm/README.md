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
