# Guide de Configuration d’un Cluster Proxmox avec Stockage Distribué via DRBD/LINSTOR

Ce guide présente l’intégration de LINSTOR avec Proxmox VE pour obtenir un stockage hautement disponible, facilitant la migration à chaud des machines virtuelles, la réplication en temps réel et la configuration flexible des ressources de stockage.

## 1. Introduction

L’intégration de LINSTOR avec Proxmox VE offre plusieurs avantages :
- **Migrations en direct** : Déplacer vos VMs ou conteneurs sans interruption.
- **Haute disponibilité (HA)** : Réplication complète et mise à jour des disques virtuels sur plusieurs nœuds.
- **Configuration flexible** : Définir le nombre de réplicas pour chaque ressource.
- **Snapshots thin-provisioned** : Gestion efficace de l’espace et préparation à la reprise après sinistre.

Ce guide se focalise sur une installation utilisant l’interface graphique open source LINBIT GUI afin de simplifier la configuration.

## 2. Prérequis et Hypothèses

Pour suivre ce guide, il est supposé que :
- Vous disposez d’un cluster Proxmox VE 8.x composé d’au moins trois nœuds (par exemple, `proxmox-0`, `proxmox-1`, `proxmox-2`).
- Chaque nœud possède un disque physique supplémentaire non utilisé (exemple : `/dev/vdb`).
- Les nœuds Proxmox serviront également de nœuds LINSTOR.
- Vous avez un accès root (ou sudo) pour exécuter les commandes.

## 3. Installation de LINSTOR aux Côtés de Proxmox

### 3.1. Ajouter le Dépôt de Packages Public LINBIT

Sur **tous les nœuds**, exécutez les commandes suivantes pour ajouter le dépôt public LINBIT :
```bash
wget -O /tmp/package-signing-pubkey.asc https://packages.linbit.com/package-signing-pubkey.asc
gpg --yes -o /etc/apt/trusted.gpg.d/linbit-keyring.gpg --dearmor /tmp/package-signing-pubkey.asc
PVERS=8 && echo "deb [signed-by=/etc/apt/trusted.gpg.d/linbit-keyring.gpg] http://packages.linbit.com/public/ proxmox-$PVERS drbd-9" > /etc/apt/sources.list.d/linbit.list
apt update
```
> **Note :** Modifiez la variable `PVERS` en fonction de la version majeure de votre Proxmox VE (ici, 8).

> **Avertissement :** Pour un environnement de production, il est recommandé d’utiliser les dépôts officiels clients LINBIT pour bénéficier du support et des garanties.

### 3.2. Installer les Composants de Base

Installez les paquets nécessaires sur **tous les nœuds** :
```bash
apt -y install proxmox-default-headers drbd-dkms drbd-utils
```

### 3.3. Installer les Composants LINSTOR

Sur **tous les nœuds**, installez LINSTOR avec :
```bash
apt -y install linstor-controller linstor-satellite linstor-client
```
Démarrez le service **LINSTOR Satellite** sur chaque nœud :
```bash
systemctl enable linstor-satellite --now
```
Sur le nœud destiné à être le contrôleur (par exemple, `proxmox-0`), lancez le service LINSTOR Controller :
```bash
systemctl enable linstor-controller --now
```
Sur les autres nœuds, désactivez le contrôleur :
```bash
systemctl disable linstor-controller --now
```

## 4. Installation de l’Interface Graphique LINBIT (GUI)

Puisque le paquet de la GUI n’est pas encore disponible dans les dépôts publics, installez-le depuis le code source :

1. Clonez le dépôt :
   ```bash
   git clone https://github.com/LINBIT/linstor-gui
   cd linstor-gui/
   ```
2. Passez à la dernière version taguée :
   ```bash
   git checkout v$(git describe --tags --abbrev=0 | sed 's/^v//')
   ```
3. Compilez et installez :
   ```bash
   make build
   make install
   ```

> **Important :** L’installation requiert **npm** et une version de Node.js supérieure à 14. Vérifiez avec `node -v`.

## 5. Configuration de LINSTOR via l’Interface Graphique

### 5.1. Accès à la GUI

Depuis un navigateur sur le même réseau, accédez à :
```
http://<adresse_IP_ou_hostname_du_nœud_controller>:3370
```
Par exemple, si `proxmox-0` a l’IP `192.168.222.130`, utilisez :
```
http://192.168.222.130:3370
```

### 5.2. Ajout des Nœuds

1. Dans le menu **"Inventory"**, sélectionnez **"Nodes"**.
2. Cliquez sur le bouton **"Add"** pour ajouter un nouveau nœud.
3. Remplissez le formulaire (exemple : nom `proxmox-0`, IP `192.168.222.130`). Choisissez le type :
   - `Combined` pour un nœud jouant à la fois le rôle de satellite et de contrôleur (pour `proxmox-0`).
   - `Satellite` pour les autres nœuds.

### 5.3. Création des Pools de Stockage

1. Dans le menu **"Inventory"**, choisissez **"Storage Pools"**.
2. Cliquez sur **"Add"** pour créer un nouveau pool.
3. **Préparation côté CLI (sur chaque nœud)** : Créez un groupe de volumes et un pool thin pour réserver 80 % de l’espace du disque.
   ```bash
   vgcreate linstor_vg /dev/vdb && lvcreate -l 80%FREE -T linstor_vg/thinpool
   ```
4. Dans la GUI, passez à l’onglet **"Existing Device"** et renseignez :
   - **Nom du pool** : `pve-storage`
   - **Nœuds** : `proxmox-0`, `proxmox-1`, `proxmox-2`
   - **Type** : `LVM_THIN`
   - **Volume group** : `linstor_vg/thinpool`
5. Cliquez sur **"Submit"** pour créer le pool.

### 5.4. Création d’un Groupe de Ressources LINSTOR

1. Dans le menu **"Storage Configuration"**, sélectionnez **"Resource Groups"**.
2. Cliquez sur **"Add"** et remplissez les champs suivants :
   - **Nom** : `pve-rg`
   - **Description** : `Stockage Proxmox VE`
   - **Storage Pool** : `pve-storage`
   - **Place Count** : `3` (pour répliquer sur les trois nœuds)
3. Soumettez pour finaliser la création du groupe.

## 6. Installation et Configuration du Plugin Proxmox LINSTOR

### 6.1. Installer le Plugin

Sur **tous les nœuds Proxmox**, exécutez :
```bash
apt -y install linstor-proxmox
```

### 6.2. Configurer le Plugin

1. Proxmox stocke sa configuration de stockage dans `/etc/pve/storage.cfg`. Sur le nœud maître (par exemple, `proxmox-0`), éditez ou créez le fichier avec :
   ```bash
   cat << EOF | sudo tee /etc/pve/storage.cfg
   drbd: linstor_storage
       content images, rootdir
       controller 192.168.222.130
       resourcegroup pve-rg
   EOF
   ```
   - **content** : Types de données (images pour VM et rootdir pour conteneurs).
   - **controller** : Adresse IP du nœud LINSTOR controller (ici, `192.168.222.130`).
   - **resourcegroup** : Le groupe de ressources défini précédemment.

2. Les changements se propagent automatiquement dans le cluster Proxmox.

3. Redémarrez les services Proxmox sur **tous les nœuds** :
   ```bash
   systemctl restart pve-cluster pvedaemon pvestatd pveproxy pve-ha-lrm
   ```

## 7. Utilisation de LINSTOR dans Proxmox

### 7.1. Accès à l’Interface Web de Proxmox

- Connectez-vous via l’URL adaptée (par exemple, `https://192.168.222.130:8006`) avec vos identifiants habituels.
- Dans la vue du serveur, développez chaque nœud pour vérifier que le stockage `linstor_storage` est bien présent.

### 7.2. Création et Gestion des Machines Virtuelles

- Lors de la création d’une nouvelle VM, sélectionnez `linstor_storage` comme stockage pour les disques.
- Dès la création d’un VM, LINSTOR crée automatiquement une ressource associée dans le groupe `pve-rg` et réplique les données sur l’ensemble des nœuds.
- **Migration et HA** :  
  - Vous pouvez migrer une VM (à condition qu’elle ne possède pas de lecteur CD/DVD monté avec un ISO actif) d’un nœud à l’autre via l’interface Proxmox.
  - Pour une haute disponibilité, sélectionnez une VM et activez la gestion HA via le menu **"More" > "Manage HA"**. En cas de défaillance du nœud hôte, la VM sera redéployée sur un nœud sain grâce à la réplication en temps réel assurée par DRBD.

## 8. Conclusion et Prochaines Étapes

Ce guide vous permet de déployer un stockage distribué et hautement disponible avec Proxmox et LINSTOR. Vous bénéficiez ainsi d’une infrastructure résiliente, adaptée aux environnements virtualisés exigeants.

**Prochaines étapes :**
- Testez la migration et la haute disponibilité de vos machines virtuelles.
- Explorez la documentation de LINSTOR pour approfondir des options avancées (comme la configuration d’un contrôleur LINSTOR hautement disponible).
- Consultez les forums et ressources LINBIT pour partager vos retours d’expérience et poser des questions en cas de besoin.
