# Patterns agentiques

## Boucle d’orchestration
- Chaque événement déclenche au plus une exécution métier (dédup via `event_id`).
- Les actions doivent être rejouables (idempotence + état externe stable).
- En cas d’erreur : log structuré avec IDs, pas de boucle infinie.

## Planning / découpage
- Décomposer en étapes courtes et observables.
- Valider les préconditions avant appel outil (ex. entrée non nulle, format conforme).

## Garde-fous
- Timeouts explicites par appel outil.
- Retriers bornés avec backoff, jamais infinis.
- Arrêt anticipé si mêmes entrées produisent déjà un résultat confirmé (cache/dédup).

## Tests
- Prévoir fakes/mocks pour Slack/LLM ; aucun appel réseau réel requis.
- Couvrir : routage, déduplication, transitions d’état, propagation des IDs.

## Journalisation
- Logs JSON avec `correlation_id`, `causation_id`, `event_id`, `agent_id`, `run_id`.
- Aucun « catch » silencieux ; surface les erreurs avec contexte.
