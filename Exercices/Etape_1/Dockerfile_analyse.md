# Analyse du Dockerfile

## Vue d'ensemble
Le Dockerfile utilise une stratégie **multi-stage build** pour construire et déployer une application micro-services composée d'un frontend Angular et d'un backend Spring Boot. Il produit deux images finales : une pour le frontend et une pour le backend, ainsi qu'une image `standalone` qui les combine.

---

## Étapes du Build

### 1. **Étape : `front-build`** (Lignes 1-8)
```dockerfile
FROM node as front-build
COPY ./front /src
WORKDIR /src
RUN npm ci \
    && npx @angular/cli build --optimization
```

**Objectif** : Compiler l'application Angular en artefacts statiques

**Détails** :
- **Image de base** : `node` (Node.js standard) pour la compilation
- **Copie du code** : Tout le dossier `./front` vers `/src`
- **Installation des dépendances** : `npm ci` (clean install - reproductible)
- **Build Angular** : `npx @angular/cli build --optimization` produit un bundle optimisé
- **Artefact produit** : `dist/microcrm/browser/` (fichiers statiques compilés)

---

### 2. **Étape : `back-build`** (Lignes 10-16)
```dockerfile
FROM gradle:jdk17 as back-build
COPY ./back /src
WORKDIR /src
RUN ./gradlew build
```

**Objectif** : Compiler l'application Spring Boot

**Détails** :
- **Image de base** : `gradle:jdk17` (Gradle + Java 17 intégré)
- **Copie du code** : Tout le dossier `./back` vers `/src`
- **Build Gradle** : `./gradlew build` compile et exécute les tests
- **Artefact produit** : `build/libs/microcrm-0.0.1-SNAPSHOT.jar` (JAR exécutable)

---

### 3. **Étape : `front`** (Lignes 18-30)
```dockerfile
FROM alpine:3.19 as front
COPY --from=front-build /src/dist/microcrm/browser /app/front
COPY misc/docker/Caddyfile /app/Caddyfile
RUN apk add caddy
WORKDIR /app
EXPOSE 80 443
CMD ["/usr/sbin/caddy", "run"]
```

**Objectif** : Image de production pour servir le frontend

**Détails** :
- **Image de base** : `alpine:3.19` (ultra-légère, ~5 MB)
- **Récupère l'artefact** : Fichiers compilés du `front-build`
- **Configuration Caddy** : Copie la configuration depuis `misc/docker/Caddyfile`
- **Caddy** : Serveur web inverse léger (remplace Nginx, ~20 MB)
- **Ports exposés** : 80 (HTTP) et 443 (HTTPS)
- **Commande de démarrage** : Lance le serveur Caddy

---

### 4. **Étape : `back`** (Lignes 32-42)
```dockerfile
FROM alpine:3.19 as back
COPY --from=back-build /src/build/libs/microcrm-0.0.1-SNAPSHOT.jar /app/back/
RUN apk add openjdk21-jre-headless
WORKDIR /app
EXPOSE 4200
CMD ["java", "-jar", "/app/back/microcrm-0.0.1-SNAPSHOT.jar"]
```

**Objectif** : Image de production pour le backend

**Détails** :
- **Image de base** : `alpine:3.19`
- **Récupère l'artefact** : JAR compilé du `back-build`
- **JRE headless** : Environnement Java 21 sans interface graphique (~50 MB)
- **Port exposé** : 4200 (port custom du backend)
- **Commande de démarrage** : Exécute le JAR Spring Boot

---

### 5. **Étape : `standalone`** (Lignes 44-54)
```dockerfile
FROM alpine:3.19 as standalone
COPY --from=front / /
COPY --from=back / /
COPY misc/docker/supervisor.ini /app/supervisor.ini
RUN apk add supervisor
WORKDIR /app
CMD ["/usr/bin/supervisord", "-c", "/app/supervisor.ini"]
```

**Objectif** : Image combinée contenant frontend ET backend

**Détails** :
- **Image de base** : `alpine:3.19`
- **Fusion des images** : Copie le contenu de `front` et `back` vers l'image unique
- **Supervisor** : Gestionnaire de processus pour lancer Caddy + Java ensemble
- **Configuration** : `supervisor.ini` définit les 2 services à démarrer
- **Commande de démarrage** : Lance Supervisor qui gère les deux applications

---

## Stratégie de Build 

### Multi-stage 
- **Avantage** : Les images de base `node` et `gradle:jdk17` (utilisées pour compiler) ne sont pas incluses dans l'image finale
- **Résultat** : Images finales très légères (Alpine + artefacts compilés uniquement)
- **Économie** : Réduit de ~1 GB la taille de l'image pour la production

### Deux modes de déploiement
1. **Images séparées** 
   - `docker build --target=front` → Image frontend seule (~100 MB)
   - `docker build --target=back` → Image backend seule (~100 MB)
   - Déploiement indépendant, meilleure scalabilité

2. **Image standalone** 
   - `docker build --target=standalone` → Tout-en-un (~200 MB)
   - Un seul conteneur pour tester l'application complète

---

## Résumé pour CI/CD

| Aspect | Détail |
|--------|--------|
| **Images finales** | `front` (Alpine+Caddy) et `back` (Alpine+JRE21) |
| **Artefacts attendus** | `/src/dist/microcrm/browser/` et `/src/build/libs/*.jar` |
| **Temps de build** | Long (~3-5 min) : npm install + Gradle build |
| **Taille estimée** | Front : ~100 MB, Back : ~100 MB, Standalone : ~200 MB |
| **Mode recommandé** | Déployer `front` et `back` séparément pour la production |
