+++
title = "Pourquoi vos scripts d'automatisation cassent (et comment l'éviter)"
description = "Les scripts bricolés échouent en silence dès qu'un fichier change. Voici ce qui distingue un outil d'automatisation robuste d'un script fragile."
date = 2026-06-15
updated = 2026-06-15

[taxonomies]
tags = ["automatisation", "fiabilité", "rust"]

[extra]
reading_time = 4
author = "Maxime"
+++

Un script qui marche le jour où on l'écrit n'est pas un script fiable. C'est juste
un script qui n'a pas encore rencontré le cas qui le fera tomber. La vraie
question n'est pas « est-ce que ça marche ? » mais : que se passe-t-il dans six
mois, quand le fichier source change discrètement de format et que personne ne se
souvient de l'avoir écrit ?

C'est presque toujours à ce moment-là, et pas avant, qu'on découvre qu'un script
était fragile.

## Les trois causes classiques de casse

- **Hypothèses implicites** : le script suppose qu'une colonne est toujours à la
  même place, qu'un fichier existe toujours, qu'un réseau répond toujours. Le jour
  où le fournisseur ajoute une colonne ou renomme un champ, tout décale sans
  prévenir.
- **Échec silencieux** : en cas d'erreur, le script continue comme si de rien
  n'était et produit un résultat faux. C'est le pire des cas, parce que le
  problème ne se voit pas tout de suite.
- **Aucune trace** : quand ça casse, impossible de savoir *quand*, *pourquoi*, ni
  *quelles données* ont été touchées. On rejoue à l'aveugle.

## Un scénario typique

Un script lit un export CSV chaque matin et met à jour un tableau de bord. Pendant
un an, tout va bien. Puis l'outil qui génère l'export passe les montants de
`1234.50` à `1 234,50` (espace et virgule). Le script ne plante pas : il lit
« 1 » comme un nombre, ignore le reste, et le tableau de bord affiche des chiffres
faux pendant trois semaines. Personne ne s'en rend compte avant la clôture
mensuelle. Le bug n'était pas dans le code ; il était dans une **hypothèse jamais
écrite** sur le format des données.

## Ce qui rend un outil robuste

Un outil d'automatisation fiable fait quatre choses qu'un script bricolé néglige :

1. **Il valide ses entrées** avant de les traiter, au lieu de leur faire confiance.
2. **Il gère explicitement les erreurs** : chaque cas qui peut mal tourner est
   prévu et traité.
3. **Il journalise** ce qu'il fait, pour qu'on puisse reconstituer l'historique.
4. **Il s'arrête proprement** plutôt que de produire des données fausses : mieux
   vaut une alerte « je n'ai pas pu » qu'un résultat erroné présenté comme correct.

C'est exactement là que des langages comme Rust apportent une garantie
supplémentaire : beaucoup d'erreurs (un champ manquant, un type qui ne correspond
pas, un cas non traité) sont détectées **à la compilation**, avant même que
l'outil ne tourne sur de vraies données. Le script fragile, lui, ne découvre le
problème qu'en production.

## Le coût caché des scripts fragiles

Sur le papier, un script bricolé en une après-midi coûte moins cher qu'un outil
soigné. En pratique, c'est l'inverse dès qu'il casse : données corrompues à
ressaisir, décisions prises sur de faux chiffres, confiance perdue dans l'outil,
et des heures passées à déboguer dans l'urgence un code que plus personne ne
comprend. La robustesse n'est pas un luxe d'ingénieur : c'est ce qui fait qu'un
outil vous fait gagner du temps sur la durée au lieu de vous en coûter.

Pour la vue d'ensemble du sujet, retournez à la
[page pilier Automatisation](@/blog/automatisation/_index.md).

## Pour aller plus loin

Avant d'investir dans un outil robuste, assurez-vous d'automatiser la bonne tâche :
voir [Comment identifier les tâches à automatiser en priorité](@/blog/automatisation/identifier-taches-a-automatiser.md).
