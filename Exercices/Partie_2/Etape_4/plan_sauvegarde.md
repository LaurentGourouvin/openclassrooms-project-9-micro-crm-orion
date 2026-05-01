## 4. Plan de sauvegarde des données

Ce document décrit ce qui doit être sauvegardé, à quelle fréquence,
et comment restaurer en cas d'incident.

### 4.1 Ce qui doit être sauvegardé

#### Données applicatives

| Élément | Criticité | Remarque |
| ------- | --------- | -------- |
| Base de données HSQLDB | 🔴 Élevée | Données métier (Persons, Organizations) |
| Configuration applicative | 🟡 Moyenne | `application.properties`, `logback-spring.xml` |
| Fichiers de configuration Docker | 🟡 Moyenne | `docker-compose.yml`, `Dockerfile` |

> ⚠️ **Limitation actuelle** : la base de données HSQLDB est **en mémoire**.
> Toutes les données sont perdues à chaque redémarrage du conteneur back.
> Dans un contexte de production réel, il faudrait migrer vers PostgreSQL
> avec un volume persistant pour pouvoir effectuer des sauvegardes réelles.

#### Artefacts du pipeline CI/CD

| Élément | Criticité | Remarque |
| ------- | --------- | -------- |
| Images Docker publiées | 🔴 Élevée | Disponibles sur ghcr.io avec tags SHA |
| Code source | 🔴 Élevée | Versionné dans GitHub |
| Workflows GitHub Actions | 🟡 Moyenne | Dans `.github/workflows/ci.yml` |
| Configuration SonarCloud | 🟢 Faible | Reconfigurable depuis le dashboard |

#### Configuration du monitoring

| Élément | Criticité | Remarque |
| ------- | --------- | -------- |
| `docker-compose-elk.yml` | 🟡 Moyenne | Dans le repo Git |
| `logstash.conf` | 🟡 Moyenne | Dans le repo Git |
| Dashboards Kibana | 🟢 Faible | Persistés dans le volume `kibana_data` |
| Données Elasticsearch | 🟢 Faible | Persistées dans le volume `es_data` |

### 4.2 Procédure de sauvegarde

#### Code source et configuration

Le code source et tous les fichiers de configuration sont versionnés dans
GitHub. La sauvegarde est donc **automatique et continue** à chaque commit.

```shell
# Vérifier que tout est bien commité et poussé
git status
git push origin main
```

#### Images Docker

Les images Docker sont automatiquement sauvegardées sur **ghcr.io** par
le pipeline CI/CD à chaque merge sur `main`. Chaque version est accessible
via son tag SHA court, ce qui constitue un **historique de versions**.

```shell
# Lister les versions disponibles
# → GitHub → onglet Packages → microcrm-back / microcrm-front
```

#### Base de données (contexte de production futur)

Dans un contexte de production avec PostgreSQL, la procédure serait :

```shell
# Sauvegarde quotidienne via pg_dump
docker exec postgres pg_dump -U postgres microcrm > backup_$(date +%Y%m%d).sql

# Compression et stockage distant
gzip backup_$(date +%Y%m%d).sql
```

**Fréquence recommandée** : quotidienne (nightly), avec rétention de 30 jours.

### 4.3 Procédure de restauration

#### Scénario 1 — Rollback applicatif

En cas de problème sur une nouvelle version, revenir à une version précédente
via le tag SHA disponible sur ghcr.io (cf. `plan_conteneurisation.md` section 1.7).

→ Temps estimé : **~2 minutes**.

#### Scénario 2 — Restauration complète depuis zéro

En cas de perte totale de l'environnement :

```shell
# 1. Cloner le repo
git clone https://github.com/LaurentGourouvin/openclassrooms-project-9-micro-crm-orion

# 2. Lancer l'application
cd openclassrooms-project-9-micro-crm-orion
docker compose up

# 3. Lancer le monitoring
docker compose -f monitoring/docker-compose-elk.yml up
```

→ Temps estimé : **~5 minutes** (hors téléchargement des images).

#### Scénario 3 — Restauration base de données (production future)

```shell
# Restaurer depuis une sauvegarde pg_dump
gunzip backup_20260501.sql.gz
docker exec -i postgres psql -U postgres microcrm < backup_20260501.sql
```

### 4.4 Limitations identifiées

| Limitation | Impact | Plan d'action |
| ---------- | ------ | ------------- |
| HSQLDB en mémoire | Impossible de sauvegarder les données applicatives | Migrer vers PostgreSQL (moyen terme) |
| Pas de sauvegarde automatisée des volumes Docker | Perte possible des dashboards Kibana | Ajouter un script de backup des volumes (court terme) |
| Pas de stockage distant | Sauvegarde locale uniquement | Configurer un stockage S3 ou équivalent (production) |
