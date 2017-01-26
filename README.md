# se3-zfs
outils et  configuration pour l'installation initiale ou la migration de se3 vers zfs

## Contenu du module
* fichiers de conf spécifiques à une installation SE3
* configuration du paquet zfs-auto-snapshot pour l'automatisation des instantanés
* script de backup automatisé vers un NAS zfs (FreeNAS par exemple)
* script de migration automatisée des données existantes

## prérequis matériel
Passer en ZFS n'est intéressant que si on dispose d'un matériel adapté : il faut disposer en plus du serveur SE3 d'une machine NAS ZFS pour les sauvegardes

pour le serveur SE3 :
* Serveur avec beaucoup de mémoire ECC : 16 Go est un minimum
* Controleur(s) SATA ou SAS HBA : surtout pas de raid matériel
* Au moins 4 disques pour les données : en raidz1 3 + 1 pour la redondance
* un cache SSD Sata ou PCIE 128 go mini

Pour le NAS Freenas :
* N'importe quel PC avec au moins 4 ports SATA
* 8 Go de mémoire, si possible ECC
* Au moins 4 disques
* 2 clés usb  8 Go pour le système (redondance)

Un NAS tout fait fait bien évidemment l'affaire, pourvu qu'il soit en ZFS

## Installation
### Nouvelle configuration
L'installation se fait en installant se3-zfs en premier 
### Migration d'un serveur existant
* installation du NAS FreeNAS
* copie de la clé ssh root@se3 sur root@nas
* copie des données home et varse3 sur le nas avec rsync
* installation zfsonlinux sur se3
* creation du zpool et des dataset home et varse3
* zfs send | receive pour restaurer les données
