---
title: Références
---

<style>
    @media screen and (min-width: 76em) {
        .md-sidebar--primary {
            display: none !important;
        }
    }
</style>

# Références

Cette page rassemble les principales sources et ressources utilisées dans le cadre du projet, et précise comment elles ont contribué au travail réalisé.

## Références utilisées

### Documentation technique

**Spring Framework.** *Reference Documentation* (gestion des exceptions).
https://docs.spring.io/spring-framework/reference/

Utilisée pour comprendre le fonctionnement de l'aspect `@OwnerOrAdminOperation`.

**Spring Security.** *Reference Documentation*.
https://docs.spring.io/spring-security/reference/

Utilisée pour regrouper les 4 mécanismes de contrôle des droits en un seul et corriger les accès (invitations expirées, forks).

**Hibernate ORM.** *Documentation*.
https://hibernate.org/orm/documentation/

Utilisée pour comprendre le comportement de `ddl-auto` et le verrouillage des entités.

**Flyway.** *Documentation*.
https://documentation.red-gate.com/flyway

Utilisée pour gérer le schéma de la base de données à un seul endroit, via les migrations.

**PostgreSQL.** *Documentation*.
https://www.postgresql.org/docs/

Utilisée pour les contraintes d'unicité, les index sur expression (`lower()`) et `INSERT ... ON CONFLICT`, afin de supprimer les doublons sensibles à la casse et les conditions de concurrence.

**Glottolog.** *Base de données de langues*.
https://glottolog.org

Source des données de langues utilisées par les pipelines d'import de Vignette.

### Sécurité

**MDN Web Docs.** *Content Security Policy (CSP)*.
https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP

Utilisée pour limiter l'exécution de scripts lors de l'affichage des vignettes SVG.


### Projet et outils

**Document d'audit interne.** *Backend Vignette : résumé des problèmes trouvés*, semaine 1 du projet.

Point de départ de tout le travail : liste des problèmes de doublons, de code dupliqué et de sécurité à corriger.

**Département d'informatique et de recherche opérationnelle (UdeM).** *Template de site de projet IFT3150*.
https://github.com/udem-diro/template-projet

Base du présent site de suivi.

**Zensical.** *Documentation*.
https://zensical.org

Générateur de site statique utilisé pour construire ce site.

## Utilisation de l'intelligence artificielle

**Claude (Claude Code) — Anthropic**

Utilisé comme assistant de développement sur le backend, principalement pour :

- explorer et comprendre le code existant (repérer où une logique est définie et utilisée) ;
- explication pour corriger des bugs ;
- rédiger les commentaires du code ;
- repérer et supprimer le code inutilisé ou écrit en double;
- comprendre comment fonctionne flyway (causée par une difficulté à comprendre la documentation);
- aide dans la rédaction de certains tests.

Les modifications proposées ont été relues et testées avant d'être conservées dans le projet.