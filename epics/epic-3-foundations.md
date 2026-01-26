# Epic 3 — Foundations (Functional Enablement)

## Objectif
Introduire les premières capacités fonctionnelles de COREX
sur une base runtime robuste validée (Epic 2).

## Périmètre
- Fonctionnel minimal mais réel
- Orientation usage / valeur
- Validation PO intégrée

## Hors périmètre
- Scaling distribué
- Optimisations perf avancées
- Multi-runtime / HA

## Invariants hérités (Epic 2)
- RunId = 1 action
- Déduplication obligatoire
- Backpressure non bloquante
- Runtime non crashant
- Observabilité systématique

## Nouvelles attentes Epic 3
- Chaque feature doit répondre à un usage concret
- Les comportements doivent être vérifiables sans debugger
- Les logs sont des artefacts fonctionnels

## Flux fonctionnels cibles
- (à compléter par Epic 3)
- Entrée → Décision → Action → Trace observable

## Règles de validation
- Smoke tests fonctionnels obligatoires
- Revue PO en fin d’Epic
- Aucune US “tech-only” sans justification fonctionnelle

## Risques identifiés
- Sur-ingénierie
- Fonctionnel trop abstrait
- Couplage excessif Runtime / Feature

## Stratégie de mitigation
- US petites
- Démo fréquente
- Feedback PO continu
