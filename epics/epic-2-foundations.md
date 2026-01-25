# Epic 2 — Foundations (Robustesse Runtime)

## Objectif
Durcir le Runtime COREX (Ralph loop) pour garantir robustesse, backpressure maîtrisée, résilience I/O et traçabilité stricte, en faisant de ce document le contrat unique de l'Epic 2.

## Contexte
Epic 1 a livré le Runtime, la modélisation/propagation du RunId, un modèle de concurrence single-consumer et une résilience Socket Mode simulée. Epic 2 vise à renforcer la robustesse (reconnexions, backpressure, idempotence, observabilité) sans implémenter de nouvelles fonctionnalités métier.

## Périmètre (IN)
- Invariants Runtime : RunId présent et propagé, idempotence/dédup, logs corrélés.
- Backpressure et gestion de charge : canal/queue bornée, comportements en saturation (drop/log/retry) définis.
- Résilience I/O Slack (Socket Mode) : reconnexions, backoff contrôlé, pas de crash de la boucle.
- Traçabilité renforcée : scopes de log complets (`run_id`, `event_id`, `correlation_id`, `causation_id`, `agent_id`).
- Tests de robustesse (unit/integration fakes) couvrant reconnection, saturation, cancellation, exceptions contrôlées.
- Documentation/contrat : ce document comme source de vérité pour toutes les US 2.x.

## Hors périmètre (OUT)
- Fonctionnalités métier nouvelles ou UI/Slack utilisateur/Trello utilisateur.
- Déploiement cloud ou infra managée (reste local-first).
- Modèle multi-agent avancé/LLM prod (Epic ultérieur).
- Modifications breaking sur le domaine métier hors Runtime/observabilité.

## Décisions d'architecture (règles)
- Contrat d'architecture : Copilot et toutes les US 2.x doivent se conformer à ce document; toute entorse nécessite mise à jour + validation préalable.
- RunId invariant : généré dès l'entrée, propagé dans tous les logs/messages; un RunId = au plus une action métier et une réponse.
- Idempotence/dédup : chaque événement est dédupliqué (`event_id`), actions rejouables sans effet de bord.
- Concurrence : traitement single-consumer (ou modèle choisi explicitement) avec queue bornée; aucune exécution concurrente non cadrée.
- Backpressure : canal/queue bornée, en cas de saturation -> log Warning + stratégie définie (pas de crash silencieux).
- Résilience I/O : retry + backoff sur Slack Socket Mode simulé; aucune exception d'agent/I-O ne doit arrêter l'orchestrateur.
- Observabilité : logs structurés avec `run_id`, `event_id`, `correlation_id`, `causation_id`, `agent_id`, `agent_name`; aucun catch silencieux.
- Tests obligatoires : toute évolution Runtime doit inclure tests unitaires + intégration (fakes) pour reconnection, saturation, cancellation, exceptions.

## Livrables attendus
- Document `docs/epics/epic-2-foundations.md` versionné et relu.
- Référencement par les US 2.x comme contrat unique.
- (Pas de code fonctionnel dans cette US.)

## User Stories (scope Epic 2)
- US2.x : Toute US du sprint doit référencer ce document et démontrer conformité (à détailler au fil des US 2.x).

## Critères de succès
- Le document est la source de vérité pour Copilot et l'équipe sur l'Epic 2.
- Les invariants Runtime (RunId, idempotence, backpressure, résilience, traçabilité) sont explicites et opposables.
- Toute évolution Epic 2 se réfère à ce contrat et refuse les dérives de périmètre sans mise à jour préalable.

## Écarts acceptés (si nécessaires)
- Slack Socket Mode peut rester simulé (fakes) tant qu'aucune infra réseau stable n'est disponible.
- Pas de persistance durable obligatoire (in-memory accepté) si la robustesse runtime est couverte par tests.

## Checklist fin d'Epic
- [x] Document relu/validé et utilisé par toutes les US 2.x
- [x] Invariants Runtime explicités et respectés (RunId, idempotence, backpressure, résilience, traçabilité)
- [x] Tests de robustesse (reconnect, saturation, cancellation, exceptions) en place et verts
- [x] Docs à jour (contrat + runbook si impact)
- [x] `docs/architecture.md` mis à jour (post-Epic 2)
- [x] `docs/invariants.md` mis à jour avec clarification EventId vs RunId

## Revue clôturée

**Date** : Epic 2 terminé  
**Statut** : ✅ Validé (76 tests passants, invariants verrouillés)  
**Revue architecture** : Validée avec réserves mineures (dettes documentées)

### Invariants verrouillés
- 1 RunId = 1 action = 1 réponse
- Single-consumer, traitement séquentiel
- Backpressure observable (jamais silencieux)
- Runtime jamais en crash sur déconnexion Slack
- IDs propagés dans tous les logs
- EventId ≠ RunId (dédup séparées)

### Dettes acceptées (Epic suivant)
- Persistance dédup cross-restart : in-memory OK, idempotence handlers requise
- Métriques non exposées externement (compteurs internes uniquement)
- Cleanup dedup non testé en stress

**Prochaine étape** : Epic 3 — Intégration Slack/Trello réelle

