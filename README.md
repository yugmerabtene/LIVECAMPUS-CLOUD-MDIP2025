### Cours sur le Déploiement de Code en Production avec EC2 et S3

#### Durée : 1 heure

---

## 1. Introduction
### 1.1. Objectifs du Cours
- Comprendre les bonnes pratiques de déploiement de code en production.
- Apprendre à sécuriser les données sensibles.
- Découvrir l’importance du versioning du code et des données sensibles.
- Utiliser des secrets et des variables environnementales.
- Accéder à un serveur distant via le protocole SSH.

---

## 2. Déployer du Code en Production : Bonnes Pratiques

### 2.1. Sécuriser les Données Sensibles

#### 2.1.1. Utilisation d’AWS Secrets Manager
AWS Secrets Manager vous permet de gérer l'accès aux secrets nécessaires pour accéder à vos applications, services et ressources informatiques en toute sécurité.

```python
import boto3
from botocore.exceptions import NoCredentialsError, PartialCredentialsError

def get_secret(secret_name, region_name="us-west-2"):
    client = boto3.client("secretsmanager", region_name=region_name)
    try:
        response = client.get_secret_value(SecretId=secret_name)
        return response["SecretString"]
    except (NoCredentialsError, PartialCredentialsError):
        print("Error: AWS credentials not found.")
        return None

secret = get_secret("my_secret")
print(secret)
```

### 2.2. Versioning du Code et des Données Sensibles

#### 2.2.1. Utilisation de Git pour le Versioning
Git est un système de contrôle de version décentralisé.

```bash
# Initialiser un dépôt Git
git init

# Ajouter des fichiers au dépôt
git add .

# Valider les changements
git commit -m "Initial commit"
```

#### 2.2.2. Versioning des Données Sensibles dans S3
Amazon S3 vous permet d'activer le versioning pour conserver les versions de vos objets.

```python
import boto3

s3 = boto3.client("s3")

def enable_versioning(bucket_name):
    s3.put_bucket_versioning(
        Bucket=bucket_name,
        VersioningConfiguration={"Status": "Enabled"}
    )

bucket_name = "my-bucket"
enable_versioning(bucket_name)
```

### 2.3. Utilisation de Secrets et de Variables Environnementales

#### 2.3.1. Stockage de Secrets dans AWS SSM Parameter Store
AWS Systems Manager Parameter Store vous permet de stocker des valeurs paramétrées et des secrets.

```python
import boto3

ssm = boto3.client("ssm")

def get_parameter(name):
    response = ssm.get_parameter(Name=name, WithDecryption=True)
    return response["Parameter"]["Value"]

parameter_value = get_parameter("my_parameter")
print(parameter_value)
```

#### 2.3.2. Variables Environnementales en Python
Utiliser le module `os` pour accéder aux variables d'environnement.

```python
import os

# Assigner une valeur à une variable d'environnement
os.environ["MY_VARIABLE"] = "my_value"

# Accéder à la valeur d'une variable d'environnement
my_variable = os.getenv("MY_VARIABLE")
print(my_variable)
```

### 2.4. Accès Distant au Serveur via Protocole SSH

#### 2.4.1. Génération et Utilisation de Clés SSH
Générer une paire de clés SSH pour l'authentification.

```bash
# Générer une paire de clés SSH
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# Ajouter la clé privée à l'agent SSH
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_rsa
```

#### 2.4.2. Connexion à une Instance EC2 via SSH

```bash
# Connectez-vous à une instance EC2
ssh -i "path/to/your-key-pair.pem" ec2-user@your-ec2-instance-public-dns
```

### 2.5. Script de Déploiement Automatisé

#### 2.5.1. Utilisation de Boto3 pour Déployer le Code sur EC2

```python
import boto3

ec2 = boto3.client("ec2")

def deploy_code(instance_id, script):
    ssm = boto3.client("ssm")
    response = ssm.send_command(
        InstanceIds=[instance_id],
        DocumentName="AWS-RunShellScript",
        Parameters={"commands": [script]}
    )
    return response

instance_id = "i-0123456789abcdef0"
script = "cd /path/to/your/app && git pull && ./deploy.sh"
deploy_code(instance_id, script)
```

### 2.6. Gestion de la Configuration avec AWS Systems Manager

#### 2.6.1. Création d’une Configuration Manager

```python
import boto3

ssm = boto3.client("ssm")

def create_parameter(name, value):
    ssm.put_parameter(
        Name=name,
        Value=value,
        Type="String",
        Overwrite=True
    )

create_parameter("my_app_config", "config_value")
```

---

## 3. Conclusion
### 3.1. Récapitulatif
- Sécuriser les données sensibles avec AWS Secrets Manager et SSM Parameter Store.
- Utiliser le versioning pour le code et les données sensibles.
- Gérer les secrets et les variables environnementales efficacement.
- Accéder aux serveurs distants de manière sécurisée via SSH.
- Automatiser le déploiement avec des scripts et AWS Systems Manager.

### 3.2. Questions et Réponses
Prenez un moment pour poser des questions et clarifier les concepts appris pendant cette session.
