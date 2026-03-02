# Epic 4 — Dev Accelerator (Copilot-Assisted Delivery)

## Objectif

Utiliser Copilot comme agent de développement assisté pour accélérer la livraison des US,
tout en garantissant la sécurité, l'auditabilité et le contrôle humain sur les actions critiques.

Epic 4 ne change pas le runtime COREX lui-même : il ajoute une couche d'outillage
et de gouvernance autour de l'utilisation de Copilot comme "dev accelerator".

---

## Contexte

Epics 0-3 ont livré :

- Un runtime robuste (Ralph Loop) avec invariants verrouillés (dédup, backpressure, résilience, traçabilité).
- Un flux fonctionnel end-to-end (Slack → Decision → Action → Trace observable).
- Un contrat de logs (`stage`/`outcome`/`run_id`) permettant la reconstitution sans debugger.
- 127 tests CI green, smoke tests reproductibles en < 10 min.

Epic 4 introduit un cadre formel pour que Copilot puisse :

1. Proposer des implémentations (code + tests) sur des US cadrées.
2. Créer des PRs avec des changements vérifiables.
3. Respecter des limites strictes (fichiers sensibles, invariants, blast radius).

---

## Périmètre (IN)

- Contrat Copilot : rôles, responsabilités, limites.
- Policy d'auto-merge : conditions, checks requis, exclusions.
- Liste des fichiers sensibles (escalade humaine obligatoire).
- Invariants Epic 4 spécifiques (dédup US, no-unsafe-merge, auditabilité).
- Stratégie de rollback / stop.
- Documentation de cadrage (ce document + invariants dédiés).

## Hors périmètre (OUT)

- Changements au runtime Ralph Loop / orchestrateur.
- Intégration LLM dans le pipeline de décision.
- Multi-agents / HA / scaling distribué.
- Déploiement cloud ou CI/CD automatisé.
- Modifications aux invariants Epic 2 (verrouillés).

---

## Invariants hérités (Epics 2-3)

Tous les invariants documentés dans `docs/invariants.md` restent **non négociables** :

| # | Invariant | Statut |
|---|-----------|--------|
| 1 | 1 RunId = 1 action = 1 réponse | ✅ Verrouillé |
| 2 | Single-consumer, ordre garanti | ✅ Verrouillé |
| 3 | Backpressure observable (jamais silencieux) | ✅ Verrouillé |
| 4 | Runtime jamais en crash sur déconnexion | ✅ Verrouillé |
| 5 | IDs propagés dans tous les logs | ✅ Verrouillé |
| 6 | Actions idempotentes (responsabilité handlers) | ✅ Verrouillé |
| 7 | EventId ≠ RunId (dédup séparées) | ✅ Verrouillé |

Toute PR touchant un fichier lié à ces invariants déclenche une **escalade humaine**.

---

## Nouveaux invariants Epic 4

Documentés en détail dans `docs/invariants/epic-4-dev-accelerator-invariants.md`.

### I-4.1 — Idempotence US (pas de doubles)

Relancer la même US ne crée pas de doubles PRs, doubles merges, ni doubles commits.

- Déduplication par identifiant d'US (`US-4.x`) ou `RunId` associé.
- Si une PR existe déjà pour une US, Copilot met à jour la PR existante (pas de nouvelle PR).
- Si une US est déjà mergée, aucune action n'est déclenchée.

### I-4.2 — No-Unsafe-Merge

Un merge automatique n'est autorisé que si **toutes** les conditions suivantes sont remplies :

1. Build CI green (`./scripts/build.ps1` → exit 0).
2. Tous les tests passent (`./scripts/test.ps1` → exit 0).
3. Lint clean (`./scripts/lint.ps1` → exit 0).
4. Aucun fichier sensible touché (voir liste ci-dessous).
5. Review policy satisfaite (au moins 1 approbation humaine si configurée).
6. Pas de conflits de merge.

Si **une seule** condition échoue → **pas de merge**, escalade humaine.

### I-4.3 — Blast Radius Limité

Si une PR touche un ou plusieurs fichiers de la **liste sensible**, le merge automatique est **interdit**.
L'action requise est une **review humaine obligatoire**.

### I-4.4 — Auditabilité Complète

Chaque action Copilot doit produire des traces exploitables :

- Logs structurés avec `us_id`, `run_id`, `pr_number`, `commit_sha`.
- Lien vers la PR GitHub.
- Lien vers la carte Trello (si applicable).
- Diff des fichiers modifiés dans le corps de la PR.
- Raison de l'escalade (si applicable).

---

## Fichiers sensibles (escalade humaine obligatoire)

Toute modification à ces fichiers par Copilot requiert une review humaine **avant merge**.

### Catégorie : Runtime / Orchestrateur

| Fichier | Raison |
|---------|--------|
| `src/Corex.Runtime/RalphRuntime.cs` | Boucle principale, invariants 1-5 |
| `src/Corex.Runtime/InMemoryRunIdDeduplicator.cs` | Invariant 1, 7 (dédup RunId) |
| `src/Corex.Core/Domain/AgentEventRouter.cs` | Dédup EventId, routing |
| `src/Corex.Core/Domain/RunId.cs` | Value Object critique (déterminisme SHA-256) |
| `src/Corex.Core/Domain/Inbound/IInboundEventSource.cs` | Contrat d'entrée du runtime |

### Catégorie : Bootstrap / Configuration

| Fichier | Raison |
|---------|--------|
| `src/Corex.App/Program.cs` | Bootstrap, DI, validation config |
| `src/Corex.App/Configuration/AppConfiguration.cs` | Contrat de config |
| `src/Corex.App/Configuration/AppConfigurationValidator.cs` | Validation fail-fast |

### Catégorie : Contrats / Documentation

| Fichier / Pattern | Raison |
|-------------------|--------|
| `docs/invariants.md` | Source de vérité des invariants |
| `docs/invariants/epic-4-dev-accelerator-invariants.md` | Invariants Epic 4 |
| `docs/epics/epic-4-foundations.md` | Contrat Epic 4 (ce document) |
| `.github/copilot-instructions.md` | Instructions Copilot |
| `docs/adr/*.md` | Décisions d'architecture |

### Catégorie : Infrastructure critique

| Fichier | Raison |
|---------|--------|
| `src/Corex.Infrastructure/Observability/CorrelationContext.cs` | Propagation IDs |
| `src/Corex.Infrastructure/Resilience/DefaultResiliencePipelines.cs` | Politiques de résilience |

### Catégorie : Scripts CI/CD

| Fichier | Raison |
|---------|--------|
| `scripts/build.ps1` | Pipeline de build |
| `scripts/test.ps1` | Pipeline de test |
| `scripts/run.ps1` | Lancement runtime |

---

## Composants Epic 4

### Copilot Agent (rôle)

Copilot agit en tant que **développeur assisté** avec les responsabilités suivantes :

| Responsabilité | Description |
|----------------|-------------|
| Implémentation | Écrire le code + tests pour une US donnée |
| PR Creation | Créer une PR avec description, diff, liens |
| Self-Check | Vérifier build + tests avant de proposer le merge |
| Escalade | Signaler toute situation hors limites (fichier sensible, test fail, ambiguïté) |

Copilot n'a **pas** le droit de :

| Interdit | Raison |
|----------|--------|
| Merger sans checks green | Invariant I-4.2 |
| Modifier un fichier sensible sans review | Invariant I-4.3 |
| Supprimer des tests existants | Régression |
| Modifier les invariants | Verrouillés |
| Exécuter des commandes destructives (drop DB, delete branch main, etc.) | Sécurité |
| Créer des US / changer le backlog | Responsabilité PO |

### Policy d'auto-merge

```
SI (build green)
ET (tests green)
ET (lint clean)
ET (aucun fichier sensible touché)
ET (review policy OK)
ET (pas de conflits)
ALORS → merge auto autorisé
SINON → escalade humaine obligatoire
```

### Stratégie de rollback / stop

| Situation | Action |
|-----------|--------|
| Test qui fail post-merge | Revert immédiat de la PR (git revert) |
| Fichier sensible modifié par erreur | Revert + review humaine |
| Boucle de PRs (> 3 PRs sur même US sans merge) | Stop Copilot, review humaine |
| Runtime dégradé après deploy | Rollback au dernier commit stable |
| Doute sur l'intention d'une US | Stop, demander clarification PO |

### Limites de responsabilité

| Domaine | Copilot | Humain |
|---------|---------|--------|
| Code + tests | ✅ Propose | ✅ Review + merge final |
| Fichiers sensibles | ❌ Escalade | ✅ Seul décideur |
| Invariants | ❌ Ne touche pas | ✅ Seul décideur |
| Backlog / US | ❌ Exécute | ✅ Définit |
| Architecture (ADR) | ❌ Propose draft | ✅ Valide |
| Rollback | ✅ Propose | ✅ Décide |

---

## Flux de travail type (US lifecycle)

```
1. PO crée l'US (Trello / backlog)
           ↓
2. Copilot reçoit l'US → vérifie dédup (I-4.1)
           ↓
3. Copilot implémente (code + tests)
           ↓
4. Copilot vérifie : build ✅ tests ✅ lint ✅
           ↓
5. Copilot crée PR avec description + diff + liens
           ↓
6. Check : fichiers sensibles touchés ?
    ├── OUI → escalade humaine (I-4.3)
    └── NON → auto-merge si policy OK (I-4.2)
           ↓
7. Merge → logs structurés (I-4.4)
           ↓
8. Si test fail post-merge → revert auto
```

---

## Règles de validation

- Chaque US Epic 4 doit référencer ce document comme contrat.
- Aucune US ne peut contourner la liste des fichiers sensibles.
- Smoke tests Epic 3 doivent rester green après chaque merge Epic 4.
- Les invariants Epics 2-3 sont vérifiés par les tests existants (127+).

---

## Risques identifiés

| Risque | Impact | Probabilité | Mitigation |
|--------|--------|-------------|------------|
| Copilot modifie un invariant | Critique | Faible | Liste fichiers sensibles + review humaine |
| Boucle infinie de PRs | Moyenne | Moyenne | Stop après 3 tentatives |
| Test flaky → faux green | Moyenne | Faible | Review humaine sur PR non triviales |
| Drift entre contrat et implémentation | Moyenne | Moyenne | Revue fin d'Epic |
| Copilot introduit une dépendance non souhaitée | Faible | Moyenne | Review humaine sur `.csproj` |

---

## Dettes acceptées (à traiter si nécessaire)

| Dette | Acceptée car | À traiter si |
|-------|-------------|--------------|
| Pas de CI/CD automatisé (scripts locaux) | Local-first MVP | Multi-contributeurs ou deploy auto |
| Pas de persistance état Copilot (runs, PRs) | In-memory / logs suffisants | Volume de PRs > 20/jour |
| Liste fichiers sensibles manuelle | Suffisant pour Epic 4 | Changement structurel majeur |

---

## Références

- Invariants Epics 2-3 : `docs/invariants.md`
- Invariants Epic 4 : `docs/invariants/epic-4-dev-accelerator-invariants.md`
- Architecture : `docs/architecture.md`
- Contrat de logs : `docs/log-contract.md`
- Configuration : `docs/configuration.md`
- ADR Runtime : `docs/adr/adr-1-runtime.md`
- ADR Backpressure : `docs/adr/adr-epic2-backpressure.md`
- Epic 3 : `docs/epics/epic-3-foundations.md`
- Copilot instructions : `.github/copilot-instructions.md`
