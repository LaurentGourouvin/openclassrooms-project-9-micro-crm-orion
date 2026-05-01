## 3. Plan de déploiement

Ce document décrit comment l'application MicroCRM est déployée, dans quel
ordre, et avec quels prérequis techniques.

> Ce plan complète la section 1.3 de `plan_conteneurisation.md` qui décrit
> la stratégie CD implémentée dans le pipeline GitHub Actions.

### 3.1 Prérequis techniques

| Outil | Version minimale | Usage |
| ----- | ---------------- | ----- |
| Docker Desktop | 4.x | Lancer les conteneurs |
| Git | 2.x | Cloner le repo |
| Accès Internet | — | Pull les images ghcr.io |

Aucune installation de Java, Node.js ou Gradle n'est nécessaire pour
déployer l'application via Docker. Ces outils ne sont requis que pour
le développement local sans Docker.

### 3.2 Déploiement local depuis les sources

C'est la méthode recommandée pour un développeur qui souhaite travailler
sur le projet.

**Étapes** :

1. Cloner le repository :
   ```shell
   git clone https://github.com/LaurentGourouvin/openclassrooms-project-9-micro-crm-orion
   cd openclassrooms-project-9-micro-crm-orion
   ```

2. Lancer l'application :
   ```shell
   docker compose up
   ```

3. Vérifier le bon démarrage :
   ```shell
   # Back
   curl http://localhost:8080/persons

   # Front
   open https://localhost
   ```

> ⚠️ Caddy (serveur web du front) applique une politique HTTPS-by-default.
> Le navigateur demandera d'accepter le certificat auto-signé.

4. Pour arrêter :
   ```shell
   docker compose down
   ```

### 3.3 Déploiement depuis les images publiées

Les images sont automatiquement publiées sur **GitHub Container Registry**
à chaque merge sur `main`. Elles peuvent être utilisées sans avoir le code
source.

**Étapes** :

1. Pull des images :
   ```shell
   docker pull ghcr.io/laurentgourouvin/microcrm-back:latest
   docker pull ghcr.io/laurentgourouvin/microcrm-front:latest
   ```

2. Lancer les conteneurs :
   ```shell
   docker run --rm -d -p 8080:8080 \
     ghcr.io/laurentgourouvin/microcrm-back:latest

   docker run --rm -d -p 80:80 -p 443:443 \
     ghcr.io/laurentgourouvin/microcrm-front:latest
   ```

3. Application accessible sur `https://localhost`.

#### Rollback vers une version précédente

En cas de problème, revenir à une version précédente via le tag SHA
disponible sur ghcr.io (cf. `plan_conteneurisation.md` section 1.7).

### 3.4 Déploiement automatisé via le pipeline CI/CD

Le pipeline GitHub Actions publie automatiquement les images Docker sur
ghcr.io à chaque merge sur `main`, après validation complète :

```
Commit → Pull Request → CI verte (tests + Sonar + Trivy) → Merge → Publish Docker Images
```

Aucune intervention manuelle n'est requise entre le commit et la
publication des images.

> Détails techniques du pipeline documentés dans `plan_ci.md` section 1.11.

### 3.5 Limites actuelles

| Limite | Description | Plan d'action |
| ------ | ----------- | ------------- |
| Pas de déploiement sur serveur cible | Les images sont publiées mais pas démarrées automatiquement sur un serveur | Watchtower, Argo CD ou PaaS (itération future) |
| HSQLDB en mémoire | Les données sont perdues à chaque redémarrage | Migrer vers PostgreSQL avec volume persistant |
| URL backend hardcodée | `http://localhost:8080` figée dans le bundle Angular | Externaliser via variable d'environnement au runtime |
| Certificat auto-signé | Caddy génère un certificat TLS auto-signé en local | Configurer un certificat valide en production |
