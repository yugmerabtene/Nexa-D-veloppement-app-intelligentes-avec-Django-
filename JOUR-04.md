# Jour 4 — Asynchrone & temps réel + Déploiement & CI/CD

### 1) Compétences visées en fin de journée

* Orchestrer des traitements **asynchrones** fiables avec **Celery + Redis** (workers, planification, reprise).
* Diffuser des événements **temps réel** avec **Django Channels** (ASGI, WebSockets/SSE, couche Redis).
* Mettre en production une application Django **conteneurisée** (Docker multi-images) derrière **Gunicorn + Nginx**.
* Concevoir un **pipeline CI/CD** reproductible (tests → build → scan → push → déploiement).
* Assurer **observabilité**, **sécurité** et **PRA/PCA** de base (health checks, logs, sauvegardes, restauration).

### 2) Quand l’asynchrone est indispensable

* **I/O lents** (e-mail, APIs tierces), **CPU bound** (embedding, génération d’images/rapports), **batchs** (réindexation).
* Objectifs : **robustesse** (retries), **lissage de charge**, **latence perçue** faible côté utilisateur.
* Critères de choix : taille/criticité du job, exigences de délai, idempotence, traçabilité.

### 3) Celery : concepts fondamentaux

* **Broker** (Redis) : file d’attente. **Workers** : exécutants. **Beat** : planificateur périodique.
* **File d’exécution** : files nommées (queues) par type de tâche ; **routing** dédié.
* **Idempotence** : toute tâche doit pouvoir être rejouée sans effet indésirable (clé de déduplication, verrou court).
* **Tolérance aux pannes** : `acks_late`, `visibility_timeout`, **retries** avec **backoff exponentiel**, `max_retries`.
* **Limites et quotas** : `time_limit` dur/`soft_time_limit`, `rate_limit` pour endpoints sensibles.
* **Compositions** : `chain`, `group`, `chord` pour orchestrer pipelines ; préférer **granularité** raisonnable.
* **Transactions** : publier la tâche **après** commit DB (signal `on_commit`) pour éviter les orphelins.

### 4) Planification avec Celery Beat

* Tâches **périodiques** (ex. régénérer embeddings/recommandations, purger caches).
* **Jitter** (aléa) pour éviter l’effet de pointe au top de minute.
* **Fuseaux** : harmoniser sur UTC et convertir côté présentation.

### 5) Fiabilité opérationnelle des tâches

* **Journalisation structurée** (task\_id, nom, tentative, durée, statut).
* **Quarantaine** : file des échecs ou *dead-letter* pour post-mortem.
* **Observabilité** : métriques par type de tâche (succès, échecs, retries, latence P50/P95).
* **Sécurité** : éviter d’encoder des secrets dans les arguments ; privilégier identifiants d’objets.

### 6) Temps réel avec Django Channels

* **ASGI** : serveur asynchrone, **channel layer Redis** pour diffuser aux **groupes** (ex. `orders` ou `admin:<id>`).
* **WebSockets** vs **SSE** : WebSockets bidirectionnels ; SSE unidirectionnel (souvent suffisant pour notifications).
* **Gouvernance** : authentification (session/JWT), contrôle d’accès **RBAC** avant abonnement, messages typés et versionnés.
* **Robustesse** : reconnexion, *heartbeats* (ping/pong), retour au **polling** en dégradé.
* **Back-pressure** : limiter taille/fréquence des messages, **throttling** côté serveur.

### 7) Cas d’usage temps réel (SmartMarket)

* **Canal commandes** : à chaque nouvelle commande, **notification** vers le tableau admin.
* **Panneau admin live** : flux agrégé (N derniers événements), filtres par statut.
* **Tri des priorités** : séparer flux « opération » (commandes) des tâches lourdes (ML) pour ne pas perturber l’UX.

### 8) Architecture de déploiement (conteneurs)

* **Images distinctes** : `web` (Django + Gunicorn), `worker` (Celery), `beat` (Celery Beat), `nginx` (reverse proxy).
* **Règles** : images **non-root**, variables via **.env** (jamais commit), volumes dédiés pour **media** et **logs** si nécessaire.
* **Stockage** : `static` servis par Nginx (pré-collectés), `media` sur disque dédié ou objet (S3-like) selon besoin.
* **Compatibilité** : identiques en dev/prod (parité), différences **seulement** via configuration.

### 9) Gunicorn : réglages essentiels

* **Workers** sync pour Django standard (ASGI via uvicorn/gunicorn asgi si Channels côté HTTP).
* **Dimensionnement** : règle empirique CPU-bound \~ `(2×CPU)+1` ; I/O-bound plus élevé.
* **Timeouts** : équilibrés (ni 30 s partout, ni infinis).
* **Graceful reload** : redémarrage sans coupure, *preload\_app* selon usage mémoire.

### 10) Nginx : reverse proxy sécurisé

* **TLS** moderne (HTTP/2, HSTS), redirections 80→443, ciphers à jour.
* **WebSockets** : en-têtes `Upgrade/Connection` et timeouts adaptés.
* **Sécurité** : en-têtes (`X-Frame-Options`, `Referrer-Policy`, `Content-Type-Nosniff`), limite de taille de requête, *rate limiting*.
* **Statique** : cache agressif sur `static`, contrôlé sur `media`.
* **Observabilité** : logs d’accès en **JSON** (IP, user, route, statut, durée, *request\_id*).

### 11) Gestion des secrets et des permissions

* **.env** en production **hors dépôt**, rotation périodique, cloisonnement des rôles DB.
* **Principes** : moindre privilège, pas de clés longues dans les traces, gestion centralisée (Vault/SSM si possible).
* **Conteneurs** : *read-only root FS*, *drop capabilities*, utilisateur applicatif dédié, montages précis.

### 12) CI/CD : enchaînement type

* **Étape 1 – Qualité** : lint (ruff), format (black check), tests (pytest + couverture).
* **Étape 2 – Build** : Docker **multi-stage** (wheel cache), taggage par *SHA* + *semver*.
* **Étape 3 – Sécurité** : SBOM + scan de vulnérabilités (ex. Trivy) + secrets scanning.
* **Étape 4 – Push** : registre privé.
* **Étape 5 – Déploiement** : SSH vers VPS ; **ordre** : *backup DB → migrations → collectstatic → redéploiement* (web, worker, beat) ; vérifications post-déploiement (health).
* **Stratégie** : *blue/green* ou *rolling* (compose : *down*/*up* orchestré) ; *feature flags* pour bascules progressives.

### 13) Health checks et surveillance

* **HTTP** : `/health/live` (process up), `/health/ready` (DB/redis OK).
* **Celery** : `ping` worker, *heartbeat* beat, métriques de file (longueur, temps d’attente).
* **Channels** : test de *handshake* et message de boucle.
* **Uptime** : sondes externes (latence P95, erreurs 5xx).

### 14) Sauvegardes & restauration (PRA/PCA de base)

* **Sauvegardes** : DB quotidienne + horaire incrémentale si critique ; **chiffrement** au repos ; **rétention** 7–30 jours.
* **Médias** : rsync/objet versionné.
* **Restauration** : procédure écrite, **testée** (tableau des temps), vérification d’intégrité et compatibilité schéma.
* **Objectifs** : RPO/RTO documentés et atteignables.

### 15) Journalisation et rotation

* **Logs JSON** (app, Celery, Nginx) avec **correlation\_id** par requête/tâche.
* **Rotation** : logrotate ou *options Docker* (taille/nb de fichiers).
* **Confidentialité** : pas de PII en clair ; appliquer **data minimization** dans les traces.

### 16) Contrôles de sécurité complémentaires

* **Mises à jour** régulières images/OS, dépendances épinglées (hash) ; *SAST/DAST* en CI si possible.
* **Pare-feu** (ports stricts), **Fail2Ban** côté SSH/Nginx, **WAF** en amont si disponible.
* **RBAC** applicatif et séparation des comptes techniques ; **revues** périodiques des accès.
* **Politique de clés** (rotation, révocation), **inventaire** et **audit** des secrets.

---

# TP-04 — Orchestration, WebSockets & Déploiement (noté /20)

## 1) Objectif

Livrer une version **production-ready** de SmartMarket :

* Tâches **Celery** (worker + beat) pour **régénérer embeddings/recommandations** et **envoyer les e-mails de commande**.
* **Notifications temps réel** (Channels) d’événements de commandes vers un **tableau admin**.
* **Conteneurisation complète** : images `web`, `worker`, `beat`, `nginx`, configuration **Gunicorn** et **Nginx** compatibles WebSockets.
* **CI/CD** : pipeline (lint → tests → build → scan → push → déploiement SSH), **health checks**, **runbook** (start/stop/migrate/backup/restore), **mini-audit sécurité**.

## 2) Livrables attendus

* **Archive** : `prenom_nom_lab-04.zip` contenant :

  * Dossier projet (`src/`), manifests Docker (Dockerfiles), `docker-compose.prod.yaml`, fichiers d’environnement (`.env.example`).
  * **Documentation** : `README.md` (mise en route dev/prod, variables, ports), **Runbook** (`RUNBOOK.md`) : procédures start/stop, backup/restore, migrations, rollbacks.
  * **CI/CD** : fichier de pipeline (GitHub Actions ou GitLab CI) incluant étapes qualité, build, scan, push, déploiement par SSH.
  * **Captures** (3) :

    1. Tableau admin recevant une **notification** de nouvelle commande en temps réel,
    2. Tableau des **tâches Celery** planifiées/exécutées (preuve d’exécution),
    3. **Documentation** de santé (ex. output `/health/ready` et vérification `celery ping`).
  * **Optionnel** : rapport succinct de sécurité (checklist renseignée) et résultats de scan (extrait).

## 3) Périmètre fonctionnel (check-list)

1. **Celery**

   * Worker(s) dédiés avec **queues** séparées (ex. `ml`, `emails`, `default`).
   * **Beat** planifiant : régénération embeddings/recommandations, purge du cache, envoi de résumés (si prévu).
   * **Retries** et **backoff** configurés ; **idempotence** explicitée.
   * Publication des tâches via `on_commit` pour les événements liés à la DB.
2. **Temps réel (Channels)**

   * Couche **Redis** opérationnelle ; groupes logiques (`orders`, `admin:*`).
   * Authentification et **RBAC** vérifiés avant abonnement ; **throttling** basique.
   * **Panneau admin** réactif : réception d’un event « nouvelle commande » avec informations minimales (sans PII).
   * Mode **dégradé** (reconnexion, fallback).
3. **Conteneurs & serveur**

   * Dockerfiles **multi-stage** ; images non-root ; variables via `.env`.
   * **Gunicorn** dimensionné ; **Nginx** configuré (TLS, HSTS, WebSockets, limites, en-têtes sécurité).
   * `collectstatic` intégré au processus de déploiement ; `media` persistants.
4. **CI/CD**

   * Étapes : **lint**, **tests**, **build** (tag), **scan** vulnérabilités, **push**, **deploy** (SSH + commandes : backup DB → migrate → collectstatic → redéploiement ordonné).
   * **Health gates** : refus de mise en prod si tests échouent ou scan critique non accepté.
   * **Rollback** documenté (retour image N-1 + restauration DB si nécessaire).
5. **Observabilité & sécurité**

   * Endpoints `/health/live` et `/health/ready` ; `celery ping`; métriques simples (longueur file, taux d’échecs).
   * **Logs JSON** corrélés (requête/tâche) et rotation.
   * **Sauvegardes** DB + médias avec **rétention** documentée ; **procédure de restauration** testée.
   * **Mini-audit** : checklist *headers sécurité*, *CORS/CSRF*, *ports ouverts*, *mises à jour*, *secrets* (présence/rotation).

## 4) Barème /20

1. **Celery (worker, beat, fiabilité)** — **5 pts**

   * (2) Planification et exécution des tâches clés (recs/embeddings, e-mails)
   * (1) Idempotence + retries/backoff
   * (1) Routage par files / séparation des charges
   * (1) Traces opérationnelles (journalisation, preuves d’exécution)
2. **Temps réel (Channels/WebSockets)** — **5 pts**

   * (2) Notifications fiables et RBAC au **subscribe**
   * (1) Fallback/reconnexion & throttling
   * (1) Données minimisées (pas de PII), message schema clair
   * (1) Démonstration panneau admin recevant l’événement
3. **Conteneurisation & serveur (Nginx + Gunicorn)** — **4 pts**

   * (1) Images multi-stage non-root, `.env` propre, `static/media` gérés
   * (1) Nginx compatible WebSockets + en-têtes de sécurité + limites
   * (1) Dimensionnement Gunicorn, timeouts, redémarrage gracieux
   * (1) Procédure de déploiement ordonnée (collectstatic/migrate)
4. **CI/CD & déploiement** — **4 pts**

   * (1) Pipeline complet (lint/tests/build/scan/push/deploy)
   * (1) Scan de vulnérabilités et politique de *gating*
   * (1) Déploiement par SSH scripté, **health checks** post-déploiement
   * (1) Rollback documenté et testable
5. **Observabilité, sauvegardes & sécurité** — **2 pts**

   * (1) Health checks + logs structurés + métriques élémentaires
   * (1) Stratégie de sauvegarde/restauration + mini-audit sécurité

**Total : 20 pts**
**Bonus (jusqu’à +2)**

* (+1) *Blue/green* effectif avec bascule sans coupure (preuve de continuité).
* (+1) Tableaux de bord de métriques (latence P95, files Celery, erreurs 5xx) exportés vers un outil (même simple).

## 5) Procédure de recette attendue

1. **Pré-déploiement** : tests unitaires/CI verts, images construites et scannées, variables `.env` remplies.
2. **Déploiement** : backup DB, migrations, collectstatic, redéploiement ordonné (`web` puis `worker/beat`).
3. **Vérifications** :

   * `/health/ready` renvoie OK ; `celery ping` répond ; handshake WebSocket fonctionnel.
   * Création d’une commande → **notification reçue** en temps réel côté admin.
   * Exécution planifiée (beat) visible et **résultats** cohérents (cache/embeddings).
4. **Journalisation** : présence des **logs JSON** corrélés (requête/tâche) et rotation effective.
5. **Sauvegarde/Restitution** : exécuter une **restauration de test** sur un dump récent et consigner la durée (RTO simulé).
6. **Sécurité** : vérifier en-têtes, CORS/CSRF, ports exposés, règles pare-feu, comptes et secrets (checklist jointe).
