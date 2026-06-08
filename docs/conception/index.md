# Conception

Cette section décrit les décisions structurantes du projet — ce que l'application
fait, comment les données sont organisées, et pourquoi les choix techniques ont
été faits ainsi.

À lire en priorité avant toute autre section.

---

## Contenu

### [Modèle de données](modele-donnees.md)
Les trois entités principales — Répertoire, Voyage, Randonnée — et leurs relations.
Structure complète des champs, identifiants UUID côté client, organisation en phases
Avant / Pendant / Après.

### [Stockage & Synchronisation](stockage-synchronisation.md)
Stratégie offline first : IndexedDB via Dexie.js côté client, PostgreSQL et MinIO
côté serveur. Pattern Outbox pour la synchronisation différée des actions terrain.
Gestion des photos du blob temporaire jusqu'à l'URL stable MinIO.

### [Mode public](mode-public.md)
Partage d'un voyage via une URL publique sans authentification. Token opaque,
champs visibles configurables, endpoint dédié en lecture seule.

### [Stack technique](stack-technique.md)
Choix des technologies frontend et backend — Angular 21, Quarkus, Signals,
Zoneless, Hibernate Panache — et justification de ces choix.

---

## Prochaines pages à concevoir

| Page | État |
|---|---|
| Schéma PostgreSQL | 🔲 À concevoir |
| API Quarkus — endpoints | 🔲 À concevoir |
| Squelette Angular PWA | 🔲 À concevoir |