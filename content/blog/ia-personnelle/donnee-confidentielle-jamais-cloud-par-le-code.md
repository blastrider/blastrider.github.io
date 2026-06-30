+++
title = "Garantir qu'une donnée confidentielle ne parte jamais au cloud : par le code, pas par une option"
description = "Comment Aliénor, une IA en Rust, empêche structurellement une donnée sensible de partir vers un LLM cloud. Extraits de code réels : l'invariant de confidentialité, le fail-safe et l'audit infalsifiable."
date = 2026-06-19
updated = 2026-06-19

[taxonomies]
tags = ["ia-personnelle", "rust", "souveraineté", "confidentialité", "rgpd"]

[extra]
reading_time = 7
author = "Maxime"
+++

**TL;DR** : Dans Aliénor, une donnée marquée *confidentielle* ne peut **pas**
partir vers un modèle cloud. Ce n'est pas une case à cocher qu'on pourrait oublier
de cocher : c'est une règle inscrite dans le code, vérifiée à chaque appel, avec un
comportement *fail-safe* (en cas de doute, on classe au plus restrictif) et un
journal d'audit infalsifiable. Cet article montre les extraits réels qui le
prouvent, pas du pseudo-code.

> **Note** : Aliénor est mon projet personnel et un véhicule d'apprentissage, pas
> un produit. Mais les principes décrits ici sont directement transposables aux
> outils métier que je construis pour des entreprises.

## Le problème : la confidentialité « en option » ne tient pas

La plupart des assistants IA gèrent la confidentialité par configuration : un
réglage, une politique, une promesse. Le souci, c'est qu'une option peut être mal
réglée, oubliée, ou contournée par un chemin de code qu'on n'avait pas prévu. Pour
des données soumises au RGPD (dossiers clients, code source d'entreprise, clés),
ça ne suffit pas.

C'est exactement la contrainte posée noir sur blanc dans la décision d'architecture
qui fonde le modèle (`docs/adr/0004-confidentialite-3-niveaux.md`) :

> *« Maxime utilise Claude.ai manuellement aujourd'hui ; sans garde-fou, Aliénor
> pourrait envoyer vers le cloud des données qu'il n'enverrait jamais à la main. »*

La réponse n'est pas une option de configuration. C'est un **type**.

## Étape 1 : La confidentialité est un type, pas un booléen

Le niveau de confidentialité est un `enum` ordonné, défini dans le cœur métier
(`crates/alienor-domain/src/confidentiality.rs`) :

```rust
/// Three-level confidentiality model.
/// Ordering: Public < Internal < Confidential.
pub enum ConfidentialityLevel {
    Public,
    Internal,
    Confidential,
}
```

Trois niveaux, **ordonnés** : `Public < Internal < Confidential`. Cet ordre n'est
pas décoratif : il permet de comparer des niveaux et de toujours retenir le plus
strict.

## Étape 2 : Le *fail-safe*, en cas de doute, le plus restrictif

Voici le détail qui fait la différence entre « sûr sur le papier » et « sûr en
vrai ». Quand un niveau est relu depuis le stockage et qu'il est illisible ou
inconnu, le code ne plante pas et ne choisit pas « public » par défaut : il
retombe sur **`Confidential`** :

```rust
/// Inverse of `ordinal`. Unknown values fail closed to
/// `Confidential` (the most restrictive - never leak by mis-decode).
pub fn from_ordinal(n: u8) -> Self {
    match n {
        0 => Self::Public,
        1 => Self::Internal,
        _ => Self::Confidential,
    }
}
```

C'est le principe du *fail closed* : une erreur de décodage ne peut jamais
*déclasser* une donnée vers un niveau moins protégé. Le pire cas, c'est de
sur-protéger, jamais de fuiter.

## Étape 3 : L'invariant « confidentiel → local uniquement, toujours »

Le routeur, qui décide quel modèle (local ou cloud) traite une requête, applique
une règle unique et non contournable (`crates/alienor-router/src/policy.rs`) :

```rust
pub fn validate_confidentiality(
    &self,
    provider: &dyn LlmProvider,
    confidentiality: ConfidentialityLevel,
) -> Result<(), RouterError> {
    if confidentiality == ConfidentialityLevel::Confidential
        && !provider.is_local()
        && self.confidential_always_local
    {
        return Err(RouterError::ConfidentialityViolation {
            provider: provider.name().to_string(),
        });
    }
    Ok(())
}
```

Si la donnée est `Confidential` **et** que le fournisseur n'est pas local, la
requête est **refusée** : elle renvoie une erreur `ConfidentialityViolation`. Il
n'y a pas de branche « sauf si… » : c'est une porte fermée.

## Étape 4 : La porte est franchie *avant* tout le reste

Un garde-fou ne vaut que s'il est impossible de le sauter. Dans le chemin
d'exécution du routeur (`crates/alienor-router/src/router/execute.rs`), le contrôle
de confidentialité est l'**étape 2**, juste après la résolution du fournisseur et
**avant** le cache, le budget, le circuit breaker et l'envoi réel :

```rust
// 2. Confidentiality gate (invariant: Confidential → local only, always).
self.policy
    .validate_confidentiality(provider.as_ref(), request.confidentiality)?;

// 3. Semantic cache (skip on Confidential or when cache is off).
// 4. Budget gate for cloud providers.
// 5. Circuit breaker.
// 6. Retry loop (with cascade for non-Confidential).
```

L'ordre compte : le `?` après la vérification stoppe net toute requête en
violation **avant** qu'aucun octet ne puisse partir. Et on note au passage que le
cache sémantique lui-même est sauté pour les données confidentielles : pas de fuite
indirecte par un cache partagé.

## Étape 5 : Et on peut le prouver après coup

Garantir, c'est bien. Pouvoir vérifier *a posteriori*, c'est mieux. Chaque action
est inscrite dans un journal d'audit **chaîné par hachage SHA-256**
(`crates/alienor-memory/src/audit_store/hash.rs`) :

> *« Each row's `hash` is `SHA-256` of its canonical field representation
> concatenated with the previous row's `hash`. Any deletion or modification of a
> row breaks `prev_hash` for every following row, which `verify_chain` detects. »*

Le niveau de confidentialité fait partie des champs hachés. Concrètement :
supprimer ou modifier une ligne du journal casse la chaîne pour **toutes** les
lignes suivantes, et la vérification le détecte. On ne peut pas réécrire l'histoire
discrètement.

## Pourquoi le cœur métier est « pur »

Si ces règles tiennent, c'est aussi parce qu'elles vivent dans une couche qui ne
dépend de rien d'externe. La crate `alienor-domain` (où sont définis le type de
confidentialité et les contrats) ne tire qu'une poignée de dépendances *de
données* (`uuid`, `chrono`, `serde`, `thiserror`). **Aucun** client réseau, aucune
base de données, aucun runtime. Le réseau ne peut littéralement pas être invoqué
depuis l'endroit qui définit la règle.

C'est ce qui distingue une garantie d'une intention : la règle est définie là où il
est *impossible* d'appeler le cloud, et appliquée à la seule frontière qui, elle,
le peut.

## À l'échelle : 58 crates, une centaine de décisions documentées

Ce niveau de rigueur n'est pas un cas isolé. Aliénor est un workspace Cargo de
**58 crates** (une responsabilité par crate), et chaque choix structurant est tracé
dans une **décision d'architecture** numérotée : il y en a aujourd'hui **une
centaine** (`docs/adr/0001` à `0101`). L'invariant de confidentialité décrit ici, c'est
l'ADR-0004. La nature non-contournable n'est pas un effet du hasard : c'est une
décision écrite, datée et revue.

> **À retenir** : La souveraineté des données ne se promet pas, elle se construit.
> La différence entre « on fait attention » et « c'est impossible de se tromper »
> tient dans l'architecture : un type plutôt qu'un booléen, un garde-fou placé
> avant l'action plutôt qu'après, un fail-safe vers le plus strict, et un audit
> qu'on ne peut pas falsifier.

## Ce que ça change pour une PME, une TPE ou un indépendant

Vous n'avez pas besoin de 58 crates pour appliquer ces principes à votre propre
outil métier :

- **Une donnée sensible ne doit pas pouvoir fuiter par erreur de configuration.**
  La protection se code, elle ne se documente pas seulement.
- **Le bon défaut, c'est le plus sûr.** En cas d'ambiguïté, un outil bien conçu
  sur-protège.
- **Ce qui agit sur vos données doit être traçable.** Un journal infalsifiable
  transforme « je crois que » en « je peux prouver que ».

C'est précisément ce type de garantie que j'intègre dans les outils que je
construis. Le même souci de fiabilité structurelle se retrouve côté automatisation,
dans nos articles sur [l'automatisation](@/blog/automatisation/_index.md).

## FAQ

### Une option de configuration ne suffit-elle pas pour la confidentialité ?

Non, parce qu'une option peut être mal réglée, oubliée, ou contournée par un chemin
de code imprévu. Inscrire la règle dans le système de types et la vérifier avant
toute action la rend non contournable : le compilateur et le contrôle d'exécution
font le travail à la place de la vigilance humaine.

### Que se passe-t-il si le niveau de confidentialité est corrompu en base ?

Le décodage retombe sur `Confidential`, le niveau le plus restrictif. Une donnée
illisible est donc traitée comme sensible, jamais l'inverse. C'est le principe du
*fail closed*.

### Le cloud est-il complètement interdit ?

Non : les données `Public` peuvent être routées vers le cloud quand la qualité le
justifie. C'est tout l'intérêt d'un modèle à trois niveaux plutôt que d'un binaire
« tout local ou tout cloud ». Seul le niveau `Confidential` est verrouillé en local.

### Comment vérifier après coup qu'une donnée sensible n'a pas fuité ?

Via le journal d'audit chaîné par SHA-256, où le niveau de confidentialité de
chaque action est enregistré. Toute altération du journal casse la chaîne de
hachage et devient détectable.

### Peut-on appliquer ça à un outil d'entreprise plus simple ?

Oui. Les principes (protection par le code, défaut le plus sûr, garde-fou avant
l'action, audit infalsifiable) ne dépendent pas de la taille du projet.
[Décrivez votre besoin](https://ferrix.fr/#contact) et on regarde ensemble ce qui
est pertinent pour votre cas.

## Pour aller plus loin

Cet article fait partie d'une série sur la construction d'Aliénor. Retournez à la
[présentation de la série](@/blog/ia-personnelle/_index.md) pour la vue d'ensemble,
ou lisez [pourquoi et comment j'ai construit cette IA](@/blog/ia-personnelle/ia-personnelle-rust-pourquoi-comment.md).
