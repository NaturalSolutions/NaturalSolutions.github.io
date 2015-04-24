---
layout: post
title: "Héberger une application Flask sur IIS"
date: 2015-04-24 15:00
author:
    name: MICELI Antoine
    avatar: miceli.jpg
    github: amiceli
tags: [python, Flask, IIS]
categories: webdev
comments: true
---

Héberger une application python sur IIS
-------------------------

Lors du développement du projet [FormBuilder](https://github.com/NaturalSolutions/NS.UI.FormBuilder) j'ai cherché à héberger la partie backend développée en python 3.4 avec le framework Flask sur IIS.
Ce post explique une des méthodes possibles pour héberger une application python sur IIS.

## Prérequis

Avant de commencer vous devez télécharger l'utilitaire NSSM : [page de téléchargement](https://nssm.cc/download). Cet outil permet de gérer facilement les services windows.

## Création du service

La première étape est la création d’un service qui lancera notre script python en tant que service Windows avec NSSM.

### Création du service avec NSSM

Pour créer un service il faut lancer l’utilitaire en ligne de commande :

![NSSM en ligne de commande](/images/posts/commands.jpg)

Remplacer **servicePython** par le nom que vous souhaitez donner à votre service. 

![création du service](/images/posts/service1.jpg)

Pour Flask fonctionnant sur Python 3.4 voilà notre configuration : 

![création du service](/images/posts/service2.jpg)

* **Chemin** : emplacement de python 3.4
* **Répertoire de démarrage** : configuré automatiquement par NSSM
* **Paramètre** : URL du fichier python qui lance votre serveur
* Cliquer sur Installer le service.

### Vérifier que le service est exécuté

Pour vérifier que votre service a bien été créé il faut vérifier la liste des services.

Pour obtenir la liste des services et leur état : Clic droit sur ordinateur puis **gérer**

![création du service](/images/posts/services.jpg)

Ensuite double cliquer sur **services et applications** : 

![création du service](/images/posts/services1.jpg)

Cliquez ensuite sur **Services** : 

![création du service](/images/posts/services2.jpg)

Vérifiez que votre service soit présent et qu’il soit démarré

![création du service](/images/posts/list.jpg)

Pour vérifier que votre service est en cours vous pouvez aussi allez à l’adresse habituelle de votre serveur (pour flask localhost :5000) et vérifier que le serveur vous retourne quelque chose.

![création du service](/images/posts/works.jpg)

Si votre serveur retourne bien quelque chose vous pouvez passer à la prochaine étape.

## Ajouter une application dans IIS

Ouvrez IIS et déroulez la liste **Sites**.

![création du service](/images/posts/iis.jpg)

Faites un clic droit sur Default Web site (dépend de votre configuration) et cliquez sur nouvelle application.

![création du service](/images/posts/pool.jpg)

Saisissez le nom de votre application dans **Alias** et renseignez le chemin physique de votre application c’est-à-dire le dossier racine d’application (là où se situe le fichier qui lance le serveur Flask).

Une fois l’application créée la prochaine étape est de créer un proxy inversé pour rediriger les requêtes de votre application vers votre service python créé précédemment.

## Rediriger les requêtes vers le service python

Toujours sur IIS cliquer du Default Web Site (dossier parent de votre application) et cliquer sur **Réécriture d’URL** : 

![création du service](/images/posts/url.jpg)

Cliquez ensuite sur **Ajouter de règles** : 

![création du service](/images/posts/rules.jpg)

Choisissez ensuite **proxy inversé**

![création du service](/images/posts/proxy.jpg)

Saisissez l’URL de votre serveur python : 

![création du service](/images/posts/config.jpg)

Cliquez sur **OK**.

Pour terminer la configuration du proxy double cliquez sur la règle que vous venez de créer.

![création du service](/images/posts/proxy2.jpg)


Dans la fenêtre de configuration vérifier que le modèle a pour valeur le nom de votre application IIS et /(.*) a la fin.

De même vérifier que la Réécriture d’URL a pour valeur l’adresse de votre serveur python par exemple : http://localhost:5000/{R:1}

![création du service](/images/posts/config2.jpg)

## Tester la redirection

Pour tester si la redirection fonctionne vous pouvez aller sur l’adresse localhost/nomDeLapplication par exemple localhost/formbuilderWS, vous devriez avoir le même résultat que sur localhost :5000.


