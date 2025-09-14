# Projet fil rouge « SmartMarket »

## 1) Périmètre fonctionnel obligatoire

1. **Catalogue** : gestion des catégories et produits (états actifs/inactifs, prix non négatifs, stocks), pages liste/détail côté front minimal, back-office efficace (recherche, filtres, actions).
2. **Comptes** : création, authentification, réinitialisation, rôles/groupes (`admin`, `manager`, `client`) et permissions cohérentes (RBAC).
3. **Commandes** : création/lecture authentifiées ; visibilité « objet-niveau » (chaque client ne voit que ses commandes) ; supervision globale par `manager/admin`.
4. **API REST** : endpoints versionnés `/api/v1` pour catalogue et commandes ; pagination, filtres, throttling ; documentation OpenAPI (UI Swagger/Redoc).
5. **Recommandations basiques** : bloc de recommandations produit-à-produits (au minimum basé contenu), filtré (actifs, stock), avec justification courte (« raison »).
6. **Recherche sémantique** : endpoint `/search?q=...` retournant Top-K produits pertinents ; tolérance aux reformulations ; index local (lexical et/ou vectoriel).
7. **Assistant RAG** : endpoint `/assistant/ask` adossé à un corpus interne (FAQ/politiques) ; réponses sourcées (liste des extraits/documents) ; refus explicite si hors corpus.
8. **Tâches Celery** : workers + beat pour régénération d’artefacts (embeddings/reco) et envoi d’e-mails de commande ; retries/backoff ; idempotence.
9. **Notifications e-mail** : confirmation de commande (contenu minimal, traçabilité d’envoi).
10. **Tableau admin temps réel** : notifications WebSocket/SSE (Django Channels) lors de nouvelles commandes ; authentification et contrôle d’accès.
11. **Déploiement Docker** : images séparées (`web`, `worker`, `beat`, `nginx`) ; Gunicorn/Nginx ; variables via `.env` ; `collectstatic` et migrations intégrés au processus.

## 2) Exigences de qualité (non négociables)

1. **Tests** : couverture **cible ≥ 60 %** (mesurée) ; tests positifs/négatifs ; tests API clés (auth, permissions, RGPD, throttling).
2. **Lints** : black/ruff **propres** (aucune erreur bloquante).
3. **Sécurité/RGPD** : secrets **hors dépôt** ; `.env.example` fourni ; mentions RGPD dans la documentation ; endpoints **export**/**suppression** utilisateur ; **anonymisation** des données de démonstration ; logs d’opérations sensibles (horodatage, identifiant).
4. **Documentation** : `README.md` clair (installation, démarrage dev/prod, rôles, scénarios de test), **schéma d’architecture** et **explication des flux** (requêtes web, API, Celery, Channels, e-mail).
5. **Outillage** : cibles Make obligatoires :
   `make dev` / `make up` / `make test` / `make seed`.
6. **Traçabilité/Observabilité** : endpoints de santé (`/health/live`, `/health/ready`) ; journalisation lisible (au minimum niveau INFO) ; preuve de bons paramètres en production (ex. `DEBUG=False`, `ALLOWED_HOSTS` défini).

## 3) Livrables attendus (contenu de l’archive)

1. **Archive** : `prenom_nom_projet-final.zip` (sans `.venv` ni volumes docker).
2. **Arborescence minimale** :

   * `src/` (code applicatif) ; `docker-compose.yml` (+ prod si distinct) ; `Dockerfiles/` ; `Makefile`.
   * `.env.example` ; `README.md` ; `OPENAPI.md` (ou export du schéma) ; `ARCHITECTURE.md` (diagramme + flux) ; `RUNBOOK.md` (start/stop/migrate/backup/restore).
   * Données de démonstration (fixtures/commande) ; captures demandées (cf. recette).
3. **Captures minimales** :

   * UI OpenAPI (liste d’endpoints) ;
   * Export RGPD d’un compte test ;
   * Tableau admin recevant une notification temps réel ;
   * Exécution d’une tâche Celery (preuve) ;
   * Résultat de `/health/ready`.

## 4) Procédure de recette (vérifications à effectuer)

1. **Démarrage** : `make up` → Postgres/Redis/containers OK ; `make dev` ou équivalent prod ; `collectstatic` et migrations passées.
2. **API** : `/api/v1/products` paginé ; filtres ; throttling sur endpoints sensibles (login/reset).
3. **RBAC** : `client` ne peut pas modifier le catalogue ni lire les commandes d’autrui ; `manager/admin` gèrent tout le catalogue et supervisent les commandes.
4. **Recommandations** : sur une page produit, le bloc retourne des candidats actifs en stock, avec « raison » ; latence raisonnable (cache si nécessaire).
5. **Recherche** : `/search?q=...` retourne des résultats pertinents et triés.
6. **Assistant RAG** : réponse sourcée lorsque le corpus couvre la question ; refus explicite sinon.
7. **Celery** : beat exécute une planification (ex. régénération quotidienne) ; e-mail de commande envoyé ; retries/backoff configurés.
8. **Channels** : création d’une commande → notification reçue en temps réel côté tableau admin.
9. **RGPD** : export JSON conforme (profil + commandes du propriétaire) ; suppression logique/physique documentée et opérante ; journalisation associée.
10. **Qualité** : lints clean ; `make test` vert ; couverture mesurée ≥ cible (ou valeur explicitée).
11. **Sécurité** : `DEBUG=False` en prod ; `ALLOWED_HOSTS` défini ; secrets non committés ; CORS/CSRF cohérents selon mode d’auth.

---

# Barème d’évaluation — Projet fil rouge (total **/100**, convertible **/20** en divisant par 5)

> Chaque critère précise les **preuves attendues** et l’échelle d’appréciation. Un **critère bloquant** () invalide la soutenance s’il n’est pas respecté, même si le score arithmétique est suffisant.

## 1) Architecture & Déploiement Docker (Nginx/Gunicorn, images, Compose) — **10 pts**

* **Preuves** : Dockerfiles multi-stage, images `web/worker/beat/nginx`, `docker-compose` (dev/prod si distinct), `collectstatic`/migrations dans le processus, variables via `.env`.
* **Échelle** :

  * 9–10 : images non-root, config WebSockets Nginx, process de déploiement documenté et reproductible.
  * 7–8 : images séparées et fonctionnelles, quelques lacunes mineures.
  * 5–6 : mono-image ou procédures manuelles non fiabilisées.
  * ≤4 : déploiement non fonctionnel. 

## 2) Catalogue (modèles, admin, performance basique) — **10 pts**

* **Preuves** : contraintes (unicité, prix ≥ 0), index utiles, admin avec filtres/recherche/actions, optimisation anti-N+1 (`select_related`).
* **Échelle** : 9–10 robuste/ergonomique ; 7–8 correct ; 5–6 lacunes ; ≤4 incohérences de données.  si contraintes manquantes.

## 3) Comptes & RBAC — **10 pts**

* **Preuves** : rôles groupés, permissions DRF, accès objet-niveau pour les commandes.
* **Échelle** : 9–10 strict et documenté ; 7–8 correct ; 5–6 trous d’accès ; ≤4 RBAC inopérant. 

## 4) Commandes — **10 pts**

* **Preuves** : création/lecture sécurisées ; séparation des vues client/admin ; validations minimales.
* **Échelle** : 9–10 complet ; 7–8 fonctionnel ; 5–6 lacunes ; ≤4 flux cassé. 

## 5) API REST & OpenAPI — **10 pts**

* **Preuves** : `/api/v1/...`, pagination, filtres, throttling, schéma OpenAPI + UI.
* **Échelle** : 9–10 conforme et documentée ; 7–8 mineures ; 5–6 docs incomplètes ; ≤4 API non testable. 

## 6) Recommandations basiques — **10 pts**

* **Preuves** : endpoint/bloc de reco, filtrage (actifs/stock), justification courte, latence raisonnable (cache si nécessaire).
* **Échelle** : 9–10 pertinent et stable ; 7–8 correct ; 5–6 basique sans filtres ; ≤4 inexistant.

## 7) Recherche sémantique — **10 pts**

* **Preuves** : `/search?q=...` ; pertinence observable ; index local (lexical et/ou vectoriel) ; description du pipeline.
* **Échelle** : 9–10 cohérente ; 7–8 fonctionnelle ; 5–6 approximative ; ≤4 manquante.

## 8) Assistant RAG — **10 pts**

* **Preuves** : corpus interne, réponses **sourcées**, refus si hors corpus, journalisation (trace minimale).
* **Échelle** : 9–10 garde-fous et traçabilité ; 7–8 correct ; 5–6 fragile ; ≤4 non livrable.  si hallucinations sans sources.

## 9) Asynchronisme Celery & Notifications e-mail — **10 pts**

* **Preuves** : worker + beat, tâches planifiées (reco/embeddings), envoi d’e-mail de commande, retries/backoff, idempotence.
* **Échelle** : 9–10 robuste ; 7–8 opérationnel ; 5–6 lacunes (pas de retries) ; ≤4 inopérant. 

## 10) Temps réel (Channels) — **10 pts**

* **Preuves** : notification live à la création de commande ; authentification ; contrôle d’accès ; reconnect.
* **Échelle** : 9–10 fiable ; 7–8 fonctionnel ; 5–6 instable ; ≤4 absent.

## 11) Qualité & Tests (couverture cible ≥ 60 %) — **5 pts**

* **Preuves** : `make test` vert, rapport de couverture.
* **Échelle** :

  * 5 : ≥ 60 % ;
  * 4 : 50–59 % ;
  * 3 : 35–49 % ;
  * 1–2 : <35 % ;
  * 0 : tests non exécutables.
*  si aucun test d’API critique (auth/permissions/RGPD).

## 12) RGPD & Sécurité opérationnelle — **5 pts**

* **Preuves** : mentions dans README, **export/suppression** opérants, anonymisation des démos, secrets hors dépôt, `DEBUG=False` en prod.
* **Échelle** : 5 complet ; 3–4 avec lacunes mineures ; 1–2 incomplet ; 0 absent.  si export/suppression non fonctionnels.

## 13) Documentation & Schéma d’architecture & Makefile — **3 pts**

* **Preuves** : `README.md`, `ARCHITECTURE.md` (diagramme + flux expliqués), `OPENAPI.md` (ou UI), cibles Make demandées.
* **Échelle** : 3 complet ; 2 acceptable ; 1 minimal ; 0 lacunaire.

## 14) Observabilité & Santé — **2 pts**

* **Preuves** : `/health/live` et `/health/ready`, captures de vérification, logs lisibles.
* **Échelle** : 2 complet ; 1 partiel ; 0 absent.

**Total : 100 pts** → **Note /20 = Total/5**

---

## Attendus de démonstration (séquence synthétique)

1. Démarrage (`make up`) → santé OK.
2. Parcours front minimal : liste/détail produit → bloc de recommandations.
3. API : lecture catalogue (paginée/filtrée), création commande en tant que client, interdictions RBAC vérifiées.
4. Recherche : `/search?q=...` avec résultats pertinents.
5. Assistant : question couverte → réponse sourcée ; question hors corpus → refus.
6. Création commande → e-mail envoyé ; notification live reçue côté admin.
7. Exécution beat visible (log/planification) ; export RGPD + suppression ; lints/tests/coverage présentés ; schéma d’architecture commenté.
