# COREX Epic 4 — Dev Accelerator Invariants

Ce document liste les invariants stricts de l'Epic 4 (Copilot-Assisted Delivery).
Ils complètent les invariants Epics 2-3 (`docs/invariants.md`) sans les modifier.

**Statut** : Verrouillés (début Epic 4)

---

## Invariants hérités (Epics 2-3) — rappel

Les invariants suivants restent **non négociables** et **inchangés** :

| # | Invariant | Référence |
|---|-----------|-----------|
| 1 | 1 RunId = 1 action = 1 réponse | `docs/invariants.md` §1 |
| 2 | Single-consumer, ordre garanti | `docs/invariants.md` §2 |
| 3 | Backpressure observable (jamais silencieux) | `docs/invariants.md` §3 |
| 4 | Runtime jamais en crash sur déconnexion | `docs/invariants.md` §4 |
| 5 | IDs propagés dans tous les logs | `docs/invariants.md` §5 |
| 6 | Actions idempotentes (responsabilité handlers) | `docs/invariants.md` §6 |
| 7 | EventId ≠ RunId (dédup séparées) | `docs/invariants.md` §7 |

Toute PR qui touche un fichier lié à ces invariants requiert une **review humaine obligatoire**.

---

## I-4.1 — Idempotence US (pas de doubles)

**Invariant** : Relancer la même US ne crée pas de doubles PRs, doubles merges, ni doubles commits.

### Règles

- Chaque US est identifiée par un identifiant unique (`US-4.x`).
- Avant toute action, Copilot vérifie si une PR existe déjà pour cette US.
  - Si une PR ouverte existe → mise à jour de la PR existante (push sur la même branche).
  - Si une PR mergée existe → aucune action (log `us.already_merged us_id=US-4.x`).
  - Si aucune PR n'existe → création d'une nouvelle PR.
- Le `RunId` de l'exécution Copilot est dérivé de l'identifiant US : `RunId.FromExternal("epic4:{us_id}")`.
- La dédup s'applique au niveau de la PR GitHub (titre ou label contenant l'`us_id`).

### Scénarios de dédup

| Situation | Action Copilot | Log attendu |
|-----------|---------------|-------------|
| Aucune PR pour US-4.1 | Créer PR | `copilot.pr.created us_id=US-4.1 pr=#42` |
| PR #42 ouverte pour US-4.1 | Update PR #42 | `copilot.pr.updated us_id=US-4.1 pr=#42` |
| PR #42 mergée pour US-4.1 | Aucune action | `copilot.us.already_merged us_id=US-4.1 pr=#42` |
| PR #42 fermée (rejected) pour US-4.1 | Aucune action (escalade) | `copilot.us.escalation us_id=US-4.1 reason=pr_rejected` |

### Vérification

- Test unitaire : deux appels avec le même `us_id` ne créent qu'une PR.
- Test intégration : vérifier via GitHub API que le nombre de PRs pour un `us_id` donné est ≤ 1.

---

## I-4.2 — No-Unsafe-Merge

**Invariant** : Un merge automatique n'est autorisé que si toutes les conditions de sécurité sont satisfaites.

### Conditions requises (toutes obligatoires)

| # | Condition | Vérification | Outil |
|---|-----------|--------------|-------|
| 1 | Build green | `./scripts/build.ps1` → exit 0 | Local / CI |
| 2 | Tests green | `./scripts/test.ps1` → exit 0 | Local / CI |
| 3 | Lint clean | `./scripts/lint.ps1` → exit 0 | Local / CI |
| 4 | Aucun fichier sensible touché | Diff vs liste sensible | Copilot check |
| 5 | Review policy satisfaite | ≥ 1 approbation humaine (si configurée) | GitHub |
| 6 | Pas de conflits de merge | `git merge --no-commit --no-ff` clean | Git |

### Comportement

```
SI toutes conditions OK → merge auto autorisé
    Log: copilot.merge.auto us_id=US-4.x pr=#42 checks=all_green

SINON → escalade humaine obligatoire
    Log: copilot.merge.blocked us_id=US-4.x pr=#42 reason={condition_failed}
```

### Conditions bloquantes explicites

| Condition échouée | Action |
|-------------------|--------|
| Build fail | Copilot tente un fix (max 2 itérations), sinon escalade |
| Test fail | Copilot tente un fix (max 2 itérations), sinon escalade |
| Lint fail | Copilot applique `./scripts/format.ps1`, re-check |
| Fichier sensible touché | Escalade immédiate (pas de retry) |
| Review manquante | Attente (pas de bypass) |
| Conflit de merge | Escalade humaine |

### Vérification

- Test : simuler un merge avec un test qui fail → vérifier que le merge est bloqué.
- Test : simuler un merge avec un fichier sensible touché → vérifier l'escalade.

---

## I-4.3 — Blast Radius Limité

**Invariant** : Si des fichiers sensibles sont touchés, le merge automatique est interdit.

### Liste des fichiers sensibles

La liste complète est maintenue dans `docs/epics/epic-4-foundations.md` § "Fichiers sensibles".

#### Résumé par catégorie

| Catégorie | Fichiers / Patterns | Raison |
|-----------|---------------------|--------|
| Runtime / Orchestrateur | `src/Corex.Runtime/RalphRuntime.cs`, `src/Corex.Runtime/InMemoryRunIdDeduplicator.cs`, `src/Corex.Core/Domain/AgentEventRouter.cs`, `src/Corex.Core/Domain/RunId.cs`, `src/Corex.Core/Domain/Inbound/IInboundEventSource.cs` | Invariants 1-7 |
| Bootstrap / Config | `src/Corex.App/Program.cs`, `src/Corex.App/Configuration/AppConfiguration.cs`, `src/Corex.App/Configuration/AppConfigurationValidator.cs` | DI, validation fail-fast |
| Contrats / Docs | `docs/invariants.md`, `docs/invariants/*.md`, `docs/epics/epic-4-foundations.md`, `.github/copilot-instructions.md`, `docs/adr/*.md` | Source de vérité |
| Infrastructure | `src/Corex.Infrastructure/Observability/CorrelationContext.cs`, `src/Corex.Infrastructure/Resilience/DefaultResiliencePipelines.cs` | Traçabilité, résilience |
| Scripts CI/CD | `scripts/build.ps1`, `scripts/test.ps1`, `scripts/run.ps1` | Pipeline |

#### Règles d'évolution de la liste

- Ajout d'un fichier à la liste : autorisé sans ADR (extension de protection).
- Retrait d'un fichier de la liste : requiert une ADR + validation PO.
- Toute US qui ajoute un nouveau fichier « fondation » doit proposer son ajout à la liste.

### Comportement

```
POUR CHAQUE fichier modifié dans la PR :
    SI fichier ∈ liste_sensible :
        → BLOQUER le merge auto
        → Log: copilot.merge.sensitive_file us_id=US-4.x file={path} action=escalation
        → Notifier pour review humaine
```

### Vérification

- Test : créer une PR qui modifie `RalphRuntime.cs` → vérifier que le merge est bloqué.
- Test : créer une PR qui modifie uniquement un fichier non sensible → vérifier que le merge est autorisé.

---

## I-4.4 — Auditabilité Complète

**Invariant** : Chaque action Copilot doit produire des traces exploitables et reconstructibles.

### Champs obligatoires par action

| Action | Champs requis |
|--------|---------------|
| PR créée | `us_id`, `run_id`, `pr_number`, `branch`, `commit_sha`, `files_changed[]` |
| PR mise à jour | `us_id`, `run_id`, `pr_number`, `commit_sha`, `iteration` |
| Merge auto | `us_id`, `run_id`, `pr_number`, `merge_sha`, `checks_status` |
| Merge bloqué | `us_id`, `run_id`, `pr_number`, `reason`, `blocked_by` |
| Escalade | `us_id`, `run_id`, `reason`, `sensitive_files[]` |
| Rollback | `us_id`, `run_id`, `pr_number`, `revert_sha`, `reason` |

### Format de log structuré

```
copilot.{action} us_id={us_id} run_id={run_id} pr=#{pr_number} commit={sha} [champs additionnels]
```

Exemples :

```
[INF] copilot.pr.created us_id=US-4.1 run_id=abc123 pr=#42 branch=feature/us-4.1 commit=a1b2c3d files_changed=3
[INF] copilot.merge.auto us_id=US-4.1 run_id=abc123 pr=#42 merge_sha=d4e5f6g checks=all_green
[WRN] copilot.merge.blocked us_id=US-4.2 run_id=def456 pr=#43 reason=sensitive_file blocked_by=RalphRuntime.cs
[WRN] copilot.merge.blocked us_id=US-4.3 run_id=ghi789 pr=#44 reason=test_failed blocked_by=test:RunAsync_Deduplicates_RunId
[INF] copilot.rollback us_id=US-4.1 run_id=abc123 pr=#42 revert_sha=h7i8j9k reason=test_regression
```

### Contenu de la PR (body)

Chaque PR créée par Copilot doit contenir dans son body :

```markdown
## Copilot Dev Accelerator — PR automatique

**US** : US-4.x — {titre}
**RunId** : {run_id}
**Fichiers modifiés** : {liste}
**Fichiers sensibles touchés** : {liste ou "aucun"}
**Build** : ✅ / ❌
**Tests** : ✅ / ❌ ({count} tests)
**Lint** : ✅ / ❌
**Auto-merge éligible** : ✅ / ❌ (raison si non)
```

### Vérification

- Vérifier que chaque PR Copilot contient les champs requis dans le body.
- Vérifier que les logs `copilot.*` sont présents pour chaque action.
- Reconstituer le parcours d'une US via les logs seuls (sans ouvrir GitHub).

---

## Récapitulatif des garanties Epic 4

| # | Invariant | Verrouillé |
|---|-----------|------------|
| I-4.1 | Idempotence US (pas de doubles PRs/merges) | ✅ |
| I-4.2 | No-Unsafe-Merge (toutes conditions requises) | ✅ |
| I-4.3 | Blast Radius Limité (fichiers sensibles → escalade) | ✅ |
| I-4.4 | Auditabilité (logs structurés + PR body) | ✅ |

---

## Interaction avec les invariants Epics 2-3

| Invariant Epic 2-3 | Impact Epic 4 | Protection |
|---------------------|---------------|------------|
| 1 RunId = 1 action | Le RunId Copilot est distinct (préfixe `epic4:`) | Pas de collision avec les RunIds runtime |
| 2 Single-consumer | Aucun changement | Fichier sensible (`RalphRuntime.cs`) |
| 3 Backpressure | Aucun changement | Fichier sensible (`RalphRuntime.cs`) |
| 4 Résilience Slack | Aucun changement | Fichier sensible (`RalphRuntime.cs`) |
| 5 Traçabilité | Epic 4 ajoute ses propres logs (`copilot.*`) | Pas de modification des scopes existants |
| 6 Idempotence | Renforcée par I-4.1 (dédup US) | — |
| 7 EventId ≠ RunId | Aucun changement | Fichier sensible (`InMemoryRunIdDeduplicator.cs`, `AgentEventRouter.cs`) |

---

## Stratégie de rollback

| Situation | Détection | Action | Responsable |
|-----------|-----------|--------|-------------|
| Test regression post-merge | `./scripts/test.ps1` fail sur `master` | `git revert {merge_sha}` | Copilot (propose) → Humain (valide) |
| Fichier sensible modifié par erreur | Review humaine | Revert + correction | Humain |
| Boucle de PRs (> 3 sur même US) | Compteur interne | Stop Copilot, review humaine | Humain |
| Runtime dégradé après merge | Logs `runtime.degraded=true` | Rollback au dernier commit stable | Humain |
| Doute sur l'intention d'une US | Copilot signale ambiguïté | Stop, clarification PO | Humain |

---

## Références

- Invariants Epics 2-3 : `docs/invariants.md`
- Epic 4 Foundations : `docs/epics/epic-4-foundations.md`
- Architecture : `docs/architecture.md`
- Contrat de logs (runtime) : `docs/log-contract.md`
- ADR Runtime : `docs/adr/adr-1-runtime.md`
- Copilot instructions : `.github/copilot-instructions.md`
