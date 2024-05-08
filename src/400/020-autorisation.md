# Identité et autorisation

Normalement, dans un API fermé, nous allons empêcher l'accès aux utilisateurs qui n'ont pas le droit d'y accéder.

Comme pour la sécurisation d'une installation Linux, il y a plusieurs facettes à la sécurisation d'un API :

* identité
* ressource
* droits

## Identité

Avant de se lancer dans la sécurisation d'un API il faut identifier d'abord **QUI** aurait accès à votre API.

Surtout, comment notre API va être capable d'identifier un utilisateur précis, et être 100% certain que c'est bien elle qui fait des requêtes.

Aujourd'hui, il existe plusieurs façons (flux ou **identity flows**) pour établir et vérifier l'identité d'un utilisateur.

Le principe distillé est le suivant :

1. à la première connexion, on demande à l'utilisateur de s'identifier par une valeur unique. Normalement, c'est une adresse e-mail ou nom d'utilisateur qu'il aurait choisi (ou qui lui a été attribué) à la création de son compte
2. Ensuite, l'utilisateur est demandé de **prouver** son identité via une valeur ou secret **seulement à la disposition de cet utilisateur**
3. L'API crée un jeton unique valable uniquement pendant la séance, et on passe ce jeton à l'utilisateur (via un **cookie**, par exemple)
4. Le client de l'utilisateur doit désormais fournir ce jeton avec chaque requête
5. À la réception d'une requête, l'API doit valider le jeton, et rejeter la requête si le jeton n'est pas présent ou n'est pas valable, ou bien si l'utilisateur n'a pas le droit d'accéder à la ressource demandée

### Prouver son identité

Comment l'utilisateur peut prouver son identité ?

* Un mot de passe connu uniquement par l'utilisateur
* Une valeur biométrique physique de l'utilisateur (emprunt digital, iris, identification visage, etc.)

La solution d'un mot de passe est compliquée pour plusieurs raisons :

* Un mot de passe trop simple est facile à deviner grâce aux algorithmes simples (attaque en force brute)
* Un utilisateur a tendance à oublier son mot de passe complexe. Il réutilise donc le même mot de passe partout
* Nous sommes obligés de stocker le mot de passe dans notre base de données.
  * Si on est hacké, on divulgue le mot de passe de nos utilisateurs au hackeur.
  * À ce moment-là, le hackeur aurait potentiellement accès à tous les comptes utilisant le mot de passe partagé

La solution des biométriques est compliquée aussi :

* On a besoin d'un périphérique spécial (lecteur d'emprunt digital)
* Sinon, certaines méthodes ne sont pas totalement fiables (détection faciale) et pourraient engendrer des faux positifs ou des faux négatifs

### Identité par délégation

Une autre stratégie pour prouver l'identité est de dépendre sur un tiers.

Par exemple, si on utilise l'adresse e-mail de la personne, on peut supposer que seulement cette personne aura accès à sa boîte mail.

Pour prouver son identité, on pourrait juste forcer cet utilisateur à prouver qu'il a accès aussi à sa boîte e-mail - en l'envoyant un message à cette adresse.

Normalement, on envoie un émail avec un code ou lien unique. Si l'utilisateur peut nous répéter ce code unique, il prouve qu'il a pu consulter son mail.

Pour nous, cela veut dire qu'on ne stocke plus son mot de passe dans notre base de données ! On dépend du mot de passe utilisé pour protéger son compte email :

* C'est 1 mot de passe en moins à mémoriser pour l'utilisateur
* En revanche, si le mot de passe de sa boîte email est volée, le voleur aura accès à notre service aussi

{% hint style="info" %}
Ce flux pourrait être adapté autrement : par envoi d'un SMS par exemple.
{% endhint %}

Un autre type d'identité par délégation existe aujourd'hui grâce à la norme [OAuth2](https://oauth.net/2/). L'idée de base est qu'un service centralisé s'occupe de l'identification d'un utilisateur, et crée des jetons d'accès utilisables par notre API. On en a tous utilisé :

* connexion avec votre identifiant Google
* connexion avec votre identifiant Facebook
* connexion avec votre identifiant GitHub

En revanche, cela force nos utilisateurs d'avoir un compte chez un tiers avant d'utiliser notre service (parfois pas idéal). Et, aussi, s'il y a une fuite chez un de ces services, notre API sera à risque aussi.

### La double authentification

Pour encore sécuriser nos services aujourd'hui, la tendance est d'aller vers une combinaison des différentes approches :

* Je fournis mon adresse e-mail et mot de passe
* Si le mot de passe est validé, l'API envoie un message avec un code unique à l'adresse email
* l'utilisateur doit répéter le code unique trouvé dans la boîte mail
* le jeton est créé
* ... etc

Même si le premier mot de passe est divulgué, on a une autre couche de protection via le mot de passe de la boîte mail.

## Autorisation des Ressources

Une fois l'identité connue, il faut maintenant préciser ce qu'on va protéger dans notre API. Certains endpoints qui divulgue des informations sensibles doivent être protégées.

Avec Express (NodeJS), ce processus est simple, il suffit d'ajouter un **middleware** devant une route, et devant une branche dans notre arborescence. Ce **middleware** va décoder le jeton d'accès pour vérifier l'identité de la personne et les droits qui lui sont accordés.

Un exemple middleware qui cherche le jeton d'accès sur l'en-tête `authorisation` :

```ts
export const AuthMiddleware = async (request: Request, response: Response, next: NextFunction) => {
  try {
    const authheader = request.headers.authorization || '';
    if (!authheader.startsWith('Bearer ')) {
      throw new ApiError(ErrorCode.Unauthorized, 'auth/missing-header', 'Missing authorization header with Bearer token');
    }

    const token = authheader.split('Bearer ')[1];

    /// décoder le jeton
    /// valider les informations dans le jeton

    if (!valid) {
      throw new Error(401);
    }
    
    next();

  } catch (err) {
    next(err)
  }
});

```

Nous utilisons ce middleware en le plaçant à la tête des branches à protéger dans notre API :

```ts
app.use('/user', 
  AuthMiddleware,   // Insérer un middleware pour valider que l'utilisateur est bien identifié
  ROUTES_USER
);
```

## Droits

Dans beaucoup d'APIs, il n'est pas suffisant de savoir que la personne est bien identifiée. Cet utilisateur aura peut-être des droits différents selon son **rôle**. Par exemple, un utilisateur normal vs un administrateur.

Une stratégie serait d'encoder dans le jeton le rôle de l'utilisateur, ou bien un indice des ressources à sa disposition.

On parle souvent de son **scope**, c'est l'ensemble de ressources à la disposition de l'utilisateur, qu'on aura extrait de la base de données lors de la création de son jeton d'accès. Un scope pourrait être simplement un string :

* `admin` : un scope global qui donne accès à tout
* `user` : l'utilisateur aura accès à son profil utilisateur
* `bidule` : à nous de définir ce que cela veut dire

On peut adapter notre **middleware** afin de prendre en compte des **scopes** :

```ts

export const AuthMiddleware = (...requiredScopes: string[]) => {
  return async (request: Request, response: Response, next: NextFunction) => {
    try {
      const authheader = request.headers.authorization || '';
      if (!authheader.startsWith('Bearer ')) {
        throw new ApiError(ErrorCode.Unauthorized, 'auth/missing-header', 'Missing authorization header with Bearer token');
      }

      const token = authheader.split('Bearer ')[1];

      // décoder le jeton
      // valider les informations dans le jeton

      if (!valid) {
        throw new Error(401);
      }

      // Procéder à l'autorisation : est-ce que le scope fourni dans le jeton correspond à ceux nécéssaire pour la route
      const providedScopes = token.scopes;

      let authorised = false;
      for (const required of requiredScopes) {
        for (const provided of providedScopes) {
          if (provided === required) {
            authorised = true;
            break;
          }
        }
      }

      if (!authorised) {
        throw new Error(403);
      }      

    } catch (err) {
      next(err)
    }
  }
});
```

Ensuite, pour chaque branche de notre API on peut facilement préciser les **scope** nécessaires pour accéder à la branche :

```ts
app.use('/user', 
  AuthMiddleware('admin'),   // Insérer un middleware pour valider que l'utilisateur est bien identifié
  ROUTES_USER
);
```

## Le jeton d'identité

Le jeton d'identité correspond à quoi exactement ?

* Un numéro unique (un UUID par exemple)
  * On crée et stocke un identifiant lié à la session de l'utilisateur. À chaque requête, l'API est obligé de chercher dans sa base locale l'identité de l'utilisateur et les droits associés. Ceci n'est pas très compatible avec une API totalement **stateless**, parce qu'on est obligé de stocker le UUID quelque part entre les requêtes
* Un jeton qui stocke toutes les informations d'identité et d'accès : le JWT

Un JWT (**JSON Web Token**), et un format qui permet de transmettre non-seulement des informations d'identité, mais aussi les **scopes** en un seul paquet. En plus, le JWT est **signée**, c'est-à-dire, on a le moyen de valider l'identité de la personne qui la crée initialement (nous-mêmes !).

Le JWT contient aussi une expiration — elle s'expire toute seule au-delà d'une certain période.

Le flux d'un JWT est le suivant :

1. L'identité est validée
2. On crée une structure (le **payload**) contenant le ID et les droits de l'utilisateur
3. On crée le JWT contenant notre structure d'information, et d'autres informations obligatoires pour un JWT (expiration, issuer, audience, etc)
4. On signe la JWT avec notre clé privée : on attache au JWT une partie contenant cette valeur chiffrée
5. On envoie la JWT à l'utilisateur (le JWT contient la partie en texte clair et la partie chiffrée)
6. Le client de l'utilisateur attache le JWT à l'en-tête d'une requête
7. L'API récupère le JWT de l'entête, et essaye de le décrypter en utilisant la clé publique correspondant à la clé privée. Si la valeur décryptée corresponde parfaitement à la partie en texte claire, on est sûr que c'est nous qui l'aurons créé
8. On valide que le JWT n'est pas encore expiré
9. On récupère les scopes et l'ID de l'utilisateur et on valide l'accès à la ressource

Pour un JWT sécurisé, il faut donc une paire de clés qui permet de chiffrer et déchiffrer le JWT. Il y a une librairie NodeJS qui nous aide à gérer les JWT qui s'appelle `jsonwebtoken`.


```bash
npm install jsonwebtoken
npm install @types/jsonwebtoken
```

Le code suivant est un exemple d'une classe utilitaire qui permet de créer et décoder un JWT :

```ts
import { readFileSync } from 'fs';
import jwt from 'jsonwebtoken';
import { join } from 'path';

export class JWT {

  private static PRIVATE_KEY: string;
  private static PUBLIC_KEY: string;

  constructor() {
    if (!JWT.PRIVATE_KEY) {
      JWT.PRIVATE_KEY = readFileSync(process.env.PRIVATE_KEY_FILE || join('secrets', 'signing', 'signing.key'), 'ascii')
    }

    if (!JWT.PUBLIC_KEY) {
      JWT.PUBLIC_KEY = readFileSync(process.env.PUBLIC_KEY_FILE || join('secrets', 'signing', 'signing.pub'), 'ascii')
    }
  }

  async create(payload: any, options: jwt.SignOptions) {
    return new Promise(
      (resolve, reject) => {
        jwt.sign(payload, JWT.PRIVATE_KEY, Object.assign(options, { algorithm: 'RS256' }), (err: any, encoded) => {
          if (err) {
            reject(err);
            return;
          }
          resolve(encoded);
        });
      }
    )
  }

  async decode(token: string, options: jwt.VerifyOptions) {
    return new Promise<any>(
      (resolve, reject) => {
        jwt.verify(token, JWT.PUBLIC_KEY, Object.assign(options, {
          algorithms: ['RS256']
        }), (err: any, decoded) => {
          if (err) {
            reject(err);
            return;
          }
          resolve(decoded);
        })
      }
    )
  }
}
```

# Le jeton de renouvellement

Les jetons d'accès ont généralement une durée d'expiration courte et ne donnent accès qu'à une ressource spécifique.

Un jeton d'accès doit fournir l'autorisation de créer un nouveau jeton d'accès !

Généralement, lorsque nous nous connectons à un service, deux jetons sont créés après la connexion :

- un jeton d'accès pour accéder à la ressource demandée
- un jeton de renouvellement qui nous permet de générer de nouveaux jetons d'accès.

Le jeton de renouvellement ne nous permet pas d'accéder directement aux ressources, mais de créer de nouveaux jetons d'accès ou de renouvellement. Généralement, le processus de validation d'un jeton de renouvellement nécessite une consultation de la base de données pour vérifier que le compte est toujours actif et que l'accès n'a pas été révoqué.

Un jeton de renouvellement a généralement une date d'expiration plus longue.

La procédure est la suivante :

- Une demande avec un jeton d'accès expiré est rejetée par une API
- Le client envoie une demande au service d'autorisation pour obtenir un nouveau jeton d'accès. Cette demande utilise le **jeton de renouvellement** dans son en-tête.
- Le service d'autorisation vérifie le jeton de renouvellement :
  - Si le jeton de renouvellement est expiré, une erreur est renvoyée et l'utilisateur est renvoyé à la page de connexion.
  - Sinon, l'identifiant de l'utilisateur est vérifié dans la base de données (compte toujours actif).
  - Le service d'autorisation réémet le jeton d'accès.
- Le client réessaie le point d'accès original avec le nouveau jeton d'accès.

## Communiquer et stocker des jetons

Comment transmettons-nous les jetons à nos utilisateurs et où les stockent-ils ? 
Ces questions sont souvent à l'origine des plus grandes faiblesses des plateformes en ligne.

**Comme un cookie**

Nous pouvons créer un cookie qui contient notre jeton. Mais nous nous exposons à toutes les faiblesses associées aux cookies (CSRF, XSS, ....). 

Si l'utilisateur a désactivé les cookies dans son navigateur, notre site devient inutilisable.

En fonction de l'âge du navigateur, le cookie peut également être lu par d'autres domaines, et donc par d'autres sites web.

**En `localStorage`**

Votre API peut également stocker les jetons dans le localStorage du navigateur. Cela dépend de la sécurité du navigateur, mais en général, seuls les sites web du même domaine peuvent accéder au localstorage. 


Dans tous les cas, des secrets peuvent être divulgués au client. Il est important de réémettre fréquemment des jetons afin que même les jetons volés expirent ou soient invalidés rapidement.

Il est également important de pouvoir dévalider les jetons si l'on détecte un vol.
