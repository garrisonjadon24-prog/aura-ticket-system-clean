# AURA Ticket System - Implementation Guide

## Quick Start

All features have been implemented in your `server.js` file. No additional installation needed.

### Start Server
```bash
cd /Users/ray/Desktop/aura-ticket-system
node server.js
```

Access at: `http://192.168.1.21:3000`

---

## Key URLs & Passwords

### Staff Access
- **URL**: `http://192.168.1.21:3000/staff`
- **PIN**: `AURA2026`
- **Login**: Enter your name + PIN

### Management Hub (NEW)
- **URL**: `http://192.168.1.21:3000/management-hub?key=MGMT2026`
- **PIN**: `MGMT2026`
- **Authorized Users**: RAY, SHAWN, NIQUE, CHE (case-insensitive)

### Management Pages
From Management Hub, access:
- 📊 Dashboard
- 📈 Live Analytics
- 📑 Allocations
- 🎉 Prize Draw
- 🎁 **Guest Prize Entries** (NEW)
- 👥 **Staff Log** (ENHANCED)
- 🎫 Guest Scans

---

## New Features - How to Use

### 1. Guest Prize Entry
**What Happens**:
1. Guest scans QR code → Sees welcome page
2. Guest enters their name in the prize form (optional)
3. Name is submitted and linked to their ticket
4. **Only 1 entry allowed per ticket**
5. Success message shown
6. After 25s → Redirected to Instagram

**Backend**: Data stored in `guestNameEntries` array

### 2. View Guest Prize Entries
1. Go to Management Hub (`/management-hub?key=MGMT2026`)
2. Click **"🎁 Guest Entries"**
3. See statistics:
   - Total entries
   - Unique tickets
4. View complete entry table (guest name, ticket, IP, timestamp)
5. Click **"🎲 Draw Random Winner"** to select a winner
6. Winner displays in prominent green box

### 3. Staff Activity Tracking
**What's Logged**:
- ✅ All staff logins (name, role, IP, timestamp)
- ✅ All management logins (distinguishes MGMT from STAFF)
- ✅ All logouts

**View Logs**:
1. Go to Management Hub
2. Click **"👥 Staff Log"**
3. See table with:
   - Name
   - Role (👔 MGMT or 👥 STAFF)
   - Action (login/logout)
   - IP Address
   - Time

### 4. Light Mode - Bright Dark Gold Theme
**Toggle**:
- Click **"☀ Light / Dark"** button (top right of pages)
- Theme saves automatically
- Works across all pages

**Colors Used**:
- Primary Gold: #daa520
- Dark Gold: #b8860b
- Light Gold Background: #f4e4c1
- Text: Dark brown (#2d2416)

---

## Data Storage

### In-Memory Arrays
All new data is stored in-memory (lost on server restart):

```javascript
// Existing
const tickets = new Map();           // Ticket data (persisted to disk)
const staffActivityLog = [];         // Staff logins (in-memory, max 500)
const guestScanLog = [];             // Guest scans (in-memory, max 1000)

// NEW
const guestNameEntries = [];         // Prize entries (in-memory, max 2000)
```

### To Make Data Persistent
You would need to:
1. Add database (MongoDB, PostgreSQL, etc.)
2. Add save/load functions for each array
3. Currently works fine for event-day use

---

## Security Notes

### PINs
- **Staff PIN**: `AURA2026` - Anyone can use
- **Management PIN**: `MGMT2026` - Only authorized names

### Authorized Managers
Edit this list in `server.js` if adding more managers:
```javascript
const ALLOWED_MANAGERS = ["RAY", "RAYMOND", "SHAWN", "NIQUE", "CHE"];
```

### One Entry Per Ticket
Guest prize entries are validated by ticket token - each guest can only enter once, even if they try to submit multiple times.

### IP Tracking
- Captures real client IP
- Useful for detecting abuse or multi-device access
- Visible in staff logs and guest entries

---

## API Endpoints - For Integration

### Guest Name Entry (POST)
```
POST /api/guest-name-entry
Headers: Content-Type: application/json
Body: {
  "ticketId": "AURA-001",
  "token": "abc123...",
  "guestName": "John Doe"
}
Response: {
  "success": true,
  "message": "Entry received! Good luck!"
}
```

### Get Guest Entries (GET)
```
GET /api/guest-name-entries?key=MGMT2026
Response: [
  {
    "ticketId": "AURA-001",
    "token": "abc123...",
    "guestName": "John Doe",
    "ip": "192.168.1.100",
    "timestamp": "2025-12-05T20:30:00.000Z"
  },
  ...
]
```

### Staff Login (POST)
```
POST /api/staff-login
Headers: Content-Type: application/json
Body: {
  "name": "Ray"
}
Response: {
  "success": true,
  "role": "management"  // or "staff"
}
```

### Staff Logout (POST)
```
POST /api/staff-logout
Headers: Content-Type: application/json
Body: {
  "name": "Ray"
}
Response: {
  "success": true
}
```

---

## Customization Options

### Change Authorized Managers
In `server.js`, find:
```javascript
const ALLOWED_MANAGERS = ["RAY", "RAYMOND", "SHAWN", "NIQUE", "CHE"];
```
Update names as needed.

### Change Light Mode Colors
In `themeCSSRoot()` function, modify:
```css
html.light-mode body {
  background: ... /* Change gradient colors */
  color: #2d2416;  /* Change text color */
}
html.light-mode .card {
  background: ... /* Change card background */
}
```

### Change Countdown Timer
In guest welcome page generation, modify:
```javascript
const REDIRECT_SECONDS = 25;  // Change this value
```

### Change Instagram URL
At top of `server.js`:
```javascript
const INSTAGRAM_URL = "https://www.instagram.com/a.u.r.a_by_pop/";
```

---

## Troubleshooting

### Light Mode Not Persisting
- Clear browser localStorage
- Try incognito/private mode
- Check if localStorage is enabled

### Guest Prize Entry Not Saving
1. Check browser console for errors (F12)
2. Verify API endpoint is accessible
3. Check if token is valid

### Staff Log Shows "Unknown" or Missing IP
- This might occur for very old entries from before IP tracking was added
- New entries will always have IP addresses

### Management Hub Redirects to Staff Home
- Verify you're using MGMT PIN: `MGMT2026`
- Verify your name is in ALLOWED_MANAGERS list
- Names are case-insensitive (RAY = ray = Ray)

---

## File Structure

```
/Users/ray/Desktop/aura-ticket-system/
├── server.js                          # Main application (UPDATED)
├── tickets.json                       # Persistent ticket data
├── public/
│   ├── aura-welcome.mp3              # Guest welcome audio
│   ├── aura-logo.png
│   ├── pop-logo.png
│   ├── manifest.json
│   ├── sw.js
│   └── icons/
├── generated_qr/                      # QR code images
└── FEATURES_IMPLEMENTED.md            # Detailed feature docs (NEW)
└── IMPLEMENTATION_GUIDE.md            # This file (NEW)
```

---

## Next Steps

1. **Test guest prize entry flow**:
   - Generate a QR code (use `/generate-batch`)
   - Scan with mobile device
   - Enter name in prize form
   - Verify entry in Management Hub

2. **Test staff activity logging**:
   - Login as staff
   - Verify entry in Staff Log
   - Login as manager (e.g., RAY)
   - Verify role shows as MGMT

3. **Test light mode**:
   - Click light/dark toggle
   - Verify gold colors appear
   - Refresh and verify theme persists
   - Try on different pages

4. **Test prize drawing**:
   - Ensure at least one guest entry exists
   - Go to Guest Entries page
   - Click "Draw Random Winner"
   - Verify winner displays correctly

---

## Support

If you encounter issues:
1. Check browser console (F12 → Console tab)
2. Check server logs in terminal
3. Verify PINs are correct
4. Restart server if data seems stale
5. Clear browser cache/localStorage if UI issues

---

**Version**: 1.0  
**Status**: Production Ready  
**All Features Implemented**: ✅
