Synchronisation multi-client avec CouchDB et PouchDB
====================================================

Offline First
-------------

Aujourd'hui, alors que les usages nomades et mobiles explosent, il est devenu obligatoire de penser "Mobile First" mais
aussi "Offline First" lorsque l'on souhaite développer une nouvelle application.

Il est intéressant de mettre en parallèle les 2 approches :

- Mobile First : « Design for the smallest device first and then apply progressive enhancement techniques to take 
advantage of larger screen sizes »

- Offline First : « Design for offline usage first and then apply progressive enhancement techniques to take advantage of 
network connectivity when available »

Si cela peut sembler très simple "sur le papier", il en est autrement lorsque l'on souhaite implémenter et intégrer la 
gestion du mode Offline et la synchronisation de données à son application.

En effet, la synchronisation/réplication de données a toujours été un problème compliqué en informatique (intégrité, conflit, etc...) 
et l'utilisation conjointes des 2 technologies que je vais vous présenter dans cette article va nous aider à en réduire
considérablement la complexité.

CouchDB
-------
[CouchDB](http://couchdb.apache.org/) est une base de données NoSQL orientée document et open source, elle est développée et maintenue par
la fondation apache. Comme pour MongoDB, les documents sont stockés au format JSON, la similarité avec Mongo s'arrêtant là.
En effet, à la différence des autres systèmes de base de données NoSQL, CouchDB se démarque sur 2 points essentiels (et non des
moindres) :

- CouchDB expose ces API via REST, c'est à dire que tout est accessible via http. 
C'est pourquoi nous parlons de base de données qui embrasse le web ("database that embraces the web"). 
Pas besoin d'utiliser un pilote ou une librairie dans un langage cible.

- CouchDB c'est aussi un protocole de réplication bidirectionnelle master/master via http.
Tout système de base de données implémentant ce protocole est donc capable de se répliquer
vers et depuis un autre système implémentant également ce protocole.

> Pour les plus curieux, la spécification du protocole est disponible depuis ce lien : [http://docs.couchdb.org/en/1.6.1/replication/protocol.html](http://docs.couchdb.org/en/1.6.1/replication/protocol.html)

Le schéma ci-dessous montre les différentes possibilités :

![Protocole de réplication CouchDB](./assets/CouchDB-replication-protocol.png)

Il est donc possible de synchroniser et répliquer 2 bases de données entre une application
mobile utilisant CouchBase Lite et un serveur Cloudant, les 2 systèmes implémentant 
le protocole CouchDB.

> A noter que Cloudant (IBM) et CouchBase sont 2 forks du code source de CouchDB.

Regardons à présent de plus près la technologie qui va nous permettre de rendre 
"Offline First" une application web : PouchDB. 

PouchDB
-------

[PouchDB](https://pouchdb.com/) est l'implémentation web (JavaScript) du protocole de réplication CouchDB.
(En exagérant, on pourrait même dire que Pouchdb est l'implémentation en JavaScript de CouchDB).

Ses principales caractéristiques sont :

- Cross Platform et Cross Browser (fonctionne dans tous les navigateurs et Node.js)
- Emule les API CouchDB
- Les données sont stockées localement (IndexedDB/Web SQL)
- API asynchrone (basé sur les [Promises](https://developer.mozilla.org/fr/docs/Web/JavaScript/Reference/Objets_globaux/Promise))
- Gère le mode Offline et la synchronisation avec CouchDB.

### Installation
Commençons par l'installation, plusieurs solutions s'offrent à vous. Je préfère l'utilisation
de [Bower](https://bower.io/) ou encore mieux de [npm](https://www.npmjs.com/), mais vous
pouvez aussi directement télécharger le fichier source minifié et l'intégrer à votre code
source.

#### Téléchargement direct
Téléchargez la dernière [version](https://github.com/pouchdb/pouchdb/releases/download/5.4.5/pouchdb-5.4.5.min.js)
puis insérez la ligne suivante dans votre `index.html` :
```html
<script src="pouchdb-5.4.5.min.js"></script>
```

#### Bower
Lancez la ligne de commande suivante
```sh
$ bower install pouchdb
```

puis insérez la ligne suivante dans votre `index.html` :
```html
<script src="bower_components/pouchdb/dist/pouchdb.min.js"></script>
```

#### npm
Lancez la ligne de commande suivante
```sh
$ npm install pouchdb
```

puis insérez la ligne suivante dans votre `index.html` :
```html
<script src="node_modules/pouchdb/dist/pouchdb.min.js"></script>
```

### Création d'une base de données locale
La première étape est la création d'une base de données locale :
```javascript
var db = new PouchDB("smart-meter");
console.log("Local database created");
```

PouchDB va créer une base de données locale en utilisant la technologie disponible du navigateur :
si IndexedDB est disponible (comme sur la plupart des navigateurs récent), il privilégiera 
cette technologie. Si celle-ci n'est pas disponible, il utilisera WebSQL (Safari).

### Ajout de document
Une fois notre base de données créée, nous pouvons commencer à insérer des documents :

```javascript
var db = new PouchDB("smart-meter");
// Avec la fonction post
db.post({
  date: "2014-11-12T23:27:03.794Z",
  kilowatt_hours: 14
}).then(function() {
  console.log("Document created");
}).catch(function(error) {
    console.log(error);
});

// Avec la fonction put
db.put({
  _id: "2014-11-12T23:27:03.794Z",
  kilowatt_hours: 14
}).then(function() {
  console.log("Document created");
}).catch(function(error) {
    console.log(error);
});
```

> En utilisant la fonction `db.post()`, on laisse PouchDB générer automatiquement un identifiant
unique `_id`. Alors qu'avec la fonction `db.put()`, il faut explicitement inclure la 
propriété `_id` au document.

### Modification d'un document
Modifier un document est aussi simple qu'ajouter un document, il suffit d'utiliser la fonction
`db.put()` :

```javascript
var db = new PouchDB("smart-meter");
db.put({
  _id: "2014-11-12T23:27:03.794Z",
  kilowatt_hours: 14
}).then(function(response) {
  console.log("Document created");
  // Récupérer le document
  return db.get(response.id);
}).then(function(doc) {
  console.log("Document read");
  // Modifier la valeur de kilowatt hours
  doc.kilowatt_hours = 15;
  // Enregistrer le changement dans la database
  return db.put(doc);
}).catch(function(error) {
    console.log(error);
});
```

### Suppression d'un document
Pour supprimer un document, il suffit d'utiliser la fonction `db.remove()` :

```javascript
var db = new PouchDB("smart-meter");
db.put({
  _id: "2014-11-12T23:27:03.794Z",
  kilowatt_hours: 14
}).then(function(response) {
  console.log("Document created");
  // Récupérer le document
  return db.get(response.id);
}).then(function(doc) {
  console.log("Document read");
  // Supprimer le document de la base de données
  return db.remove(doc);
}).then(function(response) {
  console.log("Document deleted");
}).catch(function(error) {
    console.log(error);
});
```

### Requêter la base de données avec `allDocs`
Commençons par une requête simple en utilisant la fonction `db.allDocs()` qui permet récupérer
tous les documents de la base de données.

```javascript
var db = new PouchDB("smart-meter");
db.bulkDocs([
  {_id: "2014-11-12T23:27:03.794Z", kilowatt_hours: 14},
  {_id: "2014-11-13T00:52:01.471Z", kilowatt_hours: 15},
  {_id: "2014-11-13T01:39:28.911Z", kilowatt_hours: 16},
  {_id: "2014-11-13T02:52:01.471Z", kilowatt_hours: 17}
]).then(function(result) {
  console.log("Documents created");
  // Récupérer tous les documents
  return db.allDocs({include_docs: true});
}).then(function(response) {
  console.log("Documents read");
  console.log(response);
}).catch(function(error) {
  console.log(error);
});
```

De premier abord, cette fonction peut nous sembler accessoire mais il s'agit d'une fausse
impression. Car en utilisant correctement les différentes options la plupart des requêtes
peuvent être faites avec cette fonction.

Voici un petit résumé des options les plus intéressantes :

- `options.startkey` et `options.endkey` : Requête les documents ayant un ID entre `startkey`
et `endkey`.
- `options.key` : Retourne les documents ayant un ID correspondant.
- `options.keys` : Tableau de chaînes qui permet de retourner les documents ayant un ID
correspondant.

> Pour plus de détails, je vous invite à consulter la [documentation](https://pouchdb.com/api.html#batch_fetch) 

### Requêter la base de données avec `query`
Même si la fonction `allDocs()` et ses options permettent de requêter la base de
données, très souvent lorsque l'on développe une application nous avons besoin de 
requêtes plus complexes. Dans de tels cas, il devient nécessaire d'utiliser la fonction
`db.query()`.

 Elle permet d'invoquer une fonction map/reduce (view) qui aura été préalablement
 écrite et stockée dans un document de design CouchDB/PouchDB (design document) :

 - La fonction `map` transforme les documents en index
 - La fonction `reduce` agrège le résultat de la fonction `map`

Pour écrire des fonctions map/reduce la [documentation CouchDB](http://docs.couchdb.org/en/latest/couchapp/views/intro.html)
s'applique à PouchDB.

```javascript
// Création d'un design doc
var ddoc = {
  _id: '_design/index',
  views: {
    index: {
      map: function mapFun(doc) {
        if (doc.title) {
          emit(doc.title);
        }
      }.toString()
    }
  }
}

// Sauvegarder le design doc
db.put(ddoc).catch(function (err) {
  if (err.name !== 'conflict') {
    throw err;
  }
  // On ignore si le document existe déjà
}).then(function () {
  // Trouver les documents avec title === 'Lisa Says'
  return db.query('index', {
    key: 'Lisa Says',
    include_docs: true
  });
}).then(function (result) {
  // Traitement du résultat
}).catch(function (err) {
  console.log(err);
});
```

### Réplication
Entrons maintenant dans le vif du sujet et ce qui fait toute la magie de CouchDB et de PouchDB,
la réplication/synchronisation de base de données locale avec une base de données CouchDB distante.

Plusieurs patterns de réplication existent :

- **One Database Per User** : Comme son nom l'indique, chaque utilisateur possède 
sa propre base de données en local et sur le serveur CouchDB.
- **Write-Only Replication** : Les données sont générées sur le device de chaque 
utilisateur et se répliquent vers le serveur CouchDB.
- **Read-Only Replication** : Les données sont générées sur le serveur CouchDB et se répliquent
vers chaque device/utilisateur distant.
- **Bidirectional Replication** : Les données sont générées à la fois sur le device et sur
le serveur CouchDB. Les données sont répliquées vers/depuis le serveur et vers/depuis les 
devices des utilisateurs.

Comme souvent dans les applications business on souhaite partager de l'information entre les
utilisateurs, c'est ce dernier pattern et son utilisation que je vais vous présenter.

#### Réplication bidirectionnelle
Voici comment procéder pour initialiser la réplication entre la base de données locale et 
la base de données distante :

```javascript
// Création de la base de données locale
var db = new PouchDB("smart-meter");
console.log("Local database created");
// Création de l'object base de données distante
var remoteDb = new PouchDB(
  "https://my-remote-couchdb-server.business.com/smart-meter"
);
console.log("Remote database created");

// Initialisation de la réplication
var sync = db.sync(remoteDb, {
    live: true,
    retry: true
});
```

Voilà, avec ces quelques lignes nous avons paramétré une replication bidirectionnelle entre
notre base de données locale et notre base de données distante.

En activant `options.live`, PouchDB va suivre tous les changements et les répliquer 
automatiquement.

En activant `options.retry`, PouchDB va tenter de relancer la réplication en cas d'échec
(de perte de connexion notamment).

Grâce à cette configuration chaque changement (création, modification et suppression de document)
va être automatiquement répliquée entre la base de données locale et distante et inversement.

> Parfois, on souhaite contrôler à la demande la fonctionnalité de synchronisation
de données, pour cela il suffit d'appeller la fonction `db.sync()` avec `options.live: false`
à chaque fois que l'on veut lancer une synchronisation.

Une fois la réplication lancée, il est important de réagir aux différents événements que
va lancer PouchDB (en cas d'erreur ou de conflit de synchronisation par exemple) :
```javascript
sync.on("change", function(info) {
    // La réplication a créée ou modifiée un document
    console.log("On change");
    console.log(info);
  }).on("complete", function(info) {
    // La réplication a été terminée ou annulée
    console.log("On complete");
    console.log(info);
  }).on("paused", function(error) {
    // La réplication est en pause (la base de données est à jour ou l'utilisateur est offline)
    console.log("On paused");
    console.log(error);
  }).on("active", function() {
    // La réplication reprend (nouvelles modifications de réplication ou l'utilisateur est de retour online)
    console.log("On active");
    console.log("active");
  }).on("denied", function(error) {
    // Un document n'a pas réussi à se répliquer
    console.log("On denied");
    console.log(error);
  }).on("error", function(error) {
    // La réplication s'est arrêtée en raison d'une erreur irrécupérable
    console.log("On error");
    console.log(error);
  })
``` 

#### Réplication filtrée
Jusqu'ici, le processus de réplication que nous avons créé répliquait et synchronisait la base
de données complète. Ceci n'est pas très efficace, d'autant plus qu'un utilisateur 
n'a pas forcément besoin d'avoir accès à la totalité des données mais uniquement les données
qui le concerne.

Pour répondre à cette problématique nous allons utilisé la **réplication filtrée** :

- On sélectionne (grâce à l'utilisation d'une fonction) quels documents répliquer.
- La fonction filtre peut être définit localement dans PouchDB, ou à distance dans CouchDB.

Voici comment utiliser la réplication filtrée cotée PouchDB:
```javascript
var db = new PouchDB("smart-meter");
var remoteDb = new PouchDB(
  "https://my-remote-couchdb-server.business.com/smart-meter"
);

// Initialisation de la réplication filtrée
var sync = db.sync(remoteDb, {
    live: true,
    retry: true,
    filter: function(doc) {
      return doc._id >= "2014-11-13T00:00:00.000Z";
    }
});
```

Cet extrait de code ne va répliquer et synchroniser que les documents ayant un `_id` supérieur
ou égal à la date du 13 novembre 2014.

Vous pouvez aussi utiliser une fonction `filter` présente dans un design document de la base
de données distante :
```javascript
// Initialisation de la réplication filtrée avec utilisation d'un filter coté CouchDB
var sync = db.sync(remoteDb, {
    live: true,
    retry: true,
    filter: 'mydesign/myfilter'
});
```

Vous trouverez plus d'information sur la réplication filtrée dans CouchDB [ici](https://wiki.apache.org/couchdb/Replication#Filtered_Replication).

Pour conclure
-------------
L'utilisation conjointe de PouchDB et de CouchDB simplifie grandement la gestion du mode
déconnecté pour les applications web et mobile. La simplicité d'utilisation des APIs de PouchDB
permet également d'être rapidement opérationnel.

Pour ma part, je suis tombé amoureux de cette technologie et je l'utilise sur presque tout mes
projets d'applications web mobile.

ToDo
====
- Pour aller plus loin parler du plugin PouchDB find en fin de l'article