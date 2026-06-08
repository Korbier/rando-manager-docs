# Rando Manager

Application PWA de gestion de randonnées, couvrant les trois phases d'une sortie —
préparation sur PC, suivi terrain sur Android, rédaction du carnet après.

Hébergée sur Raspberry Pi, accessible depuis n'importe quel appareil via HTTPS.

---

## Stack en un coup d'œil

| Composant | Technologie | Rôle |
|---|---|---|
| Frontend | Angular (PWA) | Interface utilisateur, offline first |
| Backend | Quarkus | API REST, logique métier |
| Base de données | PostgreSQL | Données structurées |
| Stockage fichiers | MinIO | Photos et fichiers GPX |
| Reverse proxy | Nginx | TLS, routage, rate limiting |
| Provisioning | Ansible | Mise en place des Raspberry Pi |
| CI/CD | GitHub Actions | Build, tests, déploiement |

---

## Organisation de la documentation

### [Conception](conception/index.md)
Modèle de données, stratégie de synchronisation offline, mode public, choix techniques.
À lire en premier pour comprendre les décisions structurantes du projet.

### [Infrastructure](infrastructure/index.md)
Architecture réseau, provisioning Ansible des Raspberry Pi, configuration des services,
gestion des certificats TLS.

### [CI/CD](cicd/index.md)
Environnements Dev / Alpha / Prod, stratégie de branches, workflows GitHub Actions,
versioning des artifacts.

### [Roadmap](roadmap/index.md)
Fonctionnalités hors périmètre MVP — visualisation 3D, export PDF, intégration météo,
multi-utilisateur.

---

## État du projet

| Périmètre | État |
|---|---|
| Conception générale | ✅ Définie |
| Infrastructure Raspberry Pi | ✅ Définie |
| CI/CD & environnements | ✅ Définie |
| Schéma PostgreSQL | 🔲 À concevoir |
| API Quarkus (endpoints) | 🔲 À concevoir |
| Squelette Angular PWA | 🔲 À concevoir |
| Développement | 🔲 Non démarré |