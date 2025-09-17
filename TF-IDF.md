#  Formule

$$
TF-IDF(t, d) = TF(t, d) \times IDF(t)
$$

* **$t$** = un **terme** (mot) du vocabulaire
* **$d$** = un **document** de ton corpus (ensemble des textes)
* **TF(t,d)** = **importance locale** du mot dans le document (fréquence du mot dans le texte)
* **IDF(t)** = **importance globale** du mot dans le corpus (rareté du mot dans l’ensemble des textes)
Donc **TF-IDF = importance locale × rareté globale**.
Cela permet de donner un poids fort à un mot **spécifique à un document**, et faible à un mot **répandu partout**.

---

# 🔹 Exemple avec 3 documents

Corpus :

1. d₁ = `"Stock market rises today"`
2. d₂ = `"Crypto market crashes today"`
3. d₃ = `"Investors optimistic about market"`

Donc $N = 3$ documents.

---

## Étape 1 : Calcul de TF(t,d₁)

Dans d₁ : 4 mots → \["stock", "market", "rises", "today"]
Chaque mot apparaît **1 fois**.

$$
TF(\text{“stock”}, d₁) = \frac{1}{4} = 0.25
$$

$$
TF(\text{“market”}, d₁) = \frac{1}{4} = 0.25
$$

$$
TF(\text{“rises”}, d₁) = \frac{1}{4} = 0.25
$$

$$
TF(\text{“today”}, d₁) = \frac{1}{4} = 0.25
$$

---

## Étape 2 : Calcul de IDF(t)

$$
IDF(t) = \ln\left(\frac{N}{n_t}\right)
$$

* “stock” → présent dans 1 doc ($n_t = 1$) → $\ln(3/1) = \ln(3) ≈ 1.099$
* “market” → présent dans 3 docs ($n_t = 3$) → $\ln(3/3) = \ln(1) = 0$
* “rises” → présent dans 1 doc ($n_t = 1$) → $\ln(3/1) = \ln(3) ≈ 1.099$
* “today” → présent dans 2 docs ($n_t = 2$) → $\ln(3/2) ≈ 0.405$

---

## Étape 3 : Calcul de TF-IDF(t,d₁)

$$
TF-IDF(t, d₁) = TF(t,d₁) \times IDF(t)
$$

* “stock” : $0.25 \times 1.099 = 0.275$
* “market” : $0.25 \times 0 = 0$
* “rises” : $0.25 \times 1.099 = 0.275$
* “today” : $0.25 \times 0.405 = 0.101$

---

# Résultat final pour d₁

Vecteur TF-IDF(d₁) ≈

$$
[stock=0.275,\ market=0,\ rises=0.275,\ today=0.101]
$$

---

# Interprétation

* “market” apparaît partout → poids nul.
* “stock” et “rises” sont **importants et spécifiques** à d₁.
* “today” a une importance moyenne (présent dans 2 docs).



