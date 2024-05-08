# Journaux

Les fichiers journaux sont un élément essentiel de la surveillance des attaques. Mais où trouver ces fichiers ?

En général, sur une machine Linux, ils se trouvent sous :

```
/var/log
```

Par exemple, les tentatives d'accès via SSH sont listées ici :

```
/var/log/auth.log
```

Si vous avez installé le serveur web Apache, vous trouverez les journaux ici :

```
/var/log/apache2
```

Il existe un certain nombre de solutions commerciales permettant d'ingérer ces journaux et d'utiliser l'intelligence artificielle ou des règles pour détecter les comportements non autorisés, les attaques par déni de service, etc.