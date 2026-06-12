# Rando Manager PWA — CI/CD, Environnements et cycle de vie

> Document de référence — intégration continue et déploiement  
> Créé le 08/06/2026 — Mis à jour le 08/06/2026

---

## 1. Environnements

L'application est déployée sur deux environnements distincts, hébergés sur un unique Raspberry Pi.

### 1.1 Vue d'ensemble

| Environnement | Rôle | Infrastructure | Mise à jour |
|---|---|---|---|
| **Alpha** | Version stable à démontrer | Pi (port 8081) | À chaque merge sur `develop` |
| **Prod** | Version en production | Pi (port 8080) | À chaque tag `vX.Y.Z` |

### 1.2 Pi — Alpha + Prod

- Héberge deux instances de l'application en parallèle (ports et répertoires distincts)
- **Alpha** : accessible publiquement sur `alpha.tondomaine.fr`, permet de démontrer les fonctionnalités à venir
- **Prod** : version stable en cours d'utilisation sur `tondomaine.fr`
- Les mises à jour Prod entraînent une courte indisponibilité assumée (pas de blue/green)

---

## 2. Stratégie de branches

### 2.1 Principe général

```
feature/xxx  ──►  (PR)  ──►  develop  ──►  (tag vX.Y.Z)  ──►  Prod
                    ↑
            CI obligatoire
         (build + tests requis)
```

### 2.2 Règles par branche

| Branche | Protection | Alimentation |
|---|---|---|
| `develop` | Push direct interdit — PR obligatoire | Merge de branches `feature/xxx` |
| `feature/xxx` | Aucune | Créée depuis `develop`, mergée via PR |

### 2.3 Cycle d'une fonctionnalité

```
1. Création d'une branche feature
   git checkout -b feature/ma-fonctionnalite

2. Développement et commits sur la branche feature

3. Ouverture d'une Pull Request vers develop
   → Déclenchement automatique du workflow CI

4. CI obligatoire : build + tests unitaires + tests d'intégration
   → Le merge est bloqué tant que le CI n'est pas vert

5. Merge de la PR
   → Déploiement automatique sur Alpha
```

### 2.4 Cycle d'une release Prod

```
1. État souhaité atteint sur develop (validé via Alpha)

2. Création d'un tag depuis develop
   git tag v1.2.0
   git push origin v1.2.0

3. Déclenchement automatique du workflow deploy-prod
   → Build with version finale (1.2.0)
   → Déploiement sur Prod avec coupure courte
```

---

## 3. Workflows GitHub Actions

### 3.1 Vue d'ensemble

| Workflow | Fichier | Déclencheur | Cible |
|---|---|---|---|
| **CI** | `ci.yml` | PR vers `develop` | Aucun déploiement |
| **Deploy Alpha** | `deploy-alpha.yml` | Push sur `develop` | Pi (Alpha) |
| **Deploy Prod** | `deploy-prod.yml` | Tag `v*` | Pi (Prod) |

### 3.2 CI (`ci.yml`)

Déclenché sur chaque PR vers `develop`. Le merge est bloqué si ce workflow échoue.

Étapes :
- Build Angular + tests (Jest / Karma)
- Build Quarkus + tests unitaires (JUnit)
- Tests d'intégration Quarkus

### 3.3 Deploy Alpha (`deploy-alpha.yml`)

Déclenché à chaque merge sur `develop`.

Étapes :
- Build Angular avec injection de la version SNAPSHOT et du short commit SHA
- Build Quarkus avec version SNAPSHOT et short commit SHA
- Déploiement sur le Pi (environnement Alpha)

### 3.4 Deploy Prod (`deploy-prod.yml`)

Déclenché à la création d'un tag `vX.Y.Z` depuis `develop`.

Étapes :
- Extraction du numéro de version depuis le tag (ex: `v1.2.0` → `1.2.0`)
- Extraction du short commit SHA
- Build Angular avec injection de la version finale et du short commit SHA
- Build Quarkus avec version finale et short commit SHA
- Déploiement sur le Pi (environnement Prod) avec coupure courte

---

## 4. Versioning des artifacts

### 4.1 Principe

Les artifacts sont versionnés différemment selon l'environnement cible :

| Environnement | Format de version | Exemple |
|---|---|---|
| Alpha | `X.Y.Z-SNAPSHOT` | `1.2.0-SNAPSHOT` |
| Prod | `X.Y.Z` | `1.2.0` |

Le numéro de version SNAPSHOT est défini dans le fichier de build du projet (`pom.xml` côté Quarkus, `package.json` côté Angular) et utilisé tel quel par le workflow Alpha.

Le numéro de version Prod est extrait du tag Git au moment du build et injecté dynamiquement — il remplace la version SNAPSHOT.

### 4.2 Informations injectées au build

À chaque build (Alpha ou Prod), deux informations sont injectées :

| Information | Source | Exemple |
|---|---|---|
| `version` | Tag Git (Prod) ou `pom.xml` (Alpha) | `1.2.0` / `1.2.0-SNAPSHOT` |
| `commitSha` | Short SHA du commit HEAD | `a3f9k2c` |

### 4.3 Injection côté Quarkus

Les deux valeurs sont passées comme propriétés Maven au moment du build :

```bash
./mvnw package -DskipTests \
  -Dapp.version=1.2.0 \
  -Dapp.commit=a3f9k2c
```

Elles sont déclarées dans `application.properties` et exposées via un endpoint dédié :

```
GET /api/info
```

Réponse :
```json
{
  "version": "1.2.0",
  "commit": "a3f9k2c",
  "environment": "prod"
}
```

### 4.4 Injection côté Angular

Le workflow génère dynamiquement le fichier `src/environments/environment.prod.ts` avant le `ng build` :

```typescript
export const environment = {
  production: true,
  version: '1.2.0',
  commitSha: 'a3f9k2c'
};
```

Ces informations sont affichées dans l'interface (pied de page ou écran "À propos").

---

