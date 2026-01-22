# Règles de prompting

## Principes
- But explicite : formuler l'intention, les entrées, les sorties attendues.
- Format strict : préciser le schéma (JSON/Markdown) et le respecter.
- Briefer les contraintes critiques : idempotence, déduplication, logs avec IDs.
- Interdire : hallucinations de faits non fournis, appels d'API implicites, changement de format sans accord.

## Structure recommandée
1) Contexte minimal + IDs (`correlation_id`, `causation_id`, `event_id`).
2) Tâche à accomplir (objectif mesurable).
3) Contraintes (format, champs obligatoires, timeouts, politique d'erreur).
4) Outils autorisés et limites (budget, retries).
5) Exemples positifs/négatifs si disponibles.

## Ton et style
- Réponses brèves, déterministes, sans spéculation.
- Préférer listes et tableaux pour la clarté.
- Pas de verbosité inutile ; pas de figures de style.

## Gestion des erreurs
- Demander précision en cas d'ambiguïté.
- Signaler explicitement les données manquantes ou incohérentes.
