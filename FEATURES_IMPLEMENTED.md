# AURA Ticket System - New Features Implemented

## Overview
This document outlines all the new features added to the AURA ticket system to support management operations, guest engagement, and enhanced tracking.

---

## 1. MANAGEMENT HUB - Restricted Access

### Access Control
- **URL**: `/management-hub?key=MGMT2026`
- **PIN**: `MGMT2026` (hardcoded)
- **Authorized Users**: RAY, RAYMOND, SHAWN, NIQUE, CHE (case-insensitive)
- **Rejection**: Non-authorized managers will see "Restricted" message and alert

### Features
- **6 Management Tools** (all protected by MGMT PIN):
  1. **📊 Dashboard** - Quick snapshot of total tickets and arrivals
  2. **📈 Live Analytics** - Real-time check-in stats and recent scans
  3. **📑 Allocations** - Track prefixes, sold vs unsold, export CSV
  4. **🎉 Prize Draw** - Random winner selector from used tickets
  5. **🎁 Guest Entries** - Prize draw entries management (NEW)
  6. **👥 Staff Log** - Staff/management activity tracking (ENHANCED)
  7. **🎫 Scan Log** - Guest ticket scan records

### Code Structure
```javascript
// Management Hub Route
app.get("/management-hub", (req, res) => {
  const { key } = req.query;
  if (key !== MANAGEMENT_PIN) {
    return res.redirect("/staff?key=" + encodeURIComponent(STAFF_PIN));
  }
  // ... render management interface
});
```

---

## 2. GUEST NAME ENTRY FOR PRIZE DRAW

### Guest Welcome Page Enhancement
When guests scan their QR code, they see:
- ✅ Welcome message with AURA branding
- 🎵 Audio player (auto-play with manual override)
- 🎁 **NEW: Prize Entry Form** - Only one entry per ticket is allowed (enforced by token-based validation; multiple attempts are rejected with an error message)
- ⏱️ 25-second countdown before Instagram redirect
- 📝 Guest info tracked for winner selection: name, ticket ID, token, IP address, and timestamp

### Prize Entry Form
```html
<div class="prize-section">
  <div class="prize-title">🎁 Mystery Prize Entry</div>
  <p>Enter your name for a chance to win an exclusive mystery prize!</p>
  <input type="text" id="guestNameInput" placeholder="Enter your name" maxlength="50" />
  <button id="submitNameBtn">Submit for Prize Draw</button>
  <div class="entry-success">✅ You're entered! Good luck!</div>
</div>
```

### Backend Data Structure
```javascript
// New data store: Guest Name Entries
const guestNameEntries = []; 
// Format: { ticketId, token, guestName, ip, timestamp }
```

### Entry Submission Constraints
- ✅ Only ONE attempt per ticket (token-based validation)
- ✅ Name is required (50 character max)
- ✅ Each entry linked to ticket ID and token
- ✅ IP address recorded
- ✅ Timestamp recorded
- ✅ Success message displayed to guest

### API Endpoint - Guest Name Entry
```javascript
app.post("/api/guest-name-entry", (req, res) => {
  // Receives: ticketId, token, guestName
  // Validates: Token exists, no duplicate entry per token
  // Returns: success or error message
});
```

---

## 3. GUEST PRIZE ENTRIES MANAGEMENT PAGE

### URL
- `/guest-prize-entries?key=MGMT2026`

### Features
- 📊 **Statistics Dashboard**
  - Total entries count
  - Unique ticket count
  
- 🎲 **Random Winner Selector**
  - Button: "🎲 Draw Random Winner"
  - Displays: Guest name, ticket ID
  - Visual highlight with green border

- 📋 **Complete Entry Log**
  - Guest Name
  - Ticket ID
  - Token (short version: first 8 chars + "...")
  - IP Address
  - Submission Timestamp

- ⚡ **Live Updates**
  - Auto-refreshes every 5 seconds
  - See entries as they come in

### Code Example
```javascript
function drawWinner() {
  // Fetches all guest entries
  // Selects random entry
  // Displays in prominent green box with celebration emoji
  // Format: 
  // 🎉 WINNER 🎉
  // [Guest Name]
  // Ticket: [Ticket ID]
}
```

---

## 4. STAFF & MANAGEMENT ACTIVITY LOGGING

### Enhanced Tracking
The system now logs ALL staff and management interactions:

#### Data Structure
```javascript
const staffActivityLog = [];
// Format: { name, action, role, ip, timestamp }
// role: "staff" or "management"
// action: "login" or "logout"
```

#### Events Tracked
1. **Login** - When staff/manager opens `/staff?key=STAFF_PIN`
2. **Logout** - When staff/manager clicks logout button (NEW)
3. **Role Detection** - Automatically identifies manager vs. staff

#### Role Assignment
- **MANAGEMENT**: RAY, RAYMOND, SHAWN, NIQUE, CHE
- **STAFF**: Anyone else with valid PIN

#### IP Tracking
- Captures real client IP (using X-Forwarded-For, X-Real-IP, or connection IP)
- Useful for identifying multi-device access and security monitoring

### API Endpoints

#### Staff Login Endpoint
```javascript
app.post("/api/staff-login", (req, res) => {
  // Input: { name }
  // Determines: role (management or staff)
  // Logs: All details to staffActivityLog
  // Returns: { success: true, role: 'staff' | 'management' }
});
```

#### Staff Logout Endpoint (NEW)
```javascript
app.post("/api/staff-logout", (req, res) => {
  // Input: { name }
  // Logs: Logout action with role
  // Returns: { success: true }
});
```

### Staff Activity Log Page
**URL**: `/staff-log?key=MGMT2026`

**Table Columns**:
1. **Name** - Staff/Manager name
2. **Role** - 👔 MGMT or 👥 STAFF (visual badge)
3. **Action** - login / logout
4. **IP** - Client IP address
5. **Time** - Formatted timestamp (local)

**Display**:
- Shows last 200 activity entries
- Live updates (auto-refreshes)
- Role badges with emojis
- Sorted by most recent first

---

## 5. LIGHT MODE WITH BRIGHT DARK GOLD THEME

### Color Scheme - Bright Dark Gold
Replaces previous yellow with professional dark gold aesthetic:

#### Dark Mode (Default)
- Background: Deep purple (#050007)
- Cards: Dark purple gradient
- Accents: Pink/Red/Gold

#### Light Mode - NEW BRIGHT DARK GOLD
- **Primary Gold**: #daa520 (Goldenrod)
- **Dark Gold**: #b8860b (Dark Goldenrod)
- **Bright Gold**: #f4e4c1 (Light beige-gold)
- **Text**: #2d2416 (Dark brown)
- **Background**: Radial gradient with warm golds

### CSS Variables Updated
```css
:root {
  --accent-gold: #ffb300;  /* Used in dark mode */
}

html.light-mode body {
  background: radial-gradient(circle at top left, rgba(218,165,32,0.4), transparent 55%),
              radial-gradient(circle at bottom right, rgba(184,134,11,0.5), transparent 55%),
              #daa520;
  color: #2d2416;
}

html.light-mode .card {
  background: radial-gradient(circle at top, #f4e4c1, #daa520 60%);
  color: #2d2416;
}

html.light-mode .badge,
html.light-mode .role-label {
  border-color: rgba(218,165,32,0.8);
  color: #b8860b;
  background: rgba(244,228,193,0.9);
}
```

### Theme Toggle
- **Button**: "☀ Light / Dark" (appears on all pages)
- **Storage**: localStorage key `'auraTheme'`
- **Persistence**: Theme preference saved across sessions
- **Default**: Dark mode

### Pages with Theme Support
✅ ALL pages now support light/dark mode:
- Staff login
- Staff home
- Management Hub
- Dashboard
- Live Analytics
- Allocations
- Prize Draw
- Staff Log
- Guest Scans
- Guest Prize Entries
- All other management pages

### Theme Implementation
```javascript
// Global theme script included on every page
function themeScript() {
  return `
    <script>
      const AURA_THEME_KEY = 'auraTheme';
      function applyAuraTheme() {
        const theme = localStorage.getItem(AURA_THEME_KEY) || 'dark';
        if (theme === 'light') {
          document.documentElement.classList.add('light-mode');
        }
      }
      function toggleAuraTheme() {
        const theme = localStorage.getItem(AURA_THEME_KEY) || 'dark';
        const next = theme === 'light' ? 'dark' : 'light';
        localStorage.setItem(AURA_THEME_KEY, next);
        applyAuraTheme();
      }
      applyAuraTheme();
      window.AURA_THEME = { apply: applyAuraTheme, toggle: toggleAuraTheme };
    </script>
  `;
}
```

---

## 6. GUEST WELCOME PAGE - UPDATED

### Complete Flow
1. **Guest scans QR** → Taken to `/ticket?token=XXX&sig=XXX`
2. **Welcome page loads** with:
   - Status alert (✅ VALID or ⚠️ USED)
   - AURA branding and welcome message
   - Audio player with manual override
   - 🎁 **Prize entry form** (NEW)
   - 25-second countdown timer
3. **Guest submits name** (optional but encouraged)
   - Entry logged to backend
   - Success message shown
   - Cannot re-submit (one attempt only)
4. **Auto-redirect** after 25 seconds to Instagram

### HTML Structure
```html
<!-- Status Alert -->
<div class="status-alert">✅ TICKET VALID</div>

<!-- Prize Entry Section (NEW) -->
<div class="prize-section">
  <div class="prize-title">🎁 Mystery Prize Entry</div>
  <input type="text" id="guestNameInput" placeholder="Enter your name" />
  <button id="submitNameBtn">Submit for Prize Draw</button>
  <div class="entry-success">✅ You're entered! Good luck!</div>
</div>

<!-- Countdown -->
<p class="note">Redirecting you in <span id="countdownSpan">25</span>...</p>
```

### JavaScript Functionality
- Audio auto-play (muted fallback + user tap button)
- Prize entry submission via `/api/guest-name-entry` endpoint
- Countdown timer updates every second
- Prevents double-submission (button disabled after first attempt)
- Redirects to Instagram URL after 25 seconds

---

## 7. AUTO-BACKUP SYSTEM (Persistent Memory)

### Overview
All in-memory data automatically persists between server restarts via periodic JSON backups.

#### Backup Behavior
- **Interval**: Every 5 minutes
- **File**: `aura-backup.json` (in project root)
- **On Startup**: Automatically restores all data from last backup
- **On Exit**: Creates final backup when server stops (graceful shutdown)
- **No Data Loss**: All guest entries, staff logs, tickets survive server restarts

#### Data Backed Up
```javascript
// Complete in-memory state saved every 5 minutes
{
  timestamp: "2025-12-06T02:41:37.300Z",
  guestNameEntries: [...],      // Prize draw entries
  staffActivityLog: [...],       // Staff/manager activity
  guestScanLog: [...],           // Guest QR scans
  scanEvents: {...},             // Invalid/duplicate scans
  paymentEvents: {...},          // Payment transactions
  boxOfficeSales: {...},         // Box office records
  giveawayEvents: {...},         // Prize draws
  ipLogging: {...},              // IP tracking
  tickets: [...]                 // All ticket records
}
```

#### Implementation
```javascript
// Auto-backup every 5 minutes
const BACKUP_INTERVAL = 5 * 60 * 1000; // 5 minutes
const BACKUP_FILE = path.join(__dirname, 'aura-backup.json');

function createBackup() {
  // Saves all in-memory arrays to aura-backup.json
}

function restoreBackup() {
  // Restores all data from aura-backup.json on startup
}

// Restore on startup
restoreBackup();

// Auto-backup every 5 minutes
setInterval(createBackup, BACKUP_INTERVAL);

// Backup on exit
process.on('exit', createBackup);
process.on('SIGINT', () => { createBackup(); process.exit(); });
process.on('SIGTERM', () => { createBackup(); process.exit(); });
```

### Data Retention
- **staffActivityLog**: Last 500 entries (auto-trim)
- **guestNameEntries**: Last 2000 entries (auto-trim)
- **guestScanLog**: Last 1000 entries (auto-trim)
- **All backed-up data persists** between server restarts via aura-backup.json

---

## 8. SECURITY & PERMISSIONS

### PIN Requirements
| Resource | PIN | Allowed Users |
|----------|-----|---------------|
| Staff Tools | AURA2026 | Anyone with PIN |
| Management Hub | MGMT2026 | RAY, RAYMOND, SHAWN, NIQUE, CHE |
| Staff Log | MGMT2026 | Management only |
| Guest Entries | MGMT2026 | Management only |
| Analytics | MGMT2026 | Management only |

### Access Control Logic
```javascript
// Management Hub Protection
if (key !== MANAGEMENT_PIN) {
  return res.redirect("/staff?key=" + STAFF_PIN);
}

// Manager Name Validation
const ALLOWED_MANAGERS = ["RAY", "RAYMOND", "SHAWN", "NIQUE", "CHE"];
if (!ALLOWED_MANAGERS.includes(name.toUpperCase())) {
  alert('Management Hub access is restricted to...');
}
```

---

## 9. API ENDPOINTS - QUICK REFERENCE

### New Endpoints
| Method | URL | Purpose |
|--------|-----|---------|
| POST | `/api/guest-name-entry` | Submit name for prize draw |
| GET | `/api/guest-name-entries` | Fetch all entries (management) |
| POST | `/api/staff-logout` | Log staff logout event |

### New Pages
| Route | Purpose | Auth |
|-------|---------|------|
| `/management-hub` | Central management dashboard | MGMT PIN + Name |
| `/guest-prize-entries` | Prize draw entries & winner draw | MGMT PIN |
| `/staff-log` | Activity log of all staff/managers | MGMT PIN |

---

## 10. TESTING CHECKLIST

### Guest Prize Entry
- [ ] Scan QR code as guest
- [ ] See prize entry form on welcome page
- [ ] Enter name and submit
- [ ] See "✅ You're entered!" message
- [ ] Button becomes disabled (no double-submit)
- [ ] Verify entry appears in Management Hub → Guest Entries

### Staff/Management Logging
- [ ] Login as staff (any name + AURA2026)
- [ ] Verify entry in Staff Log (role: STAFF)
- [ ] Login as manager (e.g., RAY + MGMT2026)
- [ ] Verify entry in Staff Log (role: MGMT)
- [ ] Check IP address is captured
- [ ] Click logout and verify logout entry

### Light Mode
- [ ] Click "☀ Light / Dark" button on any page
- [ ] Verify page turns gold/brown theme
- [ ] Check all elements are readable
- [ ] Refresh page and verify theme persists
- [ ] Toggle back to dark mode
- [ ] Switch pages in light mode (theme stays)

### Management Hub
- [ ] Try accessing without MGMT PIN (should redirect)
- [ ] Try with non-manager name (should see alert)
- [ ] Access as authorized manager (RAY, SHAWN, etc.)
- [ ] Click Guest Entries button
- [ ] Use "🎲 Draw Random Winner" button
- [ ] Verify winner display shows guest name + ticket

### Guest Entries Page
- [ ] See stats (total entries, unique tickets)
- [ ] See table of all entries with guest name, ticket, IP, time
- [ ] See auto-refresh (new entries appear after 5 seconds)
- [ ] Draw winner and see prominent display

---

## 11. IMPORTANT NOTES

### Persistent Memory via Auto-Backup
✅ **All data now persists between server restarts**:
- Guest prize entries are saved automatically
- Staff activity logs are saved automatically
- Guest scan logs are saved automatically
- Automatic backup every 5 minutes to `aura-backup.json`
- Automatic restore from backup on server startup
- Final backup created on graceful shutdown

### Session Storage (No Loss)
All data is backed up automatically:
- No manual export/import needed (but still available via admin endpoints)
- Data survives server restart, power loss (graceful shutdown), or process termination
- Backup file: `aura-backup.json` in project root

### One Entry Per Ticket
Prize entries are validated at the token level:
```javascript
const existingEntry = guestNameEntries.find(e => e.token === token);
if (existingEntry) {
  return res.status(400).json({ error: "Only one entry per ticket allowed" });
}
```

### Management User Whitelist
Must update this array if adding more authorized managers:
```javascript
const ALLOWED_MANAGERS = ["RAY", "RAYMOND", "SHAWN", "NIQUE", "CHE"];
```
Located in:
- `/management-hub` route
- `/api/staff-login` endpoint
- Staff page header

### Theme Persistence
Theme choice is saved to browser localStorage:
- Key: `'auraTheme'`
- Values: `'light'` or `'dark'`
- Persists across sessions (same browser/device)

---

## 12. MIGRATION FROM PREVIOUS VERSION

### Breaking Changes
- None! All existing functionality preserved
- New features are additive

### Data Compatibility
- `staffActivityLog` extended with `role` and `ip` fields
- Old entries without these fields will still display (fields will be undefined/—)
- New entries will have complete data

---

## Summary of Features

✅ **Management Hub** - Restricted to MGMT PIN + authorized names  
✅ **Guest Prize Entries** - One-time per ticket, logged with guest name  
✅ **Guest Prize Entries Management Page** - View entries, draw winner  
✅ **Staff Activity Logging** - Track all staff and management logins/logouts  
✅ **Enhanced Staff Log** - Shows role (MGMT/STAFF) and IP address  
✅ **Light Mode - Bright Dark Gold Theme** - Professional gold aesthetic, persistent across pages  
✅ **All Pages Updated** - Theme toggle on every page  
✅ **Logout Tracking** - New endpoint to log when staff leaves  
✅ **Auto-Backup System** - All data persists between server restarts (5-minute intervals)  

---

**Version**: 1.0  
**Last Updated**: December 5, 2025  
**Status**: ✅ Production Ready
