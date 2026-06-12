# Rando Manager — Suivi des tâches

> Créé le 11/06/2026 — Mis à jour le 12/06/2026

---

## 0. Création des repos GitHub

- [X] 0.1 `rando-manager-docs` (privé) — documentation
- [X] 0.2 `rando-manager-infra` (privé) — provisioning Ansible
- [ ] 0.3 `rando-manager-back` (privé) — backend Quarkus
- [ ] 0.4 `rando-manager-web` (privé) — frontend Angular bureau
- [ ] 0.5 `rando-manager-mobile` (privé) — frontend Angular PWA mobile

---

## 1. Mise en place du Raspberry Pi

- [X] 1.1 Création du repo `rando-manager-infra` (privé)
- [X] 1.2 Bootstrap du Pi (étapes manuelles)
- [X] 1.3 Conception du provisioning Ansible
    - [X] 1.3.1 Définition des rôles
    - [X] 1.3.2 Conception des templates et variables
- [ ] 1.4 Réalisation du playbook Ansible
    - [X] 1.4.1 Rôle `common`
    - [X] 1.4.2 Rôle `java`
    - [X] 1.4.3 Rôle `postgresql`
    - [ ] 1.4.4 Rôle `quarkus`
    - [ ] 1.4.5 Rôle `nginx`
    - [ ] 1.4.6 Rôle `certbot`
- [ ] 1.5 Déploiement Alpha + Prod sur le Pi
    - [ ] 1.5.1 Déclenchement du workflow Ansible
    - [ ] 1.5.2 Vérification de l'environnement

---

## 2. Création du repo back : `rando-manager-back`

- [ ] 2.1 Structure du projet Quarkus
- [ ] 2.2 Mise en place Flyway
- [ ] 2.3 Environnement de dev local
    - [ ] 2.3.1 Configuration PostgreSQL local
    - [ ] 2.3.2 Bouchon R2 (profil dev Quarkus)
- [ ] 2.4 Prototype : endpoint `GET /api/info` (version + commit + environnement)
- [ ] 2.5 Publication du contrat OpenAPI sur GitHub Packages
- [ ] 2.6 Mise en place de la CI/CD

---

## 3. Création du repo web : `rando-manager-web`

- [ ] 3.1 Structure du projet Angular
- [ ] 3.2 Prototype : affichage de la version (consommée depuis le package généré)
- [ ] 3.3 Mise en place de la CI/CD

---

## 4. Création du repo mobile : `rando-manager-mobile`

- [ ] 4.1 Structure du projet Angular (PWA)
- [ ] 4.2 Prototype : affichage de la version (consommée depuis le package généré)
- [ ] 4.3 Mise en place de la CI/CD

---

## 5. Développement de l'application

- [ ] 5.1 Conception du modèle de données
- [ ] 5.2 Organisation du code
    - [ ] 5.2.1 Organisation du code back
    - [ ] 5.2.2 Organisation du code front (web + mobile)
- [ ] 5.3 Développement des fonctionnalités (à détailler)