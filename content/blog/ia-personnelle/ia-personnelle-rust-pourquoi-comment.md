+++
title = "J'ai construit ma propre IA personnelle en Rust : pourquoi et comment"
description = "Retour d'expérience sur Aliénor : une IA personnelle hybride local/cloud en Rust, à mémoire long-terme, pensée comme une extension cognitive et non un wrapper ChatGPT. Et ce que ça change pour une entreprise."
date = 2026-06-18
updated = 2026-06-18

[taxonomies]
tags = ["ia-personnelle", "rust", "souveraineté", "llm-local"]

[extra]
reading_time = 6
author = "Maxime"
+++

**TL;DR** : J'ai conçu *Aliénor*, une IA personnelle écrite en Rust, qui tourne
par défaut en local (sur ma propre machine) et ne fait appel au cloud que lorsque
la qualité l'exige **et** que la confidentialité le permet. Elle a une mémoire
long-terme, peut agir sur le système sous garde-fous, et n'est pas un simple
habillage de ChatGPT. Cet article explique pourquoi un tel projet a du sens, et
ce que les PME, TPE et indépendants peuvent en retenir pour leurs propres outils.

## Pourquoi ne pas se contenter d'un abonnement ChatGPT ?

La réponse courte : **la dépendance et la confidentialité**.

Un assistant IA grand public, c'est un abonnement mensuel par utilisateur, des
données envoyées sur des serveurs étrangers, et des conditions qui changent sans
préavis. Pour un usage personnel anodin, peu importe. Mais dès qu'on manipule de
l'information sensible (un dossier client, un contrat, des données RH), la
question devient sérieuse : *où vont vraiment ces données, et qui peut les lire ?*

Aliénor part d'un principe inverse : **le local d'abord**. Le modèle tourne sur
ma machine, les données restent chez moi, et rien ne part vers un service tiers
sans une raison explicite.

### Le parallèle avec l'entreprise

C'est exactement la problématique que rencontrent beaucoup de PME et d'indépendants.
Un outil métier qui envoie tout sur un cloud propriétaire vous rend captif : du
tarif, de la disponibilité du service, et de la politique de confidentialité du
fournisseur. Construire (ou faire construire) un outil souverain, c'est reprendre
la main. C'est le même fil rouge que dans la
[présentation de cette série](@/blog/ia-personnelle/_index.md).

## Hybride local/cloud : ne pas choisir entre confidentialité et qualité

Le piège classique du « tout local », c'est la qualité : les petits modèles qui
tournent sur une machine ordinaire sont moins bons que les gros modèles cloud.
Le piège du « tout cloud », c'est la confidentialité.

Aliénor tranche par une règle simple, **inscrite dans le code et non dans une
option de configuration** : une donnée marquée `Confidential` ne part *jamais* au
cloud. Le routage choisit le modèle local par défaut, et ne bascule vers un
fournisseur cloud que si la tâche le justifie *et* que la donnée n'est pas sensible.

> **À retenir** : La confidentialité ne doit pas être une case à cocher qu'on peut
> oublier de cocher. Dans un outil bien conçu, c'est une *garantie structurelle* :
> le code rend l'erreur impossible, plutôt que de compter sur la vigilance de
> l'utilisateur.

## Une IA qui se souvient

Un assistant sans mémoire repart de zéro à chaque conversation. Aliénor dispose
d'une **mémoire long-terme** : elle se souvient des échanges passés, des préférences,
des décisions prises. Techniquement, cela repose sur une base locale (SQLite),
des *embeddings* pour la recherche par similarité, et une consolidation qui
s'exécute en arrière-plan pour résumer et ranger les souvenirs anciens.

Pour une entreprise, l'équivalent serait un assistant qui connaît vos dossiers
récurrents, vos process internes, le vocabulaire de votre métier, sans qu'on ait
à tout lui réexpliquer chaque matin.

## Une IA qui agit, mais sous garde-fous

Le vrai saut, ce n'est pas de répondre à des questions : c'est d'**agir**. Lire
l'état d'un système, éditer un fichier, lancer une commande. Aliénor en est
capable, mais chaque action passe par trois filets de sécurité :

1. **Dry-run d'abord** : l'IA *planifie* avant d'*appliquer*. La prévisualisation
   est garantie par le système de types, pas par une convention.
2. **Allow-lists et confirmation** : certaines actions exigent une validation
   humaine explicite, hors d'atteinte de l'IA elle-même.
3. **Journal d'audit** : tout est tracé de façon infalsifiable.

C'est précisément ce qui sépare un outil d'automatisation *fiable* d'un script
qui peut tout casser. Le même principe guide nos articles sur
[l'automatisation](@/blog/automatisation/_index.md).

## Pourquoi Rust ?

Rust apporte deux choses décisives pour ce genre de projet :

- **La fiabilité** : beaucoup d'erreurs sont détectées à la compilation, avant
  même d'exécuter le programme. Pour un outil qui agit sur un système, c'est la
  différence entre « ça marche » et « ça marche vraiment, tout le temps ».
- **L'autonomie** : un binaire Rust tourne sans runtime à installer, sans
  dépendance lourde. On le déploie où on veut, on n'est captif de rien.

Ce sont exactement les qualités que je recherche pour les outils métier que je
construis pour des entreprises : pas de surprise silencieuse, pas de dépendance
imposée.

## Ce qu'une PME ou un indépendant peut en retenir

Aliénor est un projet *personnel* et un véhicule d'apprentissage, pas un produit.
Mais les principes qu'il illustre sont directement transposables :

- **La souveraineté des données est un choix d'architecture**, pas une promesse
  marketing.
- **Un outil sur mesure** colle à votre métier là où un SaaS générique vous force
  à vous adapter à lui.
- **La fiabilité se construit dans le code**, pas dans la documentation.

Si l'un de ces sujets résonne avec un besoin réel de votre entreprise (un outil
interne, une automatisation, un assistant qui ne fait pas fuiter vos données),
c'est précisément le genre de problème que j'aide à résoudre.

## FAQ

### Une IA locale, c'est réaliste pour une petite entreprise ?

Oui, à condition de calibrer les attentes. Un modèle local ne remplace pas un
gros modèle cloud sur toutes les tâches, mais il couvre très bien les usages
récurrents et sensibles, au prix d'aucune fuite de données et sans abonnement
par utilisateur.

### Faut-il du matériel coûteux ?

Pas nécessairement. Aliénor est pensée pour tourner sur du matériel modeste (une
carte graphique grand public, voire en mode dégradé sur CPU). Le dimensionnement
dépend de l'usage réel, qui s'évalue au cas par cas.

### Pourquoi écrire ça en Rust plutôt qu'en Python ?

Pour la fiabilité et l'autonomie de déploiement. Python reste excellent pour
prototyper et pour l'écosystème ML, mais un outil destiné à *agir* en production
gagne énormément à être écrit dans un langage qui attrape les erreurs avant
l'exécution.

### Est-ce que mes données restent vraiment privées ?

Dans une architecture comme celle d'Aliénor, oui : une donnée marquée
confidentielle ne quitte jamais la machine, et cette garantie est imposée par le
code lui-même, pas par un réglage qu'on peut oublier.

### Comment démarrer un projet d'outil souverain dans mon entreprise ?

En partant d'un problème concret et borné (une tâche répétitive, un besoin
d'assistant sur un domaine précis) plutôt que d'un grand projet « IA » abstrait.
[Décrivez votre besoin](https://ferrix.fr/#contact) et on évalue ensemble si c'est
pertinent.

## Pour aller plus loin

Cet article est le point de départ d'une série. Les approfondissements techniques
(architecture hexagonale, routing local/cloud, mémoire, agents, fleet SSH) seront
publiés au fil de l'eau. Retournez à la
[présentation de la série](@/blog/ia-personnelle/_index.md) pour la vue d'ensemble.
