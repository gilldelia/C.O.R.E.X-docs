# Usage des outils et MCP

## Principes
- Déclarer les outils autorisés et leurs limites (quotas, timeouts).
- Toujours passer les IDs (`correlation_id`, `causation_id`, `event_id`) aux appels d’outils si supporté.
- Pas d’appels implicites : l’outil doit être explicitement choisi et paramétré.

## Sélection d’outil
- Choisir l’outil le plus spécifique (éviter le générique si un spécialisé existe).
- Vérifier les préconditions (données requises, formats, droits).

## Exécution
- Timeouts courts et explicites ; réessais bornés avec backoff.
- Validation des sorties : schéma attendu, champs obligatoires, cohérence métier.
- En cas d’échec : log structuré, pas de boucle infinie, proposer une alternative ou demander clarification.

## Sécurité
- Ne jamais transmettre de secrets/PII.
- Filtrer/valider les entrées utilisateur avant appel.

## Traçabilité
- Log de chaque appel outil avec paramètres clés (sans secrets) et IDs.
