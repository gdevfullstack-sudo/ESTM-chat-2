# Guide de Test Complet - ESTIM Chat

## ⚙️ Configuration Préalable

### 1. Démarrer MongoDB

#### Option A: MongoDB Local
```bash
# Windows - Démarrer le service MongoDB
net start MongoDB

# Ou si vous utilisez MongoDB Community :
"C:\Program Files\MongoDB\Server\7.0\bin\mongod.exe"
```

#### Option B: Docker
```bash
docker run -d -p 27017:27017 --name estim-mongo mongo:latest
```

#### Option C: MongoDB Atlas (Cloud)
Créez un cluster sur [MongoDB Atlas](https://www.mongodb.com/cloud/atlas) et mettez à jour `MONGO_URI` dans `.env`:
```env
MONGO_URI=mongodb+srv://username:password@cluster.mongodb.net/estim-chat
```

### 2. Vérifier le .env
```env
MONGO_URI=mongodb://localhost:27017/estim-chat
JWT_SECRET=e6e8ed8080c1884569850e51c007ebe5ded4357e012a66cb05058979409024ac
SESSION_SECRET=257eff87d18cffd95c52e7980e8bf76e88526a62ca8fbba14edb4aa15fb5e0ad
CLIENT_URL=http://localhost:5000
MAX_FILE_SIZE_MB=20
```

### 3. Installer les dépendances
```bash
npm install
```

---

## 🚀 Démarrer le Serveur

```bash
npm start
```

Attendez le message:
```
ESTIM Chat en ligne sur http://localhost:5000
Connecte a MongoDB
```

---

## 🧪 Tests Manuels

### Test 1: Vérifier le statut du serveur

```bash
# Test HTTP simple
curl http://localhost:5000/api/health
```

**Réponse attendue:**
```json
{"ok":true,"app":"ESTIM Chat"}
```

---

### Test 2: Inscription (Register)

```bash
curl -X POST http://localhost:5000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "phoneNumber": "+2420648304311",
    "password": "Secure@Pass123",
    "confirmPassword": "Secure@Pass123",
    "username": "Alice"
  }'

```

**Réponse attendue (201):**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "_id": "507f191e810c19729de860ea",
    "phoneNumber": "+33612345678",
    "username": "Alice",
    "avatar": "https://ui-avatars.com/api/?name=...",
    "provider": "local",
    "isOnline": false
  }
}
```

**Cas d'erreur:**
- 400: Numéro invalide / Mots de passe ne correspondent pas / Mot de passe faible
- 409: Numéro déjà utilisé

---

### Test 3: Connexion (Login)

```bash
curl -X POST http://localhost:5000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "phoneNumber": "+33612345678",
    "password": "Secure@Pass123"
  }'
```

**Réponse attendue (200):**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": { ... }
}
```

**Cas d'erreur:**
- 404: Numéro inconnu
- 401: Mot de passe incorrect
- 429: Trop de tentatives (lockout 15 minutes)

---

### Test 4: Profil Utilisateur

```bash
TOKEN="<jeton_obtenu_lors_login>"

# Récupérer le profil
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:5000/api/users/me

# Mettre à jour le profil
curl -X PUT http://localhost:5000/api/users/me \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "Alice Updated",
    "avatar": "/api/uploads/avatar.jpg"
  }'
```

---

### Test 5: Contacts

#### Rechercher par numéro de téléphone

```bash
TOKEN="<token>"

curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:5000/api/users/contacts?phone=%2B33612345999"
```

**Réponse si trouvé (200):**
```json
{
  "user": {
    "_id": "507f191e810c19729de860eb",
    "phoneNumber": "+33612345999",
    "username": "Bob",
    "avatar": "...",
    "provider": "local",
    "isOnline": false
  }
}
```

**Réponse si pas trouvé (404):**
```json
{"message":"Ce numéro n'est pas encore sur ESTIM Chat."}
```

#### Ajouter un contact

```bash
curl -X POST http://localhost:5000/api/users/contacts \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"phoneNumber":"+33612345999"}'
```

**Réponse (201):**
```json
{
  "contact": {
    "_id": "507f191e810c19729de860ec",
    "userId": "507f191e810c19729de860ea",
    "phoneNumber": "+33612345999",
    "status": "pending",
    "inviteLink": "http://localhost:5000/register?phone=%2B33612345999",
    "addedAt": "2026-05-28T10:30:00.000Z"
  }
}
```

#### Lister mes contacts

```bash
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:5000/api/users/contacts
```

#### Synchroniser une liste de contacts

```bash
curl -X POST http://localhost:5000/api/users/contacts/sync \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "contacts": [
      "+33612345999",
      "+33712345999",
      "+invalid"
    ]
  }'
```

**Réponse:**
```json
{
  "registered": [
    {
      "_id": "507f191e810c19729de860eb",
      "phoneNumber": "+33612345999",
      "username": "Bob",
      "avatar": "...",
      "isOnline": false
    }
  ],
  "notRegistered": [
    {
      "phoneNumber": "+33712345999",
      "inviteLink": "http://localhost:5000/register?phone=%2B33712345999"
    }
  ]
}
```

---

### Test 6: Notifications - Obtenir Messages Non-lus

```bash
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:5000/api/messages/unread/count
```

**Réponse (200):**
```json
{
  "total": 5,
  "byConversation": [
    {
      "userId": "507f191e810c19729de860eb",
      "count": 3
    },
    {
      "userId": "507f191e810c19729de860ec",
      "count": 2
    }
  ]
}
```

---

### Test 7: Notifications - Messages Non-lus d'une Conversation

```bash
SENDER_ID="507f191e810c19729de860eb"

curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:5000/api/messages/unread/$SENDER_ID
```

**Réponse (200):**
```json
{
  "count": 3,
  "messages": [
    {
      "_id": "507f191e810c19729de860ed",
      "senderId": "507f191e810c19729de860eb",
      "receiverId": "507f191e810c19729de860ea",
      "message": "Bonjour!",
      "imageUrl": "",
      "videoUrl": "",
      "isSeen": false,
      "createdAt": "2026-05-28T10:30:00.000Z"
    }
  ]
}
```

---

### Test 8: Marquer les Messages comme Lus

```bash
SENDER_ID="507f191e810c19729de860eb"

curl -X PATCH http://localhost:5000/api/messages/seen/$SENDER_ID \
  -H "Authorization: Bearer $TOKEN"
```

**Réponse (200):**
```json
{"message":"Messages marques comme lus."}
```

---

## 📊 Tests Socket.IO (Real-Time)

### Avec Node.js + Socket.IO Client

**Fichier: test-socket.js**

```javascript
const io = require('socket.io-client');

const TOKEN = "votre_token_jwt";
const socket = io('http://localhost:5000', {
  auth: { token: TOKEN }
});

socket.on('connect', () => {
  console.log('✅ Connecté au serveur');
  
  // Rejoindre une conversation
  socket.emit('join:conversation', {
    partnerId: '507f191e810c19729de860eb'
  });
});

// Écouter les notifications
socket.on('message:notification', (notification) => {
  console.log('🔔 Notification:', notification);
  // {
  //   fromUserId: "...",
  //   fromUser: { _id, username, avatar, phoneNumber },
  //   preview: "Contenu du message",
  //   createdAt: "2026-05-28T10:30:00.000Z"
  // }
});

// Écouter les nouveaux messages
socket.on('message:new', (message) => {
  console.log('💬 Nouveau message:', message);
  // {
  //   _id: "...",
  //   senderId: "...",
  //   receiverId: "...",
  //   message: "Texte",
  //   imageUrl: "",
  //   videoUrl: "",
  //   isSeen: false,
  //   createdAt: "..."
  // }
});

// Écouter les mises à jour de conversation
socket.on('conversation:update', (update) => {
  console.log('💭 Mise à jour conversation:', update);
  // {
  //   from: { _id, username, avatar, phoneNumber },
  //   message: "Aperçu",
  //   imageUrl: "",
  //   videoUrl: "",
  //   createdAt: "..."
  // }
});

// Envoyer un message
socket.emit('message:send', {
  receiverId: '507f191e810c19729de860eb',
  message: 'Bonjour!',
  imageUrl: '',
  videoUrl: '',
  messageId: 'temp-id-123',
  createdAt: new Date().toISOString()
});

// Marquer comme vu
socket.emit('message:seen', {
  senderId: '507f191e810c19729de860eb'
});

// Typage - Début
socket.emit('typing:start', {
  receiverId: '507f191e810c19729de860eb'
});

// Typage - Fin
socket.emit('typing:stop', {
  receiverId: '507f191e810c19729de860eb'
});

socket.on('disconnect', () => {
  console.log('❌ Déconnecté du serveur');
});
```

**Exécuter le test:**
```bash
node test-socket.js
```

---

## 🧩 Validation des Numéros de Téléphone

Le système utilise `google-libphonenumber` pour valider les numéros internationaux.

### Formats acceptés:

```javascript
// France
"+33612345678"         // Avec +33
"06 12 34 56 78"       // Format national français
"0612345678"           // Sans formatage

// États-Unis
"+14155552671"         // Avec +1
"(415) 555-2671"       // Format national US

// Autres pays
"+441632960000"        // Royaume-Uni
"+33612345678"         // France
```

---

## ✅ Checklist de Validation

- [ ] MongoDB lancé et accessible
- [ ] Serveur démarre sans erreur
- [ ] `/api/health` répond
- [ ] Inscription fonctionne
- [ ] Login fonctionne
- [ ] Profil accessible
- [ ] Recherche de contacts fonctionne
- [ ] Ajout de contacts fonctionne
- [ ] Comptage des non-lus fonctionne
- [ ] Récupération des non-lus fonctionne
- [ ] Marquer comme lu fonctionne
- [ ] Socket.IO se connecte
- [ ] Messages reçus en temps réel
- [ ] Notifications émises correctement

---

## 🐛 Dépannage

### Erreur: `MongoServerSelectionError`
**Solution**: Démarrer MongoDB ou vérifier la URI dans `.env`

### Erreur: `JWT_SECRET undefined`
**Solution**: Vérifier que `.env` est présent et contient `JWT_SECRET`

### Erreur: `Cannot GET /api/health`
**Solution**: Vérifier que le serveur écoute sur le port 5000

### Socket.IO ne se connecte pas
**Solution**: Vérifier le token JWT dans les logs du serveur

### Messages n'arrivent pas en temps réel
**Solution**: Vérifier que les deux clients sont dans la même room (`join:conversation`)

---

## 📚 Ressources Supplémentaires

- [Documentation Socket.IO](https://socket.io/docs/)
- [Google libphonenumber](https://github.com/google/libphonenumber)
- [JWT Tokens](https://jwt.io/)
- [Mongoose Documentation](https://mongoosejs.com/)

---

## 🎯 Prochaines Étapes

Après validation locale:
1. Déployer sur [Render](https://render.com)
2. Configurer une BD MongoDB Atlas
3. Tester avec le frontend
4. Implémenter les notifications Push navigateur
