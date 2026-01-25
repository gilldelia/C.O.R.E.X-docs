# COREX Runtime Invariants

Ce document liste les invariants stricts du runtime COREX. Toute modification doit être validée et documentée.

## 1. RunId — Unicité d'exécution

**Invariant** : Un `RunId` ne peut déclencher qu'une seule action métier et une seule réponse.

- Génération :
  - Externe (Slack) : `RunId.FromExternal(externalId)` → GUID déterministe (SHA-256).
  - Runtime : `RunId.NewRuntime(seed?)` → seedé par `event_id` ou nouveau GUID.
- Propagation : `run_id` présent dans tous les scopes de log dès l'entrée.
- Déduplication : `InMemoryRunIdDeduplicator` dans `RalphRuntime` ; duplicats logués (`duplicate_runid_dropped`).

**Testé par** : `RunAsync_Deduplicates_RunId`, `RunAsync_ConcurrentDuplicateRunIds_DeduplicatesCorrectly`.

## 2. Concurrence — Single-consumer

**Invariant** : Le traitement des événements est séquentiel (un seul worker).

- Modèle : `Channel<AgentInboundEvent>` bounded (1024), single-reader.
- Conséquence : pas de locks côté handlers métier, ordre garanti.
- Extension : multi-consumer hors périmètre actuel.

**Testé par** : `ConsumeLoop_ProcessesEvents_Serially`.

## 3. Backpressure — DropWrite observable

**Invariant** : Les drops sont loggés, jamais silencieux.

- Mode : `BoundedChannelFullMode.DropWrite`.
- Logs : `backpressure_drop` avec `events.dropped`, `rate_per_minute`, `Agent`, `EventId`, `RunId`.
- Compteur : `_droppedCount` incrémenté à chaque drop.

**Testé par** : `RunAsync_LogsBackpressureDrops`, `EventHandler_LogsWarning_WhenChannelIsFull`.

## 4. Résilience Slack — Pas de crash

**Invariant** : Le runtime ne s'arrête jamais sur une déconnexion Slack.

- Reconnexion : `RunInboundWithReconnectAsync` avec backoff exponentiel (1s → 10s max).
- État dégradé : après 3 échecs consécutifs, `runtime.degraded=true` loggé.
- Récupération : reset du compteur et log `runtime.degraded=false` à la reconnexion réussie.

**Testé par** : `RunAsync_NeverCrashes_OnRepeatedDisconnections`, `RunAsync_EntersDegradedState_AfterRepeatedFailures`, `RunAsync_ExitsDegradedState_OnRecovery`.

## 5. Traçabilité — IDs propagés

**Invariant** : Tous les logs d'événements contiennent les IDs de traçabilité.

- Scopes obligatoires : `correlation_id`, `causation_id`, `event_id`, `agent_id`, `agent_name`, `run_id`.
- Pas de catch silencieux : toute exception loguée avec contexte + IDs.

**Testé par** : `RunAsync_PushesRunIdIntoLogScope`, `ConsumeLoop_IncludesContextIds_InExceptionLog`, `AllEventLogs_ContainRunId`, `CorrelationCausation_PropagatedCorrectly`.

## 6. Idempotence — Actions rejouables

**Invariant** : Toute action déclenchée depuis un événement doit être idempotente.

- Conséquence : replay safe, dédup en mémoire ne bloque pas les retries légitimes post-restart.
- Responsabilité : handlers métier doivent être idempotents (hors runtime).

**Testé par** : tests d'intégration avec fakes.

---

## Références

- ADR : `docs/adr/adr-sprint1-runtime.md`
- Epic 1 : `docs/epics/epic-1-foundations.md`
- Epic 2 : `docs/epics/epic-2-foundations.md`
- Copilot instructions : `.github/copilot-instructions.md`
