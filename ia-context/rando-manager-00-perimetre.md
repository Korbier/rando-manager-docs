# Rando Manager PWA — Périmètre du MVP

> Document de référence — fonctionnalités du périmètre initial  
> Créé le 11/06/2026 — Mis à jour le 11/06/2026

---

## Applications

Le MVP est composé de **deux applications Angular distinctes** consommant le même backend Quarkus.

| Application | Cible | Périmètre |
|---|---|---|
| `rando-manager-web` | PC, navigateur bureau | AVANT + APRÈS |
| `rando-manager-mobile` | Android, PWA installée | Mise en cache + PENDANT + Synchronisation |

---

## Cycle de vie d'une randonnée

```
PLANIFIEE → PRETE → DEMARREE → REALISE_NON_SYNCHRONISEE → REALISEE
```

| Statut | Signification | Déclenché par | Côté                                                 |
|---|---|---|------------------------------------------------------|
| `PLANIFIEE` | Randonnée créée sur l'appli web | Création dans `rando-manager-web` | Serveur                                              |
| `PRETE` | Données téléchargées sur le mobile, démarrable hors ligne | Mise en cache dans `rando-manager-mobile` | Serveur mais passage déclenchée par le client mobile  |
| `DEMARREE` | Suivi terrain en cours | Démarrage dans `rando-manager-mobile` | **Client mobile uniquement**                         |
| `REALISE_NON_SYNCHRONISEE` | Suivi terminé, données en attente de synchronisation | Arrêt dans `rando-manager-mobile` | **Client mobile uniquement**                         |
| `REALISEE` | Données synchronisées et persistées côté serveur | Synchronisation dans `rando-manager-mobile` | Serveur mais passage déclenchée par le client mobile |

**Règle :** une seule randonnée peut être à l'état `DEMARREE` à la fois.

> Les états `DEMARREE` et `REALISE_NON_SYNCHRONISEE` sont des états **purement locaux** (IndexedDB). Le backend ne les connaît pas. Du point de vue du serveur, une randonnée passe directement de `PRETE` à `REALISEE` au moment de la synchronisation.

## Cycle de vie d'un voyage

Le statut du voyage est **calculé automatiquement** à partir des statuts de ses randonnées — il n'est jamais saisi manuellement.

| Statut | Condition |
|---|---|
| `PLANIFIE` | Toutes les randonnées sont `PLANIFIEE` ou `PRETE` |
| `EN_COURS` | Au moins une randonnée est `DEMARREE`, `REALISE_NON_SYNCHRONISEE` ou `REALISEE` (mais pas toutes `REALISEE`) |
| `TERMINE` | Toutes les randonnées sont `REALISEE` |

---

## Authentification

- JWT avec refresh token, durée de session **30 jours**
- Reconnexion silencieuse sur les deux applications tant que le refresh token est valide
- Politique identique pour `rando-manager-web` et `rando-manager-mobile`

---

## rando-manager-web

### Phase AVANT

#### Randonnée

- **Création**
  - Saisie d'un titre
  - Upload d'un fichier GPX d'itinéraire prévisionnel
  - Saisie des informations d'hébergement à l'arrivée (nom, adresse, dates, numéro de confirmation, prix, notes)
  - Calculé et renseigné automatiquement par le backend au parsing du GPX :
    - Distance
    - Dénivelé positif
    - Dénivelé négatif
    - Horodatage de création
- **Modification**
- **Suppression**
- **Déplacement / affectation dans un répertoire**

#### Voyage

- **Création**
  - Saisie d'un titre
  - Ajout des randonnées associées (ordonnées par numéro d'étape)
  - Calculé et renseigné automatiquement :
    - Distance totale (somme des distances des randonnées)
    - Dénivelé positif total
    - Dénivelé négatif total
    - Horodatage de création
- **Modification**
- **Suppression**
- **Déplacement / affectation dans un répertoire**

> Les statistiques totales du voyage sont calculées à la volée par le backend en sommant les statistiques des randonnées associées.

#### Répertoire

- **Création**
- **Modification**
- **Suppression**

### Phase APRÈS

- **Consultation de la randonnée**
  - Affichage de l'itinéraire prévu
  - Affichage des photos enregistrées
- **Modification de la randonnée**
  - Saisie / modification d'un commentaire
  - Ajout / modification des légendes de photos
  - Upload d'un fichier GPX d'itinéraire réellement parcouru (optionnel)
    - Calculé et renseigné automatiquement par le backend au parsing du GPX :
      - Distance réelle
      - Dénivelé positif réel
      - Dénivelé négatif réel

---

## rando-manager-mobile

> Fonctionnement **offline first**. Entre les états `PRETE` et `REALISE_NON_SYNCHRONISEE`, l'application n'effectue aucun appel au backend. Toutes les actions terrain sont stockées localement dans IndexedDB.

### Mise en cache (avant de partir)

- Listing de toutes les randonnées à l'état `PLANIFIEE`
- Téléchargement en une seule action de toutes les randonnées listées :
  - Points GPX parsés (itinéraire disponible offline)
  - Informations d'hébergement
  - Métadonnées de la randonnée
- Passage de toutes les randonnées concernées de `PLANIFIEE` à `PRETE` (côté serveur et IndexedDB)
- **Prérequis : être en ligne.** Cette étape doit être réalisée avant de partir sur le terrain.

### Phase PENDANT

> À partir du démarrage du suivi, **aucun appel backend n'est effectué**. Toutes les transitions d'état et toutes les données produites sont gérées localement dans IndexedDB jusqu'à la synchronisation.

#### Démarrage du suivi

- Sélection d'une randonnée à l'état `PRETE`
- Passage de l'état `PRETE` à `DEMARREE` **(local uniquement)**
- Horodatage automatique du démarrage
- Peut être effectué **hors ligne**

#### Enregistrement d'une photo

- Prise de vue via l'appareil photo du téléphone (**1 clic** — déclenchement immédiat, aucune saisie obligatoire)
- Renseigné automatiquement :
  - Coordonnées GPS de la prise de vue
  - Horodatage de la prise de vue
  - Association à la randonnée `DEMARREE`
- Légende optionnelle — saisie non bloquante, modifiable ultérieurement dans `rando-manager-web`
- Si aucune randonnée n'est `DEMARREE` au moment de la prise de vue : proposition de démarrer une randonnée

#### Saisie d'une observation

- Saisie libre de texte
- Renseigné automatiquement :
  - Horodatage de la saisie
  - Association à la randonnée `DEMARREE`
- Si aucune randonnée n'est `DEMARREE` au moment de la saisie : proposition de démarrer une randonnée

#### Consultation de la randonnée en cours

- Affichage de l'itinéraire prévu (points GPX pré-chargés, disponible offline)
- Affichage des photos enregistrées jusqu'à présent
- Affichage des informations sur l'hébergement du soir

#### Arrêt du suivi

- Passage de l'état `DEMARREE` à `REALISE_NON_SYNCHRONISEE` **(local uniquement)**
- Horodatage automatique de l'arrêt
- Affichage du bouton de synchronisation

### Synchronisation

- Bouton affiché uniquement si au moins une randonnée est à l'état `REALISE_NON_SYNCHRONISEE`
- Déclenchement **manuel**
- **Nécessite d'être en ligne**
- À la synchronisation réussie :
  - Photos uploadées sur Cloudflare R2
  - Données persistées en base (observations, horodatages, métadonnées terrain)
  - Horodatage de synchronisation enregistré
  - Passage de l'état `REALISE_NON_SYNCHRONISEE` à `REALISEE` (côté serveur et IndexedDB)

---

## Règles métier

- Une seule randonnée peut être à l'état `DEMARREE` à la fois, toutes randonnées confondues.
- Une randonnée ne peut être démarrée que si elle est à l'état `PRETE` (données terrain disponibles localement).
- Entre les états `PRETE` et `REALISE_NON_SYNCHRONISEE`, l'appli mobile fonctionne intégralement hors ligne — aucun appel backend n'est requis.
- Il est possible d'enchaîner plusieurs randonnées passant par `DEMARREE` → `REALISE_NON_SYNCHRONISEE` avant de synchroniser — cas typique d'un voyage multi-étapes sans réseau.
- Il est possible de préparer plusieurs randonnées (phase AVANT) avant d'en démarrer une — cas typique de la préparation complète d'un voyage avant le départ.
- L'état `REALISEE` garantit que toutes les données sont persistées côté serveur. Rien ne reste en attente côté client.
- La synchronisation est portée **uniquement par `rando-manager-mobile`**.
