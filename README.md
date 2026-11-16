# Vignette – Suivi du projet (IFT3150)

Site de suivi du projet informatique **Vignette** (cours IFT3150, Université de Montréal), construit avec [Zensical](https://zensical.org).

Il sert à :

* présenter le projet et ses objectifs ;
* documenter, semaine par semaine, le travail réalisé ;
* synthétiser les études, réalisations et résultats ;
* conserver les références et l'utilisation de l'IA.

## Prérequis

* Python **3.11** ou plus récent
* `pip`
* Git

## Installation

```bash
git clone https://github.com/RyanSari/Vignette-Suivi.git
cd Vignette-Suivi
pip install -r requirements.txt
```

## Utilisation

Lancer le site en local (rechargement automatique) :

```bash
zensical serve          # http://localhost:8000
zensical serve --open   # ouvre aussi le navigateur
```

Générer la version statique dans `site/` :

```bash
zensical build --clean
```

## Structure

```text
.
├── docs/
│   ├── index.md        # Présentation du projet
│   ├── suivi.md        # Suivi hebdomadaire
│   ├── synthese.md     # Synthèse : études, réalisation, évaluation, bilan
│   ├── references.md   # Références et utilisation de l'IA
│   └── css/
├── zensical.toml       # Configuration du site et de la navigation
└── requirements.txt    # Dépendances Python
```

## Pages

| Page | Contenu |
|------|---------|
| `index.md` | Contexte, problématique, objectifs, équipe |
| `suivi.md` | Une entrée par semaine : travail réalisé, décisions, difficultés, prochaines étapes |
| `synthese.md` | Vue d'ensemble en quatre phases : études préliminaires, réalisation, évaluation, bilan |
| `references.md` | Sources utilisées et déclaration d'usage d'outils d'IA |

## Mettre à jour le suivi

Ajouter une section `## Semaine N (dates)` dans [docs/suivi.md](docs/suivi.md), puis :

```bash
git add .
git commit -m "Suivi semaine N"
git push
```

## Configuration

Le nom du site, la navigation, l'URL publiée et le dépôt associé se modifient dans [zensical.toml](zensical.toml).

> Le dépôt ne contient pas encore de workflow GitHub Actions : le déploiement sur GitHub Pages reste à configurer si nécessaire.
