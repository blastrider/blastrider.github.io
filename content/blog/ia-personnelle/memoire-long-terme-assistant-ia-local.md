+++
title = "Donner une mémoire long-terme à un assistant IA local, sans tout renvoyer au cloud"
description = "Comment Aliénor, une IA en Rust, se souvient sur la durée : SQLite comme seule source de vérité, embeddings calculés en local, rappel hybride (lexical + sémantique + graphe) et consolidation qui range la mémoire sans jamais rien supprimer. Extraits de code réels."
date = 2026-06-30
updated = 2026-06-30

[taxonomies]
tags = ["ia-personnelle", "rust", "souveraineté", "llm-local", "mémoire"]

[extra]
reading_time = 8
author = "Maxime"
+++

**TL;DR** : Un assistant sans mémoire repart de zéro à chaque conversation. Dans
Aliénor, la mémoire long-terme tient dans un seul fichier SQLite (sessions, faits
*et* vecteurs au même endroit), les *embeddings* sont calculés en local par Ollama,
le rappel combine trois signaux (mots-clés, similarité sémantique, graphe de
relations), et une consolidation en arrière-plan range la mémoire sans jamais
*supprimer* quoi que ce soit. Cet article montre les extraits de code réels qui font tenir tout ça.

> **Note** : Aliénor est mon projet personnel et un véhicule d'apprentissage, pas un
> produit. Mais les principes décrits ici sont directement transposables aux outils
> métier que je construis pour des entreprises.

## Le problème : se souvenir, mais sans fuiter ni mentir

Donner une mémoire à un assistant pose trois questions piégeuses :

1. **Où la stocker** sans qu'elle parte chez un fournisseur tiers ?
2. **Comment retrouver** le bon souvenir au bon moment, parmi des milliers ?
3. **Comment éviter** qu'elle se transforme en décharge de doublons et de
   contradictions qui finit par dégrader les réponses ?

La plupart des solutions « assistant à mémoire » répondent à la première question
en envoyant tout dans une base vectorielle hébergée. Pour une donnée sensible
(dossier client, code interne), c'est exactement ce qu'on cherche à éviter. Aliénor
part de la contrainte inverse, posée dès la décision d'architecture qui fonde la
mémoire (`docs/adr/0003-sqlite-vec-phase2.md`) : *SQLite est la seule source de
vérité, et une donnée confidentielle ne quitte jamais la machine.*

## Étape 1 : tout dans un seul fichier, vecteurs compris

Première décision contre-intuitive : **pas de base vectorielle dédiée**. Les
embeddings sont stockés comme un simple `BLOB` de `float32` dans la même table
SQLite que le reste, et la similarité est calculée en Rust. L'ADR-0003 le tranche
noir sur blanc :

> *« Les embeddings sont stockés dans une colonne `embedding BLOB` (float32
> little-endian) de la table `memories`. Le calcul cosine est fait en Rust sur le
> résultat de la requête SQL. […] Zéro dépendance C native, stack 100 % safe Rust. »*

La conversion vecteur ↔ octets tient en quelques lignes pures, partagées par tout
le code qui manipule des embeddings (`crates/alienor-domain/src/vector.rs`) :

```rust
/// Encode an embedding as little-endian `f32` bytes (4 bytes per dim).
pub fn embedding_to_bytes(v: &[f32]) -> Vec<u8> {
    v.iter().flat_map(|f| f.to_le_bytes()).collect()
}
```

L'avantage concret n'est pas que technique : **un seul fichier `.db` contient les
sessions, les faits et les vecteurs**. La sauvegarde de la mémoire, c'est la copie
d'un fichier. Pas de service externe à synchroniser, rien qui puisse diverger.

## Étape 2 : la similarité, calculée en local et *fail-safe*

Retrouver un souvenir « proche » d'une question, c'est comparer deux vecteurs par
similarité cosinus. Le détail qui compte, c'est le comportement aux cas limites :
deux vecteurs de tailles différentes, vides, ou nuls ne doivent pas faire planter
le rappel ni propager de `NaN`. La fonction retombe sur `0.0`, jamais sur une
erreur (`crates/alienor-domain/src/vector.rs`) :

```rust
/// Cosine similarity clamped to `[0.0, 1.0]`. Returns `0.0` for empty,
/// mismatched-length, or zero vectors (no NaN propagation).
pub fn cosine_similarity(a: &[f32], b: &[f32]) -> f32 {
    if a.len() != b.len() || a.is_empty() {
        return 0.0;
    }
    let dot: f32 = a.iter().zip(b).map(|(x, y)| x * y).sum();
    let norm_a: f32 = a.iter().map(|x| x * x).sum::<f32>().sqrt();
    let norm_b: f32 = b.iter().map(|x| x * x).sum::<f32>().sqrt();
    if norm_a == 0.0 || norm_b == 0.0 {
        return 0.0;
    }
    (dot / (norm_a * norm_b)).clamp(0.0, 1.0)
}
```

Les vecteurs eux-mêmes sont produits par un modèle d'embedding local
(`nomic-embed-text`, 768 dimensions) appelé via Ollama. Rien ne part au cloud, même
pour transformer un texte en vecteur : la confidentialité tient *aussi* à cet
endroit, pas seulement au moment de répondre.

## Étape 3 : un rappel hybride, pas seulement « sémantique »

La recherche purement sémantique a un défaut : elle rate les correspondances
exactes (un nom propre, une référence, un code produit) que de simples mots-clés
trouveraient immédiatement. Aliénor combine donc **trois signaux** pondérés
(`crates/alienor-memory/src/recall/mod.rs`) :

```rust
impl Default for RecallWeights {
    fn default() -> Self {
        Self {
            lexical: 0.3,   // FTS5 BM25 (mots-clés)
            semantic: 0.5,  // similarité cosinus (sens)
            graph: 0.2,     // bonus de relation à 1 saut
        }
    }
}
```

Concrètement, pour une question donnée, le store fait une passe par mots-clés
(jusqu'à 50 candidats via le moteur plein-texte FTS5 de SQLite), une passe par
similarité (top-50 par cosinus), puis ajoute un **bonus de graphe** : si un souvenir
est relié à un souvenir déjà retenu, il remonte dans le classement. Les souvenirs
ne vivent pas isolés, ils forment un réseau.

> **À retenir** : « mémoire IA » ne veut pas dire « tout balancer dans une base
> vectorielle ». Le bon rappel mélange le littéral (mots exacts), le sémantique
> (le sens) et le relationnel (le contexte autour). Chacun rattrape les angles
> morts des autres.

## Étape 4 : le filtre de confidentialité vit *dans la requête SQL*

Le rappel pourrait, par accident, remonter une mémoire plus sensible que la session
en cours. Pour rendre ça impossible, le filtre de confidentialité n'est pas appliqué
*après coup* en Rust : il est inscrit dans la requête SQL elle-même. L'ADR-0003 en
fait un invariant (D5) :

> *« Appliqué en SQL, pas en post-filtrage Rust. Garantit qu'aucun chemin de code
> ne peut retourner une mémoire plus confidentielle que la session courante. »*

C'est le même principe que pour [le routage local/cloud](@/blog/ia-personnelle/donnee-confidentielle-jamais-cloud-par-le-code.md) :
la règle est placée à un endroit où il est structurellement impossible de la
contourner, pas laissée à la discipline du développeur.

## Étape 5 : consolider sans jamais supprimer

Une mémoire qui grandit sans entretien finit par accumuler la même préférence
reformulée cinq fois, des faits jamais réutilisés, et des contradictions que rien
ne résout. Aliénor fait tourner en arrière-plan une **consolidation** en trois
temps (`crates/alienor-cognition/`) : regrouper les souvenirs très proches
(cosinus ≥ 0,92), les fusionner via le modèle local, puis faire décroître
l'importance des souvenirs jamais réutilisés.

Deux garde-fous rendent l'opération sûre.

**D'abord, on n'efface jamais.** Tout ce qui sort de la table active est *archivé*,
avec la raison et un dump complet, jamais supprimé (ADR-0014) :

> *« Pas de suppression hard. Tout ce qui sort de `memories` passe en
> `memory_archive` avec timestamp et raison. Rien n'est jamais perdu, on peut
> reconstituer. »*

**Ensuite, une fusion peu sûre est refusée.** Le risque, quand un LLM résume cinq
souvenirs en un, c'est qu'il *invente*. La fusion demande donc au modèle un score
de confiance, et sous le seuil, on jette la fusion plutôt que de polluer la mémoire
(`crates/alienor-cognition/src/consolidator/merge.rs`) :

```rust
const MIN_CONFIDENCE: f32 = 0.7;

// ...
if parsed.confidence < MIN_CONFIDENCE {
    warn!(
        confidence = parsed.confidence,
        "merge: confidence too low, skipping cluster"
    );
    return Ok(None);
}
```

Et ce n'est pas le seul cas : une erreur réseau, un flux interrompu ou un JSON
illisible renvoient aussi `Ok(None)`. La règle est uniforme : **en cas de doute,
on ne fusionne pas.** Le pire cas est de garder deux souvenirs séparés, jamais de
créer un faux souvenir.

> **À retenir** : la qualité d'une mémoire long-terme ne tient pas qu'à ce qu'elle
> retient, mais à ce qu'elle refuse de faire. Archiver plutôt que supprimer, et
> renoncer à une fusion douteuse, c'est ce qui sépare un assistant qui s'améliore
> avec le temps d'un assistant qui dérive lentement vers le faux.

## Ce que ça change pour une PME, une TPE ou un indépendant

L'équivalent côté entreprise, ce serait un assistant qui connaît vos dossiers
récurrents, vos process internes et le vocabulaire de votre métier, sans qu'on ait
à tout lui réexpliquer chaque matin, et sans que ces informations partent chez un
tiers. Trois principes transposables, quelle que soit la taille du projet :

- **La mémoire d'un outil métier peut tenir en local.** Un fichier qu'on sauvegarde
  d'une copie vaut mieux qu'un service externe de plus à sécuriser.
- **Bien retrouver l'information, c'est mélanger les approches.** Le tout-sémantique
  est à la mode mais rate les correspondances exactes ; un bon outil combine
  plusieurs signaux.
- **Un système qui se nettoie tout seul doit le faire prudemment.** Archiver au lieu
  de supprimer, et refuser une opération incertaine, ça se conçoit dès le départ.

C'est précisément ce type de garantie que j'intègre dans les outils que je construis.
[Décrivez votre besoin](https://ferrix.fr/#contact) et on regarde ensemble ce qui est
pertinent pour votre cas.

## FAQ

### Pourquoi SQLite plutôt qu'une vraie base vectorielle ?

Parce que pour le volume visé (quelques milliers à quelques dizaines de milliers de
souvenirs), un scan en mémoire avec calcul cosinus en Rust est largement assez
rapide, et que tout tient alors dans un seul fichier facile à sauvegarder. Une base
vectorielle dédiée se justifie à plus grande échelle ; l'architecture d'Aliénor
prévoit d'ailleurs explicitement de la reconsidérer au-delà d'un seuil mesuré.

### Les embeddings partent-ils au cloud ?

Non. Le modèle d'embedding (`nomic-embed-text`) tourne en local via Ollama. La
transformation d'un texte en vecteur ne sort jamais de la machine, au même titre
que les réponses sur données confidentielles.

### Comment l'assistant retrouve-t-il le bon souvenir ?

Par un rappel hybride : une recherche par mots-clés, une recherche par similarité
de sens, et un bonus pour les souvenirs reliés à ceux déjà retenus. Les trois
scores sont pondérés et combinés, ce qui rattrape à la fois les correspondances
exactes et les reformulations.

### La consolidation peut-elle détruire une information importante ?

Non : rien n'est supprimé, tout ce qui sort de la mémoire active est archivé avec sa
raison et peut être reconstitué. Et une fusion dont le modèle n'est pas suffisamment
sûr (score sous le seuil) est rejetée plutôt qu'appliquée.

### Peut-on appliquer ça à un outil d'entreprise plus simple ?

Oui. Stockage local dans un fichier, rappel combinant plusieurs signaux, nettoyage
prudent qui archive au lieu de supprimer : ces principes ne dépendent pas de la
taille du projet.

## Pour aller plus loin

Cet article fait partie d'une série sur la construction d'Aliénor. Retournez à la
[présentation de la série](@/blog/ia-personnelle/_index.md) pour la vue d'ensemble,
lisez [pourquoi et comment j'ai construit cette IA](@/blog/ia-personnelle/ia-personnelle-rust-pourquoi-comment.md),
ou voyez comment [une donnée confidentielle ne part jamais au cloud](@/blog/ia-personnelle/donnee-confidentielle-jamais-cloud-par-le-code.md).
