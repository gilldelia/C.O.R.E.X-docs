# Epic 1 — Foundations (Runtime & Traçabilité)

## Objectif
Mettre en place le runtime COREX (Corex.Runtime) et la traçabilité de bout en bout, en migrant la boucle Ralph, en modélisant le RunId et en assurant la résilience minimale côté Slack (Socket Mode) sans dépendance cloud ni fonctionnalités utilisateur Slack/Trello.

## Contexte
Epic 0 a établi un socle local-first (config, logs, erreurs, scripts, fakes). Epic 1 se concentre sur l’orchestrateur runtime, la boucle Ralph, la modélisation/propagation du RunId et la résilience I/O (Slack Socket Mode) tout en restant local, traçable, idempotent et dédupliquable.

## Périmètre (IN)
- Création du projet `Corex.Runtime` et intégration dans la solution.
- Migration/implémentation de la boucle Ralph (orchestrateur) dans Runtime.
- Modélisation du `RunId` dans le domaine et propagation dans tous les logs/événements.
- Choix et implémentation du modèle de concurrence de la boucle (scheduler, workers, cancellation).
- Résilience Slack (Socket Mode) avec retry/circuit breaker fakes ou clients simulés (local-first).
- Documentation des décisions d’architecture et des invariants du runtime.

### RunId — génération et propagation
- Source externe (ex. message Slack) : `RunId.FromExternal(externalId)` → GUID déterministe dérivé de l’ID externe (SHA-256 → GUID).
- Source runtime : `RunId.NewRuntime(seed?)` ; seed par `event_id` pour stabilité intra-event, sinon GUID nouveau.
- RunId attaché dès l’entrée (`AgentInboundEvent`, `InboundEnvelope`) et propagé dans scopes de log (`run_id`).

## Hors périmètre (OUT)
- Fonctionnalités Slack utilisateur réelles (messages publics, threads, uploads, RTM live).
- Fonctionnalités Trello utilisateur réelles (cartes, listes, boards).
- Toute infrastructure cloud ou déploiement (Azure/GCP/etc.).
- Fonctionnalités Epic 2 et ultérieures (mémoire longue, RAG, multi-agent dynamique avancé, LLM prod).

## Décisions d’architecture (règles)
- Local-first : aucune dépendance obligatoire à des services externes pour exécuter la boucle.
- Orchestrateur dans `Corex.Runtime` : décide/supervise, ne code pas l’action métier (agents exécutent uniquement).
- Concurrence : modèle explicite (ex. boucle event-driven + workers coopératifs, cancellation tokens, pas de threads sauvages).
- Traçabilité : propagation stricte de `run_id`, `correlation_id`, `event_id`, `agent_id`, `causation_id` dans les logs et messages internes.
- Idempotence/dédup : chaque RunId au plus une action métier et une réponse Slack ; dédup sur `event_id`.
- Résilience I/O : retry contrôlé + circuit breaker minimal sur Slack (Socket Mode) avec fakes ou clients simulés ; aucune exception I/O ne doit arrêter l’orchestrateur.
- Aucune logique métier dans les adapters ; ils restent I/O (Slack/Trello/LLM).
- Tests nécessaires pour toute fonctionnalité runtime (unit + intégration fakes) ; pas de catch silencieux.

## Livrables attendus
- Projet `Corex.Runtime` ajouté à la solution, référencé par l’app.
- Implémentation de la boucle Ralph dans Runtime avec gestion du RunId et cancellation propre.
- Propagation `run_id` dans logs/événements et contexte d’observabilité.
- Résilience Slack (retry/circuit breaker) intégrée côté adapter/runtime (local-first, fakes acceptés).
- Documentation des décisions (ce document) et mise à jour des scripts/doc si nécessaire.
- Jeux de tests unitaires et d’intégration (fakes) couvrant orchestration, dédup, idempotence, résilience.

## User Stories (scope Epic 1)
- US1.1 : Ajouter le projet `Corex.Runtime` et dépendances dans la solution.
- US1.2 : Implémenter la boucle Ralph (orchestration) avec cancellation et logs corrélés.
- US1.3 : Modéliser `RunId` dans le domaine et propager dans tous les logs/événements du runtime.
- US1.4 : Choisir et implémenter le modèle de concurrence (scheduler/worker) de la boucle.
- US1.5 : Intégrer résilience I/O côté Slack (Socket Mode) avec retry/circuit breaker et fakes.
- US1.6 : Couvrir par tests unitaires et intégration (fakes) la traçabilité, la dédup, l’idempotence et la résilience.
- US1.7 : Documenter les décisions et écarts éventuels.

## Critères de succès
- La solution contient `Corex.Runtime` et l’app s’appuie dessus pour orchestrer la boucle Ralph.
- Chaque exécution possède un `RunId` propagé dans les logs/événements ; un RunId déclenche au plus une action métier et une réponse Slack.
- La boucle gère cancellation propre et exceptions sans tuer l’orchestrateur (handlers globaux + résilience I/O).
- Les tests (unit + intégration fakes) passent en local et en CI ; pas de catch silencieux.
- Résilience Slack : indisponibilité n’interrompt pas le runtime ; retry/circuit breaker actifs.

## Écarts acceptés (si nécessaires)
- Slack en mode fake/local plutôt que Socket Mode réel tant que l’infrastructure réseau n’est pas stabilisée.
- Pas de connectivité Trello/LLM réelle (fakes uniquement) durant Epic 1.
- Absence de cloud/deploiement ; exécution uniquement locale.

## Checklist fin d’Epic
- [ ] Structure solution mise à jour (`Corex.Runtime` présent, références propres)
- [ ] Boucle Ralph opérationnelle dans Runtime (cancellation, concurrence choisie, logs avec IDs)
- [ ] `RunId` modélisé et propagé dans logs/événements
- [ ] Résilience Slack (retry + circuit breaker) active et testée (fakes acceptés)
- [ ] Tests unitaires et intégration (fakes) OK
- [ ] Docs à jour (ce contrat, runbook si impact)

## Revue clôturée
- (À remplir à la fin de l’Epic)
