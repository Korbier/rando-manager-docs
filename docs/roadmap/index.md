# Roadmap

Cette section regroupe les fonctionnalités identifiées mais volontairement exclues
du périmètre MVP. Ce ne sont pas des idées abandonnées — ce sont des directions
connues, documentées au moment où elles ont été pensées.

---

## Périmètre MVP

L'application MVP couvre les trois phases d'une randonnée pour un usage
mono-utilisateur, avec synchronisation offline sur le terrain.

Ce qui est **dans** le MVP :

- Gestion des répertoires, voyages et randonnées
- Import et affichage des traces GPX
- Suivi terrain offline — photos, journal horodaté
- Synchronisation automatique au retour réseau
- Mode public en lecture seule pour le partage d'un voyage

---

## Fonctionnalités différées

| Fonctionnalité | Résumé |
|---|---|
| [Visualisation 3D](visualisation-3d.md) | Renderer Vulkan/Rust, endpoint `/render`, données MNT |
| [Fonctionnalités futures](fonctionnalites-futures.md) | Export PDF, météo automatique, multi-utilisateur |

---

## Critères de sortie du MVP

Le MVP sera considéré terminé quand les éléments suivants seront opérationnels :

| Critère | État |
|---|---|
| Schéma PostgreSQL défini et migré | 🔲 |
| API Quarkus — endpoints CRUD complets | 🔲 |
| Squelette Angular PWA — offline opérationnel | 🔲 |
| Déploiement sur Pi 1 et Pi 2 fonctionnel | 🔲 |
| Premier voyage complet enregistré en production | 🔲 |