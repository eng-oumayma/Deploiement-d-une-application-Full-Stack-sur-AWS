# ☁️ Déploiement Full-Stack sur AWS avec Terraform

<div align="center">

![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-%235835CC.svg?style=for-the-badge&logo=terraform&logoColor=white)
![Angular](https://img.shields.io/badge/Angular-%23DD0031.svg?style=for-the-badge&logo=angular&logoColor=white)
![Node.js](https://img.shields.io/badge/Node.js-%23339933.svg?style=for-the-badge&logo=nodedotjs&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-%234479A1.svg?style=for-the-badge&logo=mysql&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-%23009639.svg?style=for-the-badge&logo=nginx&logoColor=white)

**Infrastructure as Code · Auto Scaling · High Availability**

</div>

---

## 📋 Table des matières

- [Vue d'ensemble](#-vue-densemble)
- [Architecture](#-architecture)
- [Stack technique](#-stack-technique)
- [Prérequis](#-prérequis)
- [Démarrage rapide](#-démarrage-rapide)
- [Structure du projet](#-structure-du-projet)
- [Configuration](#-configuration)
- [Déploiement](#-déploiement)
- [Dépannage](#-dépannage)
- [Workflow par session lab](#-workflow-par-session-lab)
- [Commandes utiles](#-commandes-utiles)

---

## 🎯 Vue d'ensemble

Ce projet déploie une application full-stack complète sur AWS en **Infrastructure as Code** avec Terraform. En une seule commande, Terraform provisionne l'intégralité de l'infrastructure : réseau, sécurité, calcul, base de données et répartition de charge.

### Pourquoi Terraform ?

| Sans Terraform | Avec Terraform |
|---|---|
| ~80 clics dans la console AWS | `terraform apply` |
| Une erreur = tout recommencer | Corriger le code → re-apply |
| Infra documentée en captures d'écran | Infra documentée en code versionnable |
| Suppression ressource par ressource | `terraform destroy` |
| Non reproductible | Reproductible à l'infini |

---

## 🏗️ Architecture

```
                            ┌─────────────────┐
                            │    Internet      │
                            └────────┬────────┘
                                     │ HTTP :80
                                     ▼
┌────────────────────────────────────────────────────────────────┐
│                    VPC  10.0.0.0/16                            │
│                                                                │
│  ┌──────────────────── SOUS-RÉSEAUX PUBLICS ─────────────────┐ │
│  │                                                           │ │
│  │   AZ-A  (10.0.1.0/24)           AZ-B  (10.0.2.0/24)      │ │
│  │  ┌──────────────────┐          ┌──────────────────┐       │ │
│  │  │  ALB  (nœud A)   │          │  ALB  (nœud B)   │       │ │
│  │  │  NAT Gateway     │          │                  │       │ │
│  │  │  EC2 Frontend    │          │                  │       │ │
│  │  └──────────────────┘          └──────────────────┘       │ │
│  └───────────────────────────────────────────────────────────┘ │
│                         │  ALB distribue                       │
│  ┌──────────────────── SOUS-RÉSEAUX PRIVÉS ──────────────────┐ │
│  │                                                           │ │
│  │   AZ-A  (10.0.3.0/24)           AZ-B  (10.0.4.0/24)      │ │
│  │  ┌──────────────────┐          ┌──────────────────┐       │ │
│  │  │  EC2 Backend     │          │  EC2 Backend     │ ← ASG │ │
│  │  │  (Node.js API)   │          │  (Node.js API)   │       │ │
│  │  └────────┬─────────┘          └────────┬─────────┘       │ │
│  │           └──────────┬──────────────────┘                 │ │
│  │                      ▼                                    │ │
│  │              ┌───────────────┐                            │ │
│  │              │  Amazon RDS   │                            │ │
│  │              │    MySQL      │                            │ │
│  │              └───────────────┘                            │ │
│  └───────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
```

### Composants déployés

| Composant | Rôle | Réseau |
|-----------|------|--------|
| **VPC + 4 Subnets** | Réseau isolé sur 2 zones de disponibilité | — |
| **Internet Gateway** | Point d'entrée depuis internet | Public |
| **NAT Gateway** | Accès internet sortant pour les instances privées | Public |
| **Application Load Balancer** | Répartition de charge + health checks | Public |
| **EC2 Frontend** | Sert l'app Angular via Nginx | Public |
| **ASG + Launch Template** | 2 à 4 instances backend auto-scaling | Privé |
| **Amazon RDS MySQL** | Base de données relationnelle | Privé |
| **4 Security Groups** | Pare-feux en couches (ALB, frontend, backend, RDS) | — |
| **Scaling Policy** | Scale out automatique si CPU > 70% | — |

---

## 🛠️ Stack technique

- **Frontend** : Angular 17+ · Nginx
- **Backend** : Node.js · Express.js
- **Base de données** : Amazon RDS MySQL 8.0
- **Infrastructure** : Terraform · AWS (VPC, EC2, ALB, ASG, RDS, NAT GW)
- **Déploiement** : PM2 · cloud-init (user_data)

---

## ✅ Prérequis

- Compte **AWS Academy** avec accès au Learner Lab (Vocareum)
- [Terraform](https://developer.hashicorp.com/terraform/downloads) ≥ 1.0
- [AWS CLI](https://aws.amazon.com/cli/) configuré
- Git + une clé SSH
- Dépôt GitHub contenant :
  - `backend/` — API Node.js avec une route `GET /health` retournant `200`
  - `client/` — Application Angular

---

## ⚡ Démarrage rapide

```bash
# 1. Cloner le repo
git clone https://github.com/votre-user/votre-repo.git
cd votre-repo/terraform

# 2. Créer la clé SSH
ssh-keygen -t rsa -b 4096 -f ~/.ssh/projet-cloud-key -N ""
chmod 400 ~/.ssh/projet-cloud-key

# 3. Configurer les variables
cp terraform.tfvars.example terraform.tfvars
# Éditez terraform.tfvars avec vos valeurs

# 4. Exporter les credentials AWS (depuis Vocareum)
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."
export AWS_DEFAULT_REGION="us-east-1"

# 5. Déployer
terraform init
terraform apply

# 6. Accéder à l'application (~5-8 min après apply)
terraform output frontend_url    # → URL du frontend
terraform output alb_dns_name    # → URL de l'API
```

---

## 📁 Structure du projet

```
votre-projet/
│
├── terraform/
│   ├── main.tf                  # Provider AWS + AMI Ubuntu
│   ├── variables.tf             # Déclaration de toutes les variables
│   ├── outputs.tf               # URLs, IPs, DNS affichés après apply
│   ├── terraform.tfvars         # Vos valeurs (⚠️ ne pas commiter !)
│   │
│   ├── vpc.tf                   # VPC, subnets, IGW, NAT GW, routes
│   ├── security_groups.tf       # 4 Security Groups
│   ├── rds.tf                   # RDS MySQL + DB Subnet Group
│   ├── backend.tf               # ALB + Target Group + ASG + Scaling Policy
│   ├── frontend.tf              # EC2 frontend
│   │
│   ├── user_data_backend.sh     # Script de démarrage des instances backend
│   ├── user_data_frontend.sh    # Script de démarrage du frontend Angular
│   └── .gitignore
│
├── backend/                     # API Node.js/Express
│   └── index.js                 # Point d'entrée (doit exposer GET /health)
│
├── client/                      # Application Angular
│   └── src/environments/
│       ├── environment.ts       # Config développement (localhost)
│       └── environment.prod.ts  # Config production (URL ALB injectée automatiquement)
│
└── README.md
```

---

## ⚙️ Configuration

Créez un fichier `terraform/terraform.tfvars` :

```hcl
# Votre IP publique (curl ifconfig.me)
my_ip = "x.x.x.x/32"

# URL de votre dépôt GitHub
github_repo = "https://github.com/votre-user/votre-repo.git"

# Credentials RDS
db_password = "VotreMotDePasseSecurise123!"

# Optionnel — valeurs par défaut déjà définies
project_name  = "projet-cloud"
instance_type = "t2.micro"
app_port      = 3000
db_name       = "appdb"
db_username   = "admin"
```

> ⚠️ Ne commitez jamais `terraform.tfvars` — il est dans `.gitignore`.

### Variables disponibles

| Variable | Description | Défaut |
|----------|-------------|--------|
| `aws_region` | Région AWS | `us-east-1` |
| `project_name` | Préfixe pour toutes les ressources | `projet-cloud` |
| `instance_type` | Type d'instance EC2 | `t2.micro` |
| `key_pair_name` | Nom de la paire de clés SSH | `projet-cloud-key` |
| `my_ip` | Votre IP pour le SSH | **obligatoire** |
| `github_repo` | URL de votre dépôt GitHub | **obligatoire** |
| `app_port` | Port du backend | `3000` |
| `db_engine` | Moteur RDS | `mysql` |
| `db_name` | Nom de la base | `appdb` |
| `db_username` | Utilisateur RDS | `admin` |
| `db_password` | Mot de passe RDS | **obligatoire** |

---

## 🚀 Déploiement

### Déploiement complet

```bash
terraform init      # Télécharger les providers
terraform validate  # Vérifier la syntaxe
terraform plan      # Prévisualiser les changements
terraform apply     # Déployer (~8-10 min)
```

### Outputs après déploiement

```
alb_dns_name       = "http://projet-cloud-alb-xxxxxxx.us-east-1.elb.amazonaws.com"
frontend_url       = "http://xx.xx.xx.xx"
frontend_public_ip = "xx.xx.xx.xx"
rds_endpoint       = "projet-cloud-db.xxxxxxx.us-east-1.rds.amazonaws.com"
ssh_frontend       = "ssh -i ~/.ssh/projet-cloud-key ubuntu@xx.xx.xx.xx"
asg_name           = "projet-cloud-asg"
vpc_id             = "vpc-xxxxxxxxxxxxxxxxx"
```

### Vérifier que tout fonctionne

```bash
# Health check du backend via ALB
curl http://$(terraform output -raw alb_dns_name)/health
# → {"status":"ok"}

# API utilisateurs
curl http://$(terraform output -raw alb_dns_name)/api/users
# → [{"id":1,"name":"John Doe",...}]

# Frontend
open http://$(terraform output -raw frontend_public_ip)
```

### Mettre à jour les instances backend

Si vous modifiez `user_data_backend.sh` sans détruire l'infrastructure :

```bash
terraform apply   # Met à jour le Launch Template

aws autoscaling start-instance-refresh \
  --auto-scaling-group-name projet-cloud-asg \
  --preferences '{"MinHealthyPercentage": 0}' \
  --region us-east-1
```

### Détruire l'infrastructure

```bash
terraform destroy   # Supprime TOUTES les ressources AWS
```

---

## 🔧 Dépannage

### Frontend affiche "Welcome to nginx!"
Le build Angular n'a pas abouti. Connectez-vous et vérifiez :
```bash
ssh -i ~/.ssh/projet-cloud-key ubuntu@<frontend-ip>
sudo tail -50 /var/log/user-data.log
```

### Backend 502 Bad Gateway
Les instances backend sont unhealthy. Vérifiez :
1. AWS Console → EC2 → Target Groups → Targets → statut des instances
2. Logs de démarrage : `sudo cat /var/log/cloud-init-output.log`
3. Que la route `GET /health` existe dans votre `index.js` et retourne `200`

### Erreur CORS dans le navigateur
L'URL de l'ALB dans `environment.prod.ts` est incorrecte ou l'ALB a changé (nouveau lab).
Le script `user_data_frontend.sh` injecte automatiquement le bon DNS au moment du build.

### Erreur "Duplicate variable declaration"
Des variables sont déclarées à la fois dans `main.tf` et `variables.tf`. Supprimez-les de `main.tf`.

### ERR_NAME_NOT_RESOLVED
L'ancien DNS ALB est utilisé (lab expiré). Supprimez le state et redéployez :
```bash
rm -f terraform.tfstate terraform.tfstate.backup
terraform apply
```

| Erreur | Cause | Solution |
|--------|-------|----------|
| `502 Bad Gateway` | Backend ne tourne pas | Vérifier `/var/log/user-data.log` sur une instance backend |
| `403 Forbidden` | `index.html` pas dans `/var/www/html/` | Vérifier le chemin `dist/client/browser/` |
| `ERR_NAME_NOT_RESOLVED` | Ancien DNS ALB | Supprimer le state, `terraform apply` |
| `npm: command not found` | Node.js non installé | Vérifier le script `user_data_backend.sh` |
| `Unhandled non-multipart userdata` | `#!/bin/bash` manquant ou précédé d'un `#` | Corriger la première ligne du script |

---

## 🔄 Workflow par session lab

> Les labs AWS Academy expirent après quelques heures. Voici la procédure complète à chaque session.

```bash
# ── DÉBUT DE SESSION ──────────────────────────────────────────

# 1. Démarrer le lab sur Vocareum et copier les credentials
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."
export AWS_DEFAULT_REGION="us-east-1"

# 2. Nettoyer l'ancien state (lab précédent = ressources supprimées)
rm -f terraform.tfstate terraform.tfstate.backup

# 3. Déployer toute l'infrastructure
terraform init
terraform apply

# 4. Attendre ~8 minutes puis tester
curl http://$(terraform output -raw alb_dns_name)/health

# ── FIN DE SESSION ────────────────────────────────────────────

# Avant que le lab expire → détruire proprement
terraform destroy
```

---

## 📚 Commandes utiles

### Terraform

| Commande | Description |
|----------|-------------|
| `terraform init` | Initialiser et télécharger les providers |
| `terraform validate` | Vérifier la syntaxe HCL |
| `terraform plan` | Prévisualiser les changements |
| `terraform apply` | Déployer l'infrastructure |
| `terraform apply -target=aws_instance.frontend` | Déployer une seule ressource |
| `terraform destroy` | Supprimer toutes les ressources |
| `terraform output` | Afficher les outputs |
| `terraform state list` | Lister les ressources gérées |
| `terraform state rm <resource>` | Supprimer une ressource du state |

### AWS CLI

```bash
# Vérifier les credentials
aws sts get-caller-identity

# Lister les instances de l'ASG
aws ec2 describe-instances \
  --filters "Name=tag:aws:autoscaling:groupName,Values=projet-cloud-asg" \
  --query "Reservations[*].Instances[*].[InstanceId,PrivateIpAddress,State.Name]" \
  --output table \
  --region us-east-1

# Forcer le remplacement des instances backend
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name projet-cloud-asg \
  --preferences '{"MinHealthyPercentage": 0}' \
  --region us-east-1

# Logs de démarrage d'une instance
aws ec2 get-console-output \
  --instance-id <instance-id> \
  --region us-east-1 \
  --output text
```

---

## 🏆 Ce que ce projet démontre

- ✅ **Infrastructure as Code** avec Terraform (HCL)
- ✅ **Haute disponibilité** : ASG sur 2 zones de disponibilité
- ✅ **Auto Scaling** : scale out automatique si CPU > 70%
- ✅ **Sécurité en couches** : Security Groups par tier (ALB → frontend → backend → RDS)
- ✅ **Réseau isolé** : backend et RDS dans des sous-réseaux privés
- ✅ **Déploiement automatisé** : cloud-init pour configurer les instances au démarrage
- ✅ **Load Balancing** : ALB avec health checks automatiques
- ✅ **Séparation des environnements** : variables d'environnement Angular injectées dynamiquement

---

<div align="center">

Fait avec ❤️ · AWS Academy · Terraform · Infrastructure as Code

</div>
