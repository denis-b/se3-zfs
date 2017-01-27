
#installation

Pour faire simple, on installera le système sur une partition ext3 classique, pour ne pas s'embêter mettre un ssd de 30 Go à 40€ ! mais n'importe quel disque fera l'affaire. Il est tout à fait possible de migrer un se3 existant, il suffit d'arrêter tous les services, de démonter /home et /var/se3, puis d'installer et configurer ZFS

##ZfsOnLinux

Il faut ajouter les sources du paquet, puis l'installer :

 apt-get install lsb-release
 wget http://archive.zfsonlinux.org/debian/pool/main/z/zfsonlinux/zfsonlinux_6_all.deb
 dpkg -i zfsonlinux_6_all.deb
 wget http://zfsonlinux.org/4D5843EA.asc -O - | apt-key add -

puis ensuite on installe...

 apt-get update
 apt-get install debian-zfs

Normalement ensuite 
 zpool -v
doit renvoyer une info sur zfs

##Création du pool
On fait du raidz2, qui est en gros l'équivalent d'un RAID6. Ce qui veut dire que 2 disques peuvent tomber en panne. C'est le meilleur compromis pour être tranquille...

Si on a un seul paquet de disques de perfs équivalentes, on fait un seul pool. Si on a des paquets de disques différents, on les regroupera dans des pools différents. On peut ensuite agrandir un pool en faisant l'équivalent d'un RAID0...

Par exemple voici ma config : 

 # zpool list -v
 NAME   SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
 data0  1,36T   779G   613G         -    52%    55%  1.00x  ONLINE  -
  raidz2  1,36T   779G   613G         -    52%    55%
    wwn-0x5000c5000f3fca33      -      -      -         -      -      -
    wwn-0x5000c5000f3ffbef      -      -      -         -      -      -
    wwn-0x5000c50012dd7287      -      -      -         -      -      -
    wwn-0x5000c50012ddfb6f      -      -      -         -      -      -
    wwn-0x5000c50012dee903      -      -      -         -      -      -
 data1  4,53T  1,92T  2,61T         -    23%    42%  1.00x  ONLINE  -
  raidz2  2,27T  1,58T   700G         -    38%    69%
    ata-ST500NM0003-9ZM172_Z1W406ZJ      -      -      -         -      -      -
    scsi-SATA_ST3500514NS_9WJ055D3      -      -      -         -      -      -
    scsi-SATA_ST3500514NS_9WJ057YX      -      -      -         -      -      -
    scsi-SATA_ST3500514NS_9WJ05FYQ      -      -      -         -      -      -
    scsi-SATA_ST3500514NS_9WJ05G2H      -      -      -         -      -      -
  raidz2  2,27T   346G  1,93T         -     8%    14%
    scsi-SATA_ST3500514NS_9WJ051XJ      -      -      -         -      -      -
    scsi-SATA_ST3500514NS_9WJ055WD      -      -      -         -      -      -
    scsi-SATA_ST3500514NS_9WJ05769      -      -      -         -      -      -
    scsi-SATA_ST3500514NS_9WJ0576F      -      -      -         -      -      -
    scsi-SATA_ST3500514NS_9WJ05GJG      -      -      -         -      -      -

J'ai 2 pools, qui correspondent à des type de disques différents (SAS et SATA). Dans le deuxième, j'ai mis 2 raidz2 de 5 disques en parallèle, ce qui me permet d'améliorer les perfs. L'optimal est de grouper les disques par paquets de 5-8, donc après tout dépend de la config...

pour créer la config c'est tout simple : on repère l'uid des disques qui nous intéresse : 

 ls -l /dev/disk/by-id/
 
 /dev/disk/by-id/scsi-SATA_ST3500514NS_9WJ051XJ
 /dev/disk/by-id/scsi-SATA_ST3500514NS_9WJ055D3
 /dev/disk/by-id/scsi-SATA_ST3500514NS_9WJ055WD
 /dev/disk/by-id/scsi-SATA_ST3500514NS_9WJ05769
 
et ensuite on crée le pool : 

 zpool -f create -o ashift=12 data0 raidz2 scsi-SATA_ST3500514NS_9WJ051XJ scsi-SATA_ST3500514NS_9WJ055D3 scsi-SATA_ST3500514NS_9WJ055WD scsi-SATA_ST3500514NS_9WJ05769

par exemple pour un raidz2 à 4 disques

##montage des disques
ZFS ne crée pas de partitions, mais des volumes Zvol. Donc pas besoin de se compliquer la vie ! Le montage est fait automatiquement par zfs, il faud donc penser à supprimer /home et /var/se3 dans /etc/fstab

 zfs create data0/se3home 
 zfs create data0/var/se3 
 zfs set mountpoint=/home data0/se3home
 zfs set mountpoint=/var/se3 data0/varse3

Et c'est fini ! on notera qu'il n'y a pas besoin de donner une taille de partition.

En revanche il est fortement conseillé d'activer la compression des données : l'impact cpu est faible, et on gagne 30% d'espace de stockage et de meilleures perfs !

 zfs set compression=lz4 data0

##Optimisation
On a un disque SSD de 256 Go, on va s'en servir de cache L2ARC : cela veut dire que zfs va avec des algorithmes sophistiqués mettre en cache sur le SSD toutes les données fréquemment utilisées. L'effet est garanti : les accès aux fichiers depuis les clients deviennent ultra-rapides !

 zfs -f add data0 scsi-SATA_Kingston_SHPM2250026B7253006A69

pour optimiser le fonctionnement avec samba, il faut passer les options : 

 zfs set xattr=sa data0
 zfs set relatime=off data0
 zfs set atime=off data0
 zfs set acltype=posixacl data0

C'est l'équivalent des options passées au montage dans fstab

##migration des données

Rien de spécial, restaurer les données avec rsync depuis le backup...

Redémarrer les services se3, rafraîchir les quotas en lançant
 /usr/share/se3/scripts/quota_fixer_mysql.sh Toutlemonde Toutespartitions actu

##mise en place des instantanés

Une des fonctionnalités les plus intéressantes est la possibilité de faire de façon instantanée et à coût nul des snapshots (shadow copy) qui seront directement visibles depuis les clients windows dans l'explorateur ( menu contextuel du bouton de droite ).
