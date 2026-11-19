# Gestion des incidents

Quand quelque chose casse en production, on **répare d'abord**, puis on **comprend** et on **écrit** ce qu'on a appris.
Un post-mortem n'a pas pour but de trouver un coupable mais d'éviter que l'incident se reproduise. Il est **sans
blâme** : on parle de processus et de systèmes, pas de personnes. Chaque post-mortem se termine par au moins une
**issue** qui améliore le code, le pipeline ou la supervision.

```
alerte ─▶ incident (diagnostic, correction) ─▶ post-mortem sans blâme ─▶ issue ─▶ amélioration ─▶ (moins d'incidents, détectés plus tôt)
```

## Supervision en place

| Outil | Adresse | Sert à |
|---|---|---|
| Mailpit | http://localhost:8025 | recevoir les notifications d'Alertmanager |
| Alertmanager | http://localhost:9093 | alertes actives, inhibées ; poser un **silence** pendant une intervention |
| Prometheus | http://localhost:9090 | *Alerts*, *Targets*, requêtes PromQL |
| Grafana | http://localhost:3001 | dashboard *TaskFlow : RED et disponibilité* |

| Alerte | Signification | Premier réflexe |
|---|---|---|
| `TaskFlowApiDown` (critical) | `/health` ne répond pas `ok` à travers le front depuis 1 min | méthode ci-dessous, dès l'étape 1 |
| `TaskFlowMetricsDown` (warning) | Prometheus ne lit plus `api:3000/metrics` | `ps` : l'API tourne-t-elle ? |
| `TaskFlowHighErrorRate` (warning) | plus de 5 % de 5xx pendant 5 min | logs `level=error` de l'API, état de la base |
| `TaskFlowHighLatency` (warning) | p95 > 300 ms pendant 5 min | panneau Duration par route ; base, ressources du poste |

## Méthode de diagnostic

Du plus visible au plus profond (dans `~/taskflow-prod`) :

1. **Alerte** : laquelle, depuis quand (`activeAt`), quelle cible ? Notez l'heure : c'est le début de la chronologie.
2. **Dashboard** : depuis quand la disponibilité est-elle tombée ? La version a-t-elle changé (*Version en ligne*,
   annotations de déploiement) ? Les courbes RED montent-elles (erreurs) ou **s'arrêtent**-elles (API morte) ?
3. **Symptôme** : `curl -sk https://taskflow.localhost/health` ; que voit l'utilisateur ?
4. **Conteneurs** : `docker compose -f compose.prod.yaml ps` (qui est `unhealthy`, `Restarting` ?)
5. **Logs** du service suspect, filtrés :
   `docker compose -f compose.prod.yaml logs --no-log-prefix --since 15m api | jq -c 'select(.level != "info")'`
6. **Cause racine** : configuration, code, base, ressources ? **Prouvez-la** (`docker inspect`, fichier, requête).
7. **Correction**, puis **vérification** : `ps`, `/health`, une action utilisateur, sonde verte, mail `[RESOLVED]`.

Pendant une intervention longue : un **silence** dans Alertmanager (avec un commentaire et une durée) évite de
recevoir la même alerte en boucle, sans la supprimer.

## Modèle de post-mortem

Copiez ce modèle pour chaque incident, du plus récent au plus ancien.

```markdown
## Incident n° N - Titre court

**Gravité** : faible / moyenne / élevée
**Détection** : alerte <nom> / utilisateur / autre - délai entre le début et la détection : X min
**Durée** : début hh:mm - fin hh:mm (durée totale = temps de rétablissement)
**Rédaction** : ...

### Résumé
Deux ou trois phrases : ce qui s'est passé et l'impact pour les utilisateurs.

### Chronologie (heure, UTC ou locale : préciser)
- hh:mm - Changement à l'origine de l'incident
- hh:mm - Premier signal (alerte, message d'un utilisateur...)
- hh:mm - Diagnostic : ce qu'on a regardé, ce qu'on a trouvé
- hh:mm - Correction appliquée
- hh:mm - Retour à la normale confirmé (sonde verte, [RESOLVED])

### Cause racine (5 pourquoi)
1. Pourquoi ... ?
...

### Ce qui a bien fonctionné
### Ce qui a moins bien fonctionné

### Actions
| Action | Type | Issue | Statut |
| --- | --- | --- | --- |
| ... | prévention / détection / processus | #N | à faire |
```

---

## Incident n° 1 - L'API redémarre en boucle après une modification du `.env`

**Gravité** : élevée (aucune tâche consultable ni modifiable)
**Détection** : alerte `TaskFlowApiDown`, reçue 1 min 26 s après le début de la panne
**Durée** : 14:03:10 - 14:21:40 UTC (18 min 30 s)
**Rédaction** : binôme TaskFlow

### Résumé

Pendant 18 minutes, l'application affichait sa page mais ne chargeait plus aucune tâche (« Erreur de l'API (HTTP
502) »). L'API redémarrait en boucle : la variable `DATABASE_URL` avait été renommée par erreur en `DATABASE_UR` dans le
`.env` de production, et l'API retombait sur une adresse de base par défaut (`localhost`) qui n'existe pas dans son
conteneur. La supervision a prévenu l'équipe avant tout utilisateur.

### Chronologie (UTC)

- 14:01 - Le `.env` de production est modifié à la main ; `DATABASE_URL` devient `DATABASE_UR`.
- 14:03:10 - Le conteneur `api` est recréé avec ce `.env` ; il échoue au démarrage et redémarre en boucle.
- 14:03:25 - Prometheus : `probe_success` et `up{job="taskflow-api"}` à 0 ; les alertes passent *pending*.
- 14:04:36 - Mail `[FIRING:1] TaskFlowApiDown prod` reçu. `TaskFlowMetricsDown` est inhibée (une seule notification).
- 14:06 - Grafana : disponibilité « EN PANNE » depuis 14:03 ; les courbes RED **s'arrêtent** (pas de données) ; la
  version en ligne n'a pas changé : ce n'est pas l'image.
- 14:08 - `curl -sk https://taskflow.localhost/health` : 502. `docker compose ps` : `api` en `Restarting (1)`.
- 14:11 - Logs de l'API filtrés avec `jq` :
  `{"level":"fatal","message":"Démarrage impossible : connect ECONNREFUSED 127.0.0.1:5432","hint":"Vérifiez DATABASE_URL..."}`.
  L'API vise `127.0.0.1` : la valeur **par défaut** du code, donc `DATABASE_URL` n'est pas définie dans le conteneur.
- 14:15 - Preuve : `docker inspect` montre `DATABASE_UR=postgres://...@db:5432/taskflow` ; `grep -n DATABASE .env` aussi.
- 14:18 - Nom corrigé dans `.env`, `IMAGE_TAG=$(cat .deployed_tag) docker compose -f compose.prod.yaml up -d --force-recreate api`.
- 14:19:30 - `api` *healthy*, `/health` → `{"status":"ok","db":"up",...}`, création d'une tâche réussie.
- 14:21:40 - Mail `[RESOLVED]` ; sonde verte dans Grafana. `.env.bak` supprimé.

### Cause racine (5 pourquoi)

1. Pourquoi le site renvoyait-il 502 ? Le front ne joignait plus l'API.
2. Pourquoi ? L'API redémarrait en boucle et n'était jamais prête.
3. Pourquoi ? Elle tentait de joindre `localhost:5432`, où il n'y a pas de base dans un conteneur.
4. Pourquoi ? `DATABASE_URL` était absente (variable renommée) et le code retombe **silencieusement** sur une valeur par
   défaut de développement, même en production.
5. Pourquoi l'erreur a-t-elle atteint la production ? Le `.env` de production est modifié à la main, sans relecture ni
   contrôle automatique de sa complétude.

### Ce qui a bien fonctionné

- L'alerte a prévenu en moins d'une minute et demie, avant tout signalement utilisateur ; une seule notification grâce
  à l'inhibition.
- Le dashboard a orienté tout de suite : « l'API ne répond plus », « la version n'a pas changé ».
- Les logs JSON, filtrés avec `jq`, ont donné l'indice décisif (`127.0.0.1`) en une commande.

### Ce qui a moins bien fonctionné

- Rien n'empêche l'API de démarrer en production avec une configuration incomplète.
- Le `.env` de production est modifié sans second regard et sans comparaison avec `.env.example`.
- 13 minutes entre l'alerte et la correction : le runbook ne dit pas quoi vérifier en premier pour cette alerte.

### Actions

| Action | Type | Issue | Statut |
| --- | --- | --- | --- |
| En production (`APP_ENV=production`), l'API refuse de démarrer si `DATABASE_URL` est absente, avec un message explicite ; test dans `tests/config.test.js` | prévention | #29 | à faire |
| `deploy.sh` compare les noms de variables de `.env` et de `.env.example` et s'arrête s'il en manque | prévention | #13 | à faire |
| Compléter le tableau « Premier réflexe » de ce document pour `TaskFlowApiDown` (ps, logs `fatal`, `docker inspect`) | processus | #14 | fait |
| Router `TaskFlowApiDown` vers le canal d'équipe (et pas seulement un mail) | détection | #15 | à faire |

<!-- Ajoutez vos post-mortems au-dessus de celui-ci, du plus récent au plus ancien. -->
