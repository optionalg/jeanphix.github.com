---
layout: post
lang: fr
title: "Nouvelles du Front: Internet Explorer nous retourne le cerveau"
tags:
- frontend
- css
---

Depuis quelques temps, je fais de plus en plus de front, ce qui ne me déplaît pas. C'est un domaine très riche où doivent cohabiter: standards, navigateurs et utilisateurs.

C'est sans aucun doute la part de mon métier la plus gourmande en veille et apprentissage. Aussi je me permettrai autour de quelques petits billets de vous faire part, ici même, de mes humeurs et réflexions à ce sujet.

Aujourd'hui, je voudrais vous parler des media queries et des pratiques qui semblent imposées par le retard d'implémentation d'Internet Explorer (les media queries n'y sont interprétées que depuis sa version 9).

## Intégrer naturellement

En général quand je commence une feuille de style, elle débute par la mise en place de la typographie: police de caractères, rythme (line-height, margin des hX, ul, p...) ce qui amène généralement un style suffisant et utilisable sur les plus petits devices.

Ensuite, je commence à mettre en place le layout cible en fonction des dimensions du viewport, exemples:

* si le viewport est assez grand, je peux faire "flotter" le menu de navigation.
* si le viewport est encore plus grand, je peux me permettre un affichage sur trois colonnes
* ...

En gros : je vais du plus petit au plus grand en incrémentant la feuille de style à l'aide de media queries, qui sont donc du type :

{% highlight css %}
@media screen and (min-width: 800px) {
    ...
}
{% endhighlight %}

C'est de loin la façon de faire qui me semble la plus naturelle.

## Intégrer en ciblant Internet Explorer

J'ai pu constater en décortiquant certaines intégrations, certaines librairies de style que nombre d'entre nous procèdent à l'inverse: En montant d'abord les styles pour un viewport de dimensions desktop (pour un affichage correct par IE8-) et ensuite en les dégradant pour des viewports plus petits.

Les media queries employées deviennent donc du type :

{% highlight css %}
@media screen and (max-width: 800px) {
    /*
        j'annule des flottants, des fixés...
    */
}
{% endhighlight %}

Cela s'appelle tout simplement : "raisonner à l'envers", et tout ceci, car monsieur Internet Explorer l'impose.

## La solution

Elle est pour moi des plus simples : ne pas supporter IE8-.

Sinon, si vraiment vous n'avez pas le choix, il existe une petite librairie javascript sympa et légère qui permet de rendre les media queries supportables par les navigateurs obsolètes: [Respond.js](https://github.com/scottjehl/Respond).

ps: une autre petite librairie à mettre dans votre boîte à outils: [selectivizr](http://selectivizr.com/).
