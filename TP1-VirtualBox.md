# Compte rendu du TP1 : Installation de VirtualBox et création d'une VM Ubuntu
**Cours :** Introduction au Cloud  
**Classe :** L3 SI RIST & BDGL  

---

## PARTIE 1 — Préparer l'hôte et installer VirtualBox

###  Commandes exécutées sur l'hôte (PowerShell Administrateur)
```powershell
# 1. Vérification de la présence d'un hyperviseur Windows (Hyper-V / WSL2)
bcdedit /enum | findstr hypervisorlaunchtype
systeminfo | findstr /C:"Hyper-V"

# 2. Ajout de VirtualBox au PATH de la session utilisateur
$env:Path += ";C:\Program Files\Oracle\VirtualBox"

# 3. Vérification de la bonne installation et récupération des infos de l'hôte
VBoxManage --version
VBoxManage list hostinfo
```

###  Réponses aux questions

#### Question 1 : Qu'est-ce qu'un hyperviseur de type 1 et de type 2 ? Dans quelle catégorie se range VirtualBox ? Citez un hyperviseur de type 1 utilisé par un fournisseur cloud.
Un hyperviseur est le logiciel qui permet de créer, gérer et exécuter des machines virtuelles (VM). On distingue deux catégories :
* **Hyperviseur de type 1 (Bare Metal) :** Il s'installe directement sur le matériel physique (le matériel nu) d'un serveur. Il est très performant, stable et sécurisé car il n'a pas la surcharge d'un OS hôte.
* **Hyperviseur de type 2 (Hosted) :** Il s'installe comme une application par-dessus un système d'exploitation déjà existant (Windows, macOS, Linux).

**VirtualBox se range dans la catégorie des hyperviseurs de type 2.**  
Un exemple d'hyperviseur de type 1 utilisé par un fournisseur Cloud est **AWS Nitro Hypervisor** (utilisé par Amazon Web Services).

#### Question 2 : Pourquoi la virtualisation matérielle (VT-x / AMD-V) is-elle indispensable pour faire tourner une VM 64 bits ?
La virtualisation matérielle permet à l'hyperviseur de transmettre directement les instructions lourdes et complexes de la machine virtuelle au processeur physique de l'ordinateur, sans passer par une émulation logicielle (qui ralentirait drastiquement le système). Sans cette option activée dans le BIOS/UEFI, un hyperviseur de type 2 ne possède pas les privilèges de sécurité nécessaires pour exécuter un système d'exploitation invité en architecture 64 bits de manière stable et performante.

#### Question 3 : Quelle version de VirtualBox avez-vous installée ? Que renvoie `VBoxManage list hostinfo` sur le nombre de coeurs et la RAM de l'hôte ?
* **Version de VirtualBox installée :** 7.2.20
* **Ressources de l'hôte détectées :**
  * **Nombre de coeurs :** 4 cœurs physiques et 8 cœurs logiques (threads).
  * **Quantité de RAM totale :** 16 227 Mo (environ 16 Go).

---

## PARTIE 2 — Télécharger et vérifier l'image ISO Ubuntu

###  Commandes exécutées sur l'hôte (PowerShell)
```powershell
# 1. Navigation vers le dossier des téléchargements
cd $HOME\Downloads

# 2. Calcul de l'empreinte de l'image ISO téléchargée
Get-FileHash .\ubuntu-26.04-live-server-amd64.iso -Algorithm SHA256

# 3. Recherche et affichage de la ligne correspondante dans le fichier officiel SHA256SUMS
Select-String -Path .\SHA256SUMS -Pattern "live-server-amd64"
```

###  Réponses aux questions

#### Question 1 : Les deux empreintes sont-elles identiques ? Pourquoi faut-il vérifier l'empreinte d'une image avant de l'installer ?
* **Résultat :** Oui, les deux empreintes SHA256 sont strictement identiques. La suite de caractères générée en local correspond exactement à celle fournie sur le site officiel.
* **Pourquoi vérifier l'empreinte ?** Il est indispensable de vérifier l'empreinte cryptographique d'une image ISO avant de l'installer pour s'assurer que le fichier n'a pas été corrompu ou tronqué lors du téléchargement, et qu'il n'a pas été altéré ou piraté par un code malveillant (injection de malware).

#### Question 2 : Quelle différence entre Ubuntu Desktop et Ubuntu Server ? Pourquoi choisit-on Server pour un serveur cloud ?
* **Différences :** Ubuntu Desktop intègre une interface graphique lourde (bureau, fenêtres, gestion de la souris) et des logiciels pour l'utilisateur final (navigateur internet, suite bureautique). À l'inverse, Ubuntu Server ne possède aucune interface graphique (il se pilote exclusivement en ligne de commande CLI) et intègre par défaut des services orientés réseau (comme un serveur SSH).
* **Choix pour le Cloud :** On choisit l'édition Server dans le Cloud car elle consomme beaucoup moins de ressources (RAM, CPU), est plus stable, plus performante et présente une surface d'attaque réduite (moins de failles de sécurité) que la version Desktop.

#### Question 3 : Que signifie LTS ? Combien d'années de mises à jour de sécurité standard une version LTS reçoit-elle ?
* **Signification :** LTS signifie **Long Term Support** (Support à Long Terme).
* **Durée du support :** Une version LTS d'Ubuntu Server bénéficie de **5 ans** de mises à jour de sécurité standard gratuites.

---

## PARTIE 3 — Créer et configurer la machine virtuelle

###  Commandes exécutées (Création alternative via CLI / VBoxManage)
```powershell
# Définition des variables d'environnement pour la création
$vm  = "ubuntu-srv"
$dir = "$HOME\VirtualBox VMs\$vm"
$iso = "$HOME\Downloads\ubuntu-26.04-live-server-amd64.iso"

# 1. Création et enregistrement de la machine virtuelle
VBoxManage createvm --name $vm --ostype Ubuntu_64 --register

# 2. Configuration du matériel de base et de la règle de redirection de port NAT pour le SSH
VBoxManage modifyvm $vm --cpus 2 --memory 2048 --vram 16 --nic1 nat --natpf1 "ssh,tcp,127.0.0.1,2222,,22"

# 3. Création du disque dur virtuel dynamique de 25 Go
VBoxManage createmedium disk --filename "$dir\$vm.vdi" --size 25600 --format VDI

# 4. Ajout et liaison du contrôleur de stockage SATA
VBoxManage storagectl $vm --name SATA --add sata --controller IntelAhci
VBoxManage storageattach $vm --storagectl SATA --port 0 --device 0 --type hdd --medium "$dir\$vm.vdi"

# 5. Attachement de l'image ISO Ubuntu au lecteur DVD virtuel
VBoxManage storageattach $vm --storagectl SATA --port 1 --device 0 --type dvddrive --medium $iso

# 6. Vérification complète des informations de la VM créée
VBoxManage showvminfo $vm
```

###  Réponses aux questions

#### Question 1 : Quelle différence entre un disque alloué dynamiquement et un disque de taille fixe ? Quelle place occupe le fichier `.vdi` juste après sa création ?
* **Différences :** Un disque de taille fixe réserve immédiatement la totalité de l'espace alloué (25 Go) sur le disque dur physique de l'ordinateur hôte. Un disque alloué dynamiquement n'occupe au départ que l'espace des structures de base et grandit automatiquement au fur et à mesure de l'écriture de nouveaux fichiers dans la VM.
* **Espace initial :** Juste après sa création, le fichier `.vdi` dynamiquement alloué n'occupe que **quelques mégaoctets** (généralement autour de 2 Mo).

#### Question 2 : Pourquoi désactive-t-on l'installation sans surveillance pour ce TP ?
On désactive l'installation sans surveillance afin de pouvoir réaliser manuellement toutes les étapes de configuration à l'écran : choix de la langue (Français), configuration du stockage avec LVM, et création manuelle du compte utilisateur `etudiant` requis par le sujet.

#### Question 3 : En mode NAT, pourquoi l'hôte ne peut-il pas joindre directement la VM ? À quoi sert la redirection 2222 → 22 ?
* **Isolement NAT :** En mode NAT, la VM est isolée dans un sous-réseau privé virtuel et dissimulée derrière le pare-feu/routeur intégré de l'hyperviseur. L'hôte ne peut pas envoyer de paquets directement à l'adresse IP interne de la VM (`10.0.2.15`) car celle-ci n'est pas routable depuis le système Windows hôte.
* **Utilité de la redirection :** La redirection de port agit comme une passerelle. Elle configure VirtualBox pour écouter sur le port `2222` du PC hôte Windows (`127.0.0.1`) et rediriger automatiquement tout ce trafic entrant vers le port `22` (port standard SSH) de la VM.

---

## PARTIE 4 — Installer Ubuntu Server

###  Commandes de fin d'installation (Dans la console de la VM)
Aucune commande textuelle n'est requise pendant l'assistant visuel d'installation d'Ubuntu. Une fois le processus terminé et le système redémarré, la première connexion s'effectue sur l'invite de commande (TTY) de la console :
```bash
# Saisie de l'identifiant : etudiant
# Saisie du mot de passe configuré lors de l'installation
```

###  Réponses aux questions

#### Question 1 : Combien de temps a duré l'installation ? Comparez avec la création d'une instance cloud à partir d'une image prête.
* **Temps mesuré localement :** L'installation complète d'Ubuntu Server a duré environ **30 minutes** (29 minutes et quelques secondes constatées).
* **Comparaison avec le Cloud (IaaS) :** Ce délai met parfaitement en évidence l'intérêt du Cloud IaaS. Alors qu'en local il faut 30 minutes pour installer manuellement l'OS de A à Z via le fichier ISO, un fournisseur cloud comme AWS ou Azure automatise entièrement cette étape et livre une instance fonctionnelle en moins d'une minute grâce au déploiement instantané d'images de disques préinstallées (comme les AMI ou les Azure Images).

#### Question 2 : Quelle adresse IP l'installateur a-t-il obtenue sur `enp0s3` ? Qui l'a attribuée ?
* **Adresse IP obtenue :** `10.0.2.15` avec un masque de sous-réseau `/24` (soit `255.255.255.0`).
* **Attribution :** Cette adresse IP a été attribuée dynamiquement par le **serveur DHCP intégré au moteur NAT de VirtualBox**.

#### Question 3 : Pourquoi installe-t-on le serveur OpenSSH dès l'installation ? Que se passerait-il sur un serveur cloud sans SSH ?
* **Raison de l'installation immédiate :** On installe OpenSSH dès le départ pour permettre l'administration à distance du serveur en ligne de commande de manière sécurisée dès son premier démarrage, évitant ainsi d'avoir à utiliser la console graphique de l'hyperviseur.
* **Scénario dans le Cloud :** Un serveur cloud est une machine distante dépourvue d'écran physique, de clavier et d'interface graphique. S'il était déployé sans serveur SSH actif, il serait totalement inaccessible et inutilisable pour les administrateurs (sauf en passant par une console de secours série propre au fournisseur cloud, ce qui reste une procédure complexe et non standard).

---

## PARTIE 5 — Premiers pas, réseau et accès SSH

###  Commandes exécutées dans la VM Ubuntu (via console ou SSH)
```bash
# 1. Identification du système et de ses ressources matérielles
lsb_release -a
uname -r
hostnamectl
nproc
free -h
df -h /
lsblk

# 2. Observation de la configuration réseau et test de la connectivité Internet
ip -4 addr show
ip route
resolvectl status | grep "DNS Servers"
ping -c 3 ubuntu.com
sudo apt update && sudo apt upgrade -y

# 3. Vérification du statut d'écoute du serveur SSH
systemctl status ssh --no-pager
sudo ss -tlnp | grep :22
```

###  Commandes exécutées sur l'hôte Windows (PowerShell)
```powershell
# 1. Connexion initiale SSH à la VM en utilisant le port redirigé
ssh -p 2222 etudiant@127.0.0.1

# 2. Création et copie d'un fichier texte depuis l'hôte vers la VM via SCP
"Bonjour depuis l'hôte" | Out-File -Encoding utf8 bonjour.txt
scp -P 2222 .\bonjour.txt etudiant@127.0.0.1:~

# 3. Exécution d'une commande à distance via SSH pour lire le fichier copié
ssh -p 2222 etudiant@127.0.0.1 "cat ~/bonjour.txt"

# 4. BONUS : Configuration d'une authentification par clé Ed25519 (sans mot de passe)
ssh-keygen -t ed25519
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 2222 etudiant@127.0.0.1 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
ssh -p 2222 etudiant@127.0.0.1
```

###  Réponses aux questions

#### Question 1 : Relevez l'adresse IP, le masque, la passerelle et le serveur DNS de la VM. À quoi correspond la passerelle `10.0.2.2` ?
* **Adresse IP :** `10.0.2.15`
* **Masque de sous-réseau :** `/24` (soit `255.255.255.0`)
* **Passerelle (Gateway) :** `10.0.2.2`
* **Serveur DNS :** `10.0.2.3` (adresse assignée par défaut par le service réseau VirtualBox)
* **Correspondance de la passerelle `10.0.2.2` :** Cette adresse correspond au moteur NAT virtuel intégré de VirtualBox. Elle agit comme un routeur intermédiaire virtuel qui permet à la machine virtuelle de sortir de son sous-réseau privé isolé pour accéder à Internet en traduisant ses requêtes via la carte réseau physique de l'ordinateur hôte.

#### Question 2 : Pourquoi se connecte-t-on à `127.0.0.1:2222` et non directement à `10.0.2.15:22` ?
L'adresse `10.0.2.15` est une IP privée interne propre au sous-réseau de l'hyperviseur. Elle est totalement invisible et inaccessible depuis le réseau extérieur, y compris depuis votre PC hôte Windows. Pour y accéder, on cible donc notre propre machine via sa boucle locale (localhost `127.0.0.1`) sur le port ouvert `2222`. La règle de redirection de ports configurée dans VirtualBox intercepte ce flux et le transmet automatiquement à la VM sur son port SSH standard (`22`).

#### Question 3 : Que contient le message affiché à la première connexion SSH (empreinte) ? Quel risque permet-il d'éviter ?
* **Contenu :** Ce message affiche l'empreinte cryptographique (fingerprint SHA256) de la clé publique d'hôte générée automatiquement par le serveur OpenSSH installé sur la VM.
* **Risque évité :** Il permet d'éviter l'attaque de l'homme du milieu (**Man-In-The-Middle / MITM**). En validant l'empreinte au premier démarrage, le client SSH mémorise l'identité de la VM. Si un attaquant tente plus tard d'intercepter la connexion réseau en usurpant l'identité de la VM, l'empreinte reçue sera différente et SSH bloquera instantanément la connexion pour des raisons de sécurité.

#### Question 4 : Quels avantages offre l'authentification par clé par rapport au mot de passe ? Pourquoi les fournisseurs cloud l'imposent-ils par défaut ?
* **Avantages :**
  * **Sécurité absolue :** Une clé asymétrique moderne (comme Ed25519) possède une complexité mathématique impossible à deviner ou à casser par force brute informatique.
  * **Automatisation fluide :** Les clés permettent d'automatiser des scripts d'administration et des déploiements d'une machine à une autre de manière hautement sécurisée, sans jamais avoir à stocker ou à manipuler de mot de passe en texte clair.
* **Raison de l'imposition par le Cloud :** Dès qu'une instance de machine virtuelle est démarrée chez un fournisseur cloud, elle reçoit une IP publique exposée sur l'Internet mondial. Elle subit instantanément des milliers de scans et de tentatives d'intrusion par des robots malveillants. Les fournisseurs imposent l'usage exclusif de clés SSH afin d'éradiquer complètement le risque lié aux mots de passe faibles dès le provisionnement de l'instance.

---


## PARTIE 6 — Instantané, clone et arrêt

###  Commandes exécutées sur l'hôte Windows (PowerShell)
```powershell
# 1. Prise d'un instantané (Snapshot) de la VM propre
VBoxManage snapshot ubuntu-srv take apres-installation --description "Ubuntu installé, OpenSSH actif"

# 2. Liste des sauvegardes / instantanés installés
VBoxManage snapshot ubuntu-srv list
```

###  Commandes exécutées dans la VM (via la connexion SSH)
```bash
# 3. Modification visible : installation d'un outil et création d'un fichier témoin
sudo apt install -y htop
touch ~/avant-restauration.txt

# 4. Extinction propre de la VM depuis l'intérieur
sudo poweroff
```

###  Commandes exécutées sur l'hôte Windows (PowerShell)
```powershell
# 5. Restauration de l'instantané initial (pour annuler les modifications précédentes)
VBoxManage snapshot ubuntu-srv restore apres-installation

# 6. Démarrage de la VM principale en mode arrière-plan (Headless)
VBoxManage startvm ubuntu-srv --type headless

# 7. Vérification à distance que la VM a bien été réinitialisée (htop et fichier doivent avoir disparu)
ssh -p 2222 etudiant@127.0.0.1 "which htop; ls ~"

# 8. Extinction propre de la VM principale via le signal ACPI pour le clonage
VBoxManage controlvm ubuntu-srv acpipowerbutton

# 9. Clonage complet et enregistrement de la machine virtuelle sous un nouveau nom
VBoxManage clonevm ubuntu-srv --name ubuntu-clone --register

# 10. Suppression de la règle de redirection SSH existante héritée du parent
VBoxManage modifyvm ubuntu-clone --natpf1 delete ssh

# 11. Configuration du nouveau port local 2223 pour la redirection SSH du clone
VBoxManage modifyvm ubuntu-clone --natpf1 "ssh,tcp,127.0.0.1,2223,,22"

# 12. Démarrage du clone en headless
VBoxManage startvm ubuntu-clone --type headless

# 13. Connexion SSH au clone pour modifier son nom d'hôte interne
ssh -p 2223 etudiant@127.0.0.1 "sudo hostnamectl set-hostname ubuntu-clone; hostname"

# 14. Liste des machines virtuelles actuellement en cours d'exécution
VBoxManage list runningvms

# 15. Analyse comparative de l'espace disque utilisé sur l'hôte par chaque dossier de VM
Get-ChildItem "$HOME\VirtualBox VMs" -Directory | ForEach-Object { "{0} : {1:N2} Go" -f $_.Name, ((Get-ChildItem $_.FullName -Recurse -File | Measure-Object Length -Sum).Sum / 1GB) }

# 16. Extinction du clone via signal ACPI
VBoxManage controlvm ubuntu-clone acpipowerbutton

# 17. Suppression propre et définitive du clone du disque de l'hôte
VBoxManage unregistervm ubuntu-clone --delete
```

###  Réponses aux questions

#### Question 1 : Quelle différence entre un instantané et un clone ? Un instantané est-il une sauvegarde ? Justifiez.
* **Différence :** Un instantané (snapshot) enregistre l'état d'une VM à un instant T au sein de sa propre lignée, mais il reste techniquement dépendant du fichier de disque virtuel d'origine. Un clone est une copie complète et totalement indépendante de la VM, exportée dans de nouveaux fichiers autonomes.
* **Sauvegarde :** **Non, un instantané n'est pas une sauvegarde.** Comme l'instantané dépend techniquement du fichier de disque principal (`.vdi`), si ce fichier d'origine est corrompu ou supprimé sur l'hôte, l'instantané devient totalement inutilisable. Une vraie sauvegarde exige une copie complète des données sur un support ou emplacement physique distinct (ce qui se rapproche davantage du clone).

#### Question 2 : Quels sont les équivalents cloud de l'instantané et du clone (pensez aux snapshots de disque et aux images) ?
* **Équivalent de l'instantané :** Les **snapshots de volume de stockage** (comme les *EBS Snapshots* sur AWS ou les *Azure Disk Snapshots*). Ils permettent de capturer l'état d'un disque cloud à un moment précis pour pouvoir y restaurer le système en cas de mauvaise manipulation.
* **Équivalent du clone :** Les **images système personnalisées** (comme les *AMI* sur AWS ou les *Custom Images* sur Azure). Elles servent de modèles complets pour déployer instantanément de nouvelles instances identiques à la volée.

#### Question 3 : Quelle différence entre `acpipowerbutton` et `poweroff` dans `VBoxManage controlvm` ?
* **`acpipowerbutton` :** Envoie un signal ACPI logiciel au système d'exploitation invité pour lui ordonner de s'éteindre proprement. C'est l'équivalent d'un appui bref sur le bouton physique d'un PC (les services s'arrêtent correctement, les caches disques sont vidés).
* **`poweroff` :** Coupe instantanément et brutalement l'alimentation électrique virtuelle de la machine. C'est l'équivalent d'un débranchement direct de la prise électrique, ce qui présente un risque majeur de perte ou de corruption de données sur le système de fichiers.

#### Question 4 : Pourquoi démarrer une VM en mode headless ? Quel lien avec une instance cloud ?
* **Raison :** Le mode *headless* permet de lancer la machine virtuelle en arrière-plan sans ouvrir de fenêtre d'interface graphique sur l'ordinateur hôte. Cela permet d'économiser les ressources graphiques et CPU de l'ordinateur physique.
* **Lien cloud :** C'est le mode de fonctionnement natif de toutes les instances cloud (IaaS). Les serveurs chez AWS ou Azure tournent sur des hyperviseurs distants sans aucun écran raccordé ; on interagit avec eux exclusivement via le réseau (en SSH ou par API).

#### Question 5 : Pourquoi faut-il changer le port hôte et le nom d'hôte du clone ?
* **Changement de port hôte :** Cela permet d'éviter un conflit d'écoute sur le PC hôte Windows. La première VM occupe déjà le port local `2222`. Si le clone tentait d'utiliser le même port sur l'hôte, VirtualBox ne pourrait pas l'allouer et la redirection réseau échouerait. En choisissant le port `2223`, chaque machine dispose de son propre canal d'accès.
* **Changement de nom d'hôte (`hostname`) :** Cela assure l'unicité de l'identité des machines sur le réseau. Conserver le même nom créerait des ambiguïtés et des conflits dans les fichiers de configuration, la gestion des logs, ou les services d'annuaires réseau (comme le DNS).
