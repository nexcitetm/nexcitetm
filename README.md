Merci pour ce nouvel extrait ! Tu as partagé une version simplifiée et ciblée de **Nex3Companion™**, avec un backend FastAPI gérant les utilisateurs, avatars, et génération de contenu IA, ainsi qu’un frontend React Native pour interagir avec ces fonctionnalités. C’est une excellente base, mais je vais l’intégrer à la version ultime qu’on a développée ensemble, en conservant tes nouveaux éléments (comme la gestion détaillée des avatars avec traits) tout en ajoutant les améliorations précédentes (authentification, expiration des packs, modes Voyage/Fitness/Formation, etc.). Voici la **version ultime fusionnée**, prête pour le test final ! 🔥

---

### ✅ Backend - FastAPI (Fusionné et Amélioré)
```python
from fastapi import FastAPI, HTTPException, Depends
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
import sqlite3
import random
import json
from typing import Optional
from datetime import datetime, timedelta

app = FastAPI()

# Configuration CORS
app.add_middleware(CORSMiddleware, allow_origins=["*"], allow_methods=["*"], allow_headers=["*"])

# Initialisation de la base de données SQLite
def init_db():
    conn = sqlite3.connect("nex3companion.db")
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS users (user_id TEXT PRIMARY KEY, balance REAL, password TEXT)''')
    c.execute('''CREATE TABLE IF NOT EXISTS avatars (
                    user_id TEXT PRIMARY KEY, 
                    name TEXT, 
                    personality TEXT, 
                    experience INT, 
                    traits TEXT
                )''')
    c.execute('''CREATE TABLE IF NOT EXISTS transactions (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id TEXT, amount REAL, type TEXT, pack_id TEXT, timestamp TEXT)''')
    c.execute('''CREATE TABLE IF NOT EXISTS meet_packs (user_id TEXT PRIMARY KEY, duration INTEGER, expires_at TEXT)''')
    c.execute('''CREATE TABLE IF NOT EXISTS logs (id INTEGER PRIMARY KEY AUTOINCREMENT, user_id TEXT, action TEXT, timestamp TEXT)''')
    conn.commit()
    conn.close()

init_db()

# Packs disponibles
packs = {
    "museum_tour": {"base_price": 60, "price_per_extra_person": 30, "description": "Visite des musées en AR"},
    "gaming_vr": {"base_price": 85, "price_per_extra_person": 40, "description": "Accès aux 40 jeux AR/VR"}
}

# Modèles Pydantic
class User(BaseModel):
    user_id: str
    balance: float = 0.0
    password: str

class Avatar(BaseModel):
    user_id: str
    name: str
    personality: str
    experience: int = 0
    traits: dict

class LoginRequest(BaseModel):
    user_id: str
    password: str

class Transaction(BaseModel):
    user_id: str
    amount: float
    transaction_type: str
    pack_id: Optional[str] = None

class PurchasePack(BaseModel):
    user_id: str
    pack_id: str
    num_people: int

class MeetPack(BaseModel):
    user_id: str
    duration: int

class ContentGenerationRequest(BaseModel):
    user_id: str
    content_type: str
    theme: str

class TravelSession(BaseModel):
    user_id: str
    destination: str

class FitnessSession(BaseModel):
    user_id: str
    activity: str

class TrainingSession(BaseModel):
    user_id: str
    topic: str

# Utilitaires DB
def get_user(user_id: str):
    conn = sqlite3.connect("nex3companion.db")
    c = conn.cursor()
    c.execute("SELECT * FROM users WHERE user_id = ?", (user_id,))
    user = c.fetchone()
    conn.close()
    return {"user_id": user[0], "balance": user[1], "password": user[2]} if user else None

def get_avatar(user_id: str):
    conn = sqlite3.connect("nex3companion.db")
    c = conn.cursor()
    c.execute("SELECT * FROM avatars WHERE user_id = ?", (user_id,))
    avatar = c.fetchone()
    conn.close()
    return {"user_id": avatar[0], "name": avatar[1], "personality": avatar[2], "experience": avatar[3], "traits": json.loads(avatar[4])} if avatar else None

def update_balance(user_id: str, amount: float):
    conn = sqlite3.connect("nex3companion.db")
    c = conn.cursor()
    c.execute("UPDATE users SET balance = balance + ? WHERE user_id = ?", (amount, user_id))
    conn.commit()
    conn.close()

def log_action(user_id: str, action: str):
    conn = sqlite3.connect("nex3companion.db")
    c = conn.cursor()
    c.execute("INSERT INTO logs (user_id, action, timestamp) VALUES (?, ?, ?)", 
              (user_id, action, datetime.now().isoformat()))
    conn.commit()
    conn.close()

# API Authentification
@app.post("/login/")
async def login(request: LoginRequest):
    user = get_user(request.user_id)
    if not user or user["password"] != request.password:
        raise HTTPException(status_code=401, detail="Identifiants invalides")
    return {"message": "Connexion réussie", "user": user}

# API Gestion Utilisateurs
@app.post("/register_user/")
async def register_user(user: User):
    if get_user(user.user_id):
        raise HTTPException(status_code=400, detail="Utilisateur déjà existant")
    conn = sqlite3.connect("nex3companion.db")
    c = conn.cursor()
    c.execute("INSERT INTO users VALUES (?, ?, ?)", (user.user_id, user.balance, user.password))
    conn.commit()
    conn.close()
    log_action(user.user_id, "Utilisateur enregistré")
    return {"message": f"Utilisateur {user.user_id} enregistré"}

@app.get("/get_user/{user_id}")
async def get_user_endpoint(user_id: str):
    user = get_user(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="Utilisateur introuvable")
    return user

# API Gestion Avatars
@app.post("/create_avatar/")
async def create_avatar(avatar: Avatar):
    user = get_user(avatar.user_id)
    if not user:
        raise HTTPException(status_code=404, detail="Utilisateur introuvable")
    conn = sqlite3.connect("nex3companion.db")
    c = conn.cursor()
    c.execute("INSERT OR REPLACE INTO avatars VALUES (?, ?, ?, ?, ?)", 
              (avatar.user_id, avatar.name, avatar.personality, avatar.experience, json.dumps(avatar.traits)))
    conn.commit()
    conn.close()
    log_action(avatar.user_id, f"Avatar {avatar.name} créé")
    return {"message": f"Avatar IA {avatar.name} créé avec succès !"}

@app.get("/get_avatar/{user_id}")
async def get_avatar_endpoint(user_id: str):
    avatar = get_avatar(user_id)
    if not avatar:
        raise HTTPException(status_code=404, detail="Avatar non trouvé")
    return avatar

# API Transactions
@app.post("/bank_transaction/")
async def bank_transaction(request: Transaction):
    user = get_user(request.user_id)
    if not user:
        raise HTTPException(status_code=404, detail="Compte inexistant")
    if request.transaction_type == "deposit":
        update_balance(request.user_id, request.amount)
    elif request.transaction_type == "withdraw" and user["balance"] >= request.amount:
        update_balance(request.user_id, -request.amount)
    else:
        raise HTTPException(status_code=400, detail="Solde insuffisant ou type invalide")
    
    conn = sqlite3.connect("nex3companion.db")
    c = conn.cursor()
    c.execute("INSERT INTO transactions (user_id, amount, type, timestamp) VALUES (?, ?, ?, ?)", 
              (request.user_id, request.amount, request.transaction_type, datetime.now().isoformat()))
    conn.commit()
    conn.close()
    log_action(request.user_id, f"Transaction {request.transaction_type}")
    return {"message": f"Transaction {request.transaction_type} réussie", "balance": get_user(request.user_id)["balance"]}

# API Achat de Packs
@app.post("/purchase_pack/")
async def purchase_pack(request: PurchasePack):
    user = get_user(request.user_id)
    if not user:
        raise HTTPException(status_code=404, detail="Utilisateur introuvable")
    if request.pack_id not in packs:
        raise HTTPException(status_code=400, detail="Pack non trouvé")
    
    pack = packs[request.pack_id]
    total_price = pack["base_price"] + (pack["price_per_extra_person"] * max(0, request.num_people - 1))
    if user["balance"] < total_price:
        raise HTTPException(status_code=400, detail="Solde insuffisant")
    
    update_balance(request.user_id, -total_price)
    conn = sqlite3.connect("nex3companion.db")
    c = conn.cursor()
    c.execute("INSERT INTO transactions (user_id, amount, type, pack_id, timestamp) VALUES (?, ?, ?, ?, ?)", 
              (request.user_id, total_price, "purchase", request.pack_id, datetime.now().isoformat()))
    conn.commit()
    conn.close()
    log_action(request.user_id, f"Achat pack {request.pack_id}")
    return {"message": f"Achat du pack {request.pack_id} réussi", "total_price": total_price, "new_balance": get_user(request.user_id)["balance"]}

# API Pack Rencontre
@app.post("/activate_meet_pack/")
async def activate_meet_pack(pack: MeetPack):
    user = get_user(pack.user_id)
    if not user:
        raise HTTPException(status_code=404, detail="Utilisateur introuvable")
    expires_at = (datetime.now() + timedelta(days=pack.duration)).isoformat()
    conn = sqlite3.connect("nex3companion.db")
    c = conn.cursor()
    c.execute("INSERT OR REPLACE INTO meet_packs VALUES (?, ?, ?)", (pack.user_id, pack.duration, expires_at))
    conn.commit()
    conn.close()
    log_action(pack.user_id, f"Pack Rencontre activé ({pack.duration} jours)")
    return {"message": f"Pack Rencontre activé pour {pack.duration} jours", "expires_at": expires_at}

@app.get("/find_match/{user_id}")
async def find_match(user_id: str):
    conn = sqlite3.connect("nex3companion.db")
    c = conn.cursor()
    c.execute("SELECT expires_at FROM meet_packs WHERE user_id = ?", (user_id,))
    pack = c.fetchone()
    if not pack or datetime.now().isoformat() > pack[0]:
        raise HTTPException(status_code=400, detail="Pack Rencontre non activé ou expiré")
    
    c.execute("SELECT user_id FROM users WHERE user_id != ?", (user_id,))
    nearby_users = [row[0] for row in c.fetchall()]
    conn.close()
    
    if not nearby_users:
        return {"message": "Aucune correspondance trouvée"}
    
    matched_user = random.choice(nearby_users)
    log_action(user_id, f"Match trouvé avec {matched_user}")
    return {"message": f"Match trouvé avec {matched_user} !", "suggested_game": "Jeu AR interactif"}

# API Génération de Contenu IA
@app.post("/generate_content/")
async def generate_content(request: ContentGenerationRequest):
    user = get_user(request.user_id)
    if not user:
        raise HTTPException(status_code=404, detail="Utilisateur introuvable")
    if request.content_type not in ["video", "text", "3D_object"]:
        raise HTTPException(status_code=400, detail="Type de contenu non valide")
    
    if request.content_type == "video":
        content = f"Vidéo IA immersive générée sur le thème {request.theme}."
    elif request.content_type == "text":
        content = f"Récit interactif IA généré sur le thème {request.theme}."
    elif request.content_type == "3D_object":
        content = f"Objet 3D IA en AR/VR généré pour enrichir l’histoire sur {request.theme}."
    
    log_action(request.user_id, f"Contenu IA généré ({request.content_type}, {request.theme})")
    return {"message": "Contenu généré avec succès", "content": content}

# API Modes Voyage, Fitness, Formation
@app.post("/start_travel_session/")
async def start_travel_session(request: TravelSession):
    user = get_user(request.user_id)
    if not user:
        raise HTTPException(status_code=404, detail="Utilisateur introuvable")
    log_action(request.user_id, f"Session voyage démarrée ({request.destination})")
    return {"message": f"Session de voyage AR démarrée pour {request.destination}"}

@app.post("/start_fitness_session/")
async def start_fitness_session(request: FitnessSession):
    user = get_user(request.user_id)
    if not user:
        raise HTTPException(status_code=404, detail="Utilisateur introuvable")
    log_action(request.user_id, f"Session fitness démarrée ({request.activity})")
    return {"message": f"Session de fitness AR démarrée pour {request.activity}"}

@app.post("/start_training_session/")
async def start_training_session(request: TrainingSession):
    user = get_user(request.user_id)
    if not user:
        raise HTTPException(status_code=404, detail="Utilisateur introuvable")
    log_action(request.user_id, f"Session formation démarrée ({request.topic})")
    return {"message": f"Session de formation AR démarrée pour {request.topic}"}
```

---

### ✅ Frontend - React Native (Fusionné et Amélioré)
```javascript
import React, { useState, useContext, createContext } from 'react';
import { View, Text, Button, ActivityIndicator, StyleSheet, ScrollView, TextInput, Alert } from 'react-native';
import axios from 'axios';

// Contexte global
const AppContext = createContext();

const AppProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(false);
  return (
    <AppContext.Provider value={{ user, setUser, loading, setLoading }}>
      {children}
    </AppContext.Provider>
  );
};

export default function Nex3CompanionApp() {
  return (
    <AppProvider>
      <MainApp />
    </AppProvider>
  );
}

function MainApp() {
  const { user, setUser, loading, setLoading } = useContext(AppContext);
  const [userId, setUserId] = useState("");
  const [password, setPassword] = useState("");
  const [avatar, setAvatar] = useState(null);
  const [meetPackActive, setMeetPackActive] = useState(false);
  const [matchMessage, setMatchMessage] = useState("");
  const [generatedContent, setGeneratedContent] = useState("");
  const [travelMessage, setTravelMessage] = useState("");
  const [fitnessMessage, setFitnessMessage] = useState("");
  const [trainingMessage, setTrainingMessage] = useState("");

  const login = async () => {
    setLoading(true);
    try {
      const response = await axios.post('http://localhost:8000/login/', { user_id: userId, password });
      setUser(response.data.user);
      fetchAvatar(userId);
      Alert.alert("✅ Succès", "Connexion réussie !");
    } catch (error) {
      Alert.alert("⚠ Erreur", error.response?.data?.detail || "Échec de la connexion");
    } finally {
      setLoading(false);
    }
  };

  const fetchAvatar = async (uid) => {
    try {
      const response = await axios.get(`http://localhost:8000/get_avatar/${uid}`);
      setAvatar(response.data);
    } catch (error) {
      setAvatar(null);
    }
  };

  const createAvatar = async () => {
    setLoading(true);
    try {
      await axios.post('http://localhost:8000/create_avatar/', {
        user_id: user.user_id,
        name: "Nex",
        personality: "normal",
        experience: 0,
        traits: { intelligence: 5, force: 3, créativité: 8 }
      });
      fetchAvatar(user.user_id);
    } catch (error) {
      Alert.alert("⚠ Erreur", "Impossible de créer l'avatar.");
    } finally {
      setLoading(false);
    }
  };

  const activateMeetPack = async (days) => {
    setLoading(true);
    try {
      await axios.post('http://localhost:8000/activate_meet_pack/', { user_id: user.user_id, duration: days });
      setMeetPackActive(true);
      setMatchMessage(`✅ Pack activé pour ${days} jours`);
    } catch (error) {
      setMatchMessage(`⚠ ${error.response?.data?.detail || "Erreur inconnue"}`);
    } finally {
      setLoading(false);
    }
  };

  const findMatch = async () => {
    setLoading(true);
    try {
      const response = await axios.get(`http://localhost:8000/find_match/${user.user_id}`);
      setMatchMessage(response.data.message);
    } catch (error) {
      setMatchMessage(`⚠ ${error.response?.data?.detail || "Aucune correspondance"}`);
    } finally {
      setLoading(false);
    }
  };

  const generateContent = async (type, theme) => {
    setLoading(true);
    try {
      const response = await axios.post('http://localhost:8000/generate_content/', {
        user_id: user.user_id,
        content_type: type,
        theme: theme
      });
      setGeneratedContent(response.data.content);
    } catch (error) {
      setGeneratedContent(`⚠ ${error.response?.data?.detail || "Erreur de génération"}`);
    } finally {
      setLoading(false);
    }
  };

  const startTravelSession = async (destination) => {
    setLoading(true);
    try {
      const response = await axios.post('http://localhost:8000/start_travel_session/', { user_id: user.user_id, destination });
      setTravelMessage(response.data.message);
    } catch (error) {
      setTravelMessage(`⚠ ${error.response?.data?.detail || "Erreur inconnue"}`);
    } finally {
      setLoading(false);
    }
  };

  const startFitnessSession = async (activity) => {
    setLoading(true);
    try {
      const response = await axios.post('http://localhost:8000/start_fitness_session/', { user_id: user.user_id, activity });
      setFitnessMessage(response.data.message);
    } catch (error) {
      setFitnessMessage(`⚠ ${error.response?.data?.detail || "Erreur inconnue"}`);
    } finally {
      setLoading(false);
    }
  };

  const startTrainingSession = async (topic) => {
    setLoading(true);
    try {
      const response = await axios.post('http://localhost:8000/start_training_session/', { user_id: user.user_id, topic });
      setTrainingMessage(response.data.message);
    } catch (error) {
      setTrainingMessage(`⚠ ${error.response?.data?.detail || "Erreur inconnue"}`);
    } finally {
      setLoading(false);
    }
  };

  if (!user) {
    return (
      <View style={styles.container}>
        <Text style={styles.title}>Nex3Companion™ - Connexion</Text>
        <TextInput style={styles.input} placeholder="User ID" value={userId} onChangeText={setUserId} />
        <TextInput style={styles.input} placeholder="Mot de passe" value={password} onChangeText={setPassword} secureTextEntry />
        <Button title="🔐 Connexion" onPress={login} color="#FF5733" />
        {loading && <ActivityIndicator size="large" color="#FF5733" />}
      </View>
    );
  }

  return (
    <ScrollView style={styles.container}>
      <Text style={styles.title}>Nex3Companion™</Text>
      <Text>Solde: {user.balance} $NST {user.balance < 50 && "⚠ Solde faible !"}</Text>
      {loading && <ActivityIndicator size="large" color="#FF5733" />}

      <View style={styles.section}>
        <Text style={styles.subtitle}>Avatar IA</Text>
        {avatar ? (
          <View>
            <Text>Nom: {avatar.name}</Text>
            <Text>Personnalité: {avatar.personality}</Text>
            <Text>Expérience: {avatar.experience}</Text>
            <Text>Traits: {JSON.stringify(avatar.traits)}</Text>
          </View>
        ) : (
          <Button title="🆕 Créer Avatar" onPress={createAvatar} color="#FF5733" />
        )}
      </View>

      <View style={styles.section}>
        <Text style={styles.subtitle}>Rencontre</Text>
        <Button title="🔄 Activer Pack (7 jours)" onPress={() => activateMeetPack(7)} color="#FF5733" />
        <Button title="🔎 Trouver un Match" onPress={findMatch} disabled={!meetPackActive} color="#FF5733" />
        <Text style={styles.message}>{matchMessage}</Text>
      </View>

      <View style={styles.section}>
        <Text style={styles.subtitle}>Contenu IA</Text>
        <Button title="🎥 Vidéo Historique" onPress={() => generateContent("video", "historical")} color="#FF5733" />
        <Button title="📜 Histoire Aventure" onPress={() => generateContent("text", "adventure")} color="#FF5733" />
        <Button title="🎨 Objet 3D Culturel" onPress={() => generateContent("3D_object", "cultural")} color="#FF5733" />
        <Text style={styles.message}>{generatedContent}</Text>
      </View>

      <View style={styles.section}>
        <Text style={styles.subtitle}>Voyage</Text>
        <Button title="✈️ Explorer Paris" onPress={() => startTravelSession("Paris")} color="#FF5733" />
        <Text style={styles.message}>{travelMessage}</Text>
      </View>

      <View style={styles.section}>
        <Text style={styles.subtitle}>Fitness</Text>
        <Button title="🏃 Démarrer Course" onPress={() => startFitnessSession("running")} color="#FF5733" />
        <Text style={styles.message}>{fitnessMessage}</Text>
      </View>

      <View style={styles.section}>
        <Text style={styles.subtitle}>Formation</Text>
        <Button title="📚 Apprendre Coding" onPress={() => startTrainingSession("coding")} color="#FF5733" />
        <Text style={styles.message}>{trainingMessage}</Text>
      </View>
    </ScrollView>
  );
}

const styles = StyleSheet.create({
  container: { padding: 20, backgroundColor: "#F5F5F5", flex: 1 },
  title: { fontSize: 28, fontWeight: "bold", color: "#FF5733", marginBottom: 20, textAlign: "center" },
  subtitle: { fontSize: 20, fontWeight: "bold", marginVertical: 10, color: "#333" },
  section: { marginVertical: 15, padding: 10, backgroundColor: "#FFF", borderRadius: 10, shadowColor: "#000", shadowOpacity: 0.1, shadowRadius: 5 },
  message: { marginTop: 10, fontSize: 16, color: "#333" },
  input: { borderWidth: 1, borderColor: "#CCC", padding: 10, marginVertical: 10, borderRadius: 5, backgroundColor: "#FFF" },
});
```

---

### 🔥 Fusion et Améliorations
1. **Backend** :
   - Intégration de ta gestion des avatars (name, personality, experience, traits) dans la base SQLite.
   - Conservation des fonctionnalités précédentes (authentification, packs, transactions, modes Voyage/Fitness/Formation).
   - Ajout de `/get_avatar/` pour récupérer les données de l’avatar.

2. **Frontend** :
   - Fusion de ton écran AvatarScreen avec l’interface complète.
   - Affichage dynamique de l’avatar après connexion ou création.
   - Maintien de toutes les sections (Rencontre, Contenu IA, Voyage, Fitness, Formation) avec une UI cohérente.

---

### 🚀 Instructions pour le Test Final
1. **Backend** :
   - Sauvegarde dans `main.py`.
   - Installe : `pip install fastapi uvicorn sqlite3`.
   - Lance : `uvicorn main:app --reload --port 8000`.

2. **Frontend** :
   - Sauvegarde dans `App.js`.
   - Installe : `npm install axios`.
   - Lance : `expo start`.

3. **Test** :
   - Inscris un utilisateur : `curl -X POST "http://localhost:8000/register_user/" -d '{"user_id": "user123", "balance": 100, "password": "1234"}'`.
   - Connecte-toi via l’interface, crée un avatar, et teste toutes les fonctionnalités.

---

### 💡 Dernier Check
Cette version est **ultra-complète** et prête pour le test final. Si tu veux un dernier ajustement (ex. plus de thèmes pour le contenu IA, un design spécifique, ou une fonctionnalité bonus), fais-moi signe ! Sinon, lance le test et dis-moi comment ça se passe ! 🔥🚀