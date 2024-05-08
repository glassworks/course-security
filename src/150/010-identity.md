# Identité

Le système de fichiers Unix est la première étape de la sécurisation d'un serveur physique et de son système d'exploitation contre les attaques malveillantes.

A la base d'un système de sécurité est la notion d'identité. Il faut d'abord savoir qui vous êtes avant de valider si vous avez accès ou pas.

## Utilisateur

On ne peut pas accéder à une invite de commandes Unix sans d'abord s'identifier. Le plupart du temps on se connect avec un nom d'utilisateur et mot de passe (ou clé SSH).

L'identité unique est donc centrale !

Comment on peut lister les utilisateurs inscrits sur la machine ? C'est stocké simplement dans le fichier `/etc/passwd`;

```bash
cat /etc/passwd
```

Ce fichier de configuration contient les informations des comptes utilisateurs :

* Le nom de l’utilisateur
* Le hash du mot de passe
* Divers identifiants (user ID, groupe ID)
* L’emplacement de son dossier maison
* Le shell à utiliser quand il/elle se connecte

Comment ajouter un utilisateur ? Selon votre système Unix, un des deux commandes suivantes pourront fonctionner pour vous :

```bash
# Ajouter l'utilisateur 'kevin'
adduser kevin

# Ajouter l'utilisateur 'kevin' et lui créer un dossier maison, en l'ajoutant au groupe `students`
useradd -m -g students kevin
```

A noter, normalement il faut être administrateur pour ajouter un utilisateur, c'est à dire, dans le groupe `sudo` pour avoir le droit. Dans ce cas, il faudrait ajouter `sudo` devant chaque commande :

```bash
# Ajouter l'utilisateur 'kevin'
sudo adduser kevin

# Ajouter l'utilisateur 'kevin' et lui créer un dossier maison, en l'ajoutant au groupe `students`
sudo useradd -m -g students kevin
```

## Groupes

En tant qu'utilisateur, je pourrais aussi participer dans un ou plusieurs _groupes_.

Un groupe est juste une collection d'utilisateurs.

Par exemple. Imaginons qu'on a deux utilisateurs, `hetic` et `kevin`. Chaque utilisateur peut appartenir à zéro, un ou plusieurs groupes :

* Groupe : `sudo`
  * Utilisateur : `hetic`
  * Utilisateur : `kevin`
* Groupe : `docker`
  * Utilisateur : `kevin`

Ici, les deux utilisateurs se trouvent dans le groupe `sudo`, mais seulement `kevin` se trouve dans le groupe `docker`. Dans cette exemple, cela veut dire que `kevin` pourrait éventuellement lire et modifier les fichiers du groupe `docker`. L'utilisateur `hetic` ne pourra pas.

Comment voir tous les groupes de l'installation ?

```bash
# Lister tous les groupes sur l'installation UNIX
cat /etc/group

# Voir les groupes de l'utilisateur actuel
groups [nom d'uilisateur]

# Ajouter un nouveau groupe
groupadd [nom du groupe]

# Affecter un utilisateur à un groupe 
usermod -aG [group] [utilisateur]

# Lister les utilisateurs dans un groupe 
# (Peut changer en fonction de votre distribution)
getent group [group]

# Enlever un utilisateur d’un groupe (dépendant de la distro):
gpasswd -d [utilisateur] [group]
deluser [utilisateur] [group]

# Supprimer un group : 
groupdel [group]
# Supprimer un utilisateur : 
userdel -r [utilisateur]
# Le –r supprime aussi son répertoire home
```

Essayez de créer des utilisateurs et groupes !

## Droits

Une fois que je suis identifié, j'aurais un certain nombre de droits selon 3 catégories :

* Les droits accordés à moi-même uniquement
* Les droits accordés aux _groupes_ dont je fait parti
* Les droits accordés à _tous les utilisateurs_ de l'installation UNIX

Ces droits sont exprimés sur chaque fichier / répertoire.
