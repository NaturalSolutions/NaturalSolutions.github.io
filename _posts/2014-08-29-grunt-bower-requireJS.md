---
layout: post
title: "Grunt, bower et RequireJS"
date: 2014-08-29 15:00
author:
    name: Antoine MICELI
    avatar: miceli.jpg
    github: amiceli
tags: [javascript, grunt, bower, require.js]
categories: webdev
comments: true
---

Grunt, Bower et RequireJS
-------------------------


Lors du développement du projet [FormBuilder], j'ai cherché à rendre le projet modulaire et surtout facile à maintenir.
Cet article a pour but de présenter les bibliothèques que j'ai utilisées et leur mise en place.

## Pré-requis

Avant de commencer, vérifiez que vous avez installé NodeJS. Sinon rendez vous sur le [site officiel de nodeJS](http://nodejs.org/).

Il faudra également créer un projet NodeJS en lançant la commande suivante dans
le dossier de votre projet :

    npm init

Installer Grunt :

    npm install -g grunt-cli
    npm install grunt --save-dev

Installer Bower :

    npm install -g bower

RequireJS ne s'installe pas, ce sera simplement un dépendance de votre projet
(voir plus loin). Par contre, vous pouvez installer l'optimiseur RequireJS :

    npm install requirejs

## Bower : JavaScript package manager

Bower est l'équivalent de [composer](https://getcomposer.org/) pour PHP ou
[PIP](https://pypi.python.org/pypi/pip) pour Python. Il permet d'installer des
bibliothèques JavaScript.
La première étape est d'initialiser Bower dans votre répertoire avec la commande
:

    bower init

Cela vous créera un fichier `bower.json` ressemblant à :

~~~ json
    {
      "name"     : "formbuilder",
      "version"  : "0.0.0",
      "homepage" : "https://github.com/NaturalSolutions/NS.UI.FormBuilder",
      "authors"  : [
          "MICELI Antoine"
      ],
      "main"     : "index.html",
      "keywords" : [
         "XML"
      ],
      "license": "MIT",
      "private": true,
    }
~~~

Ce fichier présente votre projet et permet également de lister les bibliothèques
à installer, par exemple pour le projet FormBuilder :

~~~ js
{
    // ...
    "dependencies" : {
         "jquery"             : "1.11.1",
         "jquery.ui"          : null,
         "backbone"           : null,
         "underscore"         : null,
         "NS.UI.Notification" : "https://github.com/NaturalSolutions/NS.UI.Notification.git",
         "NS.UI.Navbar"       : "git@github.com:NaturalSolutions/NS.UI.NavBar.git",
         "bootstrap"          : "2.3.2",
         "blobjs"             : "https://github.com/eligrey/Blob.js.git",
         "xmljs"              : "git@github.com:kripken/xml.js.git",
         "i18n"               : "https://github.com/jamuhl/i18next.git",
         "jsdifflib"          : "https://github.com/cemerick/jsdifflib.git",
         "font-awesome"       : "https://github.com/FortAwesome/Font-Awesome.git",
         "fancytree"          : "https://github.com/mar10/fancytree.git",
         "filesaver"          : "git@github.com:eligrey/FileSaver.js.git",
         "requirejs"          : null,
         "nanoscroller"       : null
    }
}
~~~

Pour chaque bibliothèque, on peut choisir le dépôt GitHub qui correspond, une
version spécifique ou encore null pour récupérer la dernière version depuis le
[centre de paquet Bower](http://bower.io/search/).

Si vous exécutez la commande `bower install`, cela installera toutes les
bibliothèques dans un dossier (`bower_components` par défaut mais c'est configurable).

Vous pouvez également installer une bibliothèque directement en ligne de commande : 

    bower install jquery

Outre l'installation automatique de l'environnement de développement, l'intérêt
est de garder votre code source propre. Vous n'aurez plus à copier jQuery dans
votre dépôt Git par exemple !


## RequireJS

Comme vous avez pu le remarquer, le FormBuilder dépend d'un nombre
non négligeable de bibliothèques. Afin
d'optimiser leur chargement, j'ai opté pour RequireJS.

RequireJS implémente l'API [AMD (Asynchronous Module
Definition)](http://requirejs.org/docs/whyamd.html#amd) qui permet de charger
une ou plusieurs ressources (Javascript, HTML, etc) de manière asynchrone.

La mise en place de RequireJS se fait en une balise en `script` :

    <script data-main="assets/config" type="text/javascript" src="bibliothèques/requirejs/require.js"></script>

L'attribut `data-main` indique à RequireJS l'emplacement de son fichier de configuration (JavaScript).

**Attention : pour ce qui concerne les emplacements de fichier JavaScript pour
RequireJS il ne faut pas écrire l'extension `.js`.**

### Le fichier de configuration

Voici un exemple de fichier de configuration :

~~~ js
require.config({
    baseUrl: 'js/lib',
    paths: {
         jquery: 'jquery-1.9.0'
    }
});
~~~

Pour l'instant, on charge le fichier `jquery-1.9.0.js`. Dans les autres
fichiers, nous pourrons utiliser la bibliothèque jQuery en la désignant par
"l'alias" `jquery`.

Certaines bibliothèques, comme Backbone, n'adoptent pas l'API AMD. On utilise
alors les *shim* :

~~~ js
require.config({
    // ...
    shim: {
        'jquery'       : { exports: '$' },
        'underscore'   : { exports: '_', deps : ['jquery'] },
        'backbone'     : { deps : ['underscore'] }
    },
    // ...
});
~~~

Cela permet d'exporter les symboles comme `$` pour jQuery et de définir les dépendances entre bibliothèque. Cela nous permettra de charger underscore sans se soucier de jQuery, RequireJS le fera pour nous.

### Créer un module

RequireJS permet d'organiser un projet sous forme de modules.
Par exemple pour le FormBuilder :

~~~ js
define(['backbone', 'app/router'], function($, _, Backbone, Router){
   var formbuilder = {
       initialize : function(options) {
           //  Init router
           this.router = new Router(this);
           Backbone.history.start();
       }
   };
   return formbuilder;
});
~~~

On demande à RequireJS de charger la bibliothèque Backbone et le module Router
(qui se situe dans le fichier `app/router.js`).

Une fois le module défini, nous pouvons l'utiliser dans notre fichier de
configuration qui, je le rappelle, est le point d'entrée de notre application.

~~~ js
require.config({
    // .....
});

 require(['app/formbuilder'], function(formbuilder) {
    var options = {
        autocompleteURL      : 'ressources/autocomplete/',
        translationURL       : 'ressources/locales/',
        keywordAutocomplete  : 'ressources/autocomplete/keywords.json',
        protocolAutocomplete : 'ressources/autocomplete/protocols.json',
        unitURL              : 'ressources/autocomplete/units.json'
    }

    formbuilder.initialize(options);
});
~~~

Et voilà, RequireJS est prêt ! Maintenant, il faudra adopter cette architecture
en modules dans votre projet.

Ci-dessous, un exemple qui permet de visualiser l'avantage de requireJS.

Chargez un template d'une fenêtre modale au sein d'une fonction : 

~~~ js
require(['text!../templates/modals.html'], function(modalViewTemplate) {
    var modalTemplate = $(modalViewTemplate).filter('#importProtocolModal')

    $( $(modalTemplate) ).modal({})
});
~~~

Ici, on charge le template et on créé la fenêtre modale avec Bootstrap.
Vous avez surement remarqué le `text!`, c'est une extension de RequireJS qui
permet de récupérer des fichier au format texte brute, par exemple pour le fichier HTML.

## L'optimiseur r.js

[L'optimiseur r.js](http://www.requirejs.org/docs/optimization.html) permet de concaténer vos modules en un seul fichier JS.
Pour configurer l'optimiseur on doit créer un fichier de configuration JS, par exemple build.js : 

~~~ js
({
	paths: {
        backbone             : "../../librairies/backbone/backbone",
        blobjs               : "../../librairies/blobjs/Blob",
        bootstrap            : "../../librairies/bootstrap/bootstrap",
        fancytree            : "../../librairies/fancytree/jquery.fancytree-custom.min",
        filesaver            : "../../librairies/filesaver/FileSaver",
        i18n                 : "../../librairies/i18n/i18next",
        jquery               : "../../librairies/jquery/jquery",
        jqueryui             : '../../librairies/jquery-ui/jquery-ui',
        nanoscroller         : "../../librairies/nanoscroller/jquery.nanoscroller",
        underscore           : "../../librairies/underscore/underscore",
        "NS.UI.Navbar"       : "../../librairies/NS.UI.Navbar/navbar",
        "NS.UI.Notification" : "../../librairies/NS.UI.Notification/notification",
        requirejs            : "../../librairies/requirejs/require"
    },

    shim: {
        'jquery'       : { exports: '$' },
        'underscore'   : { exports: '_' },
        'backbone'     : { deps : ['underscore', 'jquery'], exports : "Backbone"},
        "jqueryui"     : { exports: "$", deps: ['jquery'] },
        "i18n"         : { exports: "$", deps: ['jquery'] },
        "nanoscroller" : { exports: "$", deps: ['jquery', 'jqueryui'] },
        "NS.UI.Navbar" : { exports: "$", deps: ['jquery', 'backbone', 'bootstrap'] },
        "bootstrap"    : { exports: "$", deps: ['jquery'] },
    },

	baseUrl                 : '../assets/js/',
	mainConfigFile          : '../assets/js/config.js',
	name                    : 'app/formbuilder',
	out                     : 'formbuilder.min.js',
	preserveLicenseComments : false,
	

    include : ['requirejs', 'backbone'],
})
~~~

`mainConfigFile` indique à r.js le point d'entrée de l'application. Et d'où il lancera "l'optimisation".
Dans notre cas les fichiers concaténés donneront le fichier `formbuilder.min.js`.
Vous pouvez également inclure des fichiers d'autres bibliothèques : ici on inclut RequireJS et Backbone.

    node r.js -0 build.js


## Grunt + Bower + RequireJS !!

Il existe une extension Grunt pour intéger Bower et RequireJS :
[grunt-bower-requirejs](https://github.com/yeoman/grunt-bower-requirejs). Elle
permet de générer votre fichier de configuration RequireJS à partir des
bibliothèques installées par Bower.

Pour installer l'extension :

     npm install grunt-bower-requirejs --save-dev

Ajouter cette ligne à votre Gruntfile :

    grunt.loadNpmTasks('grunt-bower-requirejs');

Il reste à ajouter une tâche à votre Gruntfile :

    bower: {
        target: {
            rjsConfig: 'assets/js/config.js'
        }
    }

Pour générer votre fichier de configuration, il reste à lancer la commande :

    grunt bower:target

Ajoutez maintenant la balise HTML (si ce n'est pas fait) :

    <script data-main="assets/js/config" type="text/javascript" src="bibliothèques/requirejs/require.js"></script>

**Si vous relancez `bower:target`, la commande mettra à jour la liste des
bibliothèques mais ne touchera pas au reste du fichier, vous n'aurez pas besoin
d'y revenir à chaque mise à jour.**

### grunt-bower-task

[grunt-bower-task](https://github.com/yatskevich/grunt-bower-task) est une autre
extension permettant d'installer des bibliothèques avec Bower en nettoyant les
fichiers superflus : fichiers d'exemple, de documentation, Readme, etc.

**Attention cette extension ne fonctionne pas avec grunt-bower-requirejs
(conflit) mais je pense qu'elle reste intéressante.**

Pour ceux que cela intéresse une [discussion est en cours](https://github.com/yeoman/grunt-bower-requirejs/issues/24)
 avec l'auteur de grunt-task-requireJS.

Merci pour votre attention et à bientot sur notre blog !

[FormBuilder]: https://github.com/NaturalSolutions/NS.UI.FormBuilder/
