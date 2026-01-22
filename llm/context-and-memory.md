# Contexte et mémoire

## Fenêtre de contexte
- Priorité aux données récentes et spécifiques à l'événement courant.
- Limiter le hors-sujet : ne charger que ce qui est nécessaire au run.
- Si espace contraint : résumer avant d'injecter.

## Résumés
- Résumés factuels, horodatés, sans interprétation.
- Inclure les IDs propagés (`correlation_id`, `causation_id`, `event_id`, `agent_id`, `run_id`).

## RAG (Retrieval Augmented Generation)
- Requêtes ciblées et versionnées ; inclure la source et le timestamp.
- Filtrer les résultats (pertinence, fraîcheur) avant injection.

## Politique de propagation
- Propager les IDs dans chaque message outil / sortie.
- Noter les décisions clés et garde-fous appliqués (idempotence, déduplication).

## Données sensibles
- Ne jamais injecter de secrets/jetons.
- Rédiger ou masquer les données PII avant usage.
