# volleyballstattracker
volleyball_stats/
│
├── backend/
│   ├── main.py
│   ├── database.py
│   ├── models.py
│   ├── schemas.py
│   ├── auth.py
│   ├── analytics.py
│   ├── rotation.py
│   ├── websocket_manager.py
│   ├── subscription.py
│   └── requirements.txt
│
├── frontend/
│   ├── index.html
│   ├── coach.html
│   ├── parent.html
│   ├── style.css
│   ├── app.js
│   └── manifest.json
│
├── docker-compose.yml
├── Dockerfile
└── README.md
fastapi
uvicorn
sqlalchemy
psycopg2-binary
python-jose
passlib[bcrypt]
stripe
websockets
python-multipart
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, declarative_base

DATABASE_URL = "postgresql://postgres:password@db:5432/volleyball"

engine = create_engine(DATABASE_URL)
SessionLocal = sessionmaker(bind=engine)
Base = declarative_base()
from sqlalchemy import Column, Integer, String, ForeignKey, JSON
from sqlalchemy.orm import relationship
from database import Base

class Organization(Base):
    __tablename__ = "organizations"
    id = Column(Integer, primary_key=True)
    name = Column(String)
    type = Column(String)  # HS or Club
    theme_color = Column(String)

class Team(Base):
    __tablename__ = "teams"
    id = Column(Integer, primary_key=True)
    name = Column(String)
    organization_id = Column(Integer, ForeignKey("organizations.id"))

class Player(Base):
    __tablename__ = "players"
    id = Column(Integer, primary_key=True)
    name = Column(String)
    number = Column(Integer)
    position = Column(String)
    team_id = Column(Integer, ForeignKey("teams.id"))

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    email = Column(String, unique=True)
    password = Column(String)
    role = Column(String)  # admin coach parent
    player_id = Column(Integer, nullable=True)

class Match(Base):
    __tablename__ = "matches"
    id = Column(Integer, primary_key=True)
    team_id = Column(Integer, ForeignKey("teams.id"))
    opponent = Column(String)
    stats = Column(JSON, default={})
from datetime import datetime, timedelta
from jose import jwt
from passlib.context import CryptContext

SECRET_KEY = "SUPER_SECRET"
ALGORITHM = "HS256"

pwd = CryptContext(schemes=["bcrypt"], deprecated="auto")

def hash_password(password):
    return pwd.hash(password)

def verify_password(p, h):
    return pwd.verify(p, h)

def create_token(data):
    expire = datetime.utcnow() + timedelta(hours=12)
    data.update({"exp": expire})
    return jwt.encode(data, SECRET_KEY, algorithm=ALGORITHM)
def hitting_percentage(kills, errors, attempts):
    if attempts == 0:
        return 0
    return round((kills - errors) / attempts, 3)

def serve_percentage(in_serves, total):
    if total == 0:
        return 0
    return round(in_serves / total, 3)

def pass_average(p1, p2, p3):
    total = p1 + p2 + p3
    if total == 0:
        return 0
    score = (1*p1 + 2*p2 + 3*p3)
    return round(score / total, 2)
def rotate(lineup):
    return lineup[-1:] + lineup[:-1]
from fastapi import WebSocket

class Manager:
    def __init__(self):
        self.connections = {}

    async def connect(self, websocket: WebSocket, team_id: int):
        await websocket.accept()
        if team_id not in self.connections:
            self.connections[team_id] = []
        self.connections[team_id].append(websocket)

    async def broadcast(self, team_id: int, message: dict):
        for ws in self.connections.get(team_id, []):
            await ws.send_json(message)

manager = Manager()
from fastapi import FastAPI, WebSocket, Depends
from database import Base, engine
from websocket_manager import manager

app = FastAPI()

Base.metadata.create_all(bind=engine)

@app.websocket("/ws/{team_id}")
async def websocket_endpoint(websocket: WebSocket, team_id: int):
    await manager.connect(websocket, team_id)
    while True:
        data = await websocket.receive_json()
        await manager.broadcast(team_id, data)
uvicorn main:app --host 0.0.0.0 --port 8000
body {
    font-family: Arial;
}

button {
    font-size: 22px;
    padding: 20px;
    margin: 10px;
    border-radius: 14px;
}
import stripe
stripe.api_key = "YOUR_STRIPE_KEY"
FROM python:3.11
WORKDIR /app
COPY backend/ .
RUN pip install -r requirements.txt
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
version: "3"
services:
  db:
    image: postgres
    environment:
      POSTGRES_PASSWORD: password
      POSTGRES_DB: volleyball
  backend:
    build: .
    ports:
      - "8000:8000"
    depends_on:
      - db
docker-compose up --build
