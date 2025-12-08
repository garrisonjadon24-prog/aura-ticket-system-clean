# AURA Ticket System - Quick Reference & UI Map

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    AURA TICKET SYSTEM                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────┐    ┌──────────────────────────────┐ │
│  │   GUESTS         │    │   STAFF / MANAGEMENT         │ │
│  ├──────────────────┤    ├──────────────────────────────┤ │
│  │ Scan QR Code     │    │ /staff?key=AURA2026          │ │
│  │     ↓            │    │   ↓                          │ │
│  │ Welcome Page     │    │ Staff Home (Check-In)        │ │
│  │  • Audio 🎵      │    │   ↓                          │ │
│  │  • Prize Entry✨ │    │ /management-hub?key=MGMT2026 │ │
│  │  • 25s Timer ⏱️  │    │   ↓                          │ │
│  │  • IG Link       │    │ Management Hub (6 Tools)     │ │
│  │     ↓            │    │  • Dashboard                 │ │
│  │ Instagram        │    │  • Analytics                 │ │
│  │                  │    │  • Allocations               │ │
│  └──────────────────┘    │  • Prize Draw                │ │
│                          │  • Staff Log (NEW)           │ │
│  ┌──────────────────┐    │  • Guest Prize Entries (NEW) │ │
│  │  DATA BACKENDS   │    │  • Guest Scans               │ │
│  ├──────────────────┤    └──────────────────────────────┘ │
│  │ guestNameEntries │                                     │
│  │ staffActivityLog │    ┌──────────────────────────────┐ │
│  │ guestScanLog     │    │   THEME                      │ │
│  │ tickets (Map)    │    ├──────────────────────────────┤ │
│  └──────────────────┘    │ ☀ Light / Dark Toggle        │ │
│                          │   Bright Dark Gold Theme     │ │
│                          │   (Persistent in localStorage│ │
│                          └──────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## User Flow Diagrams

### Guest Flow (NEW)
```
[Guest with Ticket]
        ↓
  Scan QR Code
        ↓
/ticket?token=XXX&sig=XXX
        ↓
  ┌─────────────────────────┐
  │  WELCOME PAGE (NEW)     │
  ├─────────────────────────┤
  │  ✅ Status Alert        │
  │  ❤️ ♠️ Hearts/Spades     │
  │  🎉 Welcome Message     │
  │  🎵 Audio Player        │
  │  ┌─────────────────────┐│
  │  │ 🎁 PRIZE ENTRY(NEW) ││
  │  │ [Name Input Field]  ││
  │  │ [Submit Button]     ││
  │  │ (One entry only)    ││
  │  └─────────────────────┘│
  │  ⏱️ Countdown: 25s       │
  └─────────────────────────┘
        ↓
  [Backend logs entry if submitted]
        ↓
  Redirects to Instagram
```

### Staff Flow
```
[Staff Member]
     ↓
/staff?key=AURA2026
     ↓
[Login with Name + PIN]
     ↓
[Staff Home]
  ├─ Manual Check-In
  ├─ Camera Scanner
  └─ Security Monitor
```

### Management Flow (NEW)
```
[Authorized Manager: RAY, SHAWN, NIQUE, CHE]
     ↓
/management-hub?key=MGMT2026
     ↓
[Authorization Check]
  └─ Name not in list? → Alert + Restricted
  └─ Correct PIN? → Full Access
     ↓
┌──────────────────────────────┐
│   MANAGEMENT HUB (6 Tools)   │
├──────────────────────────────┤
│ 📊 Dashboard                 │
│ 📈 Live Analytics            │
│ 📑 Allocations               │
│ 🎉 Prize Draw (Classic)      │
│ 🎁 Guest Prize Entries (NEW) │
│ 👥 Staff Log (ENHANCED)      │
│ 🎫 Guest Scans               │
└──────────────────────────────┘
```

---

## New Pages Map

### /guest-prize-entries (NEW)
```
┌─────────────────────────────────────────┐
│  🎁 Guest Prize Draw Entries            │
├─────────────────────────────────────────┤
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ Stats                           │   │
│  │ Total Entries: 47               │   │
│  │ Unique Tickets: 45              │   │
│  └─────────────────────────────────┘   │
│                                         │
│  [🎲 Draw Random Winner Button]         │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ 🎉 WINNER 🎉                    │   │
│  │ Sarah Johnson                   │   │
│  │ Ticket: AURA-045                │   │
│  └─────────────────────────────────┘   │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ Entry Table:                    │   │
│  │ Name | Ticket | Token | IP|Time │   │
│  │ Sarah|AURA-045| ab... |IP |8:30│   │
│  │ Mike |AURA-032| cd... |IP |8:35│   │
│  │ Jane |AURA-091| ef... |IP |8:40│   │
│  └─────────────────────────────────┘   │
│                                         │
│ [← Back to Management Hub]              │
└─────────────────────────────────────────┘
```

### /staff-log (ENHANCED)
```
┌─────────────────────────────────────────┐
│  👥 Staff Activity Log                  │
├─────────────────────────────────────────┤
│  Track staff logins and system usage    │
│                                         │
│  ┌─────────────────────────────────┐   │
│  │ Table:                          │   │
│  │ Name  |Role    |Action |IP|Time │   │
│  │ Ray   |👔 MGMT|login  |IP|8:00 │   │
│  │ Ray   |👔 MGMT|logout |IP|8:45 │   │
│  │ Sarah |👥 STAFF|login |IP|8:05 │   │
│  │ Sarah |👥 STAFF|logout|IP|8:30 │   │
│  │ Mike  |👥 STAFF|login |IP|8:10 │   │
│  └─────────────────────────────────┘   │
│                                         │
│ [← Back to Management Hub]              │
└─────────────────────────────────────────┘
```

---

## PIN & Access Control Matrix

```
┌─────────────────────────────────────────────────────────┐
│                   ACCESS CONTROL                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ URL                    │ PIN         │ Authorization   │
│ ─────────────────────────────────────────────────────── │
│ /staff                 │ AURA2026    │ Anyone          │
│ /management-hub        │ MGMT2026    │ Mgr Names + PIN │
│ /guest-prize-entries   │ MGMT2026    │ Mgr Names + PIN │
│ /staff-log             │ MGMT2026    │ Mgr Names + PIN │
│ /live-analytics        │ MGMT2026    │ Mgr Names + PIN │
│ /allocations           │ MGMT2026    │ Mgr Names + PIN │
│ /giveaway              │ MGMT2026    │ Mgr Names + PIN │
│ /guest-scans           │ MGMT2026    │ Mgr Names + PIN │
│ /dashboard             │ MGMT2026    │ Mgr Names + PIN │
│                                                         │
│ Authorized Managers:   RAY, RAYMOND, SHAWN, NIQUE, CHE │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Color Theme Reference

### Dark Mode (Default)
```
Background:  #050007 (Deep Purple)
Cards:       #110011 (Dark Purple)
Primary:     #ff1744 (Bright Red)
Accent:      #ff4081 (Pink)
Gold:        #ffb300 (Yellow-Gold)
Text:        #f5f5f5 (White)
Muted:       #aaaaaa (Gray)
```

### Light Mode - Bright Dark Gold (NEW)
```
Background:  #daa520 (Goldenrod)
Cards:       Gradient #f4e4c1 → #daa520
Primary:     #b8860b (Dark Goldenrod)
Secondary:   #8b6914 (Darker Gold)
Text:        #2d2416 (Dark Brown)
Accents:     Various gold tones
```

---

## Data Flow Diagrams

### Guest Prize Entry Flow (NEW)
```
Guest Page
    ↓
[User enters name]
    ↓
Click "Submit for Prize Draw"
    ↓
Frontend validates (not empty)
    ↓
POST /api/guest-name-entry
    {ticketId, token, guestName}
    ↓
Backend validates:
  • Token exists? ✓
  • No duplicate entry? ✓
  • Fields complete? ✓
    ↓
Save to guestNameEntries array
    ↓
Return: {"success": true}
    ↓
Frontend shows "✅ You're entered!"
    ↓
Button disabled (no double-submit)
    ↓
After 25s → Redirect to Instagram
```

### Staff Login Flow (ENHANCED)
```
Staff logs in with name + PIN
    ↓
POST /api/staff-login
    {name: "Ray"}
    ↓
Backend:
  • Get client IP (real IP captured) ✓
  • Check if name in ALLOWED_MANAGERS
  • Assign role: "management" or "staff" ✓
    ↓
Save to staffActivityLog:
  {
    name: "Ray",
    action: "login",
    role: "management",     ← NEW
    ip: "192.168.1.100",    ← NEW
    timestamp: "2025-12-05T20:30:00Z"
  }
    ↓
Return: {"success": true, "role": "management"}
    ↓
Frontend displays staff home
```

---

## Feature Checklist

### Guest Experience
- ✅ Welcome page with status alert
- ✅ Audio player with auto-play + manual override
- ✅ Prize entry form (NEW)
- ✅ One entry per ticket validation (NEW)
- ✅ 25-second countdown
- ✅ Auto-redirect to Instagram
- ✅ Light mode support

### Staff Features
- ✅ Manual ticket check-in
- ✅ Mark tickets as used
- ✅ Camera scanner (existing)
- ✅ Security monitoring (existing)
- ✅ Activity logging (ENHANCED)

### Management Features (NEW)
- ✅ Guest prize entries management
- ✅ Random winner selection
- ✅ Enhanced staff activity log
- ✅ View guest entries with stats
- ✅ All accessible with MGMT PIN + name auth
- ✅ Light/dark theme toggle
- ✅ All pages responsive

---

## Mobile Responsiveness

### Breakpoints
- Desktop: 1200px+
- Tablet: 720px - 1199px
- Mobile: < 720px

### Mobile Optimizations
- Single column layouts
- Larger touch targets (44px+ buttons)
- Readable font sizes (no smaller than 14px)
- Properly scaled images
- Audio overlay for playback permission

---

## Browser Support

| Browser | Support | Notes |
|---------|---------|-------|
| Chrome | ✅ Full | Recommended |
| Safari | ✅ Full | iOS & macOS |
| Firefox | ✅ Full | Desktop |
| Edge | ✅ Full | Windows |
| Opera | ✅ Full | Alternative |

### Features Requiring Modern Browser
- localStorage (theme persistence)
- Fetch API (guest entry submission)
- Flexbox/Grid (layouts)
- CSS gradients (styling)

---

## Common Tasks

### How to Add New Manager
1. Find ALLOWED_MANAGERS array in server.js
2. Add name to array: `["RAY", "SHAWN", "NIQUE", "CHE", "NEWNAME"]`
3. Restart server
4. New manager can login to /management-hub

### How to Change Light Mode Colors
1. Find themeCSSRoot() function
2. Modify color values:
   - `#daa520` = main gold
   - `#b8860b` = dark gold
   - `#2d2416` = text color
3. Restart server

### How to Change Countdown Timer
1. Find guest welcome page in /ticket route
2. Change: `const REDIRECT_SECONDS = 25;`
3. Restart server

### How to View Guest Entries
1. Login as manager (RAY, SHAWN, etc.)
2. Go to Management Hub
3. Click "🎁 Guest Entries"
4. See entries populate in real-time

---

## Troubleshooting Map

```
Issue                          → Solution
─────────────────────────────────────────────
Light mode not saving          → Clear localStorage
Guest entry not submitting     → Check API endpoint + network
Staff log empty                → Verify login endpoint called
Management hub redirects       → Check PIN + name auth
Theme toggle not visible       → Check CSS loaded
Mobile audio not playing       → Tap overlay button
Prize draw select same user    → Normal (random selection)
Countdown timer wrong          → Verify REDIRECT_SECONDS value
```

---

## Performance Notes

### Data Limits (Auto-trim)
- `staffActivityLog`: 500 entries max
- `guestNameEntries`: 2000 entries max
- `guestScanLog`: 1000 entries max
- `tickets`: Persisted to disk (no limit)

### Memory Usage
- Minimal (in-memory, no database)
- ~1-2MB per 500 staff entries
- ~1-2MB per 500 guest entries
- Scales well for single-event use

### Refresh Rates
- Staff log: Manual refresh
- Guest entries: Auto 5-second refresh
- Live analytics: Auto 3-second refresh
- Analytics: Live updates via Fetch API

---

**Last Updated**: December 5, 2025  
**Status**: Production Ready ✅  
**All Features Implemented**: ✅
