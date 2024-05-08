# Privilèges dans un SGBDR

Jusqu'au présent nous nous sommes connectés à la base de données en tant que `root`, qui a tous les droits d'administration.

{% hint style="danger" %}
Il ne faut *jamais* communiquer les coordonnées de l'utilisateur `root` ! 

Et en plus, normalement, on désactive les connexions en tant que `root` provenant des adresses IP en dehors de notre entreprise !
{% endhint %}

Il faut donc créer d'autres utilisateurs, et finement gérer leurs accès aux *databases* au sein du SGDBR.


## Utilisateurs

Les utilisateurs dans un SGBDR type MySQL ou MariaDB s'identifie par un *nom* et une *adresse*. L'idée est qu'on sécurise l'accès de quelqu'un en fonction de son endroit de connexion.

C'est très pratique ! On peut limiter l'accès à un utilisateur uniquement d'au sein d'un réseau privé. Par exemple :

- l'utilisateur `root` ne peut se connecter seulement via la machine locale et pas d'une machine distante. Cela veut dire qu'on doit d'abord se connecter en SSH au terminal de la machine, avant de se connecter à MySQL.
- un utilisateur pour une API qui aura des privilèges de lecture et écriture sur une certaine base est limité au réseau privé de l'entreprise.


### Créer un utilisateur 

On utilise la commande suivante :

```sql
create user 'kevin'@'%.%.%.%' identified by 'password';
```

On crée un utilisateur qui s'appelle `kevin`, qui peut se connecter de *tous les réseaux* (`%.%.%.%`) qui aura besoin d'un mot de passe `password`.

Pour créer un utilisateur limité à un réseau interne, par exemple :

```sql
create user 'kevin'@'10.1.%.%' identified by 'password';
```

Notez qu'on peut créer plusieurs utilisateurs du même nom, avec une provenance différente.

### Lister les utilisateurs

La liste d'utilisateurs actifs se trouve dans une *database* interne qui s'appelle `mysql`, sur la table `user`. Il suffit de les lister :

```sql
SELECT user, host FROM mysql.user;
```

Nous verrons le nom d'utilisateur, ainsi que l'hôte autorisé pour la connexion. Par exemple :

```
MariaDB [saas]> SELECT user, host FROM mysql.user;
+-------------+-----------+
| User        | Host      |
+-------------+-----------+
| root        | %         |
| kevin       | %.%.%.%   |
| mariadb.sys | localhost |
| root        | localhost |
+-------------+-----------+
```

### Supprimer un utilisateur

Pour supprimer un utilisateur, on utilise commande `drop` :

```sql
drop user `kevin`@`%.%.%.%`;
```

### Mettre à jour un mot de passe

On peut mettre à jour un mot de passe d'un utilisateur en le modifiant :

```sql
alter user 'kevin'@'%.%.%.%' identified by 'newpassword';
```

## Privilèges

Une fois les utilisateurs créés, il faut les accorder accès à un ou plusieurs *databases* dans notre SGBDR.

{% hint style="warning" %}
On accorde le minimum de privilèges nécessaires pour faire son travail, et jamais plus.

C'est ainsi qu'on protège au maximum notre base de données des tentatives de hack!

Pensez toujours au pire, puis voir comment on peut limiter les dégâts !
{% endhint %}

Nous allons accorder accès sur 2 dimensions principales :

- Les actions autorisées…
  - ... sur une *database* particulière
  - ... sur une table particulière

Par exemple :

```sql
grant all privileges on sakila.* to 'kevin'@'%.%.%.%';
```

Ici, on accorde à `kevin` la possibilité de *tout faire* sur la base de données `sakila`. Ce n'est pas très sécurisée, car l'utilisateur peut non-seulement tout lire, mais aussi modifier la base (par exemple, supprimer les tables) !


Quels privilèges alors ? Cela dépend de l'utilisateur :

- un *administrateur* peut créer et supprimer des bases
- une appli type API ne doit pas pouvoir modifier le schéma d’une base de données, mais uniquement *insérer*, *lire* et *modifier*, et peut-être *supprimer* des lignes dans des tables
- une appli tierce (d'un partenaire), devrait *lire* exclusivement les contenus, mais aucun droit d'écriture n'est accordée.

{% hint style="warning" %}

Imaginons qu’un app API est hacké, et le hacker arrive à imiter l’API. Le hacker aura les mêmes droits que l’API.

Si on a accordé `ALL PRIVILEGES` à l’utilisateur de l'API, le hacker peut tout voir, tout supprimer ou bien tout crypter pour demander une rançon.

{% endhint %}

Pour un API par exemple :

```sql
create user 'api'@'10.1.%.%' identified by 'very-strong-password';
grant SELECT, UPDATE, INSERT, DELETE on sakila.* to 'api'@'10.1.%.%';
```

On peut consulter les privilèges accordés à un utilisateur :

```sql
show grants for 'api'@'10.1.%.%';
```

On peut enlever les privilèges avec `revoke` :

```sql
revoke all privileges on 'sakila'.* from 'kevin'@'%.%.%.%';
```

Enfin, quand on a tout modifié à notre satisfaction, on sauvegarde nos modifications pour qu'elles soient prises en compte par le SGBDR :

```sql
flush privileges;
```

La liste de privilèges possibles dans un SGBDR type MariaDB peut être consulté au lien suivant : [https://mariadb.com/kb/en/grant/](https://mariadb.com/kb/en/grant/)

## Exercice

Créez un nouvel utilisateur sur votre base, sans accorder de privilèges. Essayez d’accéder à la base avec votre client préféré.

Accordez des privilèges pour un utilisateur du type « API » (créer, modifier, visionner, supprimer). Testez votre utilisateur avec votre client.

Essayer de modifier le schéma d’une table.

Créez un utilisateur en lecture seule. Testez votre utilisateur via votre client.

