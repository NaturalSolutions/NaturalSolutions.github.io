---
layout: post
title:  "Objets Deferred et code asynchrone"
date:   2014-07-08 15:28:09
author:
    name: Gilles Bassière
    avatar: bio-photo-alt.jpg
    github: gbassiere
tags: [javascript, jquery]
categories: webdev
---


Avec JavaScript, on est habitué à gérer du code évènementiel depuis des
lustres. C'est facile : on déclare une fonction (le *event handler*) et on
l'associe à notre évènement.

Au début, on jouait avec les évènements *click* et *mouseover*. Puis, quand
AJAX est arrivé, pour traiter les réponses asynchrones, on a utilisé des
*event handlers* à usage unique : les *callbacks*. L'évènement ne vient
plus de l'UI mais le concept reste le même du point de vue développeur. Avec
les nouvelles APIs Web ([Geolocation][apigeo], [File][apifile],
[localStorage][apidb], etc), le code asynchrone se généralise et, en même
temps que les interfaces deviennent plus riches, le code est innondé de
*callbacks*.

Cela pose diverses difficultés, par exemple :

* Pour lancer mon application, je dois attendre la fin de plusieurs tâches
asynchrones, comment combiner plusieurs *callbacks* en un seul ?

* Dans un bloc *if-then-else*, une alternative est synchrone et l'autre est
asynchrone, comment s'assurer que la suite du code s'exécute toujours après que
ce bloc soit terminé ?

* Etc.

Les objets *Deferred*
=====================

L'utilitaire [`$.Deferred()`][jqdef] offre une syntaxe uniforme pour gérer des
codes synchrones et asynchrones. Il amène à la fois plus de contrôle et
plus de souplesse. On parle d'objets différés ou, plus souvent, d'objets
*deferred*.

Le fonctionnement est le suivant :

* le constructeur `$.Deferred()` initialise un objet dans l'état *pending*,

* à tout moment, on peut attacher des *callbacks* de succès avec `.done()` et
des *callbacks* d'échec avec [`.fail()`][jqfail],

* on peut faire basculer l'objet dans l'état *resolved* (résolu) en appelant
la méthode [`.resolve()`][jqresolve],

* de même, on peut faire basculer l'objet dans l'état *rejected* (rejeté) en
appelant la méthode [`.reject()`][jqreject],

* on ne peut pas revenir ensuite à l'état *pending*,

* si un *callback* est attaché à l'objet alors qu'il est déjà *resolved*
(ou *rejected*), il s'exécute tout de suite (ou jamais).

Un premier cas simple
=====================

L'objet *deferred* permet de relayer un évènement à un nombre quelconque de
*callbacks* avec une syntaxe uniforme (la même que celle utilisée par
`$.ajax`).

~~~ js
// Déclarer un objet différé
var dfd = new $.Deferred();

// Attacher à cet objet du code à éxecuter en cas de résolution
function showCoords(coords) {
    ...
}
function centerMap(coords) {
    ...
}
dfd.done(showCoords, centerMap);

// Résoudre ou rejeter l'objet quand l'API GeoLocation répond
navigator.geolocation.getCurrentPosition(
    function(pos) {
        dfd.resolve(pos.coords);
    },
    function(err) {
        dfd.reject(err.code);
    }
);
~~~


Dans cet exemple simpliste, le code pourrait facilement être réécrit en
supprimant l'objet *deferred* et en appelant les deux fonctions directement
dans le *callback* de succès de `getCurrentPosition()`.

En revanche, l'objet *deferred* devient une solution particulièrement élégante
si l'on veut découpler la gestion de la localisation et le code qui utilise
cette information. Par exemple, un premier composant pourrait gérer l'obtention
de la position et exposer uniquement l'objet *deferred*, d'autres composants
peuvent alors ajouter des callbacks sans se soucier de la manière dont la
position est obtenue.

Synchronisation
===============

La méthode [`$.when()`][jqwhen] fournit un nouvel objet *deferred* qui agrège tous ceux
qui lui sont passés en argument. Ce nouvel objet devient :

* résolu quand tous les *deferreds* agregés sont résolus
* rejeté dès qu'un des *deferreds* agrégés est rejeté.

Exemple :

~~~ js
// Lancement de deux tâches asynchrones (requête AJAX et géolocalisation)
var maRequete = $.ajax(...),
    maPosition = new $.Deferred();
navigator.geolocation.getCurrentPosition(
    function(pos) {
        dfd.resolve(pos.coords);
    }
);

// Déclaration d'un callback pour ces deux tâches
function showMap(argsRequete, argsPosition) {
    var coords = argsPosition[0],
        data = argsRequete[0];
    ...
}

// On réagit quand les deux tâches sont terminées
$.when(maRequete, maPosition).done(showMap);
~~~

Dans certains cas, on doit lancer un nombre variable de tâches et la syntaxe
de `$.when` pose alors un problème puisqu'elle impose d'énumérer les
paramètres au moment de l'écriture du code. On contourne ce problème en
stockant les *deferreds* dans un tableau et en exploitant une astuce de
programmation fonctionnelle :

~~~ js
var dfds = [];
while (...) {
    dfds.push(...);
}
$.when.apply($, dfds).done(...);
~~~

Notez que si `dfds` est vide, `$.when.apply($, dfds)` renvoit un objet deferred
immédiatement résolu.

*Goodies*
=========

Les *callbacks* dont chaînables grâce à [`.then()`][jqthen] (la méthode
`.pipe()` est dépreciée). Lorsqu'un callback est chaîné après un autre,
il reçoit les données traitées par le callback précédent.

Tant qu'un *deferred* est *pending*, on peut gérer des notifications de
progrès avec [`.notify()`][jqnotify] et [`.progress()`][jqprogress]. Utile
pour réaliser des animations pour faire patienter l'utilisateur.

On peut condenser les déclarations avec :

- `.then(a, b, c)` qui équivaut à `.done(a).fail(b).progress(c)`,
- `.always(a)` qui équivaut à `.then(a, a)`.

On peut exposer une sorte de *deferred* "en lecture seule" avec
[`.promise()`][jqpromise] : l'objet retourné à des méthodes `.done()` et
`.fail()` mais pas `.resolve()` ni `.reject()`. A utiliser impérativement
si vous exposez un objet deferred pour qu'il soit utilisé par des composants
tiers.


Inconvénients
=============

Je n'ai pas (encore ?) trouvé la bonne pratique pour produire du code élégant et
lisible lorsqu'il y a beaucoup d'objets *deferred*, en particulier lorsqu'il y a
imbrication de tâches asynchrones.

Contrairement au `.on()` de Backbone et dans une moindre mesure au `.on()`
de jQuery, les `.done()` et `.fail()` n'offre aucune facilité pour gérer le
contexte (`this`) du *callback*. Cela contraint à un usage massif des
*closures* ou de `.bind()`.

Il faut bien garder en tête le circuit logique asynchrone et prendre en compte
tous les cas. Il est important notamment de coder des *callbacks* d'échec.



[apigeo]: https://developer.mozilla.org/en-US/docs/Web/API/Geolocation
[apifile]: https://developer.mozilla.org/en-US/docs/Web/API/File
[apidb]: https://developer.mozilla.org/en-US/docs/Web/Guide/API/DOM/Storage#localStorage
[jqdef]: http://api.jquery.com/jQuery.Deferred/
[jqwhen]: http://api.jquery.com/jQuery.when/
[jqfail]: http://api.jquery.com/deferred.fail/
[jqresolve]: http://api.jquery.com/deferred.resolve/
[jqreject]: http://api.jquery.com/deferred.reject/
[jqthen]: http://api.jquery.com/deferred.then/
[jqnotify]: http://api.jquery.com/deferred.notify/
[jqprogress]: http://api.jquery.com/deferred.progress/
[jqpromise]: http://api.jquery.com/deferred.promise/
