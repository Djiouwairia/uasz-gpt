# UASZ-GPT — Documentation DevOps & Infrastructure

> **Chef de projet** : Ibrahima Gueye  
> **Superviseurs** : Ibrahima Ndao, Babacar Ndao  
> **Chef DevOps** : Djiouwairia  
> **Équipe** : 12 développeurs — 3 Squads  
> **Version** : 1.0 | **Date** : Avril 2026

---

## Table des matières

1. [Vue d'ensemble de l'infrastructure](#1-vue-densemble)
2. [Prérequis et outils](#2-prérequis)
3. [Structure du projet](#3-structure-du-projet)
4. [Stratégie de branches Git](#4-stratégie-de-branches)
5. [Docker et services](#5-docker-et-services)
6. [Pipeline CI/CD](#6-pipeline-cicd)
7. [Infrastructure AWS](#7-infrastructure-aws)
8. [Monitoring](#8-monitoring)
9. [Procédures opérationnelles](#9-procédures-opérationnelles)
10. [Contacts et urgences](#10-contacts-et-urgences)

---

## 1. Vue d'ensemble

UASZ-GPT est déployé sur **AWS eu-north-1 (Stockholm)** avec une architecture microservices conteneurisée. Chaque commit sur `main` déclenche automatiquement un déploiement en production via GitHub Actions.

```
Développeurs → GitHub → CI/CD → AWS ECR → EC2 Production
```

### Services en production

| Service | Image | Port | Rôle |
|---------|-------|------|------|
| Frontend | React + Nginx | 80 | Interface utilisateur |
| Backend | Spring Boot | 8080 | API REST |
| IA Engine | Python + LangChain | 8000 | Moteur RAG |
| MySQL | mysql:8.0 | 3306 | Base de données |
| Redis | redis:7.2-alpine | 6379 | Cache + sessions |
| ChromaDB | chromadb/chroma | 8000 | Vecteurs IA |
| Prometheus | prom/prometheus | 9090 | Métriques |
| Grafana | grafana/grafana | 3001 | Dashboards |

---

## 2. Prérequis

### Outils à installer (une seule fois)

```powershell
# Windows — vérifier les installations
docker --version      # Docker Desktop 28+
aws --version         # AWS CLI 2+
git --version         # Git 2.46+
node --version        # Node.js 20+
```

### Accès nécessaires

- Compte GitHub avec accès au repo `Djiouwairia/uasz-gpt`
- Clé SSH EC2 : `uaszgpt-key.pem` (demander au DevOps)
- Accès AWS Console : compte `Ria-Tech`

---

## 3. Structure du projet

```
uasz-gpt/
├── backend/                    ← Spring Boot (Squad 1)
│   ├── Dockerfile
│   ├── .dockerignore
│   └── src/
├── frontend/                   ← React (Squad 2)
│   ├── Dockerfile
│   ├── nginx.conf
│   └── .dockerignore
├── ia-engine/                  ← Python LangChain (Squad 1)
│   ├── Dockerfile
│   ├── requirements.txt
│   └── .dockerignore
├── docker/
│   ├── docker-compose.yml      ← Tous les services
│   ├── docker-compose.staging.yml
│   └── .env                    ← Ne pas committer !
├── monitoring/
│   ├── prometheus/
│   │   └── prometheus.yml
│   └── grafana/
│       └── provisioning/
├── .github/
│   └── workflows/
│       └── ci-cd.yml           ← Pipeline CI/CD
├── .env.example                ← Modèle variables d'env
└── .gitignore
```

---

## 4. Stratégie de branches

### Branches permanentes

| Branche | Environnement | Déclencheur |
|---------|--------------|-------------|
| `main` | Production AWS | Merge approuvé par DevOps |
| `staging` | Pré-production AWS | Merge hebdomadaire depuis develop |
| `develop` | Tests CI | Merge des features |

### Branches temporaires (Squads)

```bash
feat/squad1-rag-engine       # Nouvelle fonctionnalité
feat/squad2-quiz-interface   # Nouvelle fonctionnalité
fix/squad1-pdf-upload-bug    # Correction de bug
hotfix/login-crash-prod      # Urgence production
chore/squad3-docker-update   # Tâche technique DevOps
```

### Workflow quotidien d'un développeur

```bash
# 1. Créer sa branche depuis develop
git checkout develop
git pull origin develop
git checkout -b feat/squad1-ma-fonctionnalite

# 2. Travailler et committer
git add .
git commit -m "feat: description de la fonctionnalité"

# 3. Pousser et créer une Pull Request
git push origin feat/squad1-ma-fonctionnalite
# → Ouvrir PR sur GitHub vers develop
# → Demander review à un coéquipier
```

### Règles de protection (configurées par DevOps)

- `main` : 2 reviews obligatoires + CI verte + merge par chef projet uniquement
- `staging` : 1 review obligatoire + CI verte
- Push direct interdit sur `main` et `staging`

---

## 5. Docker et services

### Démarrer l'environnement local

```bash
# Depuis le dossier docker/
cd docker/

# Copier le fichier d'exemple
cp ../.env.example .env
# Éditer .env avec les bonnes valeurs

# Démarrer tous les services
docker compose up -d

# Vérifier
docker compose ps

# Voir les logs d'un service
docker compose logs backend -f
docker compose logs ia-engine -f
```

### Variables d'environnement requises

Créer `docker/.env` en copiant `.env.example` :

```env
DB_USER=uaszgpt_user
DB_PASSWORD=VOTRE_MOT_DE_PASSE
DB_ROOT_PASSWORD=VOTRE_ROOT_PASSWORD
GRAFANA_PASSWORD=VOTRE_PASSWORD_GRAFANA
EC2_HOST=IP_DE_EC2
ECR_REGISTRY=282325721086.dkr.ecr.eu-north-1.amazonaws.com
```

> **ATTENTION** : Ne jamais committer le fichier `.env` sur GitHub. Il est dans `.gitignore`.

### Commandes Docker utiles

```bash
# Arrêter tous les services
docker compose down

# Rebuild un service spécifique
docker compose build backend
docker compose up -d backend

# Accéder au shell d'un conteneur
docker exec -it uaszgpt-backend bash
docker exec -it uaszgpt-mysql mysql -u root -p

# Voir l'utilisation des ressources
docker stats

# Nettoyer les images inutilisées
docker system prune -f
```

---

## 6. Pipeline CI/CD

### Déclenchement automatique

```
Push sur develop  → Tests automatiques uniquement
Push sur staging  → Tests + Build Docker + Deploy staging
Push sur main     → Tests + Build Docker + Deploy production (avec approbation)
```

### Étapes du pipeline

```
1. Tests automatiques
   ├── JUnit (Spring Boot)
   ├── Jest (React)
   └── Pytest (Python IA)

2. Build images Docker
   ├── uaszgpt/backend:SHA_COMMIT
   ├── uaszgpt/frontend:SHA_COMMIT
   └── uaszgpt/ia-engine:SHA_COMMIT
   └── Push vers AWS ECR

3. Déploiement
   ├── SSH vers EC2
   ├── docker compose pull
   ├── docker compose up -d
   └── Health check
```

### Secrets GitHub requis

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | Clé accès AWS IAM |
| `AWS_SECRET_ACCESS_KEY` | Clé secrète AWS IAM |
| `EC2_HOST` | IP publique EC2 production |
| `EC2_HOST_STAGING` | IP EC2 staging |
| `EC2_SSH_KEY` | Clé privée SSH (.pem) |
| `ECR_REGISTRY` | URL registre ECR |
| `DB_USER` | Utilisateur MySQL |
| `DB_PASSWORD` | Mot de passe MySQL |
| `DB_ROOT_PASSWORD` | Root MySQL |
| `GRAFANA_PASSWORD` | Mot de passe Grafana |

> Seul le DevOps (Djiouwairia) a accès à ces secrets.

---

## 7. Infrastructure AWS

### Région : eu-north-1 (Stockholm)

### Ressources créées

| Ressource | ID/URL | Rôle |
|-----------|--------|------|
| EC2 Instance | i-0dc1da4dcb2eb2bed | Serveur principal |
| EC2 Type | t3.medium (2CPU, 4GB) | Calcul |
| Disque | 18 Go | Stockage |
| Security Group | sg-007a7c95e062a39e3 | Pare-feu |
| ECR Backend | uaszgpt/backend | Images Docker |
| ECR Frontend | uaszgpt/frontend | Images Docker |
| ECR IA | uaszgpt/ia-engine | Images Docker |

### Se connecter à EC2

```powershell
# Depuis Windows PowerShell
ssh -i "uaszgpt-key.pem" ubuntu@ec2-51-20-47-187.eu-north-1.compute.amazonaws.com

# Tunnel SSH pour Grafana
ssh -i "uaszgpt-key.pem" -L 3001:localhost:3001 ubuntu@ec2-51-20-47-187.eu-north-1.compute.amazonaws.com -N
# Puis ouvrir http://localhost:3001
```

### Ports ouverts (Security Group)

| Port | Protocole | Source | Service |
|------|-----------|--------|---------|
| 22 | TCP | 0.0.0.0/0 | SSH |
| 80 | TCP | 0.0.0.0/0 | HTTP/Nginx |
| 443 | TCP | 0.0.0.0/0 | HTTPS (futur) |
| 3001 | TCP | IP DevOps | Grafana |
| 9090 | TCP | IP DevOps | Prometheus |
| 9100 | TCP | IP DevOps | Node Exporter |

---

## 8. Monitoring

### Accéder à Grafana

```powershell
# 1. Ouvrir le tunnel SSH (laisser ce terminal ouvert)
ssh -i "uaszgpt-key.pem" -L 3001:localhost:3001 ubuntu@EC2_HOST -N

# 2. Ouvrir dans le navigateur
http://localhost:3001
# Login : admin / mot de passe dans .env
```

### Dashboards disponibles

- **Node Exporter Full (ID: 1860)** — CPU, RAM, disque du serveur EC2
- **Spring Boot** — métriques de l'API backend
- **MySQL** — performance de la base de données

### Alertes à configurer (TODO)

- CPU > 80% pendant 5 minutes → alerte email
- RAM > 90% → alerte email
- Conteneur arrêté → alerte immédiate
- Disque > 85% → alerte email

---

## 9. Procédures opérationnelles

### Déploiement manuel d'urgence

```bash
# Se connecter à EC2
ssh -i "uaszgpt-key.pem" ubuntu@EC2_HOST

# Aller dans le dossier app
cd /app/uaszgpt

# Déployer manuellement
docker compose pull
docker compose up -d --remove-orphans
docker compose ps
```

### Rollback rapide (< 2 minutes)

```bash
# Sur EC2 — revenir à la version précédente
docker compose down
docker pull 282325721086.dkr.ecr.eu-north-1.amazonaws.com/uaszgpt/backend:VERSION_PRECEDENTE
docker compose up -d
```

### Redémarrage des services

```bash
# Redémarrer un service spécifique
docker compose restart backend
docker compose restart mysql

# Redémarrer tous les services
docker compose down && docker compose up -d

# Si EC2 a redémarré (systemd s'en charge automatiquement)
sudo systemctl status uaszgpt.service
sudo systemctl start uaszgpt.service
```

### Vérification de santé

```bash
# Sur EC2
docker compose ps                    # État des conteneurs
docker stats --no-stream             # CPU/RAM par conteneur
df -h                                # Espace disque
curl -I http://localhost             # Nginx répond ?
curl http://localhost:8080/actuator/health  # Backend Spring Boot
curl http://localhost:8000/health    # IA Engine Python
```

### Hotfix en production

```bash
# 1. Créer branche hotfix depuis main
git checkout main
git pull origin main
git checkout -b hotfix/description-du-bug

# 2. Corriger et committer
git add .
git commit -m "hotfix: correction bug critique"

# 3. Tester sur staging d'abord
git push origin hotfix/description-du-bug
# Créer PR vers staging → merger → vérifier

# 4. Si OK, merger vers main ET develop
# PR vers main (approbation DevOps requise)
# PR vers develop (pour ne pas perdre le fix)
```

---

## 10. Contacts et urgences

| Rôle | Nom | Responsabilité |
|------|-----|----------------|
| Chef de projet | Ibrahima Gueye | Décisions, merge main |
| Superviseur | Ibrahima Ndao | Validation technique |
| Superviseur | Babacar Ndao | Validation pédagogique |
| DevOps | Djiouwairia | Infrastructure, CI/CD, AWS |
| Squad 1 Lead | — | Backend Spring Boot + IA |
| Squad 2 Lead | — | Frontend React |

### En cas de panne production

1. Vérifier Grafana → identifier le service en panne
2. Tenter redémarrage : `docker compose restart SERVICE`
3. Si échec → rollback : voir section 9
4. Notifier le chef de projet
5. Ouvrir un issue GitHub avec label `incident`

### Commandes de diagnostic rapide

```bash
# Logs des dernières 100 lignes d'un service
docker logs uaszgpt-backend --tail 100

# Logs en temps réel
docker logs uaszgpt-backend -f

# Statut tous les services
docker compose ps

# Espace disque
df -h /

# Mémoire
free -h
```

---

*Document maintenu par l'équipe DevOps UASZ-GPT — Mise à jour : Avril 2026*