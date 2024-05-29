### Énoncé du Projet

**Projet Final : Création d'une Application FastAPI avec Déploiement sur AWS EC2 et S3**

Votre mission consiste à développer une application web en utilisant le framework FastAPI, puis à déployer cette application sur une instance EC2 d'AWS et utiliser S3 pour le stockage des fichiers statiques. Vous allez suivre les étapes nécessaires pour créer un environnement de développement, structurer votre projet, implémenter les fonctionnalités requises et finalement déployer votre application sur le cloud.

### Étapes du Projet

#### 1. Configuration de l'Environnement de Développement
- **Installer Python et Pip** : Téléchargez et installez Python depuis [python.org](https://www.python.org/).
- **Créer un Environnement Virtuel** :
  ```bash
  python -m venv venv
  source venv/bin/activate  # Sur Windows: venv\Scripts\activate
  ```
- **Installer les Dépendances** :
  ```bash
  pip install fastapi uvicorn sqlalchemy pymysql cryptography pydantic boto3 python-dotenv
  ```
- **Configurer AWS CLI** en suivant [ce guide](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html).

#### 2. Structure du Projet

Le projet doit être organisé comme suit :

```
project/
│
├── main.py
├── .env
├── models/
│   └── models.py
├── api/
│   ├── controllers/
│   │   └── message_controller.py
│   ├── routes/
│   │   └── message_routes.py
├── services/
│   ├── message_service.py
│   └── encryption_service.py
├── repositories/
│   └── message_repository.py
├── config/
│   └── database.py
└── exceptions/
    └── custom_exceptions.py
```

#### 3. Implémentation des Fichiers

**Fichier `.env`**
```dotenv
DATABASE_URL=mysql+pymysql://username:password@localhost/db_name
SECRET_KEY=your-secret-key
```

**Fichier `main.py`**
```python
from fastapi import FastAPI
from api.routes.message_routes import router as message_router
from config.database import engine, Base
from services.encryption_service import EncryptionService
from dotenv import load_dotenv
import os

# Charger les variables d'environnement à partir du fichier .env
load_dotenv()

# Créer l'application FastAPI
app = FastAPI()

# Créer toutes les tables de la base de données
Base.metadata.create_all(bind=engine)

# Récupérer la clé de chiffrement à partir des variables d'environnement
key = os.getenv('SECRET_KEY').encode()
encryption_service = EncryptionService(key=key)

# Inclure les routes du routeur message_router
app.include_router(message_router, prefix="/messages")

@app.on_event("startup")
async def startup_event():
    # Ajouter le service de chiffrement à l'état de l'application
    app.state.encryption_service = encryption_service
```

**Fichier `config/database.py`**
```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from dotenv import load_dotenv
import os

# Charger les variables d'environnement à partir du fichier .env
load_dotenv()

# Récupérer l'URL de la base de données à partir des variables d'environnement
DATABASE_URL = os.getenv("DATABASE_URL")

# Créer le moteur SQLAlchemy
engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

**Fichier `models/message.py`**
```python
from sqlalchemy import Column, Integer, String
from config.database import Base

# Définir le modèle de message
class Message(Base):
    __tablename__ = "messages"

    id = Column(Integer, primary_key=True, index=True)
    content = Column(String, index=True)
    encrypted_content = Column(String)
    hash_value = Column(String)
```

**Fichier `repositories/message_repository.py`**
```python
from sqlalchemy.orm import Session
from models.message import Message
from exceptions.custom_exceptions import MessageNotFoundException

# Définir le référentiel de messages
class MessageRepository:
    def __init__(self, db: Session):
        self.db = db

    def create_message(self, content: str, encrypted_content: str, hash_value: str):
        try:
            db_message = Message(content=content, encrypted_content=encrypted_content, hash_value=hash_value)
            self.db.add(db_message)
            self.db.commit()
            self.db.refresh(db_message)
            return db_message
        except Exception as e:
            self.db.rollback()
            raise e

    def get_message_by_id(self, message_id: int):
        try:
            message = self.db.query(Message).filter(Message.id == message_id).first()
            if not message:
                raise MessageNotFoundException(message_id)
            return message
        except Exception as e:
            raise e

    def get_all_messages(self):
        try:
            return self.db.query(Message).all()
        except Exception as e:
            raise e
```

**Fichier `services/encryption_service.py`**
```python
from cryptography.fernet import Fernet
import hashlib

# Définir le service de chiffrement
class EncryptionService:
    def __init__(self, key: bytes):
        self.cipher = Fernet(key)

    def encrypt(self, message: str) -> str:
        return self.cipher.encrypt(message.encode()).decode()

    def decrypt(self, encrypted_message: str) -> str:
        return self.cipher.decrypt(encrypted_message.encode()).decode()

    def generate_hash(self, message: str) -> str:
        return hashlib.sha256(message.encode()).hexdigest()

    def compare_hash(self, message: str, hash_value: str) -> bool:
        return self.generate_hash(message) == hash_value
```

**Fichier `services/message_service.py`**
```python
from sqlalchemy.orm import Session
from repositories.message_repository import MessageRepository
from services.encryption_service import EncryptionService
from exceptions.custom_exceptions import MessageNotFoundException

# Définir le service de messages
class MessageService:
    def __init__(self, db: Session, encryption_service: EncryptionService):
        self.message_repository = MessageRepository(db)
        self.encryption_service = encryption_service

    def create_message(self, content: str):
        encrypted_content = self.encryption_service.encrypt(content)
        hash_value = self.encryption_service.generate_hash(content)
        return self.message_repository.create_message(content, encrypted_content, hash_value)

    def get_message(self, message_id: int):
        message = self.message_repository.get_message_by_id(message_id)
        decrypted_content = self.encryption_service.decrypt(message.encrypted_content)
        return {"content": decrypted_content, "hash_value": message.hash_value}

    def list_messages(self):
        messages = self.message_repository.get_all_messages()
        return [{"id": msg.id, "content": self.encryption_service.decrypt(msg.encrypted_content), "hash_value": msg.hash_value} for msg in messages]
```

**Fichier `api/controllers/message_controller.py`**
```python
from fastapi import Depends, HTTPException
from sqlalchemy.orm import Session
from services.message_service import MessageService
from services.encryption_service import EncryptionService
from config.database import get_db
from exceptions.custom_exceptions import MessageNotFoundException

# Définir le contrôleur de messages
class MessageController:
    def __init__(self, db: Session = Depends(get_db), encryption_service: EncryptionService = Depends()):
        self.db = db
        self.encryption_service = encryption_service
        self.message_service = MessageService(db, encryption_service)

    def create_message(self, content: str):
        try:
            return self.message_service.create_message(content)
        except Exception as e:
            raise HTTPException(status_code=500, detail=str(e))

    def get_message(self, message_id: int):
        try:
            return self.message_service.get_message(message_id)
        except MessageNotFoundException as e:
            raise HTTPException(status_code=404, detail=str(e))
        except Exception as e:
            raise HTTPException(status_code=500, detail=str(e))

    def list_messages(self):
        try:
            return self.message_service.list_messages()
        except Exception as e:
            raise HTTPException(status_code=500, detail=str(e))
```

**Fichier `api/routes/message_routes.py`**
```python
from fastapi import APIRouter, Depends
from pydantic import BaseModel
from api.controllers.message_controller import MessageController

router = APIRouter()

# Définir le schéma de création de message
class MessageCreate(BaseModel):
    content: str

# Route pour créer un message
@router.post("/")
def create_message(message: MessageCreate, controller: MessageController = Depends()):
    return controller.create_message(message.content)

# Route pour lire un message par son ID
@router.get("/{message_id}")
def read_message(message_id: int, controller: MessageController = Depends()):
    return controller.get_message(message_id)

# Route pour lister tous les messages
@router.get("/")
def list_messages(controller: MessageController = Depends()):
    return controller.list_messages()
```

**Fichier `exceptions/custom_exceptions.py`**
```python
# Définir l'exception MessageNotFoundException
class MessageNotFoundException(Exception):
    def __init__(self, message

_id: int):
        self.message = f"Message with ID {message_id} not found"
        super().__init__(self.message)
```

### Sécurité

1. **Utiliser HTTPS** : Assurez-vous que votre API utilise HTTPS pour sécuriser les communications.
2. **Authentification et Autorisation** : Implémentez des mécanismes d'authentification (comme OAuth2, JWT) pour protéger les endpoints de votre API.
3. **Gestion des Erreurs** : Manipulez correctement les exceptions pour éviter la divulgation d'informations sensibles.

### Déploiement sur AWS EC2 et S3

#### 1. Préparation de l'Instance EC2
- **Lancer une Instance EC2** : Connectez-vous à AWS Management Console et lancez une nouvelle instance EC2 (Amazon Linux 2 ou Ubuntu).
- **Configurer l'Instance EC2** : Connectez-vous à l'instance EC2 via SSH et installez les dépendances nécessaires :
  ```bash
  sudo apt update
  sudo apt install python3 python3-pip
  ```

#### 2. Déploiement de l'Application
- **Transférer le Code Source** : Transférez votre code source sur l'instance EC2 (utilisez `scp` ou `rsync`).
- **Installer les Dépendances** dans un environnement virtuel :
  ```bash
  python3 -m venv venv
  source venv/bin/activate
  pip install fastapi uvicorn sqlalchemy pymysql cryptography pydantic boto3 python-dotenv
  ```

#### 3. Configuration du Serveur Web et du Service Systemd
- **Configurer NGINX** pour servir votre application :
  ```nginx
  server {
      listen 80;
      server_name your_domain_or_ip;

      location / {
          proxy_pass http://127.0.0.1:8000;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
      }
  }
  ```
  Redémarrez NGINX :
  ```bash
  sudo systemctl restart nginx
  ```

- **Configurer un Service Systemd** pour démarrer votre application :
  ```ini
  [Unit]
  Description=Gunicorn instance to serve FastAPI
  After=network.target

  [Service]
  User=ubuntu
  Group=www-data
  WorkingDirectory=/path/to/your/project
  Environment="PATH=/path/to/your/project/venv/bin"
  ExecStart=/path/to/your/project/venv/bin/gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app

  [Install]
  WantedBy=multi-user.target
  ```
  Redémarrez le service :
  ```bash
  sudo systemctl daemon-reload
  sudo systemctl start your_service_name
  sudo systemctl enable your_service_name
  ```

#### 4. Configuration de S3 pour Stocker les Fichiers Statics
- **Créer un Bucket S3** et configurez-le pour héberger des fichiers statiques.
- **Configurer l'Application** pour utiliser S3 pour le stockage des fichiers statiques.

### Tests Unitaires

**Fichier `tests/test_message_service.py`**
```python
import pytest
from services.encryption_service import EncryptionService
from services.message_service import MessageService
from repositories.message_repository import MessageRepository
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from config.database import Base

# Définir les paramètres de la base de données de test
DATABASE_URL = "sqlite:///./test.db"
engine = create_engine(DATABASE_URL)
TestingSessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# Fixture pour la base de données de test
@pytest.fixture(scope="module")
def db():
    Base.metadata.create_all(bind=engine)
    db = TestingSessionLocal()
    yield db
    db.close()
    Base.metadata.drop_all(bind=engine)

# Fixture pour le service de chiffrement
@pytest.fixture(scope="module")
def encryption_service():
    key = b'test-key-12345678901234567890123456789012'
    return EncryptionService(key)

# Fixture pour le service de messages
@pytest.fixture(scope="module")
def message_service(db, encryption_service):
    return MessageService(db, encryption_service)

# Test pour la création de message
def test_create_message(message_service):
    content = "Hello, world!"
    message = message_service.create_message(content)
    assert message.content == content

# Test pour la récupération de message
def test_get_message(message_service):
    content = "Hello, world!"
    created_message = message_service.create_message(content)
    fetched_message = message_service.get_message(created_message.id)
    assert fetched_message['content'] == content
```

### Documentation

Utilisez Swagger pour générer automatiquement la documentation de votre API. FastAPI intègre déjà Swagger, accessible via `/docs`.