# Évaluations et non-régression

## Objectifs
- Mesurer la qualité des prompts/agents.
- Prévenir les régressions (boucles, doublons, pertes d'IDs).
- Vérifier l'adhérence au format et aux garde-fous.

## Types de tests
- Unitaires de prompt : conformité de format, présence des IDs, réponses déterministes.
- Intégration : fakes Slack/LLM, scénarios de déduplication et d'idempotence.
- Régression : jeux de cas versionnés avec attentes figées.

## Métriques clés
- Taux de conformité de format (JSON/Markdown attendu).
- Taux d'échec contrôlé (erreurs signalées proprement).
- Absence de doublons (dédup par `event_id`), propagation des IDs.
- Latence moyenne et respect des timeouts.

## Procédure
- Versionner les datasets d'éval (fixtures).
- Exécuter les tests en CI à chaque changement de prompt/agent.
- Documenter les garde-fous activés et les mitigations en cas de risque identifié.
