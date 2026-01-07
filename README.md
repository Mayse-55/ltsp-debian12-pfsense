
# Guide d'installation LTSP sur Debian 12 avec pfSense

---

## 📋 Table des matières
1. [Prérequis](#prérequis)
2. [Installation du système de base](#installation-du-système-de-base)
3. [Installation du bureau XFCE](#installation-du-bureau-xfce)
4. [Installation et configuration de LTSP](#installation-et-configuration-de-ltsp)
5. [Configuration du mot de passe root](#configuration-du-mot-de-passe-root)
6. [Configuration de ltsp.conf](#configuration-de-ltspconf)
7. [Création du compte utilisateur](#création-du-compte-utilisateur)
8. [Configuration réseau](#configuration-réseau)
9. [Déploiement du client](#déploiement-du-client)
10. [Intégration avec pfSense](#intégration-avec-pfsense)

---

## Prérequis

### Infrastructure nécessaire
- **Serveur LTSP** : VM Debian 12
- **Routeur/Firewall** : pfSense
- **Client(s)** : Machines compatibles PXE boot
- **VM Windows** (facultatif) : Pour administrer pfSense via interface graphique

### Configuration réseau minimale
- Serveur LTSP : IP fixe sur le réseau local
- pfSense : Configuré avec DHCP (sera modifié plus tard)
- Clients : Boot PXE activé dans le BIOS

---

## Installation du système de base

### 1. Mise à jour du système
```bash
sudo apt update
sudo apt upgrade -y
```

### 2. Installation d'OpenSSH (facultatif mais recommandé)
```bash
sudo apt install openssh-server -y
sudo systemctl enable ssh
sudo systemctl start ssh
```

---

## Installation du bureau XFCE

Installation complète de l'environnement de bureau XFCE avec tous les composants nécessaires :

```bash
sudo apt install xfce4 xfce4-goodies lightdm firefox-esr dbus-x11 -y
```

**Composants installés :**
- `xfce4` : Environnement de bureau principal
- `xfce4-goodies` : Applications supplémentaires XFCE
- `lightdm` : Gestionnaire de connexion graphique
- `firefox-esr` : Navigateur web
- `dbus-x11` : Bus de messages pour l'environnement graphique

---

## Installation et configuration de LTSP

### 1. Installation des paquets LTSP
```bash
sudo apt install ltsp dnsmasq nfs-kernel-server squashfs-tools tftpd-hpa ipxe -y
```

**Paquets installés :**
- `ltsp` : Linux Terminal Server Project
- `dnsmasq` : Serveur DHCP/DNS/TFTP léger
- `nfs-kernel-server` : Partage de fichiers réseau
- `squashfs-tools` : Création d'images compressées
- `tftpd-hpa` : Serveur TFTP (sera désactivé au profit de dnsmasq)
- `ipxe` : Firmware de boot réseau

### 2. Arrêt du service tftpd-hpa
LTSP utilise dnsmasq comme serveur TFTP, donc on désactive tftpd-hpa :

```bash
sudo systemctl stop tftpd-hpa.service
sudo systemctl disable tftpd-hpa.service
sudo systemctl status tftpd-hpa.service
```

### 3. Construction de l'image LTSP
```bash
sudo ltsp image /
```
⏱️ **Cette commande peut prendre plusieurs minutes** - elle crée une image compressée du système.

### 4. Configuration des services LTSP
Exécutez ces commandes dans l'ordre :

```bash
sudo ltsp dnsmasq    # Configure dnsmasq pour LTSP
sudo ltsp initrd     # Génère l'initramfs pour le boot
sudo ltsp ipxe       # Configure iPXE
sudo ltsp kernel     # Configure le kernel
sudo ltsp nfs        # Configure les exports NFS
```

---

## Configuration du mot de passe root

### 1. Installation de l'outil de génération de hash
```bash
sudo apt install whois -y
```

### 2. Génération du hash du mot de passe
```bash
mkpasswd -m yescrypt
```
Entrez votre mot de passe souhaité et **copiez le hash généré** (vous en aurez besoin pour ltsp.conf).

### 3. Application du mot de passe root sur le serveur
```bash
sudo usermod --password 'VOTRE_HASH_ICI' root
```
Remplacez `VOTRE_HASH_ICI` par le hash généré à l'étape précédente.

---

## Configuration de ltsp.conf

### 1. Édition du fichier de configuration
```bash
sudo nano /etc/ltsp/ltsp.conf
```

### 2. Contenu du fichier ltsp.conf
Copiez et adaptez la configuration suivante :

```ini
[server]
# IP du serveur LTSP
SERVER="192.168.1.100"

[common]
# Timeout du menu de boot (-1 = pas de timeout)
MENU_TIMEOUT="-1"

[clients]
# Connexion automatique avec le compte 'internet'
AUTOLOGIN=internet
# Pas de demande de reconnexion
RELOGIN=0
# Définir le mot de passe root sur les clients
POST_INIT_SET_ROOT_HASH="section_set_root_hash"
# Inclure la configuration des moniteurs CRT
INCLUDE=crt_monitor
# Montage automatique de la partition /home
FSTAB_x="LABEL=home     /home   ext4    defaults        0       0"
# Serveur d'impression
CUPS_SERVER="localhost"

[set_root_hash]
# Commande pour définir le mot de passe root (remplacez par votre hash)
sed 's|^root:[^:]*:|root:$y$j9T$VOTRE_HASH_COMPLET_ICI:|' -i /etc/shadow

[crt_monitor]
# Configuration pour différentes résolutions d'écran
X_HORIZSYNC="28.0-87.0"
X_VERTREFRESH="43.0-87.0"
X_MODES='"1920x1080" "1680x1050" "1280x720" "1280x800" "1024x768" "800x600" "640x480"'
```

**⚠️ Important :** Remplacez :
- `192.168.1.100` par l'IP de votre serveur LTSP
- `VOTRE_HASH_COMPLET_ICI` par le hash généré précédemment (tout le hash, y compris les `$`)

---

## Création du compte utilisateur

### 1. Création du répertoire home dans /etc
```bash
sudo mkdir -p /etc/home/internet
```

### 2. Création du compte utilisateur
```bash
sudo adduser internet
```
Entrez le mot de passe souhaité pour l'utilisateur `internet`.

---

## Configuration réseau

### Activation du routage IP
```bash
sudo nano /etc/sysctl.conf
```

Décommentez ou ajoutez ces lignes :
```bash
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

Appliquez les changements :
```bash
sudo sysctl -p
```

### Configuration de dnsmasq pour LTSP
```bash
sudo nano /etc/dnsmasq.d/ltsp-dnsmasq.conf
```

Modifiez la plage DHCP selon vos besoins et commentez la ligne proxy si nécessaire :
```conf
# Commentez cette ligne pour désactiver le mode proxy DHCP
#dhcp-range=set:proxy,30.31.8.0,proxy,255.255.255.0

# Exemple de plage DHCP standard
dhcp-range=192.168.1.50,192.168.1.150,255.255.255.0,24h
```

### Redémarrage des services
```bash
sudo systemctl restart dnsmasq.service
sudo reboot
```

### Vérification du compte utilisateur
Après le redémarrage, testez la connexion avec le compte `internet` via la console ou SSH.

---

## Déploiement du client

### 1. Préparation du serveur
Avant de booter un client, assurez-vous que toutes les modifications sont prises en compte :

```bash
sudo ltsp image /
sudo ltsp initrd
sudo ltsp ipxe
```

### 2. Synchronisation du home utilisateur
```bash
cd /home
sudo rsync -av --progress internet /etc/home/
```

### 3. Boot du client en PXE
1. Démarrez le client en mode PXE
2. Attendez environ **1m30** pour le chargement initial
3. Connectez-vous en tant que **root** avec le mot de passe configuré

### 4. Configuration du disque dur du client

#### Identifier la partition
```bash
fdisk -l
```
Repérez la partition que vous souhaitez utiliser pour /home (exemple: `/dev/sda1`)

#### Formater la partition
```bash
sudo mkfs.ext4 /dev/sda1
```
⚠️ **Attention :** Cette commande efface toutes les données de la partition !

#### Définir le label
```bash
sudo e2label /dev/sda1 home
```

#### Redémarrer le client
```bash
sudo reboot
```

### 5. Copie finale du home utilisateur
1. Connectez-vous avec le compte `internet`
2. Ouvrez un terminal
3. Passez en root :
```bash
su -
```
4. Synchronisez le home depuis /etc/home vers /home :
```bash
cd /etc/home
rsync -av --progress internet /home/
```

✅ **Le client est maintenant entièrement configuré !**

---

## Intégration avec pfSense

Cette section permet de transférer la gestion DHCP à pfSense tout en conservant le boot PXE LTSP.

### 1. Désactivation de dnsmasq sur le serveur LTSP
```bash
sudo systemctl stop dnsmasq.service
sudo systemctl disable dnsmasq.service
```

### 2. Activation de tftpd-hpa
```bash
sudo systemctl start tftpd-hpa.service
sudo systemctl enable tftpd-hpa.service
```

### 3. Configuration du serveur DHCP pfSense

#### Accéder à l'interface web pfSense
1. Connectez-vous à l'interface web de pfSense
2. Allez dans **Services → DHCP Server**

#### Configuration TFTP
1. Activez le serveur DHCP si ce n'est pas déjà fait
2. Descendez jusqu'à la section **"TFTP Server"**
3. Cliquez sur **"Display Advanced"**
4. Dans le champ **"TFTP Server"**, entrez l'IP de votre serveur LTSP (exemple: `192.168.1.100`)
5. Cliquez sur **"Hide Advanced"**

#### Configuration Network Booting
1. Trouvez la section **"Network Booting"**
2. Cochez **"Enable network booting"**
3. Dans **"Next Server"**, entrez l'IP de votre serveur LTSP (exemple: `192.168.1.100`)
4. Dans **"Default BIOS file name"**, entrez : `ltsp/ltsp.ipxe`
5. Cliquez sur **"Save"**

### 4. Test du démarrage client
1. Redémarrez un client en mode PXE
2. Le client devrait maintenant :
   - Obtenir une IP du serveur DHCP pfSense
   - Booter via iPXE depuis le serveur LTSP

✅ **Configuration terminée !** Vous pouvez maintenant mettre en place un portail captif ou d'autres fonctionnalités pfSense.

---

## 🔧 Dépannage

### Le client ne boot pas en PXE
- Vérifiez que le boot PXE est activé dans le BIOS
- Vérifiez que le câble réseau est bien branché
- Vérifiez les logs dnsmasq : `sudo journalctl -u dnsmasq -f`

### Erreur "No such file or directory" lors du boot
- Vérifiez que les fichiers existent dans `/srv/tftp/ltsp/`
- Relancez : `sudo ltsp initrd && sudo ltsp ipxe`

### Le client boot mais ne monte pas /home
- Vérifiez que la partition a bien le label "home" : `sudo e2label /dev/sda1`
- Vérifiez la ligne FSTAB_x dans `/etc/ltsp/ltsp.conf`

### Firefox ne se lance pas
- Installez les dépendances manquantes dans l'image :
```bash
sudo ltsp chroot
apt install --install-recommends firefox-esr libgtk-3-0 libdbus-glib-1-2
exit
sudo ltsp image /
```

---

## 📚 Ressources

- [Documentation officielle LTSP](https://ltsp.org/)
- [Documentation pfSense](https://docs.netgate.com/pfsense/)
- [Wiki Debian](https://wiki.debian.org/)

---

## 📝 Notes importantes

- Ce guide est conçu pour un **environnement de test (hors production)**
- Pensez à faire des snapshots de vos VMs avant les modifications importantes
- Adaptez les plages IP et configurations réseau à votre environnement

---

**Auteur :** Mayse  
**Dernière mise à jour :** Janvier 2026  
**Version :** 1.0
