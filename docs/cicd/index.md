# CI/CD & Environnements

Cette section décrit les environnements de déploiement, la stratégie de branches
et les workflows GitHub Actions qui automatisent build, tests et déploiements.

---

## Les trois environnements

| Environnement | Rôle | Infrastructure | Mise à jour |
|---|---|---|---|
| **Dev** | Version en cours de développement | Pi 1 | À chaque merge sur `develop` |
| **Alpha** | Version stable à démontrer | Pi 2 | À chaque merge sur `develop` |
| **Prod** | Version en production | Pi 2 | À chaque tag `vX.Y.Z` |

---

## Stratégie de branches

````
feature/xxx  ──►  (PR)  ──►  develop  ──►  (tag vX.Y.Z)  ──►  Prod
                    ↑
            CI obligatoire
         (build + tests requis)
````

---

## Workflows GitHub Actions

| Workflow | Déclencheur | Cible |
|---|---|---|
| **CI** | PR vers `develop` | Aucun déploiement |
| **Deploy Alpha** | Push sur `develop` | Pi 1 (Dev) + Pi 2 (Alpha) |
| **Deploy Prod** | Tag `v*` | Pi 2 (Prod) |

---

## Contenu

### [Branches & Workflows](branches-workflows.md)
Cycle de vie d'une fonctionnalité, cycle d'une release Prod, détail des trois
workflows GitHub Actions — CI, Deploy Alpha, Deploy Prod.

### [Versioning & Artifacts](versioning-artifacts.md)
Format des versions SNAPSHOT et release, injection de la version et du commit SHA
au build côté Quarkus et Angular, endpoint `/api/info`.