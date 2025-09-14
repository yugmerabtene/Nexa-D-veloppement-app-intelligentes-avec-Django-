# Syllabus – Développement d’applications web intelligentes avec Django

## Présentation du cours

**Durée recommandée :** 4 jours (≈ 28 h) — alternant théorie le matin, TP l’après‑midi.
**Pédagogie :** « learn‑by‑building » avec projet fil rouge, revues de code, tests et mini‑audits sécurité.

---

## Prérequis

* Bases de Python (fonctions, classes, virtualenv/poetry, pip).
* Connaissances web : HTTP, REST, JSON, Git.
* Notions de bases de données (SQL).
* Environnement : PC avec Docker, Python ≥ 3.11, VS Code, Git, make.

**Packages/Services utilisés** (selon ateliers) : Django 5.x, Django REST Framework, PostgreSQL, Redis, Celery, Channels, Pydantic, pytest, scikit‑learn, pandas, numpy, sentence-transformers/embeddings (locale), FAISS/chroma (optionnel), OAuth2/OIDC (Keycloak/Auth0 ou social auth), API LLM (fournisseur au choix), OpenAPI/Swagger, Nginx, Gunicorn, Docker Compose.

---

## Objectifs pédagogiques


1. Concevoir une application Django modulaire, testée et sécurisée (RBAC, permissions DRF, CSRF, XSS, ORM safe).
2. Exposer une API REST/JSON robuste (DRF, schéma OpenAPI, validation).
3. Intégrer des fonctions intelligentes : recommandations, NLP (classification, recherche sémantique), scoring.
4. Orchestrer des tâches asynchrones (Celery/Redis), temps réel (Channels/WebSockets).
5. Mettre en production (Docker, Nginx/Gunicorn, migration, migrations, observabilité) avec CI/CD.
6. Respecter RGPD et bonnes pratiques de sécurité (mots de passe, logs, secrets, data minimization).

---

## Organisation & livrables

* **Projet fil rouge** : “SmartMarket” (mini‑e‑commerce / marketplace) intégrant : catalogue, comptes, panier, API, back‑office, moteur de recommandation, recherche sémantique, chatbot d’aide, notifications (mail), tableau d’observabilité.
* **Livrables par journée** : `prenom_nom_lab-0X.zip` (code + README + captures éventuelles).
* **Livrable final** : `prenom_nom_projet-final.zip` (code, docker-compose, `.env.example`, schéma d’architecture ASCII, scripts de démarrage, jeu de données de démo, rapport court sécurité/RGPD).


---

## Plan détaillé par journée

### Jour 1 — Démarrage Django + ORM & Admin

**Objectifs** : poser l’architecture, modéliser le catalogue, maîtriser l’ORM et l’admin.
**Théorie**

* Architecture Django (MTV), settings par environnement, apps, URLs, views, templates, ORM.
* Organisation pro du dépôt (`src/`, `.env`, `.env.example`, gestion des secrets), 12‑Factor.
* Modélisation relationnelle (FK/M2M/OneToOne), contraintes (`UniqueConstraint`, `CheckConstraint`), index.
* Admin avancé : actions, filtres, permissions, inlines; performance ORM (N+1, `select_related`, `prefetch_related`).

**TP‑01 — Bootstrap du projet SmartMarket**

1. Créer le projet `smartmarket` + app `catalog` (modèles `Product`, `Category`).
2. Postgres via Docker Compose, migrations, fixtures de démo, debug toolbar.
3. Templates liste/détail produits; admin personnalisé (recherche, filtres, actions).
4. Qualité : `black`, `ruff`, `pytest` + tests ORM basiques; `README` avec commandes `make`.

---

### Jour 2 — API REST (DRF) + Auth/RBAC + Sécurité & RGPD

**Objectifs** : exposer une API proprement documentée ; sécuriser et gérer les comptes.
**Théorie**

* DRF : serializers, viewsets, routers, pagination, filtres, throttling, schéma OpenAPI.
* Auth : `AbstractUser` vs `AbstractBaseUser`, JWT/Session, permissions objet/rôle, rate limit.
* Sécurité web : CSRF, XSS, SSRF, headers `SECURE_*`, stockage mots de passe.
* RGPD : minimisation, base légale, consentement, export/suppression, journalisation.

**TP‑02 — API publique + Comptes & RGPD**

1. Endpoints DRF pour `Product`, `Category`, `Order` (CRUD contrôlé).
2. `CustomUser`, inscription/connexion, reset mot de passe, groupes & permissions DRF (admin, manager, client).
3. Throttling endpoints sensibles; configuration des headers de sécurité.
4. Export JSON des données utilisateur et suppression (logique/physique) ; tests API (`APIClient`).
5. Swagger/OpenAPI exposé; jeu de tests de sécurité basiques.

---

### Jour 3 — Intelligence : recommandations, recherche sémantique & assistant RAG

**Objectifs** : ajouter des fonctionnalités « intelligentes » directement exploitables.
**Théorie**

* Recs : popularité, contenu (TF‑IDF/embeddings), aperçu collaboratif, métriques (precision/recall).
* NLP/recherche : nettoyage texte, vectorisation, index local (FAISS/chroma).
* RAG responsable : ingestion, chunking, retrieval, guardrails, auditabilité.

**TP‑03 — Recs + Recherche + Assistant**

1. Module `ml/` : pipeline contenu‑based (scikit‑learn) + endpoint `/products/{id}/recommendations/`.
2. Index d’embeddings produits + endpoint `/search?q=...` (recherche sémantique).
3. Assistant d’aide à l’achat : `/assistant/ask` (retrieval → réponse) avec traçabilité (questions/contexte/réponses) et opt‑out RGPD.
4. Cache Redis + invalidation sur update de catalogue; tests offline + tests API.

---

### Jour 4 — Asynchrone & temps réel + Déploiement & CI/CD

**Objectifs** : fiabiliser (tâches), temps réel, livrer en production avec observabilité.
**Théorie**

* Celery + Redis : workers, retries, tâches planifiées (beat).
* Channels : WebSockets/SSE (notifications commandes, dashboard).
* Prod : Gunicorn + Nginx, statics/media, variables secrètes, sauvegardes/restauration.
* CI/CD : tests, build images, déploiement sur VPS; logs structurés, health checks.

**TP‑04 — Orchestration, WebSockets & Déploiement**

1. Celery (worker + beat) pour régénérer embeddings/recs et envoyer e‑mails de commande.
2. Channels : notifier un canal `orders` et afficher un tableau admin temps réel.
3. Dockerfiles (app, worker) + `docker-compose.prod.yaml`, Nginx reverse proxy, Gunicorn.
4. Pipeline CI (GitHub Actions/GitLab CI) : lint, tests, build & push images, déploiement SSH.
5. Runbook (start/stop/migrate/backup/restore), healthcheck et rotation des logs; mini‑audit sécurité.



## Projet fil rouge – Spécification

**Nom :** SmartMarket.
**Modules obligatoires :** catalogue, comptes, commandes, API REST, recommandations basiques, recherche sémantique, assistant RAG, tâches Celery, notifications mail, tableau admin temps réel, déploiement Docker.
**Exigences qualité :**

* Couverture tests ≥ 60 % (cible), lints clean, `README.md` clair.
* `.env.example`, secrets non commités, user admin par script, données de démo.
* Mesures RGPD : mentions, export/suppression, logs, anonymisation de démo.
* Schéma d’architecture ASCII + explication des flux.
* Script `make dev` / `make up` / `make test` / `make seed`.

**Barème indicatif (50 pts)**

* Architecture & code propre (10)
* API DRF & sécurité (10)
* Intelligence (recs + RAG) (12)
* Asynchrone/temps réel (8)
* CI/CD + déploiement (6)
* Documentation & RGPD (4)

---

## Ressources & lectures conseillées

* Documentation Django, DRF, Celery, Channels.
* OWASP Top 10, RGPD (Article 32 — sécurité du traitement).
* Notions ML : scikit‑learn user guide, Text/Embeddings.
* Observabilité : 12‑factor app, logging structuré, health checks.









