# Sécuriser MariaDB


MySQL (et MariaDB) contient par défaut un script qui permet d’affecter des règles de sécurité de base

* Désactiver les connexions anonymes
* Assurer des utilisateurs admin/root ne peuvent connexion uniquement par la machine locale
* Assurer que l’utilisateur `root` a un mot de passe (et qu’il est suffisamment sécurisé)
* Supprimer la base de données de test (parfois installé par défaut)

```sh
docker exec -it [Container ID] mysql_secure_installation
```