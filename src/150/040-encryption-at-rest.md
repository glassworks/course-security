# Cryptage au repos

Outre les privilèges sur les fichiers, nous voulons nous assurer que les données sont cryptées sur le périphérique de stockage. 

Même si l'appareil est volé, ou si un pirate informatique accède à la machine et peut lire directement l'appareil, il ne devrait pas être en mesure de comprendre le contenu sans une clé cryptographique.

Dans cette section, nous allons apprendre à attacher un volume à une VM, à le formater et à ajouter une couche cryptographique.

# Disques vs partitions

Un disque est un périphérique comme un disque dur ou une clé USB. Sur un disque, on peut créer une ou plusieurs _partitions_, qui sont des espaces indépendantes.

Comment voir les disques et partitions ? Cela change en fonction de votre parfum d'Unix. Sur Ubuntu, on utilise la commande `lsblk` (_l_i_s_t _bl_o_c_k devices) :

```bash
kevin%nguni.fr@unix-shell-students:~$ lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
loop0     7:0    0  103M  1 loop /snap/lxd/23541
loop1     7:1    0 63,2M  1 loop /snap/core20/1695
loop2     7:2    0 49,6M  1 loop /snap/snapd/17576
vda     252:0    0 74,5G  0 disk 
├─vda1  252:1    0 74,4G  0 part /
├─vda14 252:14   0    4M  0 part 
└─vda15 252:15   0  106M  0 part /boot/efi
```

Ici, on voit que le disque `vda` contient 3 partitions.

Une partition ou un disque n'est pas obligatoirement monté dans la hiérarchie. `lsblk` nous donne de l'information des périphériques connectés seulement. Il faudrait la monter avant de l'utiliser, et de les voir avec `df`.

## Provisionner un volume dans votre Cloud

Rendez-vous chez votre fournisseur de services en nuage et approvisionnez un nouveau volume (le plus petit et le moins cher que vous puissiez trouver) qui sera attaché à votre VM.

Nous allons maintenant apprendre à formater et à monter ce volume.

## Formater un disque ou partition

Formater un disque ou une partition dépende du parfum Unix que vous utilisez. Dans Ubuntu, on utilise `mkfs` (_m_a_k_e _f_ile _s_ystem).

Par exemple :

```bash
# Formater le périphérique à /dev/sda avec le format "ext4" 
mkfs.ext4 /dev/sda
```

## Monter un périphérique

Si vous branchez une clé USB ou disque dur interne (ou, si vous êtes chez un hébergeur cloud, et vous aimeriez attacher une volume externe), on commence par d'abord lister les périphériques (avec `lsblk`), puis on le monte à l'endroit où on veut :

```bash
mount -o defaults [chemin vers l’appareil block] [chemin de montage]
```

Exemple :

```bash
# Monter le périphérique /dev/sda sur le dossier /mnt/data
mount -o defaults /dev/sda /mnt/data
```

## Monter les disques/partitions/volumes au démarrage

Si on redémarre notre machine UNIX après avoir monter un disque, on perdra nos modifications.

Il faudrait sauvegarder notre manipulation dans un fichier spécial, qui est lu au démarrage. Dans notre cas, le fichier se trouve à `/etc/fstab` :

```bash
kevin%nguni.fr@unix-shell-students:~$ cat /etc/fstab 
LABEL=cloudimg-rootfs   /        ext4   discard,errors=remount-ro       0 1
LABEL=UEFI      /boot/efi       vfat    umask=0077      0 1
```

Pour notre exemple, monter le disque `/dev/sda` (format ext4) sur `/mnt/data` automatiquement au démarrage :

```bash
nano /etc/fstab
# Ajoutez la ligne au fichier, puis sauvegardez :
# /dev/sda /mnt/data ext4 defaults 0 0
```

> :warning: Attention, il vous faut des droits suffisants pour modifier ce fichier !

> :book: Consultez `man fstab` pour en savoir plus du format de ce fichier !

Vous pouvez tester votre modification avec umount et mount:

```bash
# Demonter votre volume
umount /mnt/data

# Re-monter le volume, sans préciser /dev/sda : mount va chercher dans fstab automatiquement
mount /mnt/data
```

Si mount fonctionne correctement, vous pouvez redémarrer votre instance avec :&#x20;

```bash
reboot
```

Actuellement, votre volume est monté et persiste après le redémarrage. Mais il n'est pas crypté.

Forçons Linux à chiffrer toutes les données écrites sur le volume.

Tout d'abord, démontez le volume, et supprimez la ligne que nous avons ajoutée à `/etc/fstab` :

```bash
umount /mnt/data
nano /etc/fstab
# -> ensuite commentez la ligne ajoutée auparavant
```

## Cryptage : LUKS

LUKS est le système utilisé pour chiffrer un volume.

Nous installons LUKS avec la commande `cryptsetup` :

```bash
# Installer
sudo apt install cryptsetup
```

Nous avons déjà vu que quand on parle de la cryptographie, il faudrait nécessairement un (ou plusieurs) clé(s) qui permettent de crypter et décrypter des données. Avec LUKS, nous allons créer et gérer ces clés.

Nous commençons par générer une clé forte, en utilisant l'outil `openssh` par exemple. Plus la clé est longue, plus elle est sécurisée :


```bash
# Générer un mot de passe bien unique
openssl rand -base64 32

# Formater le disque pour cryptage LUKS
cryptsetup -c aes-xts-plain64 -v luksFormat /dev/sda

# On vous demande de fournir un mot de passe fort. 
# Utilisez le mot de passe généré par openssl juste avant
```

Nous avons formaté le volume crypté. Ce volume ne peut pas être lu comme un volume normal. Il faudrait fournir la clé de cryptage. Donc avant de le monter, il faut configurer le périphérique :


```bash
# Configurer le peripherique 
# On précise :
# - l'opération: luksOpen (ouvrir un volume)
# - le péripherique physique concerné : /dev/sda
# - l'alias pour ce volume dans LUKS : encrypteddata
cryptsetup luksOpen /dev/sda encrypteddata

# On vous demande de fournir le mot de passe utilisé à la création du volume

# Le nouveau péripherique devrait être présent
ls /dev/mapper
```

Nous disposons maintenant d'un nouveau périphérique "virtuel" qui se trouve à `/dev/mapper/encrypteddata`. Ce périphérique peut être formaté et monté comme n'importe quel autre périphérique :

```bash
# Formater le nouveau volume
mkfs.ext4 /dev/mapper/encrypteddata

# Si pas déjà fait, créer le point de montage /mnt/encrypteddata
mkdir /mnt/encrypteddata

# Monter
mount /dev/mapper/encrypteddata /mnt/encrypteddata
```

Nous pouvons donc naviguer à `/mnt/encrypteddata` et créer et modifier les fichiers. Nous pouvons aussi préciser ce chemin pour le stockage de nos données d'une base de données SGBDR par exemple.

Mais, avant de le faire, il y a un problème : le volume crypté n'existera uniquement tant que l'instance ne redémarre pas. Au redémarrage, il faudrait fournir de nouveau le mot de passe de cryptage. 

L'avantage de LUKS est qu'on peut ajouter plusieurs clés en parallèle. Nous allons créer une autre clé qui sera plutôt stocké dans un fichier sur un autre volume. Au démarrage, nous allons utiliser ce fichier pour déverrouiller notre volume.


```bash
# Créer uns chaîne de caractères aléatoire, et les sauvegarder dans /etc/.luks-encrypteddata-key
dd if=/dev/random of=/etc/.luks-encrypteddata-key bs=32 count=1

# Regarder ce fichier :
cat /etc/.luks-encrypteddata-key
```

Nous ajoutons cette clé parmi les clés possibles pour LUKS :

```bash
# Ajouter la clé /etc/.luks-encrypteddata-key parmi le trousseau pour /dev/sda
cryptsetup luksAddKey /dev/sda /etc/.luks-encrypteddata-key

# Attention: il va vous demander le mot de passe précédemment crée.
# C'est normale ! Il faut connaître déjà un mot de passe avant d'en ajouter un autre !
```

A tout moment, nous pouvons visionner l'état de nos clés :

```bash
cryptsetup luksDump /dev/sda
```

Testons cette nouvelle clé. D'abord, fermons le volume qui a été ouvert via la clé initiale :

```bash
# Démonter le volume
umount /mnt/encrypteddata 

# Fermer le périphérique
cryptsetup luksClose encrypteddata
```

Ensuite, montons le volume en précisant plutôt le fichier comme clé :

```bash
cryptsetup -v luksOpen /dev/sda encrypteddata --key-file /etc/.luks-encrypteddata-key

# Key slot 1 unlocked.
# Command successful.
```

C'est super. Maintenant, on peut déverrouiller un volume sans fournir manuellement un mot de passe. Comment le faire au démarrage ?

Vous vous souvenez du fichier `/etc/fstab` ? Nous avons l'équivalent pour les volumes cryptés : `/etc/crypttab`, qui prend le format suivant :


```
<target name> <source device> <key-file> <options>
```

Les options sont :
* `target name` : le nom de notre volume, pour nous `encrypteddata`
* `source device` : le UUID de notre volume. On peut trouver le UUID avec `blkid /dev/sda`. Dans mon exemple, j'ai trouvé le UUID `9096ac61-20bf-4c88-a770-35e6b71897b7`.
* `key-file` : le chemin absolu du fichier avec la clé, pour nous `/etc/.luks-encrypteddata-key`
* `options` : les options du système de cryptographie. Pout nous `luks`

Nous ajoutons alors la ligne suivante :

```bash
# Ajouter cette ligne à /etc/crypttab, ex: 
# nano /etc/crypttab
encrypteddata  UUID="9096ac61-20bf-4c88-a770-35e6b71897b7"  /etc/.luks-encrypteddata-key  luks
```

Cette procédure assurera que le périphérique virtuel est créé (le volume crypté est ouvert). En revanche, il faut quand même monter le volume au démarrage, comme nous avons fait avec un volume normal. On modifie donc `/etc/fstab` :

```bash
# Ajouter cette ligne à /etc/fstab, ex: 
# nano /etc/fstab
/dev/mapper/encrypteddata  /mnt/encrypteddata ext4 defaults,nofail 0 0
```
Vérifiez bien que vous votre volume monte correctement :

```bash
umount /mnt/encrypteddata
mount /mnt/encrypteddata/
```

Ensuite, redémarrez votre instance pour tester.


{% hint style="success" %}
 
**La rotation des clés**

Vous avez sûrement remarqué qu'on peut ajouter plusieurs clés pour décrypter notre volume. Comment cela fonctionne ?

Il y a deux niveaux de clé :

1. Une clé pour nous, l'utilisateur
2. Une clé "interne" connu et utilisé uniquement par LUKS

Les données sur le disque sont cryptées avec la deuxième clé. 

Notre clé à nous sert uniquement à décrypter la clé interne. 

Cela veut dire qu'on peut crypter plusieurs fois la clé interne avec différentes clés "externe", sans crypter à nouveau toutes les données sur le volume !

L'autre avantage est qu'on peut implémenter de **la rotation des clés**. Tous les X jours/semaines/mois, nous ajoutons une nouvelle clé, et on supprime l'ancienne. L'idée est d'encore minimiser les dégâts si une des clés est divulguée, car on rend obsolète les clés précédentes.



{% endhint %}

{% hint style="warning" %}

**Le stockage des clés**

Pour le moment, nous avons 2 clés :
- La clé initiale qu'on a mémorisé ou stocké sur notre machine locale
- La clé dans `/etc/.luks-encrypteddata-key` qui est stocké sur un autre volume

Vous remarquez qu'aucune clé n'est stockée sur le même volume que les données. Donc si quelqu'un pénètre la data-center et vol le disque dur avec nos données, il ne pourra pas décrypter nos données.

Vous pouvez également protéger le fichier de clé en utilisant les permissions de base de Linux (uniquement lisible par `root`) en utilisant `chmod`.

{% endhint %}


Références:
* [https://www.percona.com/blog/mysql-encryption-at-rest-part-1-luks/](https://www.percona.com/blog/mysql-encryption-at-rest-part-1-luks/)
* [https://kifarunix.com/automount-luks-encrypted-device-in-linux/](https://kifarunix.com/automount-luks-encrypted-device-in-linux/)

