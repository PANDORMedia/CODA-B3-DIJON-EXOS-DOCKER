# J1 Docker · Cahier d'exercices

Oct 11, 2026 · @Sean Red

## Mode d'emploi

Objectif : tu cherches, tu testes, tu casses, et on formalise ensuite.

Chaque exercice a un **objectif**, une **mission** et une **vérification**. Il est réussi quand la vérification donne le résultat attendu, pas avant.&#32;

Ressources autorisées : `docker --help`, `docker <commande> --help`, [docs.docker.com](https://docs.docker.com), ton voisin.

### Avant de commencer

- [ ] `docker version` affiche une partie Client et une partie Server
- [ ] `docker info` s'exécute sans erreur
- [ ] Le dossier `nerdnotes/` du repo starter est décompressé sur ta machine
- [ ] Sous Windows :  WSL ou Git Bash, car les vérifications utilisent `curl` et `grep`

## 8 défis exploratoires

### Défi 1 · Premier contact

**Objectif** : lancer ton premier conteneur.

**Mission** : lance l'image `hello-world` et lis le message affiché en entier.

**Vérification** : `docker images` liste `hello-world`.

**À noter** : d'où vient cette image ? Que change le deuxième lancement ?

### Défi 2 · Une machine dans la machine

**Objectif** : ouvrir un shell dans un conteneur.

**Mission** : lance un conteneur `ubuntu` avec un terminal interactif. Dedans, crée le fichier `/preuve.txt` contenant ton prénom, puis quitte. Lance un nouveau conteneur `ubuntu` : ton fichier existe-t-il ? Récupère-le quand même sur ta machine.

**Vérification** : `docker ps -a` montre au moins deux conteneurs `ubuntu` arrêtés, et `cat preuve.txt` sur ta machine affiche ton prénom.

**Indice** : `docker cp` fonctionne aussi sur un conteneur arrêté.

**À noter** : pourquoi le second conteneur n'a pas ton fichier ?

### Défi 3 · Un vrai serveur

**Objectif** : rendre un serveur web accessible depuis ta machine.

**Mission** : lance `nginx` pour que sa page soit accessible sur http://localhost:8080.

**Vérification** : `curl -s localhost:8080 | grep title` affiche `<title>Welcome to nginx!</title>`.

**Indice** : dans le conteneur, nginx écoute sur le port 80.

**À noter** : dans `-p A:B`, lequel des deux est ta machine ?

### Défi 4 · Rends-moi mon terminal

**Objectif** : faire tourner un conteneur en arrière-plan.

**Mission** : relance nginx en arrière-plan, nommé `web`, toujours sur le port 8080. Ton terminal doit rester libre.

**Vérification** : `docker ps --filter name=web` affiche `web` avec le statut `Up`.

### Défi 5 · Qui frappe à la porte ?

**Objectif** : lire les logs d'un conteneur.

**Mission** : dans un terminal, suis en direct les logs de `web`. Dans un autre, envoie quelques requêtes, puis demande la page `/nexistepas`.

**Vérification** : une ligne `GET /nexistepas` avec le code `404` apparaît dans les logs pendant que tu regardes.

### Défi 6 · Entre dans la boîte

**Objectif** : exécuter une commande dans un conteneur qui tourne.

**Mission** : sans arrêter `web`, remplace sa page d'accueil par `<h1>Salut, je suis Prénom</h1>`.

**Vérification** : `curl -s localhost:8080` affiche ton `<h1>`.

**Indice** : nginx sert ses fichiers depuis `/usr/share/nginx/html`. Il n'y a pas d'éditeur dans l'image, `echo` suffit.

**À noter** : que devient ta page si tu supprimes `web` et que tu le recrées ?

### Défi 7 · Le conteneur qui meurt

**Objectif** : diagnostiquer un conteneur qui s'arrête.

**Mission** : lance `docker run -d --name mystere postgres:16`. Il s'arrête tout seul. Trouve pourquoi et fais-le tourner.

**Vérification** : `docker exec mystere pg_isready` affiche `accepting connections`.

**Indice** : un conteneur arrêté garde ses logs.

**À noter** : pourquoi `docker start` ne suffit pas à le réparer ?

### Défi 8 · Grand ménage

**Objectif** : gérer l'espace disque de Docker.

**Mission** : mesure l'espace utilisé par Docker. Supprime en une seule commande tous les conteneurs arrêtés, puis supprime les images `hello-world` et `ubuntu`. Mesure à nouveau. `web` et `mystere` doivent rester en vie.

**Interdit** : `docker system prune -a`. Il effacerait les images préchargées pour cet après-midi.

**Vérification** : `docker system df` affiche moins d'espace qu'avant, et `docker ps` montre toujours `web` et `mystere`.

**À noter** : pourquoi Docker refuse de supprimer l'image `nginx` ?

### Bonus · Deux serveurs

**Mission** : lance un second nginx nommé `web2` sans arrêter `web`.

**Vérification** : `curl -s localhost:8080` et `curl -s localhost:8081` répondent tous les deux.

## Conteneuriser nerdnotes

`nerdnotes` est une petite API Python de prise de notes. C'est l'application qu'on va faire grandir toute la semaine, jusqu'à Kubernetes.

| Route | Méthode | Ce qu'elle fait |
| --- | --- | --- |
| `/` | GET | Nom, version, hostname et propriétaire de l'instance |
| `/notes` | GET | Liste les notes |
| `/notes` | POST | Ajoute une note : `{"text": "..."}` |
| `/health` | GET | État de santé de l'app |
| `/metrics` | GET | Métriques au format Prometheus |

### 3.1 · Première image

**Objectif** : écrire ton premier Dockerfile.

**Mission** : dans `nerdnotes/`, écris un `Dockerfile` qui part de `python:3.12`, copie le code, installe les dépendances de `requirements.txt` et lance l'app avec `gunicorn -b 0.0.0.0:5000 app:app`. Construis l'image `nerdnotes:v1`.

**Vérification** : `docker images nerdnotes` affiche le tag `v1`.

**Indice** : cinq instructions suffisent. Pas besoin de lire le Python : les dépendances s'installent avec pip install -r requirements.txt, comme une liste de paquets apt.

### 3.2 · Lance-la

**Objectif** : faire tourner ta propre image.

**Mission** : lance `nerdnotes:v1` en arrière-plan, nommé `notes`, accessible sur le port 5000 de ta machine.

**Vérification** : `curl -s localhost:5000/` renvoie un JSON contenant `"app": "nerdnotes"`.

**À noter** : compare le champ `hostname` avec la sortie de `docker ps`. À quoi correspond-il ?

### 3.3 · Tes premières notes

**Objectif** : utiliser l'API dans son conteneur.

**Mission** : ajoute deux notes, puis liste-les.

```bash
curl -s -X POST localhost:5000/notes \
  -H 'Content-Type: application/json' \
  -d '{"text": "ma première note"}'
```

Ensuite, supprime `notes` et recrée-le avec la même commande qu'en 3.2. Liste à nouveau les notes.

**Vérification** : avant la suppression, `curl -s localhost:5000/notes` renvoie tes deux notes.

**À noter** : que sont devenues tes notes ? On y répond demain.

### 3.4 · Configurer sans reconstruire

**Objectif** : paramétrer un conteneur au lancement.

**Mission** : l'app lit la variable d'environnement `OWNER`. Lance un second conteneur `notes-perso` sur le port 5001, dont le propriétaire est ton prénom. Interdiction de toucher au code ou de reconstruire l'image.

**Vérification** : `curl -s localhost:5001/` contient `"owner": "<ton prénom>"`, et `curl -s localhost:5000/` contient toujours `"owner": "anonyme"`.

## Optimiser l'image

Ton image marche. Maintenant elle doit être légère, rapide à construire et sûre. Chaque exercice produit un nouveau tag : garde les anciens pour comparer.

### 4.1 · Combien ça pèse ?

**Objectif** : mesurer le poids d'une image.

**Mission** : note la taille de `nerdnotes:v1`. Construis `nerdnotes:v2` en changeant uniquement l'image de départ pour `python:3.12-slim`. Compare.

| Tag | Image de départ | Taille mesurée |
| --- | --- | --- |
| v1 | `python:3.12` |  |
| v2 | `python:3.12-slim` |  |

**Vérification** : `docker images nerdnotes` montre `v2` sous les 300 Mo.

**À noter** : qu'est-ce qui a disparu entre les deux ? L'app en a-t-elle besoin ?

### 4.2 · Le cache

**Objectif** : comprendre le cache de build.

**Mission** : modifie le champ `version` dans `app.py`, puis reconstruis `v2` avec `time docker build -t nerdnotes:v2 .`. Repère l'étape qui se relance alors que `requirements.txt` n'a pas changé. Réorganise ton Dockerfile pour qu'elle ne se relance plus quand seul le code change.

**Vérification** : après une nouvelle modification de `app.py`, le build affiche `CACHED` sur l'étape `RUN pip install`.

### 4.3 · Le contexte de build

**Objectif** : ne pas envoyer n'importe quoi à Docker.

**Mission** : crée un gros fichier inutile dans `nerdnotes/`, puis reconstruis. Observe la ligne `transferring context`. Écris un `.dockerignore` pour corriger.

```bash
dd if=/dev/zero of=dump.bin bs=1M count=200
```

**Vérification** : `transferring context` retombe à quelques Ko, et `docker run --rm nerdnotes:v2 ls` ne liste pas `dump.bin`.

### 4.4 · Qui est aux commandes ?

**Objectif** : ne plus tourner en root.

**Mission** : lance `docker exec notes whoami`. Construis `nerdnotes:v3` qui fait tourner l'app avec un utilisateur dédié nommé `app`.

**Vérification** : `docker run --rm nerdnotes:v3 whoami` affiche `app`, et un conteneur lancé depuis `v3` répond toujours sur `/`.

**À noter** : que pourrait faire un attaquant qui prend le contrôle de l'app quand elle tourne en root ?

### Bonus 4.5 · Le conteneur se surveille

**Mission** : ajoute une instruction `HEALTHCHECK` qui interroge `/health` toutes les 10 secondes. Attention : `curl` n'existe pas dans l'image slim.

**Vérification** : `docker ps` affiche `(healthy)` dans la colonne STATUS.

### Bonus 4.6 · Multi-stage

**Mission** : installe les dépendances dans une première étape `builder`, puis ne copie que le résultat dans l'image finale.

**Vérification** : l'image finale démarre et répond sur `/`. Compare sa taille avec `v3`, et explique le résultat.

## Réserve : déployer des apps existantes

Pour ceux qui ont fini. Pas de Dockerfile ici : en entreprise, la plupart du temps, tu déploies des images écrites par d'autres. Tes seules sources : la page de l'image sur Docker Hub et `docker image inspect`.

### R1 · Le détective

**Objectif** : lire une image sans sa documentation.

**Mission** : pour chaque image, trouve avec `docker image inspect` le port exposé, l'utilisateur, la commande de lancement et les volumes déclarés.

| Image | Port exposé | Utilisateur | Commande | Volumes |
| --- | --- | --- | --- | --- |
| `nginx` |  |  |  |  |
| `postgres:16` |  |  |  |  |
| `louislam/uptime-kuma:1` |  |  |  |  |

**Vérification** : tu expliques chaque case au formateur sans avoir ouvert Docker Hub.

**Indice** : `docker image inspect --format '{{json .Config}}' nginx`

**À noter** : une image qui ne déclare aucun utilisateur tourne sous quel compte ?

### R2 · Une boîte à outils web

**Mission** : déploie `corentinth/it-tools`, accessible sur le port 8090 de ta machine.

**Vérification** : `curl -s localhost:8090 | grep -i '<title>'` affiche le titre de l'app.

**Indice** : la méthode de R1 te donne le port interne.

### R3 · Un outil de supervision

**Mission** : déploie Uptime Kuma (`louislam/uptime-kuma:1`) sur le port 3001. Crée le compte admin et ajoute une sonde HTTP qui surveille ton conteneur `web` du défi 4. Ensuite, supprime le conteneur Kuma et recrée-le.

**Vérification** : avant la suppression, la sonde est verte dans l'interface.

**Indice** : dans Kuma, `localhost` désigne le conteneur Kuma lui-même. Trouve une autre adresse pour joindre `web`.

**À noter** : après la recréation, que sont devenus ton compte et ta sonde ?

### R4 · Une forge Git

**Mission** : déploie Gitea (`gitea/gitea:1.22`), interface web sur le port 3000 et SSH sur le port 2222 de ta machine. Termine l'assistant d'installation avec SQLite, crée un compte et un dépôt, puis pousse ton dossier `nerdnotes/` en HTTP.

**Vérification** : ton code apparaît dans l'interface de Gitea.

**À noter** : pourquoi 2222 et pas 22 côté machine ?

### R5 · Une interface pour Postgres

**Mission** : déploie `adminer` sur le port 8088 et connecte-toi à la base `mystere` du défi 7.

**Vérification** : Adminer affiche les bases de `mystere`.

**Indice** : Adminer et `mystere` sont deux conteneurs différents. Le champ « Serveur » attend une adresse qu'Adminer peut joindre.

**À noter** : qu'aurais-tu aimé pouvoir écrire dans le champ « Serveur » ?
