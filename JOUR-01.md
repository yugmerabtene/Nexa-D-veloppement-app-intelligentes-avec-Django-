# Jour 1 — Démarrage Django + ORM & Admin

## Cours (théorie approfondie, sans code)

### 1) Compétences visées en fin de journée

* Structurer une application Django selon le modèle MVT avec un découpage par domaines fonctionnels (apps).
* Paramétrer des environnements distincts (développement/production), isoler les secrets et respecter les principes 12-Factor.
* Concevoir un modèle de données cohérent (catalogue) avec contraintes et index au niveau de l’ORM et de la base.
* Utiliser l’ORM de manière performante (gestion des N+1, `select_related`/`prefetch_related`, pagination).
* Exploiter l’interface d’administration pour un usage interne efficace (filtres, recherche, actions, inlines).
* Mettre en place une hygiène de code minimale (formatage, linting, tests unitaires ciblés) et des données de démonstration.

### 2) Architecture Django (MVT) et découpage en applications

* **Model** : définition du schéma, des règles d’intégrité et de l’API objet ; pivot de la cohérence métier.
* **View** : orchestration de la logique ; sélection et préparation des jeux de données ; contrôle des accès.
* **Template** : présentation ; logique minimale ; pas d’accès direct aux services lourds.
* **Chaîne de traitement** : URL → résolveur → vue → ORM/services → contexte → template → réponse.
* **Applications** : modules fonctionnels autonomes (`catalog`, `accounts`, `orders`) : modèles, admin, vues, URLs, tests. Minimiser les dépendances croisées ; privilégier des services internes dédiés lorsque nécessaire.
* **WSGI/ASGI** : privilégier WSGI (sync) pour le Jour-01 ; aborder ASGI/Channels au Jour-04.

### 3) Paramétrage par environnement et gestion des secrets

* **Séparation des paramètres** : `base.py` (commun), `dev.py` (outils de debug), `prod.py` (sécurité et performances).
* **Sélection du jeu de paramètres** via `DJANGO_SETTINGS_MODULE`.
* **Secrets et configuration** : variables d’environnement, `.env` non suivi en contrôle de version, `.env.example` partagé.
* **Points d’attention** : `SECRET_KEY` robuste, `DEBUG=False` en production, `ALLOWED_HOSTS` explicite, fuseaux/locale corrects, en production activer les paramètres `SECURE_*`.

### 4) Organisation du dépôt et principes 12-Factor

* **Arborescence claire** centrée sur `src/` pour éviter les ambiguïtés d’import.
* **Fichiers transverses** : `Makefile` (raccourcis), configuration unifiée dans `pyproject.toml` (formatage/lint/tests), `docker-compose.yml` pour les services de développement, `README.md` d’initialisation.
* **Discipline de versionnement** : branches de fonctionnalités, messages de commit explicites ; migrations fréquentes et maîtrisées.

### 5) Base de données et contrat d’intégrité

* **PostgreSQL** aligné dev/prod pour limiter les écarts de comportement.
* **Types adaptés** : `Decimal` pour les montants ; proscrire `float` pour la monnaie.
* **Contraintes au niveau base** (unicité, `CHECK`, clefs étrangères avec stratégie `PROTECT` lorsque la suppression cascade serait risquée).
* **Indexation ciblée** (slug, combinaisons réellement filtrées), en gardant à l’esprit le coût sur les écritures.
* **Migrations** : distinguer schéma et données ; découper et tester ; prévoir la réversibilité lorsque pertinent.

### 6) Modélisation du catalogue (catégories et produits)

* **Périmètre minimal** : `Category` (nom, slug), `Product` (catégorie, nom, slug, description, prix `Decimal`, stock, actif, timestamps).
* **Invariants** : slug lisible et stable, prix non négatif, désactivation plutôt que suppression pour la traçabilité.
* **Normalisation** : génération contrôlée des slugs ; validations côté modèle en complément des contraintes en base.

### 7) ORM Django : principes d’usage et performance

* **Évaluation paresseuse** des QuerySets : chaîner sans exécuter ; évaluer uniquement au besoin.
* **Éviter le N+1** : `select_related` pour les relations 1-à-1 / FK ; `prefetch_related` pour M2M / relations inverses.
* **Pagination systématique** sur les listes pour maîtriser latence et mémoire.
* **Réduction de charge** : `values()`, `only()`, `defer()` lorsque le besoin est avéré.
* **Opérations atomiques** avec `F()` pour éviter les conditions de concurrence simples ; **transactions** (`atomic`) pour la cohérence multi-écritures.
* **Bonnes pratiques** : `count()` plutôt que `len(qs)` si seul le comptage est requis ; `exists()` pour l’existence.

### 8) Interface d’administration : productivité interne encadrée

* **Usages** : outil interne pour opérations et support ; non destiné à un public externe.
* **Ergonomie** : `list_display` pertinent, `list_filter` utile, `search_fields` sur des colonnes indexées.
* **Actions** : opérations de masse (activer/désactiver) pour gagner en efficience.
* **Performance** : `list_select_related` et surcharge de `get_queryset` pour supprimer les N+1.
* **Sécurité** : URL connue mais protégée ; comptes avec MFA en production ; délégation par groupes et permissions spécifiques.

### 9) Routage et réversibilité des URLs

* **Nommage** systématique (`app_name`, `name="..."`) pour permettre le reverse dans les templates et vues.
* **Converters** (`<slug:slug>`) pour typer les segments d’URL et sécuriser le routage.
* **Cohérence** sur la barre oblique finale (convention unique sur l’ensemble du projet).

### 10) Vues (CBV) : simplicité, lisibilité, contrôles

* **Class-Based Views** (`ListView`, `DetailView`) pour couvrir l’essentiel.
* **`get_queryset()`** comme point central de filtrage/tri/optimisation (dont `select_related`).
* **Gestion des erreurs** : 404 cohérente sur slug manquant ou inactif ; éviter la logique lourde dans les templates.

### 11) Templates : héritage, sûreté et accessibilité

* **Héritage de gabarits** pour factoriser l’ossature (`base.html`, blocs `title`, `content`).
* **Sécurité** : auto-échappement activé par défaut ; usage mesuré de `|safe`.
* **Accessibilité** : titres sémantiques, pagination claire, messages pour états vides ou erreurs.

### 12) Observation et diagnostic en développement

* **Django Debug Toolbar** pour analyser les requêtes SQL, repérer les N+1, mesurer le temps par panneau.
* **Journalisation** : niveaux adaptés au contexte (développement verbeux, production maîtrisée) ; consolidation ultérieure au Jour-04.

### 13) Qualité logicielle minimale et tests

* **Formatage** (black) et **linting** (ruff) pour homogénéiser et prévenir les anti-patterns.
* **Tests** ciblés dès le départ : unicité (slug par catégorie), non-négativité du prix, tests “fumée” sur listes et détails.
* **Automatisation locale** : commandes standardisées via `Makefile` (démarrage, migrations, données de démonstration, tests, formatage, lint).

### 14) Premiers réflexes de sécurité et conformité (RGPD)

* **Secrets** hors dépôt ; `.env` ignoré ; `.env.example` partagé.
* **`DEBUG=False`** en production avec `ALLOWED_HOSTS` défini.
* **Prévention** de l’injection SQL par l’usage exclusif de l’ORM et des requêtes paramétrées.
* **Données de démonstration** neutres (pas d’informations personnelles réelles).
* **Traçabilité** minimale par timestamps ; approfondissements (authentification, permissions, en-têtes de sécurité, CSRF, XSS) au Jour-02.

---

# TP-01 — Bootstrap du projet SmartMarket (énoncé inchangé, noté /20)

**Objectif**
Mettre en place un socle Django propre et productif : modèles `Category` & `Product` avec contraintes et index, administration avancée, pages liste/détail, Postgres via Docker, debug toolbar, qualité (black, ruff), tests de base et données de démonstration.

**Livrables attendus**

* **Archive** : `prenom_nom_lab-01.zip`
  Contient le dépôt (sans `.venv`/`pgdata`) avec :

  * Code (`src/`), `docker-compose.yml`, `Makefile`, `pyproject.toml` (ou `requirements.txt` + `pytest.ini`), `.env.example`, `README.md`.
  * **Captures** :

    1. Admin → liste des produits avec filtres et actions visibles
    2. Debug toolbar affichant les requêtes sur la liste des produits
    3. Page liste des produits et page détail

* **README.md** clair (prérequis, commandes `make`, démarrage, comptes).

**Cahier des charges (check-list)**

1. Projet et application : projet `smartmarket` + app `catalog` déclarée dans `INSTALLED_APPS`.
2. Paramètres : séparation `base.py` / `dev.py` (avec debug toolbar) / `prod.py`.
3. Base de données : Postgres via `docker-compose.yml`; variables dans `.env`.
4. Modèles : `Category(name, slug unique)` ; `Product` (FK category, `name`, `slug`, `price≥0`, `stock`, `is_active`, `created_at`, `updated_at`)

   * Contrainte `UniqueConstraint(category, slug)`
   * `CheckConstraint(price__gte=0)`
   * Index sur `slug` et `(category, is_active)`
5. Administration :

   * `list_display`, `list_filter`, `search_fields`, `prepopulated_fields`, `autocomplete_fields`
   * Actions d’activation/désactivation
   * Optimisation `list_select_related` / surcharge `get_queryset`
6. Vues & URLs :

   * `ProductListView` paginée (12) avec optimisation
   * `ProductDetailView` par `slug`
   * Racine → liste ; `/p/<slug>/` → détail
7. Templates : `product_list.html`, `product_detail.html` opérationnels.
8. Données de démo : fixture ou commande `seed_demo`.
9. Qualité : configuration `black`, `ruff` et vérification `--check`.
10. Tests : au minimum 2 (contrainte d’unicité, vues liste/détail).
11. Debug toolbar : active en développement (`__debug__/`).
12. Makefile : cibles `up`, `migrate`, `seed`, `dev`, `test`, `lint`, `fmt`.

**Barème /20**

1. Structure du dépôt & paramètres (3 pts)

   * (1) Arborescence maîtrisée + `.env.example`
   * (1) `base.py`/`dev.py`/`prod.py` opérationnels
   * (1) Variables d’environnement correctement consommées
2. Base de données & Docker Compose (2 pts)

   * (1) Service Postgres, volume et healthcheck
   * (1) Connexion Django → Postgres validée
3. Modélisation, contraintes, index (4 pts)

   * (1) Champs et validateurs adaptés
   * (1) Unicité `(category, slug)`
   * (1) `CHECK price≥0`
   * (1) Index pertinents (`slug`, `(category,is_active)`)
4. Administration & performances ORM (3 pts)

   * (1) Filtres/recherche/actions/autocomplete
   * (1) `list_select_related`/`get_queryset` optimisés
   * (1) Préremplissage `slug` et ergonomie
5. Vues/URLs/Templates (3 pts)

   * (1) Liste paginée et optimisée
   * (1) Détail par slug
   * (1) Templates fonctionnels
6. Qualité & tests (3 pts)

   * (1) black/ruff configurés et propres
   * (1) Test de contrainte modèle
   * (1) Tests des vues
7. README & Makefile (2 pts)

   * (1) README d’usage clair
   * (1) Makefile utile et fonctionnel

**Total : 20 pts**
**Bonus** (jusqu’à +2) : tri exposé en pagination, ou champ `SKU` unique avec validation et test.
