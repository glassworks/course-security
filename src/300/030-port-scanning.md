# Port scanning

Lorsque nous avons approvisionné notre base de données, nous avons délibérément ouvert le port 7100 pour pouvoir l'administrer à distance.

L'ouverture d'un port expose notre serveur à des vecteurs d'attaque. Des bogues ou des faiblesses de type "zero day" peuvent être exploités par des pirates informatiques s'ils détectent qu'un service X est à l'écoute sur un port.

Des scanners portuaires existent en ligne et peuvent être téléchargés gratuitement, par exemple :

[PenTest port scanner](https://pentest-tools.com/network-vulnerability-scanning/port-scanner-online-nmap?view_report=true)

Lancez une analyse de votre serveur. Y a-t-il des ports ouverts inattendus ?

## Fermeture des ports : iptables et UFW

Il arrive qu'un port doive être ouvert pour un certain service, mais que nous souhaitions régler avec précision qui y a accès, et d'où (comme nous l'avons fait avec le port exposé pour MariaDB).

Les pare-feu commerciaux sont chargés de cette tâche et permettent de créer des **règles** pour le trafic entrant et sortant.

Chaque machine Linux dispose aussi de son propre tableau associatif d'adresses IP et ports, qui permet de définir comment accepter et router des paquets.

```bash
iptables --list
```

Nous pourrions par la suite configurer ce tableau pour rediriger les paquets comme on veut selon plusieurs critères :

- la carte réseau impliquée (*loopback*, carte ethernet, cart wifi, etc)
- le port
- le protocole
- l'adresse IP source
- l'adresse IP destination
- ...

[iptables cheatsheet](https://andreafortuna.org/2019/05/08/iptables-a-simple-cheatsheet/)

Cet outil est puissant, mais difficile à configurer.

Une application *enveloppe* existe qui s'appelle UFW (**U**ncomplicated **F**ire**W**all) qui permet de créer un pare-feu simple, basé sur `iptables`. Via des règles/expressions simples, nous allons préciser quels paquets accepter ou rejeter. UFW ajoute ensuite des lignes au `iptables`.


```bash
ufw status
```

Pour tester `ufw`, nous allons d'abord installer un serveur nginx sur une instance Ubuntu 22.04.

```bash
apt install nginx
```

Normalement, on pourrait faire une requête à notre serveur dans un navigateur pour avoir la page défaut de nginx.

Avant d'activer UFW, pour ne pas perdre la connexion à notre serveur, on va activer les connexions SSH :

D'abord, on va préciser les règles de base :

```bash
# On refuse TOUT paquets entrants
ufw default deny incoming
# On accepte l'envoie de tout les paquets sortants
ufw default allow outgoing
```

Ensuite, **important**, activer les règles pour SSH, sinon on perdra toute connexion avec notre serveur !

```bash
# Vérifier quels profils application existent
ufw app list
# Activer l'application OpenSSH
ufw allow OpenSSH
```

Maintenant activez UFW :

```bash
ufw enable
```

Essayez de nouveau de charger la page web servie par nginx. On ne peut pas !

Analysons les règles de UFW :

```bash
ufw status numbered
```

Regardons maintenant IPTables :

```bash
iptables --list
```

Vous remarquerez plus de règles !

Créons des règles qui permettent l'accès du monde extérieur pour nginx :

```bash
# On utilise le profil nginx
ufw allow 'Nginx Full'
```

Notre page fonctionne de nouveau !

Il y a d'autres scénarios de sécurisation d'un serveur :

* Accepter des connexions SSH d'une adresse IP fixe : `allow from 203.0.113.88 to any port 22`
* Accepter des connexions SSH d'une adresse IP fixe à MySQL : `allow from 203.0.113.88 to any port 3306`
* Désactiver des connexions HTTP (port 80) : `ufw deny http`


... et bien plus. Par exemple, on peut aussi désactiver des paquets sortants, pour empêcher du malware à envoyer des secrets vers l'extérieur. En pratique, c'est difficile, surtout si notre serveur doit se mettre à jour tout seul !

Pour enlever une règle :

```bash
# Afficher les règles dans l'ordre et numéroté
ufw status numbered
# Supprimer un règle
ufw delete 4
```


Pour désactiver UFW (et donc enlever toutes les règles de `iptables`) :

```bash
ufw disable
```

[Plus sur UFW](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-ubuntu-18-04)

{% hint style="warning" %}

UFW ne remplace pas des pare-feux plus avancés qui vont réellement regarder les contenus des paquets, utiliser les algorithmes type IA afin de gérer les flux, etc.

En revanche, c'est une première couche de sécurisation !

{% endhint %}