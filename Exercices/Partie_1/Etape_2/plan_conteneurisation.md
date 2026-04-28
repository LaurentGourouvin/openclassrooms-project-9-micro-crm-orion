## 1. Plan de conteneurisation et de déploiement

### 1.1 Rôle des Dockerfiles existants

Le projet fournit un **Dockerfile multi-stage** unique à la racine, contenant
plusieurs cibles (`targets`) :

| Stage          | Rôle                                                          |
| -------------- | ------------------------------------------------------------- |
| `front-build`  | Compile l'application Angular (Node + npm + ng build)         |
| `back-build`   | Compile le JAR Spring Boot via Gradle                         |
| `front`        | Image finale du front (Alpine + Caddy + bundle Angular)       |
| `back`         | Image finale du back (Alpine + JRE + JAR Spring Boot)         |
| `standalone`   | Image combinée (front + back via Supervisor)                  |

> Cf. analyse détaillée dans `Dockerfile_analyse.md`. Plusieurs ajustements ont
> été identifiés (versions à pinner, cohérence Java 17, port back à corriger,
> exécution non-root, déplacement des tests hors du build Docker). Ces
> améliorations seront appliquées à l'étape de mise en œuvre.

### 1.2 Rôle de `docker-compose`

Un fichier `docker-compose.yml` orchestre les deux services en local :

- **`back`** : conteneur Spring Boot (port 8080)
- **`front`** : conteneur Caddy servant le bundle Angular (port 80)

Objectifs du `docker-compose` :
- Permettre à un développeur de lancer l'application complète avec `docker compose up`
- Garantir un environnement reproductible et identique en local et en CI
- Servir de base à la stratégie de déploiement et au monitoring (étape ELK
  en partie 2)

> La base de données HSQLDB est embarquée dans le conteneur back (en mémoire).
> En conséquence, **les données sont perdues à chaque redémarrage** du conteneur.
> Dans un contexte de production réel, il faudrait extraire la BDD vers un
> service dédié (PostgreSQL/MySQL) avec volume persistant. Ce point est
> documenté comme dette technique mais hors périmètre de la mission CI/CD.

### 1.3 Stratégie de déploiement

Le périmètre de la mission cible un **déploiement local containerisé** + une
**publication automatisée des images Docker** sur un registre.

**Étapes envisagées dans la CD**

1. **Build des images** (front et back) à partir du Dockerfile multi-stage
2. **Publication conditionnée** : uniquement sur `main` ou tags `vX.Y.Z`
   (pas sur les pull requests pour éviter de polluer le registre)
3. **Tag des images** :
    - `latest` pour la branche `main`
    - `vX.Y.Z` pour les tags de release (versionnage sémantique)
4. **Publication** sur **GitHub Container Registry (ghcr.io)**
5. **Documentation** des commandes pour relancer l'application à partir des
   images publiées

**Hors périmètre de cette mission**
- Déploiement sur un environnement cloud (AWS, Azure, GCP)
- Orchestration Kubernetes

Ces points peuvent être envisagés dans une itération ultérieure et seront
mentionnés dans le plan de mise à jour de la documentation finale.

### 1.4 Améliorations apportées au Dockerfile

L'analyse du Dockerfile fourni (cf. `Dockerfile_analyse.md`) a identifié plusieurs
points d'amélioration. Les corrections suivantes ont été appliquées :

| Amélioration | Avant | Après | Bénéfice |
| ------------ | ----- | ----- | -------- |
| Image Node pinnée | `node` (sans tag) | `node:20.9.0-slim` | Reproductibilité, compatibilité Angular 17.3 garantie |
| Cohérence Java | Build JDK 17 / Runtime JRE 21 | Build JDK 17 / Runtime JRE 17 | Élimine les comportements imprévisibles liés à un runtime différent du build |
| Port `back` corrigé | `EXPOSE 4200` (port Angular dev) | `EXPOSE 8080` (port Spring Boot) | Documentation cohérente avec la réalité |
| Tests sortis du build Docker | `./gradlew build` (inclut tests) | `./gradlew bootJar -x test` | Build Docker plus rapide, séparation des responsabilités (les tests sont exécutés en CI) |
| Cache `apk` non conservé | `apk add caddy` | `apk add --no-cache caddy` | Images finales plus légères, bonne pratique Alpine |

> ⚠️ Le pinning d'image en version Alpine pour le stage `back-build` (ex.
> `eclipse-temurin:17-jdk-alpine`) n'a pas pu être appliqué en raison de
> contraintes de plateforme (incompatibilité ARM64 sur certaines variantes
> Alpine). L'image `gradle:jdk17` est conservée pour ce stage uniquement,
> qui est de toute façon **détruit après le build** et n'impacte pas la
> taille de l'image finale.

### 1.5 Améliorations identifiées non appliquées

Certaines améliorations identifiées dans l'analyse ne sont pas encore
appliquées et sont reportées au plan d'action :

- **Utilisateur non-root** dans les conteneurs (sécurité OWASP A02 –
  Security Misconfiguration)
- **Pinning de version précise** sur le stage `back-build`
- **Vérification du `.dockerignore`** pour optimiser le contexte de build

Ces points sont documentés dans `Dockerfile_analyse.md` et seront traités
dans une itération ultérieure.

## Ressources

- https://docs.docker.com/compose/intro/features-uses/
- https://docs.docker.com/build/building/multi-stage/
- https://docs.docker.com/build/cache/
- https://docs.docker.com/build/ci/