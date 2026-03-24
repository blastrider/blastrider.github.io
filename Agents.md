# Documentation Projet Zola - Site Perso (Ferrix)

Ce document est destiné à un assistant IA (Claude) pour comprendre rapidement et intégralement l'architecture et les spécificités du projet.

## 1. Vue d'ensemble et Architecture Globale

Ce projet est un site web statique généré par **Zola**, un générateur de site statique (SSG) extrêmement rapide, écrit en **Rust**.
L'architecture suit les principes "Jamstack" poussés à l'extrême : simplicité, performance, et sécurité maximale.

- **Frontend** : Génération statique complète. Pas de framework lourd (ni React, ni Vue). Seulement du HTML (via les templates Tera), du CSS pur servi depuis `static/style.css`, et quelques lignes de Vanilla JS pour l'interactivité basique (comme l'IntersectionObserver pour les animations).
- **Backend / Serveur** : Aucun backend applicatif dynamique n'est nécessaire ni souhaité. Le site est servi comme un ensemble de fichiers statiques simples (Hébergement basique de type Nginx, Caddy, Vercel ou Github Pages).
- **Formulaire de contact** : Géré par un service externe (Formspree, point d'entrée défini dans `config.toml` via `formspree_endpoint`) pour éviter la gestion d'un backend et maintenir l'hermétisme du front.

## 2. L'empreinte Rust & Sécurité (Bonnes Pratiques)

Même si le site produit in fine du HTML, il tire parti de la robustesse de l'écosystème Rust via Zola :
- **Sécurité "by design"** : En tant que site statique, la surface d'attaque est quasiment nulle. Il n'y a pas de base de données à injecter (pas d'injection SQL), ni de point d'entrée d'API dynamique exposé aux attaques DDoS classiques sur backend.
- **Performance** : La compilation Zola est quasi-instantanée, permettant des itérations très rapides. Les pages servies étant préconstruites (statiques), le TTFB (Time To First Byte) est minimal et indépendant de la charge serveur.
- **Fiabilité absolue** : Le SSG étant en Rust, il garantit l'absence de crashs inattendus liés à la mémoire lors de la génération. Le développeur (Ferrix) étant lui-même sysadmin/dev Rust, l'idéologie est la stabilité, l'absence de dette technique et la sobriété numérique.
- **Zero-Dependency** : Zola est distribué sous forme de binaire unique autonome. Pas besoin de polluer l'environnement CI/CD avec `node_modules` ou `requirements.txt`.

## 3. Arborescence du Projet & Rôle des Dossiers

Voici l'arborescence actuelle du projet avec le rôle détaillé de chaque élément :

```text
.
├── content
│   └── _index.md               # Le noyau dur du projet : Contenu principal de la page d'accueil. Utilise le front-matter (format TOML) sous `[extra]` pour structurer les textes et données (héro, offres, cibles, crédibilité) qui seront injectés dynamiquement dans les templates.
├── sass                        # Dossier prévu pour le préprocesseur Sass (désactivé dans la config actuelle, `compile_sass = false`), prouvant l'inclinaison vers le CSS Vanilla.
├── static
│   ├── apple-touch-icon.png    # Icônes pour le référencement iOS et l'UX mobile.
│   ├── favicon-192.png         # Favicon haute résolution.
│   ├── favicon.ico             # Favicon standard.
│   ├── logo_FERRIX.svg         # Actifs visuels vectoriels.
│   └── style.css               # Fichier CSS Vanilla principal de 100% des styles du site.
├── templates
│   ├── base.html               # Le squelette HTML parent. Contient le `<head>`, la navigation (`<header>`), le footer, le bloc principal `{% block content %}`, le JS inline, la logique SEO balises Meta, Open Graph et JSON-LD Schema.org.
│   └── index.html              # (Déduit de `_index.md`) Le template spécifique pour la page d'accueil, étendant `base.html` (« extends "base.html" ») et bouclant sur le contenu de `page.extra`.
├── themes                      # Pour brancher des thèmes Zola communautaires (actuellement vide car design custom).
├── config.toml                 # Fichier de configuration maître Zola (variables globales, paramètres SEO, Formspree).
├── zola.toml                   # Alternative/Alias reconnu par Zola pour la config.
├── logo_FERRIX_engrenage_only.svg
└── transparent_logo.png
```

## 4. Documentation Intégrée de la CLI Zola

Aide-mémoire des commandes de base pour manipuler le projet :

```text
zola -h                                  
A fast static site generator with everything built-in

Usage: zola [OPTIONS] <COMMAND>

Commands:
  init        Create a new Zola project
  build       Deletes the output directory if there is one and builds the site
  serve       Serve the site. Rebuild and reload on change automatically
  check       Try to build the project without rendering it. Checks links
  completion  Generate shell completion
  help        Print this message or the help of the given subcommand(s)

Options:
  -r, --root <ROOT>      Directory to use as root of project [default: .]
  -c, --config <CONFIG>  Path to a config file other than zola.toml or config.toml in the root of project
  -h, --help             Print help
  -V, --version          Print version

License: EUPL-1.2 <https://eupl.eu>
```

**Workflow de développement courant :**
1. Utilise `zola serve` pour lancer un serveur web local ultra-rapide avec hot-reload.
2. Utilise `zola check` (en CI/CD) avant de valider pour tester la syntaxe et la validité de toutes les routes/liens/assets du site.
3. Utilise `zola build` pour l'étape de génération en production (crée un répertoire `./public` par défaut).

## 5. Points Cruciaux Additionnels (Ce qu'il ne faut pas oublier !)

1. **Moteur de Templates `Tera`** :
   Zola utilise le moteur **Tera** (fortement inspiré par Jinja2 et Django templates).
   - Variables : `{{ variable }}`
   - Logiques/Boucles : `{% if condition %}`, `{% for item in collection %}`
   - Attention : l'accès aux variables du front-matter Markdown définies sous l'en-tête TOML `+++` ! Le bloc `[extra.hero]` dans `content/_index.md` est appelé dans les templates via `page.extra.hero`.

2. **Philosophie "Data-Driven" / "Headless" via Markdown** :
   Le fichier `content/_index.md` agit comme un véritable **CMS sans base de données**. Son Front-Matter TOML contient des paires clés-valeurs, listes et tableaux modélisant les données des blocs de la page d'accueil (offres, cibles, crédibilité). L'objectif est une séparation parfaite : le fond éditorial dans le TOML de `_index.md`, la forme itérative dans le template `index.html`.

3. **SEO & Open Graph très strict** :
   Le fichier `templates/base.html` n'est pas qu'un layout. Il sert de fondation vitale pour le SEO. Il intègre le standard Schema.org en format balisage JSON-LD pour définir la page en tant que "ProfessionalService". Toute refonte du frontend ne doit en aucun cas briser la mécanique SEO et les balises Open Graph (`og:title`, etc...) ni le manifeste d'identité Schema !

4. **Styles minimaux (Vanilla & Sobriété)** :
   Ce site cible un public tech (CTO) et PME/TPE en prônant les processus incassables. Le code source est le reflet de cette expertise : pas de dépendance NPM lourde, pas de Framework JS ou CSS lourd, très frugal. Le design est en Vanilla CSS et quelques lignes de Vanilla JS dans le `base.html` pour implémenter un `IntersectionObserver` minimaliste pour les animations (classes `.reveal`). Toute nouvelle feature devra suivre cette contrainte "No Bulshit / No Bloat".

5. **Mobile First (Règle absolue)** :
   Tout CSS doit être écrit pour mobile en premier, desktop ensuite via `@media (min-width: ...)`. Implications concrètes :
   - Layouts en colonne unique par défaut, grilles CSS activées à partir des breakpoints définis (`760px`, `980px`).
   - Taille de police, paddings et espacements dimensionnés pour le pouce, pas pour la souris.
   - Aucune interaction `hover`-only sans fallback tactile.
   - Touch targets ≥ 44×44px (boutons, liens de navigation).
   - Le menu navigation utilise `<details>`/`<summary>` natif (zéro JS) — ne pas remplacer par un pattern JS.

## 6. Stratégie SEO & Orientation SaaS (Adapté aux moteurs et aux IA de Scraping)

L'objectif central est de faire évoluer ce socle vers une plateforme d'Acquisition SaaS optimisée à la fois pour Google (SEO traditionnel) et pour les robots/LLM d'intelligence artificielle (SGE, Perplexity, OpenAI Bot, etc.).
Selon les préceptes exhaustifs du livre fourni (*The Art of SEO*), voici les piliers imposés à l'assistant IA pour le développement du projet :

1. **Lead Generation & Direct Marketing (SaaS)** :
   - La structure du site doit privilégier la capture d'intentions. L'UX et l'UI doivent converger de manière chirurgicale vers des entonnoirs clairs (Call-To-Actions ciblés, Capture de leads, Formulaires).
   - Séparation de l'intention des requêtes (Transactionnel vs. Informationnel) : les pages d'offres/SaaS répondent aux requêtes transactionnelles, tandis que d'autres pages (ou futurs articles) se chargeront stratégiquement de la "long tail" pour alimenter le haut du tunnel de conversion.

2. **Accessibilité Parfaite pour les Crawlers (Scraping IA & Spiders traditionnels)** :
   - *Indexable Content & Spiderable Structure* : Le HTML rendu par les templates Tera doit être sémantique (balises `<article>`, `<section>`, `<nav>`, `<aside>`) et l'information doit être lisible instantanément dans le code source brut, car les scrappers des modèles d'IA (LLMs) peinent fortement avec le JS client-side lourd. L'usage exclusif d'un SSG pur (Zola) est l'approche optimale.
   - *XML Sitemaps* : Utilisation assidue du sitemap statique généré (`generate_sitemap = true`) et structure d'URLs claire (répertoires logiques) dictés par les bonnes pratiques on-page de l'industrie pour orienter les spiders efficacement.

3. **Semantic Search, Entités & Knowledge Graph (Compréhension par les Intelligences Artificielles)** :
   - Assurer un balisage **Schema.org** rigoureux via des blocs JSON-LD injectés côté serveur (dans le `<head>`). Pour le contexte SaaS, l'utilisation de schémas spécifiques (`SoftwareApplication`, `Organization`, `FAQPage`, etc.) est obligatoire.
   - Construire des entités sémantiques robustes au lieu du simple "Keyword Stuffing". Le contenu textuel doit fournir une vérité factuelle forte et répondre directement aux frictions du marché pour avoir la chance d'être extrait et cité comme source de référence (Réponse Directe / Feature Snippet) par les intelligences génératives type Gemini, ChatGPT ou Claude.
