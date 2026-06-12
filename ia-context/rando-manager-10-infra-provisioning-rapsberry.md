# Rando Manager — Infra Ansible

> Document de référence — provisioning du Raspberry Pi  
> Créé le 08/06/2026 — Mis à jour le 08/06/2026

---

## 1. Contexte et responsabilités

### 1.1 Périmètre

Ce projet Ansible a une responsabilité unique et bien délimitée : **préparer une machine vierge** pour qu'elle soit prête à recevoir les déploiements applicatifs.

Il ne touche jamais aux artifacts applicatifs (jars Quarkus, builds Angular) — c'est le rôle des GitHub Actions des repos applicatifs `rando-manager-back`, `rando-manager-web` et `rando-manager-mobile`.

### 1.2 Machine cible

| Machine | Rôle | Environnements hébergés |
|---|---|---|
| **Pi** | Serveur unique | Alpha + Prod |

Les deux environnements sont hébergés en parallèle sur le même Pi, différenciés par des ports, des répertoires et des domaines distincts. Cette différenciation est portée par les variables d'inventory.

### 1.3 Relation avec le repo applicatif

```
rando-manager-infra   (repo Ansible)
  └── Provisionne le Pi (OS, services, config système)
  └── GitHub Actions : déclenchement manuel uniquement

rando-manager         (repo app)
  └── CI : build + tests (PR → develop)
  └── Deploy Alpha : push sur develop → Pi (env alpha)
  └── Deploy Prod  : tag vX.Y.Z → Pi (env prod)
```

Le repo applicatif suppose que le Pi est **déjà provisionné** avant le premier déploiement. L'ordre de mise en place est donc :

```
1. Pousser le repo Ansible sur GitHub
2. Configurer les secrets dans les deux repos
3. Déclencher le workflow Ansible (provision du Pi)
4. Le Pi est prêt — le repo app peut déployer
```

---

## 2. Structure du projet

```
rando-manager-infra/
│
├── inventory/
│   ├── hosts.yml                      ← Pi déclaré
│   └── group_vars/
│       ├── all.yml                    ← variables communes
│       └── pi.yml                     ← environnements Alpha et Prod
│
├── roles/
│   ├── common/                        ← système de base, hostname, paquets, utilisateur système
│   ├── java/                          ← installation JDK 21 (Amazon Corretto)
│   ├── postgresql/                    ← install, création bases + user
│   ├── quarkus/                       ← répertoires et services systemd par environnement
│   ├── nginx/                         ← install, vhosts, rate limiting
│   └── certbot/                       ← Let's Encrypt, swap config HTTPS, hook renouvellement
│
├── playbooks/
│   └── provision.yml                  ← playbook principal (orchestre les rôles)
│
└── .github/
    └── workflows/
        └── provision.yml              ← GitHub Actions : déclenche Ansible
```

Chaque rôle suit la structure Ansible standard :

```
roles/<nom>/
├── tasks/
│   └── main.yml          ← tâches du rôle
├── handlers/
│   └── main.yml          ← handlers (ex: reload nginx)
├── templates/
│   └── ...               ← fichiers de config avec variables Jinja2
└── defaults/
    └── main.yml          ← valeurs par défaut des variables du rôle
```

---

## 3. Inventory

### 3.1 Déclaration de l'hôte

```yaml
# inventory/hosts.yml
all:
  children:
    pi:
      hosts:
        pi.tondomaine.fr:
```

### 3.2 Variables communes (`all.yml`)

```yaml
# inventory/group_vars/all.yml
app_user: rando-user
app_dir: /opt/rando-manager
```

> Les secrets (mots de passe, clés R2) ne sont **jamais** dans ce fichier. Ils sont injectés au moment de l'exécution via les secrets GitHub Actions.

### 3.3 Variables Pi (`pi.yml`)

Le Pi héberge deux environnements en parallèle sur des ports et domaines distincts.

```yaml
# inventory/group_vars/pi.yml
environments:
  - name: alpha
    domain: alpha.tondomaine.fr
    quarkus_port: 8081
    frontend_dir: /var/www/rando-alpha
    db_name: rando_alpha
    bucket_photos: rando-photos-alpha
    bucket_gpx: rando-gpx-alpha

  - name: prod
    domain: tondomaine.fr
    quarkus_port: 8080
    frontend_dir: /var/www/rando-prod
    db_name: rando_prod
    bucket_photos: rando-photos-prod
    bucket_gpx: rando-gpx-prod
```

Les rôles `postgresql`, `quarkus`, `nginx` et `certbot` itèrent sur cette liste via une boucle `loop`. Un seul template est rendu autant de fois qu'il y a d'environnements, ce qui évite toute duplication de logique.

---

## 4. Playbook principal

```yaml
# playbooks/provision.yml
- name: Provision Rando Manager Pi
  hosts: pi
  become: true
  roles:
    - common
    - java
    - postgresql
    - quarkus
    - nginx
    - certbot
```

L'ordre des rôles est significatif :

- `quarkus` est joué avant `nginx` pour que les répertoires applicatifs existent quand Nginx configure ses proxies.
- `certbot` est joué après `nginx` car il s'appuie sur la configuration HTTP-only déployée par Nginx pour répondre au challenge ACME.

Le playbook est **idempotent** : il peut être rejoué autant de fois que nécessaire sans effets de bord.

---

## 5. Rôles

### 5.1 Vue d'ensemble

| Rôle | Responsabilité |
|---|---|
| `common` | `apt upgrade`, paquets de base, hostname, création de l'utilisateur système `rando-user` |
| `java` | Installation JDK 21 (Amazon Corretto) |
| `postgresql` | Installation, création des bases `rando_alpha` et `rando_prod`, utilisateur `rando_user` |
| `quarkus` | Répertoires applicatifs et services systemd par environnement (créés, non démarrés) |
| `nginx` | Installation, vhosts par environnement (HTTP-only en passe 1), rate limiting |
| `certbot` | Émission des certificats, swap vers config HTTPS, hook de renouvellement |

### 5.2 Rôle `common`

- `apt update && apt upgrade`
- Installation des paquets utilitaires : `curl`, `wget`, `git`, `unzip`, `gnupg2`, `ca-certificates`, `lsb-release`, `apt-transport-https`
- Configuration du hostname via `inventory_hostname`
- Création de l'utilisateur système `rando-user` (sans shell, sans home)

### 5.3 Rôle `java`

- Ajout du dépôt Amazon Corretto
- Installation de `java-21-amazon-corretto-jdk`
- Vérification de la version installée (`java -version`)

### 5.4 Rôle `postgresql`

- Installation de `postgresql` et `postgresql-contrib`
- `systemctl enable` + `systemctl start`
- Création de l'utilisateur `rando_user` avec le mot de passe injecté (`pg_password`)
- Boucle sur `environments` : création de chaque base (`db_name`) et attribution des droits
- Vérification que `listen_addresses = 'localhost'` (isolation réseau)

### 5.5 Rôle `quarkus`

Boucle sur `environments`. Pour chaque entrée :

**Répertoires :**

- Création de `/opt/rando-manager/{name}/`
- Création de `/opt/rando-manager/{name}/config/`
- Création de `/opt/rando-manager/{name}/config/application.properties` via template `application.properties.j2`
- `chown` de l'arborescence vers `rando-user`
- Création de `{frontend_dir}/`
- `chown` de `{frontend_dir}` vers `www-data`

**Service systemd :**

- Déploiement de `rando-{name}.service` via template `rando.service.j2`
- `systemctl daemon-reload`
- Le service est créé mais **ni activé ni démarré** — le repo applicatif prend le relais au premier déploiement

Templates :

```
roles/quarkus/templates/
├── application.properties.j2   ← config Quarkus par environnement
└── rando.service.j2            ← /etc/systemd/system/rando-{name}.service
```

Le template `application.properties.j2` injecte les variables propres à chaque environnement : port, base de données, buckets R2, domaine JWT. Les clés R2 sont injectées via les secrets GitHub Actions (`r2_access_key`, `r2_secret_key`, `r2_endpoint`).

### 5.6 Rôle `nginx`

**Passe 1 — Bootstrap HTTP-only :**

- Installation de Nginx
- Ajout de la zone de rate limiting dans le bloc `http {}` de `nginx.conf`
- Désactivation du vhost par défaut
- Boucle sur `environments` : déploiement d'un vhost HTTP-only par environnement via template `vhost-http-only.conf.j2`
  - Expose `/.well-known/acme-challenge/` depuis `/var/www/certbot/{name}/`
  - Redirige tout le reste vers HTTPS (sauf le challenge ACME)
- `nginx -t` + `systemctl enable` + `systemctl start`

**Passe 2 — Swap HTTPS (déclenché par `certbot`) :**

- Le template `vhost-https.conf.j2` est déjà déployé par le rôle mais non activé
- Un handler `nginx : activate https config` est notifié par le rôle `certbot` après émission du certificat
- Ce handler remplace le vhost HTTP-only par le vhost HTTPS complet et recharge Nginx

**Contenu du vhost HTTPS (`vhost-https.conf.j2`) par environnement :**

- Redirection HTTP → HTTPS
- TLS avec le certificat Let's Encrypt du domaine `{domain}`
- Headers de sécurité : HSTS, X-Frame-Options, X-Content-Type-Options
- Serving des fichiers statiques Angular (`{frontend_dir}`, SPA routing)
- Proxy vers Quarkus sur `{quarkus_port}` (`/api/`)
- Rate limiting sur `/api/auth`

Templates :

```
roles/nginx/templates/
├── vhost-http-only.conf.j2   ← config bootstrap (passe 1)
└── vhost-https.conf.j2       ← config finale (passe 2, activée par certbot)
```

### 5.7 Rôle `certbot`

- Installation de `certbot` et `python3-certbot-nginx`
- Boucle sur `environments` :
  - Si le certificat n'existe pas encore → émission via `--webroot -w /var/www/certbot/{name}` pour le domaine `{domain}`
  - Si le certificat existe déjà → skip (idempotence)
  - Après émission réussie → notification du handler `nginx : activate https config` pour cet environnement
- Déploiement du hook `/etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh` via template
- Vérification que le timer systemd `certbot.timer` est actif

Templates :

```
roles/certbot/templates/
└── reload-nginx.sh.j2    ← hook systemctl reload nginx post-renouvellement
```

---

## 6. Stratégie de déploiement du certificat TLS (bootstrap)

La séquence complète lors d'un provisioning initial sur un Pi vierge :

```
1. Rôle nginx (passe 1)
   → Déploie les vhosts HTTP-only pour Alpha et Prod
   → Expose /.well-known/acme-challenge/ via webroot
   → Nginx démarre

2. Rôle certbot
   → Pour chaque environnement (alpha, prod) :
       Émet le certificat en mode webroot (Nginx reste actif, aucune coupure)
       Notifie le handler "nginx : activate https config"

3. Handler nginx
   → Swap le vhost HTTP-only par le vhost HTTPS complet
   → systemctl reload nginx

4. Résultat : Nginx sert Alpha et Prod en HTTPS
```

Lors des provisioning suivants (re-provisioning), les certificats existent déjà : le rôle `certbot` les détecte et skipe l'émission.

---

## 7. Workflow GitHub Actions

### 7.1 Déclenchement

Le provisioning est déclenché **manuellement** via `workflow_dispatch`.

```yaml
# .github/workflows/provision.yml
name: Provision

on:
  workflow_dispatch:

jobs:
  provision:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Ansible
        run: pip install ansible

      - name: Write SSH private key
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.PI_SSH_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519

      - name: Run playbook
        run: |
          ansible-playbook playbooks/provision.yml \
            -i inventory/hosts.yml \
            --extra-vars "pg_password=${{ secrets.PG_PASSWORD }} \
                          r2_access_key=${{ secrets.R2_ACCESS_KEY }} \
                          r2_secret_key=${{ secrets.R2_SECRET_KEY }} \
                          r2_endpoint=${{ secrets.R2_ENDPOINT }}"
```

### 7.2 Cas d'usage typiques

| Situation | Action |
|---|---|
| Installation initiale du Pi | Déclencher le workflow |
| Re-provisioning après réinstallation OS | Déclencher le workflow |
| Mise à jour de la configuration Nginx ou d'un service | Modifier le rôle concerné, pousser sur `main`, déclencher manuellement |

---

## 8. Secrets GitHub Actions

À configurer dans **Settings → Secrets and variables → Actions** du repo `rando-manager-infra` :

| Secret | Description |
|---|---|
| `PI_SSH_KEY` | Clé privée SSH (ed25519) |
| `PG_PASSWORD` | Mot de passe PostgreSQL pour l'utilisateur `rando_user` |
| `R2_ACCESS_KEY` | Clé d'accès Cloudflare R2 |
| `R2_SECRET_KEY` | Clé secrète Cloudflare R2 |
| `R2_ENDPOINT` | Endpoint R2 (`https://<account-id>.r2.cloudflarestorage.com`) |

> La même clé SSH `PI_SSH_KEY` doit également être déclarée dans le repo `rando-manager` pour que les workflows de déploiement applicatif puissent accéder au Pi.

---

## 9. Ce qui reste hors périmètre Ansible

| Opération | Raison |
|---|---|
| Configuration NAT sur la box (ports 80 et 443) | Hors reach d'Ansible |
| Génération de la clé SSH et ajout dans `authorized_keys` | Bootstrap initial, prérequis à toute connexion Ansible |
| Génération de la clé privée JWT (`jwt-private.pem`) | Opération sensible, à réaliser manuellement et déposer dans `/opt/rando-manager/{name}/config/` |
| Création du domaine, configuration DNS et sous-domaine `alpha` | Hors infrastructure Pi |
| Création des buckets Cloudflare R2 | Réalisé une seule fois via l'interface Cloudflare |

---

## 10. Séquence de mise en place complète

```
1. Configurer le NAT sur la box (ports 80 et 443 → IP fixe du Pi)

2. Configurer les entrées DNS
   tondomaine.fr       → IP fixe du Pi
   alpha.tondomaine.fr → IP fixe du Pi

3. Créer les buckets sur Cloudflare R2
   rando-photos-alpha, rando-photos-prod
   rando-gpx-alpha, rando-gpx-prod

4. Premier accès SSH au Pi (mot de passe)
   → Copier la clé publique SSH dans authorized_keys
   → Désactiver l'authentification par mot de passe

5. Pousser le repo rando-manager-infra sur GitHub

6. Configurer les secrets GitHub Actions du repo infra
   (PI_SSH_KEY, PG_PASSWORD, R2_ACCESS_KEY, R2_SECRET_KEY, R2_ENDPOINT)

7. Déclencher le workflow Ansible
   → Le Pi est provisionné
   → Répertoires, services systemd et certificats TLS en place

8. Déposer manuellement jwt-private.pem sur le Pi pour chaque environnement
   sudo cp jwt-private.pem /opt/rando-manager/alpha/config/
   sudo cp jwt-private.pem /opt/rando-manager/prod/config/
   sudo chown rando-user:rando-user /opt/rando-manager/alpha/config/jwt-private.pem
   sudo chown rando-user:rando-user /opt/rando-manager/prod/config/jwt-private.pem

9. Configurer les secrets GitHub Actions du repo rando-manager
   (PI_HOST, PI_USER, PI_SSH_KEY)

10. Premier déploiement applicatif depuis le repo rando-manager
```

---

## 11. Stratégie de déploiement du repo Ansible

Aucun déploiement automatique au push — le provisioning reste un acte intentionnel.
