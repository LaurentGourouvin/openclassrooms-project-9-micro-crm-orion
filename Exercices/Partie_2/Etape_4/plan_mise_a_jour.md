## 5. Plan de mise à jour

Ce document décrit comment maintenir l'application et ses dépendances
à jour, et comment mettre à jour le pipeline CI/CD.

### 5.1 Mise à jour de l'application

#### Back-end (Spring Boot + Gradle)

| Composant | Version actuelle | Priorité de mise à jour | Raison |
| --------- | ---------------- | ----------------------- | ------ |
| Spring Boot | 3.2.5 | 🔴 Haute | 5 CVE CRITICAL Tomcat embarqué |
| Java | 17 | 🟢 Faible | LTS jusqu'en 2029 |
| Gradle | 8.7 | 🟢 Faible | Version récente et stable |
| HSQLDB | Runtime | 🟡 Moyenne | À remplacer par PostgreSQL |

**Procédure de mise à jour Spring Boot** :

```gradle
// back/build.gradle
plugins {
    id 'org.springframework.boot' version '3.4.0'   // ← mettre à jour ici
}
```

Puis vérifier que les tests passent :

```shell
cd back
./gradlew build
```

> ⚠️ Une mise à jour de Spring Boot implique une mise à jour de Tomcat
> embarqué, ce qui résoudra les 5 CVE CRITICAL identifiées par Trivy.

#### Front-end (Angular + npm)

| Composant | Version actuelle | Priorité de mise à jour | Raison |
| --------- | ---------------- | ----------------------- | ------ |
| Angular | 17.3.7 | 🟡 Moyenne | Hors LTS (LTS = Angular 18+) |
| Node.js | 20.9.0 | 🟢 Faible | LTS active |
| TypeScript | 5.4.2 | 🟢 Faible | Compatible Angular 17 |

**Procédure de mise à jour des dépendances npm** :

```shell
cd front
npm outdated              # lister les dépendances obsolètes
npm update                # mettre à jour les mineures
npx npm-check-updates -u  # mettre à jour les majeures (avec précaution)
npm install
npm test                  # vérifier que les tests passent
```

#### Images Docker

| Image | Version actuelle | Fréquence de vérification |
| ----- | ---------------- | ------------------------- |
| `node:20.9.0-slim` | 20.9.0 | Mensuelle |
| `gradle:jdk17` | jdk17 (tag flottant) | Mensuelle |
| `alpine:3.19` | 3.19 | Mensuelle |

**Procédure** : vérifier les nouvelles versions sur [Docker Hub](https://hub.docker.com),
mettre à jour le `Dockerfile`, pousser sur main pour déclencher le pipeline.

### 5.2 Mise à jour du pipeline CI/CD

#### Actions GitHub (pinning par hash)

Toutes les actions GitHub sont pinnées par hash de commit. Pour les mettre
à jour :

1. Consulter les releases de chaque action sur GitHub
2. Récupérer le hash du commit de la nouvelle version
3. Mettre à jour le hash dans `ci.yml` avec un commentaire de version

```yaml
# Exemple de mise à jour
uses: actions/checkout@<nouveau_hash> # v4.4.0
```

| Action | Version actuelle | Fréquence de vérification |
| ------ | ---------------- | ------------------------- |
| `actions/checkout` | v4.3.1 | Trimestrielle |
| `actions/setup-java` | v4.8.0 | Trimestrielle |
| `actions/setup-node` | v4.4.0 | Trimestrielle |
| `SonarSource/sonarqube-scan-action` | v6.0.0 | Trimestrielle |
| `aquasecurity/trivy-action` | v0.36.0 | Mensuelle (outil sécurité) |
| `docker/build-push-action` | v6.9.0 | Trimestrielle |

> ⚠️ `trivy-action` fait l'objet d'une surveillance mensuelle en raison
> de l'incident supply chain CVE-2026-33634 (mars 2026). Voir
> `plan_security.md` section 1.4.

#### Automatisation avec Dependabot

Dependabot peut automatiser la surveillance des dépendances et ouvrir
des Pull Requests de mise à jour automatiquement.

Configuration dans `.github/dependabot.yml` :

```yaml
version: 2
updates:
  # Actions GitHub
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"

  # Dépendances npm (front)
  - package-ecosystem: "npm"
    directory: "/front"
    schedule:
      interval: "weekly"

  # Dépendances Gradle (back)
  - package-ecosystem: "gradle"
    directory: "/back"
    schedule:
      interval: "weekly"
```

### 5.3 Mise à jour du monitoring (stack ELK)

| Composant | Version actuelle | Fréquence de vérification |
| --------- | ---------------- | ------------------------- |
| Elasticsearch | 8.11.0 | Semestrielle |
| Logstash | 8.11.0 | Semestrielle |
| Kibana | 8.11.0 | Semestrielle |

> Mettre à jour les 3 composants ELK **simultanément** et vers la même
> version pour éviter les incompatibilités.

**Procédure** :

```yaml
# monitoring/docker-compose-elk.yml
# Mettre à jour les 3 images vers la même version
image: docker.elastic.co/elasticsearch/elasticsearch:8.13.0
image: docker.elastic.co/logstash/logstash:8.13.0
image: docker.elastic.co/kibana/kibana:8.13.0
```

### 5.4 Fréquence et bonnes pratiques

| Type de mise à jour | Fréquence | Déclencheur |
| ------------------- | --------- | ----------- |
| Dépendances npm/Gradle (mineures) | Hebdomadaire | Dependabot |
| Actions GitHub | Trimestrielle | Revue manuelle |
| Images Docker de base | Mensuelle | Revue manuelle |
| Spring Boot / Angular (majeures) | À la demande | CVE critique ou fin de LTS |
| Stack ELK | Semestrielle | Revue manuelle |

**Règle d'or** : toute mise à jour passe par une **Pull Request** vers
`main`, ce qui déclenche automatiquement le pipeline CI (tests, Sonar,
Trivy). Une mise à jour qui casse les tests ne peut pas être mergée.

> La branch protection sur `main` garantit qu'aucune mise à jour dégradante
> ne peut être intégrée sans validation complète du pipeline.
