## 1) Vue d’ensemble

Un moteur de recommandation cherche à classer des items (ici, produits) pour un contexte donné (page produit, liste, panier, e-mail…), selon un **score de pertinence**. Trois familles simples et complémentaires :

* **Popularité** : recommander ce qui “marche” globalement (vu/acheté/liké).
* **Basé contenu** : recommander des items **similaires** à celui consulté (ou au profil textuel de l’utilisateur) en s’appuyant sur le **texte** (TF-IDF) ou des **vecteurs sémantiques** (embeddings).
* **Collaboratif** : recommander selon les **comportements** des utilisateurs (ceux qui ont aimé A ont aussi aimé B).
  L’évaluation se fait hors-ligne (datasets de tests) et en ligne (A/B), avec des métriques comme **Precision\@K** et **Recall\@K**.

---

## 2) Recommandations par popularité

### 2.1 Données et signaux

* **Vues** (page\_view), **ajouts au panier**, **achats**, **notes** ou **clics** sur recommandations.
* Choisir des **poids** par type d’événement (ex. achat > ajout panier > vue).

### 2.2 Calcul de score (exemples)

1. **Fréquence brute pondérée**
   $\text{score}_i = w_{\text{buy}}\cdot \#\text{achats}_i + w_{\text{cart}}\cdot \#\text{paniers}_i + w_{\text{view}}\cdot \#\text{vues}_i$
2. **Tendance temporelle (décroissance)**
   Appliquer un **time-decay** (demi-vie $h$) :
   $\text{score}_i = \sum_{e \in E_i} w_e \cdot e^{-\lambda \cdot \Delta t_e}$  avec $\lambda = \ln(2)/h$
3. **Lissage bayésien** pour notes moyennes (éviter qu’un produit avec 1 note 5★ domine)
   $\bar{r}_i^{\text{bayes}} = \frac{\mu\cdot m + \sum r_{i}}{m + n_i}$ (où $\mu$=moyenne globale, $m$=poids de l’a-priori, $n_i$=nb de notes)

### 2.3 Atouts / limites

* **Atouts** : robuste, simple, **bon “fallback”** (démarrage à froid).
* **Limites** : **biais de popularité** (toujours les mêmes produits), **faible personnalisation**.

### 2.4 Bonnes pratiques

* **Contraintes métier** (is\_active, stock>0, diversité de catégories).
* **Décroissance temporelle** pour suivre les tendances.
* **Désamorcer le biais** : quotas de diversité, plafond par catégorie, exploration contrôlée.

---

## 3) Recommandations basées contenu

### 3.1 Principe

Représenter chaque produit par un **vecteur de caractéristiques** dérivé de ses **textes** (titre, description, attributs) puis recommander les items **les plus similaires** (p. ex. cosinus).

### 3.2 Variante A : TF-IDF (lexicale)

1. **Prétraitement** : normalisation (casse, ponctuation), éventuellement lemmatisation ; concaténer *titre + catégorie + attributs + description* avec **pondérations** (titre plus lourd).
2. **TF-IDF**

   * $\text{tf}(t,i) = \frac{\#(t \text{ dans } i)}{\sum_{t'} \#(t' \text{ dans } i)}$
   * $\text{idf}(t) = \log \frac{N}{1 + |\{ i: t \in i \}|}$
   * $\text{tfidf}(t,i) = \text{tf}(t,i)\cdot \text{idf}(t)$
3. **Similarité** : **cosinus** entre vecteurs (valeurs dans \[0,1]).
4. **Hyper-paramètres** : n-grammes (1-2), vocabulaire max, min\_df, normalisation L2.

**Atouts** : rapide, interprétable, local, peu coûteux.
**Limites** : ne capture pas bien synonymes/paraphrases ; sensible au vocabulaire.

### 3.3 Variante B : Embeddings (sémantique)

1. **Encodage** : modèle d’**embeddings** (ex. *sentence-transformers*, multilingue si besoin) → vecteur dense (p. ex. 384–1024 dims) pour chaque produit.
2. **Similarité** : cosinus (souvent avec vecteurs **normalisés L2**).
3. **Indexation** : pour > quelques milliers d’items, index **ANN** (FAISS/Chroma) pour requêtes **Top-K** rapides.
4. **Mise à jour** : recalcul embeddings sur *create/update*, job périodique pour la fraîcheur.
5. **Évolution** : possibilité d’**adapter** (fine-tuning) si corpus spécifique.

**Atouts** : robustesse sémantique (synonymes, reformulations), multi-lingue, meilleur rappel.
**Limites** : coût de calcul (offline/online), gestion de versions (modèle/index), **explicabilité** plus faible.

### 3.4 Filtrage & ré-ordonnancement

* **Filtres** : is\_active, stock, catégorie, fourchette de prix, conformité.
* **Diversité** : **MMR (Maximal Marginal Relevance)**
  $\text{MMR} = \arg\max_{d \in C\setminus S}\ \big[\lambda \cdot \text{sim}(q,d) - (1-\lambda)\cdot \max_{d'\in S}\text{sim}(d,d')\big]$
  ($S$=résultats déjà choisis, $C$=candidats).
* **Règles métier** : forcer X % hors catégorie courante, bannir marques en rupture, etc.

### 3.5 Cas d’usage

* **Page produit** : « Produits similaires ».
* **Liste** : « Vous pourriez aussi aimer ».
* **Panier** : compléments (proches mais non redondants).

---

## 4) Recommandations collaboratives (aperçu)

### 4.1 Idée

Utiliser la **matrice interactions** $R_{u,i}$ (clics, achats, notes) :

* **User-based** : utilisateurs similaires à $u$ ont aimé $i$.
* **Item-based** : items proches de $i$ (co-consommation).

### 4.2 Méthodes courantes

1. **k-NN** sur similarités (cosinus, Jaccard pondéré) entre **lignes** (users) ou **colonnes** (items).
   Prédiction (item-based) pour un utilisateur $u$ et item $i$ :
   $\hat{r}_{u,i} = \frac{\sum_{j \in N_k(i)} \text{sim}(i,j)\cdot r_{u,j}}{\sum_{j \in N_k(i)} |\text{sim}(i,j)|}$
2. **Facteur latent** (ALS pour feedback implicite, MF) : décompose $R \approx P Q^\top$ (utilisateur et item dans un espace latent).

### 4.3 Spécificités “implicites”

* Convertir signaux (vue/ajout/achat) en **poids** et/ou **confiances** (ex. Implicit-ALS).
* **Sparsité** élevée → choisir régularisation et **échantillonnage négatif**.

### 4.4 Atouts / limites

* **Atouts** : capte des relations **non textuelles**, parfois très pertinentes.
* **Limites** : **froid** (nouvel item ou nouvel utilisateur), **données nécessaires** (volume), risques de **bulles de filtres** et biais de popularité.
* **Pratique** : parfait **en complément** du contenu ; en début de vie, se limiter à contenu+popularité.

---

## 5) Métriques d’évaluation (hors-ligne) : Precision / Recall

### 5.1 Protocoles de découpe

* **Leave-one-out** par utilisateur : cacher 1 interaction récente, recommander, vérifier si l’item caché est dans le Top-K.
* **Découpe temporelle** : train jusqu’à $T$, test après $T$ (réaliste pour e-commerce).

### 5.2 Définitions

* **Hit\@K** : 1 si au moins un item pertinent est dans le Top-K, 0 sinon.
* **Precision\@K** : proportion d’items recommandés qui sont **pertinents**.
  $\text{P@K}(u) = \frac{|\text{TopK}(u) \cap \text{Rel}(u)|}{K}$
* **Recall\@K** : proportion d’items pertinents effectivement **retrouvés**.
  $\text{R@K}(u) = \frac{|\text{TopK}(u) \cap \text{Rel}(u)|}{|\text{Rel}(u)|}$
* **Moyennes** : macro-moyenne sur les utilisateurs test.
* **Autres utiles** : **MAP**, **NDCG** (prend en compte la **position**), **coverage** (part des items jamais recommandés), **diversity/novelty**.

### 5.3 Interprétation

* **P\@K** élevé : peu d’erreurs en tête de liste.
* **R\@K** élevé : bon taux de récupération des pertinents.
* **Équilibre** : choisir $K$ selon l’UI (p. ex. 6, 10, 20) ; viser **P\@K** élevé en haut de page, **R\@K** pour recherche.

---

## 6) Hybridation (combiner popularité, contenu, collaboratif)

### 6.1 Fusion de scores

* **Pondération linéaire** :
  $\text{score} = \alpha \cdot \text{score}_{\text{contenu}} + \beta \cdot \text{score}_{\text{pop}} + \gamma \cdot \text{score}_{\text{collab}}$
  avec $\alpha + \beta + \gamma = 1$.
* **Règles de fallback** : si collaboratif indisponible (froid), utiliser contenu + un **peu** de popularité.
* **Ré-ordonnancement** : appliquer **MMR** en post-traitement pour la diversité.

### 6.2 Contraintes métier

* **Filtres** d’éligibilité (disponible, actif, marge, conformité).
* **Quotas** par marque/catégorie pour éviter la sur-exposition.

---

## 7) Mise en œuvre pratique (sans code)

### 7.1 Pipeline minimal “contenu”

1. Construire un **corpus** (titre+catégorie+attributs+description) avec pondérations.
2. Calculer **TF-IDF** **ou** **embeddings** (puis normaliser).
3. Stocker vecteurs + **index ANN** si besoin.
4. À la requête : récupérer **Top-K** par cosinus → **filtres** → **MMR** → livrer.
5. **Cache Redis** sur résultats chauds et **invalidation** sur update produit.

### 7.2 Popularité

1. Maintenir des **compteurs** pondérés et/ou **décroissants** dans le temps.
2. Générer périodiquement un **Top-N** par catégorie + global (avec diversité).
3. Exposer ces listes comme **fallback**.

### 7.3 Collaboratif (aperçu)

1. Construire une **matrice interactions** (implicite).
2. Tester un **item-based k-NN** simple pour commencer (co-vision/co-achat).
3. Évaluer sur **Hit\@K / P\@K / R\@K**, comparer au contenu.

### 7.4 Évaluation & suivi

* Définir **K** (ex. 10) et objectifs (ex. P\@10 ≥ 0,6 sur corpus démo).
* Reporter résultats, tracer latence P95, coverage, diversité.

---

## 8) RGPD, conformité et équité

* **Aucune donnée personnelle** dans l’index vectoriel.
* **Transparence** : expliquer le principe (« similaires par caractéristiques »).
* **Droit d’opposition** au profilage si personnalisation individuelle est activée.
* **Biais** : surveiller le **biais de popularité** et la **sous-représentation** de certaines catégories ; ajouter quotas/diversité.

---

### Résumé opérationnel

* Démarrer avec **contenu (TF-IDF)** + **popularité** (fallback).
* Passer à **embeddings** pour une meilleure sémantique (index ANN si nécessaire).
* Ajouter un **collaboratif** dès que le volume d’interactions est suffisant.
* **Évaluer** systématiquement (P\@K, R\@K, NDCG), imposer **filtres** + **diversité (MMR)**, et **surveiller** la latence et le biais.
