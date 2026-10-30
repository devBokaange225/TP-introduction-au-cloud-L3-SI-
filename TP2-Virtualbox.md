# TP 2 — Deux machines virtuelles en réseau avec VirtualBox
## Trinôme : GRAH GRACE DIVINE SEPHORA / BOKA ANGE ERNEST / ZOBO YVANN JUNIOR

---

## PARTIE 1 — Créer les VM par clonage

###  Commandes exécutées dans l'environnement

**1. Éteindre l'image source et vérifier le statut (PowerShell Hôte) :**
```powershell
$env:Path += ";C:\Program Files\Oracle\VirtualBox"
VBoxManage controlvm Ubuntu-srv acpipowerbutton
VBoxManage list runningvms
```

**2. Cloner les instances et configurer les ressources (PowerShell Hôte) :**
```powershell
Measure-Command { VBoxManage clonevm Ubuntu-srv --name web --register } | Select-Object TotalSeconds
VBoxManage clonevm Ubuntu-srv --name client --register
VBoxManage modifyvm web    --cpus 1 --memory 1024
VBoxManage modifyvm client --cpus 1 --memory 1024
```

**3. Configurer le réseau NAT « tp-cloud » et ses redirections (PowerShell Hôte) :**
```powershell
VBoxManage natnetwork add --netname tp-cloud --network "10.10.0.0/24" --enable --dhcp off
VBoxManage natnetwork modify --netname tp-cloud --port-forward-4 "ssh-web:tcp::2222::22"
VBoxManage natnetwork modify --netname tp-cloud --port-forward-4 "ssh-client:tcp::2223::22"
VBoxManage natnetwork modify --netname tp-cloud --port-forward-4 "http-web:tcp::8080::80"
VBoxManage natnetwork list
```

**4. Brancher les cartes réseau et valider (PowerShell Hôte) :**
```powershell
VBoxManage modifyvm web    --nic1 natnetwork --nat-network1 tp-cloud
VBoxManage modifyvm client --nic1 natnetwork --nat-network1 tp-cloud
VBoxManage showvminfo web | Select-String "Memory size|Number of CPUs|NIC 1"
```

**5. Configuration interne et IP de la VM « web » (Console Graphique VM web) :**
```bash
sudo hostnamectl set-hostname web
sudo sed -i 's/ubuntu-srv/web/g' /etc/hosts
sudo rm -f /etc/machine-id && sudo systemd-machine-id-setup
sudo rm -f /etc/ssh/ssh_host_* && sudo ssh-keygen -A

sudo tee /etc/netplan/99-tp.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [10.10.0.10/24]
      routes:
        - to: default
          via: 10.10.0.1
      nameservers:
        addresses:
EOF
sudo chmod 600 /etc/netplan/99-tp.yaml
sudo netplan apply
sudo reboot
```

**6. Configuration interne et IP de la VM « client » (Console Graphique VM client) :**
```bash
sudo hostnamectl set-hostname client
sudo sed -i 's/ubuntu-srv/client/g' /etc/hosts
sudo rm -f /etc/machine-id && sudo systemd-machine-id-setup
sudo rm -f /etc/ssh/ssh_host_* && sudo ssh-keygen -A

sudo tee /etc/netplan/99-tp.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses: [10.10.0.20/24]
      routes:
        - to: default
          via: 10.10.0.1
      nameservers:
        addresses:
EOF
sudo chmod 600 /etc/netplan/99-tp.yaml
sudo netplan apply
sudo reboot
```

**7. Nettoyage des empreintes SSH et tests d'identification (PowerShell Hôte) :**
```powershell
ssh-keygen -R ":2222"
ssh-keygen -R ":2223"
ssh -p 2222 etudiant@127.0.0.1 "hostnamectl; lsb_release -a; uname -r"
ssh -p 2223 etudiant@127.0.0.1 "hostname; ip -4 -br addr"
```

###  Réponses aux Questions (Partie 1)

**Q1. Combien de temps a pris le clonage de web ? Comparez avec la durée de l'installation d'Ubuntu au TP précédent, puis avec la mise en service d'un serveur physique.**
- **Temps mesuré :** Le clonage de la machine virtuelle `web` a pris précisément **33,96 secondes** (environ 34 s).
- **Comparaison avec l'installation :** Lors du TP précédent, l'installation complète à partir de l'image ISO a nécessité environ 30 minutes. Le clonage est donc **environ 50 fois plus rapide** (34 secondes contre 1800 secondes).
- **Comparaison avec un serveur physique :** La mise en service d'un serveur physique prend plusieurs jours voire semaines (achat, livraison, montage en rack, câblage).
- **Conclusion :** Le clonage met en évidence l'agilité du Cloud Computing : un serveur est instancié à la volée en quelques secondes à partir d'un modèle préexistant.

**Q2. Pourquoi faut-il changer le nom d'hôte, l'identifiant de machine et les clés SSH d'un clone ? Quel risque y aurait-il à garder les mêmes clés SSH sur 50 serveurs ?**
- **Nécessité de personnalisation :** Un clone est une copie conforme bit à bit. Sans cela, les machines partagent le même nom d'hôte (conflits réseau), le même identifiant de machine (bogues de supervision) et les mêmes clés SSH. Dans le Cloud public, ce rôle est géré automatiquement par `cloud-init`.
- **Le risque des clés SSH identiques :** Si 50 serveurs partagent les mêmes clés privées SSH hôtes (`/etc/ssh/ssh_host_*`) :
  - *Effet domino :* Si un seul serveur est compromis, l'attaquant récupère la clé privée et prend le contrôle des 49 autres sans forcer leurs protections.
  - *Homme du milieu (MitM) :* Un attaquant peut usurper l'identité de n'importe quel serveur pour intercepter les flux et voler les identifiants à leur insu.

**Q3. Relevez la version d'Ubuntu, le noyau, le nombre de vCPU et la RAM de web. Quel service cloud joue le rôle de ubuntu-srv (pensez aux AMI d'AWS) ?**
- **Caractéristiques de la VM web :** 
  - Version d'Ubuntu : Ubuntu 26.04 LTS
  - Nombre de vCPU : 1 vCPU
  - Mémoire RAM : 1024 Mo (1 Go)
- **Rôle cloud de ubuntu-srv :** Elle sert d'image de référence. Dans le Cloud, ce rôle correspond aux **AMI (Amazon Machine Images)** chez AWS, ou aux **Images de la Place de marché** chez Microsoft Azure et GCP.

---

## PARTIE 2 — Relier les VM en réseau

###  Commandes exécutées dans l'environnement

**1. Connexion SSH à la VM client depuis l'hôte (PowerShell Hôte) :**
```powershell
ssh -p 2223 etudiant@127.0.0.1
```

**2. Tests de connectivité réseau (Console Interne VM client) :**
```bash
ping -c 4 10.10.0.10
ping -c 2 10.10.0.1
ping -c 2 1.1.1.1
ip route
resolvectl status | grep "DNS Servers"
```

**3. Configuration du fichier hosts local et test par nom (Console Interne VM client) :**
```bash
echo "10.10.0.10 web" | sudo tee -a /etc/hosts
ping -c 2 web
exit
```

**4. Test d'isolation depuis le PC de l'hôte (PowerShell Hôte) :**
```powershell
ping -n 2 10.10.0.10
```

###  Réponses aux Questions (Partie 2)

**Q1. Les deux VM sont-elles dans le même sous-réseau ? Justifiez avec leurs adresses et le masque /24.**
- **Réponse :** Oui, les deux machines virtuelles appartiennent au même sous-réseau.
- **Justification :** Le masque `/24` (`255.255.255.0`) indique que les 3 premiers octets identifient le réseau, soit `10.10.0.0`. La VM `web` a pour adresse `10.10.0.10` et la VM `client` `10.10.0.20`. Partageant la même portion réseau (`10.10.0.X`), elles se trouvent dans la même plage d'adresses hôtes valides (de `10.10.0.1` à `10.10.0.254`).

**Q2. Quelle est la passerelle par défaut de client ? Quel est son rôle ? À quel élément d'un VPC (AWS) ou d'un VNet (Azure) correspond-elle ?**
- **Passerelle par défaut :** L'adresse est `10.10.0.1` (le Routeur NAT virtuel).
- **Rôle :** Servir de point de sortie. Quand la VM envoie des paquets en dehors de son sous-réseau (ex: Internet `1.1.1.1`), elle les transmet à cette passerelle qui les route vers l'extérieur.
- **Équivalent Cloud :** Cela correspond à une **Internet Gateway (IGW)** sur AWS, ou à une **Route Table par défaut** gérant le trafic sortant sur Azure.

**Q3. Quel est le temps de réponse moyen du ping de client vers web ? Pourquoi est-il si faible ?**
- **Temps moyen constaté :** Inférieur à 1 milliseconde (généralement entre `0.2 ms` et `0.5 ms`).
- **Justification :** Aucun câble physique ni commutateur réel n'est traversé. Les deux instances s'exécutent sur la même machine hôte physique. Le flux circule intégralement en mémoire vive (RAM) via le commutateur virtuel de l'hyperviseur.

**Q4. Le ping depuis Windows vers 10.10.0.10 aboutit-il ? Pourquoi ? Comment l'hôte arrive-t-il malgré tout à joindre web en SSH ?**
- **Résultat du ping depuis Windows :** Non, le ping échoue par dépassement de délai (*Request timed out*).
- **Le Pourquoi :** L'hôte Windows ne possède aucune route vers la plage privée virtuelle `10.10.0.0/24`, isolée derrière le routeur NAT de VirtualBox.
- **Comment l'hôte réussit le SSH :** Grâce aux **redirections de ports (Port Forwarding)**. En se connectant sur son adresse locale `127.0.0.1` au port `2222`, VirtualBox intercepte le flux et le transmet au port `22` (SSH) de la VM `web` (`10.10.0.10`).

---

## PARTIE 3 — Publier une page web et la protéger

###  Commandes exécutées dans l'environnement

**1. Connexion SSH à la VM web depuis l'hôte (PowerShell Hôte) :**
```powershell
ssh -p 2222 etudiant@127.0.0.1
```

**2. Déploiement du site et du pare-feu (Console Interne VM web) :**
```bash
sudo apt update && sudo apt install -y nginx
systemctl status nginx --no-pager
echo "<h1>Binôme NOM1 - NOM2 : ma première VM en réseau</h1>" | sudo tee /var/www/html/index.html
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
exit
```

**3. Test de blocage réseau (Console Interne VM client, via SSH 2223) :**
```bash
curl -m 5 http://web
```

**4. Ouverture du port HTTP (Console Interne VM web, via SSH 2222) :**
```bash
sudo ufw allow 'Nginx HTTP'
sudo ufw status numbered
sudo ss -tlnp
exit
```

**5. Test de validation final (Console Interne VM client ou Navigateur Hôte) :**
```bash
curl -s http://web
```

###  Réponses aux Questions (Partie 3)

**Q1. Que renvoie curl quand le port 80 est fermé (message et code retour) ? Pourquoi le navigateur reste-t-il bloqué au lieu d'afficher une erreur immédiate ?**
- **Sortie de curl :** Elle n'affiche aucune page et renvoie une erreur de délai dépassé (*Connection timed out*), associée au code retour **28**.
- **Comportement du navigateur :** Le navigateur reste bloqué en attente (il charge dans le vide) car la règle par défaut du pare-feu `ufw` est configurée sur **DROP** (rejeter silencieusement). Au lieu d'envoyer un message de rejet explicite au client (comme un paquet `REJECT`), le pare-feu détruit purement et simplement le paquet entrant. Le client attend donc jusqu'à l'expiration du *timeout* de son propre navigateur.

**Q2. Sur quel port et sur quelles adresses Nginx écoute-t-il ? Appuyez-vous sur la sortie de ss -tlnp.**
- **Port d'écoute :** Le processus Nginx écoute spécifiquement sur le port **80** (dédié au trafic HTTP).
- **Adresses associées :** La colonne d'adresse locale affiche `0.0.0.0:80` (et/ou `[::]:80` pour l'IPv6). Cette syntaxe avec des zéros indique que le serveur web est configuré pour écouter et accepter les requêtes sur **toutes les interfaces réseau de la VM** (la boucle locale `127.0.0.1` ainsi que l'interface réseau `enp0s3`).

**Q3. Le pare-feu ufw joue ici le rôle d'un groupe de sécurité du cloud. Donnez deux ressemblances et une différence.**
- **Deux ressemblances :**
  1. *Principe du moindre privilège :* Tout le trafic entrant est bloqué par défaut, seules les connexions explicitement déclarées et nécessaires (SSH, HTTP) sont ouvertes.
  2. *Filtrage ciblé :* Définition de règles précises basées sur les protocoles (TCP/UDP) et les **numéros de ports**.
- **Une différence :** `ufw` est un pare-feu logiciel interne à l'OS (il consomme des ressources CPU/RAM de la VM pour analyser les paquets). Un **Groupe de Sécurité (Security Group)** Cloud est un pare-feu virtuel externe géré au niveau de l'infrastructure réseau de l'hyperviseur, bloquant le trafic indésirable avant même qu'il n'atteigne la carte réseau de la VM.

**Q4. Dans ce TP, vous êtes le client d'un service IaaS. Reprenez le tableau « Qui gère quoi ? » du cours : quelles couches avez-vous gérées vous-même, et lesquelles VirtualBox a-t-il fournies ?**
En tant que client d'un modèle IaaS (pile à 9 couches), la responsabilité est partagée :
- **Les couches gérées par nous-mêmes (le Client) — 5 couches :** Nous avons pris la responsabilité de tout ce qui s'exécute dans la VM (la partie logicielle) :
  - *Application & Données :* Le code HTML personnalisé et le fichier `index.html` contenant les noms du binôme.
  - *Middleware & Runtime :* L'installation et la gestion opérationnelle de **Nginx** ainsi que l'environnement d'exécution.
  - *Système d'exploitation :* La configuration interne d'Ubuntu Server (Netplan IP fixes, hostname, pare-feu `ufw`, clés SSH).
- **Les couches fournies par VirtualBox (le Fournisseur Cloud) — 4 couches :** Le matériel et sa virtualisation :
  - *Virtualisation :* L'hyperviseur gérant le clonage, l'isolement et la limite des ressources (1 vCPU, 1 Go RAM).
  - *Serveurs & Stockage :* La puissance de calcul brute et les fichiers disques virtuels (`.vdi` dynamiques).
  - *Réseau :* Le sous-réseau privé virtuel `tp-cloud` (`10.10.0.0/24`), sa passerelle (`10.10.0.1`) et les règles de redirection de ports.

---

## PARTIE 4 — Mesurer et nettoyer

###  Commandes exécutées dans l'environnement

**1. Analyse des ressources internes (Console Interne VM web) :**
```bash
free -h
df -h /
nproc
exit
```

**2. Analyse de la RAM physique consommée sur Windows (PowerShell Hôte) :**
```powershell
Get-Process -Name "VirtualBoxVM" | Select-Object Id, ProcessName, @{Name="RAM Réelle (Mo)"; Expression={[math]::round($_.WorkingSet / 1MB, 2)}}
```

**3. Analyse de la taille des disques réels sur Windows (PowerShell Hôte) :**
```powershell
Get-ChildItem "$HOME\VirtualBox VMs" -Directory | ForEach-Object { "{0} : {1:N2} Go" -f $_.Name, ((Get-ChildItem $_.FullName -Recurse -File | Measure-Object Length -Sum).Sum / 1GB) }
```

**4. Arrêt propre et suppression complète du laboratoire (PowerShell Hôte) :**
```powershell
$env:Path += ";C:\Program Files\Oracle\VirtualBox"
VBoxManage controlvm web acpipowerbutton
VBoxManage controlvm client acpipowerbutton
VBoxManage list runningvms
VBoxManage unregistervm web --delete
VBoxManage unregistervm client --delete
VBoxManage natnetwork remove --netname tp-cloud
VBoxManage list vms
```

###  Réponses aux Questions (Partie 4)

**Q1. La VM web voit-elle 1 Go de RAM ? Combien en consomme-t-elle réellement sur l'hôte ? Comment expliquez-vous l'écart ?**
- **Mesures au sein de la VM web :** Oui, la VM web détecte bien une mémoire virtuelle totale allouée de **956 Mi** (environ 1 Go). D'après la commande `free -h`, l'OS et Nginx au repos n'utilisent activement que **193 Mi** (`used`), le reste étant réparti entre mémoire libre (**492 Mi** `free`), cache (**291 Mi** `buff/cache`) et mémoire disponible (**762 Mi** `available`).
- **Consommation réelle sur l'hôte Windows :** D'après notre mesure PowerShell (`Get-Process`), le processus principal de la machine virtuelle (`VirtualBoxVM.exe`) consomme réellement **793.02 Mo** de mémoire RAM physique sur l'hôte Windows.
- **Explication de l'écart :** C'est le résultat du mécanisme d'**allocation dynamique de la mémoire** (*Thin Provisioning* de la RAM) de l'hyperviseur. VirtualBox présente 1 Go virtuel à la VM web, mais ne réserve sur la RAM physique de Windows que ce que l'OS invité a manipulé (RAM active + structures noyau + cache en RAM). Ici, la consommation s'arrête à **793.02 Mo**, ce qui permet d'économiser plus de 200 Mo de RAM réelle sur le PC physique.

**Q2. Le disque d'un clone fait 25 Go pour la VM. Quelle place occupe-t-il réellement sur l'hôte ? Pourquoi ?**
- **Place réelle occupée sur l'hôte Windows :** D'après notre mesure PowerShell des dossiers, la VM web n'occupe réellement que **7,17 Go** (et la VM client **7,18 Go**).
- **Explication de l'écart :** Cet écart s'explique par l'utilisation de disques virtuels au format **dynamiquement alloué** (fichiers `.vdi`). Le conteneur sur le disque dur Windows ne grandit que lorsque de nouvelles données (paquets, logs) sont écrites dans la VM. De plus, le fait que les clones ne pèsent que ~7 Go prouve qu'ils s'appuient efficacement sur l'image de base `ubuntu-srv` qui ne pèse que **1,52 Go**.

**Q3. Quelle différence entre acpipowerbutton, poweroff et unregistervm --delete ?**
- `acpipowerbutton` : Transmet une consigne d'extinction logicielle propre à l'OS (appui court sur le bouton physique). Elle offre le temps à Ubuntu d'interrompre proprement ses processus (Nginx, SSH) et de démonter ses disques sans aucun risque de corruption.
- `poweroff` : Coupe instantanément l'alimentation électrique virtuelle de la machine (débrancher la prise). L'arrêt est immédiat mais expose à des pertes de données ou à une corruption du système de fichiers.
- `unregistervm --delete` : Désenregistre la VM de VirtualBox et **efface de manière définitive et irréversible** tous les fichiers de configuration et disques virtuels (`.vdi`) associés sur l'hôte Windows.

**Q4. Dans le cloud, une VM arrêtée est-elle encore facturée ? Indice : pensez au disque et à l'adresse IP publique.**
- **Réponse :** Oui, une VM arrêtée (*Stopped*) continue de générer des coûts. Le fournisseur de Cloud interrompt uniquement la facturation des ressources de calcul dynamiques (vCPU, RAM) car elles sont libérées pour d'autres clients. En revanche, le client reste facturé pour le **stockage persistant** (le volume de disque conservant l'OS et les fichiers) et pour la réservation exclusive de **l'adresse IP publique statique** bloquée pour son compte. Pour stopper toute facturation, il est obligatoire de détruire définitivement l'instance (**Terminate ou Delete**).
