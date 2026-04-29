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
> améliorations sont détaillées en section 1.4 (appliquées) et 1.5
> (non appliquées, reportées).

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

#### CD implémenté

La phase de déploiement continu est **opérationnelle** via un job
`publish-docker` dans le workflow GitHub Actions :

1. **Build des images** (back + front) à partir du Dockerfile multi-stage
2. **Publication conditionnée** : uniquement sur push `main`, après
   validation complète du Quality Gate SonarCloud et du scan Trivy
3. **Tag des images** :
    - `latest` pour la dernière version stable
    - SHA court du commit pour la traçabilité (ex: `2c4b0e2`)
4. **Registre cible** : **GitHub Container Registry (ghcr.io)**
5. **Authentification** : `GITHUB_TOKEN` automatique (pas de secret manuel)
6. **Visibilité** : packages publics (le repo étant public)

> Détails techniques (conditions d'exécution, stratégie de tags,
> authentification) documentés dans `plan_ci.md` section 1.11.

#### Hors périmètre de cette mission

- **Déploiement effectif** sur un environnement cible (la publication
  rend les images disponibles, mais ne les fait pas tourner sur un
  serveur de production)
- **Orchestration Kubernetes** (Argo CD, Flux, Helm)
- **Cloud public** (AWS, Azure, GCP)

Ces évolutions sont envisageables pour des itérations ultérieures et
mentionnées dans le plan de mise à jour de la documentation finale.

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

### 1.6 Scan de vulnérabilités des images Docker

Le brief mentionne Twistlock comme outil possible pour scanner les images
Docker. Cet outil étant commercial (Prisma Cloud par Palo Alto Networks),
il a été remplacé par **Trivy** (Aqua Security), standard open-source de
l'industrie pour le scan d'images Docker.

#### Mise en œuvre

Trivy est intégré au pipeline CI via un job dédié `image-scan` qui tourne
en parallèle des autres jobs. À chaque exécution :

1. Les images `back` et `front` sont buildées localement sur le runner
2. Trivy scanne chaque image (sévérité `CRITICAL,HIGH`)
3. Les résultats sont uploadés au format **SARIF** dans
   `GitHub Security → Code Scanning`

#### Choix de version

L'action `aquasecurity/trivy-action` est pinnée en **v0.36.0** par hash
de commit, version publiée par Aqua après la remédiation de l'incident
de supply chain CVE-2026-33634 (mars 2026). Détails dans `plan_security.md`
section 1.4.

#### Couverture complémentaire à SonarCloud

| Outil | Couvre |
| ----- | ------ |
| **SonarCloud** | Code source applicatif (bugs, code smells, hotspots, vulnérabilités d'écriture) |
| **Trivy** | Images Docker (CVE connues dans les dépendances système : JRE, Tomcat embarqué, paquets Alpine) |

Cette double couverture met en place une **défense en profondeur** :
SonarCloud sécurise ce que l'on écrit, Trivy sécurise ce que l'on
embarque.

> Référentiel : OWASP Top 10:2025 — A03 Software Supply Chain Failures.

### 1.7 Utilisation des images publiées

Les images sont disponibles publiquement sur ghcr.io et peuvent être
récupérées sans authentification.

#### Pull des images

```shell
# Dernière version stable
docker pull ghcr.io/laurentgourouvin/microcrm-back:latest
docker pull ghcr.io/laurentgourouvin/microcrm-front:latest

# Version spécifique (rollback)
docker pull ghcr.io/laurentgourouvin/microcrm-back:<sha_court>
```

L'historique complet des versions publiées est consultable sur la page
**Packages** du repository GitHub.

#### Lancement local depuis les images publiées

```shell
# Back en arrière-plan
docker run --rm -d -p 8080:8080 \
  ghcr.io/laurentgourouvin/microcrm-back:latest

# Front (les deux ports 80 et 443 sont nécessaires car Caddy applique
# une politique HTTPS-by-default avec certificat auto-signé)
docker run --rm -d -p 80:80 -p 443:443 \
  ghcr.io/laurentgourouvin/microcrm-front:latest
```

L'application est ensuite accessible sur `https://localhost` (le navigateur
demandera d'accepter le certificat auto-signé).

#### Validation fonctionnelle

Le test end-to-end a été effectué : pull des images depuis ghcr.io,
démarrage local, vérification que le front charge et communique
correctement avec le back (récupération de la fixture `John Doe` /
`Orion Incorporated`). Cela confirme que les images publiées par la
CI/CD sont **autonomes et fonctionnelles**.

#### Limitations identifiées

Deux points sont à connaître pour un usage en production :

1. **Caddy en HTTPS-by-default** : le serveur web Caddy refuse d'écouter
   en HTTP simple et génère automatiquement un certificat TLS auto-signé.
   C'est une bonne propriété de sécurité (OWASP A02 — secure-by-default),
   mais demande de mapper le port 443 et d'accepter le certificat en
   environnement de développement.

2. **URL backend hardcodée dans le bundle Angular** : le fichier
   `front/src/app/config.ts` fige l'URL `http://localhost:8080` au
   moment du build. Pour un déploiement multi-environnement, cette URL
   devrait être externalisée via une variable d'environnement injectée
   au runtime. Identifié comme dette technique applicative dans
   `front_analyse.md`.

## Ressources

- https://docs.docker.com/compose/intro/features-uses/
- https://docs.docker.com/build/building/multi-stage/
- https://docs.docker.com/build/cache/
- https://docs.docker.com/build/ci/
- https://github.com/aquasecurity/trivy/security/advisories/GHSA-69fq-xp46-6x23 (supply chain trivy)