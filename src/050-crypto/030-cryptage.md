# Cryptage d'un fichier


Connectez-vous au serveur `unixshell.hetic.glassworks.tech`, qui a déjà le logiciel correct installé pour cet exercice. Nous allons utiliser GPG (GNU Privacy Guard).


## Cryptage symétrique

Tout d'abord, créez un fichier à crypter :

```bash
echo "Hello world !" > greetings.txt
```

Cryptage de votre fichier :

```bash
gpg --output greetings.txt.gpg --symmetric greetings.txt
```

Votre fichier a été crypté à l'aide de l'algorithme par défaut AES256. Vous pouvez utiliser l'option `--cipher-algo` pour choisir un autre algorithme. La liste des algorithmes supportés [se trouve ici](https://www.gnupg.org/documentation/manuals/gcrypt/Available-ciphers.html).


Afficher le contenu du fichier crypté :

```bash
cat greetings.txt.gpg
```

Décryptez maintenant votre fichier :

```bash
gpg --output greetings1.txt --decrypt greetings.txt.gpg
```

Vérifier le contenu du fichier décrypté :

```bash
cat greetings1.txt 
```

Pourquoi GPG ne demande-t-il pas un mot de passe lors du décryptage ? Il stocke le mot de passe dans une variable d'environnement qui sera supprimée lorsque vous quitterez le shell.

## Cryptage asymétrique  

Je vous laisse le soin de vous entraîner à crypter/décrypter des fichiers à l'aide d'une paire de clés.

- [Avec GPG](https://www.baeldung.com/linux/encrypt-decrypt-files)
- [Avec OpenSSL](https://opensource.com/article/21/4/encryption-decryption-openssl)