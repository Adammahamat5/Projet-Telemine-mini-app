// Backend complet Telemine avec Node.js (Express) et Firebase pour l'authentification, la base de données, et la logique de minage.

// 📦 Dépendances à installer // npm install express cors firebase-admin nodemailer uuid dotenv body-parser

const express = require('express'); const cors = require('cors'); const bodyParser = require('body-parser'); const nodemailer = require('nodemailer'); const { v4: uuidv4 } = require('uuid'); const admin = require('firebase-admin'); require('dotenv').config();

// 🔐 Initialisation Firebase Admin SDK const serviceAccount = require('./serviceAccountKey.json'); // à obtenir depuis Firebase Console admin.initializeApp({ credential: admin.credential.cert(serviceAccount) }); const db = admin.firestore();

// 🚀 Initialisation serveur const app = express(); app.use(cors()); app.use(bodyParser.json()); const PORT = process.env.PORT || 3001;

// ✉️ Transporteur email (Gmail conseillé) const transporter = nodemailer.createTransport({ service: 'gmail', auth: { user: process.env.EMAIL_FROM, pass: process.env.EMAIL_PASS } });

// 📩 Génération et envoi de code app.post('/send-code', async (req, res) => { const { email } = req.body; const code = Math.floor(100000 + Math.random() * 900000).toString(); await db.collection('verifications').doc(email).set({ code, verified: false });

await transporter.sendMail({ from: process.env.EMAIL_FROM, to: email, subject: 'Telemine Verification Code', text: Your verification code is: ${code} });

res.status(200).json({ success: true }); });

// ✅ Vérification du code app.post('/verify-code', async (req, res) => { const { email, code } = req.body; const doc = await db.collection('verifications').doc(email).get(); if (!doc.exists || doc.data().code !== code) return res.status(400).json({ success: false }); await db.collection('verifications').doc(email).update({ verified: true }); await db.collection('users').doc(email).set({ email, createdAt: Date.now(), balance: 0, mining: false, lastClaim: 0, referrals: 0, kyc: false }); res.status(200).json({ success: true }); });

// ⛏️ Lancer le minage (après avoir vu les pubs) app.post('/start-mining', async (req, res) => { const { email } = req.body; const userRef = db.collection('users').doc(email); const user = await userRef.get(); if (!user.exists) return res.status(404).json({ success: false }); await userRef.update({ mining: true, lastClaim: Date.now() }); res.status(200).json({ success: true }); });

// ⏱️ Mise à jour automatique du solde app.post('/update-mining', async (req, res) => { const { email } = req.body; const userRef = db.collection('users').doc(email); const user = await userRef.get(); if (!user.exists || !user.data().mining) return res.status(400).json({ success: false }); const now = Date.now(); const lastClaim = user.data().lastClaim || now; const secondsPassed = Math.floor((now - lastClaim) / 1000); const tlmEarned = Math.min(secondsPassed * 0.000023, 2); // max 2 $TLM/day await userRef.update({ balance: user.data().balance + tlmEarned, lastClaim: now }); res.status(200).json({ success: true, tlm: user.data().balance + tlmEarned }); });

// 👨‍💼 Tableau de bord admin (exemple simplifié) app.get('/admin/dashboard', async (req, res) => { const snapshot = await db.collection('users').get(); const users = snapshot.docs.map(doc => doc.data()); const totalUsers = users.length; const totalMined = users.reduce((acc, u) => acc + (u.balance || 0), 0); res.status(200).json({ totalUsers, totalMined }); });

// 🪂 Lancer un airdrop (admin only) app.post('/admin/airdrop', async (req, res) => { const { amount, message } = req.body; const snapshot = await db.collection('users').get(); for (const doc of snapshot.docs) { await doc.ref.update({ balance: (doc.data().balance || 0) + amount }); } res.status(200).json({ success: true, message }); });

// ▶️ Lancement serveur app.listen(PORT, () => console.log(Telemine backend running on port ${PORT}));

