# Rando Manager — Bootstrap Raspberry Pi

> Préparer le Pi pour recevoir un déploiement Ansible déclenché depuis GitHub Actions  
> Créé le 09/06/2026 — Mis à jour le 09/06/2026

---

## Contexte

Ce document couvre uniquement le **bootstrap initial** du Raspberry Pi : les opérations manuelles à réaliser une seule fois, depuis ton poste Windows, pour que le Pi soit joignable par Ansible via GitHub Actions.

La solution retenue pour connecter GitHub Actions au Pi est **Tailscale** : un VPN maillé basé sur WireGuard qui relie le Pi, ton poste Windows et les runners GitHub Actions dans un réseau privé commun, sans exposer aucun port SSH sur Internet.

Une fois ces étapes terminées, tout le reste (installation des services, configuration Nginx, certificats TLS…) est pris en charge par le playbook Ansible déclenché depuis GitHub Actions — sans aucune intervention manuelle supplémentaire.

```
Ce document          →  Pi prêt à recevoir Ansible
rando-manager-infra  →  Pi entièrement provisionné
```

---

## Prérequis matériels et réseau

Avant de commencer :

- Raspberry Pi 4 avec **Raspberry Pi OS Lite 64-bit** flashé sur la carte SD
- Pi connecté au réseau local (câble Ethernet recommandé)
- Accès à l'interface de ta box pour configurer le NAT et identifier l'IP du Pi
- Un poste Windows avec un terminal SSH disponible (Windows Terminal ou PowerShell)
- Un compte Tailscale créé sur https://tailscale.com (gratuit pour un usage personnel)

> Windows 10/11 intègre un client SSH natif. Aucun logiciel supplémentaire n'est nécessaire pour SSH.

---

## Étape 1 — Premier démarrage du Pi

Au premier démarrage avec Raspberry Pi OS Lite, le Pi démarre sans interface graphique.

> **Note Raspberry Pi Imager** : si tu utilises Raspberry Pi Imager pour flasher la carte SD, tu peux préconfigurer le nom d'utilisateur, le mot de passe et activer SSH directement depuis l'outil (icône engrenage). C'est la méthode recommandée — elle évite d'avoir à brancher clavier/écran.

Connecte-toi en SSH depuis ton terminal Windows :

```powershell
ssh pi@<IP-DU-PI>
```

> Pour trouver l'IP du Pi : consulte l'interface de ta box (liste des équipements connectés), ou utilise `ping raspberrypi.local` depuis ton terminal Windows.

**Changer immédiatement le mot de passe par défaut :**

```bash
passwd
```

---

## Étape 2 — Attribuer une IP fixe au Pi sur la box

Le Pi doit toujours avoir la même IP locale pour que les règles NAT restent stables.

Sur l'interface de ta box (généralement accessible sur `192.168.1.1`) :

1. Trouver le Pi dans la liste des équipements DHCP
2. Lui attribuer un **bail DHCP statique** (réservation d'IP) basée sur son adresse MAC
3. Noter l'IP attribuée

> Ne pas configurer l'IP fixe directement sur le Pi via `/etc/dhcpcd.conf` : la réservation DHCP côté box est plus simple à maintenir.

---

## Étape 3 — Configurer le NAT sur la box

Deux règles de redirection de port à créer dans l'interface de ta box (nécessaires pour Let's Encrypt et pour le trafic HTTPS applicatif) :

| Port externe | Port interne | Destination   |
|--------------|--------------|---------------|
| 80           | 80           | IP fixe du Pi |
| 443          | 443          | IP fixe du Pi |

> Le port 22 (SSH) **ne doit pas être exposé sur Internet**. C'est Tailscale qui assure la connectivité SSH depuis GitHub Actions — sans aucune règle NAT supplémentaire.

---

## Étape 4 — Installer Tailscale sur le Pi

Depuis la session SSH ouverte sur le Pi :

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Démarrer Tailscale et authentifier le Pi :

```bash
sudo tailscale up
```

Une URL s'affiche dans le terminal. Ouvre-la dans ton navigateur et authentifie-toi avec ton compte Tailscale. Le Pi apparaît alors dans ta console Tailscale (https://login.tailscale.com/admin/machines) avec une adresse IP Tailscale de la forme `100.x.x.x`.

**Noter cette adresse IP Tailscale du Pi** — elle sera utilisée comme `PI_HOST` dans les secrets GitHub Actions.

Activer le démarrage automatique de Tailscale :

```bash
sudo systemctl enable tailscaled
```

---

## Étape 5 — Installer Tailscale sur ton poste Windows

Télécharger et installer le client Windows depuis https://tailscale.com/download.

Une fois installé, se connecter avec le même compte Tailscale. Ton poste Windows et le Pi apparaissent tous les deux dans la console Tailscale et peuvent se voir directement.

**Vérifier la connectivité depuis Windows :**

```powershell
ping 100.x.x.x   # IP Tailscale du Pi
```

Tu peux désormais te connecter au Pi via SSH en utilisant son adresse Tailscale, sans passer par l'IP locale ni ouvrir de port :

```powershell
ssh pi@100.x.x.x
```

---

## Étape 6 — Créer les credentials OAuth Tailscale pour GitHub Actions

GitHub Actions a besoin de rejoindre ton réseau Tailscale de façon authentifiée. La méthode recommandée est un **client OAuth** associé à un tag dédié.

### 6.1 Créer le tag `tag:ci` dans les ACL Tailscale

Dans la console Tailscale → **Access controls** (https://login.tailscale.com/admin/acls/file), ajouter dans la section `tagOwners` :

```json
"tagOwners": {
    "tag:ci": []
}
```

Ce tag identifie les nœuds éphémères créés par GitHub Actions. Il permet de contrôler précisément leurs droits sur le réseau.

### 6.2 Créer le client OAuth

Dans la console Tailscale → **Settings → OAuth clients** (https://login.tailscale.com/admin/settings/oauth) :

1. Cliquer **Generate OAuth client**
2. Cocher **Auth Keys** en lecture et écriture
3. Sélectionner le tag `tag:ci`
4. Générer — noter le **Client ID** et le **Client Secret**

Ces deux valeurs seront ajoutées comme secrets GitHub Actions (`TS_OAUTH_CLIENT_ID` et `TS_OAUTH_SECRET`).

---

## Étape 7 — Générer la paire de clés SSH dédiée à GitHub Actions

Cette clé est **exclusivement réservée à GitHub Actions** — ne pas réutiliser une clé personnelle existante.

Depuis ton terminal Windows :

```powershell
ssh-keygen -t ed25519 -C "github-actions-rando" -f "$env:USERPROFILE\.ssh\id_ed25519_github_actions"
```

Deux fichiers sont créés :

| Fichier                         | Contenu    | Destination            |
|---------------------------------|------------|------------------------|
| `id_ed25519_github_actions`     | Clé privée | Secret GitHub Actions  |
| `id_ed25519_github_actions.pub` | Clé publique | Pi (`authorized_keys`) |

---

## Étape 8 — Déposer la clé publique sur le Pi

Depuis ton terminal Windows, copier la clé publique sur le Pi via son adresse Tailscale :

```powershell
type "$env:USERPROFILE\.ssh\id_ed25519_github_actions.pub" | ssh pi@100.x.x.x "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Vérifier que la connexion par clé fonctionne :

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_ed25519_github_actions" pi@100.x.x.x
```

Si la connexion s'établit sans demander de mot de passe, la clé est correctement en place.

**Désactiver l'authentification par mot de passe SSH** (durcissement) :

```bash
sudo nano /etc/ssh/sshd_config
```

Modifier ou ajouter :

```
PasswordAuthentication no
```

Redémarrer SSH :

```bash
sudo systemctl restart ssh
```

> À partir de ce moment, seule la clé SSH permet de se connecter au Pi, et uniquement via Tailscale. Conserve précieusement la clé privée.

---

## Étape 9 — Adapter le workflow Ansible pour Tailscale

Le workflow `provision.yml` dans `rando-manager-infra` doit connecter le runner à Tailscale **avant** de lancer Ansible :

```yaml
name: Provision

on:
  workflow_dispatch:

jobs:
  provision:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Connect to Tailscale
        uses: tailscale/github-action@v3
        with:
          oauth-client-id: ${{ secrets.TS_OAUTH_CLIENT_ID }}
          oauth-secret:    ${{ secrets.TS_OAUTH_SECRET }}
          tags: tag:ci

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

Le runner rejoint le réseau Tailscale, contacte le Pi via son adresse `100.x.x.x`, exécute le playbook, puis le nœud éphémère est automatiquement supprimé à la fin du job.

---

## Étape 10 — Configurer les secrets GitHub Actions

Dans le repo `rando-manager-infra` → **Settings → Secrets and variables → Actions → New repository secret** :

| Secret              | Valeur                                                        |
|---------------------|---------------------------------------------------------------|
| `PI_HOST`           | Adresse Tailscale du Pi (`100.x.x.x`)                        |
| `PI_USER`           | `pi`                                                          |
| `PI_SSH_KEY`        | Contenu complet de `id_ed25519_github_actions` (clé privée)  |
| `PG_PASSWORD`       | Mot de passe PostgreSQL choisi pour `rando_user`              |
| `R2_ACCESS_KEY`     | Clé d'accès Cloudflare R2                                     |
| `R2_SECRET_KEY`     | Clé secrète Cloudflare R2                                     |
| `R2_ENDPOINT`       | `https://<account-id>.r2.cloudflarestorage.com`               |
| `TS_OAUTH_CLIENT_ID`| Client ID OAuth Tailscale (étape 6)                           |
| `TS_OAUTH_SECRET`   | Client Secret OAuth Tailscale (étape 6)                       |

Pour copier la clé privée depuis Windows :

```powershell
Get-Content "$env:USERPROFILE\.ssh\id_ed25519_github_actions" | Set-Clipboard
```

Coller directement dans le champ de valeur du secret GitHub.

---

## Étape 11 — Vérification finale avant de déclencher Ansible

Checklist avant de lancer le workflow `provision.yml` :

- [ ] Pi accessible en SSH depuis ton poste Windows via l'adresse Tailscale
- [ ] Authentification par mot de passe SSH désactivée sur le Pi
- [ ] IP fixe attribuée au Pi sur la box
- [ ] Ports 80 et 443 redirigés vers le Pi (NAT box)
- [ ] DNS configuré : `tondomaine.fr` et `alpha.tondomaine.fr` pointent vers l'IP publique
- [ ] Accès Internet depuis le Pi opérationnel (`curl https://example.com`)
- [ ] Pi visible dans la console Tailscale avec statut **Connected**
- [ ] Tag `tag:ci` créé dans les ACL Tailscale
- [ ] Client OAuth Tailscale créé avec les bons droits
- [ ] Tous les secrets GitHub Actions renseignés

---

## Récapitulatif de la séquence

```
1. Premier démarrage Pi — connexion SSH par mot de passe
        │
        ▼
2. Attribution IP fixe sur la box
        │
        ▼
3. Règles NAT ports 80 et 443 (pas de port 22)
        │
        ▼
4. Installation Tailscale sur le Pi + authentification
        │
        ▼
5. Installation Tailscale sur le poste Windows
        │
        ▼
6. Création tag:ci et client OAuth Tailscale
        │
        ▼
7. Génération paire de clés SSH dédiée GitHub Actions (poste Windows)
        │
        ▼
8. Dépôt clé publique sur le Pi (via Tailscale) + désactivation auth mot de passe
        │
        ▼
9. Adaptation du workflow Ansible (ajout étape Tailscale)
        │
        ▼
10. Secrets GitHub Actions renseignés
        │
        ▼
11. Pi prêt — déclencher le workflow Ansible depuis GitHub
```
