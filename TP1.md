# TP1 --- Installation de VirtualBox et création d'une VM Ubuntu

**Cours :** Introduction au Cloud --- L3 SI RIST & BDGL\
**Étudiant(s) :** \[À compléter\]\
**Date :** \[À compléter\]

> **À vérifier avant le dépôt :** les valeurs entre crochets doivent
> être remplacées par vos résultats réels. La capture envoyée pendant le
> TP montrait `ubuntu-26.04.1-desktop-amd64.iso` attaché à la VM, alors
> que la fiche demande Ubuntu Server. Confirmez la version réellement
> installée et ne déclarez pas Ubuntu Server installé si ce n'est pas le
> cas.

## 1. Objectif général

Ce TP vise à découvrir la virtualisation en installant et en utilisant
VirtualBox, en créant une machine virtuelle Ubuntu, en accédant à
celle-ci avec SSH, puis en manipulant des instantanés et des clones. Il
permet de comprendre une partie du fonctionnement du Cloud IaaS :
création, configuration et administration d'une machine virtuelle sur un
hôte physique.

## 2. Environnement et prérequis

### Configuration observée

  -----------------------------------------------------------------------
  Élément                             Résultat
  ----------------------------------- -----------------------------------
  Système hôte                        Windows 10, version
                                      `10.0.19045.6466`

  Processeur                          Intel Core i5-5300U @ 2.30 GHz

  Cœurs physiques                     2

  Processeurs logiques                4

  Mémoire rapportée par VirtualBox    12163 MByte

  Virtualisation matérielle           Activée ; prise en charge HW
                                      signalée par VirtualBox

  VirtualBox                          `7.2.20r175154`

  Espace libre                        Au moins 30 Go, d'après la
                                      vérification préalable
  -----------------------------------------------------------------------

Commandes utilisées dans PowerShell :

``` powershell
Test-Path "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe"
$env:Path += ";C:\Program Files\Oracle\VirtualBox"
VBoxManage --version
VBoxManage list hostinfo
```

Le test `Test-Path` a renvoyé `True`. Au départ, PowerShell ne
reconnaissait pas `VBoxManage`, car le dossier du programme n'était pas
présent dans le `PATH` de la session. L'ajout temporaire au `PATH` a
permis d'exécuter les commandes.

## 3. Partie 1 --- Préparer l'hôte et vérifier VirtualBox

### Résultats

La commande `VBoxManage --version` a renvoyé `7.2.20r175154`. La
commande `VBoxManage list hostinfo` a rapporté 2 cœurs physiques, 4
processeurs logiques, une mémoire totale de 12163 MByte et la prise en
charge de la virtualisation matérielle.

### Questions

**1. Différence entre hyperviseur de type 1 et type 2.**\
Un hyperviseur de type 1 fonctionne directement sur le matériel. Un
hyperviseur de type 2 fonctionne comme une application au-dessus d'un
système d'exploitation hôte. VirtualBox installé sur Windows est un
hyperviseur de type 2. VMware ESXi est un exemple d'hyperviseur de type
1.

**2. Pourquoi VT-x / AMD-V est-il important ?**\
Ces technologies fournissent au processeur des mécanismes permettant
d'exécuter des machines virtuelles efficacement. Si elles sont
indisponibles ou désactivées, certaines VM 64 bits peuvent ne pas
démarrer correctement.

**3. Version, cœurs et RAM.**\
Version : `7.2.20r175154`. La sortie indique 2 cœurs physiques, 4
processeurs logiques et 12163 MByte de mémoire totale.

**Capture à ajouter :** `[Capture des commandes VBoxManage]`

## 4. Partie 2 --- Télécharger et vérifier l'ISO Ubuntu

### Objectif et commandes

Une image ISO contient les fichiers nécessaires à l'installation du
système d'exploitation. L'empreinte SHA-256 permet de comparer le
fichier téléchargé à la somme de contrôle officielle et de détecter une
différence de contenu.

Télécharger Ubuntu Server depuis <https://ubuntu.com/download/server> et
le fichier `SHA256SUMS` correspondant à la même version, puis exécuter
dans PowerShell (adapter le nom exact du fichier si nécessaire) :

``` powershell
cd $HOME\Downloads
Get-FileHash .\ubuntu-26.04-live-server-amd64.iso -Algorithm SHA256
Select-String -Path .\SHA256SUMS -Pattern "live-server-amd64"
```

-   Version et nom exacts de l'ISO : `[À compléter]`
-   Taille de l'ISO : `[À compléter]`
-   Empreinte calculée : `[Coller le résultat]`
-   Empreinte officielle : `[Coller la ligne correspondante]`
-   Empreintes identiques : `[Oui / Non, selon les résultats]`

### Questions

**1. Pourquoi vérifier l'empreinte ?**\
Une empreinte identique indique que le fichier correspond au contenu
associé à la somme de contrôle publiée. Une différence peut indiquer un
téléchargement incomplet, une corruption ou un fichier différent ; il
faut alors vérifier le fichier avant de l'utiliser.

**2. Ubuntu Desktop et Ubuntu Server.**\
Desktop vise principalement l'utilisation sur un poste de travail et
inclut un environnement graphique. Server est destiné aux usages serveur
et s'administre souvent en ligne de commande. Server convient aux VM
cloud lorsqu'un environnement graphique n'est pas nécessaire.

**3. Que signifie LTS ?**\
LTS signifie *Long Term Support* (support à long terme). La durée exacte
des mises à jour de sécurité standard doit être confirmée pour la
version effectivement utilisée :
`[À vérifier dans la documentation officielle de cette version]`.

**Capture à ajouter :**
`[Résultat SHA-256 et ligne officielle correspondante]`

## 5. Partie 3 --- Créer et configurer la VM

### Configuration attendue par la fiche

  Paramètre                        Valeur attendue
  -------------------------------- -----------------------------------------------
  Nom                              `ubuntu-srv`
  Type                             Linux / Ubuntu (64-bit)
  Installation sans surveillance   Désactivée
  RAM                              2048 Mo
  Processeurs                      2
  Disque                           VDI, 25 Go, allocation dynamique
  Réseau                           NAT
  Redirection SSH                  TCP `127.0.0.1:2222` vers le port invité `22`

### Résultats visibles

La capture de VirtualBox montrait la VM `ubuntu-srv` éteinte, avec 2048
Mo de RAM, 2 processeurs, un disque `ubuntu.vdi` de 25 Gio et un réseau
NAT. Elle montrait aussi l'ISO `ubuntu-26.04.1-desktop-amd64.iso`
attachée. Ce point diffère de la consigne Ubuntu Server et doit être
vérifié.

### Questions

**1. Disque dynamique et disque fixe.**\
Un disque dynamique utilise progressivement de l'espace sur l'hôte à
mesure que les données sont écrites, jusqu'à la capacité maximale. Un
disque fixe réserve généralement sa capacité dès sa création. Taille
réelle initiale du VDI observée : `[À compléter]`.

**2. Pourquoi désactiver l'installation sans surveillance ?**\
Pour réaliser manuellement les étapes, comprendre les choix de
l'installateur et recueillir les informations demandées.

**3. NAT et redirection 2222 → 22.**\
En mode NAT, la VM se trouve derrière le réseau virtuel de VirtualBox et
son adresse interne n'est pas nécessairement directement accessible
depuis l'hôte. La redirection permet d'accéder au service SSH de la VM
via le port 2222 de Windows, transmis au port 22 de la VM.

**Captures à ajouter :**
`[Paramètres de la VM, stockage et règle de redirection NAT/SSH]`

## 6. Partie 4 --- Installer Ubuntu Server

La fiche demande de choisir le français, le clavier French/AZERTY,
Ubuntu Server non minimisé, le réseau DHCPv4 sur `enp0s3`, le disque
entier avec LVM, le nom `ubuntu-srv`, l'utilisateur `etudiant`, de
passer Ubuntu Pro, d'installer OpenSSH et de ne sélectionner aucun snap
proposé.

Résultats réels à consigner :

-   Système et version installés : `[À compléter : Server ou Desktop]`
-   Durée de l'installation : `[À compléter]`
-   Adresse IP attribuée à `enp0s3` : `[À compléter]`
-   OpenSSH installé et actif : `[À confirmer dans la partie 5]`

### Questions

**1. Durée et comparaison au cloud.**\
Durée mesurée : `[À compléter]`. Une installation manuelle prend le
temps de parcourir les étapes et de configurer le système. Une instance
cloud créée à partir d'une image prête peut être disponible plus
rapidement car le système de base est déjà préparé.

**2. Adresse IP et attribution.**\
Adresse observée : `[À compléter]`. Avec DHCP, l'adresse est attribuée
automatiquement par le serveur DHCP du réseau virtuel de VirtualBox.

**3. Pourquoi installer OpenSSH ?**\
SSH permet d'administrer le serveur à distance en ligne de commande.
Sans SSH ni autre moyen distant configuré, l'administration à distance
serait plus difficile et demanderait une console alternative.

**Capture à ajouter :**
`[Écran ou sortie montrant la version installée et la connexion console]`

## 7. Partie 5 --- Premiers pas, réseau et SSH

### Commandes Linux utilisées

Dans la VM :

``` bash
lsb_release -a
uname -r
hostnamectl
nproc
free -h
df -h /
lsblk
ip -4 addr show
ip route
resolvectl status | grep "DNS Servers"
ping -c 3 ubuntu.com
sudo apt update && sudo apt upgrade -y
systemctl status ssh --no-pager
sudo ss -tlnp | grep :22
```

Ces commandes servent à identifier le système, le noyau, le nom de
l'hôte, les ressources matérielles et le stockage, puis à examiner le
réseau, tester Internet et vérifier le service SSH.

Valeurs relevées :

-   Adresse IPv4 : `[À compléter]`
-   Masque ou préfixe : `[À compléter]`
-   Passerelle : `[À compléter]`
-   DNS : `[À compléter]`
-   Résultat du ping : `[À compléter]`
-   Statut SSH : `[À compléter]`

### Connexion depuis Windows et transfert de fichier

Depuis PowerShell :

``` powershell
ssh -p 2222 etudiant@127.0.0.1
"Bonjour depuis l'hôte" | Out-File -Encoding utf8 bonjour.txt
scp -P 2222 .\bonjour.txt etudiant@127.0.0.1:~
ssh -p 2222 etudiant@127.0.0.1 "cat ~/bonjour.txt"
```

-   Connexion SSH réussie : `[Oui / Non et résultat]`
-   Transfert et lecture du fichier réussis : `[Oui / Non et résultat]`

### Questions

**1. Adresse, masque, passerelle et DNS.**\
Recopier les valeurs des commandes réseau ci-dessus. Dans le réseau NAT
classique de VirtualBox, `10.0.2.2` correspond généralement à la
passerelle virtuelle.

**2. Pourquoi `127.0.0.1:2222` plutôt que `10.0.2.15:22` ?**\
`127.0.0.1` est l'adresse de boucle locale de l'hôte. Le port 2222 de
l'hôte est redirigé vers le port 22 de la VM. `10.0.2.15` est une
adresse interne typique du NAT VirtualBox, qui n'est pas nécessairement
directement accessible depuis Windows.

**3. Que contient l'empreinte affichée à la première connexion SSH ?**\
Elle identifie la clé d'hôte SSH du serveur. Sa vérification aide à
repérer une clé inattendue et à réduire le risque d'interception de
connexion (*man-in-the-middle*).

**4. Avantages des clés SSH.**\
Les clés peuvent offrir une authentification robuste sans saisir un mot
de passe à chaque connexion et résistent mieux aux tentatives de deviner
un mot de passe, à condition de protéger la clé privée. Les fournisseurs
cloud les favorisent souvent pour sécuriser l'administration distante.

**Captures à ajouter :**
`[Sorties réseau, statut SSH, connexion et transfert de fichier]`

## 8. Partie 6 --- Instantané, clone et arrêt

### Instantané et restauration

Depuis PowerShell sur l'hôte :

``` powershell
VBoxManage snapshot ubuntu-srv take apres-installation --description "Ubuntu installé, OpenSSH actif"
VBoxManage snapshot ubuntu-srv list
```

Dans la VM :

``` bash
sudo apt install -y htop
touch ~/avant-restauration.txt
sudo poweroff
```

Puis sur l'hôte :

``` powershell
VBoxManage snapshot ubuntu-srv restore apres-installation
VBoxManage startvm ubuntu-srv --type headless
ssh -p 2222 etudiant@127.0.0.1 "which htop; ls ~"
```

Résultats réels : `htop` absent après restauration `[À confirmer]` ;
fichier `avant-restauration.txt` absent `[À confirmer]`.

### Clonage et suppression

``` powershell
VBoxManage controlvm ubuntu-srv acpipowerbutton
VBoxManage clonevm ubuntu-srv --name ubuntu-clone --register
VBoxManage modifyvm ubuntu-clone --natpf1 delete ssh
VBoxManage modifyvm ubuntu-clone --natpf1 "ssh,tcp,127.0.0.1,2223,,22"
VBoxManage startvm ubuntu-clone --type headless
ssh -p 2223 etudiant@127.0.0.1 "sudo hostnamectl set-hostname ubuntu-clone; hostname"
VBoxManage list runningvms
VBoxManage controlvm ubuntu-clone acpipowerbutton
VBoxManage unregistervm ubuntu-clone --delete
```

Résultats à confirmer :

-   Instantané créé et listé : `[À compléter]`
-   Restauration vérifiée : `[À compléter]`
-   Clone créé et démarré : `[À compléter]`
-   Nom d'hôte du clone modifié : `[À compléter]`
-   Clone supprimé : `[À compléter]`
-   Espace disque comparé : `[À compléter si mesuré]`

### Questions

**1. Différence entre instantané et clone. L'instantané est-il une
sauvegarde ?**\
Un instantané conserve un état de la même VM afin d'y revenir. Un clone
est une copie de la VM qui peut fonctionner indépendamment. Un
instantané ne remplace pas une sauvegarde complète, car il dépend des
fichiers de la VM et de l'hôte.

**2. Équivalents cloud.**\
L'instantané correspond à un snapshot de disque ou de volume. Le clone
ressemble à la création d'une nouvelle instance à partir d'une image ou
d'un modèle.

**3. `acpipowerbutton` et `poweroff`.**\
`acpipowerbutton` demande au système invité de s'arrêter proprement via
ACPI. `poweroff` force l'arrêt de la VM et peut faire perdre les données
non enregistrées.

**4. Pourquoi le mode headless ?**\
Il permet de démarrer la VM sans fenêtre graphique, pratique pour une
machine administrée à distance, comme une instance cloud.

**5. Pourquoi changer le port et le nom d'hôte du clone ?**\
Un port différent évite les conflits de redirection SSH. Un nom d'hôte
distinct aide à reconnaître chaque machine dans les commandes et
journaux.

**Captures à ajouter :**
`[Instantané, restauration, clone démarré et preuve de suppression]`

## 9. Conclusion

Ce TP permet de découvrir la virtualisation avec VirtualBox et les bases
de l'administration d'une machine Ubuntu. Il aborde la configuration des
ressources, le réseau NAT, la redirection de ports, SSH, les instantanés
et les clones. Ces notions sont directement liées à certaines opérations
proposées dans le Cloud IaaS.

La version exacte du système installé, les valeurs réseau, les
empreintes, les durées et les tests doivent être complétés avec les
résultats réellement observés avant de remettre le rapport.

## 10. Sources

-   Fiche de TP : *TP 1 --- Installation de VirtualBox et création d'une
    VM Ubuntu*, Introduction au Cloud, L3 SI RIST & BDGL.
-   VirtualBox : <https://www.virtualbox.org/wiki/Downloads>
-   Ubuntu Server : <https://ubuntu.com/download/server>
-   Sommes de contrôle Ubuntu : <https://releases.ubuntu.com/>

## Vérification avant dépôt

-   [ ] Remplacer les champs `[À compléter]`.
-   [ ] Confirmer que l'ISO utilisée est bien Ubuntu Server et noter sa
    version.
-   [ ] Ajouter les captures et sorties de commandes.
-   [ ] Confirmer les tests SSH, l'instantané, la restauration et le
    clone.
-   [ ] Enregistrer sous le nom exact `TP1.md` et déposer sur le dépôt
    Git du binôme.
