# Système de Notifications - ESTIM Chat

## 📱 Vue d'ensemble

Le système de notifications alerte les utilisateurs des nouveaux messages en utilisant Socket.IO en temps réel et des endpoints REST.

---

## 🔔 Événements Socket.IO

### 1. `message:notification`
**Destinataire**: Utilisateur recevant un message (hors conversation)

```javascript
socket.on("message:notification", (data) => {
  console.log("Nouveau message de", data.fromUser.username);
  console.log("Aperçu:", data.preview);
  console.log("Créé à:", data.createdAt);
  
  // data = {
  //   fromUserId: "...",
  //   fromUser: { _id, username, avatar, phoneNumber },
  //   preview: "Contenu du message ou type de media",
  //   createdAt: "ISO 8601 timestamp"
  // }
});
```

**Cas d'usage**:
- L'utilisateur reçoit une notification badge/toast quand un nouveau message arrive
- La notification apparaît **seulement si** l'utilisateur n'est pas en train de lire la conversation

### 2. `message:new`
**Destinataire**: Tous les participants de la conversation (room)

```javascript
socket.on("message:new", (message) => {
  console.log("Nouveau message dans la conversation:", message);
  
  // message = {
  //   _id: "MongoDB ID",
  //   senderId: "...",
  //   receiverId: "...",
  //   message: "Texte du message",
  //   imageUrl: "/api/uploads/...", (optionnel)
  //   videoUrl: "/api/uploads/...", (optionnel)
  //   isSeen: false,
  //   createdAt: "ISO 8601 timestamp"
  // }
});
```

**Cas d'usage**:
- Afficher le message immédiatement dans la conversation ouverte

### 3. `conversation:update`
**Destinataire**: Tous les utilisateurs connectés

```javascript
socket.on("conversation:update", (update) => {
  console.log("Conversation mise à jour avec:", update);
  
  // update = {
  //   from: { _id, username, avatar, phoneNumber },
  //   message: "Aperçu du message",
  //   imageUrl: "/api/uploads/..." ou "",
  //   videoUrl: "/api/uploads/..." ou "",
  //   createdAt: "ISO 8601 timestamp"
  // }
});
```

**Cas d'usage**:
- Mettre à jour la liste des conversations avec le dernier message
- Afficher l'aperçu et l'horodatage dans la liste

---

## 🔗 Endpoints REST

### GET `/api/messages/unread/count`
**Authentification**: JWT requis

Obtient le nombre total de messages non-lus et le décompte par conversation.

```bash
curl -H "Authorization: Bearer <token>" \
  http://localhost:5000/api/messages/unread/count
```

**Réponse** (200):
```json
{
  "total": 5,
  "byConversation": [
    {
      "userId": "507f1f77bcf86cd799439011",
      "count": 3
    },
    {
      "userId": "507f1f77bcf86cd799439012",
      "count": 2
    }
  ]
}
```

**Cas d'usage**:
- Afficher le nombre total de messages non-lus dans l'app
- Afficher le badge sur chaque conversation (nombre de messages non-lus)

---

### GET `/api/messages/unread/:userId`
**Authentification**: JWT requis

Obtient les 50 derniers messages non-lus d'un utilisateur spécifique.

```bash
curl -H "Authorization: Bearer <token>" \
  http://localhost:5000/api/messages/unread/507f1f77bcf86cd799439011
```

**Réponse** (200):
```json
{
  "count": 3,
  "messages": [
    {
      "_id": "507f191e810c19729de860ea",
      "senderId": "507f1f77bcf86cd799439011",
      "receiverId": "507f1f77bcf86cd799439010",
      "message": "Bonjour!",
      "imageUrl": "",
      "videoUrl": "",
      "isSeen": false,
      "createdAt": "2026-05-28T10:30:00.000Z"
    }
  ]
}
```

**Erreurs**:
- `400`: Utilisateur invalide
- `500`: Erreur serveur

**Cas d'usage**:
- Afficher la liste des messages non-lus d'une conversation
- Synchroniser l'état des messages non-lus au démarrage de l'app

---

### PATCH `/api/messages/seen/:userId`
**Authentification**: JWT requis

Marque tous les messages d'un utilisateur comme lus.

```bash
curl -X PATCH \
  -H "Authorization: Bearer <token>" \
  http://localhost:5000/api/messages/seen/507f1f77bcf86cd799439011
```

**Réponse** (200):
```json
{
  "message": "Messages marques comme lus."
}
```

**Erreurs**:
- `400`: Utilisateur invalide
- `500`: Erreur serveur

**Cas d'usage**:
- Marquer tous les messages comme lus quand on ouvre une conversation
- Synchroniser l'état "vu" côté serveur

---

## 📊 Flux de Notification Complet

### Scénario 1: Utilisateur reçoit un message pendant qu'il lit la conversation

```
1. Expéditeur envoie message:send
   ↓
2. Serveur sauvegarde le message en BD
   ↓
3. Serveur émet message:new à la room (les deux utilisateurs la reçoivent)
   ↓
4. Serveur émet conversation:update aux deux utilisateurs
   ↓
5. Client affiche le message dans la conversation ouverte
```

### Scénario 2: Utilisateur reçoit un message en étant hors de la conversation

```
1. Expéditeur envoie message:send
   ↓
2. Serveur sauvegarde le message en BD
   ↓
3. Serveur émet message:notification uniquement au destinataire
   ↓
4. Serveur émet conversation:update aux deux utilisateurs
   ↓
5. Client affiche une notification badge/toast
   ↓
6. Client met à jour la liste des conversations
```

### Scénario 3: Utilisateur synchronise les messages non-lus au démarrage

```
1. App appelle GET /api/messages/unread/count
   ↓
2. Serveur retourne le nombre total et par conversation
   ↓
3. App affiche les badges sur chaque conversation
   ↓
4. (Optionnel) App appelle GET /api/messages/unread/:userId pour une conversation
   ↓
5. App affiche les messages non-lus
```

---

## 🎯 Implémentation Frontend

### 1. Écouter les notifications

```javascript
// Écouter les notifications de nouveaux messages
socket.on("message:notification", (notification) => {
  showNotificationBadge(notification.fromUser.username);
  playNotificationSound();
});

// Écouter les nouveaux messages dans la conversation ouverte
socket.on("message:new", (message) => {
  addMessageToConversation(message);
  markAsRead();
});

// Écouter les mises à jour de la liste des conversations
socket.on("conversation:update", (update) => {
  updateConversationList(update);
});
```

### 2. Marquer comme lu

```javascript
// Quand on ouvre une conversation
async function openConversation(userId) {
  // Marquer les messages comme lus
  await fetch(`/api/messages/seen/${userId}`, {
    method: "PATCH",
    headers: { "Authorization": `Bearer ${token}` }
  });
  
  // Rejoindre la room Socket
  socket.emit("join:conversation", { partnerId: userId });
}
```

### 3. Synchroniser au démarrage

```javascript
// Au démarrage de l'app
async function initNotifications() {
  const response = await fetch("/api/messages/unread/count", {
    headers: { "Authorization": `Bearer ${token}` }
  });
  
  const { total, byConversation } = await response.json();
  
  // Afficher les badges
  byConversation.forEach(conv => {
    updateConversationBadge(conv.userId, conv.count);
  });
}
```

---

## 🔒 Sécurité

- ✅ Tous les endpoints API nécessitent JWT
- ✅ Socket.IO vérifie le JWT avant de connecter
- ✅ Les utilisateurs ne reçoivent que leurs propres messages
- ✅ Rate limiting sur les messages (30 msg/min)
- ✅ Les notifications ne sont émises que si nécessaire

---

## 📈 Performance

- ✅ Sauvegarde asynchrone des messages en BD
- ✅ Notifications émises uniquement aux utilisateurs concernés
- ✅ Index sur `receiverId` et `isSeen` pour les requêtes rapides
- ✅ Pagination sur le frontend pour les listes longues

---

## ❓ FAQ

**Q: Comment obtenir le nombre de messages non-lus?**
A: Appeler `GET /api/messages/unread/count` au démarrage et écouter `message:notification` pour les mises à jour en temps réel.

**Q: Les notifications persistent-elles si l'utilisateur est hors ligne?**
A: Oui, les messages sont sauvegardés en BD. Au prochain login, l'app peut les récupérer via `GET /api/messages/unread/count`.

**Q: Comment désactiver les notifications?**
A: Supprimer l'écouteur `socket.off("message:notification")` ou implémenter une préférence utilisateur.

**Q: Peut-on gérer les notifications locales du navigateur?**
A: Oui, utiliser l'API `Notification` du navigateur avec l'événement `message:notification`.
