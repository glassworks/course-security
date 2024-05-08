# Actions

Une fois l'identité établie, on aimerait savoir et contrôler l'action à effectuer. Sur UNIX il y a 3 actions :

* Lire
* Ecrire
* Exécuter

L'action _lire_ regroupe non-seulement l'ouverture d'un fichier pour le visionner (avec `nano` par exemple), mais aussi la possibilité de _voir_ le fichier avec `ls`. Si un utilisateur n'a pas les droits de lecture sur un fichier, ce fichier sera invisible, malgré les options qu'on passe à `ls`.

L'action _ecrire_ est simple : je ne peux pas modifier ou supprimer un fichier si je n'ai pas ce droit. Je ne peux pas non plus créer des nouveaux fichiers dans un répertoire dont je n'ai pas ce droit.

L'action _exécuter_ regroupe deux actes :

* naviguer dans un répertoire (avec `cd` par exemple)
* lancer une commande ou un fichier executable.