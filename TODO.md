# TODO — Audit & corrections ESTM Chat

- [x] CRITIQUE 1: Migrer JWT vers cookie HttpOnly (server/routes/auth.js, server/middleware/auth.js, Socket.IO auth, public/script.js)
- [x] CRITIQUE 2: Centraliser non-lus dans server/routes/message.js et supprimer/neutraliser versions duplicées côté server/routes/user.js; mettre à jour public/script.js
- [ ] IMPORTANT 3: Remplacer setInterval scheduler par jobs persistés (BullMQ ou Agenda.js)
- [x] IMPORTANT 4: Pagination dynamique sur /api/messages/unread/:userId et ajuster frontend

- [x] IMPORTANT 5: Validation upload renforcée (magic bytes via file-type, tailles/dimensions)

- [x] IMPORTANT 6: Nettoyage onlineUsers robuste au disconnect multi-onglets
- [ ] Générer rapport PDF du projet (après corrections)


