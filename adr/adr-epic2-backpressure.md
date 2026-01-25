# ADR Epic 2 — Backpressure Observable et Maîtrisée

## Contexte
Epic 2 vise la robustesse du runtime Ralph. Le canal d’événements est en DropWrite (bounded, single-consumer). Avant cette ADR, les drops n’étaient pas métrologiés ni clairement logués.

## Décision
- Conserver le modèle single-consumer et le mode DropWrite sur le channel runtime.
- Ajouter un compteur in-memory `events.dropped` incrémenté à chaque drop.
- Journaliser chaque drop en Warning avec message structuré `backpressure_drop` incluant :
  - events.dropped (compteur cumulatif),
  - rate_per_minute (calculé depuis le premier drop),
  - run_id, event_id, agent_name.
- Pas de drop silencieux : tout drop produit un log exploitable.
- Aucun changement de modèle concurrent ni de capacité par défaut (1024), mais capacité overridable pour les tests.

## Rationale
- Assurer la traçabilité et l’observabilité de la backpressure sans introduire de threads supplémentaires.
- Le compteur/rate permet de détecter les phases de saturation et d’ajuster en conséquence.
- Local-first, pas de persistance : compteur en mémoire suffisant pour Epic 2.

## Conséquences
- En cas de saturation, les événements sont droppés mais toujours logués ; le runtime ne crash pas.
- Les tests peuvent réduire la capacité pour forcer les drops et vérifier les logs.

## Alternatives écartées
- Passage en DropNewest/Wait : rejeté (modèle explicite = DropWrite).
- Multi-consumer : hors périmètre Epic 2.
- Persistance des drops : rejeté (local-first, coût inutile pour ce sprint).

## Vérification / Tests
- Test d’intégration : canal capacité réduite, flood → logs Warning `backpressure_drop` avec compteur et rate.

## Statut
Accepté (Epic 2, Sprint 2.2).
