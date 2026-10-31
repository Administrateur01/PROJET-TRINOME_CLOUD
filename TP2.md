# TP 2 • Séance 2 — Deux machines virtuelles en réseau avec VirtualBox

**Binôme :** DOSSO - SAVANE - KONATE  
**Date :** 24 Octobre 2026  
**Cours :** Licence 3 — RIST / BDGL — Modèles de service et de déploiement

---

## Introduction

L'objectif de ce TP était de simuler le déploiement d'une infrastructure cloud (IaaS) en local. Nous avons cloné une image de base (`ubuntu-srv`) pour créer deux instances (`web` et `client`), les avons reliées dans un réseau NAT privé (`tp-cloud`), avons déployé un serveur web Nginx, configuré un pare-feu (`ufw`) et mesuré les ressources.

---

## Partie 1 — Créer les VM par clonage

### 1.1 Clonage et création du réseau

Nous avons cloné la VM `ubuntu-srv` pour créer `web` et `client`. Le clonage de `web` a pris **108,47 secondes**.

Nous avons ensuite créé le réseau NAT `tp-cloud` (10.10.0.0/24) avec les redirections de ports suivantes :
- `ssh-web` : `127.0.0.1:2222` → `10.10.0.10:22`
- `ssh-client` : `127.0.0.1:2223` → `10.10.0.20:22`
- `http-web` : `127.0.0.1:8080` → `10.10.0.10:80`

> **Capture 1 :** *Clonage des VM `web` et `client` avec chronométrage et réduction des ressources (1 vCPU, 1 Go RAM).*   
> ![Capture 1](images/capture1_vms.png)

### 1.2 Personnalisation des clones

Pour chaque clone, nous avons modifié le nom d'hôte, régénéré l'ID machine et les clés SSH :

```bash
sudo hostnamectl set-hostname web  # (ou client)
sudo sed -i 's/ubuntu-srv/web/g' /etc/hosts
sudo rm -f /etc/machine-id && sudo systemd-machine-id-setup
sudo rm -f /etc/ssh/ssh_host_* && sudo ssh-keygen -A

Configuration réseau (fichier /etc/netplan/99-tp.yaml) :

network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [10.10.0.10/24] # 10.10.0.20/24 pour client
      routes:
        - to: default
          via: 10.10.0.1
      nameservers:
        addresses: [10.10.0.1, 1.1.1.1]

Note technique : Sur Ubuntu 26.04, un bug de Netplan (Cannot find unique matching interface for enp0s3) a nécessité l'utilisation de l'adresse MAC (match: macaddress: ...) pour forcer l'application de la configuration sur l'interface enp0s3 de la VM client.

Réponses aux questions (Partie 1)
Q1. Combien de temps a pris le clonage de web ? Comparez avec la durée de l'installation d'Ubuntu au TP précédent, puis avec la mise en service d'un serveur physique.
Le clonage a pris 108,47 secondes (environ 1 min 48). L'installation complète d'Ubuntu Server prend environ 15 à 20 minutes. La mise en service d'un serveur physique prend des jours (commande, livraison, installation en baie, câblage). Le clonage est donc la méthode la plus rapide, typique du cloud (lancer une instance depuis une AMI).

Q2. Pourquoi faut-il changer le nom d'hôte, l'identifiant de machine et les clés SSH d'un clone ? Quel risque y aurait-il à garder les mêmes clés SSH sur 50 serveurs ?
Un clone est une copie exacte. S'il garde le même ID machine et les mêmes clés SSH, le réseau et les outils de gestion (comme Ansible) verront des conflits d'identité. Si 50 serveurs partagent la même clé SSH privée, la compromission d'un seul serveur permet à un attaquant d'accéder aux 49 autres.

Q3. Relevez la version d'Ubuntu, le noyau, le nombre de vCPU et la RAM de web. Quel service cloud joue le rôle de ubuntu-srv ?

OS : Ubuntu 26.04 LTS

Noyau : 7.0.0-38-generic

vCPU : 1

RAM : 1 Go (956 MiB vus par l'OS)

Le service cloud qui joue le rôle de ubuntu-srv est une AMI (Amazon Machine Image) sur AWS, ou une Image de VM sur Azure/GCP.

Partie 2 — Adressage et connectivité
Tests effectués depuis client
ping -c 4 10.10.0.10 (web) : 4 reçus, 0% perte.

ping -c 2 10.10.0.1 (passerelle) : 2 reçus.

ping -c 2 1.1.1.1 (Internet) : 2 reçus.

Ajout de 10.10.0.10 web dans /etc/hosts pour résoudre le nom.

ping -c 2 web : fonctionne.

Capture 2 : Tests de connectivité (ping vers web, passerelle et Internet) depuis la VM client.
https://images/capture2_ping.png

Réponses aux questions (Partie 2)
Q1. Les deux VM sont-elles dans le même sous-réseau ? Justifiez avec leurs adresses et le masque /24.
Oui. web a l'IP 10.10.0.10/24 et client a l'IP 10.10.0.20/24. Le masque /24 (255.255.255.0) signifie que les trois premiers octets (10.10.0) définissent le réseau. Les deux IP partagent ce réseau, elles peuvent donc communiquer directement sans routeur.

Q2. Quelle est la passerelle par défaut de client ? Quel est son rôle ? À quel élément d'un VPC (AWS) ou d'un VNet (Azure) correspond-elle ?
La passerelle est 10.10.0.1 (le routeur NAT de VirtualBox). Son rôle est de router le trafic sortant vers Internet (NAT) et de servir de DNS. Elle correspond à la route par défaut (Internet Gateway / NAT Gateway) dans un VPC AWS ou un VNet Azure.

Q3. Quel est le temps de réponse moyen du ping de client vers web ? Pourquoi est-il si faible ?
Le temps moyen était de 1.258 ms (min 0.335 ms, max 2.182 ms). Il est très faible car les deux VM sont sur le même hôte physique et communiquent via un réseau virtuel interne (VirtualBox), sans passer par un câble réseau physique ni par Internet.

Q4. Le ping depuis Windows vers 10.10.0.10 aboutit-il ? Pourquoi ? Comment l'hôte arrive-t-il malgré tout à joindre web en SSH ?
Non, le ping depuis Windows vers 10.10.0.10 échoue, car l'hôte Windows n'est pas dans le réseau NAT 10.10.0.0/24. Il ne peut pas joindre directement les VM. L'hôte arrive à joindre web en SSH grâce aux redirections de port configurées sur le réseau NAT (le port 2222 de Windows est redirigé vers le port 22 de web).

Partie 3 — Page web publiée, pare-feu maîtrisé
3.1 Installation de Nginx
Installation réussie sur web. La page par défaut a été remplacée par :
<h1>Trinome DOSSO-SAVANE-KONATE : Ma premiere VM en reseau</h1>

Test depuis client : curl -s http://web affiche bien le HTML.

Capture 3 : Page web personnalisée affichée dans le navigateur de l'hôte (via la redirection de port 8080).
https://images/capture3_nginx.png

3.2 Configuration du pare-feu ufw
sudo ufw allow OpenSSH (pour ne pas perdre l'accès).

sudo ufw enable (pare-feu actif).

Test depuis client : curl -m 5 http://web → Timeout (port 80 bloqué).

sudo ufw allow 'Nginx HTTP' (ouverture du port 80).

Test depuis client : curl -s http://web → HTML affiché (port 80 ouvert).

Capture 4 : Statut de ufw montrant le port 80 fermé (timeout) puis ouvert (HTML).
https://images/capture4_ufw.png

Réponses aux questions (Partie 3)
Q1. Que renvoie curl quand le port 80 est fermé (message et code retour) ? Pourquoi le navigateur reste-t-il bloqué au lieu d'afficher une erreur immédiate ?
curl renvoie curl: (28) Connection timed out after 5002 milliseconds. Le code retour est 28. Le navigateur reste bloqué car le paquet TCP SYN est envoyé mais aucune réponse ICMP (Reject) n'est renvoyée. Le client attend jusqu'au timeout TCP avant d'afficher une erreur.

Q2. Sur quel port et sur quelles adresses Nginx écoute-t-il ? Appuyez-vous sur la sortie de ss -tlnp.
Nginx écoute sur le port 80 sur toutes les adresses IPv4 (0.0.0.0:80) et IPv6 ([::]:80). La commande ss -tlnp affichait : LISTEN 0 511 0.0.0.0:80 0.0.0.0:* users:(("nginx",pid=2748,fd=5)).

Q3. Le pare-feu ufw joue ici le rôle d'un groupe de sécurité du cloud. Donnez deux ressemblances et une différence.

Ressemblances : (1) Il filtre le trafic entrant et sortant par port et protocole. (2) Il applique une politique par défaut (tout bloquer sauf ce qui est explicitement autorisé).

Différence : ufw est un pare-feu local installé sur la VM (logiciel), alors qu'un groupe de sécurité cloud (AWS/Azure) est un pare-feu réseau appliqué au niveau de l'infrastructure (en amont de la VM), sans consommer de ressources de la VM.

Q4. Dans ce TP, vous êtes le client d'un service IaaS. Reprenez le tableau « Qui gère quoi ? » du cours : quelles couches avez-vous gérées vous-même, et lesquelles VirtualBox a-t-il fournies ?

Géré par nous (client IaaS) : Le système d'exploitation (mises à jour), le runtime (Nginx), les données (page HTML), l'application, la configuration réseau interne (netplan), et le pare-feu (ufw).

Fourni par VirtualBox (le "cloud") : La virtualisation (hyperviseur), les serveurs physiques, le stockage (disques virtuels), le réseau physique (commutateur virtuel), et l'infrastructure réseau de base (le réseau NAT).

Partie 4 — Mesures comparées, nettoyage
Mesures effectuées
Sur web : free -h → 956 MiB total, 218 MiB utilisés. df -h / → 11.21 Go total, 5.56 Go utilisés. nproc → 1.

Sur Windows : Dossier web → 6.30 Go. Dossier client → 6.26 Go.

Capture 5 : Résultats de free -h sur web et taille réelle des dossiers de VM sur l'hôte.
https://images/capture5_ressources.png

Réponses aux questions (Partie 4)
Q1. La VM web voit-elle 1 Go de RAM ? Combien en consomme-t-elle réellement sur l'hôte ? Comment expliquez-vous l'écart ?
La VM voit 956 MiB (presque 1 Go). Sur l'hôte, elle consomme un peu plus que ce qu'elle voit, car VirtualBox a besoin de mémoire pour émuler le matériel virtuel (BIOS, carte réseau, etc.). L'écart vient aussi du fait que la VM n'utilise que 218 MiB sur les 956 MiB disponibles (elle a de la marge).

Q2. Le disque d'un clone fait 25 Go pour la VM. Quelle place occupe-t-il réellement sur l'hôte ? Pourquoi ?
Le disque virtuel est configuré pour 25 Go, mais il n'occupe que 6.30 Go sur l'hôte. C'est parce que VirtualBox utilise des disques dynamiquement alloués : le fichier ne prend que la place des données réellement écrites, pas la taille maximale.

Q3. Quelle différence entre acpipowerbutton, poweroff et unregistervm --delete ?

acpipowerbutton : Envoie un signal ACPI pour éteindre la VM proprement (fermeture des services, sauvegarde).

poweroff : Coupe l'alimentation brutalement (comme débrancher la prise), risque de corruption.

unregistervm --delete : Supprime la VM de VirtualBox et efface tous ses fichiers (disque, configuration). Irréversible.

Q4. Dans le cloud, une VM arrêtée est-elle encore facturée ? Indice : pensez au disque et à l'adresse IP publique.
Oui, en partie. Le disque (stockage) reste facturé car les données sont conservées. Si une adresse IP publique est réservée, elle peut aussi être facturée. En revanche, le CPU et la RAM ne sont plus facturés tant que la VM est éteinte.

Conclusion
Ce TP nous a permis de manipuler concrètement les concepts du cloud IaaS : clonage d'image, réseau privé virtuel (NAT), adressage IP fixe, pare-feu (groupe de sécurité), et gestion des ressources. Nous avons également été confrontés à des aléas (bug de Netplan sur Ubuntu 26.04, décalage d'horloge) qui nous ont appris à diagnostiquer et contourner des problèmes réels d'administration système.

Toutes les ressources ont été nettoyées à la fin du TP (VBoxManage unregistervm ... --delete), ne laissant que la VM de base ubuntu-srv pour les prochains TP.

Capture 6 : Vérification finale : seule la VM ubuntu-srv reste dans VirtualBox.
https://images/capture6_nettoyage.png






