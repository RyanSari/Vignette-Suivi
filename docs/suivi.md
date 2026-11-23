---
title: Suivi du projet
---

<style>
    @media screen and (min-width: 76em) {
        .md-sidebar--primary {
            display: none !important;
        }
    }
</style>

# Suivi de projet

Cette page documente l’évolution du projet Vignette semaine par semaine.

---

## Semaine 1 (30 octobre – 5 novembre)

### Objectifs de la période
- Prendre en main le projet et vérifier qu'il fonctionne en local
- Comprendre l'architecture du backend (contrôleurs, services, repositories, sécurité)
- Faire un état des lieux du backend et lister les problèmes, en particulier ceux liés aux doublons

### Travail réalisé

!!! abstract "Avancement"
    - [x] Exécution du code et vérification du bon fonctionnement sur mon poste
    - [x] Lecture du backend : authentification, scénarios, médias (images, miniatures, audios), collaborations, forks, likes/favoris, notifications, langues (Glottolog)
    - [x] Rédaction du document « Backend Vignette : résumé des problèmes trouvés », organisé en trois sections

#### 1.1 Doublons de données

| Où | Problème |
|----|----------|
| `AuthApiController` / `UserService` | Vérification du username et de l'email faite deux fois (contrôleur + service) avec des messages différents (« already used » / « taken »), et sensible à la casse : `Alice` et `alice` peuvent coexister. |
| `ScenarioService` (renommage) | Quand un éditeur ou un admin renomme le scénario de quelqu'un d'autre, les titres en double sont cherchés chez l'éditeur et non chez l'auteur : le doublon passe la vérification et l'enregistrement plante. |
| Titre de scénario | Comparaison sensible à la casse (« Test » ≠ « test »), vérification dupliquée entre `ScenarioApiController` et le service, et `generateUniqueScenarioTitle` exposé à une condition de concurrence. |
| `ScenarioCollaborationService` | Une invitation DECLINED ou EXPIRED reste en BD ; `existsByScenarioIdAndUserId` la trouve encore, donc la personne ne peut plus jamais être réinvitée. |
| Likes, favoris, suivi de langue | Aucune vérification de visibilité ; le schéma « vérifier puis insérer » laisse passer deux requêtes simultanées, la seconde finit en erreur. |
| Ordre (`idx`) des images et audios | Le prochain `idx` est calculé avec `max(idx)+1` sans verrou : deux uploads parallèles peuvent obtenir le même `idx`. |
| Dédoublonnage de fichiers | Rien n'empêche d'uploader dix fois la même image dans un scénario. |
| `maxUses` des liens d'invitation | `useCount` incrémenté sans verrou : la limite peut être dépassée avec des requêtes concurrentes. |
| Notifications | Seule la notification « nouveau scénario » est protégée contre les doublons ; invitations, demandes de review de fork, etc. peuvent être envoyées plusieurs fois. |

#### 1.2 Code et logique dupliqués

- **Exceptions** : deux gestionnaires, dont `RestExceptionHandler` (un `@RestControllerAdvice` vide) à côté de `GlobalApiExceptionHandler`.
- **Contrôle des droits** fait de quatre façons : l'aspect `@OwnerOrAdminOperation`, les classes `ScenarioSecurity` / `ThumbnailSecurity` / `AudioSecurity`, les méthodes `ScenarioService.assertCan*`, et `canEdit` / `canView` / `getUserRole` de `ScenarioCollaborationService` (jamais appelées). `AdminApiController` a en plus son propre `assertAdmin()` alors que `@AdminOperation` existe, et beaucoup d'endpoints vérifient les droits deux fois (aspect + service).
- **Méthodes copiées-collées** : `isAdmin`, `requireAuthenticated` et `authenticatedUsername` dans 5 ou 6 services.
- **Copie de fork** faite à la main dans `ScenarioService`, en deux blocs presque identiques et incomplets : `activeBackgroundAudioId` n'est pas copié (le fork perd sa musique de fond) et les audios de scène perdent `sourceLabel` et `volume`.
- **Création d'utilisateur** : `BootstrapData.bootstrapAdminAccount` refait ce que fait `UserService.register`.
- **Schéma de BD** géré à trois endroits : migrations Flyway, `ddl-auto=update` et classes `SchemaMaintenance`.
- **Doublons d'infrastructure** : deux pipelines Glottolog (Java : `LanguageImportService`, `GlottologUpdateService`, `GlottologUpdateJobService`, `GlottologScheduledUpdateService` ; et un worker Python), deux systèmes de notifications (`Notification` et `GlottologNotification`), deux configurations OpenAPI (`OpenApiConfig` et `SwaggerConfig`).
- **Code mort dans les repositories** : `existsByTitle`, `findByTitle`, `findVisibleToUsername`, `findAllByAuthorUsernameOrderByCreatedAtDesc`, `findAllByVisibilityStatusOrderByCreatedAtDesc`, `getThumbnailsByScenarioId`, `existsByNormalizedName`, `findAllByLanguageId`, ainsi que `isOwnerByThumbnailId` et `isOwnerByAudioId` dans `ScenarioSecurity`.

#### 1.3 Sécurité et contrôle d'accès

- **Unicité des usernames incohérente** : unique en tenant compte de la casse, mais `findAllByAuthorUsernameWithTags` et `findVisibleToUsernameWithTags` comparent avec `lower()`. Si `Alice` et `alice` existent, chacune voit les brouillons de l'autre dans « mes scénarios ».
- **XSS stockée** : les fichiers SVG et XML sont acceptés comme images et renvoyés tels quels ; un SVG contenant du JavaScript s'exécute chez ceux qui l'ouvrent.
- **Invitation expirée** : elle donne encore accès en lecture, car `hasPendingInvitation` ne vérifie pas `expiresAt`.
- **Approbation des forks contournable** : une fois le fork APPROVED, son auteur peut encore modifier tout le contenu et le publier.
- **Identifiants admin en dur** (`admin` / `admin12345`) dans `application-pgdev.properties`, activés par défaut en dev.

### Livrable
- Document « Backend Vignette : résumé des problèmes trouvés » (3 pages), qui sert de feuille de route pour les semaines suivantes.

### Prochaines étapes
- Corriger les sections 1.1 et 1.2 en priorité, puis la sécurité (1.3) et la base de données

---

## Semaine 2 (6 – 12 novembre)

### Objectifs de la période
- Corriger les problèmes identifiés dans le document de la semaine 1, en commençant par les doublons de données (1.1) puis le code dupliqué (1.2)

### Travail réalisé

!!! abstract "Avancement"
    - [x] Correction de tous les problèmes de la section **1.1 (doublons de données)**
    - [x] Correction de tous les problèmes de la section **1.2 (code et logique dupliqués)**, à l'exception de la gestion du schéma de la BD
    - [ ] Gestion du schéma de la BD (Flyway / `ddl-auto=update` / `SchemaMaintenance`) : reportée à la semaine 3
    - [ ] Section **1.3 (sécurité et contrôle d'accès)** : reportée à la semaine 3

#### Section 1.1 : doublons de données (corrigée)

- [x] Vérification du username et de l'email : une seule vérification, un seul message d'erreur, insensible à la casse
- [x] Titres de scénario : vérification sur les scénarios de l'**auteur** (et non de l'éditeur/admin qui renomme), insensible à la casse, sans duplication contrôleur/service ; `generateUniqueScenarioTitle` protégé contre la concurrence
- [x] Invitations DECLINED / EXPIRED : il est de nouveau possible de réinviter une personne
- [x] Likes, favoris et suivi de langue : vérification de visibilité et gestion des requêtes simultanées
- [x] `idx` des images et audios : plus de collision entre deux uploads parallèles
- [x] Dédoublonnage des fichiers uploadés dans un même scénario
- [x] `maxUses` des liens d'invitation : la limite ne peut plus être dépassée par des requêtes concurrentes
- [x] Notifications : protection contre les doublons étendue au-delà de la notification « nouveau scénario »

#### Section 1.2 : code et logique dupliqués (corrigée, sauf schéma de BD)

- [x] Un seul gestionnaire d'exceptions (suppression de `RestExceptionHandler`, vide)
- [x] Contrôle des droits unifié : fin des quatre mécanismes parallèles et des doubles vérifications (aspect + service) ; `AdminApiController` n'utilise plus son `assertAdmin()` maison
- [x] `isAdmin`, `requireAuthenticated` et `authenticatedUsername` factorisés au lieu d'être copiés dans 5 ou 6 services
- [x] Copie de fork factorisée et complétée : `activeBackgroundAudioId`, `sourceLabel` et `volume` sont maintenant copiés
- [x] `BootstrapData.bootstrapAdminAccount` réutilise la logique de création d'utilisateur
- [x] Pipelines Glottolog, systèmes de notifications et configurations OpenAPI en double traités
- [x] Méthodes de repository inutilisées ou redondantes supprimées
- [ ] Schéma de la BD géré à trois endroits → semaine 3

### Prochaines étapes
- Unifier la gestion du schéma de la BD
- Corriger les problèmes de sécurité de la section 1.3

---

## Semaine 3 (13 – 19 novembre)

### Objectifs de la période
- Terminer les points restants du document d'analyse : d'abord regrouper la gestion de la base de données en un seul endroit, puis passer à la sécurité
- Commencer le modèle de données (DTO), la gestion des priorités et les utilitaires
- Évaluer l'usage de Supabase

### Travail réalisé

!!! abstract "Avancement"
    - [x] Base de données : gestion du schéma regroupée en un seul endroit
    - [x] Sécurité (section 1.3 du document)
    - [x] Refactoring
    - [x] Priority
    - [x] Utils
    - [x] Étude de l'usage de Supabase
    - [ ] Modèle de données (DTO) ⏳ en cours

#### Base de données

- [x] Le schéma n'est plus géré à trois endroits (migrations Flyway, `ddl-auto=update`, classes `SchemaMaintenance`) mais par un seul mécanisme : les migrations Flyway, avec une migration de base `V1__baseline.sql` générée par `pg_dump`

#### Section 1.3 : sécurité et contrôle d'accès (corrigée)

- [x] Usernames : comparaison cohérente avec la règle d'unicité, un utilisateur ne voit plus les brouillons d'un autre dont le nom ne diffère que par la casse
- [x] Uploads d'images : les fichiers SVG et XML sont refusés, ce qui supprime la XSS stockée (le SVG sera réintroduit de façon sécurisée en semaine 4)
- [x] Invitations : une invitation expirée ne donne plus accès en lecture (`expiresAt` pris en compte)
- [x] Forks : l'approbation ne peut plus être contournée en modifiant le contenu après APPROVED
- [x] Identifiants admin : plus de `admin` / `admin12345` en dur activés par défaut en dev

!!! success "Bilan du document d'analyse"
    À la fin de la semaine 3, **les trois sections** du document « Backend Vignette : résumé des problèmes trouvés » (1.1, 1.2 et 1.3) sont traitées, y compris la gestion du schéma de la BD reportée de la semaine 2.

### Prochaines étapes
- Terminer le modèle de données (DTO)
- Nouveau contrôleur pour enregistrer les vignettes en SVG, avec leur stockage en base de données
- Mettre en place le site de suivi du projet

---

## Semaine 4 (20 – 26 novembre)

### Objectifs de la période
- Terminer le modèle de données
- Réorganiser le code (utilitaires dans `util/`, méthodes de repository à 2 ou 3 paramètres maximum)
- Remettre le format SVG de façon sécurisée
- Créer un nouveau contrôleur pour enregistrer les vignettes en SVG (contrôleur, service, base de données)
- Mettre en place et étoffer le site de suivi du projet

### Travail réalisé

!!! abstract "Avancement"
    - [x] Organisation du code : utilitaires regroupés dans `util/`, méthodes de repository simplifiées
    - [x] Base de données : `V1__baseline.sql` réécrit à la main
    - [x] Retour du format SVG, avec nettoyage des fichiers (`SvgSanitizer`)
    - [x] Nouveau contrôleur pour enregistrer la vignette en SVG (`VignetteApiController` + `VignetteService` + stockage du dessin en BD)
    - [x] Téléchargement des scènes en SVG ou en ZIP (image + audios)
    - [x] Suppression d'un scénario par son auteur ou par un admin depuis le Studio
    - [x] Documentation (README, fichiers à la racine)
    - [x] Tests : 389 tests backend et 205 tests frontend passent
    - [x] Rédaction du document « Backend Vignette : travail réalisé »
    - [x] Site de suivi du projet (à partir du [template IFT3150](https://github.com/udem-diro/template-projet))
    - [ ] Étoffer le suivi de projet ⏳ en cours
    - [ ] Modèle de données (DTO) ⏳ en cours (reporté de la semaine 3)

#### 1. Organisation du code

| Où | Avant | Ce qui a été fait |
|----|-------|-------------------|
| `security/AuthUtils.java` | Classe utilitaire rangée dans `security`, à part des autres utilitaires. | Déplacée dans `util/` avec son test (`AuthUtilsTest`) ; imports mis à jour. |
| `CountryGeoService`, `OpenApiConfig` | `firstNonBlank` copiée dans les deux classes. | Une seule version, dans `util/Text`. |
| `BootstrapData` | Méthode privée `trimmedOrEmpty`. | Déplacée dans `util/Text` pour être réutilisable. |
| `LanguageService`, `UserService` | Rôle `"ROLE_ADMIN"` écrit en dur. | Constante `AuthUtils.ROLE_ADMIN`. |
| `AccreditationRequestRepository`, `CommunityAccreditationRepository` | Méthodes à 4 ou 5 paramètres (ex. `existsByRequestedByUserIdAndPermissionTypeAndScopeTypeAndTargetIdAndStatus`). | Nouveau record `AccreditationScope` (permission, portée, cible) ; méthodes à 2 paramètres maximum : `listByScope`, `existsPending`, `findGrant`, `hasGrant`. Le cas GLOBAL, dupliqué dans `CommunityService`, est géré une seule fois. |

#### 2. Base de données

| Où | Avant | Ce qui a été fait |
|----|-------|-------------------|
| `V1__baseline.sql` | Généré par `pg_dump` : 667 lignes, 59 `ALTER TABLE`, options inutiles (`NO MINVALUE`, `CACHE 1`, `public.`). | Réécrit à la main : 393 lignes, aucun `ALTER`. Clés primaires, `UNIQUE`, clés étrangères et colonnes `IDENTITY` dans chaque `CREATE TABLE`, `CHECK` en `IN (...)`, tables classées par domaine. |
| Colonne `vignette_scene_json` | Ajoutée par une migration V3 avec un `ALTER TABLE`. | Intégrée directement au `CREATE TABLE thumbnail` de V1 ; V3 supprimée. |
| Bases existantes | V1 modifiée → Flyway refuse de démarrer (checksum différent). | Marche à suivre documentée dans le README des migrations : `ALTER TABLE` pour la colonne, puis mise à jour du checksum (valeur vérifiée avec la librairie Flyway). |

#### 3. Sécurité : retour du format SVG

- **`FileStorageService`** : le SVG est de nouveau accepté pour les images de scène, mais nettoyé avant d'être enregistré ; le XML reste refusé ; taille limitée à 5 Mo. Les formats audio ne changent pas (mp3, wav, ogg, webm, m4a, flac…), mais on ne peut pas envoyer un SVG à la place d'un enregistrement audio.
- **`util/SvgSanitizer`** (nouveau) : ne garde que le dessin (formes, couleurs, texte, images intégrées) et supprime tout ce qui pourrait exécuter du code ou charger une ressource externe. Un fichier abîmé ou piégé est refusé. Il lit aussi la largeur et la hauteur du dessin.
- **Frontend** : les sélecteurs de fichiers acceptent `image/svg+xml` et les messages indiquent « PNG, JPG, WebP, GIF, SVG ».

#### 4. Nouveau contrôleur pour enregistrer la vignette en SVG

Avant, une vignette du Vignette Maker était envoyée comme une simple image : impossible de la rouvrir.

| Où | Ce qui a été fait |
|----|-------------------|
| `VignetteApiController` (nouveau) | `POST /api/scenarios/{id}/vignettes` enregistre la vignette (SVG) comme scène, avec son dessin. `GET /api/thumbnails/{id}/vignette` renvoie le dessin. `PUT /api/thumbnails/{id}/vignette` remplace le SVG et le dessin en gardant la position, le titre et les audios. |
| `VignetteService` (nouveau) | Vérifie les droits (`ScenarioAccessService`), n'accepte qu'une image SVG, valide le dessin (objet JSON, 1 Mo max) et écrit l'historique du scénario. |
| `ThumbnailService`, `Thumbnail`, `ThumbnailRowDto`, `ScenarioForkService` | Champ `vignetteSceneJson`, méthode `replaceVignette` (ancien fichier supprimé après commit s'il n'est plus utilisé), champ « vignette » dans la liste des scènes, dessin copié lors d'un fork, largeur et hauteur lues dans le SVG. |
| Base de données | Pas de nouvelle colonne : la ligne `thumbnail` garde le type `image/svg+xml`, le chemin du `.svg`, la largeur, la hauteur et le dessin (`vignette_scene_json`). |
| Vignette Maker (`VignetteMakerModal.vue`) | « Insérer dans le cadre » exporte un SVG (personnages et bulles vectoriels, décor intégré), envoyé au nouvel endpoint (`api/vignettes.js`) avec le dessin. |

#### 5. Téléchargement des scènes : SVG ou ZIP

Aucune page ne permettait de télécharger une scène. Un bouton de téléchargement (`components/ui/DownloadMenu.vue`) a été ajouté partout où l'on voit les scènes, avec deux choix :

- **SVG** (`utils/sceneDownload.js`) : l'image de chaque scène ; une vignette est téléchargée telle quelle, une photo importée est placée dans un SVG de même taille.
- **ZIP** (`utils/scenarioExport.js`, `utils/zip.js`) : un dossier par scène (image SVG + prises audio Take A, Take B…), l'ambiance du scénario et un `scenario.txt` (scènes, prises, transcriptions, traductions). Les audios gardent leur format d'origine ; le ZIP est créé dans le navigateur, sans nouvelle librairie.

Le bouton est présent dans le lecteur de scénario (scène ouverte ou scénario complet), dans le Studio (chaque scène) et sur les cartes de la liste des scénarios et des favoris (`useScenarioDownload.js`). `apiFetch` (`api/rest.js`) accepte maintenant `responseType: "blob"`, avec les mêmes droits que l'affichage.

**Vérification** : un ZIP créé avec de vrais fichiers du serveur s'ouvre dans le Finder, et les fichiers extraits sont identiques aux originaux.

#### 6. Suppression d'un scénario (auteur et admin)

- **Problème** : le serveur autorisait l'admin à supprimer n'importe quel scénario, mais le Studio n'affichait « Delete this scenario » qu'à l'auteur, et cachait même le bouton des réglages à l'admin.
- **Correction** (`ScenarioDetailView.vue`) : nouvelle règle `canDeleteScenario` (auteur ou admin). Le bouton des réglages s'affiche si l'on peut modifier ou supprimer le scénario.
- **Règles serveur inchangées** : seuls l'auteur et un admin peuvent supprimer (pas un collaborateur) ; la suppression retire aussi likes, favoris, collaborateurs, liens d'invitation, historique, scènes, audios et ambiance.
- **Vérification** : sur une copie du serveur avec une base temporaire et un compte `ROLE_USER`, suppression d'un brouillon et d'un scénario publié réussie (204). Seule une requête avec un jeton périmé est refusée (401).

#### 7. Documentation

- README principal réorganisé : démarrage rapide, tableaux (prérequis, terminaux, commandes, profils), doublons retirés, section dépannage
- Un README dans chaque dossier du backend (api, dto, docs, security, config, bootstrap, persistence, model, repo, service, storage, geo, util) et dans `db/migration`
- README des tests backend (`src/test/README.md`) et frontend (`vite/tests/README.md`)
- Fichiers à la racine : `CONTRIBUTORS.md`, `SECURITY.md`, `API_DOCUMENTATION.md` (corrigé), `CODE_OF_CONDUCT.md`

#### 8. Tests

| Sujet | Détail |
|-------|--------|
| Nouveaux tests backend | `SvgSanitizerTest` (23), `FileStorageServiceTest` (SVG accepté et nettoyé, XML refusé), `VignetteServiceTest` (20), `VignetteApiControllerWebMvcTest` (7), `ThumbnailServiceTest` (vignette SVG enregistrée avec sa taille) |
| Nouveaux tests frontend | Enregistrement des vignettes en SVG, `sceneDownload.test.js` (9), export du Vignette Maker avec caractères spéciaux, `zip.test.js` (3), `scenarioExport.test.js` (5), `ScenarioDetailView.test.js` (2) |
| Tests modifiés | `CommunityServiceTest` (nouvelles méthodes de repository), `AuthUtilsTest` (déplacé) |
| Résultat | **389 tests backend** et **205 tests frontend** passent, le build Vite réussit. Les 16 tests d'intégration n'ont pas pu tourner (ils demandent Docker). |

!!! warning "Limites connues"
    - **Décors** : personnages et bulles sont vectoriels, mais le décor reste une image dans le SVG (il faudrait réécrire chaque décor de `vignetteBackgrounds.js` en SVG).
    - **Scènes plus anciennes** : celles créées avant le nouveau contrôleur n'ont pas de dessin enregistré ; leur SVG contient seulement l'image d'origine.
    - **Rouvrir une vignette** : `GET` et `PUT /api/thumbnails/{id}/vignette` existent, mais aucun bouton « Edit vignette » ne les utilise encore.
    - **Connexion après un redémarrage** : la clé qui signe les jetons est recréée à chaque démarrage (`JwtConfig`) ; il faut se reconnecter, sinon les actions protégées répondent 401.

### Prochaines étapes
- Terminer le modèle de données (DTO)
- Rendre la clé de signature des jetons persistante entre deux redémarrages
- Faire tourner les tests d'intégration (avec Docker)
