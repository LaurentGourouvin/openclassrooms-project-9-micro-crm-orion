## 1. Plan de sécurité

### 1.1 Rôle de SonarCloud

SonarCloud est l'outil central d'analyse statique du projet. Il intervient
**à chaque exécution du pipeline** sur les branches principales et les pull
requests. Son rôle est triple :

- **Détecter** les vulnérabilités, bugs, code smells et hotspots de sécurité
- **Mesurer** la qualité du code (coverage, duplication, complexité)
- **Bloquer** l'intégration via le **Quality Gate** lorsque les seuils ne sont
  pas respectés sur le code nouveau

### 2.2 Types de problèmes surveillés

| Catégorie         | Exemples                                              | Sévérité |
| ----------------- | ----------------------------------------------------- | -------- |
| Vulnérabilités    | Injections (A05), authentification cassée (A07)       | Critique |
| Bugs              | NullPointer, logique défectueuse                      | Élevée   |
| Hotspots sécurité | Code à revoir manuellement (CORS large, secrets…)     | À revoir |
| Code smells       | Méthodes trop longues, duplication, mauvais nommage   | Mineure  |
| Coverage          | Pourcentage de lignes/branches couvertes par les tests| Indicatif|

Référentiels appliqués : règles SonarSource pour Java/TypeScript, et alignement
avec le **OWASP Top 10:2025**. Risques particulièrement pertinents pour ce projet :

- [**A01 – Broken Access Control**](https://owasp.org/Top10/2025/A01_2025-Broken_Access_Control/) : l'API Spring Data REST expose tous les
  repositories sans contrôle d'accès → à durcir
- [**A02 – Security Misconfiguration**](https://owasp.org/Top10/2025/A02_2025-Security_Misconfiguration/) : CORS ouvert, configuration Spring Boot
  par défaut, HSQLDB en mémoire → à externaliser et restreindre
- [**A03 – Software Supply Chain Failures**](https://owasp.org/Top10/2025/A03_2025-Software_Supply_Chain_Failures/) : surveillance des dépendances
  npm/Gradle via Dependabot et SonarCloud
- [**A05 – Injection**](https://owasp.org/Top10/2025/A05_2025-Injection/) : prévenu via JPA/Spring Data REST, à confirmer par Sonar
- [**A07 – Authentication Failures**](https://owasp.org/Top10/2025/A07_2025-Authentication_Failures/) : aucune authentification implémentée
  actuellement → identifié comme dette de sécurité majeure

### 1.3 Bonnes pratiques attendues dans la CI

**Gestion des secrets**
- Aucun secret en clair dans le code, les workflows ou les images Docker
- Utilisation exclusive des **GitHub Secrets** (`SONAR_TOKEN`, etc.)
- Référence dans les workflows via `${{ secrets.NOM_DU_SECRET }}`

**Gestion des dépendances**
- `npm ci` côté front (build reproductible, basé sur `package-lock.json`)
- Gradle wrapper côté back (version figée dans le repo)
- Surveillance automatique des vulnérabilités via **Dependabot** (alertes
  GitHub natives)

**Bonnes pratiques code**
- CORS à durcir (actuellement `@CrossOrigin` sans restriction explicite)
- URL backend à externaliser côté front (actuellement hardcodée)
- Configuration BDD à externaliser côté back (actuellement HSQLDB par défaut)

### 1.4 Plan d'action de remédiation

| Horizon         | Actions                                                          |
| --------------- | ---------------------------------------------------------------- |
| Immédiat        | Mise en place SonarCloud + Quality Gate + GitHub Secrets         |
| Court terme     | Externalisation de la config (URL back, BDD), durcissement CORS  |
| Moyen terme     | Enrichissement des suites de tests, mise à jour Angular vers LTS |

## Ressources

- https://docs.sonarsource.com/sonarqube-cloud/analyzing-source-code/test-coverage/overview
- https://docs.sonarsource.com/sonarqube-cloud/analyzing-source-code/automatic-analysis
- https://docs.sonarsource.com/sonarqube-cloud/analyzing-source-code/scanners/sonarscanner-for-gradle
- https://docs.sonarsource.com/sonarqube-cloud/standards/about-new-code
- https://owasp.org/Top10/2025/