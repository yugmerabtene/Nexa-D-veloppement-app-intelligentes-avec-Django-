# Jour 3 — Intelligence : recommandations, recherche sémantique & assistant RAG

### 1) Compétences visées en fin de journée

* Concevoir et intégrer des **mécanismes de recommandation** (popularité, contenu, amorce collaborative) dans une application Django existante.
* Mettre en place une **recherche sémantique** locale (embeddings + index vectoriel) complémentaire à la recherche lexicale.
* Construire un **assistant RAG** (Retrieval-Augmented Generation) responsable : ingestion, chunking, retrieval, formulation de réponse, **traçabilité** et **garde-fous**.
* Définir des **métriques d’évaluation** hors-ligne et des **indicateurs en ligne**, et organiser l’itération.
* Aborder la **performance** (latence, cache, invalidation), la **reproductibilité** (versioning des artefacts) et la **conformité** (RGPD, sécurité).

### 2) Positionnement fonctionnel dans SmartMarket

* **Catalogue** comme source principale : titres, descriptions, catégories, attributs.
* **Recommandations** pour : pages produit (produits similaires), liste (vous pourriez aussi aimer), panier (complémentaires simples), e-mail (réactivation).
* **Recherche** pour : découverte (q → produits pertinents), tolérance aux fautes, synonymes, sémantique.
* **Assistant** pour : aide à l’achat (questions produit), politiques (retours, livraison), comparatifs, guidance du choix.

### 3) Données, signal et gouvernance

* **Données disponibles Jour-03** : métadonnées produit (obligatoire), historiques simulés (optionnel), FAQ/politiques (assistant).
* **Qualité des données** : nettoyage minimal ; cohérence des slugs, absence d’informations personnelles dans les jeux d’entraînement.
* **Gouvernance** : définir propriétaires, cycle de vie, et périmètres autorisés (registre des traitements + base légale).

### 4) Recommandations : panorama des approches

* **Popularité** (fréquences de vues ou ventes) : robuste au démarrage, peu personnalisée, utile en fallback.
* **Basée contenu** (Content-Based) : vectoriser chaque produit (TF-IDF/embeddings), recommander par **similarité** au produit consulté.
* **Collaboratif** (aperçu) : nécessite interactions utilisateur-produit (notes, clics, achats) ; non abordé en profondeur Jour-03, mais évoqué pour la **feuille de route**.
* **Hybridation** : règles de combinaison (pondération, intercalage, contraintes de diversité).
* **Contraintes métier** : disponibilité (`is_active`, stock > 0), prix, catégories exclues, diversité (éviter doublons et « boucle d’échos »).

### 5) Représentation des textes produit

* **Lexicale** (TF-IDF) : simple, interprétable, locale, efficace pour des corpus moyens.
* **Sémantique** (embeddings) : capture synonymie/paraphrase, plus robuste aux formulations variées.
* **Prétraitements** : normalisation (casse, ponctuation), éventuellement lemmatisation ; gestion des chiffres, unités, entités.
* **Champ de texte** : concaténer *titre + catégorie + mots-clés + attributs* avec pondérations (ex. titre plus lourd que description).

### 6) Similarité & voisinage

* **Mesures** : cosinus (recommandé pour TF-IDF/embeddings), euclidienne (moins pertinente ici).
* **Top-K** : nombre de voisins à retourner (ex. K = 10), avec **filtrage** (même catégorie, actif, prix raisonnablement proche).
* **Diversité** : **Maximal Marginal Relevance (MMR)** simple (pondérer similarité et nouveauté) pour éviter des résultats trop proches entre eux.

### 7) Recherche sémantique locale

* **Index lexicaux** : utiles mais limités (strict matching).
* **Index vectoriels** : FAISS/Chroma pour *approximate nearest neighbors* (ANN) ; gains de latence pour > quelques milliers d’items.
* **Pipeline de requête** : `texte utilisateur → embedding requête → top-K vecteurs → réconciliation vers produits → ranking final`.
* **Tolérance aux fautes** : soit via une étape lexicale (fuzzy) en amont, soit via la robustesse sémantique des embeddings.

### 8) Assistant RAG (Retrieval-Augmented Generation)

* **Ingestion** : sources (FAQ, politiques, fiches techniques), extraction du texte, **chunking** (ex. 300–500 tokens), métadonnées (type document, version, date).
* **Indexation** : embeddings par chunk + index vectoriel dédié **séparé du catalogue**.
* **Retrieval** : top-K chunks filtrés par **règles** (ex. prioriser documents récents, exclure brouillons).
* **Génération** : LLM choisi (API ou local) ; **réponses ancrées** sur les extraits récupérés ; citations/métadonnées en sortie.
* **Garde-fous** : filtrage d’entrée (prompt injection, données sensibles), **bornes** (ne pas halluciner hors corpus), refus sécurisé si l’information n’existe pas.

### 9) Traçabilité, audit et explicabilité

* **Journal de décision** : requête, top-K sources, scores, version d’index, gabarit de réponse, temps de latence.
* **Explicabilité** : pour recommandations, fournir **raison court format** (« similaire par catégorie et caractéristiques »).
* **Versioning** : numéro de version des poids/embeddings/index ; horodatage d’ingestion.

### 10) Évaluation (hors-ligne)

* **Recommandations** : Precision\@K, Recall\@K, MAP, NDCG ; *offline split* (train/validation), *leave-one-out* si historique.
* **Recherche** : R\@K, MRR, NDCG, tests de pertinence à partir de paires (requête, produit pertinent).
* **Assistant** : *answerability* (capacité à répondre depuis le corpus), taux de refus justifiés, exactitude factuelle (revue humaine échantillonnée).
* **Seuils cibles** : définir des **objectifs réalistes** (ex. P\@10 ≥ 0,6 sur corpus démo).

### 11) Évaluation (en ligne)

* **Télémétrie** : CTR, sauvegardes au panier depuis blocs de reco, abandons, temps de session.
* **Expérimentations** : A/B tests simples avec buckets stables, période minimale significative, pas de personnalisation sans consentement explicite si profilage.

### 12) Performance et latence

* **Budgets** :

  * Recommandations produit : **P95 ≤ 150 ms** (hors cache, corpus modeste).
  * Recherche sémantique : **P95 ≤ 300 ms** avec ANN.
  * Assistant : **≤ 2 s** en retrieval ; génération soumise au temps du LLM.
* **Cache** : Redis pour résultats (clé composée : version + produit\_id + K + filtres) ; **invalidation** sur `create/update/delete` produit ou ré-index.
* **Warmup** : pré-calcul de vecteurs, top-K statiques pour pages très vues.

### 13) Ingestion, planification et cohérence

* **Pipelines incrémentaux** : recalcul embeddings uniquement pour items modifiés.
* **Planification** : job périodique (ex. quotidien) + **événementiel** (post-save signal) pour maintenir la fraîcheur.
* **Transactions** : aligner commit DB et mise à jour d’index (modèle → file → worker → index → cache).

### 14) Sécurité et RGPD appliqués

* **Minimisation** : pas de données personnelles en index vectoriel.
* **Transparence** : mention claire des traitements dans la documentation utilisateur.
* **Droits** : effacement/anonymisation : s’applique aux historiques de recommandation **si** conservés par utilisateur (non requis Jour-03).
* **Sécurité** : contrôle d’accès sur endpoints d’administration d’index ; chiffrement au repos si stockage externe.

### 15) Qualité, reproductibilité et MLOps léger

* **Déterminisme** : fixer les graines (random state) lorsque pertinent pour la reproductibilité des tests.
* **Artefacts** : dossier dédié (ex. `var/ml/`) avec **manifest** (versions modèle/index).
* **Surveillance** : métriques simples (latence, taux d’erreur, volumétrie), alertes basiques.

### 16) Intégration API (contrats)

* **Recommandations** : `GET /products/{id|slug}/recommendations/?k=10` → liste d’objets {id, score, raison, …}.
* **Recherche** : `GET /search?q=...&k=20` → liste d’objets {id, score, *highlights* éventuels}.
* **Assistant** : `POST /assistant/ask` avec {question, user\_opt\_in?, contexte?} → {answer, sources\[], trace\_id}.
* **Erreurs** : 400 (paramètres), 429 (throttling), 503 (index indisponible).
* **Versionnement** : `/api/v1/…` cohérent avec les jours précédents.

---

# TP-03 — Recommandations, recherche sémantique & assistant (noté /20)

## 1) Objectif

Ajouter au projet **SmartMarket** :

1. un module **recommandations** (basé contenu) exposant un endpoint `/products/{id}/recommendations/`,
2. une **recherche sémantique** via embeddings avec endpoint `/search?q=…`,
3. un **assistant d’aide à l’achat** `/assistant/ask` fondé sur un RAG local,
   avec **cache Redis**, **invalidation** sur mises à jour du catalogue, **tests** (hors-ligne + API) et **traçabilité** (journal minimal).

## 2) Livrables attendus

* **Archive** : `prenom_nom_lab-03.zip` contenant :

  * Code (`src/`), configuration, fichiers d’artefacts (manifest d’index/vectorisations), scripts de (ré)indexation.
  * **README.md** décrivant : prérequis, commande(s) de construction d’index, endpoints, paramètres, limites, budget de latence visé.
  * **Captures** (3) :

    1. Exécution d’un **export d’index** (ou manifest/versionning affiché),
    2. Appel réussi à `/products/{id}/recommendations/` avec les **raisons** visibles,
    3. Appel à `/assistant/ask` montrant la **réponse** et la **liste des sources**.
  * **Optionnel** : petit **jeu de tests** de pertinence (fichier JSON de paires requête→produits attendus).

## 3) Périmètre fonctionnel (check-list)

1. **Module ML**

   * Dossier dédié (ex. `ml/`) structurant : prétraitements, vectorisation (TF-IDF et/ou embeddings), similarités, ranking, manifest versionné.
   * **Recommandations** basées contenu : calcul de similarité produit→produits, avec filtrage des inactifs et contrainte de diversité minimale.
2. **Recherche sémantique**

   * Construction d’un index vectoriel local (ANN recommandé si corpus > 5 000 items) ; endpoint `/search` avec `q`, `k`.
   * Réponse triée par similarité, avec possibilité d’un champ **raison** (ex. « correspondance sur caractéristiques »).
3. **Assistant RAG**

   * Ingestion de **documents internes** (FAQ, politiques, descriptifs techniques) → chunking → embeddings → index séparé.
   * Endpoint `/assistant/ask` retournant réponse + **sources** (identifiants/métadonnées) + **trace\_id**.
   * **Garde-fous** : si information introuvable, la réponse doit l’indiquer explicitement (pas d’hallucination).
4. **Cache & invalidation**

   * Cache Redis sur résultats de reco et recherche (clé incluant version d’index).
   * Invalidation automatique lors d’une mise à jour du catalogue (création, modification, suppression).
5. **API & sécurité**

   * Endpoints sous `/api/v1/…` ; throttling raisonnable sur `/assistant/ask`.
   * Journalisation minimale : temps, paramètres résumés, top-K scores, version d’index, **sans** données personnelles.
6. **Tests & évaluation**

   * Tests unitaires/fonctionnels :

     * pipeline de similarité (ex. produit proche de lui-même = score max),
     * recherche (requête attendue renvoie item cible dans le top-K),
     * assistant (retourne sources cohérentes ou refus lorsqu’aucune source pertinente n’est trouvée).
   * Tests API (statuts, schémas de réponse, throttling).
   * **Cible** indicative : P\@10 ≥ 0,6 sur le petit corpus démo fourni/construit (si protocole d’évaluation inclus).

## 4) Critères d’évaluation (barème /20)

1. **Recommandations basées contenu** — **5 pts**

   * (2) Pipeline de vectorisation + similarité correct et stable
   * (1) Filtrages métier (actif, stock) et diversité minimale
   * (1) Endpoint `/products/{id}/recommendations/` conforme (paramètres, top-K, raisons)
   * (1) Latence raisonnable ou cache efficace
2. **Recherche sémantique** — **5 pts**

   * (2) Index vectoriel fonctionnel (construction, chargement, versionning)
   * (1) Endpoint `/search` avec paramètres `q`, `k`, réponses triées
   * (1) Résultats pertinents sur le corpus de démonstration
   * (1) Cache et invalidation documentés/opérationnels
3. **Assistant RAG** — **5 pts**

   * (2) Ingestion + chunking + embeddings + retrieval dédiés (corpus distinct du catalogue)
   * (1) Réponse adossée à des **sources** (citations/métadonnées)
   * (1) Garde-fous (refus si hors corpus, messages sobres)
   * (1) Journalisation (trace\_id, top-K, version d’index)
4. **Qualité, tests et documentation** — **5 pts**

   * (1) README clair (mise en route, limitations, budgets de latence)
   * (1) Lint/format OK (black/ruff)
   * (1) Tests unitaires/fonctionnels pertinents et verts
   * (1) Tests API (statuts, throttling, schémas)
   * (1) Manifest/versionning des artefacts et scripts de (ré)indexation

**Total : 20 pts**
**Bonus (jusqu’à +2)**

* (+1) Diversification MMR implémentée et mesurée (avant/après).
* (+1) Indicateurs **en ligne** (CTR recommandations, temps de réponse P95) exportés dans un tableau simple.

## 5) Procédure de recette attendue

1. Construire les artefacts : vectorisations, index recherche, index RAG (manifest avec versions/horodatage).
2. Vérifier les endpoints :

   * `/products/{id}/recommendations/?k=10` → résultats filtrés, justifiés, stables.
   * `/search?q=...&k=20` → produits pertinents ; vérifier cohérence sur plusieurs requêtes.
   * `/assistant/ask` → réponse documentée par **sources** ou refus explicite si hors corpus.
3. Vérifier le **cache** : second appel plus rapide ; invalidation après mise à jour d’un produit.
4. Lancer la batterie de **tests** et prendre les **captures** demandées.
5. Préparer l’archive `prenom_nom_lab-03.zip`.
