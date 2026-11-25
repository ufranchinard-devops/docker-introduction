# TP : Sécurisation d'un conteneur Docker

Application : **quiz-docker**, un site statique (HTML / CSS / JavaScript) construit avec **Vite**.
Point de départ : le Dockerfile de l'exercice `dockerfile-exercice2`.

Outil d'analyse des vulnérabilités : **Docker Scout**.

---

## Mission 1 : audit de l'image Docker existante

### Dockerfile initial (`Dockerfile.initial`)

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package.json package-lock.json ./
COPY . .
RUN npm install && npm install -g serve && npm run build && rm -fr node_modules
EXPOSE 3000
CMD [ "serve", "-s", "dist" ]
```

### Commandes utilisées

```powershell
docker build -t quiz-docker:initial .
docker run -d --name quiz-initial -p 3001:3000 quiz-docker:initial
docker exec quiz-initial whoami
docker images quiz-docker:initial
docker history quiz-docker:initial
docker scout quickview quiz-docker:initial
docker scout cves quiz-docker:initial
```

Les résultats bruts sont dans les fichiers `audit-quickview-initial.txt`, `audit-cves-initial.txt`, `taille-initiale.txt` et `couches-initiales.txt`.

### État initial de l'image et risques identifiés

| Critère | État initial | Risque identifié |
|---|---|---|
| Image de base | `node:18-alpine` | Node.js 18 n'est plus maintenu (fin de support en avril 2025) : plus de correctifs de sécurité |
| Utilisateur | `root` | En cas de faille, l'attaquant a tous les droits dans le conteneur |
| Taille | 221 Mo sur disque (55 Mo compressés) | Surface d'attaque inutilement grande |
| Paquets | 272 | Beaucoup d'outils inutiles à l'exécution (npm, yarn, corepack, serve…) |
| Vulnérabilités | 7 critiques / 42 hautes / 31 moyennes / 11 basses | Failles connues et exploitables |
| Failles corrigeables | Les 7 critiques et 42 hautes ont un correctif | Risque évitable par une simple mise à jour |
| Build | Une seule étape | Les outils de build restent dans l'image finale |
| Serveur web | `serve`, installé globalement avec npm | Dépendance npm supplémentaire, exécutée en root |
| Limites de ressources | Aucune | Risque de déni de service sur l'hôte |
| Attestations supply chain | Absentes | Origine de l'image non vérifiable |
| Note Docker Scout | **D (50 %)**, 3 politiques respectées sur 7 | Image non conforme aux bonnes pratiques |

### Réponses aux questions

**L'image utilisée est-elle adaptée à une utilisation en production ?**
Non. L'image Alpine est légère, mais la version de Node.js est obsolète. L'image contient 49 vulnérabilités critiques ou hautes, et le processus tourne en root.

**Quels composants sont réellement nécessaires pour exécuter l'application ?**
L'application est un site statique. Une fois construite (`npm run build`), elle se résume au dossier `dist` (HTML, CSS, JS) et à un simple serveur web. Node.js, npm, Vite et `serve` ne servent qu'à la construction : ils sont inutiles à l'exécution.

**Quels risques de sécurité identifiez-vous ?**
- Exécution en root.
- Version de Node.js en fin de vie, et vulnérabilités critiques connues dans l'image de base.
- Présence d'outils inutiles (npm, yarn, serve…) qui augmentent la surface d'attaque.
- Aucune limite de ressources ni de privilèges au lancement du conteneur.
- Pas de `.dockerignore` : risque de copier des fichiers sensibles dans l'image (`.git`, `.env`…).

**Quelles améliorations recommanderiez-vous en priorité ?**
1. Faire un build multi-étapes : Node.js uniquement pour construire, puis un serveur web minimal pour servir `dist`.
2. Utiliser un utilisateur non root.
3. Utiliser une image de base maintenue, avec un tag précis, et la mettre à jour.
4. Ajouter un fichier `.dockerignore`.
5. Limiter les ressources et les privilèges au lancement du conteneur.

---

## Mission 2 : sécuriser l'image et le conteneur Docker

### Dockerfile sécurisé (`Dockerfile`)

```dockerfile
# ---- Etape 1 : build du site avec Vite ----
FROM node:26-alpine AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# ---- Etape 2 : image finale minimale, sans Node ----
FROM nginxinc/nginx-unprivileged:1.30-alpine
USER root
RUN apk upgrade --no-cache \
 && apk del --no-cache nginx-module-geoip nginx-module-image-filter nginx-module-njs nginx-module-xslt curl
COPY --from=build /app/dist /usr/share/nginx/html
USER 101
EXPOSE 8080
```

### Fichier `.dockerignore`

```
node_modules
npm-debug.log
.git
.gitignore
.env
Dockerfile*
.dockerignore
*.md
*.txt
```

### Lancement du conteneur avec limitation des ressources et des privilèges

```powershell
docker build -t quiz-docker:secure .
docker run -d --name quiz-secure -p 3002:8080 `
  --memory 128m --cpus 0.5 --pids-limit 100 `
  --read-only --tmpfs /tmp `
  --cap-drop ALL --security-opt no-new-privileges `
  quiz-docker:secure
```

### Application des recommandations

| Recommandation | Mise en œuvre |
|---|---|
| Images de base minimales | `nginx-unprivileged` Alpine : l'image finale ne contient ni Node.js, ni npm, ni Vite |
| Ne pas exécuter en root | Image `nginx-unprivileged` + `USER 101`. Vérifié : `docker exec quiz-secure whoami` renvoie `nginx` |
| Utiliser `.dockerignore` | Le fichier exclut `.git`, `.env`, `node_modules`, les Dockerfiles et les fichiers d'audit |
| Tags d'images précis | `node:26-alpine` et `nginx-unprivileged:1.30-alpine` au lieu de tags flottants |
| COPY plutôt que ADD | Seule l'instruction `COPY` est utilisée |
| Limiter les ressources | `--memory 128m`, `--cpus 0.5`, `--pids-limit 100` |
| Limiter les privilèges | `--cap-drop ALL`, `--security-opt no-new-privileges`, `--read-only` (seul `/tmp` est accessible en écriture) |
| Builds multi-étapes | L'étape `build` (Node.js + Vite) est séparée de l'image finale (Nginx) |
| Ne pas exposer le socket Docker | Aucun montage de `/var/run/docker.sock` |
| Vérifier régulièrement la sécurité | Scan avec Docker Scout après chaque build, puis `apk upgrade` pour appliquer les correctifs |

---

## Mission 3 : comparer les résultats avant et après sécurisation

### Construction, test et publication

```powershell
# Construction de l'image sécurisée
docker build -t quiz-docker:secure .

# Démarrage et vérification
docker run -d --name quiz-secure -p 3002:8080 `
  --memory 128m --cpus 0.5 --pids-limit 100 `
  --read-only --tmpfs /tmp `
  --cap-drop ALL --security-opt no-new-privileges `
  quiz-docker:secure
docker ps --filter name=quiz-secure      # STATUS : Up
docker exec quiz-secure whoami           # nginx
# Application accessible sur http://localhost:3002

# Publication sur Docker Hub
docker login
docker tag quiz-docker:secure ufranchinard/quiz-docker:secure
docker tag quiz-docker:dhi ufranchinard/quiz-docker:dhi
docker push ufranchinard/quiz-docker:secure
docker push ufranchinard/quiz-docker:dhi
```

Images publiées : https://hub.docker.com/r/ufranchinard/quiz-docker (tags `secure` et `dhi`)

### Tableau comparatif

| Objectif | Critère | Image initiale | Image sécurisée | Justification |
|---|---|---|---|---|
| **Bonnes pratiques** | Image de base | `node:18-alpine` | `nginx-unprivileged:1.30-alpine` | Node 18 n'est plus maintenu. Nginx suffit pour servir un site statique |
| | Tag précis | Partiel | ✅ `26-alpine` / `1.30-alpine` | Builds reproductibles et maîtrisés |
| | `.dockerignore` | ❌ Non | ✅ Oui | Évite de copier `.git`, `.env` et `node_modules` dans l'image |
| | Multi-stage build | ❌ Non | ✅ Oui | Les outils de build restent dans l'étape de build, pas dans l'image finale |
| | `COPY` / `ADD` | `COPY` | `COPY` | `ADD` peut télécharger ou extraire des fichiers de façon implicite |
| | Mise à jour des paquets | ❌ Non | ✅ `apk upgrade` | Applique les correctifs de sécurité disponibles |
| **Surface d'attaque** | Outils de développement | ❌ Node, npm, yarn, serve | ✅ Aucun | Moins d'outils exploitables par un attaquant |
| | Modules inutiles | ❌ Présents | ✅ Supprimés (`apk del`) | geoip, njs, xslt, image-filter et curl ne servent pas au site |
| | Nombre de paquets | 272 | 26 | −90 % de composants à surveiller |
| | Taille sur disque | 221 Mo | 104 Mo | −53 % : image plus légère et plus rapide à déployer |
| | Taille compressée | 55 Mo | 29 Mo | −47 % |
| **Privilèges et ressources** | Utilisateur | ❌ root | ✅ nginx (UID 101) | Une faille ne donne pas les droits root |
| | Capacités Linux | ❌ Par défaut | ✅ `--cap-drop ALL` | Retire tous les privilèges système inutiles |
| | Élévation de privilèges | ❌ Possible | ✅ `no-new-privileges` | Bloque toute élévation de privilèges |
| | Système de fichiers | ❌ Lecture / écriture | ✅ `--read-only` | Empêche de modifier ou d'installer quoi que ce soit dans le conteneur |
| | Mémoire / CPU | ❌ Illimités | ✅ 128 Mo / 0,5 CPU | Protège l'hôte contre un déni de service |
| | Processus | ❌ Illimités | ✅ `--pids-limit 100` | Empêche une « fork bomb » |
| | Port | 3000 | 8080 | Port non privilégié, compatible avec un utilisateur non root |
| | Socket Docker | Non monté | Non monté | Monter le socket donnerait le contrôle total de l'hôte |
| **Analyse des vulnérabilités** (Docker Scout) | CVE critiques | 7 | 5 | Correctifs appliqués avec `apk upgrade` |
| | CVE élevées | 42 | 2 | −95 % |
| | CVE moyennes | 31 | 3 | −90 % |
| | CVE faibles | 11 | 0 | −100 % |
| | **Total CVE** | **91** | **10** | **−89 %** |
| | Note Scout | D (50 %) | C (56 %) | 4 politiques respectées sur 7, au lieu de 3 |
| **Publication** | Registry | Locale | Docker Hub | Image partageable et déployable |

Pour mémoire, la première version sécurisée, construite sans `apk upgrade` ni `apk del`, affichait 14C / 32H / 14M / 6L (voir l'analyse ci-dessous).

### Analyse des vulnérabilités

Après le premier passage à l'image Nginx, le nombre total de vulnérabilités a baissé (66 contre 91), mais le nombre de failles **critiques** a augmenté (14 contre 7).

Docker Scout explique pourquoi : l'image `nginx-unprivileged` utilisée datait de plusieurs semaines. Plusieurs de ses paquets (`nginx`, `curl`, `libxml2`, `libxslt`, `expat`, `brotli`, `openssl`) avaient reçu des correctifs depuis. L'image embarquait aussi des modules inutiles pour un site statique (`geoip`, `image-filter`, `njs`, `xslt`).

Toutes ces failles étant corrigeables, le Dockerfile final ajoute deux actions :
- `apk upgrade` applique les correctifs de sécurité disponibles ;
- `apk del` supprime les modules et outils inutiles, ce qui réduit encore la surface d'attaque.

Résultat : on passe de 86 à **26 paquets** et de 66 à **10 vulnérabilités** (5C / 2H / 3M / 0L), soit 89 % de failles en moins par rapport à l'image initiale. Les 5 failles critiques restantes ont un correctif annoncé, mais il n'est pas encore disponible dans les dépôts Alpine de cette version. Elles disparaîtront à la prochaine reconstruction de l'image, une fois la mise à jour publiée.

**Enseignement :** changer d'image de base ne suffit pas. Même une image officielle et minimale accumule des vulnérabilités avec le temps. Il faut **scanner régulièrement** les images et les **reconstruire** pour intégrer les correctifs.

### Questions d'analyse

**Quelles modifications ont apporté les améliorations les plus significatives ?**
Le **build multi-étapes** combiné au **changement d'image de base** a eu le plus d'effet. L'application est un site statique : une fois construite, elle n'a plus besoin de Node.js. En servant le dossier `dist` avec Nginx, on supprime Node.js, npm, yarn, `serve` et leurs dépendances, soit 246 paquets et la majorité des CVE. Viennent ensuite `apk upgrade` (14 → 5 CVE critiques) et le passage à un **utilisateur non root**, qui limite l'impact d'une éventuelle compromission.

**La taille de l'image a-t-elle diminué ? Pourquoi ?**
Oui : de 221 Mo à 104 Mo sur disque (−53 %), et de 55 Mo à 29 Mo compressés (−47 %). Le build multi-étapes ne copie dans l'image finale que le résultat du build (`COPY --from=build /app/dist`). Le runtime Node.js, npm, les dépendances et les outils de build restent dans l'étape de build, qui n'est pas publiée. `apk del` a aussi retiré des modules Nginx inutiles, et `.dockerignore` empêche d'envoyer des fichiers superflus dans le contexte de build.

**Quelles recommandations concernent le Dockerfile, et lesquelles concernent l'exécution du conteneur ?**

| Dans le Dockerfile (construction de l'image) | Au lancement du conteneur (`docker run`) |
|---|---|
| Image de base minimale | Limitation des ressources (`--memory`, `--cpus`, `--pids-limit`) |
| Utilisateur non root (`USER 101`) | Restriction des privilèges (`--cap-drop ALL`, `--security-opt no-new-privileges`) |
| Tags d'images précis | Système de fichiers en lecture seule (`--read-only`, `--tmpfs /tmp`) |
| `COPY` plutôt que `ADD` | Ne pas monter le socket Docker (`-v /var/run/docker.sock`) |
| Build multi-étapes | |
| `.dockerignore` (utilisé au build) | |
| Vérification régulière : scan de l'image (Docker Scout) | Vérification régulière : contrôle des conteneurs en cours d'exécution |

**Votre image est-elle désormais totalement sécurisée ? Justifiez votre réponse.**
Non. Aucune image n'est totalement sécurisée. Il reste 10 CVE (dont 5 critiques), dont les correctifs ne sont pas encore disponibles dans Alpine. De nouvelles failles sont découvertes chaque jour : une image saine aujourd'hui sera vulnérable dans quelques semaines, comme l'a montré la première version Nginx. Par ailleurs, les protections d'exécution (`--read-only`, `--cap-drop`, limites…) dépendent de la commande `docker run` et disparaissent si le conteneur est lancé sans elles. Enfin, l'image n'a ni SBOM ni attestation de provenance, et la configuration Nginx n'est pas durcie (pas d'en-têtes de sécurité, pas de HTTPS). La surface d'attaque est fortement réduite, mais la sécurité reste un processus continu.

**Quelles mesures complémentaires proposeriez-vous pour un déploiement en production ?**
- Intégrer un scan automatique en CI/CD (GitHub Actions + Docker Scout ou Trivy) qui bloque le déploiement si une CVE critique corrigeable est détectée.
- Reconstruire l'image régulièrement (par exemple chaque semaine) pour intégrer les correctifs.
- Utiliser une **Docker Hardened Image** (voir mission 4) : quasiment aucune CVE, SBOM et provenance fournis.
- Épingler les images par digest (`@sha256:...`), générer un SBOM et une attestation de provenance (`--sbom=true --provenance=mode=max`), et signer les images.
- Définir les options de sécurité dans un `docker-compose.yml` ou dans un manifeste Kubernetes (`securityContext`), plutôt qu'à la main.
- Placer le conteneur derrière un reverse proxy avec HTTPS, et ajouter des en-têtes de sécurité HTTP (CSP, HSTS, X-Frame-Options).
- Activer les profils seccomp / AppArmor, isoler le conteneur dans un réseau dédié et ajouter un `HEALTHCHECK`.
- Centraliser les logs et surveiller les conteneurs en cours d'exécution.

---

## Mission 4 : utiliser une Docker Hardened Image (DHI)

### Atelier pratique

L'atelier officiel (https://docs.docker.com/guides/lab-dhi-node/) se lance en local :

```powershell
docker compose -p labspace -f oci://dockersamples/labspace-dhi-node up -d
# Ouvrir http://localhost:3030 et suivre les modules
docker compose -p labspace down   # à la fin de l'atelier
```

L'atelier fait migrer une application Node.js de `node:24-trixie-slim` vers une DHI, avec un build multi-étapes utilisant les variantes `-dev` (pour le build) et runtime (pour l'image finale). Il compare ensuite les CVE, la taille et la conformité aux politiques avec Docker Scout, puis inspecte les attestations (SBOM, provenance SLSA, VEX).

### Application au projet quiz-docker (`Dockerfile.dhi`)

```dockerfile
# ---- Etape 1 : build avec la DHI Node (variante dev : shell + npm) ----
FROM dhi.io/node:24-dev AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# ---- Etape 2 : runtime avec la DHI Nginx (sans shell, non root) ----
FROM dhi.io/nginx:1.31.10-debian13
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 8080
```

```powershell
docker login dhi.io
docker build -f Dockerfile.dhi -t quiz-docker:dhi .
docker run -d --name quiz-dhi -p 3003:8080 --memory 128m --cpus 0.5 --pids-limit 100 `
  --read-only --tmpfs /tmp:mode=1777 --tmpfs /var/cache/nginx:mode=1777 --tmpfs /run/nginx:mode=1777 `
  --cap-drop ALL --security-opt no-new-privileges quiz-docker:dhi
docker image inspect quiz-docker:dhi --format '{{.Config.User}}'   # 65532 (nonroot)
docker images quiz-docker
docker scout quickview quiz-docker:dhi
```

Remarque : avec `--read-only`, le premier lancement a échoué avec `mkdir() "/var/cache/nginx/client_temp" failed (13: Permission denied)`. Un `tmpfs` appartient à root par défaut, et l'utilisateur non root de la DHI (UID 65532) ne pouvait pas y écrire. L'option `mode=1777` corrige le problème, ce qui confirme que l'image DHI fonctionne entièrement sans privilèges.

### Comparaison des trois images

| Critère | Initiale (`node:18-alpine`) | Sécurisée (`nginx-unprivileged`) | DHI (`dhi.io/nginx`) |
|---|---|---|---|
| Image de base finale | `node:18-alpine` | `nginxinc/nginx-unprivileged:1.30-alpine` | `dhi.io/nginx:1.31.10-debian13` |
| Utilisateur | root | nginx (UID 101) | nonroot (UID 65532) |
| Shell et gestionnaire de paquets | Oui | Oui (`sh`, `apk`) | Non (variante runtime) |
| Taille sur disque | 221 Mo | 104 Mo | **50 Mo** |
| Taille compressée | 55 Mo | 29 Mo | **11 Mo** |
| Nombre de paquets | 272 | 26 | 31 |
| CVE critiques | 7 | 5 | **0** |
| CVE élevées | 42 | 2 | 3 |
| CVE moyennes | 31 | 3 | 2 |
| CVE faibles | 11 | 0 | 25 |
| CVE critiques / élevées corrigeables | 49 | 7 | **0** |
| Note Docker Scout | D (50 %) | C (56 %) | **B (78 %)** |
| Politiques Scout respectées | 3 / 7 | 4 / 7 | **5 / 7** |
| SBOM, provenance, VEX de l'image de base | Non | Non | Oui (fournis par Docker) |
| Suivi des correctifs | Manuel | Manuel (`apk upgrade`) | Assuré par Docker |

### Conclusion de la mission 4

La Docker Hardened Image donne le meilleur résultat sur presque tous les critères. L'image fait 50 Mo (−52 % par rapport à l'image sécurisée, −77 % par rapport à l'image initiale), n'a **aucune CVE critique** et **aucune CVE critique ou élevée corrigeable**, et obtient la meilleure note Docker Scout (B, 78 %). Elle tourne en non-root par défaut et ne contient ni shell ni gestionnaire de paquets. Même en cas de faille, un attaquant ne dispose donc d'aucun outil dans le conteneur.

Le total de CVE (30) est plus élevé qu'avec l'image sécurisée (10). Mais il s'agit surtout de CVE **faibles** (25) de la base Debian, pour lesquelles aucun correctif n'existe. Docker les documente dans ses attestations VEX, qui indiquent si une CVE est réellement exploitable. Le critère pertinent reste l'absence de CVE critique et de faille corrigeable non traitée.

Deux politiques restent non respectées : les attestations de supply chain (notre image construite à partir de la DHI doit générer ses propres SBOM et provenance, avec `--sbom=true --provenance=mode=max`) et les licences copyleft (critère de conformité juridique, pas de sécurité).

Contreparties : il faut s'authentifier sur `dhi.io`, et l'absence de shell rend le débogage plus difficile (`docker exec ... sh` est impossible). Les dossiers en écriture doivent aussi être explicitement autorisés pour l'utilisateur non root.

**Recommandation :** pour la production, la Docker Hardened Image est le meilleur choix. Elle réduit au minimum la surface d'attaque et délègue à Docker le suivi des correctifs de sécurité.

---

## Bilan : objectifs du TP atteints

| Objectif | Ce qui a été réalisé | Preuve |
|---|---|---|
| Identifier les risques de sécurité d'une image et d'un conteneur | Audit de l'image initiale : Node.js 18 en fin de vie, exécution en root, outils inutiles, 91 CVE, aucune limite au lancement | Mission 1 : tableau « État initial » et fichiers `audit-*-initial.txt` |
| Appliquer les bonnes pratiques de création d'images | Tags précis, `.dockerignore`, `COPY` au lieu de `ADD`, build multi-étapes, utilisateur non root, mise à jour des paquets (`apk upgrade`) | Mission 2 : Dockerfile sécurisé et tableau des recommandations |
| Réduire la surface d'attaque | Suppression de Node.js, npm, yarn et `serve` dans l'image finale, puis des modules Nginx inutiles et de `curl` : on passe de 272 à 26 paquets et de 221 à 104 Mo | Mission 3 : lignes taille et paquets du tableau comparatif |
| Contrôler les privilèges et les ressources | Utilisateur `nginx`, `--cap-drop ALL`, `no-new-privileges`, `--read-only`, limites mémoire / CPU / processus, socket Docker non exposé | Commande `docker run` et `docker exec quiz-secure whoami` (renvoie `nginx`) |
| Utiliser un outil d'analyse de vulnérabilités | Docker Scout : `quickview`, `cves` et `compare` sur les deux images | Fichiers d'audit, résultats D (50 %) puis C (56 %) |
| Comparer et justifier les améliorations | 91 CVE puis 10 (−89 %), avec l'analyse de la régression intermédiaire (14 CVE critiques) et sa correction | Mission 3 : tableau comparatif et analyse des vulnérabilités |

---

## Synthèse

### Améliorations apportées

L'image initiale embarquait tout l'environnement de développement (Node.js 18 en fin de vie, npm, yarn, `serve`), s'exécutait en root et contenait 91 vulnérabilités. Pour servir un simple site statique, c'était une surface d'attaque disproportionnée.

L'image sécurisée repose sur un build multi-étapes : Node.js sert uniquement à construire le site, et l'image finale ne contient qu'un serveur Nginx non root et les fichiers du site. Les paquets ont été mis à jour, et les modules inutiles supprimés. Résultat : une image deux fois plus légère, 26 paquets au lieu de 272, et 10 vulnérabilités au lieu de 91 (−89 %).

Au lancement, le conteneur est confiné : utilisateur non privilégié, aucune capacité Linux, pas d'élévation de privilèges, système de fichiers en lecture seule, limites de mémoire, de CPU et de processus, et aucun accès au socket Docker.

### Risques résiduels

| Risque résiduel | Explication | Piste de correction |
|---|---|---|
| 5 CVE critiques et 2 élevées | Des correctifs existent, mais ils ne sont pas encore publiés dans les dépôts Alpine de l'image de base | Rescanner et reconstruire régulièrement, par exemple avec un scan automatique en CI (GitHub Actions + Docker Scout) |
| Protections appliquées seulement au lancement | Les options `--read-only`, `--cap-drop`, `--memory`… ne sont pas dans l'image : un `docker run` sans ces options annule ces protections | Les définir dans un fichier `docker-compose.yml` ou dans la configuration de l'orchestrateur |
| Tags modifiables | `1.30-alpine` peut pointer vers une autre image demain | Épingler l'image par son digest (`@sha256:...`) |
| Attestations supply chain absentes | Pas de SBOM ni de provenance : l'origine de l'image ne peut pas être vérifiée (politique Scout non respectée) | `docker build --sbom=true --provenance=mode=max` et signature de l'image |
| Configuration Nginx par défaut | Pas d'en-têtes de sécurité HTTP (CSP, X-Frame-Options…), et pas de HTTPS | Ajouter une configuration Nginx durcie, avec TLS assuré par un reverse proxy |
| Image publique sur Docker Hub | Tout le monde peut la télécharger et l'analyser | Acceptable ici (aucun secret dans l'image), à surveiller pour une application réelle |

**En résumé :** la sécurité d'une image n'est jamais acquise définitivement. Elle demande une vérification et une reconstruction régulières.