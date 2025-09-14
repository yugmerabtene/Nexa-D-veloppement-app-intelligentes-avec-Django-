# Jour 2 — API REST (DRF) + Auth/RBAC + Sécurité & RGPD

### 1) Compétences visées en fin de journée

* Concevoir une API REST cohérente (ressources, verbes, statuts, erreurs) et versionnée.
* Mettre en place DRF : serializers, viewsets, routers, pagination, filtres, throttling, schéma OpenAPI.
* Implémenter l’authentification (session ou JWT), le RBAC par groupes/permissions, et des permissions objet.
* Appliquer les contrôles de sécurité web (CSRF, XSS, SSRF, CORS, en-têtes `SECURE_*`) et des limites de débit.
* Respecter le RGPD : minimisation, base légale, information, export/suppression, journalisation.
* Garantir la qualité : tests d’API (positifs/négatifs), couverture minimale, documentation exploitable.

### 2) Positionnement de l’API dans SmartMarket

* **Fronts** (templates, SPA ultérieurement) et **intégrations** (clients tiers) consomment l’API.
* L’API est **publique en lecture** sur le catalogue (selon les besoins) et **contrôlée** pour la gestion (création/édition).
* Les endpoints « comptes » et « données personnelles » sont **authentifiés** et soumis à **throttling** renforcé.

### 3) Conception REST : ressources, URLs, verbe, statuts

* **Ressources principales** : `categories`, `products`, `orders`, `users` (scopées selon rôle).
* **URLs** : `/api/v1/products/`, `/api/v1/products/{slug}/`, `/api/v1/orders/{id}/…`.
* **Verbes** : `GET` (lecture), `POST` (création), `PUT/PATCH` (modification), `DELETE` (suppression).
* **Statuts** : `200/201/204`, erreurs `400/401/403/404/409/422/429/500` selon cas.
* **Idempotence** : `GET`, `PUT`, `DELETE` doivent être idempotents ; `POST` ne l’est pas.
* **Versionning** : inclure `v1` dans l’URL (ou entête) pour permettre l’évolution ultérieure.

### 4) DRF : serializers, viewsets, routers

* **Serializer** : définit la **forme** et la **validation** des données entrantes/sortantes.
* **ModelSerializer** accélère le CRUD mais ne dispense pas de **champ explicite** pour masquer/sanitariser.
* **ViewSet** : regroupe les actions d’une ressource (list, retrieve, create, update, destroy).
* **Router** : génère des URLs cohérentes ; réduit le code « plomberie ».
* **Bonnes pratiques** :

  * Séparer **représentation de lecture** (champs exposés) et **écriture** (champs modifiables).
  * Valider **contrats métier** côté serializer (ex. prix ≥ 0) en plus des contraintes BD.
  * Gérer **messages d’erreur** précis et stables (ne pas exposer d’internals).

### 5) Pagination, filtres, recherche, tri

* **Pagination** obligatoire (page/limit, offset, ou cursor) avec métadonnées (`count`, `next`, `previous`).
* **Filtres** : champs fréquents et indexés (ex. catégorie, état actif).
* **Recherche** : texte sur champs limités et raisonnables (nom/slug) ; attention aux performances.
* **Tri** : borne blanche (liste de tris autorisés) pour prévenir l’abus et l’anti-pattern « tri libre sur colonnes non indexées ».

### 6) Throttling (limitation de débit)

* **Niveaux** : global (anonyme/IP), authentifié (par utilisateur), **renforcé sur endpoints sensibles** (login/reset).
* **Objectif** : atténuer force brute, griefing et abus de ressources.
* **Comportement** : retour `429 Too Many Requests` avec fenêtre de remise (entêtes optionnels).

### 7) Authentification : session vs JWT (choix éclairé)

* **Session (cookie)** : simple, intégrée à Django, **CSRF requis** pour les méthodes mutatives, pratique pour back-office.
* **JWT (Authorization: Bearer)** : stateless côté serveur, adapté aux clients multiples (SPAs, mobiles), nécessite **rotation/expiration**, stockage **cookie `HttpOnly`** (recommandé) ou mémoire (exposition risque XSS).
* **Choix pédagogique Jour-02** :

  * **Minimal** : session pour admin + endpoints API protégés via SESSION (plus rapide à mettre).
  * **Optionnel/bonus** : JWT pour l’API publique authentifiée (clients externes), avec **refresh** court et rotation.
* **Remarques** : pour JWT, prévoir **révocation** (liste noire) ou rotation + court TTL ; pour session, exiger **CSRF**.

### 8) RBAC et permissions (DRF)

* **Groupes** Django : `admin`, `manager`, `client` (exemple).
* **Permissions DRF** :

  * Niveau **vue** : `IsAuthenticated`, `IsAdminUser`, `DjangoModelPermissions`, combinaisons personnalisées.
  * Niveau **objet** : contrôle du propriétaire (ex. un client ne voit **que** ses commandes).
* **Principe de moindre privilège** : lecture publique sélective, écriture réservée, opérations sensibles limitées aux rôles habilités.

### 9) Sécurité web : CSRF, XSS, SSRF, CORS, en-têtes

* **CSRF** : obligatoire avec **session/cookies** ; pas requis avec **JWT** en en-tête, mais garder **mêmes protections** côté front.
* **XSS** : ne jamais réinjecter du contenu non nettoyé ; côté API, **sanitiser** ou **restreindre** les champs texte.
* **SSRF** : ne pas autoriser d’URL externes non filtrées en entrée ; aucune « proxyfication » non contrôlée.
* **CORS** : liste blanche stricte des origines ; pas de `*` en production ; cookies `SameSite`/`Secure` selon contexte.
* **En-têtes de sécurité** (production) :

  * `SECURE_SSL_REDIRECT`, `SECURE_HSTS_SECONDS` (+ include subdomains, preload),
  * `SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE`, `CSRF_COOKIE_HTTPONLY (False par défaut, garder le défaut)`,
  * `X_FRAME_OPTIONS='DENY'`, `SECURE_REFERRER_POLICY`, `SECURE_CONTENT_TYPE_NOSNIFF`.
* **Erreurs** : messages sobres (pas de tracebacks) ; journalisation interne détaillée.

### 10) RGPD : principes opérationnels pour l’API

* **Base légale** : exécution du contrat (commande), intérêt légitime (sécurité), consentement (communications marketing).
* **Minimisation** : exposer le **strict nécessaire** dans les serializers de sortie ; filtrer les champs sensibles.
* **Transparence** : documenter les finalités et droits dans le **README/API docs** ; fournir un contact DPO.
* **Droits** :

  * **Accès/portabilité** : export JSON structuré des données utilisateur (scope maîtrisé).
  * **Suppression** : suppression **logique** ou **physique** selon la donnée ; conserver ce qui est légalement requis (facturation) et **anonymiser** au besoin.
* **Journalisation** : traces des exports/suppressions, identité de l’opérateur (self-service ou admin), horodatage.

### 11) Documentation de l’API (OpenAPI)

* **Schéma** généré automatiquement (DRF + extension) ; accessible via `/api/schema/` et **UI** (Swagger/Redoc).
* **Exemples** : requêtes et réponses types (statuts d’erreur inclus).
* **Contrats stables** : gérer la dépréciation (entête `Sunset`, champ `deprecated`) avant la suppression.

### 12) Stratégie d’erreurs et messages

* **Structure uniforme** (code interne, message humain, éventuels champs invalides).
* **Éviter la divulgation** d’internals (noms de colonnes sensibles, stack trace).
* **Corrélation** : inclure un identifiant de requête (header) pour le support.

### 13) Tests d’API

* **Couverture minimale** :

  * Authentication (flux succès/échec, verrouillage/bruteforce via throttling).
  * Permissions (accès refusé où attendu).
  * Contrats (schéma, champs requis, contraintes).
  * Export/suppression (RGPD) et absence de fuite de données.
* **Jeu de données** : fixtures/factories maîtrisées ; tests indépendants et reproductibles.
* **Négatifs** : invalidations, dépassement de quotas (`429`), accès cross-user interdit (`403/404`).

### 14) Observabilité minimale côté API

* **Logs structurés** (niveau, route, statut, latence, identifiant requête).
* **Compteurs** simples (appel endpoints sensibles) ; approfondissement au Jour-04 (metrics, healthchecks).

---

# TP-02 — API publique + Comptes & RGPD (noté /20)

## 1) Objectif

Fournir une **API REST documentée** pour les ressources **Product**, **Category**, **Order**, avec **authentification**, **RBAC**, **throttling**, **contrôles de sécurité**, et **mécanismes RGPD** d’export/suppression des données utilisateur. L’API doit être **paginée, filtrable** et **documentée en OpenAPI**.

## 2) Livrables attendus

* **Archive** : `prenom_nom_lab-02.zip` contenant :

  * Code (`src/`), configuration DRF, schéma OpenAPI, `README.md` clair (pré-requis, commandes, scénarios de test).
  * Fichiers de configuration qualité/tests (black, ruff, pytest, coverage).
  * **Captures** :

    1. UI Swagger/Redoc montrant les endpoints et modèles,
    2. Résultat d’un export RGPD utilisateur,
    3. Exemple de refus d’accès à une ressource d’un autre utilisateur (`403/404`).
  * **Optionnel** : collection Postman/Insomnia ou scripts `curl` reproductibles.

## 3) Périmètre fonctionnel (check-list)

1. **Endpoints DRF (versionnés `/api/v1/`)**

   * `categories` : liste/détail (lecture publique).
   * `products` : liste/détail (lecture publique), création/édition/suppression **réservées** (rôles `admin/manager`).
   * `orders` : création/lecture **authentifiées** ; chaque utilisateur ne consulte **que ses propres commandes** ; administration : lecture globale/gestion par `admin/manager`.
2. **Authentification**

   * **Obligatoire** : authentification **session** (ou **JWT** si vous le préférez).
   * **Sécurité** : mots de passe stockés de manière sûre (mécanisme natif Django), endpoints sensibles sous **throttling**.
3. **Comptes & rôles (RBAC)**

   * Utilisateur modèle : CustomUser **ou** User Django + Groupes (`admin`, `manager`, `client`).
   * Permissions DRF cohérentes :

     * `client` : lecture catalogue, gestion **de ses propres** commandes.
     * `manager`/`admin` : gestion CRUD du catalogue et supervision commandes.
4. **Sécurité web**

   * **CSRF** pour flux session (méthodes mutatives).
   * **CORS** : liste blanche stricte en dev ; pas de wildcard en prod.
   * **En-têtes** `SECURE_*` paramétrés pour l’environnement de production (dans un fichier dédié aux settings).
   * **Throttling** :

     * Public (anonyme) : ex. 60 req/min,
     * Authentifié : ex. 120 req/min,
     * Endpoints sensibles (`login`, `password reset`, `export`, `erase`) : ex. 10 req/min.
5. **RGPD**

   * **Export** JSON des données utilisateur (profil + commandes personnelles) via endpoint **authentifié**.
   * **Suppression** : suppression **logique** du compte (désactivation + anonymisation commandes) **ou** suppression physique des données non soumises à obligations légales (au choix, mais documenté).
   * **Journalisation** : log interne de l’opération (qui, quand, quoi).
6. **API design**

   * Pagination, filtres et tri documentés.
   * Codes d’erreur et messages uniformes.
   * Schéma **OpenAPI** exposé + UI (Swagger/Redoc).
7. **Qualité & tests**

   * Lint/format propres.
   * **Tests d’API** couvrant :

     * Authentification (succès/échec),
     * Permissions (rôles, objet-niveau sur `orders`),
     * Throttling (retour `429`),
     * Export RGPD (données correctes) et suppression (effet vérifiable),
     * Contrats (champs requis, erreurs 400/404/403).
   * Couverture **≥ 60 %** (indication cible).

## 4) Critères d’évaluation (barème /20)

1. **Conception de l’API & DRF (serializers/viewsets/routers)** — **4 pts**

   * (1) Ressources et routes cohérentes, versionnées
   * (1) Pagination/filtres/tri implementés et documentés
   * (1) Gestion d’erreurs et codes HTTP appropriés
   * (1) Schéma OpenAPI complet et UI accessible
2. **Authentification & RBAC** — **4 pts**

   * (1) Auth fonctionnelle (session ou JWT)
   * (1) Groupes/permissions appliqués
   * (1) Permissions **objet** sur `orders` (propriété)
   * (1) Flux sensibles protégés (login/reset)
3. **Sécurité web & throttling** — **4 pts**

   * (1) CSRF (si session) / CORS configuré
   * (1) En-têtes de sécurité prévus en prod (`SECURE_*`)
   * (1) Throttling global/utilisateur/sensible
   * (1) Messages d’erreur sobres (pas d’internals)
4. **RGPD (export/suppression + journalisation)** — **4 pts**

   * (1) Export JSON conforme et limité au périmètre utilisateur
   * (1) Suppression logique/physique documentée et opérante
   * (1) Journalisation de l’opération (au moins horodatage + identifiant)
   * (1) Minimisation (pas de champs superflus exposés)
5. **Qualité, tests et documentation projet** — **4 pts**

   * (1) Lint/format OK (black/ruff)
   * (1) Batterie de **tests API** pertinente et verte
   * (1) Couverture **≥ 60 %** (ou progression crédible et mesurée)
   * (1) README d’exploitation clair (installation, exemples d’appels, rôles)

**Total : 20 pts**
**Bonus (jusqu’à +2)**

* (+1) Mise en place **conjointe** session **et** JWT (avec refresh/rotation, cookies `HttpOnly Secure`), documentation des choix.
* (+1) Politique de **mots de passe renforcée** (complexité/longueur, HIBP check simulé) et **verrouillage progressif** (compteur d’échecs → temporisation).

## 5) Attendue avant remise

1. Démarrer l’environnement (DB + app), créer les rôles, un compte admin, un compte manager, un compte client.
2. Vérifier : lecture publique du catalogue, impossibilité de créer/modifier en anonyme.
3. Vérifier : `client` ne peut lire que **ses** commandes ; `manager/admin` voient et gèrent tout.
4. Tester throttling (login/reset/export) et contrôler le retour `429`.
5. Générer et ouvrir la documentation OpenAPI ; comparer les exemples aux réponses réelles.
6. Lancer les tests, contrôler la couverture, réaliser les trois captures demandées, préparer l’archive.

