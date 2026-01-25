# COREX Runtime Invariants

Ce document liste les invariants stricts du runtime COREX. Toute modification doit être validée et documentée.

**Statut** : Verrouillés (fin Epic 2)

---

## 1. RunId — Unicité d'exécution

**Invariant** : Un `RunId` ne peut déclencher qu'une seule action métier et une seule réponse.

- Génération :
  - Externe (Slack) : `RunId.FromExternal(externalId)` → GUID déterministe (SHA-256).
  - Runtime : `RunId.NewRuntime(seed?)` → seedé par `event_id` ou nouveau GUID.
- Propagation : `run_id` présent dans tous les scopes de log dès l'entrée.
- Déduplication : `InMemoryRunIdDeduplicator` dans `RalphRuntime` ; duplicats logués (`duplicate_runid_dropped`).

**Testé par** : `RunAsync_Deduplicates_RunId`, `RunAsync_ConcurrentDuplicateRunIds_DeduplicatesCorrectly`, `RunAsync_MixedUniqueAndDuplicateRunIds_ProcessesOnlyUnique`.

---

## 2. Concurrence — Single-consumer

**Invariant** : Le traitement des événements est séquentiel (un seul worker).

- Modèle : `Channel<AgentInboundEvent>` bounded (1024 par défaut), single-reader.
- Conséquence : pas de locks côté handlers métier, ordre garanti.
- Extension : multi-consumer hors périmètre actuel (nécessite ADR).

**Testé par** : `ConsumeLoop_ProcessesEvents_Serially`.

---

## 3. Backpressure — DropWrite observable

**Invariant** : Les drops sont loggés, jamais silencieux.

- Mode : `BoundedChannelFullMode.Wait` + `TryWrite` (drop explicite si plein).
- Logs : `backpressure_drop` avec `events.dropped`, `rate_per_minute`, `Agent`, `EventId`, `RunId`.
- Compteur : `_droppedCount` incrémenté atomiquement à chaque drop.

**Testé par** : `RunAsync_LogsBackpressureDrops`, `EventHandler_LogsWarning_WhenChannelIsFull`.

---

## 4. Résilience Slack — Pas de crash

**Invariant** : Le runtime ne s'arrête jamais sur une déconnexion Slack.

- Reconnexion : `RunInboundWithReconnectAsync` avec backoff exponentiel (1s → 10s max).
- État dégradé : après 3 échecs consécutifs (`DegradedThreshold`), `runtime.degraded=true` loggé.
- Récupération : reset du compteur et log `runtime.degraded=false` à la reconnexion réussie.

**Testé par** : `RunAsync_NeverCrashes_OnRepeatedDisconnections`, `RunAsync_EntersDegradedState_AfterRepeatedFailures`, `RunAsync_ExitsDegradedState_OnRecovery`.

---

## 5. Traçabilité — IDs propagés

**Invariant** : Tous les logs d'événements contiennent les IDs de traçabilité.

- Scopes obligatoires : `correlation_id`, `causation_id`, `event_id`, `agent_id`, `agent_name`, `run_id`.
- Pas de catch silencieux : toute exception loguée avec contexte + IDs.

**Testé par** : `RunAsync_PushesRunIdIntoLogScope`, `ConsumeLoop_IncludesContextIds_InExceptionLog`, `AllEventLogs_ContainRunId`, `CorrelationCausation_PropagatedCorrectly`.

---

## 6. Idempotence — Actions rejouables

**Invariant** : Toute action déclenchée depuis un événement doit être idempotente.

- Conséquence : replay safe, dédup en mémoire ne bloque pas les retries légitimes post-restart.
- Responsabilité : handlers métier doivent être idempotents (hors runtime).
- Persistance : in-memory accepté ; cross-restart = idempotence handlers.

**Testé par** : tests d'intégration avec fakes.

---

## 7. Déduplication — EventId vs RunId

**Invariant** : La déduplication RunId (métier) est distincte de la déduplication EventId (technique).

| Concept | Localisation | Rôle |
|---------|--------------|------|
| `EventId` | `Corex.Core.Domain.InMemoryEventDeduplicator` | Éviter traitement technique d'un même message |
| `RunId` | `Corex.Runtime.InMemoryRunIdDeduplicator` | Garantir 1 action métier par unité de travail |

- **Plusieurs EventIds peuvent partager le même RunId** (ex. retry Slack avec même externalId).
- **Seul le premier événement avec un RunId donné déclenche une action.**

**Testé par** : `TryMarkProcessed_DuplicateCall_ReturnsFalse`, `TryMarkProcessed_ConcurrentCalls_OnlyOneReturnsTrue`.

---

## Récapitulatif des garanties

| # | Invariant | Verrouillé |
|---|-----------|------------|
| 1 | 1 RunId = 1 action = 1 réponse | ✅ |
| 2 | Single-consumer, ordre garanti | ✅ |
| 3 | Backpressure observable (jamais silencieux) | ✅ |
| 4 | Runtime jamais en crash sur déconnexion | ✅ |
| 5 | IDs propagés dans tous les logs | ✅ |
| 6 | Actions idempotentes (responsabilité handlers) | ✅ |
| 7 | EventId ≠ RunId (dédup séparées) | ✅ |

---

## Références

- ADR Runtime : `docs/adr/adr-sprint1-runtime.md`
- ADR Backpressure : `docs/adr/adr-epic2-backpressure.md`
- Architecture : `docs/architecture.md`
- Epic 1 : `docs/epics/epic-1-foundations.md`
- Epic 2 : `docs/epics/epic-2-foundations.md`
- Copilot instructions : `.github/copilot-instructions.md`
