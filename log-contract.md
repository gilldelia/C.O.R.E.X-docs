# Log Contract — Epic 3 Observabilité

Ce document définit le **contrat de logs** permettant de reconstituer un scénario complet sans lire le code.

---

## Champs obligatoires (sur chaque log de flux)

| Champ | Description | Exemple |
|-------|-------------|---------|
| `run_id` | Identifiant unique du run (dérivé de l'event Slack) | `d4c70ad7-dddc-abe2-bb4c-148c246fc187` |
| `event_id` | Identifiant unique de l'événement interne | `8e831239-d0ad-40b8-adad-7337ca858eb3` |
| `stage` | Étape du pipeline | `Received` / `Decision` / `Action` |
| `outcome` | Résultat de l'étape | `Success` / `Dropped` / `Skipped` / `Failed` |
| `reason` | Raison (si outcome != Success) | `duplicate_runid` / `channel_missing` / `empty_payload` |

---

## Logs par étape

### Stage: Received

| Log | Champs | Quand |
|-----|--------|-------|
| `runtime.event.received` | `run_id`, `event_id`, `stage=Received`, `outcome=Success`, `channel`, `text` | Message Slack reçu et enqueued |
| `runtime.event.received` | `run_id`, `event_id`, `stage=Received`, `outcome=Dropped`, `reason=duplicate_runid` | RunId déjà traité |

### Stage: Decision

| Log | Champs | Quand |
|-----|--------|-------|
| `runtime.event.decision` | `run_id`, `event_id`, `stage=Decision`, `outcome=Success`, `decision`, `decision_reason` | Décision calculée |

### Stage: Action

| Log | Champs | Quand |
|-----|--------|-------|
| `runtime.event.action` | `run_id`, `event_id`, `stage=Action`, `outcome=Success`, `action`, `slack_ts` | Post Slack réussi |
| `runtime.event.action` | `run_id`, `event_id`, `stage=Action`, `outcome=Skipped`, `action`, `reason` | Action sautée (NoAction, channel_missing, empty_payload) |
| `runtime.event.action` | `run_id`, `event_id`, `stage=Action`, `outcome=Failed`, `action`, `reason` | Erreur lors du post |

---

## Exemple complet : `echo hello`

```
[INF] runtime.event.received run_id=abc123 event_id=evt456 stage=Received outcome=Success channel=C0AAV text="echo hello"
[INF] runtime.event.decision run_id=abc123 event_id=evt456 stage=Decision outcome=Success decision=Echo decision_reason=prefix=echo
[INF] runtime.event.action   run_id=abc123 event_id=evt456 stage=Action   outcome=Success action=Echo slack_ts=1769520482.284249
```

### Scénario doublon (retry Slack)

```
[INF] runtime.event.received run_id=abc123 event_id=evt456 stage=Received outcome=Success ...
[INF] runtime.event.decision ...
[INF] runtime.event.action   ...
[WRN] runtime.event.received run_id=abc123 event_id=evt789 stage=Received outcome=Dropped reason=duplicate_runid
```

### Scénario NoAction

```
[INF] runtime.event.received run_id=def456 event_id=evt111 stage=Received outcome=Success channel=C0AAV text="bonjour"
[INF] runtime.event.decision run_id=def456 event_id=evt111 stage=Decision outcome=Success decision=NoAction decision_reason=default
[INF] runtime.event.action   run_id=def456 event_id=evt111 stage=Action   outcome=Skipped action=NoAction reason=no_action_required
```

---

## Corrélation

Tous les logs d'un même run partagent le même `run_id`. Le scope Serilog inclut également :
- `correlation_id` : identifiant de corrélation (pour chaîner plusieurs runs si nécessaire)
- `causation_id` : identifiant du run parent (bootstrap)
- `agent_id` / `agent_name` : agent concerné

Ces champs sont dans le scope (suffixe du template Serilog) et non répétés dans le message.

---

## Template Serilog recommandé

```json
{
  "outputTemplate": "{Timestamp:O} [{Level:u3}] {Message:lj} | run_id={run_id} event_id={event_id} corr={correlation_id}{NewLine}{Exception}"
}
```

---

## Validation

Pour valider le contrat :
1. Exécuter `echo hello` sur Slack.
2. Copier les logs.
3. Vérifier que chaque étape (`Received`, `Decision`, `Action`) est présente avec le même `run_id`.
4. Vérifier que `outcome` et `reason` sont cohérents.

Temps estimé : < 2 minutes.
