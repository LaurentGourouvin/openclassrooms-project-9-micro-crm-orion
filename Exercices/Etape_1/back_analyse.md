# Analyse

## Analyse de l'application BACK

L'application back repose sur : 
- **Spring Boot 3.2.5**
- **Java 17**
- **Spring Boot Data JPA**
- **Spring Boot Data Rest**
- **Gradle 8.7** (build tool)
- **JUnit 5** (test framework)
- **HSQLDB** (base de données en mémoire)

## Tests 

### Tests existants

Pour exécuter les tests de l'application Spring Boot :

```bash
cd back/
./gradlew test
```

Un total de 2 tests seront exécutés.

#### Tests d'intégration

Le `MicroCRMApplicationTests` contient 1 test :
- `contextLoads` : vérifie que le contexte Spring démarre correctement

Le `PersonRepositoryIntegrationTest` contient 1 test :
- `whenFindByEmail_thenReturnPerson` : test d'intégration JPA pour la recherche par email

## Les services

L'application ne contient pas de couche service explicite. À la place, elle expose les repositories via **Spring Data REST**, qui génère automatiquement une API REST.

### PersonRepository

Interface héritant de `PagingAndSortingRepository` et `CrudRepository` :
- Hérite des opérations CRUD standards (create, read, update, delete)
- Opération custom : `findByEmail(String email)` pour rechercher une personne par email
- Expose l'API à l'URL `/persons`
- CORS enabled via `@CrossOrigin`

### OrganizationRepository

Interface héritant de `PagingAndSortingRepository` et `CrudRepository` :
- Hérite des opérations CRUD standards
- Expose l'API à l'URL `/organizations`
- CORS enabled via `@CrossOrigin`

## Endpoints REST

Générés automatiquement par Spring Data REST :

**Persons**
- `GET /persons` : lister toutes les personnes (avec pagination)
- `POST /persons` : créer une nouvelle personne
- `GET /persons/{id}` : récupérer une personne
- `PUT /persons/{id}` : mettre à jour une personne
- `DELETE /persons/{id}` : supprimer une personne
- `GET /persons/search/findByEmail?email=...` : recherche custom par email

**Organizations**
- `GET /organizations` : lister toutes les organisations (avec pagination)
- `POST /organizations` : créer une nouvelle organisation
- `GET /organizations/{id}` : récupérer une organisation
- `PUT /organizations/{id}` : mettre à jour une organisation
- `DELETE /organizations/{id}` : supprimer une organisation

## Les entités

### Person

Entité JPA avec les champs suivants :
- `id` : identifiant généré automatiquement
- `firstName` : prénom
- `lastName` : nom
- `email` : adresse email
- `phone` : numéro de téléphone
- `bio` : biographie
- `createdAt` : timestamp de création (auto-généré)
- `updatedAt` : timestamp de mise à jour (auto-généré)
- `organizations` : relation ManyToMany avec Organization

Contient une méthode `@PreRemove` qui supprime la personne de toutes ses organisations lors de la suppression.

### Organization

Entité JPA avec les champs suivants :
- `id` : identifiant généré automatiquement
- `name` : nom de l'organisation
- `createdAt` : timestamp de création (auto-généré)
- `updatedAt` : timestamp de mise à jour (auto-généré)
- `persons` : relation ManyToMany avec Person (en cascade)

Contient les méthodes :
- `addPerson(Person)` : ajoute une personne à l'organisation
- `removePerson(Person)` : retire une personne de l'organisation

## Configuration

L'application possède une configuration minimale :
- `application.properties` : définit uniquement le nom de l'application (`spring.application.name=microcrm`)
- Base de données : HSQLDB en mémoire (via `runtimeOnly` dans Gradle)
- Pas de configuration explicite de port, datasource ou JPA (utilise les defaults Spring Boot)

⚠️ Recommandation : externaliser les configurations sensibles les configurations sensibles (port, datasource, profils) via des variables d'environnement ou des fichiers de propriété par environnement.

## Couverture de code - JaCoCo & SonarQube

Pour intégrer JaCoCo et SonarQube dans le pipeline CI/CD :

**1. Ajouter JaCoCo au `build.gradle`** :
```gradle
plugins {
    id 'jacoco'
}

jacoco {
    toolVersion = "0.8.10"
}

jacocoTestReport {
    dependsOn test
    reports {
        xml.required = true
        csv.required = false
        html.required = true
    }
}
```

**2. Configuration SonarQube** :
- Ajouter le plugin SonarQube Gradle (ou utiliser SonarScanner)
- Fichier `sonar-project.properties` :
```
sonar.projectKey=micro-crm
sonar.sources=src/main
sonar.tests=src/test
sonar.coverage.jacoco.xmlReportPaths=build/reports/jacoco/test/jacocoTestReport.xml
sonar.java.coveragePlugin=jacoco
```

**3. Commandes CI** :
```bash
./gradlew clean build jacocoTestReport
```

Cela génère un rapport HTML dans `build/reports/jacoco/test/html/index.html`

## Synthèse Back

- **Image Docker à utiliser** : **Java 17 LTS** (Alpine ou Temurin pour la légèreté)
- **Build tool** : Gradle 8.7
- **Étapes CI** : `./gradlew clean build` (inclut la compilation et les tests)
- **Artefacts** : `build/libs/microcrm-0.0.1-SNAPSHOT.jar` (JAR exécutable)
- **Tests** : 2 tests simples (1 contexte Spring, 1 intégration JPA)
- **API** : Générée automatiquement via Spring Data REST (endpoints `/persons` et `/organizations`)
- **CORS** : Activé pour tous les repositories
- **Port par défaut** : 8080 (Spring Boot default)
- **Dette technique identifiée** : 
  - Faible couverture de tests (contexte + 1 test d'intégration)
  - Configuration minimale, pas de gestion multi-environnements
  - Absence de contrôleur métier/logique applicative (tout est délegué à Spring Data REST)
