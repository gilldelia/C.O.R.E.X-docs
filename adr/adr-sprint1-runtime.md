# ADR Sprint 1 — Runtime, RunId, Concurrence, Résilience

## Contexte
Epic 1 a introduit le runtime (Ralph loop), la modélisation du RunId, la propagation des IDs d’observabilité, un modèle de concurrence single-consumer, et la résilience Slack (Socket Mode simulée). Objectif : garantir idempotence, dédup, traçabilité et robustesse locale-first.

## Décisions
1) RunId (Domain VO)
- RunId = Value Object (source Runtime/External).
- Génération :
  - Externe (ex. message Slack): `RunId.FromExternal(externalId)` -> GUID déterministe (hash SHA-256). 
  - Runtime: `RunId.NewRuntime(seed?)`, seedé par `event_id` pour stabilité intra-event.
- RunId présent dès l’entrée (`AgentInboundEvent`, `InboundEnvelope`), propagé dans logs/scopes (`run_id`).
- Raison: traçabilité et idempotence par exécution; éviter multiples réponses par RunId.

2) Concurrence Ralph
- Modèle: single-consumer sur `Channel<AgentInboundEvent>` (bounded 1024), inbound multi-writer possible.
- Traitement séquentiel (un worker) -> pas de locks côté handlers métier.
- Backpressure: `TryWrite` + warning si plein; dédup/idempotence restent requis.
- Raison: simplicité, ordre garanti, sûreté contre double exécution.

3) Résilience Slack (Socket Mode simulée)
- Inbound démarré via `RunInboundWithReconnectAsync`: retry avec backoff exponentiel (1s -> 10s) en cas d’échec.
- Exceptions I/O loguées (Warning) sans crasher le runtime; consumer avale cancellation proprement.
- Raison: maintenir la boucle vivante malgré coupures; respecter invariant "le runtime ne doit jamais perdre la connexion Slack suite à une exception d’agent".

4) Observabilité
- Scopes de log enrichis systématiquement: `correlation_id`, `causation_id`, `event_id`, `agent_id`, `agent_name`, `run_id`.
- Exceptions loguées avec IDs; pas de catch silencieux.

## Alternatives considérées
- Multi-consumer concurrent: rejeté pour Epic 1 (complexité de synchronisation, ordre non garanti). Évolutif plus tard via config.
- RunId aléatoire uniquement runtime: rejeté (perte de déterminisme externe et de lien avec messageId).

## Mises à jour Epic 2

### Déduplication RunId (Sprint 2.1)
- `InMemoryRunIdDeduplicator` ajouté dans `Corex.Runtime`.
- Un RunId ne peut déclencher qu'une seule action métier (invariant strict).
- Duplicats loggés avec `duplicate_runid_dropped` et compteur incrémenté.
- Dédup appliquée dans la boucle consume de `RalphRuntime`, avant tout traitement.

### Backpressure observable (Sprint 2.2)
- Compteur `events.dropped` et taux par minute (`rate_per_minute`) loggés sur chaque drop.
- Log structuré `backpressure_drop` avec Agent, EventId, RunId.
- Aucun drop silencieux ; DropWrite conservé, single-consumer préservé.

### État dégradé (Sprint 2.3)
- Seuil de 3 échecs consécutifs inbound avant entrée en mode `degraded`.
- Log `runtime.degraded=true` à l'entrée, `runtime.degraded=false` à la sortie.
- Runtime jamais en crash sur Slack down ; backoff max 10s.

## Impacts
- Tests ajoutés/ajustés: RunId déterministe, propagation scopes, backoff/retry inbound, ordre séquentiel, backpressure, dédup RunId, état dégradé.
- CI exécute format + tests; runtime reste local-first, sans dépendance cloud.

## Liens
- Epic 1 Foundations: `docs/epics/epic-1-foundations.md`
- Epic 2 Foundations: `docs/epics/epic-2-foundations.md`
- Code: `Corex.Runtime` (RalphRuntime, InMemoryRunIdDeduplicator), `Corex.Core.Domain` (RunId, AgentInboundEvent, InboundEnvelope), tests runtime/core.
