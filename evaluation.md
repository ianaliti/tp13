TP13 : Évaluation Docker
Contexte
Vous allez construire une stack Docker complète autour d'une API Node.js en appliquant l'ensemble des compétences vues pendant le cours : Dockerfile propre, Compose multi-services, registry privé, load-balancing Nginx, sécurité, observabilité, gestion des volumes et pipeline CI/CD.

Le rendu est individuel.

Modalités de rendu
Vous rendez un dépôt Git contenant l'ensemble de vos fichiers. Le lien vers le dépôt est déposé sur Teams.

Le dépôt doit contenir un fichier README.md à la racine qui sert de rapport de rendu. C'est ce fichier que je vais ouvrir en premier. Il doit présenter chaque partie dans l'ordre, avec les captures d'écran intégrées directement dans le Markdown, de façon à ce que la correction puisse se faire entièrement depuis l'interface web de GitHub/GitLab sans avoir à cloner le projet.

Le carnet de notes est rendu séparément, au format de votre choix (Notion, PDF...), directement dans le devoir Teams, attention toutefois, je ne dois avoir de problème à ouvrir votre carnet (ex: pas de lien vers un Notion privé) sinon je noterais 0/10. Le carnet doit être relativement agréable à lire. Je ne demande pas un rapport de 100 pages, mais un effort de mise en forme est attendu (titres, sous-titres, etc.) pour que la lecture soit fluide.

Consignes
Partie 1 — API & Dockerfile
Vous devez créer une API Node.js avec Express dans un dossier api/.

L'API expose deux routes : - GET / répond en JSON avec le hostname du conteneur (os.hostname()), la valeur de la variable d'environnement PET (qui vaut dog ou cat) et un compteur du nombre de requêtes reçues depuis le démarrage du conteneur. - GET /healthz répond avec { "status": "ok" } et le code HTTP 200. Cette route est utilisée par le Healcheck Docker pour vérifier que l'application est opérationnelle.

Le compteur est fait grâve a library prom-client et exposé sur la route /metrics pour être scrappé par Prometheus plus tard. Le Dockerfile doit utiliser node:20-alpine comme image de base, copier uniquement les fichiers nécessaires (pas de node_modules, pas de .env), installer les dépendances avec npm install --only=production, et faire tourner l'application sous un utilisateur non-root. Un fichier .dockerignore doit être présent pour exclure node_modules, .env et .git du contexte de build. Le Healcheck doit cibler la route /healthz et être configuré avec un interval, un timeout et un start_period adaptés.

Critère évalué	Points
Routes / et /healthz fonctionnelles : GET / retourne hostname, PET et compteur ; GET /healthz répond 200 avec { "status": "ok" }	1,50
Dockerfile : node:20-alpine, utilisateur non-root, .dockerignore présent, npm install --only=production	0,50
Dockerfile : Healcheck configuré sur /healthz avec des paramètres adaptés	1,00
Partie 2 — Registry privé
Vous devez déployer un registry Docker privé en local avec une interface web, pousser votre image API dessus, puis l'utiliser depuis votre stack principale.

Le registry (registry:2) et son interface web (joxit/docker-registry-ui) sont décrits dans un fichier docker-compose.registry.yml séparé, lancé indépendamment de la stack principale. Votre image API est taguée et poussée vers localhost:5000/mon-api:1.0.0 (ou équivalent).

Rendu attendu pour cette partie : une capture d'écran de l'interface web (http://localhost:8080) montrant votre image listée dans le registry, et le champ image: de votre docker-compose.yml principal pointant vers localhost:5000/... (visible dans le fichier versionné dans le repo).

Critère évalué	Points
Registry et interface web déployés et fonctionnels (capture de l'interface web avec l'image listée)	1,00
Image API poussée vers le registry privé et utilisée dans le docker-compose.yml principal	1,00
Partie 3 — Stack Compose & Nginx
Vous devez créer un fichier docker-compose.yml qui orchestre trois services sur un réseau Docker personnalisé :

cat : une instance de votre API avec la variable d'environnement PET=cat. Ce service n'expose aucun port directement à l'hôte, il est uniquement accessible depuis le réseau interne Docker.
dog : une instance de votre API avec la variable d'environnement PET=dog. Même configuration réseau que cat.
nginx : un reverse proxy et load balancer qui écoute sur le port 80 (ou équivalent) de l'hôte et distribue le trafic vers cat et dog.
Les services cat et dog doivent démarrer uniquement quand leur healthcheck est passé. Nginx doit dépendre de cat et dog avec la condition service_healthy.

La configuration Nginx doit :

distribuer les requêtes sur GET / en round-robin entre cat et dog (les deux dans le même upstream) ;
rediriger GET /cat exclusivement vers le service cat ;
rediriger GET /dog exclusivement vers le service dog.
Critère évalué	Points
docker-compose.yml avec les trois services sur un réseau personnalisé, variables d'environnement correctes, aucun port exposé pour cat et dog	2,00
Configuration Nginx avec un upstream round-robin sur / équilibrant entre les deux services	1,0
Locations /cat et /dog redirigeant exclusivement vers le bon service	0,5
depends_on avec condition: service_healthy sur cat et dog	0,5
Partie 4 — Sécurité
Toutes les valeurs configurables de votre stack doivent passer par des variables d'environnement définies dans un fichier .env. Cela inclut au minimum le port de l'API, la valeur de PET pour chaque service, et le port exposé par Nginx. Aucune valeur ne doit apparaître en dur dans le docker-compose.yml ni dans le Dockerfile (hors valeurs par défaut techniques).

Vous devez scanner votre image API avec Trivy et justifier votre choix d'image de base : pourquoi node:20-alpine plutôt que node:latest ? Quel est l'impact sur le nombre de CVE ?

Le Dockerfile doit copier uniquement les fichiers strictement nécessaires : d'abord package*.json pour profiter du cache Docker lors des npm install, puis le reste du code. Le dossier node_modules ne doit jamais entrer dans le contexte de build.

Rendu attendu pour cette partie : une capture d'écran d'une partie de la sortie de trivy image <votre-image> dans le dossier captures/, et la justification du choix d'image.

Critère évalué	Points
Scan Trivy effectué et capture jointe, justification du choix d'image	0,5
Toutes les valeurs configurables passent par le fichier .env (port, PET, etc.), sans aucune valeur en dur dans les fichiers de config	1,0
Dockerfile : COPY package*.json en premier, puis COPY . ., sans node_modules dans le contexte de build	0,5
Partie 5 — Validation de la stack
Rendu attendu pour cette partie : quatre captures d'écran dans le dossier captures/, correspondant aux critères ci-dessous.

Critère évalué	Points
docker compose ps montre tous les services en état Up (healthy)	1
En rafraîchissant http://localhost/ plusieurs fois, les hostnames dans la réponse JSON alternent entre les deux conteneurs (deux appels successifs côte à côte)	1
/cat répond avec PET: cat, /dog avec PET: dog, et les compteurs diffèrent entre les deux services	1
Partie 6 — Questions théoriques
Ces trois questions doivent être répondues dans le README.md de votre dépôt (comme le reste des questions de cette évaluation), dans une section dédiée.

Question Swarm (1 pt) : Expliquez la différence entre docker compose up et docker stack deploy. Pourquoi la directive build: n'est-elle pas utilisable dans une stack déployée en mode Swarm ?
Question Secrets (1 pt) : Expliquez la différence entre passer un mot de passe via une variable d'environnement et via un Docker Secret. Dans quel fichier le secret est-il accessible à l'intérieur du conteneur, et comment le lire depuis du code Node.js ?
Question Backup (1 pt) : Dans une architecture Docker en production, quels éléments faut-il impérativement sauvegarder pour pouvoir reconstruire entièrement la stack après une panne ? Distinguez ce qui est recréable automatiquement de ce qui est irremplaçable.
Partie 7 — Observabilité & Production
A la suite des deux premières parties, vous devez ajouter à votre stack de quoi monitorer votre application. Le but ici est de faire un dashboard grafana complet avec un node exporter, cadvisor et le client prometheus pour scrapper les métriques de votre application (exposées sur /metrics grâce à prom-client).

Critère évalué	Points
Prometheus et Grafana ajoutés à la stack et accessibles	0,5
Dashboard Grafana provisionné automatiquement (fichier de provisioning versionné dans le dépôt)	1
Fichier docker-compose.prod.yml en override avec limites CPU et RAM sur chaque service	1,00
Portainer ajouté à la stack et accessible	0,5
Partie 8 — Volumes
Votre stack utilise des volumes à plusieurs endroits (registry, Grafana, configuration Nginx et Prometheus).

Vous devez : - Utiliser des volumes nommés pour les données persistantes (données Grafana, données du registry). - Utiliser des bind mounts pour les fichiers de configuration injectés depuis l'hôte (prometheus.yml, default.conf Nginx, fichiers de provisioning Grafana).

Rendu attendu pour cette partie : une capture de docker volume ls montrant les volumes nommés de votre stack, et une capture de docker volume inspect <volume> sur l'un d'eux.

Critère évalué	Points
Volumes nommés pour les données et bind mounts pour les configs, cohérents dans tous les fichiers Compose	1
Capture de docker volume ls et docker volume inspect, justification du choix dans le README.md	1
Partie 9 — CI/CD avec GitHub Actions
Vous devez créer un fichier .github/workflows/docker.yml qui automatise le build et le contrôle de sécurité de votre image API à chaque push.

Le workflow doit : - Se déclencher sur chaque push sur la branche main. - Builder l'image Docker de votre API. - Scanner l'image avec Trivy et faire échouer le pipeline si des CVE de sévérité CRITICAL sont détectées. - Pousser l'image vers un registry public au choix : Docker Hub (docker.io/<user>/mon-api:git-abc1234) ou GitHub Container Registry (ghcr.io/<user>/mon-api:git-abc1234), avec un tag basé sur le SHA court du commit.

Docker Hub nécessite de configurer deux secrets GitHub : DOCKERHUB_USERNAME et DOCKERHUB_TOKEN. ghcr.io utilise le GITHUB_TOKEN fourni automatiquement par GitHub Actions, aucun secret supplémentaire n'est nécessaire.

Rendu attendu pour cette partie : le fichier .github/workflows/docker.yml versioné dans le dépôt, et une capture d'écran de l'onglet Actions de GitHub montrant le pipeline en succès (ou en échec justifié si des CVE CRITICAL sont présentes).

Critère évalué	Points
Workflow déclenché sur push sur main, étape de build fonctionnelle (capture de l'onglet Actions)	1
Scan Trivy intégré dans le pipeline, le pipeline échoue si des CVE CRITICAL sont détectées	1
Image poussée vers Docker Hub ou ghcr.io avec un tag SHA Git	1
Partie 10 — Déploiement sur VPS
Si l'ensemble de votre stack est déployée et accessible depuis un VPS (serveur distant), vous obtenez les points de cette partie. Fournissez l'URL ou l'IP publique dans le README.md, ainsi qu'une capture montrant la stack fonctionnelle depuis le serveur.

Critère évalué	Points
Stack complète déployée sur VPS, accessible depuis un navigateur externe (URL fournie dans le README)	2
docker compose ps exécuté sur le VPS montrant tous les services Up (healthy) (capture depuis le serveur)	1
Partie 11 — Clarté & lisibilité du README
Le README.md est le fichier rendu qui centralise votre travail. Il doit être structuré, lisible et permettre de retrouver rapidement les preuves attendues.

Critère évalué	Points
Structure claire : titres hiérarchisés, sections dans l'ordre des parties, captures bien intégrées et légendées	1
Rédaction soignée : phrases complètes, mise en forme visible, absence de fautes grossières	1
Arborescence attendue
evaluation/
  .github/
    workflows/
      docker.yml
  api/
    app.js
    [...]
  nginx/
    [...]
  monitoring/
    [...]
  docker-compose.yml
  docker-compose.registry.yml
  docker-compose.prod.yml
  README.md
  captures/
Grille de notation récapitulative
Note 1 — Devoir
Partie	Critère	Points
1 — API & Dockerfile	Routes / et /healthz fonctionnelles avec les bonnes réponses	1,5
Dockerfile : node:20-alpine, non-root, .dockerignore, --only=production	0,5
Dockerfile : healthcheck sur /healthz avec paramètres adaptés	1
2 — Registry privé	Registry + UI déployés, capture de l'interface avec l'image listée	1
Image poussée vers le registry et utilisée dans docker-compose.yml	1
3 — Stack Compose & Nginx	3 services, réseau perso, variables d'env, ports internes uniquement	2
Upstream round-robin sur / entre cat et dog	1
Locations /cat et /dog vers le bon service	0,5
depends_on: condition: service_healthy sur cat et dog	0,5
4 — Sécurité	Scan Trivy + capture, justification image dans le carnet	0,5
Toutes les valeurs via .env, aucune valeur en dur	1
COPY package*.json en premier, pas de node_modules dans le contexte	0,5
5 — Validation	docker compose ps tous Up (healthy)	1
Load balancing visible (hostnames alternés)	1
/cat → PET: cat, /dog → PET: dog + compteurs différents	1
6 — Questions théoriques	Question Swarm (dans le README)	1
Question Secrets (dans le README)	1
Question Backup (dans le README)	1
7 — Observabilité & Production	Prometheus + Grafana accessibles	1
Dashboard Grafana provisionné automatiquement	0,5
docker-compose.prod.yml avec limites CPU/RAM	1
Portainer dans la stack	0,5
8 — Volumes	Volumes nommés pour les données, bind mounts pour les configs, cohérents dans la stack	1
Capture docker volume ls + inspect, justification dans le README	1
9 — CI/CD GitHub Actions	Workflow push main + build fonctionnel (capture Actions)	1
Scan Trivy en CI, pipeline échoue si CVE CRITICAL	1
Push vers Docker Hub ou ghcr.io avec tag SHA Git	1
10 — VPS	Stack déployée sur VPS, accessible publiquement (URL dans le README)	2
docker compose ps sur le VPS, tous Up (healthy)	1
11 — Clarté du README	Structure, sections dans l'ordre, captures légendées	1
Rédaction soignée, mise en forme visible	1
Total devoir		30
Note 2 — Carnet
Note	Critères
10/10	Carnet complet. Réponses claires, précises et bien rédigées. Justifications des choix techniques présentes.
8/10	Carnet complet. Quelques imprécisions ou manque de détails mineurs.
6/10	Carnet globalement correct, mais plusieurs points sont peu développés ou manquants.
4/10	Carnet partiellement rempli. Réponses incomplètes ou peu détaillées.
2/10	Carnet très insuffisant. Peu d'efforts visibles.
0/10	Carnet non rendu ou lien inaccessible.