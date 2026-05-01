## 1. Plan de sécurité

### 1.1 Rôle de SonarCloud

SonarCloud est l'outil central d'analyse statique du projet. Il intervient
**à chaque exécution du pipeline** sur les branches principales et les pull
requests. Son rôle est triple :

- **Détecter** les vulnérabilités, bugs, code smells et hotspots de sécurité
- **Mesurer** la qualité du code (coverage, duplication, complexité)
- **Bloquer** l'intégration via le **Quality Gate** lorsque les seuils ne sont
  pas respectés sur le code nouveau

### 1.2 Types de problèmes surveillés

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
- Conteneurs Docker à exécuter en utilisateur non-root (actuellement root)

**Garde-fou Quality Gate**
- L'action `sonarqube-quality-gate-action` fait échouer le pipeline si
  le Quality Gate est KO
- Branch protection activée sur `main` : aucun merge
  possible si la CI échoue
- Les contournements admin sont désactivés (`Do not allow bypassing`)

### 1.4 Choix de version pour `trivy-action`

L'action `aquasecurity/trivy-action` a subi une attaque de supply chain en
mars 2026 (CVE-2026-33634) durant laquelle 76 des 77 tags ont été
force-pushés vers du code malveillant pendant ~12 heures.

**Mesures appliquées** :

- Utilisation de la **v0.36.0**, version publiée par Aqua Security après
  la remédiation complète de l'incident et la mise en place des
  protections (immutable releases, lockdown des automatisations).
- **Pinning par commit SHA** plutôt que par tag — c'est la défense
  ultime contre ce type d'attaque, les hashes de commit étant immuables
  contrairement aux tags qui peuvent être force-pushés.
- Cohérent avec la politique de pinning par hash appliquée à
  l'ensemble du workflow CI/CD.

**Références officielles** :
- Security advisory : https://github.com/aquasecurity/trivy/security/advisories/GHSA-69fq-xp46-6x23
- Discussion publique : https://github.com/aquasecurity/trivy/discussions/10425
- Communication Aqua : https://www.aquasec.com/blog/trivy-supply-chain-attack-what-you-need-to-know/

**Référentiel** : OWASP Top 10:2025 — A03 Software Supply Chain Failures

### 1.5 Résultats du scan Trivy (snapshot initial)

Le premier scan Trivy a révélé **61 vulnérabilités** dans les images Docker :

| Sévérité | Nombre approx. | Origine principale |
| -------- |----------------| ------------------ |
| CRITICAL | 5              | Apache Tomcat embarqué (Spring Boot) |
| HIGH     | 19             | musl libc (Alpine), autres |
| Autres   | 37             | Dépendances système Alpine |

**Analyse** :
- Les CVE Tomcat CRITICAL concernent des fonctionnalités (console JMX,
  authentification client, multi-host) **non exploitées** dans un contexte
  Spring Boot embarqué
- Les CVE Alpine système sont des risques potentiels au niveau OS

**Plan d'action** :
- **Court terme** : revue manuelle des CVE critiques sur GitHub Security,
  acknowledgment des non-applicables avec justification
- **Moyen terme** : mise à jour Spring Boot 3.2.5 → 3.4+ pour bénéficier
  d'un Tomcat plus récent (cf. plan de mise à jour - Partie 2)
- **Long terme** : intégration de la mise à jour automatique des dépendances
  via Dependabot

### 1.6 Résultats de l'analyse SonarCloud (snapshot initial)

#### Qualité du code

| Catégorie | Nombre | Sévérité dominante |
| --------- | ------ | ------------------ |
| Bugs | 2 | Medium |
| Code Smells | 36 | Major (25), Minor (12) |
| Reliability issues | 13 | — |
| Maintainability issues | 30 | — |

#### Répartition par sévérité

| Sévérité | Nombre |
| -------- | ------ |
| Blocker | 0 |
| High | 1 |
| Medium | 27 |
| Low | 12 |

#### Quality Gate

✅ **Passed** — le pipeline est bloquant sur le code nouveau.

#### Coverage

**37.4%** de couverture globale (front + back unifiés).
Dette identifiée : 8 tests front, 2 tests back. Objectif à moyen terme : > 80%.

#### Security Hotspots

| Statut | Nombre | Détail |
| ------ | ------ | ------ |
| Safe | 3 | `@CrossOrigin` CORS — revus et acceptés (Spring Data REST, contexte interne) |
| To Review | 1 | `front/src/index.html` — resource integrity check (Web:S5725, priorité Low) |

**75% des hotspots sont reviewés.**

Le hotspot restant (`Web:S5725`) concerne l'absence d'attribut `integrity`
sur une balise `<link>` dans `index.html`. La priorité est **Low** et le
risque réel est faible dans le contexte d'une application interne sans CDN
externe critique. Ce point est documenté comme dette technique à traiter
dans une itération ultérieure.

#### Alignement OWASP

| Risque OWASP | Résultat SonarCloud |
| ------------ | ------------------- |
| A01 – Broken Access Control | Pas de vulnérabilité détectée par Sonar. Risque identifié manuellement (Spring Data REST sans auth) |
| A02 – Security Misconfiguration | 3 hotspots CORS → marqués Safe après revue |
| A05 – Injection | Pas de vulnérabilité détectée (JPA prévient les injections SQL) |
| A07 – Authentication Failures | Non couvert par Sonar (pas d'auth implémentée → dette identifiée) |

> Aucun résultat dans les catégories OWASP Top 10 2025 et 2021 dans SonarCloud,
> ce qui confirme l'absence de vulnérabilité critique liée aux règles de sécurité
> applicative. Les risques restants sont documentés comme dette de sécurité.

### 1.7 Plan d'action de remédiation

| Horizon         | Actions                                                          |
| --------------- | ---------------------------------------------------------------- |
| Immédiat        | Mise en place SonarCloud + Quality Gate + Trivy + GitHub Secrets |
| Court terme     | Externalisation config (URL back, BDD), durcissement CORS, revue des CVE Trivy |
| Moyen terme     | Enrichissement des suites de tests, MAJ Spring Boot, MAJ Angular vers LTS |

## Ressources

- https://docs.sonarsource.com/sonarqube-cloud/analyzing-source-code/test-coverage/overview
- https://docs.sonarsource.com/sonarqube-cloud/analyzing-source-code/automatic-analysis
- https://docs.sonarsource.com/sonarqube-cloud/analyzing-source-code/scanners/sonarscanner-for-gradle
- https://docs.sonarsource.com/sonarqube-cloud/standards/about-new-code
- https://owasp.org/Top10/2025/