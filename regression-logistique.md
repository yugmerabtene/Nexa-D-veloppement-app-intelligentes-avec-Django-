# 1. Rappel théorique

## 1.1 TF (Term Frequency)

Le TF mesure l’importance d’un mot **dans un document particulier**.
Formule (version relative) :

$$
TF(t,d) = \frac{\#(t \ \text{dans } d)}{\text{nombre total de mots dans } d}
$$

Exemple :
Doc = `"stock market rises today"`, 4 mots.
Chaque mot apparaît une fois, donc :

$$
TF(stock,d) = TF(market,d) = TF(rises,d) = TF(today,d) = \frac{1}{4} = 0.25
$$

---

## 1.2 IDF (Inverse Document Frequency)

L’IDF mesure l’importance d’un mot **dans le corpus entier** (ensemble des documents).

$$
IDF(t) = \ln \left(\frac{N}{n_t}\right)
$$

* $N$ = nombre total de documents
* $n_t$ = nombre de documents contenant le mot $t$

Exemple (corpus de 3 documents) :

1. `"Stock market rises today"`
2. `"Crypto market crashes today"`
3. `"Investors optimistic about market"`

* $N = 3$
* “stock” apparaît dans 1 document → $IDF = \ln(3/1) \approx 1.099$
* “market” apparaît dans 3 documents → $IDF = \ln(3/3) = 0$
* “today” apparaît dans 2 documents → $IDF = \ln(3/2) \approx 0.405$

---

## 1.3 TF-IDF

On combine les deux :

$$
TF\!-\!IDF(t,d) = TF(t,d) \times IDF(t)
$$

Exemple sur le document 1 `"Stock market rises today"` :

* “stock” : $0.25 \times 1.099 = 0.275$
* “market” : $0.25 \times 0 = 0$
* “rises” : $0.25 \times 1.099 = 0.275$
* “today” : $0.25 \times 0.405 = 0.101$

Vecteur TF-IDF(d1) ≈ \[0.275, 0, 0.275, 0.101]

---

# 2. Régression logistique

## 2.1 Idée

La régression logistique est un modèle qui transforme un vecteur (ici TF-IDF) en **probabilité d’appartenir à une classe**.

* Classe 1 : positif
* Classe 0 : négatif

## 2.2 Formule

1. Calcul d’une combinaison linéaire :

$$
z = w \cdot x + b = \sum_{j=1}^n w_j x_j + b
$$

2. Application de la fonction sigmoïde :

$$
P(y=1|x) = \frac{1}{1 + e^{-z}}
$$

* Si $P \geq 0.5$ → classe 1
* Sinon → classe 0

---

## 2.3 Exemple concret

Document d1 : vecteur TF-IDF = \[0.275, 0, 0.275, 0.101]
Poids appris : $w = [1.5, -0.2, 2.0, 0.5], b = 0$

1. Calcul de z :

$$
z = (1.5)(0.275) + (-0.2)(0) + (2.0)(0.275) + (0.5)(0.101)
$$

$$
z = 0.4125 + 0 + 0.55 + 0.0505 = 1.013
$$

2. Sigmoïde :

$$
P(y=1|x) = \frac{1}{1 + e^{-1.013}} \approx 0.733
$$

3. Décision : $0.733 > 0.5$ → classe positive.

---

# 3. TP en Python

Objectif : Construire un petit classifieur de sentiment basé sur TF-IDF + Régression logistique.

---

## Étape 1 : Import des bibliothèques

```python
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report
```

---

## Étape 2 : Jeu de données simple

```python
docs = [
    "Stock market rises after good news",   # Positif
    "Market crashes amid economic fears",   # Négatif
    "Investors optimistic about recovery",  # Positif
    "Oil prices plummet due to low demand", # Négatif
    "Tech stocks rally after strong results", # Positif
    "Recession fears drive markets lower"   # Négatif
]

labels = [1, 0, 1, 0, 1, 0]  # 1 = Positif, 0 = Négatif
```

---

## Étape 3 : Transformation TF-IDF

```python
vectorizer = TfidfVectorizer()
X = vectorizer.fit_transform(docs)
```

---

## Étape 4 : Division train/test

```python
X_train, X_test, y_train, y_test = train_test_split(
    X, labels, test_size=0.3, random_state=42
)
```

---

## Étape 5 : Entraînement de la régression logistique

```python
model = LogisticRegression()
model.fit(X_train, y_train)
```

---

## Étape 6 : Évaluation

```python
y_pred = model.predict(X_test)
print(classification_report(y_test, y_pred))
```

---

## Étape 7 : Tester le modèle sur de nouvelles phrases

```python
new_docs = [
    "Stock prices soar after government stimulus",
    "Global recession fears hit investors hard"
]

X_new = vectorizer.transform(new_docs)
preds = model.predict(X_new)

for doc, pred in zip(new_docs, preds):
    sentiment = "Positif" if pred == 1 else "Négatif"
    print(f"{doc} -> {sentiment}")
```

---

# 4. Travail demandé aux étudiants

1. Implémenter le code ci-dessus pas à pas.
2. Modifier le jeu de données en ajoutant leurs propres phrases positives/négatives.
3. Tester le modèle sur de nouvelles phrases.
4. Extraire et analyser les mots les plus importants :

   * `model.coef_` donne les poids associés aux mots.
   * Relier les poids aux mots avec `vectorizer.get_feature_names_out()`.

