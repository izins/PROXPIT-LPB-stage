#PROXPIT 

> Supervision centralisée, prédiction des incidents et prévention de la saturation de stockage pour les infrastructures Proxmox VE, propulsée par l'intelligence artificielle.



---

## Sommaire

- [À propos](#-à-propos)
- [Fonctionnalités](#-fonctionnalités)
- [Architecture](#-architecture)
- [Stack technique](#-stack-technique)
- [Démarrage rapide](#-démarrage-rapide)
- [Structure du projet](#-structure-du-projet)
- [Variables d'environnement](#-variables-denvironnement)
- [Pipeline IA](#-pipeline-ia)
- [Sécurité](#-sécurité)
- [Tests](#-tests)
- [Documentation](#-documentation)
- [Contribuer](#-contribuer)
- [Licence](#-licence)

---

##  À propos

Cette plateforme **on-premise** vient se greffer à une infrastructure **Proxmox VE** existante — elle ne la remplace pas. Elle ajoute une couche de supervision centralisée, d'intelligence artificielle et de gouvernance au-dessus de Proxmox, en consommant son API REST officielle via [Proxmoxer](https://github.com/proxmoxer/proxmoxer).

**Objectif principal :** anticiper la saturation du stockage des machines virtuelles avant qu'elle ne provoque une indisponibilité de service, grâce à un pipeline de détection d'anomalies et de prédiction basé sur l'apprentissage automatique.

Architecture multi-tenant : `Entreprise → Utilisateurs → Serveurs Proxmox → Nodes → VM → Métriques`.

---

##  Fonctionnalités

- **Tableau de bord centralisé** multi-serveurs et multi-nodes
- **Prédiction de saturation** — score de risque et estimation du nombre de jours restants avant incident
- **Détection d'anomalies** au-delà des seuils statiques classiques (Isolation Forest)
-  **Recommandations actionnables** hiérarchisées (suppression de snapshots, déplacement de VM, extension de stockage)
-  **Alertes multi-niveaux** (Info / Warning / Critical / Emergency) avec escalade automatique
-  **Multi-tenant** avec isolation stricte des données par entreprise
-  **RBAC** granulaire (Super Admin, Admin Entreprise, Opérateur, Auditeur)
- **Audit complet** et immuable de toutes les actions sensibles
-  **Déploiement on-premise** simple via Docker Compose

---

##  Architecture

Monolithe modulaire respectant les principes de la **Clean Architecture** et du **Domain-Driven Design**.

```
Frontend (React/Next.js)
        │  HTTPS
        ▼
   Nginx (reverse proxy)
        │
        ▼
Backend FastAPI (monolithe modulaire)
  ├─ Auth        ├─ Monitoring
  ├─ IA          ├─ Alerting
  └─ Recommandations  └─ Audit
        │
   ┌────┴────┬─────────────┐
   ▼         ▼              ▼
PostgreSQL  Redis      Celery Workers
+ Timescale (cache/broker)   │
                              ▼
                     Serveurs Proxmox VE
                       (API via Proxmoxer)
```

Le document d'architecture complet (diagrammes UML, modèle de données, sécurité, réseau, pipeline IA) est disponible dans [`/docs/architecture.md`](./docs/architecture.md).

---

##  Stack technique

| Domaine | Technologies |
|---|---|
| **Frontend** | React, Next.js, TypeScript, TailwindCSS, Recharts |
| **Backend** | FastAPI, Python, SQLAlchemy, Alembic, Pydantic |
| **Asynchrone** | Celery, Redis (broker + result backend) |
| **Base de données** | PostgreSQL 16, TimescaleDB (hypertables), Redis (cache) |
| **Intégration Proxmox** | Proxmoxer |
| **Intelligence Artificielle** | Isolation Forest (anomalies), XGBoost (prédiction de saturation) |
| **Déploiement** | Docker, Docker Compose, Nginx |
| **Monitoring** | Prometheus, Grafana, Loki, AlertManager |

---

##  Démarrage rapide

### Prérequis

- Docker ≥ 24.x et Docker Compose ≥ 2.x
- Un ou plusieurs serveurs Proxmox VE accessibles en réseau
- Un API Token Proxmox dédié à la supervision (lecture seule)

### Installation

```bash
# 1. Cloner le dépôt
git clone https://github.com/<votre-org>/proxmox-ai-supervision.git
cd proxmox-ai-supervision

# 2. Copier et configurer les variables d'environnement
cp .env.example .env
nano .env

# 3. Lancer la stack complète
docker compose up -d --build

# 4. Appliquer les migrations de base de données
docker compose exec backend alembic upgrade head

# 5. Créer le premier compte super-administrateur
docker compose exec backend python -m app.cli create-superadmin
```

L'interface est ensuite accessible sur `https://localhost` (ou le domaine configuré derrière Nginx).

### Créer un API Token Proxmox (lecture seule)

Sur l'interface Proxmox : `Datacenter → Permissions → API Tokens`, associer le token à un utilisateur dédié (ex. `monitoring@pve`) et à un rôle personnalisé limité à la lecture des métriques de nodes, VM, stockage et tâches — voir [`/docs/architecture.md#authentification`](./docs/architecture.md) pour le détail des permissions recommandées.

---

## Structure du projet

```
.
├── backend/
│   ├── app/
│   │   ├── api/            # Routers FastAPI (controllers)
│   │   ├── services/       # Cas d'usage applicatifs
│   │   ├── domain/         # Entités métier, règles, ports (aucune dépendance externe)
│   │   ├── repositories/   # Implémentations SQLAlchemy des ports
│   │   ├── ai/             # Pipelines de feature engineering et d'inférence
│   │   └── infra/          # DB, Redis, client Proxmoxer
│   ├── alembic/             # Migrations de base de données
│   └── tests/
├── frontend/
│   ├── app/                 # App Router Next.js
│   ├── features/            # Composants/hooks organisés par domaine métier
│   └── shared/               # Composants, contexts et services partagés
├── docs/
│   └── architecture.md      # Document d'architecture complet
├── docker-compose.yml
└── .env.example
```

---

## Variables d'environnement

| Variable | Description |
|---|---|
| `DATABASE_URL` | URL de connexion PostgreSQL/TimescaleDB |
| `REDIS_URL` | URL de connexion Redis (cache + broker Celery) |
| `JWT_SECRET_KEY` | Clé de signature des Access/Refresh Tokens |
| `JWT_ACCESS_EXPIRE_MINUTES` | Durée de vie de l'access token (défaut : 15) |
| `JWT_REFRESH_EXPIRE_DAYS` | Durée de vie du refresh token (défaut : 7) |
| `SECRET_ENCRYPTION_KEY` | Clé AES-256-GCM pour le chiffrement des API Tokens Proxmox au repos |
| `PROXMOX_COLLECT_INTERVAL_SECONDS` | Fréquence de collecte des métriques (défaut : 30) |

Voir `.env.example` pour la liste complète.

---

## Pipeline IA

```
Collecte → Prétraitement → Feature Engineering → Entraînement → Validation
    → Déploiement → Inférence → Alertes → Recommandations
```

- **Détection d'anomalies :** Isolation Forest (non supervisé)
- **Prédiction de saturation :** XGBoost (régression sur séries dérivées)
- **Sorties :** Risk Score, Anomaly Score, Days Remaining, Storage Forecast, Recommendations

Détails complets (comparaison des modèles, feature engineering, moteur de règles) dans [`/docs/architecture.md`](./docs/architecture.md).

---

## Sécurité

- Authentification plateforme : JWT (access + refresh) avec RBAC
- Authentification Proxmox : API Tokens à privilèges minimaux, validation déléguée à Proxmox
- TLS 1.3 de bout en bout, secrets chiffrés au repos (AES-256-GCM)
- Isolation multi-tenant renforcée par PostgreSQL Row-Level Security
- Conformité OWASP Top 10 — voir [`/docs/architecture.md#sécurité`](./docs/architecture.md)

Pour signaler une vulnérabilité, merci de ne **pas** ouvrir d'issue publique : contactez `security@<votre-domaine>.com`.

---

## Tests

```bash
# Backend
docker compose exec backend pytest

# Frontend
docker compose exec frontend npm run test

# Tests de charge
locust -f tests/load/locustfile.py
```

---

## Documentation

- [Document d'architecture complet](./docs/architecture.md) — diagrammes UML, modèle de données, sécurité, réseau, pipeline IA
- [Contrats API (OpenAPI)](https://localhost/docs) — généré automatiquement par FastAPI

---

## Contribuer

Les contributions sont les bienvenues. Merci de :

1. Créer une branche depuis `main` (`feature/ma-fonctionnalité`)
2. Respecter les conventions de code (`ruff`, `mypy` pour le backend ; `eslint`, `prettier` pour le frontend)
3. Ajouter des tests pour toute nouvelle fonctionnalité
4. Ouvrir une Pull Request avec une description claire du changement

---

##Licence

Projet propriétaire — usage interne et partenaires. Tous droits réservés.
