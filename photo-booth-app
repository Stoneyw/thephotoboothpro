const express = require('express');
const multer = require('multer');
const path = require('path');
const fs = require('fs');
const nodemailer = require('nodemailer');
const qr = require('qrcode');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const cors = require('cors');
const twilio = require('twilio');
const cron = require('node-cron');
const app = express();
const PORT = process.env.PORT || 3000;
const SECRET_KEY = 'your_secret_key';
const twilioClient = new twilio(process.env.TWILIO_ACCOUNT_SID, process.env.TWILIO_AUTH_TOKEN);

app.use(cors());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(express.static(path.join(__dirname, 'public')));
app.use('/uploads', express.static(path.join(__dirname, 'uploads')));

const users = {}; // Store users in-memory for simplicity
const events = {}; // Store event data

// Authentication Middleware
const authenticate = (req, res, next) => {
    const token = req.header('Authorization');
    if (!token) return res.status(401).json({ error: 'Access Denied' });
    try {
        const verified = jwt.verify(token, SECRET_KEY);
        req.user = verified;
        next();
    } catch (err) {
        res.status(400).json({ error: 'Invalid Token' });
    }
};

// User Registration
app.post('/register', async (req, res) => {
    const { username, password } = req.body;
    if (users[username]) return res.status(400).json({ error: 'User already exists' });
    const hashedPassword = await bcrypt.hash(password, 10);
    users[username] = hashedPassword;
    res.json({ message: 'User registered successfully' });
});

// User Login
app.post('/login', async (req, res) => {
    const { username, password } = req.body;
    const hashedPassword = users[username];
    if (!hashedPassword || !(await bcrypt.compare(password, hashedPassword))) {
        return res.status(400).json({ error: 'Invalid credentials' });
    }
    const token = jwt.sign({ username }, SECRET_KEY, { expiresIn: '1h' });
    res.json({ token });
});

// Create an event
app.post('/create_event', authenticate, (req, res) => {
    const { eventId, eventName } = req.body;
    const eventUrl = `https://thephotoboothpro.onrender.com/gallery/${eventId}`;
    events[eventId] = { eventName, photos: [], sharedPhotos: [], createdAt: new Date(), qrCode: '' };
    
    qr.toDataURL(eventUrl, (err, qrCode) => {
        if (!err) {
            events[eventId].qrCode = qrCode;
        }
    });
    
    res.json({ message: 'Event created successfully', qrCode: events[eventId].qrCode });
});

// Invite guests via SMS
app.post('/invite_guest', authenticate, (req, res) => {
    const { phoneNumber, eventId } = req.body;
    const eventUrl = `https://thephotoboothpro.onrender.com/gallery/${eventId}`;
    twilioClient.messages.create({
        body: `You're invited to view and share photos for ${events[eventId].eventName}. Click here: ${eventUrl}`,
        from: process.env.YOUR_TWILIO_PHONE_NUMBER,
        to: phoneNumber
    }).then(() => res.json({ message: 'Invitation sent successfully' }))
    .catch(error => res.status(500).json({ error: 'Error sending SMS' }));
});

// View event album with QR code
app.get('/gallery/:eventId', authenticate, (req, res) => {
    const eventId = req.params.eventId;
    if (!events[eventId]) return res.status(404).json({ error: 'Event not found' });
    res.json({ eventName: events[eventId].eventName, photos: events[eventId].photos, sharedPhotos: events[eventId].sharedPhotos, qrCode: events[eventId].qrCode, notice: 'Event photos will be deleted after 15 days.' });
});

// Upload photo to event album
const storage = multer.diskStorage({
    destination: (req, file, cb) => {
        const eventId = req.params.eventId;
        const dir = path.join(__dirname, 'uploads', eventId);
        if (!fs.existsSync(dir)) {
            fs.mkdirSync(dir, { recursive: true });
        }
        cb(null, dir);
    },
    filename: (req, file, cb) => {
        cb(null, file.fieldname + '-' + Date.now() + path.extname(file.originalname));
    }
});
const upload = multer({ storage: storage });

app.post('/upload/:eventId', authenticate, upload.single('photo'), (req, res) => {
    const eventId = req.params.eventId;
    if (!events[eventId]) return res.status(404).json({ error: 'Event not found' });
    const filePath = `/uploads/${eventId}/${req.file.filename}`;
    events[eventId].photos.push(filePath);
    res.json({ message: 'Photo uploaded successfully', filePath });
});

// Auto-delete events after 15 days
cron.schedule('0 0 * * *', () => {
    const now = new Date();
    Object.keys(events).forEach(eventId => {
        if ((now - events[eventId].createdAt) > 15 * 24 * 60 * 60 * 1000) {
            delete events[eventId];
            fs.rmdirSync(path.join(__dirname, 'uploads', eventId), { recursive: true });
        }
    });
    console.log('Old events deleted');
});

app.listen(PORT, () => {
    console.log(`Server running on https://thephotoboothpro.onrender.com`);
});


app.listen(PORT, () => {
    console.log(`Server running on https://thephotoboothpro.onrender.com`);
});
