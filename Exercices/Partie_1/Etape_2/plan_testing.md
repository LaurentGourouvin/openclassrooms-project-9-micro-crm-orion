## 1. Plan de testing périodique

### 1.1 Types de tests exécutés

**Front (Angular)**
- Tests unitaires Jasmine/Karma (8 tests existants, mode headless en CI)
- Génération de la coverage au format `lcov` pour SonarCloud

**Back (Spring Boot)**
- Tests d'intégration JUnit 5 (2 tests existants : `contextLoads`
  et `whenFindByEmail_thenReturnPerson`)
- Aucun test unitaire → identifié comme dette de tests
- Génération de la coverage XML via JaCoCo pour SonarCloud

**Transverse**
- Analyse statique SonarCloud (qualité + sécurité) sur front et back

> ⚠️ La couverture de tests actuelle est faible (8 tests front, 2 tests back).
> Le pipeline est conçu pour l'industrialisation : l'enrichissement des suites
> de tests est identifié comme une action de moyen terme (cf. plan de sécurité).

> La stratégie de branches retenue est GitHub Flow : `main` comme branche
> principale, branches de feature mergées via pull request.

### 1.2 Moments d'exécution

| Événement                          | Tests exécutés                                | Objectif                          |
| ---------------------------------- | --------------------------------------------- | --------------------------------- |
| `push` sur `main`                  | Tous les tests + Sonar + build Docker         | Mise à jour de la production      |
| `pull_request` vers `main`         | Tous les tests + analyse Sonar                | Garde-fou avant intégration       |
| Création d'un tag `vX.Y.Z`         | Tous les tests + build + publication d'images | Validation de release             |

> Pas de nightly build prévu à ce stade : volume de tests insuffisant pour
> justifier une exécution périodique. À envisager si la suite de tests s'étoffe.

### 1.3 Critères de réussite et d'alerte

- **Bloquant** : un test en échec stoppe le pipeline et empêche le merge
- **Bloquant** : Quality Gate SonarCloud KO sur les *new code* (vulnérabilité,
  bug critique, hotspot non revu)
- **Informatif** : seuil de coverage non atteint (avertissement, pas blocage à
  ce stade vu la faible couverture initiale)

### 1.4 Objectifs associés

- **Validation fonctionnelle** : confirmer que les fonctionnalités existantes
  marchent à chaque modification
- **Non-régression** : éviter qu'un changement casse l'existant
- **Qualité du code** : maintenir un niveau de qualité minimum mesurable via
  Sonar (duplication, complexité, code smells)