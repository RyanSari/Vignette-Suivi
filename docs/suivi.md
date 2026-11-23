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
- Prendre en main le projet et le faire tourner en local
- Comprendre l'architecture du backend (contrôleurs, services, repositories, sécurité)
- Faire un état des lieux du backend et lister les problèmes, en particulier ceux liés aux doublons

### Travail réalisé

!!! abstract "Avancement"
    - [x] Installation et exécution du projet (backend Spring Boot + frontend Vite) sur mon poste
    - [x] Mise à jour de versions pour que le projet compile et tourne : Lombok 1.18.36 → 1.18.42 (`pom.xml`) et `package-lock.json` du frontend (commit du 4 novembre)
    - [x] Lecture du backend : authentification, scénarios, médias (images, miniatures, audios), collaborations et invitations, forks et leur review, likes/favoris, notifications, langues (Glottolog)
    - [x] Rédaction du document « Backend Vignette : résumé des problèmes trouvés » (3 pages), organisé en trois sections : doublons de données, code dupliqué, sécurité

#### Démarche d'audit

Pour chaque fonctionnalité, j'ai suivi le chemin d'une requête du contrôleur jusqu'à la base de données, en cherchant :

- les **vérifications faites deux fois** (contrôleur et service) ou incohérentes entre elles (casse, messages d'erreur) ;
- les **conditions de concurrence** du type « vérifier puis insérer » ou `max(idx)+1` sans verrou ;
- le **code copié-collé** et les mécanismes qui font la même chose de plusieurs façons ;
- le **code mort** (méthodes de repository ou de sécurité jamais appelées) ;
- les **failles d'accès** (qui peut voir, modifier ou publier quoi).

Les problèmes trouvés sont listés ci-dessous.

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
- Ajouter des tests pour chaque correction

### Travail réalisé

!!! abstract "Avancement"
    - [x] Correction de tous les problèmes de la section **1.1 (doublons de données)**, avec tests
    - [x] Correction de la section **1.2 (code et logique dupliqués)** : droits, exceptions, utilitaires, copie de fork, création d'utilisateur, pipeline Glottolog Java, configuration OpenAPI, code mort
    - [x] Découpage de `ScenarioService` en services spécialisés (`ScenarioAccessService`, `ScenarioForkService`)
    - [x] Corrections de bugs trouvés en chemin (mot de passe, chargement des tags)
    - [ ] Gestion du schéma de la BD (Flyway / `ddl-auto=update` / `SchemaMaintenance`) : reportée à la semaine 3
    - [ ] Section **1.3 (sécurité et contrôle d'accès)** : reportée à la semaine 3

#### Section 1.1 : doublons de données (corrigée)

| Problème | Correction |
|----------|------------|
| Username / email vérifiés deux fois, sensibles à la casse | La vérification n'est plus que dans `UserService.register` (le contrôleur l'appelle simplement). Le username est nettoyé (`trim`), l'email est mis en minuscules, et l'unicité utilise `existsByUsernameIgnoreCase` / `existsByEmailIgnoreCase` : `Alice` et `alice` sont le même compte. Un seul message : « username already used » / « email already used ». |
| Titres de scénario : casse, renommage par un éditeur, duplication contrôleur/service | `existsByTitleIgnoreCaseAndAuthorUsernameAndLanguageId` partout ; la vérification est faite sur l'**auteur** du scénario (et non sur la personne qui renomme) ; la vérification en double de `ScenarioApiController` est retirée. Pour un fork, `saveAndFlush` + capture de `DataIntegrityViolationException` : si deux requêtes passent la vérification en même temps, la seconde reçoit une erreur claire au lieu de planter. |
| Invitations DECLINED / EXPIRED bloquantes | La ligne existante (unique sur `scenario_id + user_id`) est **réutilisée** pour réinviter ; seule une invitation ACCEPTED ou PENDING non expirée bloque une nouvelle invitation. |
| Likes, favoris, suivi de langue | Vérification de visibilité (`getVisibleScenario`, mêmes règles que la page du scénario : brouillons et scénarios privés inaccessibles). L'insertion se fait dans sa propre transaction avec `saveAndFlush` ; si une requête simultanée (double clic, deux onglets) a déjà inséré la ligne, l'erreur de contrainte unique est ignorée, car l'état final est celui voulu. |
| `idx` des images et audios | Verrou de ligne `SELECT … FOR UPDATE` sur le scénario (images) ou la scène (audios) avant de calculer `max(idx)+1` : les uploads simultanés passent l'un après l'autre. |
| Même image uploadée plusieurs fois | Le stockage est adressé par le contenu (chemin dérivé du SHA-256) ; `existsByScenarioIdAndImageSha256` refuse la même image dans le même scénario (« This image is already used in this scenario »), et le fichier orphelin éventuel est supprimé. |
| `maxUses` des liens d'invitation | `findByTokenForUpdate` avec `@Lock(PESSIMISTIC_WRITE)` : les utilisations simultanées vérifient et incrémentent `useCount` l'une après l'autre. |
| Notifications en double | Nouvelle méthode `saveAndPush` utilisée pour toutes les notifications : une notification identique (même destinataire, type, référence et message) encore **non lue** n'est pas recréée. Une fois lue, une nouvelle occurrence est de nouveau notifiée (ex. réinvitation après un refus). |

Tests ajoutés : `ScenarioInteractionServiceTest` (+139 lignes), `ScenarioCollaborationServiceTest`, `LanguageFollowServiceTest`, `NotificationServiceTest`, `ThumbnailServiceTest`, `AudioServiceTest`, `UserServiceTest`.

#### Section 1.2 : code et logique dupliqués (corrigée, sauf schéma de BD)

| Problème | Correction |
|----------|------------|
| Deux gestionnaires d'exceptions | `RestExceptionHandler` (vide) supprimé ; seul `GlobalApiExceptionHandler` reste. |
| Quatre mécanismes de contrôle des droits | Suppression de l'aspect `OwnerOrAdminAspect` et des classes `ScenarioSecurity`, `ThumbnailSecurity`, `AudioSecurity`, `OwnershipSecurityService(Impl)`. `@OwnerOrAdminOperation` ne sert plus qu'à la documentation Swagger. Les droits sont vérifiés à **un seul endroit** : d'abord `ScenarioService.assertCan…`, puis le nouveau `ScenarioAccessService` (« single source of truth » : voir / modifier / publier / supprimer). Un scénario qu'on n'a pas le droit de voir est renvoyé comme « introuvable » pour ne pas révéler son existence. |
| `assertAdmin()` maison dans `AdminApiController` | Supprimé (16 appels retirés) au profit du mécanisme admin existant. |
| `isAdmin`, `requireAuthenticated`, `authenticatedUsername` copiés dans 5 ou 6 services | Regroupés dans une nouvelle classe `AuthUtils` (avec `AuthUtilsTest`). |
| Copie de fork faite à la main, incomplète | Deux méthodes communes `copyThumbnail` et `copyAudio` : tous les champs de contenu sont copiés (dont `sourceLabel`, volume et boucle de fond), et `activeBackgroundAudioId` pointe vers la copie de la musique de fond. Le fichier physique est partagé (stockage adressé par le contenu), seule une nouvelle ligne est créée. La logique est ensuite déplacée dans `ScenarioForkService`. |
| `BootstrapData` refait `UserService.register` | `BootstrapData` passe par `UserService.register` (même normalisation, unicité et hachage du mot de passe), puis `UserService.grantAdminRole`. Test : `BootstrapDataTest`. |
| Deux pipelines Glottolog | Pipeline Java supprimé (`LanguageImportService`, `GlottologUpdateService`, `GlottologUpdateJobService`, `GlottologScheduledUpdateService`, le `languoid.csv` de 27 000 lignes et leurs tests) : **−29 622 lignes**. Seul le worker Python reste ; la doc `glottolog-import.md` et le tableau de bord admin sont mis à jour. |
| Deux configurations OpenAPI | `SwaggerConfig` fusionné dans `OpenApiConfig`. Les réponses d'erreur sont documentées par des annotations réutilisables (`@BadRequestResponse`, `@UnauthorizedResponse`, `@ForbiddenResponse`, `@NotFoundResponse`, `@ConflictResponse`) au lieu d'être répétées sur chaque endpoint. Test : `OpenApiDocsIntegrationTest`. |
| Code mort dans les repositories | Méthodes inutilisées retirées de `ScenarioRepository`, `AudioRepository`, `ThumbnailRepository`, `ScenarioTagRepository`. |
| `ScenarioService` trop gros | Découpé : `ScenarioAccessService` (droits, +139 lignes) et `ScenarioForkService` (forks et leur review, +252 lignes) ; `ScenarioService` passe de 842 à 510 lignes. Les contrôleurs sont allégés (ex. `ScenarioApiController` −239 lignes, `CommunityApiController` −170 lignes). |
| Schéma de BD géré à trois endroits | ⏳ Reporté à la semaine 3. |

!!! note "Reste à faire"
    Les deux systèmes de notifications (`Notification` et `GlottologNotification`) existent toujours séparément.

#### Bugs corrigés en chemin

- **Mot de passe** : le frontend acceptait 6 caractères alors que le backend en exige 8 (`RegisterView.vue`).
- **`LazyInitializationException`** sur `Scenario.tags` (« could not initialize proxy - no Session ») : la liste des scénarios charge maintenant les tags dans la même requête (`findAllWithTagsOrderByCreatedAtDesc`).
- **Renommage** : la vérification de titre utilisait le nom de la personne connectée au lieu de celui de l'auteur.
- Mise à jour du plugin JaCoCo (0.8.12 → 0.8.14).

#### Commits de la semaine

| Date | Commit | Contenu |
|------|--------|---------|
| 6 nov | Enforce case-insensitive username and email uniqueness | Unicité username/email |
| 6 nov | Front and back passwords weren't the same | Mot de passe 8 caractères |
| 6 nov | Update ScenarioService.java | `LazyInitializationException` sur les tags |
| 7 nov | Scenario title | Titres de scénario |
| 7 nov | End of Data Duplicates | Fin de la section 1.1 (18 fichiers, +546 lignes) |
| 8 nov | Removes duplicates / same code | Droits, exceptions, `AuthUtils` (30 fichiers, −269 lignes nettes) |
| 8 nov | Remove forks | Copie de fork complète |
| 9 nov | Removed java pipeline in Glottolog + register duplicates | Pipeline Glottolog Java, `BootstrapData` |
| 10 nov | Removed duplicates + divided java class | Découpage de `ScenarioService`, OpenAPI, repositories |

### Prochaines étapes
- Unifier la gestion du schéma de la BD
- Corriger les problèmes de sécurité de la section 1.3

---

## Semaine 3 (13 – 19 novembre)

### Objectifs de la période
- Terminer les points restants du document d'analyse : d'abord regrouper la base de données en une seule, puis passer à la sécurité
- Continuer le refactoring (services trop gros, utilitaires)
- Commencer le modèle de données (DTO) et la gestion des priorités
- Évaluer l'usage de Supabase

### Travail réalisé

!!! abstract "Avancement"
    - [x] Base de données regroupée en une seule : PostgreSQL partout (local, tests, Supabase en production), schéma géré uniquement par Flyway
    - [x] Sécurité (section 1.3 du document)
    - [x] Refactoring : nouveaux services spécialisés, `NotificationService` et `CountryGeoService` allégés
    - [x] Utils : nouvelle classe `util/Text`
    - [x] Priority
    - [x] Étude et mise en place de Supabase pour la production
    - [x] Intégration de la branche `lea_studio` (PR #3)
    - [ ] Modèle de données (DTO) ⏳ en cours

#### Base de données : une seule base, un seul responsable du schéma

| Avant | Après |
|-------|-------|
| H2 (fichier ou mémoire) en dev et en test, PostgreSQL avec le profil `pgdev`. | **PostgreSQL partout** : en local (profil `dev`, `pgdev` gardé comme alias), dans les tests (Testcontainers : un PostgreSQL temporaire dans Docker, `PostgresTestContainerConfig`) et en production (Supabase). |
| Schéma géré à trois endroits : migrations Flyway, `ddl-auto=update` et classes `AudioSchemaMaintenance`, `ScenarioSchemaMaintenance`, `ThumbnailSchemaMaintenance`. | **Flyway seul** ; Hibernate est en `ddl-auto=validate` (il vérifie que les entités correspondent au schéma, sans le modifier). Les trois classes `SchemaMaintenance` et leurs tests sont supprimées. |
| Une vingtaine de migrations dispersées (`V1`–`V16` dans `common/`, `V5`–`V9` Glottolog). | Deux migrations : `V1__baseline.sql` (schéma complet, généré par `pg_dump`) et `V2__indexes_and_glottolog_wake_trigger.sql`. |
| Bases locales créées avant Flyway (par `ddl-auto=update`). | `baseline-on-migrate` en dev : une base existante est marquée « déjà en V1 » au lieu de faire échouer le démarrage. En production, ce cas est refusé sauf avec `FLYWAY_BASELINE_ON_MIGRATE=true`. |

Le README est réécrit en conséquence (prérequis PostgreSQL 15+, Docker pour les tests, tableau des profils, section « Deploying on Supabase »), et un workflow GitHub `publish-api-docs.yml` publie la documentation de l'API.

#### Étude de Supabase

Supabase est un PostgreSQL hébergé : il peut donc remplacer la base de production sans changer de moteur, à condition que le schéma soit géré par Flyway. C'est ce qui a motivé le passage à PostgreSQL partout. Le profil `prod` lit la connexion dans `DB_URL`, `DB_USERNAME` et `DB_PASSWORD`, ce qui permet de pointer vers Supabase.

#### Section 1.3 : sécurité et contrôle d'accès

| Problème | Correction |
|----------|------------|
| `Alice` / `alice` voient les brouillons l'une de l'autre | Les requêtes « mes scénarios » ne comparent plus avec `lower()` ; avec l'unicité insensible à la casse de la semaine 2, deux comptes ne peuvent plus se confondre. |
| SVG / XML acceptés (XSS stockée) | `FileStorageService` n'accepte plus que des images matricielles et de l'audio : SVG et XML sont refusés, car ils peuvent contenir des scripts et les fichiers sont renvoyés tels quels. Tests ajoutés dans `FileStorageServiceTest`. (Le SVG est réintroduit de façon sécurisée en semaine 4.) |
| Invitation expirée encore valide en lecture | `ScenarioAccessService` ne compte qu'une invitation PENDING **non expirée** (pas de date = n'expire pas). |
| Approbation des forks contournable | Toute modification d'un fork APPROVED le renvoie en review (`sendApprovedForkBackToReview`) : statut PENDING, retour en DRAFT s'il était publié, et l'auteur de l'original reçoit une notification pour le relire. |
| Identifiants admin en dur dans `application-pgdev.properties` | Le profil `pgdev` est fusionné dans `dev`, et les identifiants sont maintenant configurables par variables d'environnement (`APP_BOOTSTRAP_ADMIN_*`). ⚠️ `admin` / `admin12345` restent les valeurs par défaut en dev. |

#### Refactoring

Objectif : des contrôleurs minces et des services qui ont chacun une seule responsabilité.

| Nouveau | Rôle |
|---------|------|
| `ScenarioDeletionService` | Supprime un scénario avec tout ce qui y est rattaché (likes, favoris, collaborateurs, liens d'invitation, historique, scènes, audios). Un fichier n'est retiré du stockage que si aucune autre ligne ne l'utilise (un fork partage les fichiers de l'original). |
| `ThumbnailManagementService` | Opérations sur les images de scène pour un utilisateur : vérifie les droits, appelle `ThumbnailService` et écrit l'historique du scénario. `ThumbnailApiController` passe de 434 à 334 lignes. |
| `AudioManagementService` | Même rôle pour les audios ; `AudioApiController` est allégé d'autant. |
| `NotificationStream` | Envoi en direct des notifications aux navigateurs connectés (Server-Sent Events), séparé de `NotificationService` qui décide quoi notifier et à qui. |
| `geo/TopoJsonDecoder`, `geo/CountryShape` | Décodage des frontières des pays (TopoJSON) sorti de `CountryGeoService`, qui passe de 395 à 137 lignes. |
| `util/Text` | Nettoyage et vérification de longueur des textes saisis par les utilisateurs, partagés par tous les services (avec `TextTest`). |

`ScenarioService` est réorganisé en sections (création, recherche, listes, administration, publication, modification, validation, DTO).

#### Modèle de données (DTO) et Priority

- Premiers DTO retravaillés : `AdminUserRowDto`, `LanguageRowDto`, `ThumbnailRowDto` (⏳ en cours).
- Priority (commit « Added prio », 16 novembre) : nouvelle classe de tests `AudioManagementServiceTest` (+264 lignes) et allègement des tests des contrôleurs audio et image, qui testent maintenant seulement le contrôleur.

!!! success "Bilan du document d'analyse"
    À la fin de la semaine 3, les sections 1.1, 1.2 et 1.3 du document « Backend Vignette : résumé des problèmes trouvés » sont traitées, y compris le schéma de la BD reporté de la semaine 2. Restent : les deux systèmes de notifications séparés et les identifiants admin par défaut en dev.

#### Commits de la semaine

| Date | Commit | Contenu |
|------|--------|---------|
| 15 nov | Database merged into one (Supabase / Postgre) | PostgreSQL partout, Flyway seul, migrations regroupées (42 fichiers) |
| 15 nov | Merge pull request #3 from ceduni/lea_studio | Intégration de la branche Studio |
| 15 nov | Refactoring + security | Section 1.3 et nouveaux services (44 fichiers, +1 837 / −1 360) |
| 16 nov | Added prio | Tests `AudioManagementService` |

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

Le travail de la semaine est résumé dans le document « Backend Vignette : travail réalisé ». Cette série de changements couvre :

- l'organisation du code (`util/`, repositories) ;
- la base de données ;
- le retour du format SVG ;
- un nouveau contrôleur pour enregistrer les vignettes en SVG ;
- le téléchargement des scènes (SVG, ou ZIP avec l'audio) ;
- la suppression d'un scénario par un admin ;
- la documentation et les tests.

!!! abstract "Avancement"
    - [x] **Organisation du code** : `AuthUtils` déplacé dans `util/`, `firstNonBlank` et `trimmedOrEmpty` regroupés dans `util/Text`, constante `ROLE_ADMIN`, méthodes de repository à 2 paramètres maximum (`AccreditationScope`)
    - [x] **Base de données** : `V1__baseline.sql` réécrit à la main (667 → 393 lignes, 59 → 0 `ALTER TABLE`), migration V3 supprimée, procédure documentée pour les bases existantes
    - [x] **Retour du format SVG** sécurisé : nouveau nettoyeur `SvgSanitizer`, XML toujours refusé, limite de 5 Mo, frontend mis à jour
    - [x] **Nouveau contrôleur vignette** : `VignetteApiController` (POST / GET / PUT) + `VignetteService` + dessin stocké en BD (`vignette_scene_json`) + Vignette Maker qui exporte en SVG
    - [x] **Téléchargement des scènes** en SVG ou en ZIP (image + prises audio + ambiance + `scenario.txt`), depuis le lecteur, le Studio, la liste des scénarios et les favoris
    - [x] **Suppression d'un scénario** par son auteur ou par un admin depuis le Studio (`canDeleteScenario`)
    - [x] **Documentation** : README principal réorganisé, un README par dossier du backend, README des tests, `CONTRIBUTORS.md`, `SECURITY.md`, `API_DOCUMENTATION.md`, `CODE_OF_CONDUCT.md`
    - [x] **Tests** : plus de 60 nouveaux tests ; 389 tests backend et 205 tests frontend passent, le build Vite réussit
    - [x] Rédaction du document « Backend Vignette : travail réalisé » (6 pages)
    - [x] Site de suivi du projet (à partir du [template IFT3150](https://github.com/udem-diro/template-projet))
    - [ ] Étoffer le suivi de projet ⏳ en cours
    - [ ] Modèle de données (DTO) ⏳ en cours (reporté de la semaine 3)

#### 1. Organisation du code

Objectif : ranger tous les utilitaires au même endroit (`util/`), supprimer les copies de méthodes et simplifier les méthodes de repository trop longues (2 ou 3 paramètres maximum).

| Où | Avant | Ce qui a été fait |
|----|-------|-------------------|
| `security/AuthUtils.java` | Classe utilitaire rangée dans le package `security`, à part des autres utilitaires. | Déplacée dans `util/` avec son test (`AuthUtilsTest`). Les imports des contrôleurs et services sont mis à jour. |
| `CountryGeoService.java`, `OpenApiConfig.java` | La méthode `firstNonBlank` était copiée dans les deux classes. | Une seule version, dans `util/Text`. |
| `BootstrapData.java` | Méthode privée `trimmedOrEmpty`. | Déplacée dans `util/Text` pour être réutilisable. |
| `LanguageService.java`, `UserService.java` | Le rôle `"ROLE_ADMIN"` écrit en dur. | Utilisation de la constante `AuthUtils.ROLE_ADMIN`. |
| `AccreditationRequestRepository`, `CommunityAccreditationRepository` | Méthodes à 4 ou 5 paramètres, ex. `existsByRequestedByUserIdAndPermissionTypeAndScopeTypeAndTargetIdAndStatus`. | Nouveau record `AccreditationScope` (permission, portée, cible). Les services appellent des méthodes à 2 paramètres maximum : `listByScope`, `existsPending`, `findGrant`, `hasGrant`. Le cas GLOBAL, qui était dupliqué dans `CommunityService`, est géré une seule fois. |

#### 2. Base de données

Objectif : avoir une migration de base lisible et maintenable, sans migration supplémentaire inutile.

| Où | Avant | Ce qui a été fait |
|----|-------|-------------------|
| `V1__baseline.sql` | Généré par `pg_dump` : 667 lignes, 59 `ALTER TABLE`, options inutiles (`NO MINVALUE`, `CACHE 1`, préfixe `public.`). | Réécrit à la main : **393 lignes, aucun `ALTER`**. Clés primaires, contraintes `UNIQUE`, clés étrangères et colonnes `IDENTITY` sont déclarées dans chaque `CREATE TABLE`, les `CHECK` sont écrits en `IN (...)`, et les tables sont classées par domaine. |
| Colonne `vignette_scene_json` | Ajoutée d'abord avec une migration V3 contenant un `ALTER TABLE`. | Mise directement dans le `CREATE TABLE thumbnail` de V1, et V3 supprimée. |
| Bases existantes | Comme V1 a changé, Flyway refuse de démarrer (checksum différent). | Marche à suivre documentée dans le README des migrations : `ALTER TABLE` pour ajouter la colonne, puis mise à jour du checksum (valeur vérifiée avec la librairie Flyway). |

#### 3. Sécurité : retour du format SVG

En semaine 3, le SVG avait été refusé pour corriger la faille XSS. Cette semaine, il est de nouveau accepté, mais de façon sécurisée.

| Où | Avant | Ce qui a été fait |
|----|-------|-------------------|
| `FileStorageService.java` | Le SVG était refusé, car un fichier SVG peut contenir du code caché. | Le SVG est de nouveau accepté pour les images de scène, mais il est **nettoyé avant d'être enregistré**. Le XML reste refusé. Taille limitée à 5 Mo. L'audio ne change pas : tous les formats audio (mp3, wav, ogg, webm, m4a, flac…) sont toujours acceptés ; on ne peut simplement pas envoyer un fichier SVG (une image) à la place d'un enregistrement audio. |
| `util/SvgSanitizer.java` (nouveau) | Un SVG peut cacher du code qui s'exécute chez la personne qui l'ouvre. | Le nettoyeur ne garde que le dessin : formes, couleurs, texte et images intégrées. Il supprime tout ce qui pourrait exécuter du code ou charger quelque chose depuis un autre site. Un fichier abîmé ou piégé est refusé. Il sait aussi lire la largeur et la hauteur du dessin. |
| Frontend | Les fichiers `.svg` étaient filtrés. | Les sélecteurs de fichiers acceptent `image/svg+xml` et les messages indiquent « PNG, JPG, WebP, GIF, SVG ». |

#### 4. Nouveau contrôleur pour enregistrer la vignette en SVG ✔

Objectif : qu'une vignette créée dans le Vignette Maker soit enregistrée en SVG **avec son dessin**, pour pouvoir la rouvrir et la modifier plus tard.

| Où | Avant | Ce qui a été fait |
|----|-------|-------------------|
| `VignetteApiController.java` (nouveau) | Une vignette du Vignette Maker était envoyée comme une simple image : impossible de la rouvrir. | `POST /api/scenarios/{id}/vignettes` enregistre la vignette (fichier SVG) comme scène, avec son dessin.<br>`GET /api/thumbnails/{id}/vignette` renvoie le dessin.<br>`PUT /api/thumbnails/{id}/vignette` remplace le SVG et le dessin en gardant la position, le titre et les audios. |
| `VignetteService.java` (nouveau) | — | Vérifie les droits (`ScenarioAccessService`), n'accepte qu'une image SVG, valide le dessin (objet JSON, 1 Mo max) et écrit l'historique du scénario. |
| `ThumbnailService`, `Thumbnail`, `ThumbnailRowDto`, `ScenarioForkService` | Une scène ne gardait que l'image. | Champ `vignetteSceneJson` ; méthode `replaceVignette` (l'ancien fichier est supprimé après le commit s'il n'est plus utilisé) ; champ « vignette » dans la liste des scènes ; dessin copié lors d'un fork. La largeur et la hauteur d'un SVG sont lues dans le fichier. |
| Base de données | — | Pas de nouvelle colonne nécessaire : la ligne `thumbnail` garde le type `image/svg+xml`, le chemin du fichier `.svg`, la largeur, la hauteur et le dessin (`vignette_scene_json`). |
| Vignette Maker (`VignetteMakerModal.vue`) | Le dessin était perdu à l'insertion. | « Insérer dans le cadre » exporte un SVG : personnages et bulles restent vectoriels, le décor est intégré. Il est envoyé au nouvel endpoint (`api/vignettes.js`) avec le dessin. |

#### 5. Téléchargement des scènes : SVG ou ZIP ✔

Avant, aucune page ne permettait de télécharger une scène. Un bouton de téléchargement a été ajouté partout où l'on voit les scènes. Il ouvre un petit menu avec deux choix : **SVG** (l'image seulement) ou **ZIP** (l'image et tous les sons).

| Où | Avant | Ce qui a été fait |
|----|-------|-------------------|
| Choix SVG (`utils/sceneDownload.js`) | — | Télécharge l'image de chaque scène en SVG. Une vignette est déjà un SVG : elle est téléchargée telle quelle. Une photo importée est placée dans un SVG de même taille. |
| Choix ZIP (`utils/scenarioExport.js`, `utils/zip.js`) | Un SVG ne peut pas contenir de son : l'audio n'était pas récupérable. | Un fichier `.zip` avec un dossier par scène (l'image SVG et ses prises audio : Take A, Take B…), l'ambiance du scénario, et un fichier `scenario.txt` qui liste les scènes, les prises, les transcriptions et les traductions. Les audios gardent leur format d'origine. Le ZIP est créé dans le navigateur, **sans nouvelle librairie**. |
| Menu de téléchargement (`components/ui/DownloadMenu.vue`) | — | Composant réutilisé partout : un bouton, puis le choix SVG ou ZIP. Se ferme avec Échap ou en cliquant ailleurs. |
| `api/rest.js` | `apiFetch` ne renvoyait que du JSON ou du texte. | Option `responseType: "blob"` pour récupérer un fichier (image ou audio). Les droits d'accès sont ceux de l'affichage. |
| Lecteur de scénario | Pas de bouton. | Bouton « Download » : la scène ouverte, ou tout le scénario depuis la grille. |
| Studio | Pas de bouton. | Bouton sur chaque scène, à côté de la corbeille : la scène seule (SVG, ou ZIP avec ses prises). |
| Liste des scénarios et favoris | Pas de bouton. | Bouton sur chaque carte : tout le scénario (composable `useScenarioDownload.js`). |
| Vérification | — | Un ZIP créé avec de vrais fichiers du serveur (image d'une scène et audio d'ambiance) s'ouvre dans le Finder ; les fichiers extraits sont identiques aux originaux. |

#### 6. Suppression d'un scénario (auteur et admin) ✔

| Où | Avant | Ce qui a été fait |
|----|-------|-------------------|
| `ScenarioDetailView.vue` (Studio) | Le serveur autorisait l'admin à supprimer n'importe quel scénario, mais le Studio n'affichait le bouton « Delete this scenario » qu'à l'auteur. Pour un admin, même le bouton des réglages était caché : impossible de supprimer. | Nouvelle règle `canDeleteScenario` (auteur ou admin). Le bouton des réglages s'affiche si l'on peut modifier **ou** supprimer le scénario, et la zone de suppression s'affiche selon `canDeleteScenario`. Chemin pour l'admin : onglet Storyboard → réglages → « Delete this scenario ». |
| Règles côté serveur (inchangées) | — | Seuls l'auteur et un admin peuvent supprimer un scénario ; un collaborateur ne peut pas. La suppression retire aussi les likes, favoris, collaborateurs, liens d'invitation, historique, scènes, audios et ambiance. |
| Où l'auteur supprime | — | Page « My scenarios » (bouton Delete sur ses propres scénarios) et Studio (onglet Storyboard → réglages → « Delete this scenario »). L'auteur n'a pas besoin d'être admin. |
| Vérification | Un créateur non admin semblait ne pas pouvoir supprimer son scénario. | Testé sur une copie du serveur avec une base temporaire et un compte normal (`ROLE_USER`) : suppression d'un brouillon et d'un scénario publié avec une scène réussies (**204**). Seule une requête avec un jeton de connexion périmé est refusée (**401**) : c'est le cas après un redémarrage du serveur (voir Limites connues). |

#### 7. Documentation

| Sujet | Ce qui a été fait |
|-------|-------------------|
| README principal | Réorganisé : démarrage rapide en haut, tableaux (prérequis, terminaux, commandes, profils), doublons retirés (les URL Swagger apparaissaient 3 fois), section dépannage. |
| README par dossier | Un README dans chaque dossier du backend (`api`, `dto`, `docs`, `security`, `config`, `bootstrap`, `persistence`, `model`, `repo`, `service`, `storage`, `geo`, `util`) et dans `db/migration`. |
| README des tests | `src/test/README.md` (backend) et `vite/tests/README.md` (frontend) : commandes, types de tests, organisation, comment écrire un test. |
| Fichiers à la racine | `CONTRIBUTORS.md` (équipe et contributions), `SECURITY.md` (signalement, protections en place), `API_DOCUMENTATION.md` (corrigé : publication sur `master`), `CODE_OF_CONDUCT.md` (contact par email). |

#### 8. Tests

| Sujet | Ce qui a été fait |
|-------|-------------------|
| Nouveaux tests backend | `SvgSanitizerTest` (23), `FileStorageServiceTest` (SVG accepté et nettoyé, XML refusé), `VignetteServiceTest` (20, dont le refus des images qui ne sont pas en SVG), `VignetteApiControllerWebMvcTest` (7), `ThumbnailServiceTest` (vignette SVG enregistrée avec sa taille). |
| Nouveaux tests frontend | Enregistrement des vignettes en SVG ; téléchargement en SVG (`sceneDownload.test.js`, 9) ; export du Vignette Maker valide même avec des caractères spéciaux dans une bulle ; création du ZIP (`zip.test.js`, 3) et contenu du ZIP : dossiers, prises, ambiance, `scenario.txt` (`scenarioExport.test.js`, 5) ; Studio : un admin peut supprimer le scénario d'un autre, un simple lecteur ne voit pas le bouton (`ScenarioDetailView.test.js`, 2). |
| Tests modifiés | `CommunityServiceTest` (nouvelles méthodes de repository), `AuthUtilsTest` (déplacé). |
| Résultat | **389 tests backend** et **205 tests frontend** passent, le build Vite réussit. Les 16 tests d'intégration n'ont pas pu tourner : ils demandent Docker. |

#### 9. Limites connues

| Sujet | Détail |
|-------|--------|
| Décors | Personnages et bulles sont vectoriels, mais le décor reste une image dans le SVG. Pour qu'il soit vectoriel, il faudrait réécrire chaque décor de `vignetteBackgrounds.js` en SVG. |
| Scènes plus anciennes | Les scènes créées avant le nouveau contrôleur n'ont pas de dessin enregistré : leur SVG contient seulement l'image d'origine. |
| Rouvrir une vignette | `GET` et `PUT /api/thumbnails/{id}/vignette` existent, mais aucun bouton « Edit vignette » ne les utilise encore dans l'interface. |
| Connexion après un redémarrage | La clé qui signe les jetons de connexion est recréée à chaque démarrage du serveur (`JwtConfig`). Après un redémarrage, il faut se reconnecter, sinon les actions protégées (supprimer, modifier) répondent 401. |

### Prochaines étapes
- Terminer le modèle de données (DTO)
- Rendre la clé de signature des jetons persistante entre deux redémarrages
- Faire tourner les tests d'intégration (avec Docker)
