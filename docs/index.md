---
title: Vue d'ensemble du projet
---

<style>
    @media screen and (min-width: 76em) {
        .md-sidebar--primary {
            display: none !important;
        }
    }
</style>

# Vue d'ensemble du projet

!!! info "Informations générales"
    **Session**: Automne 2026  
    **Auteur(s)**: Ryan Sari 20266065 
    **Thème(s)**: Développement backend, qualité du code, sécurité, base de données  
    **Superviseur(s)**: Louis-Edouard Lafontant (DIRO, Université de Montréal)  

## Description du projet

### Contexte

Vignette est une application web dont le backend (Java) gère des scénarios composés d'images, de miniatures et d'audios, avec des comptes utilisateurs, des collaborations par invitation, des forks soumis à approbation, des likes et favoris, des notifications et un suivi des langues (données Glottolog).

Le backend a évolué par couches successives, ce qui a laissé de la logique dupliquée, des règles de validation incohérentes et plusieurs failles de sécurité.

### Problématique

Un audit du backend (réalisé en semaine 1) a identifié trois familles de problèmes :

- **Doublons de données** : vérifications dupliquées et sensibles à la casse (usernames, titres), invitations impossibles à renvoyer, conditions de concurrence sur les likes, l'ordre des médias, les liens d'invitation et les notifications.
- **Code et logique dupliqués** : plusieurs gestionnaires d'exceptions, quatre mécanismes de contrôle des droits, méthodes copiées dans plusieurs services, copie de fork incomplète, schéma de BD géré à trois endroits, deux pipelines Glottolog.
- **Sécurité** : fichiers SVG/XML acceptés (XSS stockée), invitations expirées encore valides, contournement de l'approbation des forks.

Ces défauts rendent le système difficile à maintenir et exposent des données et des utilisateurs.

### Proposition et objectifs

Assainir le backend de Vignette, puis préparer son évolution :

1. Corriger les problèmes de doublons de données et de concurrence.
2. Refactoriser le code dupliqué (droits, exceptions, utilitaires, copie de fork).
3. Corriger les failles de sécurité et de contrôle d'accès.
4. Unifier la gestion du schéma de la base de données.
5. Définir un modèle de données et un système de priorités.
6. Évaluer l'usage de Supabase.
7. Enregistrer les vignettes du Vignette Maker en SVG (avec leur dessin) et réintroduire le format SVG de façon sécurisée.

### Méthodologie

Démarche en trois temps : **audit** (documenter les problèmes), **correction** par ordre de priorité (doublons, refactoring, puis sécurité et base de données), puis **évolution** (modèle de données, priorités). Le travail est suivi semaine par semaine dans la page [Suivi](suivi.md).

### Validation et Évaluation

<!-- À compléter : tests ajoutés, vérification de chaque point du document d'audit, scénarios de non-régression. -->

## Échéancier

!!! info
    Le suivi complet est disponible dans la page [Suivi de projet](suivi.md).

| Semaine | Période | Activités | Livrable | Statut |
|---------|---------|-----------|----------|--------|
| 1 | 30 oct – 5 nov | Prise en main du projet et audit du backend | Document « Backend Vignette : résumé des problèmes trouvés » | ✅ Terminé |
| 2 | 6 – 12 nov | Correction des doublons de données et du code dupliqué | Code corrigé (sections 1.1 et 1.2) | ✅ Terminé |
| 3 | 13 – 19 nov | Schéma de BD unifié, sécurité, refactoring, priority, utils, étude Supabase | Code corrigé (section 1.3 et BD) | ✅ Terminé |
| 4 | 20 – 26 nov | Organisation du code, baseline SQL réécrite, retour du SVG sécurisé, contrôleur vignette SVG, téléchargement SVG/ZIP, suppression par l'admin, documentation, tests, site de suivi | Document « Backend Vignette : travail réalisé », site de suivi | 🔄 En cours |
| 3 – 4 | 13 nov – | Modèle de données (DTO) | Modèle de données | 🔄 En cours |
| — | <!-- dates --> | Présentation + Rapport | Présentation + Rapport | ⏳ À venir |
