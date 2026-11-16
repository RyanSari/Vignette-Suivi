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
- Faire un état des lieux du backend

### Travail réalisé

!!! abstract "Avancement"
    - [x] Exécution du code et vérification du bon fonctionnement sur mon poste
    - [x] Rédaction du document « Backend Vignette : résumé des problèmes trouvés »
        - **1.1 Doublons de données** : vérifications username/email et titres en double (sensibles à la casse, dupliquées contrôleur/service), invitations DECLINED/EXPIRED, likes/favoris, `idx` des médias, `maxUses`, notifications, etc.
        - **1.2 Code et logique dupliqués** : deux gestionnaires d'exceptions, quatre mécanismes de droits, méthodes copiées-collées dans plusieurs services, copie de fork incomplète, schéma de BD géré à 3 endroits, deux pipelines Glottolog, etc.
        - **1.3 Sécurité et contrôle d'accès** : SVG/XML acceptés (XSS stockée), invitations expirées encore valides, contournement de l'approbation des forks, identifiants admin en dur

---

## Semaine 2 (6 – 12 novembre)

### Objectifs de la période
- Corriger les problèmes identifiés dans le document de la semaine 1

### Travail réalisé

!!! abstract "Avancement"
    - [x] Correction de tous les problèmes de la section **1.1 (doublons de données)**
    - [x] Correction de tous les problèmes de la section **1.2 (code et logique dupliqués)**, à l'exception de la gestion du schéma de la BD
    - [ ] Gestion du schéma de la BD (Flyway / `ddl-auto=update` / `SchemaMaintenance`) : reportée à la semaine 3
    - [ ] Section **1.3 (sécurité et contrôle d'accès)** : reportée à la semaine 3

---

## Semaine 3 (13 – 19 novembre)

### Objectifs de la période
- Terminer les points restants du document d'analyse (BD et sécurité)
- Commencer le modèle de données, la gestion des priorités et les utilitaires
- Évaluer l'usage de Supabase

### Travail réalisé

!!! abstract "Avancement"
    - [x] Refactoring
    - [x] Sécurité (section 1.3 du document)
    - [x] Base de données (gestion du schéma à un seul endroit)
    - [ ] Modèle de données ⏳ en cours
    - [x] Priority
    - [x] Utils
    - [x] Étude de l'usage de Supabase

### Prochaines étapes
- Terminer le modèle de données

---

## Semaine 4 (20 – 26 novembre)

### Objectifs de la période
- Terminer le modèle de données
- Gérer les vignettes SVG côté backend
- Mettre en place le site de suivi du projet

### Travail réalisé

!!! abstract "Avancement"
    - [ ] Modèle de données ⏳ en cours (reporté de la semaine 3)
    - [ ] Nouveau contrôleur vignette pour le SVG, avec stockage en base de données ⏳ en cours
    - [ ] Nouveau contrôleur pour enregistrer la vignette ⏳ en cours
    - [ ] Site de suivi du projet (à partir du [template IFT3150](https://github.com/udem-diro/template-projet)) ⏳ en cours
