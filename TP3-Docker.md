# TP 2 : Premiers conteneurs avec Docker
## Trinôme : GRAH GRACE DIVINE SEPHORA / BOKA ANGE ERNEST / ZOBO YVANN JUNIOR

### PARTIE 0 — Préparer la VM et installer Docker

**Q1. Quelle version de Docker est installée ? Quel noyau Docker utilise-t-il (ligne Kernel Version) ?**
*   **Réponse :** La version de Docker installée sur la machine virtuelle est la **29.1.3** (tant pour le Client que pour le Server Engine). Le noyau Linux utilisé par Docker est le **7.0.0-38-generic**.

**Q2. Être membre du groupe docker revient presque à être administrateur de la machine. Pourquoi ? Quelle précaution cela impose-t-il sur un serveur partagé ?**
*   **Réponse :** Un membre du groupe `docker` peut exécuter des conteneurs avec des privilèges élevés. Par exemple, il peut lancer un conteneur en montant la racine du disque de la VM hôte (`-v /:/host`) et modifier n'importe quel fichier système avec les droits `root`.
*   **Précaution :** Sur un serveur partagé, il ne faut ajouter au groupe `docker` que des utilisateurs de confiance absolue (administrateurs). Pour les autres utilisateurs, il convient d'utiliser Docker en mode sans privilèges (*Rootless mode*) ou restreindre l'accès à la socket Docker (`docker.sock`).

---

### PARTIE 1 — Lancer et inspecter des conteneurs

**Q1. D'après le message de hello-world, quelles sont les quatre étapes réalisées par Docker ?**
*   **Réponse :** D'après le message affiché par le conteneur à l'écran, les quatre étapes sont :
    1. Le client Docker a contacté le démon Docker en arrière-plan (*The Docker client contacted the Docker daemon*).
    2. Le démon Docker a téléchargé l'image depuis le registre Docker Hub car elle n'était pas présente localement (*The Docker daemon pulled the "hello-world" image from the Docker Hub*).
    3. Le démon Docker a créé un nouveau conteneur à partir de cette image et a exécuté le programme à l'intérieur (*The Docker daemon created a new container from that image which runs the executable...*).
    4. Le démon Docker a renvoyé la sortie textuelle au client Docker, qui l'a affichée dans votre terminal pour que vous puissiez lire ce message (*The Docker daemon streamed that output to the Docker client, which sent it to your terminal*).

**Q2. Le conteneur web et la VM ont-ils la même distribution ? Le même noyau ? Expliquez à l'aide du cours.**
*   **Distribution :** Non, ils n'ont pas la même distribution. Le conteneur web utilise une distribution **Debian GNU/Linux 13 (trixie)** (issue de l'image officielle nginx), tandis que votre VM utilise **Ubuntu 24.04.1 LTS**.
*   **Noyau :** Oui, ils ont exactement le même noyau. La commande `uname -r` renvoie **7.0.0-38-generic** à la fois dans le conteneur et dans la VM.
*   **Explication :** Contrairement à une machine virtuelle qui émule un système d'exploitation complet avec son propre noyau, un conteneur Docker partage nativement le noyau Linux de la machine hôte (ici, la VM). Il n'embarque que les bibliothèques applicatives et l'espace utilisateur (*user space*) propres à sa distribution (Debian).

**Q3. Les processus Nginx du conteneur apparaissent-ils dans ps aux sur la VM ? Qu'est-ce que cela montre sur la nature d'un conteneur ?**
*   **Réponse :** Oui, le processus maître de Nginx (lancé par root avec l'UID 3867 ou similaire sur vos captures) est visible directement depuis la VM avec la commande `ps aux`.
*   **Nature d'un conteneur :** Cela démontre qu'un conteneur n'est pas une machine virtuelle isolée matériellement. Un conteneur est simplement un processus classique isolé s'exécutant sur l'hôte, confiné grâce aux fonctionnalités du noyau Linux telles que les *namespaces* (pour l'isolation visuelle) et les *cgroups* (pour la limitation des ressources).

**Q4. Retracez le chemin d'une requête du navigateur de l'hôte jusqu'à Nginx : quelles redirections de port traverse-t-elle ?**
*   **Réponse :** La requête traverse deux redirections successives :
    1. **Première redirection (Hôte Windows → VM Ubuntu) :** Le navigateur Windows attaque `127.0.0.1:8080`. Grâce à la règle NAT configurée dans VirtualBox (`web,tcp,127.0.0.1,8080,,8080`), la requête est redirigée vers le port `8080` de la VM Ubuntu.
    2. **Deuxième redirection (VM Ubuntu → Conteneur Nginx) :** Sur la VM, Docker écoute sur le port `8080` grâce à l'option `-p 8080:80`. Le moteur Docker intercepte la requête et la redirige vers le port `80` interne du conteneur où écoute le serveur Nginx.

**Q5. Pourquoi la page modifiée a-t-elle disparu après docker rm ? Que change l'option -v ?**
*   **Disparition :** Par défaut, la couche supérieure d'un conteneur (la couche d'écriture) est éphémère. Tout fichier créé ou modifié directement à l'intérieur du conteneur (comme votre premier `index.html`) est définitivement supprimé lorsque le conteneur est détruit avec `docker rm`.
*   **L'apport de l'option -v :** L'option `-v ~/site:/usr/share/nginx/html:ro` crée un volume (*bind mount*). Elle lie un dossier persistant de la VM hôte (`~/site`) à un dossier interne du conteneur. Les données restent stockées en toute sécurité sur le disque de la VM. Si le conteneur est supprimé puis recréé, le nouveau conteneur remonte le dossier et retrouve instantanément la page modifiée.

---

### PARTIE 2 — Comparer conteneur et VM

#### Tableau de mesures comparatives

| Mesure | VM ubuntu-srv | Conteneur nginx | Conteneur alpine |
| :--- | :--- | :--- | :--- |
| **Taille sur disque** | ~4.5 Go (Taille du fichier .vdi VirtualBox) | 198 Mo (Taille virtuelle stockée) | 3.94 Mo (Image Alpine ultra-légère) |
| **Temps de démarrage** | 2 min 20.319 s (Total d'après systemd-analyze) | ~0.5 s (Démarrage quasi instantané) | 0.899 s (Mesuré avec la commande time) |
| **Mémoire utilisée** | 467 Mo (Colonne used de free -h) | 5.1 Mo (Consommation moyenne au repos) | Quelques Ko (Le processus s'éteint après l'écho) |
| **Système** | Ubuntu complet + noyau (Ubuntu 24.04.1 LTS) | Debian GNU/Linux 13 (Espace utilisateur de l'image) | Alpine Linux v3.20 (Distribution minimale basée sur musl) |

**Question 1. Dans quel rapport (environ ×10, ×100…) le conteneur is-il plus léger et plus rapide que la VM ? Vos mesures confirment-elles le tableau du cours ?**
*   **Poids (Taille) :** L'image Alpine fait environ 4 Mo, alors qu'une VM Linux minimale pèse généralement entre 2 et 4 Go. Le conteneur est donc environ **1000 fois plus léger (× 1000)**.
*   **Vitesse (Démarrage) :** Le conteneur démarre en moins d'une seconde (0.899s), tandis que la VM met 140 secondes (2min 20s). Le conteneur est donc environ **150 fois plus rapide (× 150)**.
*   **Conclusion :** Oui, ces mesures confirment parfaitement la théorie du cours. Les conteneurs ont une empreinte mémoire et disque minime et s'initialisent de façon quasi instantanée par rapport aux VM.

**Question 2. Pourquoi le premier time docker run est-il plus long que le second ?**
*   **Réponse :** De façon générale, le premier run inclut un léger surcoût lié à l'initialisation des structures de namespaces par le démon Docker. Le second bénéficie du cache du système d'exploitation et des mécanismes internes déjà chauds de Docker, ce qui stabilise le temps d'exécution réel de l'isolation du processus.

**Question 3. Citez un cas où vous choisiriez quand même une VM plutôt qu'un conteneur.**
*   **Réponse :** On choisira une machine virtuelle lorsque l'on a besoin d'une isolation de sécurité forte au niveau matériel (*multi-tenancy* sur du cloud public), ou lorsqu'on doit exécuter un système d'exploitation avec un noyau différent de l'hôte (par exemple, faire tourner un serveur Windows ou exécuter des modules de noyau Linux spécifiques incompatibles avec le noyau de l'hôte).

---

### PARTIE 3 — Conteneuriser l'API fil rouge

**Question 1. Quelle est la taille de l'image api:1.0 ? Combien de couches docker history affiche-t-il, et lesquelles viennent de votre Dockerfile ?**
*   **Taille de l'image :** D'après la commande `docker images api`, la taille totale de l'image `api:1.0` est de **48.3MB** (taille virtuelle disque de 198MB).
*   **Nombre de couches :** La commande `docker history api:1.0` affiche au total **15 couches** (lignes).
*   **Couches venant de votre Dockerfile :** Les 6 couches supérieures de la pile ont été directement générées par les instructions écrites dans votre Dockerfile :
    *   `CMD python app.py` (0 B)
    *   `EXPOSE 5000` (0 B)
    *   `COPY . .` (28.5 KB)
    *   `RUN /bin/sh -c pip install --no-cache-dir -r...` (15.3 MB)
    *   `COPY requirements.txt .` (12.3 KB)
    *   `WORKDIR /app` (8.19 KB)
    *   *Note : Les autres couches inférieures proviennent de l'image de base python:3.12-slim.*

**Question 2. Pourquoi copie-t-on requirements.txt et installe-t-on les dépendances avant de copier le code ? Appuyez-vous sur les lignes CACHED de l'étape 6.**
*   **Réponse :** Docker utilise un système de mise en cache par couche. Si aucun fichier n'a changé lors d'une étape, Docker réutilise la couche précédemment construite. Le fichier `requirements.txt` change très rarement, alors que le code source (`app.py`) est modifié constamment pendant le développement.
*   **Preuve par l'étape 6 :** Lors de la construction de la version `api:1.1`, on constate à l'écran que les étapes `[2/5] WORKDIR`, `[3/5] COPY requirements.txt` et `[4/5] RUN pip install` portent la mention **`CACHED`**. Docker n'a pas réinstallé Flask inutilement. Il a directement sauté à l'étape `[5/5] COPY . .` pour intégrer vos modifications de code, faisant gagner énormément de temps de build.

**Question 3. Que renvoie le champ conteneur pour les trois copies de l'étape 5 ? Quelle caractéristique du cloud (cours, séance 1) cette expérience illustre-t-elle ?**
*   **Valeurs renvoyées :** Le champ conteneur renvoie l'ID abrégé de la machine qui exécute le script. Pour vos trois copies esclaves, les requêtes renvoient des identifiants uniques distincts :
    *   Conteneur sur port 5001 : **`23d54681510a`**
    *   Conteneur sur port 5002 : **`56676d5bcc75`**
    *   Conteneur sur port 5003 : **`decff2cc7796`**
*   **Caractéristique du Cloud illustrée :** Cette expérience illustre l'**élasticité** et l'**extensibilité horizontale (Scale-Out)**. À partir d'une seule et unique image immuable (`api:1.0`), il est possible de dupliquer et d'instancier instantanément de multiples réplicas de l'application en parallèle pour absorber une hausse de trafic ou effectuer de la répartition de charge (*Load Balancing*).

**Question 4. Pourquoi passer le nom du Trinôme par une variable d'environnement (-e) plutôt que de l'écrire dans le code ? Quel lien avec les secrets (mots de passe, clés) ?**
*   **Découplage :** Passer des configurations par variable d'environnement permet d'isoler la logique de l'application (le code) de ses paramètres d'exécution. Cela permet de garder l'image Docker générique et réutilisable dans n'importe quel environnement (Développement, Test, Production) sans modifier une seule ligne de code.
*   **Lien avec les secrets :** C'est une règle de sécurité fondamentale. Écrire des secrets (mots de passe de base de données, clés d'API tierces) en dur dans le code presents le risque de les exposer publiquement sur un dépôt Git ou dans les couches de l'image Docker. Les injecter via l'option `-e` à l'initialisation du conteneur sécurise la chaîne de déploiement.

---

### PARTIE 4 — Publier l'image et nettoyer

**Question 1. Indiquez l'adresse complète de votre image publiée. Est-elle publique ou privée ?**
*   **Réponse :** L'adresse complète de l'image est **`sephora25/api-fil-rouge:1.1`** (sur Docker Hub) ou **`ghcr.io/gracesephoragh/api-fil-rouge:1.1`** (sur GitHub Packages). L'image est configurée en mode **public** afin que n'importe quel serveur ou instance cloud puisse la télécharger (`docker pull`) sans nécessiter de clés d'authentification privées.

**Question 2. Pourquoi étiqueter l'image avec un numéro de version (1.1) plutôt que de toujours utiliser latest ?**
*   **Réponse :** L'utilisation d'un tag fixe comme `1.1` garantit la reproductibilité des déploiements. Le conteneur fonctionnera toujours de manière identique, quel que soit le moment ou la machine sur laquelle il est relancé. À l'inverse, l'étiquette `latest` est mouvante et pointe automatiquement vers la toute dernière version poussée sur le registre. Si une mise à jour instable ou comportant des erreurs est publiée, un redémarrage de conteneur utilisant `latest` déploiera cette version défectueuse en production à votre insu. Suivre un versioning strict sécurise les cycles de développement.

**Question 3. Quel espace disque docker image prune a-t-il libéré ? Dans le cloud, pourquoi faut-il aussi nettoyer les registres ?**
*   **Espace libéré :** D'après la sortie de la commande sur la capture d'écran, l'exécution de `docker image prune -a -f` a libéré un espace total de **3.987 MB** (*Total reclaimed space*). Le tableau `docker system df` confirme également que le nombre total d'images stockées localement est passé de 8 à 3.
*   **Nettoyage dans le Cloud :** Dans un environnement de cloud computing, l'espace de stockage des registres privés (comme AWS ECR, Azure ACR ou GitHub Packages) est faituré au Go par mois. Si l'on ne met pas en place des politiques de nettoyage automatique pour purger les anciennes images de build obsolètes, les coûts de stockage cloud augmentent inutilement de manière exponentielle.
