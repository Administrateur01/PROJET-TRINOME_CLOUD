# Compte rendu TP 3 — Premiers conteneurs avec Docker

**Binôme :** KONATE Abdoulaye, SAVANE Lacina, DOSSO Sarah  
**Date :** 28 octobre 2026  
**Lien vers l'image publiée :** https://hub.docker.com/r/konateabdoulaye01/api-fil-rouge

---

## Partie 0 — Installation de Docker

Docker a été installé depuis les dépôts Ubuntu (`docker.io`, `docker-buildx`, `docker-compose-v2`). L'utilisateur `etudiant` a été ajouté au groupe `docker` pour pouvoir utiliser Docker sans `sudo`.

**Vérification :**
```bash
docker version
docker info | grep -E "Server Version|Storage Driver|Cgroup|Kernel Version"

Résultat :

Server Version : 29.1.3

Storage Driver : overlay2

Cgroup Version : 2

Kernel Version : 7.0.0-38-generic

Réponses aux questions
Q1. Quelle version de Docker est installée ? Quel noyau Docker utilise-t-il ?

"La version de Docker est 29.1.3. Le noyau utilisé est 7.0.0-38-generic, c'est-à-dire le noyau de la VM Ubuntu elle-même."

Q2. Être membre du groupe docker revient presque à être administrateur de la machine. Pourquoi ? Quelle précaution cela impose-t-il sur un serveur partagé ?

"Le démon Docker tourne en tant que root. En étant membre du groupe docker, un utilisateur peut lancer des conteneurs avec des privilèges élevés (par exemple monter le dossier / de l'hôte dans un conteneur), ce qui équivaut à être root. Sur un serveur partagé, il faut donc limiter l'accès au groupe docker aux personnes de confiance, car un membre pourrait lire ou modifier les fichiers des autres utilisateurs."

Partie 1 — Conteneurs lancés et inspectés
1.1 Premier conteneur
bash
docker run hello-world
docker images
docker ps -a
Le conteneur hello-world s'est exécuté puis arrêté (Exited (0)).

1.2 Un serveur web en une commande
bash
docker run -d --name web -p 8080:80 nginx
curl -s http://localhost:8080 | head -n 5
La page "Welcome to nginx!" s'affiche. Le navigateur de l'hôte y accède via http://127.0.0.1:8080.

1.3 Regarder à l'intérieur
bash
docker exec web cat /etc/os-release | head -n 2
cat /etc/os-release | head -n 2
docker exec web uname -r
uname -r
Résultats :

Conteneur : Debian GNU/Linux 13 (trixie)

VM : Ubuntu 26.04.1 LTS

Noyau (les deux) : 7.0.0-38-generic

bash
docker logs --tail 5 web
docker top web
ps aux | grep "nginx: master" | grep -v grep
docker stats --no-stream web
Résultats :

Les logs montrent les requêtes curl et navigateur (codes 200).

docker top web montre les 3 processus Nginx (master + 2 workers).

ps aux sur la VM montre le même processus master (PID 5148).

docker stats : le conteneur consomme ~18,8 Mo de RAM et a 3 processus.

1.4 Un conteneur est éphémère
bash
docker exec web sh -c 'echo "<h1>Binôme KONATE - SAVANE - DOSSO</h1>" > /usr/share/nginx/html/index.html'
curl -s http://localhost:8080
docker rm -f web
docker run -d --name web -p 8080:80 nginx
curl -s http://localhost:8080 | grep -o "<title>.*</title>"
Après recréation, la modification a disparu : on retrouve <title>Welcome to nginx!</title>.

Volume
bash
mkdir -p ~/site
echo "<h1>Binôme KONATE - SAVANE - DOSSO : servi depuis un volume</h1>" > ~/site/index.html
docker rm -f web
docker run -d --name web -p 8080:80 -v ~/site:/usr/share/nginx/html:ro nginx
curl -s http://localhost:8080
Même après suppression du conteneur, la page persiste car elle est stockée sur la VM.

Réponses aux questions
Q1. D'après le message de hello-world, quelles sont les quatre étapes réalisées par Docker ?

"1. Le client Docker a contacté le démon. 2. Le démon a téléchargé l'image depuis Docker Hub. 3. Le démon a créé un conteneur à partir de l'image. 4. Le démon a renvoyé la sortie au client Docker."

Q2. Le conteneur web et la VM ont-ils la même distribution ? Le même noyau ? Expliquez.

"Non, ils n'ont pas la même distribution : la VM est sous Ubuntu 26.04, le conteneur sous Debian 13. En revanche, ils partagent le même noyau (7.0.0-38-generic). C'est la caractéristique d'un conteneur : il partage le noyau de l'hôte, contrairement à une VM qui a son propre noyau."

Q3. Les processus Nginx du conteneur apparaissent-ils dans ps aux sur la VM ? Qu'est-ce que cela montre ?

"Oui, le processus master de Nginx apparaît sur la VM avec le même PID (5148). Cela montre qu'un conteneur n'est pas une machine virtuelle, mais un ensemble de processus isolés par les namespaces du noyau Linux, qui partagent le même noyau que l'hôte. C'est ce qui explique sa légèreté."

Q4. Retracez le chemin d'une requête du navigateur de l'hôte jusqu'à Nginx.

"Navigateur Windows (127.0.0.1:8080) → Redirection VirtualBox (NAT) → Port 8080 de la VM → Docker (-p 8080:80) → Port 80 du conteneur Nginx."

Q5. Pourquoi la page modifiée a-t-elle disparu après docker rm ? Que change l'option -v ?

"La page a disparu car le système de fichiers d'un conteneur est éphémère. L'option -v monte un dossier de la VM dans le conteneur, ce qui permet de persister les données même après suppression du conteneur."

Partie 2 — Mesures comparées
bash
docker pull alpine
docker images
time docker run --rm alpine echo "Bonjour depuis un conteneur"
time docker run --rm alpine echo "Bonjour depuis un conteneur"
systemd-analyze
free -h
Tableau comparatif
Mesure	VM ubuntu-srv	Conteneur nginx	Conteneur alpine
Taille sur disque	~25 Go (dossier VM)	243 Mo	9,49 Mo
Temps de démarrage	36,5 s	~1 s	0,78 s
Mémoire utilisée	470 Mo	18,8 Mo	~5 Mo
Système	Ubuntu 26.04 LTS	Debian 13 (trixie)	Alpine Linux
Réponses aux questions
Q1. Dans quel rapport le conteneur est-il plus léger et plus rapide que la VM ? Vos mesures confirment-elles le tableau du cours ?

"Le conteneur est environ ×46 plus rapide à démarrer (0,78 s contre 36,5 s) et environ ×25 plus léger en mémoire (18,8 Mo contre 470 Mo). Ces mesures confirment le tableau du cours."

Q2. Pourquoi le premier time docker run est-il plus long que le second ?

"Le premier inclut le temps de téléchargement de l'image (si elle n'est pas en cache). Le second utilise l'image déjà présente localement, donc le démarrage est quasi instantané."

Q3. Citez un cas où vous choisiriez quand même une VM plutôt qu'un conteneur.

"On choisit une VM pour une isolation forte (code non fiable), pour faire tourner un OS différent de l'hôte (Windows sur Linux), ou pour migrer un serveur existant tel quel."

Partie 3 — Dockerfile, image construite, cache expliqué
Fichiers du projet (~/api)
app.py :

python
import os
import socket
from flask import Flask, jsonify

app = Flask(__name__)

@app.get("/")
def accueil():
    return jsonify(
        message="API fil rouge",
        binome=os.environ.get("BINOME", "inconnu"),
        conteneur=socket.gethostname(),
    )

@app.get("/sante")
def sante():
    return jsonify(statut="ok")

@app.get("/version")
def version():
    return jsonify(version="1.1")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
requirements.txt :

text
flask>=3,<4
Dockerfile :

dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
.dockerignore :

text
__pycache__/
*.pyc
.git
Construction et inspection
bash
docker build -t api:1.0 .
docker images api
docker history api:1.0
Résultats :

Taille de api:1.0 : 198 Mo.

docker history affiche 15 couches. Les 6 premières viennent du Dockerfile, les autres de l'image de base python:3.12-slim.

Lancement et test
bash
docker run -d --name api -p 5000:5000 -e BINOME="KONATE-SAVANE-DOSSO" api:1.0
curl -s http://localhost:5000
curl -s http://localhost:5000/sante
Résultat : {"binome":"KONATE-SAVANE-DOSSO","conteneur":"cef123bedd77","message":"API fil rouge"} et {"statut":"ok"}.

Trois copies de l'image
bash
for p in 5001 5002 5003; do docker run -d --name api-$p -p $p:5000 -e BINOME="KONATE-SAVANE-DOSSO" api:1.0; done
for p in 5001 5002 5003; do curl -s http://localhost:$p; echo; done
docker rm -f api-5001 api-5002 api-5003
Résultat : Les trois conteneurs ont des hostnames différents (ce703d4cf9c9, 8e62217aaa19, 20d19d1f345d) mais le même message.

Ajout de la route /version et cache
bash
docker build -t api:1.1 .
docker rm -f api
docker run -d --name api -p 5000:5000 api:1.1
curl -s http://localhost:5000/version
Résultat : {"version":"1.1"}. Le build affiche CACHED pour les couches WORKDIR, COPY requirements.txt et RUN pip install.

Réponses aux questions
Q1. Quelle est la taille de l'image api:1.0 ? Combien de couches docker history affiche-t-il, et lesquelles viennent de votre Dockerfile ?

"L'image api:1.0 fait 198 Mo. docker history affiche 15 couches. Les 6 premières viennent de mon Dockerfile : CMD, EXPOSE, COPY . . (20.5kB), RUN pip install (15.3MB), COPY requirements.txt (12.3kB) et WORKDIR. Les autres viennent de l'image de base python:3.12-slim."

Q2. Pourquoi copie-t-on requirements.txt et installe-t-on les dépendances avant de copier le code ? Appuyez-vous sur les lignes CACHED.

"On copie requirements.txt et on installe les dépendances avant le code, car Docker construit l'image couche par couche. Si le code change, Docker ne reconstruit que la dernière couche (COPY . .) et réutilise les couches précédentes (CACHED). Si on avait copié le code avant, chaque modification aurait invalidé le cache et forcé la réinstallation de Flask."

Q3. Que renvoie le champ conteneur pour les trois copies ? Quelle caractéristique du cloud cette expérience illustre-t-elle ?

"Le champ conteneur renvoie un identifiant différent pour chaque copie, car chaque conteneur a son propre hostname. Cela illustre la scalabilité horizontale : on peut lancer plusieurs répliques de la même image pour répartir la charge."

Q4. Pourquoi passer le nom du binôme par une variable d'environnement (-e) plutôt que de l'écrire dans le code ? Quel lien avec les secrets ?

"On utilise une variable d'environnement pour ne pas écrire le nom en dur dans le code, ce qui permet de réutiliser la même image pour différents binômes. C'est le même principe que pour les secrets (mots de passe, clés) : on ne les met jamais dans l'image, car toute personne qui la télécharge pourrait les lire. On les injecte au lancement."

Partie 4 — Image publiée, vérifiée, nettoyage
Publication
bash
docker login -u konateabdoulaye01
docker tag api:1.1 konateabdoulaye01/api-fil-rouge:1.1
docker push konateabdoulaye01/api-fil-rouge:1.1
Image publiée : https://hub.docker.com/r/konateabdoulaye01/api-fil-rouge

Vérification
bash
docker rm -f api
docker rmi konateabdoulaye01/api-fil-rouge:1.1 api:1.1
docker run -d --name api -p 5000:5000 konateabdoulaye01/api-fil-rouge:1.1
curl -s http://localhost:5000/version
Résultat : {"version":"1.1"} (l'image a bien été retéléchargée depuis Docker Hub).

Nettoyage
bash
docker rm -f $(docker ps -aq)
docker system df
docker image prune -a -f
docker system df
Espace libéré : 40,93 Mo.

Réponses aux questions
Q1. Indiquez l'adresse complète de votre image publiée. Est-elle publique ou privée ?

"L'adresse est konateabdoulaye01/api-fil-rouge:1.1. Elle est publique."

Q2. Pourquoi étiqueter l'image avec un numéro de version (1.1) plutôt que de toujours utiliser latest ?

"Pour garantir la reproductibilité et la traçabilité. Avec latest, on ne sait pas quelle version est déployée, et une mise à jour imprévue peut casser l'application. Avec un tag précis (1.1), on sait exactement ce qu'on déploie et on peut revenir en arrière si besoin."

Q3. Quel espace disque docker image prune a-t-il libéré ? Dans le cloud, pourquoi faut-il aussi nettoyer les registres ?

"docker image prune a libéré 40,93 Mo. Dans le cloud, il faut aussi nettoyer les registres car chaque image publiée occupe de l'espace de stockage facturé. De plus, garder des images obsolètes peut poser des problèmes de sécurité (vulnérabilités non corrigées) et ralentir les déploiements."

Conclusion
Ce TP a permis de :

Installer Docker et configurer l'utilisateur sans sudo.

Manipuler les conteneurs (run, ps, logs, exec, top, stats).

Comprendre l'éphémérité et les volumes.

Mesurer la différence de performance entre une VM et un conteneur.

Écrire un Dockerfile, construire une image, observer le cache.

Publier une image sur Docker Hub et la vérifier.

text

---

