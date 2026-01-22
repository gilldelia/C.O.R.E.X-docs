# Epic 0 — Foundations (Local-first)

## Objectif
Mettre en place les fondations techniques de COREX pour un développement local fiable, reproductible et testable, avant toute intégration Slack/Trello/LLM “réelle”.

Résultat attendu : une base de code propre et opérable localement, avec conventions, configuration, logs structurés, gestion d’erreurs, tests, scripts et documentation d’exécution.

## Contexte
COREX vise à orchestrer des agents IA via une boucle type Ralph Loop. La robustesse (anti-boucles, idempotence, traçabilité) dépend d’un socle technique strict dès le départ.

Epic 0 est volontairement “local-first” :
- pas d’URL publique,
- pas d’infra cloud,
- pas de dépendance obligatoire à des services externes,
- tout doit être exécutable sur le poste.

## Périmètre (IN)
- Structure de solution et séparation des responsabilités :
  - Core / Runtime / Adapters / Infrastructure / Tests
- Configuration locale et gestion des secrets (sans commit)
- Encodage et conventions repo (UTF-8)
- Observabilité minimale (logs structurés + correlation IDs)
- Gestion globale des erreurs et résilience process
- Base de tests (unit + intégration via fakes)
- Scripts de dev (run/test/format) et dev loop
- Documentation “Quickstart” + Runbook local

## Hors périmètre (OUT)
- Intégration Slack réelle (Socket Mode) et routing complet
- Intégration Trello
- Mémoire long terme / RAG avancé
- Évaluations LLM / benchmarks (non requis)
- Déploiement Azure / Terraform / CI avancée
- Multi-agent dynamique complet

## Décisions d’architecture (règles)
- Aucun SDK Slack/Trello/HTTP dans le Domain (`Core`).
- Les Adapters font uniquement I/O (aucune logique métier).
- La boucle (Ralph Loop) réside dans `Runtime` et contrôle le flux.
- Toute action doit être idempotente, traçable, dédupliquable.
- Encodage repo : UTF-8 uniquement.

## Livrables attendus
1. Arborescence repo standardisée :
   - `/src`, `/tests`, `/docs`, `/scripts`
2. `.editorconfig` à la racine avec `charset=utf-8`
3. Gestion config/secrets :
   - `appsettings.json` + `appsettings.Development.json` (sans secrets)
   - secrets via user-secrets / env vars
   - validation “fail fast” au démarrage
4. Logs structurés :
   - correlation_id, causation_id, event_id, run_id, agent_id
   - sinks console + fichier (local)
5. Gestion d’erreurs :
   - aucun catch silencieux
   - exceptions loguées avec IDs
   - process reste vivant si un composant échoue
6. Tests :
   - unitaires : routing/dedup/state (même si minimal)
   - intégration : fakes (FakeSlack/FakeLLM)
7. Scripts :
   - `run`, `test`, `format` (ou équivalent)
8. Docs :
   - `README` Quickstart (5 minutes)
   - `docs/runbook-local.md` (diagnostic + commandes)

## User Stories (scope Epic 0)
- US0.1 Initialisation repo & structure
- US0.2 Configuration locale & secrets + validation
- US0.3 Observabilité minimale (logs structurés + IDs)
- US0.4 Gestion globale des erreurs & résilience process
- US0.5 Tests de base (unit + intégration via fakes)
- US0.6 Scripts & dev loop (run/test/format)
- US0.7 CI minimale (optionnel si déjà GitHub)
- US0.X Initialisation LLM (docs + règles + UTF-8)

## Critères de succès
- Un `git clone` + config minimale permet de lancer COREX en local.
- Les logs permettent de diagnostiquer un incident sans “deviner”.
- Une erreur dans un module ne tue pas l’orchestrateur.
- Les tests s’exécutent sans réseau, sans Slack réel, sans LLM réel.
- Aucun fichier texte en encodage non UTF-8.

## Checklist fin d’Epic
- [ ] Structure solution validée (dépendances propres)
- [ ] `.editorconfig` présent + UTF-8 respecté
- [ ] Secrets hors repo + fail fast
- [ ] Logs structurés avec IDs
- [ ] Erreurs loguées, pas de crash silencieux
- [ ] Tests OK (unit + intégration fakes)
- [ ] Scripts OK (run/test/format)
- [ ] Quickstart + Runbook local à jour
