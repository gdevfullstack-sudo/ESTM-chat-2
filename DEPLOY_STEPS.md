# 🚀 Guide Rapide Déploiement - Actions Prioritaires

## ⏱️ Temps Estimé: 45 minutes

---

## 🔴 ÉTAPE 1: Sécuriser les Secrets (10 min)

### 1.1 Générer nouveaux secrets

```powershell
# Générer JWT_SECRET
$newJWT = [System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes(32)
$hexJWT = -join ($newJWT | ForEach-Object { $_.ToString("X2") })
Write-Host "JWT_SECRET=$hexJWT"

# Générer SESSION_SECRET
$newSession = [System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes(32)
$hexSession = -join ($newSession | ForEach-Object { $_.ToString("X2") })
Write-Host "SESSION_SECRET=$hexSession"
```

### 1.2 Mettre à jour .env localement

```env
MONGO_URI=mongodb://localhost:27017/estim-chat
JWT_SECRET=<nouveau_secret_generé>
SESSION_SECRET=<nouveau_secret_generé>
CLIENT_URL=http://localhost:5000
MAX_FILE_SIZE_MB=20
```

### 1.3 Retirer .env du Git

```bash
git rm --cached .env
echo ".env" >> .gitignore
git commit -m "Remove .env from version control"
git push
```

✅ **Vérifier**: `.env` not in `.gitignore`?

---

## 🟦 ÉTAPE 2: Créer MongoDB Atlas (15 min)

### 2.1 Créer compte Atlas
1. Aller sur https://www.mongodb.com/cloud/atlas
2. Sign up (gratuit)
3. Créer une organisation

### 2.2 Créer cluster
1. Create Deployment
2. Choisir **M0 (Free Forever)**
3. Provider: **AWS** ou **Azure**
4. Region: **eu-west-1** (Irlande, proche Europe)
5. Cluster Name: `estim-chat-prod`

### 2.3 Configurer Security
1. Database Access → Add New Database User
   - Username: `estim_user`
   - Password: (générer forte)
   - Role: `Atlas admin`
2. Network Access → Add IP Address
   - Allow access from: `0.0.0.0/0` (provisoire)
   - Ajouter IP Render après déploiement

### 2.4 Obtenir URI
1. Clusters → Connect
2. Drivers → Node.js → Copy connection string
3. Exemple: `mongodb+srv://estim_user:password@estim-chat-prod.xxxxx.mongodb.net/estim-chat?retryWrites=true&w=majority`

✅ **Copier URI complète**

---

## 🟪 ÉTAPE 3: Configurer Render (15 min)

### 3.1 Créer nouveau Web Service
1. https://dashboard.render.com
2. New → Web Service
3. Build & deploy from Git
4. Sélectionner votre repo
5. Settings:
   - Name: `estim-chat-api`
   - Runtime: `Node`
   - Build Command: `npm install`
   - Start Command: `npm start`
   - Plan: `Free` (pour MVP)

### 3.2 Ajouter Environment Variables
Dans **Environment**:

```
MONGO_URI=mongodb+srv://estim_user:password@estim-chat-prod.xxxxx.mongodb.net/estim-chat?retryWrites=true&w=majority
JWT_SECRET=<nouveau_secret_generé_ci-dessus>
SESSION_SECRET=<nouveau_secret_generé_ci-dessus>
CLIENT_URL=https://<your-frontend-domain>.com
NODE_ENV=production
```

### 3.3 Déployer

1. Appuyer sur **Create Web Service**
2. Render build et déploie automatiquement
3. Attendre le message vert **Live**
4. Copier l'URL: `https://estim-chat-api.onrender.com`

✅ **Note URL publique**

---

## 🟢 ÉTAPE 4: Valider Déploiement (5 min)

### 4.1 Tester Health Check

```powershell
$ProgressPreference = 'SilentlyContinue'
(Invoke-WebRequest -Uri "https://estim-chat-api.onrender.com/api/health").Content
```

**Réponse attendue:**
```json
{"ok":true,"app":"ESTIM Chat"}
```

### 4.2 Tester Inscription

```powershell
$body = @{
    phoneNumber = "+2420648304311"
    password = "SecurePass123@"
    confirmPassword = "SecurePass123@"
    username = "TestUser"
} | ConvertTo-Json


(Invoke-WebRequest -Uri "https://estim-chat-api.onrender.com/api/auth/register" `
    -Method Post `
    -Headers @{"Content-Type"="application/json"} `
    -Body $body).Content
```

### 4.3 Tester Login

```powershell
$body = @{
    phoneNumber = "+2420648304311"
    password = "SecurePass123@"
} | ConvertTo-Json


(Invoke-WebRequest -Uri "https://estim-chat-api.onrender.com/api/auth/login" `
    -Method Post `
    -Headers @{"Content-Type"="application/json"} `
    -Body $body).Content
```

✅ **Si répond avec token**: Déploiement réussi!

---

## 🔐 ÉTAPE 5: Sécurité Post-Déploiement (5 min)

### 5.1 Configurer IP Whitelist MongoDB
1. Atlas → Network Access
2. Remplacer `0.0.0.0/0` par IP Render:
   - Aller sur Render dashboard
   - Settings → Custom Domain
   - Voir "Outbound IP" ou demander support

### 5.2 Forcer HTTPS
Dans `.env` production:
```env
NODE_ENV=production
```

Cela force Render à utiliser HTTPS

### 5.3 Valider Certificat SSL
```powershell
# Vérifier HTTPS fonctionne
(Invoke-WebRequest -Uri "https://estim-chat-api.onrender.com/api/health" -UseBasicParsing).StatusCode
```

---

## 🧪 ÉTAPE 6: Tests Complets (5 min)

Suivre [TESTING.md](TESTING.md) mais avec URL de production:
- Remplacer `http://localhost:5000` par `https://estim-chat-api.onrender.com`

---

## ✅ Checklist Final

- [ ] Secrets générés et NOT dans Git
- [ ] .env ajouté à .gitignore
- [ ] MongoDB Atlas cluster créé
- [ ] URI MongoDB copiée
- [ ] Render Web Service créé
- [ ] Environment variables configurées
- [ ] Déploiement réussi (badge vert)
- [ ] /api/health répond
- [ ] Inscription fonctionne
- [ ] Login fonctionne
- [ ] HTTPS actif

---

## 🚨 Troubleshooting

### Erreur: `JWT_SECRET undefined`
→ Ajouter `JWT_SECRET` dans Render Environment

### Erreur: `MongoServerSelectionError`
→ Vérifier `MONGO_URI` correcte et IP whitelist OK

### Erreur: CORS
→ Vérifier `CLIENT_URL` correspond au frontend domain

### Render freezes après inactivité
→ Normal avec plan Free (redémarrage 15min)
→ Utiliser `render-deploy` webhook pour restart

---

## 📞 Support Render

- Docs: https://render.com/docs
- Status: https://status.render.com
- Email: support@render.com

---

## 🎉 Résultat Final

Après ces étapes:
- ✅ Backend sécurisé
- ✅ BD MongoDB en ligne
- ✅ API accessible publiquement
- ✅ Prêt pour frontend

**URL API**: `https://estim-chat-api.onrender.com`
**URL WebSocket**: `wss://estim-chat-api.onrender.com` (Socket.IO)

---

**Durée totale**: 45 minutes
**Complexité**: Facile
**Coût**: Gratuit (Render Free + MongoDB Free Tier)
