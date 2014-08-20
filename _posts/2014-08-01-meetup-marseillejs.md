---
layout: post
title:  "Rencontre MarseilleJS"
date:   2014-08-01 12:50:00
author:
    name: Gilles Bassière
    avatar: bio-photo-alt.jpg
    github: gbassiere
tags: [javascript, communauté]
categories: evenement
comments: true
---

Nous avons eu le plaisir d'accueillir hier soir le meetup [MarseilleJS] dans
les locaux de Natural Solutions. Au menu : des trucs et astuces JavaScript,
des chips et de la bière :)

J'essaye de résumer ci-dessous les principaux sujets abordés.

Quand Javascript fait ramer l'UI et les animations
--------------------------------------------------

[Jean-Pierre Vincent] ([@theystolemynick]) a fait un point sur un problème
trop fréquent : pendant l'exécution d'un bout de JavaScript un peu gourmand en
ressources, les animations deviennent saccadées, l'UI du navigateur peut même
complètement se figer. On peut le vérifier avec cet [exemple de
code][fiddle1].

Le fond du problème est la manière dont le navigateur partage la ressource CPU
entre le rendu, le DOM et l'interpréteur javascript.

Plusieurs solutions sont envisageables :

* On peut lancer le traitement dans un [*Web worker*] pour le déleguer
  à un autre *thread*. Cette solution est excellente du point de vue de la
  performance puisque c'est la seule à mettre en œuvre un vrai parallélisme. En
  revanche, les *Web Workers* ne sont [pas disponibles sur tous les
  navigateurs][workerdispo] et cela pose des contraintes dans l'organisation du
  code (communication par message, pas d'accès au DOM dans le *worker*).

* Une solution basée sur `requestAnimationFrame()` est évoquée mais elle
  s'avère médiocre pour répondre au problème de compétition pour les ressources.
  En effet, cette fonction est liée à des évènements de rendu du navigateur qui
  ralentissent le traitement que l'on souhaite faire. Il est préférable de
  réserver cette fonction à la coordination d'animation.

* Microsoft a proposé la méthode [`setImmediate()`] pour résoudre ce problème
  mais l'implémentation n'est faite que pour IE. Il existe un
  [polyfill][polyfillSI] efficace cependant.

* La solution recommandée consiste à faire régulièrement une pause dans
  le traitement et à le relancer avec `setTimeout(..., 0)`. On fragmente donc
  notre boucle. Le *setTimeout* introduit de l'asynchronisme et permet de
  laisser régulièrement la main au navigateur pour que l'interface reste
  réactive. Cette solution a l'avantage d'être très largement supportée. Un 
  exemple de code est disponible ci-dessous et [ici en live][fiddle2].

~~~ js
// i est initialisé avant de commencer pour que les boucles puissent
// reprendre là où la précédente s'est arrêtée
var i=0;

// l'algo gourmand est isolé dans une IIFE qui est nommé
// pour pouvoir être rappelé par setTimeout
(function traitement() {
    var dernierDebut = new Date();

    // NB: le premier terme est vide, on reprend i dans
    // le contexte extérieur avec sa valeur courante.
    for (; i<10000000; i++) {

        // Toutes les 50ms, on interrompt la boucle et on demande au
        // navigateur de la reprendre dès que possible
        if ((new Date() - dernierDebut) > 50) {
            setTimeout(traitement, 0);
            return;
        }

        // Placer ici le corps "utile" de votre boucle, par exemple
        'abcdefghijklmnopqrstuvwxyz'.search(/js/);
    }
}());
~~~

À noter : il s'agit d'un *pattern* bien établi et implémenté notamment dans les
méthode [debounce] et [throttle] d'Underscore.

Performance, gestion du cache, de la mémoire...
-----------------------------------------------

Après la présentation de Jean-Pierre, on a échangé autour du thème de la
performance. Plein de questions et d'astuces ont émergé, j'ai noté celles-ci,
en vrac :

* Lorsqu'on utilise JavaScript de manière un peu intensive, on peut saturer les
ressources du navigateur et tout devient lent. Il peut y avoir plein de raisons
à cela mais on peut commencer par essayer de traquer les fuites ou les abus de
mémoires grâce aux outils de développement de Chrome, notamment avec le "[Heap
snapshot]". L'idée est de faire plusieurs *snapshots* d'affilée puis d'utiliser
la fonction de comparaison pour comprendre quelles variables s'accumulent dans
la mémoire indûment. La [TimeLine] de Chrome est également utile pour analyser
le comportement de l'application. L'outil payant dynaTrace peut également être
utile pour analyser les piles d'appel de fonctions et éventuellement repérer
des appels inutile.

* On ne peut pas envisager la performance sans mettre en cache les ressources
statiques. La seule bonne manière consiste à publier ces ressources avec un
délai d'expiration très long tout en incluant un *token* dans le chemin. Ainsi,
le cache du navigateur est très bien exploité, mais on peut invalider à tout
moment ce cache en changeant la valeur du *token*. Cela suppose évidemment
d'avoir un système de *build* ou de gestion de contenu qui soit capable
d'introduite ce token dans les chemins et le code HTML. À noter : on voit
souvent ce *token* ajouté à la fin de l'URL dans la *querystring* mais ça peut
poser des problèmes à certains proxy, il est donc préférable de mettre le
*token* directement dans le chemin.

* [Cyril Moreau] nous racontait son expérience avec *base64*. Il a travaillé
sur un projet où les images n'étaient plus référencées dans les CSS avec
`url()` mais directement incluses, formatées en *base64*. Il y a au moins
deux intérêts : on réduit le nombre de requêtes HTTP pour charger la page et,
surtout, les images sont déjà disponibles lorsque le navigateur applique le
style. En revanche, il faut disposer d'un processus de *build* pour convertir
automatiquement les images en *base64* et les intégrer aux CSS, sans quoi le
code serait très difficilement lisible.

* Les outils de développement de Chrome permettent de consulter un site mais de
[charger une partie des ressources depuis le disque local][workspaces]. Le cas
d'utilisation est le suivant : je navigue sur mon site en prod mais Chrome
remplace les scripts par ceux que je lui indique sur mon disque, je peux ainsi
tester/déboguer mes scripts facilement. Notez que même si les fichiers sont
minifiés, Chrome utilise la *source map* pour charger les fichiers locaux non
minifiés. Enfin, cerise sur le gateau, quand vous éditez ces fichiers locaux
dans la console, vous pouvez sauver vos modifications avec un `Ctrl-S`. Le
développement Web n'aura jamais été aussi fluide !

* Pour l'analyse de performance, [webpagetest.org] est un outil très pratique,
il permet de faire charger son site par divers navigateurs sur divers OS à
différents endroits du globe. Le site fournit ensuite la *timeline* du
chargement, des captures d'écrans et plein d'autres informations utiles.

* [SPOF-O-Matic] est un extension Chrome qui analyse les pages consultées et
détecte les risque de SPOF (*Single Point of Failure*) que constituent les
scripts externes chargés en haut de page.

Pour suivre les nouvelles de MarseilleJS, rejoignez le [groupe de discussion
Google].



[MarseilleJS]: http://francejs.org/MarseilleJS/
[Jean-Pierre Vincent]: http://braincracking.org
[@theystolemynick]: http://www.twitter.com/theystolemynick
[fiddle1]: http://jsfiddle.net/qucrnfv3/1/
[*Web worker*]: https://developer.mozilla.org/docs/Web/API/Worker
[workerdispo]: http://caniuse.com/#feat=webworkers
[`setImmediate()`]: http://msdn.microsoft.com/library/ie/hh920766.aspx
[polyfillSI]: https://github.com/YuzuJS/setImmediate
[fiddle2]: http://jsfiddle.net/qucrnfv3/2/
[debounce]: http://underscorejs.org/#debounce
[throttle]: http://underscorejs.org/#throttle
[Heap snapshot]: https://developer.chrome.com/devtools/docs/heap-profiling
[TimeLine]: https://developer.chrome.com/devtools/docs/timeline
[Cyril Moreau]: http://sojavascript.com/
[workspaces]: https://developer.chrome.com/devtools/docs/workspaces
[webpagetest.org]: http://www.webpagetest.org/
[SPOF-O-Matic]: https://chrome.google.com/webstore/detail/spof-o-matic/plikhggfbplemddobondkeogomgoodeg
[groupe de discussion Google]: https://groups.google.com/forum/?fromgroups#!forum/marseillejs
