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
- Faire un état des lieux du backend

### Travail réalisé

!!! abstract "Avancement"
    - [x] Projet installé et lancé en local (mise à jour de Lombok 1.18.36 → 1.18.42 pour qu'il compile)
    - [x] Lecture du backend : auth, scénarios, médias, collaborations, forks, likes/favoris, notifications, Glottolog
    - [x] Rédaction du document « Backend Vignette : résumé des problèmes trouvés » (3 pages)

| Section | Principaux problèmes trouvés |
|---------|------------------------------|
| **1.1 Doublons de données** | Username/email et titres vérifiés deux fois et sensibles à la casse (`Alice` ≠ `alice`) ; invitations refusées ou expirées impossibles à renvoyer ; conditions de concurrence sur les likes/favoris, l'`idx` des médias, `maxUses` des liens d'invitation et les notifications ; même image uploadable plusieurs fois. |
| **1.2 Code dupliqué** | Deux gestionnaires d'exceptions ; droits vérifiés de 4 façons différentes ; `isAdmin` / `requireAuthenticated` copiés dans 5-6 services ; copie de fork incomplète ; schéma de BD géré à 3 endroits ; deux pipelines Glottolog ; méthodes de repository jamais utilisées. |
| **1.3 Sécurité** | SVG/XML acceptés (XSS stockée) ; invitation expirée encore valide ; approbation des forks contournable ; identifiants admin en dur. |

---

## Semaine 2 (6 – 12 novembre)

### Objectifs de la période
- Corriger les sections 1.1 (doublons de données) et 1.2 (code dupliqué) du document

### Travail réalisé

!!! abstract "Avancement"
    - [x] Section **1.1** corrigée, avec tests
    - [x] Section **1.2** corrigée, sauf le schéma de BD
    - [x] Bugs trouvés en chemin : mot de passe (6 caractères côté front, 8 côté back), `LazyInitializationException` sur les tags
    - [ ] Schéma de BD et section 1.3 → reportés à la semaine 3

| Sujet | Ce qui a été fait |
|-------|-------------------|
| Username / email / titres | Une seule vérification, dans le service, insensible à la casse (`…IgnoreCase`) ; le titre est vérifié chez l'auteur du scénario. |
| Concurrence | Verrous `FOR UPDATE` pour l'`idx` des médias et `maxUses` ; likes/favoris/suivi de langue tolèrent un double clic ; notifications identiques non lues pas recréées. |
| Invitations | Une invitation refusée ou expirée peut être renvoyée. |
| Images en double | Refusées dans un même scénario (comparaison par SHA-256). |
| Droits | Un seul endroit : `ScenarioAccessService`. Aspect et classes `*Security` supprimés, `AuthUtils` remplace les copies. |
| Forks | Copie complète (musique de fond, `sourceLabel`, volume), dans `ScenarioForkService`. |
| Nettoyage | `RestExceptionHandler` et `SwaggerConfig` supprimés ; pipeline Glottolog Java supprimé (−29 600 lignes) ; `BootstrapData` passe par `UserService.register` ; `ScenarioService` passe de 842 à 510 lignes. |

---

## Semaine 3 (13 – 19 novembre)

### Objectifs de la période
- Regrouper la base de données en une seule, puis corriger la sécurité (1.3)
- Continuer le refactoring, commencer le modèle de données (DTO)
- Évaluer l'usage de Supabase

### Travail réalisé

!!! abstract "Avancement"
    - [x] Base de données regroupée en une seule
    - [x] Sécurité (section 1.3)
    - [x] Refactoring, Utils, Priority
    - [x] Étude de Supabase
    - [ ] Modèle de données (DTO) ⏳ en cours

| Sujet | Ce qui a été fait |
|-------|-------------------|
| Base de données | PostgreSQL partout (local, tests avec Testcontainers, Supabase en production). Flyway est seul à gérer le schéma (`ddl-auto=validate`), les classes `SchemaMaintenance` sont supprimées, une vingtaine de migrations regroupées en 2. |
| Supabase | C'est un PostgreSQL hébergé : il sert de base de production sans changer de moteur. |
| Sécurité | SVG/XML refusés ; invitation expirée = plus d'accès ; modifier un fork approuvé le renvoie en review ; identifiants admin configurables par variables d'environnement. |
| Refactoring | Nouveaux services `ScenarioDeletionService`, `ThumbnailManagementService`, `AudioManagementService`, `NotificationStream` ; `CountryGeoService` passe de 395 à 137 lignes ; nouvelle classe `util/Text`. |

!!! note "Reste à faire"
    Les deux systèmes de notifications (`Notification` / `GlottologNotification`) existent encore, et `admin` / `admin12345` restent les valeurs par défaut en dev.

---

## Semaine 4 (20 – 26 novembre)

### Objectifs de la période
- Réorganiser le code (`util/`, méthodes de repository à 2-3 paramètres max)
- Remettre le format SVG de façon sécurisée
- Créer un contrôleur pour enregistrer les vignettes en SVG
- Mettre en place le site de suivi

### Travail réalisé

!!! abstract "Avancement"
    - [x] Organisation du code et base de données
    - [x] Retour du SVG sécurisé
    - [x] Nouveau contrôleur vignette SVG
    - [x] Téléchargement des scènes (SVG ou ZIP)
    - [x] Suppression d'un scénario par un admin
    - [x] Documentation et tests
    - [x] Site de suivi du projet (à partir du [template IFT3150](https://github.com/udem-diro/template-projet))
    - [ ] Modèle de données (DTO) ⏳ en cours

| Sujet | Ce qui a été fait |
|-------|-------------------|
| Organisation du code | `AuthUtils` déplacé dans `util/`, `firstNonBlank` et `trimmedOrEmpty` dans `util/Text`. Repositories d'accréditation : méthodes à 2 paramètres max grâce au record `AccreditationScope`. |
| Base de données | `V1__baseline.sql` réécrit à la main : 667 → 393 lignes, plus aucun `ALTER TABLE`. |
| Retour du SVG | Nouveau `SvgSanitizer` : ne garde que le dessin, supprime tout ce qui peut exécuter du code. XML toujours refusé, 5 Mo max. |
| Contrôleur vignette | `VignetteApiController` (POST / GET / PUT) + `VignetteService` : la vignette du Vignette Maker est enregistrée en SVG **avec son dessin** (`vignette_scene_json`), pour pouvoir la rouvrir. |
| Téléchargement | Bouton partout où l'on voit les scènes : SVG (image) ou ZIP (image + prises audio + ambiance + `scenario.txt`), créé dans le navigateur. |
| Suppression | Un admin peut maintenant supprimer n'importe quel scénario depuis le Studio (`canDeleteScenario`). |
| Documentation | README principal réorganisé, un README par dossier du backend, `CONTRIBUTORS.md`, `SECURITY.md`… |
| Tests | 389 tests backend et 205 frontend passent (les 16 tests d'intégration demandent Docker). |

!!! warning "Limites connues"
    - Le décor des vignettes reste une image dans le SVG.
    - Pas encore de bouton « Edit vignette » dans l'interface.
    - Après un redémarrage du serveur, il faut se reconnecter (clé JWT recréée).

### Prochaines étapes
- Terminer le modèle de données (DTO)
- Rendre la clé de signature des jetons persistante entre deux redémarrages
- Faire tourner les tests d'intégration (avec Docker)
