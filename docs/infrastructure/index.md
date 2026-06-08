# Infrastructure

Cette section décrit l'architecture réseau et le provisioning des Raspberry Pi —
comment les services sont installés, configurés et exposés.

---

## Architecture réseau

```
Internet (HTTPS :443)
        │
        ▼
    Box / Routeur
    NAT 443 → Raspberry Pi :443
        │
        ▼
┌───────────────────────────────────────┐
│           Nginx (Raspberry Pi)        │
│                                       │
│  /*         → Angular (statique)      │
│  /api/*     → Quarkus :8080           │
│  /storage/* → MinIO :9000             │
│                                       │
│  TLS : Let's Encrypt                  │
└───────────────────────────────────────┘
        │
        ├── Quarkus     :8080  (localhost uniquement)
        ├── PostgreSQL  :5432  (localhost uniquement)
        └── MinIO       :9000  (localhost uniquement)
                        :9001  (console admin, localhost uniquement)
```
---

## Ports

| Service | Port | Exposition | Accès |
|---|---|---|---|
| Nginx (HTTPS) | 443 | Internet | Public |
| Nginx (HTTP) | 80 | Internet | Redirige vers 443 |
| Quarkus | 8080 | localhost | Nginx → Quarkus |
| PostgreSQL | 5432 | localhost | Quarkus uniquement |
| MinIO (API S3) | 9000 | localhost | Nginx + Quarkus |
| MinIO (console) | 9001 | localhost | Tunnel SSH uniquement |

---

## Contenu

### [Provisioning Ansible](provisioning-ansible.md)
Structure du projet Ansible, rôles, inventory, playbook principal, workflow
GitHub Actions de déclenchement manuel.

### [Services](services.md)
Installation et configuration de chaque service — PostgreSQL, MinIO, Nginx,
Quarkus. Commandes de diagnostic et ordre de démarrage.

### [TLS & Let's Encrypt](tls-lets-encrypt.md)
Fonctionnement du protocole ACME, configuration Certbot, renouvellement
automatique, hook de rechargement Nginx.