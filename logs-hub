// server.js (updated with requested changes)

// CORE SETUP ----------------------------------------------------
const express = require("express");
const path = require("path");
const fs = require("fs");
const crypto = require("crypto");
const QRCode = require("qrcode");

const app = express();
const PORT = process.env.PORT || 3000;

// AURA STAFF LOGIN HOMEPAGE
app.get("/", (req, res) => {
  res.send(`
    <!doctype html>
    <html lang="en">
    <head>
      <meta charset="utf-8" />
      <title>AURA Ticket System — Staff Login</title>
      <meta name="viewport" content="width=device-width, initial-scale=1" />
      <style>
        body {
          margin:0;
          min-height:100vh;
          display:flex;
          align-items:center;
          justify-content:center;
          background: radial-gradient(circle at top left,#4b004b,#050510 55%,#000000);
          font-family: system-ui,-apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif;
          color:#fff;
        }
        .card {
          background:rgba(8,8,24,0.95);
          border-radius:18px;
          padding:26px 28px 22px;
          max-width:380px;
          width:100%;
          box-shadow:0 22px 40px rgba(0,0,0,0.7);
          border:1px solid rgba(255,255,255,0.07);
        }
        h1 {
          margin:0 0 6px;
          font-size:22px;
          background:linear-gradient(135deg,#ff4b9a,#ffb347);
          -webkit-background-clip:text;
          color:transparent;
        }
        h2 {
          margin:0 0 14px;
          font-size:13px;
          text-transform:uppercase;
          letter-spacing:0.16em;
          color:#f3c1ff;
        }
        label {
          display:block;
          font-size:12px;
          margin-bottom:6px;
          text-transform:uppercase;
          letter-spacing:0.09em;
          color:#9ea0ff;
        }
        input {
          width:100%;
          padding:9px 11px;
          border-radius:999px;
          border:1px solid #2c2e5a;
          background:#050515;
          color:#fff;
          font-size:14px;
          margin-bottom:14px;
          box-sizing:border-box;
        }
        button {
          width:100%;
          padding:11px 12px;
          border-radius:999px;
          border:none;
          font-weight:600;
          font-size:14px;
          cursor:pointer;
          background:linear-gradient(135deg,#ff3c88,#ffb347);
          color:#050510;
          text-transform:uppercase;
          letter-spacing:0.14em;
        }
        button:active {
          transform:translateY(1px);
        }
        .note {
          margin-top:10px;
          font-size:11px;
          color:#8889c7;
          text-align:center;
        }
      </style>
    </head>
    <body>
      <div class="card">
        <h2>A.U.R.A by POP</h2>
        <h1>AURA Ticket System</h1>
        <p style="margin:0 0 18px;font-size:13px;color:#d4d5f7;">
          Enter your name and staff PIN to unlock ticket tools.
        </p>

        <form onsubmit="event.preventDefault(); goToStaff();">
          <label for="name">Your Name</label>
          <input id="name" name="name" placeholder="e.g. Ray" autocomplete="off" required />

          <label for="pin">Staff PIN</label>
          <input id="pin" name="pin" type="password" required />

          <button type="submit">Enter Aura</button>
          <div class="note">Internal use only • Hearts &amp; Spades // Event Ops</div>
        </form>
      </div>

      <script>
        function goToStaff() {
          var name = document.getElementById('name').value.trim();
          var pin = document.getElementById('pin').value.trim();
          if (!name || !pin) return;
          var url = '/staff?key=' + encodeURIComponent(pin) + '&name=' + encodeURIComponent(name);
          window.location.href = url;
        }
      </script>
    </body>
    </html>
  `);
});

// 👇 HOST is now configurable (better for ngrok / Wi-Fi changes)
const HOST = process.env.HOST || "0.0.0.0"; // listen on all interfaces by default
// Optional: external/public base URL (e.g. ngrok HTTPS URL)
// If BASE_URL is not set, we fall back to whatever host/protocol the request used.
const BASE_URL = process.env.BASE_URL || "";




function getBaseUrl(req) {
  // If you set BASE_URL in env, always use that
  if (BASE_URL) {
    return BASE_URL;
  }

  // Otherwise, use the actual request (works with ngrok, local IP, etc.)
  const proto = (req.headers["x-forwarded-proto"] || req.protocol || "http").split(",")[0];
  const host =
    req.headers["x-forwarded-host"] ||
    req.headers.host ||
    `${HOST}:${PORT}`;

  return `${proto}://${host}`;
}

// Where normal guests go
const INSTAGRAM_URL = "https://www.instagram.com/a.u.r.a_by_pop/";

// Staff PIN to protect staff tools
const STAFF_PIN = "AURA2026";

// Management PIN (for dashboard / analytics / allocations / prize draws / logs)
const MANAGEMENT_PIN = "POP!";

// HMAC secret key for QR signature verification (prevents forged QR codes)
// IMPORTANT: Must be fixed/persistent so QR signatures remain valid across server restarts
const HMAC_SECRET = Buffer.from('aura_event_hmac_secret_key_2026_fixed_persistent_12345', 'utf-8').slice(0, 32);

// File where tickets are saved
const DATA_FILE = path.join(__dirname, "tickets.json");

// Helper: parse cookies from request
function parseCookies(req) {
  const header = req.headers && req.headers.cookie;
  const out = {};
  if (!header) return out;
  header.split(';').forEach(pair => {
    const idx = pair.indexOf('=');
    if (idx === -1) return;
    const key = pair.slice(0, idx).trim();
    const val = pair.slice(idx + 1).trim();
    out[key] = decodeURIComponent(val);
  });
  return out;
}

function pushWithLimit(arr, item, limit) {
  arr.push(item);
  if (arr.length > limit) arr.shift();
}

// Helper: check management authorization (query key OR cookie)
function isMgmtAuthorizedReq(req) {
  try {
    const q = req.query || {};
    if (q.key === MANAGEMENT_PIN) return true;
    const cookies = parseCookies(req);
    if (cookies.mgmtAuth === '1' && cookies.mgmtName) {
      const allowed = ["RAY","RAYMOND","SHAWN","NIQUE","CHE"];
      return allowed.includes(String(cookies.mgmtName).trim().toUpperCase());
    }
    return false;
  } catch (e) { return false; }
}

// Folder where QR images are saved
const QR_DIR = path.join(__dirname, "generated_qr");
app.use("/generated_qr", express.static(QR_DIR));

// In-memory ticket store: token -> { id, type, status }
const tickets = new Map();

// Track invalid and duplicate scan attempts
const scanEvents = {
  invalid: [],   // tokens that don't exist in tickets map
  duplicates: [] // tokens scanned multiple times
};

// Track payment transactions (no currency conversion any more)
const paymentEvents = {
  transactions: [] // { ticketId, method, amount, currency, notes, timestamp }
};

// Track box office sales
const boxOfficeSales = {
  prefix: "BOX-",
  nextNumber: 1,
  sales: [] // { ticketId, qrPath, timestamp, soldTo, amount }
};

// Track giveaways / prize draws
const giveawayEvents = {
  draws: [] // { prizeId, winnerTicketId, timestamp, prizeDescription }
};

// Track IP activity and suspicious behavior
const ipLogging = {
  events: [], // { ip, token, ticketId, timestamp, status }
  suspicious: new Map() // { ip: [event1, event2, ...] for IPs with >3 scans/min }
};

// NEW: Staff activity log (logins, etc.)
const staffActivityLog = []; // { name, action, ip, timestamp }

// NEW: Guest scan log (for all non-staff scans)
const guestScanLog = []; // { ticketId, token, ip, timestamp }

// NEW: Guest name entries for prize draw (ONE attempt per ticket)
const guestNameEntries = []; // { ticketId, token, guestName, guestEmail, guestPhone, ip, timestamp }

// NEW: ticket allocations (which seller has which ticket)
const ticketAllocations = new Map();
// ticketId -> { sellerName, sellerPhone, sellerEmail }

// NEW: helper to find a display name for a ticket ID
function getNameForTicketId(ticketId) {
  if (!ticketId) return null;

  // NEW: helper to get latest guest info (for logs, mailing list, etc.)
function getGuestInfoForTicket(ticketId) {
  if (!ticketId) return null;

  // Walk backwards so we get the most recent entry
  for (let i = guestNameEntries.length - 1; i >= 0; i--) {
    const e = guestNameEntries[i];
    if (e.ticketId === ticketId) {
      return {
        name: (e.guestName || "").trim(),
        email: (e.guestEmail || "").trim(),
        phone: (e.guestPhone || "").trim(),
        subscribed: !!e.subscribe,
        timestamp: e.timestamp || null,
      };
    }
  }
  return null;
}

  // 1) Prefer guest prize-draw entries (they typed their name there)
  for (let i = guestNameEntries.length - 1; i >= 0; i--) {
    const entry = guestNameEntries[i];
    if (entry.ticketId === ticketId && entry.guestName && entry.guestName.trim()) {
      return entry.guestName.trim();
    }
  }

  // 2) Fall back to box office sales "soldTo" name
  if (boxOfficeSales && Array.isArray(boxOfficeSales.sales)) {
    for (let i = boxOfficeSales.sales.length - 1; i >= 0; i--) {
      const sale = boxOfficeSales.sales[i];
      if (sale.ticketId === ticketId && sale.soldTo && sale.soldTo.trim()) {
        return sale.soldTo.trim();
      }
    }
  }

  // Nothing found
  return null;
}


// Helper function to extract client IP
function getClientIP(req) {
  return (
    req.headers["x-forwarded-for"]?.split(",")[0].trim() ||
    req.headers["x-real-ip"] ||
    req.connection.remoteAddress ||
    req.socket.remoteAddress ||
    "unknown"
  );
}

// Helper function to check for suspicious activity (>3 scans per minute from same IP)
function checkSuspiciousActivity(ip) {
  const now = Date.now();
  const oneMinuteAgo = now - 60000; // 1 minute in ms

  // Count scans from this IP in the last minute
  const recentScans = ipLogging.events.filter(
    (e) => e.ip === ip && e.timestamp > oneMinuteAgo
  );

  return recentScans.length > 3; // Suspicious if more than 3 scans/minute
}

// Ensure QR directory exists
if (!fs.existsSync(QR_DIR)) {
  fs.mkdirSync(QR_DIR, { recursive: true });
}

// Serve anything in /public if we add files later (icons, manifest, audio, etc.)
app.use(express.static(path.join(__dirname, "public")));
app.use(express.json()); // Parse JSON request bodies for POST endpoints

// ------------------------------------------------------
// ADMIN: Clear generated QR PNG files
// ------------------------------------------------------
app.post("/admin/clear-qr-files", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.status(403).json({ ok: false, error: "Unauthorized" });
  }

  try {
    const files = fs.readdirSync(QR_DIR);
    for (const name of files) {
      if (name.toLowerCase().endsWith(".png")) {
        fs.unlinkSync(path.join(QR_DIR, name));
      }
    }
    return res.json({ ok: true });
  } catch (err) {
    console.error("Error clearing QR files:", err);
    return res.status(500).json({ ok: false, error: "Server error" });
  }
});

// ------------------------------------------------------
// API: Full allocation log (per ticket, joined with guest info)
// Management only
// ------------------------------------------------------
app.get("/api/allocations-detail", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.status(403).json({ ok: false, error: "Unauthorized" });
  }

  const items = [];

  // Loop through all allocations
  for (const [ticketId, alloc] of ticketAllocations.entries()) {
    // Find the ticket record (status/type)
    let ticketRec = null;
    for (const [_token, rec] of tickets.entries()) {
      if (rec.id === ticketId) {
        ticketRec = rec;
        break;
      }
    }

    // Find the most recent guest entry (if any) for this ticket
    let guest = null;
    for (let i = guestNameEntries.length - 1; i >= 0; i--) {
      const e = guestNameEntries[i];
      if (e.ticketId === ticketId) {
        guest = e;
        break;
      }
    }

    items.push({
      ticketId,
      // seller / allocation info
      sellerName: alloc.sellerName || "",
      sellerPhone: alloc.sellerPhone || "",
      sellerEmail: alloc.sellerEmail || "",
      sold: !!alloc.sold,

      // ticket technical info
      ticketStatus: ticketRec ? ticketRec.status : "missing",
      ticketType: ticketRec ? ticketRec.type : null,

      // buyer / guest info captured by scanner
      guestName: guest ? guest.guestName : null,
      guestEmail: guest ? guest.guestEmail : null,
      guestPhone: guest ? guest.guestPhone : null,
      lastGuestEntryAt: guest ? guest.timestamp : null
    });
  }

  res.json({ ok: true, allocations: items });
});

// ------------------------------------------------------
// API: Assign a ticket to a seller (allocation)
// Management only
// ------------------------------------------------------
app.post("/api/allocate-ticket", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.status(403).json({ error: "Unauthorized" });
  }

  const { ticketId, sellerName, sellerPhone, sellerEmail } = req.body || {};
  if (!ticketId || !sellerName) {
    return res.status(400).json({ error: "Missing ticketId or sellerName" });
  }

  // Make sure ticket exists
  let exists = false;
  for (const [_token, rec] of tickets.entries()) {
    if (rec.id === ticketId) {
      exists = true;
      break;
    }
  }
  if (!exists) {
    return res.status(404).json({ error: "Ticket not found" });
  }

  ticketAllocations.set(ticketId, {
    sellerName: sellerName.trim(),
    sellerPhone: (sellerPhone || "").trim(),
    sellerEmail: (sellerEmail || "").trim(),
    sold: false // Initialize sold status as false
  });

  return res.json({ ok: true });
});

// ------------------------------------------------------
// API: Seller allocation summary (sold vs unsold)
// Management only
// ------------------------------------------------------
app.get("/api/seller-allocations-summary", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.status(403).json({ ok: false, error: "Unauthorized" });
  }

  const sellers = {};
  let totalAllocated = 0;
  let totalSold = 0;

  // Loop through all allocations
  for (const [ticketId, alloc] of ticketAllocations.entries()) {
    const name = alloc.sellerName || "Unknown";

    if (!sellers[name]) {
      sellers[name] = {
        sellerName: name,
        totalAllocated: 0,
        sold: 0
      };
    }

    sellers[name].totalAllocated++;
    totalAllocated++;

    // NEW LOGIC:
    // We do NOT check ticket.status anymore.
    // We only check alloc.sold which is set on GUEST scans.
    if (alloc.sold === true) {
      sellers[name].sold++;
      totalSold++;
    }
  }

  const totalUnsold = totalAllocated - totalSold;

  return res.json({
    ok: true,
    sellers: Object.values(sellers),
    totalAllocated,
    totalSold,
    totalUnsold
  });
});

// ------------------------------------------------------
// API: Detailed Allocation Log (per ticket)
// Management only
// ------------------------------------------------------
app.get("/api/allocations-detail", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.status(403).json({ ok: false, error: "Unauthorized" });
  }

  const allocations = [];

  for (const [ticketId, alloc] of ticketAllocations.entries()) {
    // 1) Look up ticket type + status
    let status = "unknown";
    let type = null;

    for (const [_token, rec] of tickets.entries()) {
      if (rec.id === ticketId) {
        status = rec.status || "unused";
        type = rec.type || null;
        break;
      }
    }

    // 2) Guest info (from prize draw / guest entry)
    let guestName = null;
    let guestEmail = null;
    let guestPhone = null;

    for (let i = guestNameEntries.length - 1; i >= 0; i--) {
      const e = guestNameEntries[i];
      if (e.ticketId === ticketId) {
        if (e.guestName && !guestName) guestName = e.guestName;
        if (e.guestEmail && !guestEmail) guestEmail = e.guestEmail;
        if (e.guestPhone && !guestPhone) guestPhone = e.guestPhone;
        break;
      }
    }

    // 3) Fallback guest name from box office sales
    if (!guestName && boxOfficeSales && Array.isArray(boxOfficeSales.sales)) {
      for (let i = boxOfficeSales.sales.length - 1; i >= 0; i--) {
        const sale = boxOfficeSales.sales[i];
        if (sale.ticketId === ticketId && sale.soldTo) {
          guestName = sale.soldTo;
          break;
        }
      }
    }

    // 4) Last activity time (guest scan or guest entry)
    let lastActivityMs = 0;

    for (let i = guestScanLog.length - 1; i >= 0; i--) {
      const evt = guestScanLog[i];
      if (evt.ticketId === ticketId && evt.timestamp) {
        const t = Date.parse(evt.timestamp);
        if (!Number.isNaN(t) && t > lastActivityMs) lastActivityMs = t;
        break;
      }
    }

    for (let i = guestNameEntries.length - 1; i >= 0; i--) {
      const evt = guestNameEntries[i];
      if (evt.ticketId === ticketId && evt.timestamp) {
        const t = Date.parse(evt.timestamp);
        if (!Number.isNaN(t) && t > lastActivityMs) lastActivityMs = t;
        break;
      }
    }

    allocations.push({
      ticketId,
      ticketType: type,
      ticketStatus: status,
      sellerName: alloc.sellerName || "",
      sellerPhone: alloc.sellerPhone || "",
      sellerEmail: alloc.sellerEmail || "",
      sold: !!alloc.sold,
      guestName: guestName || null,
      guestEmail: guestEmail || null,
      guestPhone: guestPhone || null,
      lastActivity: lastActivityMs ? new Date(lastActivityMs).toISOString() : null,
    });
  }

  // Optional: sort by seller then ticket ID
  allocations.sort((a, b) => {
    const ns = (a.sellerName || "").localeCompare(b.sellerName || "");
    if (ns !== 0) return ns;
    return (a.ticketId || "").localeCompare(b.ticketId || "");
  });

  const soldCount = allocations.filter(a => a.sold).length;

  res.json({
    ok: true,
    total: allocations.length,
    sold: soldCount,
    unsold: allocations.length - soldCount,
    allocations,
  });
});


// Generate a random token for each ticket
function createToken() {
  return crypto.randomBytes(16).toString("hex"); // 32-char random string
}

// Sign a token with HMAC-SHA256 to prevent QR forgery
function signToken(token) {
  return crypto
    .createHmac("sha256", HMAC_SECRET)
    .update(token)
    .digest("hex")
    .substring(0, 16); // Use first 16 chars of signature
}

// Verify a token's signature
function verifyToken(token, signature) {
  const expectedSignature = signToken(token);
  return crypto.timingSafeEqual(
    Buffer.from(expectedSignature),
    Buffer.from(signature)
  );
}

// ---- PERSISTENCE: LOAD + SAVE TICKETS ----------------

function loadTickets() {
  try {
    if (!fs.existsSync(DATA_FILE)) {
      console.log("No existing tickets file, starting fresh.");
      return;
    }
    const raw = fs.readFileSync(DATA_FILE, "utf8");
    if (!raw.trim()) return;

    const arr = JSON.parse(raw);
    arr.forEach((entry) => {
      tickets.set(entry.token, {
        id: entry.id,
        type: entry.type,
        status: entry.status || "unused",
      });
    });
    console.log(`Loaded ${tickets.size} tickets from disk.`);
  } catch (err) {
    console.error("Error loading tickets:", err);
  }
}

function saveTickets() {
  try {
    const arr = [];
    for (const [token, record] of tickets.entries()) {
      arr.push({
        token,
        id: record.id,
        type: record.type,
        status: record.status,
      });
    }
    fs.writeFileSync(DATA_FILE, JSON.stringify(arr, null, 2));
    console.log(`Saved ${arr.length} tickets to disk.`);
  } catch (err) {
    console.error("Error saving tickets:", err);
  }
}

// SAVE TICKET ALLOCATIONS (seller assignment data)
function saveTicketAllocations() {
  try {
    const allocArr = Array.from(ticketAllocations.entries()).map(([ticketId, alloc]) => {
      return { ticketId, ...alloc };
    });

    fs.writeFileSync(
      path.join(__dirname, "ticket-allocations.json"),
      JSON.stringify(allocArr, null, 2)
    );

    console.log(`[ALLOC] Saved ${allocArr.length} ticket allocations.`);
  } catch (err) {
    console.error("Error saving ticket allocations:", err);
  }
}

function loadTicketAllocations() {
  try {
    const file = path.join(__dirname, "ticket-allocations.json");
    if (!fs.existsSync(file)) return;

    const arr = JSON.parse(fs.readFileSync(file, "utf8"));
    ticketAllocations.clear();

    arr.forEach(a => {
      ticketAllocations.set(a.ticketId, {
        sellerName: a.sellerName,
        sellerPhone: a.sellerPhone,
        sellerEmail: a.sellerEmail,
        sold: a.sold || false
      });
    });

    console.log(`[ALLOC] Loaded ${arr.length} allocations.`);
  } catch (err) {
    console.error("Error loading allocations:", err);
  }
}

// LOAD TICKET ALLOCATIONS
function loadTicketAllocations() {
  try {
    const file = path.join(__dirname, "ticket-allocations.json");
    if (!fs.existsSync(file)) return;

    const arr = JSON.parse(fs.readFileSync(file, "utf8"));
    ticketAllocations.clear();

    arr.forEach(a => {
      ticketAllocations.set(a.ticketId, {
        sellerName: a.sellerName,
        sellerPhone: a.sellerPhone,
        sellerEmail: a.sellerEmail,
        sold: a.sold || false
      });
    });

    console.log(`[ALLOC] Loaded ${arr.length} allocations.`);
  } catch (err) {
    console.error("Error loading allocations:", err);
  }
}

loadTickets();
loadTicketAllocations();   // ✅ ADD THIS EXACT LINE HERE
// Save tickets and allocations every 5 minutes


// === AUTO-BACKUP SYSTEM (periodic JSON saves) ===
const BACKUP_FILE = path.join(__dirname, 'aura-backup.json');
const BACKUP_INTERVAL = 5 * 60 * 1000; // 5 minutes

function createBackup() {
  try {
    const backup = {
      timestamp: new Date().toISOString(),
      guestNameEntries,
      staffActivityLog,
      guestScanLog,
      scanEvents,
      paymentEvents,
      boxOfficeSales,
      giveawayEvents,
      ipLogging,
      ticketAllocations: Array.from(ticketAllocations.entries()),   // 👈 NEW
      tickets: Array.from(tickets.entries()).map(([token, record]) => ({
        token,
        id: record.id,
        type: record.type,
        status: record.status

        function createBackup() {
  return {
    tickets: [...tickets.entries()],
    ticketAllocations: [...ticketAllocations.entries()],
    guestScanLog,
    guestNameEntries,
    // NEW:
    cancelledTicketsLog,
  };
}

      }))
    };
  
    fs.writeFileSync(BACKUP_FILE, JSON.stringify(backup, null, 2));
    console.log(`[BACKUP] Created at ${backup.timestamp}`);
  } catch (e) {
    console.error('[BACKUP] Error:', e);
  }
}

function restoreBackup() {
  try {
    if (!fs.existsSync(BACKUP_FILE)) {
      console.log('[BACKUP] No backup file found, starting fresh.');
      return;
    }

      if (backup.cancelledTicketsLog) {
    cancelledTicketsLog.length = 0;
    cancelledTicketsLog.push(...backup.cancelledTicketsLog);
  }

    const backup = JSON.parse(fs.readFileSync(BACKUP_FILE, 'utf8'));
    guestNameEntries.length = 0;
    guestNameEntries.push(...(backup.guestNameEntries || []));
    staffActivityLog.length = 0;
    staffActivityLog.push(...(backup.staffActivityLog || []));
    guestScanLog.length = 0;
    guestScanLog.push(...(backup.guestScanLog || []));
    Object.assign(scanEvents, backup.scanEvents || {});
    Object.assign(paymentEvents, backup.paymentEvents || {});
    Object.assign(boxOfficeSales, backup.boxOfficeSales || {});
    Object.assign(giveawayEvents, backup.giveawayEvents || {});
Object.assign(ipLogging, backup.ipLogging || {});

// restore ticket allocations
ticketAllocations.clear();
(backup.ticketAllocations || []).forEach(([ticketId, alloc]) => {
  ticketAllocations.set(ticketId, alloc);
});

tickets.clear();
(backup.tickets || []).forEach(t => {
  tickets.set(t.token, { id: t.id, type: t.type, status: t.status });
});
    console.log(`[BACKUP] Restored ${backup.tickets?.length || 0} tickets and logs from ${backup.timestamp}`);
  } catch (e) {
    console.error('[BACKUP] Restore error:', e);
  }
}

// Restore on startup
restoreBackup();

// Auto-backup every 5 minutes
setInterval(createBackup, BACKUP_INTERVAL);

// Backup on exit
process.on('exit', createBackup);
process.on('SIGINT', () => { createBackup(); process.exit(); });
process.on('SIGTERM', () => { createBackup(); process.exit(); });

// ------------------------------------------------------
// THEME HELPER SNIPPET (shared across pages)
// ------------------------------------------------------
function themeScript() {
  return `
    <script>
      const AURA_THEME_KEY = 'auraTheme';
      function applyAuraTheme() {
        const theme = localStorage.getItem(AURA_THEME_KEY) || 'dark';
        if (theme === 'light') {
          document.documentElement.classList.add('light-mode');
        } else {
          document.documentElement.classList.remove('light-mode');
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
      // Global back button removed per UX request: no automatic back control injected
    </script>
  `;
}

function themeCSSRoot() {
  return `
    :root {
      --bg-dark: #050007;
      --card-dark: #110011;
      --accent-red: #ff1744;
      --accent-pink: #ff4081;
      --accent-gold: #ffb300;
      --text-main: #f5f5f5;
      --text-muted: #aaaaaa;
    }
    html, body {
      transition: background 0.25s ease, color 0.25s ease;
    }
    html.light-mode body {
      background:
        radial-gradient(circle at top left, rgba(218,165,32,0.4), transparent 55%),
        radial-gradient(circle at bottom right, rgba(184,134,11,0.5), transparent 55%),
        #daa520;
      color: #2d2416;
    }
    html.light-mode .card {
      background: radial-gradient(circle at top, #f4e4c1, #daa520 60%);
      box-shadow:
        0 0 0 1px rgba(218,165,32,0.6),
        0 18px 40px rgba(0,0,0,0.35);
      color: #2d2416;
    }
    html.light-mode .badge,
    html.light-mode .role-label {
      border-color: rgba(218,165,32,0.8);
      color: #b8860b;
      background: rgba(244,228,193,0.9);
    }
    html.light-mode .action-button {
      box-shadow: 0 10px 24px rgba(218,165,32,0.5);
    }
    html.light-mode h1, html.light-mode h2 {
      color: #8b6914;
    }
    html.light-mode input[type="text"],
    html.light-mode input[type="password"] {
      background: rgba(255,255,255,0.95);
      border-color: rgba(218,165,32,0.7);
      color: #2d2416;
    }
    html.light-mode input:focus {
      box-shadow: 0 0 0 1px rgba(218,165,32,0.9), 0 0 16px rgba(218,165,32,0.6);
    }
    html.light-mode table th {
      color: #8b6914;
      background: rgba(244,228,193,0.3);
    }
    html.light-mode table td {
      border-color: rgba(218,165,32,0.2);
    }
    html.light-mode .subtitle,
    html.light-mode .note {
      color: #8b6914 !important;
    }
    html.light-mode a {
      color: #b8860b;
    }
  `;
}

// Utility: limit array size
function pushWithLimit(arr, item, limit = 500) {
  arr.push(item);
  if (arr.length > limit) arr.shift();
}

// Always start at staff login page via "/" route defined above.


// ------------------------------------------------------
// ROUTE 1: STAFF PAGE (PIN + MANUAL CHECK)
// ------------------------------------------------------
app.get("/staff", (req, res) => {
  const { key } = req.query;
  // Prevent caching (helps mobile browsers and service workers deliver fresh page)
  res.setHeader("Cache-Control", "no-store, no-cache, must-revalidate, proxy-revalidate");
  res.setHeader("Pragma", "no-cache");
  res.setHeader("Surrogate-Control", "no-store");

  // Log staff access attempts for debugging mobile issues
  try {
    console.log(
      `/staff requested key=${key || "(none)"} UA=${
        req.headers["user-agent"] || "(unknown)"
      }`
    );
  } catch (e) {}

  // WRONG OR MISSING PIN → LOGIN SCREEN
  if (key !== STAFF_PIN) {
    return res.send(`<!DOCTYPE html>
      <html>
      <head>
        <!-- Manifest excluded from staff pages to prevent PWA caching issues -->
        <meta name="theme-color" content="#050007">
        <meta charset="UTF-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1" />
        <meta name="theme-color" content="#ff1744" />
        <title>AURA Staff Login</title>
        <style>
          ${themeCSSRoot()}
          * { box-sizing: border-box; }

          body {
            margin: 0;
            padding: 0;
            min-height: 100vh;
            font-family: system-ui, -apple-system, BlinkMacSystemFont, sans-serif;
            background:
              radial-gradient(circle at top left, rgba(255,64,129,0.25), transparent 55%),
              radial-gradient(circle at bottom right, rgba(255,23,68,0.35), transparent 55%),
              var(--bg-dark);
            color: var(--text-main);
            display: flex;
            align-items: center;
            justify-content: center;
          }

          .card {
            width: 100%;
            max-width: 420px;
            background: radial-gradient(circle at top, #220018, #070008 60%);
            border-radius: 24px;
            padding: 28px 24px 24px;
            box-shadow:
              0 0 0 1px rgba(255,64,129,0.35),
              0 20px 50px rgba(0,0,0,0.95);
            position: relative;
            overflow: hidden;
          }

          .card::before {
            content: "";
            position: absolute;
            inset: -40%;
            background:
              radial-gradient(circle at 10% 0%, rgba(255,255,255,0.06), transparent 40%),
              radial-gradient(circle at 100% 100%, rgba(255,64,129,0.18), transparent 50%);
            opacity: 0.7;
            pointer-events: none;
          }

          .card-inner { position: relative; z-index: 1; }

          .logo-row {
            display:flex;
            align-items:center;
            gap:10px;
            margin-bottom:10px;
          }

          .logo-img {
            height:34px;
            border-radius:8px;
            box-shadow:0 0 14px rgba(255,64,129,0.65);
          }

          .brand-text {
            font-size:0.95rem;
            letter-spacing:0.18em;
            text-transform:uppercase;
            color:var(--text-muted);
          }

          h1 {
            font-size:1.45rem;
            margin:0 0 4px;
          }

          h1 span {
            background: linear-gradient(120deg, #ffb300, #ff4081, #ff1744);
            -webkit-background-clip: text;
            color: transparent;
          }

          .tagline {
            font-size:0.9rem;
            color:var(--text-muted);
            margin-bottom:18px;
          }

          label {
            display:block;
            margin-bottom:6px;
            font-weight:600;
            font-size:0.88rem;
          }

          input[type="password"],
          input[type="text"] {
            width:100%;
            padding:11px 12px;
            border-radius:999px;
            border:1px solid rgba(255,64,129,0.6);
            background:rgba(3,0,5,0.9);
            color:var(--text-main);
            outline:none;
            margin-bottom:10px;
          }

          input[type="password"]:focus,
          input[type="text"]:focus {
            box-shadow:0 0 0 1px rgba(255,183,77,0.9),
                       0 0 16px rgba(255,64,129,0.6);
          }

          button {
            margin-top:14px;
            padding:11px 16px;
            width:100%;
            border-radius:999px;
            border:none;
            cursor:pointer;
            font-weight:650;
            letter-spacing:0.12em;
            text-transform:uppercase;
            background:linear-gradient(120deg,#ff1744,#ff4081);
            color:#050005;
            box-shadow:0 10px 24px rgba(0,0,0,0.9);
          }

          button:hover { filter:brightness(1.08); }

          .footer {
            margin-top:14px;
            font-size:0.8rem;
            color:var(--text-muted);
          }

          .footer span {
            color:var(--accent-gold);
          }

          /* Mobile adjustments */
          body { -webkit-text-size-adjust: 100%; }
          @media (max-width: 520px) {
            .card { max-width: 94%; padding: 18px; border-radius: 16px; margin: 12px; }
            h1 { font-size: 1.2rem; }
            .logo-img { height:28px; }
            .tagline { font-size:0.85rem; }
            label { font-size:0.85rem; }
            input[type="password"] { padding:12px 14px; }
            button { padding:12px 14px; font-size:0.95rem; }
            .footer { font-size:0.78rem; }
          }
        </style>
      </head>
      <body>
        <div class="card">
          <div class="card-inner">
            <div class="logo-row">
              <img src="/aura-logo.png" alt="AURA logo" class="logo-img" />
              <img src="/pop-logo.png" alt="POP logo" class="logo-img" />
              <div>
                <div class="brand-text">A.U.R.A by POP</div>
                <div style="font-size:0.8rem;color:#bbb;">Hearts &amp; Spades // Event Ops</div>
              </div>
            </div>

            <!-- CHANGED HEADING TEXT HERE -->
            <h1><span>AURA Ticket System</span></h1>
            <div class="tagline">Enter your name and staff PIN to unlock ticket tools.</div>

            <form onsubmit="handleLogin(event)">
              <label for="staffName">Your Name</label>
              <input id="staffName" type="text" placeholder="e.g. Ray" required />

              <label for="key">Staff PIN</label>
              <input id="key" type="password" autocomplete="off" required />

              <button type="submit">Enter AURA</button>
            </form>

            <div class="footer">
              <span>Internal use only.</span> For frontline check-in at AURA events.
            </div>
          </div>
        </div>

        ${themeScript()}
        <script>
          function handleLogin(event) {
            event.preventDefault();
            const name = document.getElementById('staffName').value.trim();
            const pin = document.getElementById('key').value;

            if (name && pin) {
              sessionStorage.setItem('staffName', name);

              // NEW: log staff login to backend
              fetch('/api/staff-login', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ name })
              }).catch(() => {});

              window.location.href = '/staff?key=' + encodeURIComponent(pin);
            }
          }
        </script>
      </body>
      </html>`);
  }

  // CORRECT PIN → STAFF TOOLS HOME
  res.send(`<!DOCTYPE html>
    <html>
    <head>
      <meta charset="UTF-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1" />
      <!-- Manifest excluded from staff pages to prevent PWA caching issues -->
      <meta name="theme-color" content="#ff1744" />
      <title>AURA Staff Check-In</title>
      <style>
        ${themeCSSRoot()}
        * { box-sizing: border-box; }

        body {
          margin: 0;
          padding: 18px;
          min-height: 100vh;
          font-family: system-ui, -apple-system, BlinkMacSystemFont, sans-serif;
          background:
            radial-gradient(circle at top left, rgba(255,64,129,0.25), transparent 55%),
            radial-gradient(circle at bottom right, rgba(255,23,68,0.35), transparent 55%),
            var(--bg-dark);
          color: var(--text-main);
          display: flex;
          align-items: center;
          justify-content: center;
        }

        .card {
          width: 100%;
          max-width: 520px;
          background: radial-gradient(circle at top, #220018, #070008 60%);
          border-radius: 24px;
          padding: 24px 22px 20px;
          box-shadow:
            0 0 0 1px rgba(255,64,129,0.35),
            0 20px 50px rgba(0,0,0,0.95);
          position: relative;
          overflow: hidden;
        }

        .card::before {
          content: "";
          position: absolute;
          inset: -40%;
          background:
            radial-gradient(circle at 0% 0%, rgba(255,255,255,0.06), transparent 40%),
            radial-gradient(circle at 100% 100%, rgba(255,64,129,0.18), transparent 50%);
          opacity: 0.7;
          pointer-events: none;
        }

        .card-inner { position: relative; z-index: 1; }

        .logo-row {
          display:flex;
          align-items:center;
          gap:10px;
          margin-bottom:10px;
        }

        .logo-img {
          height:30px;
          border-radius:8px;
          box-shadow:0 0 12px rgba(255,64,129,0.55);
        }

        .title-row {
          display:flex;
          justify-content:space-between;
          align-items:center;
          gap:12px;
          margin-bottom:10px;
        }

        .title-main {
          font-size:1.35rem;
          font-weight:650;
        }

        .title-main span {
          background: linear-gradient(120deg, #ffb300, #ff4081, #ff1744);
          -webkit-background-clip: text;
          color: transparent;
        }

        .badge {
          padding:5px 10px;
          border-radius:999px;
          border:1px solid rgba(255,64,129,0.6);
          font-size:0.7rem;
          letter-spacing:0.12em;
          text-transform:uppercase;
          color:var(--text-muted);
        }

        .subtitle {
          font-size:0.9rem;
          color:var(--text-muted);
          margin-bottom:16px;
        }

        label {
          display:block;
          margin-bottom:6px;
          font-weight:600;
          font-size:0.85rem;
        }

        input[type="text"] {
          width:100%;
          padding:11px 12px;
          border-radius:999px;
          border:1px solid rgba(255,64,129,0.6);
          background:rgba(3,0,5,0.95);
          color:var(--text-main);
          outline:none;
        }

        input[type="text"]:focus {
          box-shadow:0 0 0 1px rgba(255,183,77,0.9),
                     0 0 16px rgba(255,64,129,0.6);
        }

        button {
          margin-top:12px;
          padding:10px 16px;
          border-radius:999px;
          border:none;
          cursor:pointer;
          font-weight:600;
          letter-spacing:0.12em;
          text-transform:uppercase;
          background:linear-gradient(120deg,#ff1744,#ff4081);
          color:#050005;
          box-shadow:0 10px 24px rgba(0,0,0,0.9);
        }

        button:hover { filter:brightness(1.08); }

        .note {
          font-size:0.8rem;
          color:var(--text-muted);
          margin-top:8px;
        }

        .button-row {
          display:flex;
          gap:14px;
          margin-top:22px;
          flex-wrap:wrap;
        }

        .action-button {
          flex:1;
          min-width:140px;
          padding:14px 18px;
          border-radius:999px;
          border:none;
          cursor:pointer;
          font-weight:700;
          letter-spacing:0.1em;
          text-transform:uppercase;
          font-size:0.88rem;
          text-decoration:none;
          display:inline-flex;
          align-items:center;
          justify-content:center;
          gap:8px;
          transition:filter 0.25s, transform 0.2s;
          box-shadow: 0 10px 28px rgba(0,0,0,0.4);
        }

        .action-button:hover { filter:brightness(1.15); transform: translateY(-2px); }
        .action-button:active { transform: translateY(0); }

        .scanner-btn {
          background:linear-gradient(135deg,#00bcd4,#0097a7);
          color:#fff;
          box-shadow: 0 10px 28px rgba(0,188,212,0.4);
        }

        .security-btn {
          background:linear-gradient(135deg,#d32f2f,#c62828);
          color:#fff;
          box-shadow: 0 10px 28px rgba(211,47,47,0.4);
        }

        .mgmt-hub-btn {
          background:linear-gradient(135deg,#ffc107,#ff9800);
          color:#050005;
          box-shadow: 0 10px 28px rgba(255,183,0,0.4);
        }

        .links {
          margin-top:16px;
          font-size:0.86rem;
          color:var(--text-muted);
        }

        .links a {
          color:var(--accent-gold);
          text-decoration:none;
        }

        .links a:hover { text-decoration:underline; }

        .staff-header {
          display:flex;
          justify-content:space-between;
          align-items:center;
          gap:12px;
          margin-bottom:16px;
          padding-bottom:12px;
          border-bottom:1px solid rgba(255,64,129,0.2);
        }

        .staff-name {
          font-size:0.9rem;
          color:var(--text-muted);
        }

        .logout-btn {
          padding:8px 14px;
          border-radius:999px;
          background:rgba(211,47,47,0.8);
          color:#fff;
          border:none;
          font-size:0.8rem;
          cursor:pointer;
          font-weight:600;
          text-transform:uppercase;
          letter-spacing:0.08em;
        }

        .logout-btn:hover { background:rgba(211,47,47,1); }

        .role-section {
          margin-bottom:20px;
        }

        .role-label {
          display:inline-block;
          padding:6px 12px;
          border-radius:8px;
          background:rgba(76,175,80,0.15);
          border:1px solid rgba(76,175,80,0.4);
          color:#34c759;
          font-size:0.75rem;
          font-weight:700;
          text-transform:uppercase;
          letter-spacing:0.1em;
          margin-bottom:14px;
        }

        .role-section.management .role-label {
          background:rgba(156,39,176,0.15);
          border-color:rgba(156,39,176,0.4);
          color:#9c27b0;
        }

        .theme-toggle {
          padding:6px 10px;
          border-radius:999px;
          border:1px solid rgba(255,255,255,0.25);
          background:rgba(0,0,0,0.3);
          color:#fff;
          font-size:0.75rem;
          cursor:pointer;
          margin-left:8px;
        }

        body { -webkit-text-size-adjust: 100%; }
        @media (max-width: 720px) {
          .card { max-width: 96%; padding: 18px; border-radius: 16px; margin: 12px; }
          .title-main { font-size: 1.05rem; }
          .logo-img { height:26px; }
          .subtitle { font-size:0.88rem; }
          .action-button { min-width: 120px; padding: 12px 14px; font-size:0.9rem; }
          .button-row { gap:10px; }
          .note { font-size:0.86rem; }
        }
      </style>
    </head>
    <body>
      <div class="card">
        <div class="card-inner">
          <div class="staff-header">
            <div class="staff-name">👤 <span id="staffNameDisplay">Staff</span></div>
            <div>
              <button class="theme-toggle" type="button" onclick="AURA_THEME.toggle()">☀ Light / Dark</button>
              <button class="logout-btn" onclick="handleLogout()">🚪 Logout</button>
            </div>
          </div>

          <div class="title-row">
            <div>
              <div class="logo-row">
                <img src="/aura-logo.png" alt="AURA logo" class="logo-img" />
                <img src="/pop-logo.png" alt="POP logo" class="logo-img" />
              </div>
              <div class="title-main"><span>AURA</span> Staff Check-In</div>
              <div style="font-size:0.8rem;color:#bbb;">Hearts &amp; Spades • Door &amp; VIP Control</div>
            </div>
            <div class="badge">INTERNAL ONLY</div>
          </div>

          <div class="subtitle">
            Paste the scanned <strong>ticket URL</strong> or <strong>token</strong> below to view and mark tickets.
          </div>
          <form id="checkForm">
            <label for="code">Ticket link or token</label>
            <input id="code" type="text" placeholder="Paste QR link here" autocomplete="off" />
            <button type="submit">Check ticket</button>
            <div class="note">
              • You can paste the full URL (e.g. <code>http://.../ticket?token=...</code>)<br/>
              • or just the token string itself.
            </div>
          </form>

          <!-- STAFF ONLY SECTION -->
          <div class="role-section">
            <div class="role-label">👥 STAFF ACCESS ONLY</div>
            <div class="button-row">
              <a href="/staff-scanner?key=${encodeURIComponent(STAFF_PIN)}" class="action-button scanner-btn">📷 Camera Scanner</a>
              <a href="/security?key=${encodeURIComponent(STAFF_PIN)}" class="action-button security-btn">🔒 Security</a>
            </div>
          </div>

          <!-- MANAGEMENT ONLY SECTION → now routed to MANAGEMENT HUB -->
          <div class="role-section management">
            <div class="role-label">⚙️ MANAGEMENT ACCESS ONLY</div>
           <div class="button-row">
  <button type="button" class="action-button mgmt-hub-btn" onclick="openManagementHub()">
    🧭 Management Hub
  </button>
  
</div>

            </div>
          </div>
        </div>
      </div>
<!-- Management Login Modal (styled) -->
<div id="mgmtLoginModal" style="display:none;position:fixed;inset:0;align-items:center;justify-content:center;background:rgba(0,0,0,0.6);z-index:9999;">
  <div style="width:100%;max-width:420px;border-radius:12px;padding:18px;background:linear-gradient(180deg,#120012,#1a0018);box-shadow:0 12px 40px rgba(0,0,0,0.8);border:1px solid rgba(255,64,129,0.12);">
    <h3 style="margin:0 0 6px;color:#ffd86b;font-size:1.1rem">Management Login</h3>
    <div style="color:#ccc;font-size:0.9rem;margin-bottom:10px">Enter manager name and PIN to continue.</div>

    <form id="mgmtLoginForm" onsubmit="submitMgmtLoginForm(event)">
      <input id="mgmtNameInput" placeholder="Manager name" autocomplete="off"
             style="width:100%;padding:10px 12px;border-radius:8px;border:1px solid rgba(255,64,129,0.25);background:rgba(3,0,5,0.6);color:#fff;margin-bottom:8px" />

      <input id="mgmtPinInput" placeholder="Management PIN" type="password" autocomplete="off"
             style="width:100%;padding:10px 12px;border-radius:8px;border:1px solid rgba(255,64,129,0.25);background:rgba(3,0,5,0.6);color:#fff;margin-bottom:8px" />

      <div id="mgmtLoginError" style="color:#ffb3b3;min-height:18px;margin-bottom:8px"></div>

      <div style="display:flex;gap:8px">
        <button type="submit"
                style="flex:1;padding:10px 12px;border-radius:8px;border:none;background:linear-gradient(90deg,#ffd86b,#ffb300);font-weight:700">
          Enter
        </button>
      </div>
    </form>

    <!-- NEW: Back to staff page link -->
    <a href="#"
       onclick="closeMgmtModal(); return false;"
       style="display:block;margin-top:10px;font-size:0.85rem;color:#ffb347;text-align:center;text-decoration:none;">
      ← Back to Staff Page
    </a>
  </div>
</div>


      ${themeScript()}
      <script>
        const STAFF_KEY = ${JSON.stringify(STAFF_PIN)};
        const MGMT_KEY = ${JSON.stringify(MANAGEMENT_PIN)};
const ALLOWED_MANAGERS = [
  "RAY",
  "SHAWN",
  "NIQUE",
  "CHE"
];

        function displayStaffName() {
          const staffName = sessionStorage.getItem('staffName') || 'Staff';
          document.getElementById('staffNameDisplay').textContent = staffName;
        }

        function handleLogout() {
          sessionStorage.removeItem('staffName');
          window.location.href = '/staff';
        }

        function isManagerName(name) {
          if (!name) return false;
          const normalized = name.trim().toUpperCase();
          return ALLOWED_MANAGERS.includes(normalized);
        }

        // Show a styled modal for management login (name + PIN)
        function openManagementHub() {
          const modal = document.getElementById('mgmtLoginModal');
          if (modal) {
            modal.style.display = 'flex';
            document.getElementById('mgmtNameInput').focus();
          } else {
            // fallback to previous prompt behavior
            let name = sessionStorage.getItem('staffName') || '';
            if (!isManagerName(name)) {
              name = (prompt('Manager name (as listed):') || '').trim();
              if (!name) return;
            }
            const pin = (prompt('Enter Management PIN:') || '').trim();
            if (!pin) return;
            try {
              fetch('/api/mgmt-login', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ name: name, pin: pin })
              }).then(r => r.json()).then(j => {
                if (j && j.success) {
                  sessionStorage.setItem('staffName', name);
                  window.location.href = '/management-hub';
                } else {
                  alert(j && j.error ? j.error : 'Auth failed');
                }
              }).catch(err => { console.error(err); alert('Auth failed'); });
            } catch (e) { console.error(e); alert('Auth failed'); }
          }
        }

        function closeMgmtModal() {
          const modal = document.getElementById('mgmtLoginModal');
          if (modal) modal.style.display = 'none';
        }

        async function submitMgmtLoginForm(ev) {
          ev && ev.preventDefault();
          const name = (document.getElementById('mgmtNameInput').value || '').trim();
          const pin = (document.getElementById('mgmtPinInput').value || '').trim();
          const errBox = document.getElementById('mgmtLoginError');
          if (!name || !pin) {
            if (errBox) errBox.textContent = 'Please provide name and PIN.';
            return;
          }
          try {
            const res = await fetch('/api/mgmt-login', {
              method: 'POST', headers: { 'Content-Type': 'application/json' },
              body: JSON.stringify({ name, pin })
            });
            const j = await res.json();
            if (j && j.success) {
              sessionStorage.setItem('staffName', name);
              closeMgmtModal();
              window.location.href = '/management-hub';
            } else {
              if (errBox) errBox.textContent = j && j.error ? j.error : 'Auth failed';
            }
          } catch (e) {
            console.error(e);
            if (errBox) errBox.textContent = 'Auth failed';
          }
        }

        displayStaffName();

        const form = document.getElementById("checkForm");
        form.addEventListener("submit", function (e) {
          e.preventDefault();
          const raw = document.getElementById("code").value.trim();
          if (!raw) return;

          let token = raw;
          const idx = raw.indexOf("token=");
          if (idx !== -1) {
            token = raw.substring(idx + 6).split("&")[0];
          }

          window.location.href =
            "/ticket?token=" + encodeURIComponent(token) +
            "&staff=1&key=" + encodeURIComponent(STAFF_KEY);
        });

        // Prevent service worker from interfering with staff tools
        if ('serviceWorker' in navigator) {
          navigator.serviceWorker.getRegistrations().then(registrations => {
            registrations.forEach(registration => registration.unregister());
          });
        }
      </script>
    </body>
    </html>`);
});
// ------------------------------------------------------
// STAFF TICKET GENERATOR SELECTION PAGE
// ------------------------------------------------------
app.get("/staff/generate", (req, res) => {
  const { key } = req.query;

  // ✅ allow either STAFF or MANAGEMENT PIN
  if (!(key === STAFF_PIN || key === MANAGEMENT_PIN)) {
    return res.status(403).send("Forbidden: invalid staff key");
  }

  // No caching – we want fresh versions
  res.setHeader("Cache-Control", "no-store, no-cache, must-revalidate, proxy-revalidate");
  res.setHeader("Pragma", "no-cache");
  res.setHeader("Surrogate-Control", "no-store");

  res.send(`<!DOCTYPE html>
    <html>
    <head>
      <meta charset="UTF-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1" />
      <title>AURA Ticket Generator</title>
      <style>
        ${themeCSSRoot()}

        body {
          margin: 0;
          padding: 20px;
          min-height: 100vh;
          font-family: system-ui, -apple-system, BlinkMacSystemFont, sans-serif;
          background:
            radial-gradient(circle at top left, rgba(255,64,129,0.25), transparent 55%),
            radial-gradient(circle at bottom right, rgba(255,23,68,0.35), transparent 55%),
            var(--bg-dark);
          color: var(--text-main);
        }

        .card {
          max-width: 520px;
          margin: 0 auto;
          background: radial-gradient(circle at top, #220018, #070008 60%);
          border-radius: 20px;
          padding: 20px 18px;
          box-shadow:
            0 0 0 1px rgba(255,64,129,0.35),
            0 20px 50px rgba(0,0,0,0.95);
        }

        h1 {
          margin: 0 0 8px;
          font-size: 1.4rem;
        }

        .subtitle {
          font-size: 0.9rem;
          color: var(--text-muted);
          margin-bottom: 18px;
        }

        .field {
          margin-bottom: 12px;
        }

        label {
          display:block;
          font-size:0.85rem;
          margin-bottom:4px;
          text-transform:uppercase;
          letter-spacing:0.08em;
          color:var(--text-muted);
        }

        select,
        input[type="text"],
        input[type="number"] {
          width:100%;
          padding:9px 11px;
          border-radius:999px;
          border:1px solid rgba(255,64,129,0.6);
          background:rgba(3,0,5,0.9);
          color:#fff;
          font-size:0.95rem;
          outline:none;
          box-sizing:border-box;
        }

        select:focus,
        input[type="text"]:focus,
        input[type="number"]:focus {
          box-shadow:0 0 0 1px rgba(255,183,77,0.9),
                     0 0 16px rgba(255,64,129,0.6);
        }

        .split-row {
          display:flex;
          gap:10px;
        }

        .split-row .field {
          flex:1;
          margin-bottom:0;
        }

        .btn-main {
          width: 100%;
          padding: 13px 16px;
          margin-top: 16px;
          border-radius: 999px;
          border: none;
          cursor: pointer;
          font-weight: 700;
          font-size: 0.95rem;
          text-transform: uppercase;
          letter-spacing: 0.09em;
          background: linear-gradient(135deg,#ff4b9a,#ffb347);
          color: #050005;
          box-shadow: 0 10px 24px rgba(0,0,0,0.8);
        }

        .btn-main:hover { filter:brightness(1.06); }

        .back-link {
          display:inline-block;
          margin-top:14px;
          font-size:0.85rem;
          color:var(--accent-gold);
          text-decoration:none;
        }

        .back-link:hover { text-decoration: underline; }
      </style>
    </head>
    <body>
      <div class="card">
        <h1>AURA Ticket Generator</h1>
        <div class="subtitle">
          Choose a ticket type and enter how many tickets / QR codes you want.
          All tickets still use the normal generator and appear in the dashboard.
        </div>

        <div class="field">
          <label for="ticketType">Ticket type</label>
          <select id="ticketType">
            <option value="earlybird" data-prefix="EB-">Early Bird</option>
            <option value="general"   data-prefix="GEN-">General</option>
            <option value="comp"      data-prefix="COMP-">Comp</option>
            <option value="test"      data-prefix="TEST-">Test</option>
          </select>
        </div>

        <div class="field">
          <label for="prefixInput">Ticket prefix</label>
          <input id="prefixInput" type="text" value="EB-" />
        </div>

        <div class="split-row">
          <div class="field">
            <label for="startInput">Start number</label>
            <input id="startInput" type="number" min="1" value="1" />
          </div>
          <div class="field">
            <label for="countInput">How many tickets?</label>
            <input id="countInput" type="number" min="1" value="10" />
          </div>
        </div>

        <button class="btn-main" onclick="generateCustomBatch()">
          Generate Tickets / QR Codes
        </button>

    <a class="back-link" href="/management-hub">
  ← Back to Management Hub
</a>

<a href="/cancelled-tickets-log?key=${encodeURIComponent(key)}"
   class="tool-card tool-red">
  <div class="tool-title">❌ Cancelled Tickets</div>
  <div class="tool-sub">View all cancelled tickets by type & link back to allocations.</div>
</a>

      </div>

      <script>
        const STAFF_PIN = ${JSON.stringify(STAFF_PIN)};

        // Update prefix when ticket type changes
        const typeSelect   = document.getElementById('ticketType');
        const prefixInput  = document.getElementById('prefixInput');

        typeSelect.addEventListener('change', () => {
          const opt = typeSelect.selectedOptions[0];
          if (opt && opt.dataset.prefix) {
            prefixInput.value = opt.dataset.prefix;
          }
        });

        function generateCustomBatch() {
          const type   = typeSelect.value || 'general';
          const prefix = (prefixInput.value || '').trim() || 'GEN-';

          const startVal = parseInt(document.getElementById('startInput').value, 10);
          const countVal = parseInt(document.getElementById('countInput').value, 10);

          const start = Number.isFinite(startVal) && startVal > 0 ? startVal : 1;
          const count = Number.isFinite(countVal) && countVal > 0 ? countVal : 1;

          if (!confirm("Generate " + count + " " + type.toUpperCase()
                       + " tickets starting at " + prefix + String(start).padStart(3,"0") + "?")) {
            return;
          }

          const url =
            "/generate-batch"
            + "?key="    + encodeURIComponent(STAFF_PIN)
            + "&type="   + encodeURIComponent(type)
            + "&prefix=" + encodeURIComponent(prefix)
            + "&start="  + encodeURIComponent(start)
            + "&count="  + encodeURIComponent(count);

          window.location.href = url;
        }
      </script>
    </body>
    </html>`);
});


// ------------------------------------------------------
// STAFF QR LIST PAGE – view & download generated QR PNGs
// ------------------------------------------------------
app.get("/staff/qr-list", (req, res) => {
  const { key } = req.query;

  // ✅ allow staff OR management
  if (!(key === STAFF_PIN || key === MANAGEMENT_PIN)) {
    return res.status(403).send("Forbidden: invalid staff key");
  }

  const qrFolder = QR_DIR;
  let files = [];

  try {
    if (fs.existsSync(qrFolder)) {
      files = fs
        .readdirSync(qrFolder)
        .filter((f) => f.toLowerCase().endsWith(".png"))
        .sort();
    }
  } catch (err) {
    console.error("Error reading QR folder:", err);
  }

  const liveFiles = files.filter((f) => !f.toUpperCase().startsWith("TEST-"));
  const testFiles = files.filter((f) => f.toUpperCase().startsWith("TEST-"));

  const renderList = (arr) =>
    arr
      .map(
        (f) => `
          <li>
            <a class="qr-link"
               href="/generated_qr/${encodeURIComponent(f)}"
               target="_blank">
              <span>📁</span>
              <span>${f}</span>
            </a>
          </li>`
      )
      .join("");

  const qrSectionsHtml =
    files.length === 0
      ? '<p class="empty"><em>No QR PNG files found in generated_qr/ yet.</em></p>'
      : `
        ${liveFiles.length
          ? `<h2 class="section-title">Live Tickets</h2><ul>${renderList(
              liveFiles
            )}</ul>`
          : ""
        }

        ${testFiles.length
          ? `<h2 class="section-title">Test Tickets</h2><ul>${renderList(
              testFiles
            )}</ul>`
          : ""
        }
      `;

  res.send(`<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>AURA QR Files</title>
  <style>
    ${themeCSSRoot()}

    body {
      margin:0;
      min-height:100vh;
      display:flex;
      align-items:center;
      justify-content:center;
      font-family:system-ui,-apple-system,BlinkMacSystemFont,"Segoe UI",sans-serif;
      background:
        radial-gradient(circle at top left, rgba(255,64,129,0.18), transparent 55%),
        radial-gradient(circle at bottom right, rgba(255,193,7,0.10), transparent 55%),
        var(--bg-dark);
      color:var(--text-main);
    }

    .container {
      width:100%;
      max-width:720px;
      padding:24px;
    }

    .card {
      background:radial-gradient(circle at top,#220018,#050008 60%);
      border-radius:24px;
      padding:24px 26px;
      border:1px solid rgba(255,64,129,0.35);
      box-shadow:0 18px 45px rgba(0,0,0,0.85);
    }

    h1 {
      margin:0 0 6px;
      font-size:1.4rem;
      letter-spacing:0.18em;
      text-transform:uppercase;
      color:var(--accent-gold);
    }

    .subtitle {
      font-size:0.85rem;
      color:var(--text-muted);
      margin-bottom:12px;
    }

    /* 🔹 New top actions row + pill button */
    .top-actions {
      display:flex;
      justify-content:space-between;
      align-items:center;
      gap:8px;
      margin-bottom:16px;
      flex-wrap:wrap;
    }

    .pill-btn {
      display:inline-flex;
      align-items:center;
      justify-content:center;
      border-radius:999px;
      padding:8px 14px;
      border:none;
      font-size:0.78rem;
      letter-spacing:0.12em;
      text-transform:uppercase;
      cursor:pointer;
      background:linear-gradient(120deg,#ffb300,#ff4081);
      color:#050007;
      font-weight:600;
    }
    .pill-btn:active {
      transform:translateY(1px);
    }

    .section-title {
      margin:18px 0 8px;
      font-size:0.9rem;
      text-transform:uppercase;
      letter-spacing:0.16em;
      color:#f3c1ff;
    }

    ul {
      list-style:none;
      margin:0;
      padding:0;
    }

    li + li {
      margin-top:6px;
    }

    .qr-link {
      display:flex;
      align-items:center;
      gap:8px;
      padding:10px 12px;
      border-radius:12px;
      text-decoration:none;
      color:var(--text-main);
      background:rgba(0,0,0,0.35);
      border:1px solid rgba(255,255,255,0.08);
    }

    .qr-link span:first-child {
      font-size:1rem;
    }

    .qr-link:hover {
      border-color:rgba(255,183,77,0.8);
      box-shadow:0 0 0 1px rgba(255,183,77,0.6);
    }

    .empty {
      text-align:center;
      color:var(--text-muted);
      font-size:0.9rem;
    }

    .back-link {
      display:inline-block;
      margin-top:18px;
      font-size:0.85rem;
      text-decoration:none;
      color:var(--accent-gold);
      letter-spacing:0.12em;
      text-transform:uppercase;
    }
  </style>
</head>
<body>
  <div class="container">
    <div class="card">
      <h1>AURA QR Files</h1>
      <div class="subtitle">
        Click a filename to open the QR image in a new tab, then save it and drop it into your ticket artwork.
      </div>

      <!-- 🔹 New button row -->
      <div class="top-actions">
        <div style="font-size:0.78rem;color:var(--text-muted);">
          View and manage all generated QR PNGs in <code>generated_qr/</code>.
        </div>
        <button class="pill-btn" onclick="clearQrPngs()">🧹 Clear QR PNGs</button>
      </div>

      ${qrSectionsHtml}
      <a href="/management-hub?key=${encodeURIComponent(MANAGEMENT_PIN)}" class="back-link">
        ← Back to Management Hub
      </a>
    </div>
  </div>

  <script>
    const params = new URLSearchParams(location.search);
    const MGMT_KEY = params.get("key") || "";

    async function clearQrPngs() {
      if (!confirm("Delete ALL generated QR PNG files from QR FILES? Tickets remain valid, only the images are removed.")) {
        return;
      }

      try {
        const res = await fetch(
          "/admin/clear-qr-files?key=" + encodeURIComponent(MGMT_KEY),
          { method: "POST" }
        );
        const data = await res.json().catch(() => ({}));
        if (!data.ok) {
          alert(data.error || "Unable to clear QR files");
          return;
        }
        alert("QR PNG files cleared.");
        location.reload();
      } catch (err) {
        alert("Error clearing QR files: " + err.message);
      }
    }
  </script>
</body>
</html>`);
});


// ROUTE 2: Generate a single ticket & QR
app.get("/generate-ticket", (req, res) => {
  const ticketId = req.query.id || "AURA-TEST-001";
  const ticketType = (req.query.type || "general").toLowerCase(); // general / earlybird / comp

  const token = createToken();
  const signature = signToken(token);

  tickets.set(token, { id: ticketId, type: ticketType, status: "unused" });
  saveTickets();

  const baseUrl = getBaseUrl(req);
  const ticketUrl = `${baseUrl}/ticket?token=${token}&sig=${signature}`;
  const qrImagePath = path.join(QR_DIR, `${ticketId}.png`);

  QRCode.toFile(qrImagePath, ticketUrl)
    .then(() => {
      res.send(`
        <h1>Ticket generated</h1>
        <p><strong>Ticket ID:</strong> ${ticketId}</p>
        <p><strong>Type:</strong> ${ticketType}</p>
        <p><strong>QR file saved at:</strong> ${qrImagePath}</p>
        <p><strong>URL encoded in QR:</strong> ${ticketUrl}</p>
        <p>You can now print this QR image on your physical ticket.</p>
      `);
    })
    .catch((err) => {
      console.error("Error generating QR:", err);
      res.status(500).send("Error generating QR");
    });
});
// ROUTE 3: Generate a batch of tickets (STAFF-ONLY)
app.get("/generate-batch", (req, res) => {
  const adminKey = req.query.key;

  // if you want mgmt to work too, change next line to:
  // if (!(adminKey === STAFF_PIN || adminKey === MANAGEMENT_PIN)) { ... }
  if (adminKey !== STAFF_PIN) {
    return res.status(403).send("Forbidden: invalid staff key");
  }

  const ticketType = (req.query.type || "general").toLowerCase();
  const prefix = req.query.prefix || "GEN-";
  const start = parseInt(req.query.start || "1", 10);
  const count = parseInt(req.query.count || "10", 10);

  const created = [];
  const tasks = [];

  for (let i = 0; i < count; i++) {
    const num = start + i;
    const id = `${prefix}${String(num).padStart(3, "0")}`; // GEN-001 style

    const token = createToken();
    const signature = signToken(token);
    tickets.set(token, { id, type: ticketType, status: "unused" });

    const baseUrl = getBaseUrl(req);
    const ticketUrl = `${baseUrl}/ticket?token=${token}&sig=${signature}`;
    const qrImagePath = path.join(QR_DIR, `${id}.png`);

    // QRCode.toFile returns a Promise when no callback is passed
    tasks.push(QRCode.toFile(qrImagePath, ticketUrl));

    created.push({ id, type: ticketType, qr: qrImagePath });
  }

  Promise.all(tasks)
    .then(() => {
      saveTickets();

      res.send(`
        <h1>Batch generated</h1>
        <p><strong>Type:</strong> ${ticketType}</p>
        <p><strong>Count:</strong> ${created.length}</p>
        <ul>
          ${created.map((t) => `<li>${t.id} → QR: ${t.qr}</li>`).join("")}
        </ul>
        <p><a href="/staff/generate?key=${encodeURIComponent(STAFF_PIN)}">
          ← Back to Ticket Generator
        </a></p>
      `);
    })
    .catch((err) => {
      console.error("Error generating batch QRs:", err);
      res.status(500).send("Error generating batch QRs");
    });
});

// ------------------------------------------------------
// ROUTE 4: When someone scans the QR
// ------------------------------------------------------
app.get("/ticket", (req, res) => {
  let { token, staff, key, sig, id } = req.query;
  // Fallback: if a ticket `id` was provided instead of a token, try to find its token
  if (!token && id) {
    for (const [tk, rec] of tickets.entries()) {
      if (rec && rec.id === id) {
        token = tk;
        break;
      }
    }
  }
  const clientIP = getClientIP(req);

  // No token at all → just send to Instagram
  if (!token) {
    return res.redirect(INSTAGRAM_URL);
  }

  // Verify signature if provided (anti-forgery check)
  if (sig && !verifyToken(token, sig)) {
    // Signature mismatch - this is a forged or tampered QR
    const timestamp = new Date().toISOString();
    scanEvents.invalid.push({ token, timestamp, staff: "forged", reason: "signature_mismatch" });

    ipLogging.events.push({
      ip: clientIP,
      token,
      ticketId: "FORGED",
      timestamp: Date.now(),
      status: "invalid"
    });

    console.warn(
      `[SECURITY] Forged QR detected from ${clientIP}: token=${token.substring(0, 8)}...`
    );
    return res.redirect(INSTAGRAM_URL);
  }

  // Unknown / invalid token → track it and bounce to Instagram
  if (!tickets.has(token)) {
    const timestamp = new Date().toISOString();
    scanEvents.invalid.push({ token, timestamp, staff: staff === "1" ? "yes" : "no" });

    ipLogging.events.push({
      ip: clientIP,
      token,
      ticketId: "INVALID",
      timestamp: Date.now(),
      status: "invalid"
    });

    if (checkSuspiciousActivity(clientIP)) {
      if (!ipLogging.suspicious.has(clientIP)) {
        ipLogging.suspicious.set(clientIP, []);
      }
    }

    return res.redirect(INSTAGRAM_URL);
  }

  const record = tickets.get(token);

    // 🔥 If this ticket was cancelled, treat guest scans as invalid and bounce
  const isCancelled = record.status === "cancelled";

  if (isCancelled && staff !== "1") {
    const timestamp = new Date().toISOString();
    scanEvents.invalid.push({
      token,
      ticketId: record.id,
      timestamp,
      reason: "cancelled"
    });

    ipLogging.events.push({
      ip: clientIP,
      token,
      ticketId: record.id,
      timestamp: Date.now(),
      status: "cancelled"
    });

    return res.redirect(INSTAGRAM_URL);
  }

  // STAFF-SIDE logging for duplicates/valid
  if (record.status === "used" && staff === "1") {
    const timestamp = new Date().toISOString();
    scanEvents.duplicates.push({ token, ticketId: record.id, timestamp });

    ipLogging.events.push({
      ip: clientIP,
      token,
      ticketId: record.id,
      timestamp: Date.now(),
      status: "duplicate"
    });
  } else if (staff === "1") {
    ipLogging.events.push({
      ip: clientIP,
      token,
      ticketId: record.id,
      timestamp: Date.now(),
      status: "valid"
    });
  }

// ----------------------------------------------
// GUEST SCAN LOGIC
// ----------------------------------------------
if (staff !== "1") {
    // Log guest scan (same as before)
    const scanEntry = {
        ticketId: record.id,
        token,
        ip: clientIP,
        timestamp: new Date().toISOString()
    };
    pushWithLimit(guestScanLog, scanEntry, 1000);

    // If ticket was allocated to a seller, mark it as SOLD in allocations ONLY
    if (ticketAllocations.has(record.id)) {
        const alloc = ticketAllocations.get(record.id);
        if (!alloc.sold) {
            alloc.sold = true;
            ticketAllocations.set(record.id, alloc);
            saveTicketAllocations();
        }
    }

    // DO NOT TOUCH ticket.status here
}

  // ========= STAFF VIEW (must have staff=1 + correct key) =========
  if (staff === "1") {
    if (key !== STAFF_PIN) {
      return res.status(403).send("Invalid or missing staff key.");
    }

    const isUsed = record.status === "used";
    const isCancelledStaff = record.status === "cancelled";

    let statusText, statusColor, statusBg;

    if (isCancelledStaff) {
      statusText = "CANCELLED";
      statusColor = "#ff9800";
      statusBg = "rgba(255,152,0,0.18)";
    } else if (isUsed) {
      statusText = "USED";
      statusColor = "#ff3b30";
      statusBg = "rgba(255,59,48,0.1)";
    } else {
      statusText = "VALID";
      statusColor = "#34c759";
      statusBg = "rgba(52,199,89,0.12)";
    }


    return res.send(`<!DOCTYPE html>
      <html>
      <head>
        <meta charset="UTF-8" />
        <title>AURA Ticket Check-In</title>
        <style>
          body {
            font-family: system-ui, -apple-system, BlinkMacSystemFont, sans-serif;
            padding: 20px;
            background:#000;
            color:#eee;
          }
          .card {
            max-width: 480px;
            margin:auto;
            background:#111;
            padding:24px;
            border-radius:16px;
            box-shadow:0 10px 30px rgba(0,0,0,0.7);
          }
          .pill {
            display:inline-block;
            padding:8px 16px;
            border-radius:999px;
            font-weight:700;
            letter-spacing:0.08em;
            font-size:0.85rem;
            text-transform:uppercase;
            background:${statusBg};
            color:${statusColor};
          }
          h1 { margin-top:16px; margin-bottom:8px; }
          .meta { margin-top:12px; font-size:0.95rem; }
          .meta p { margin:4px 0; }
          a.button {
            display:inline-block;
            margin-top:16px;
            padding:10px 16px;
            border-radius:999px;
            text-decoration:none;
            font-weight:600;
            background:#fff;
            color:#000;
            cursor:pointer;
            border:none;
          }
          a.button:hover, button.button:hover { filter:brightness(0.9); }

          .toast {
            position: fixed;
            top: 20px;
            right: 20px;
            background: rgba(0,0,0,0.85);
            color: #fff;
            padding: 10px 16px;
            border-radius: 999px;
            font-size: 0.85rem;
            display: none;
          }
        </style>
      </head>
      <body>
        <div class="card">
          <span class="pill">${statusText}</span>
          <h1>Ticket ${record.id}</h1>
          <div class="meta">
            <p><strong>Type:</strong> ${record.type}</p>
            <p><strong>Status:</strong> ${record.status}</p>
          </div>
          ${
            !isUsed
              ? `<button class="button" onclick="markAsUsed('${encodeURIComponent(token)}')">Mark as Used</button>`
              : ""
          }
          <a class="button" href="/staff?key=${encodeURIComponent(
            STAFF_PIN
          )}">Back to Check-In</a>
        </div>
        <div id="toast" class="toast"></div>
        <script>
          async function markAsUsed(token) {
            try {
              const res = await fetch('/api/mark-ticket-used', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ token, key: ${JSON.stringify(STAFF_PIN)} })
              });
              const data = await res.json();
              if (data.ok) {
                // Refresh the page to show updated status
                window.location.reload();
              } else {
                alert('Error: ' + (data.error || 'Failed to mark ticket as used'));
              }
            } catch (e) {
              alert('Error marking ticket: ' + e.message);
            }
          }
        </script>
      </body>
      </html>`);
  }

  // ========= GUEST VIEW =========
  // Already-checked-in guests still go to Instagram
  if (record.status === "used") {
    return res.redirect(INSTAGRAM_URL);
  }

  // ========= GUEST VIEW =========
  // Show welcome page with audio, then redirect to Instagram after 25s
  const ticketStatus = record.status === "used" ? "USED (STAFF)" : "VALID";
  const statusEmoji = record.status === "used" ? "⚠️" : "✅";
  const statusColor = record.status === "used" ? "#ff9500" : "#34c759";

  return res.send(`
  <!DOCTYPE html>
  <html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
    <title>Welcome to A.U.R.A</title>
    <style>
      :root {
        --pink: #ff4081;
        --red: #ff1744;
        --gold: #ffb300;
        --bg: #050007;
      }
      * { box-sizing: border-box; }
      body {
        margin: 0;
        padding: 0;
        font-family: system-ui, -apple-system, BlinkMacSystemFont, sans-serif;
        background: radial-gradient(circle at top left, rgba(255,64,129,0.25), transparent 55%),
                    radial-gradient(circle at bottom right, rgba(255,23,68,0.35), transparent 55%),
                    var(--bg);
        color: #fff;
        display: flex;
        justify-content: center;
        align-items: center;
        height: 100vh;
        text-align: center;
      }
      .card {
        background: rgba(10,0,14,0.75);
        backdrop-filter: blur(12px);
        padding: 28px 24px;
        max-width: 360px;
        border-radius: 20px;
        box-shadow: 0 0 25px rgba(255,64,129,0.3);
        animation: fadeIn 1.2s ease-out;
        margin: 0 auto;
      }
      .status-alert {
        background: linear-gradient(135deg, ${statusColor}30, ${statusColor}10);
        border: 2px solid ${statusColor};
        padding: 16px 12px;
        border-radius: 12px;
        margin-bottom: 16px;
        font-weight: 700;
        font-size: 0.95rem;
        color: ${statusColor};
        letter-spacing: 0.05em;
      }
      h1 {
        font-size: 1.6rem;
        background: linear-gradient(120deg, var(--gold), var(--pink), var(--red));
        -webkit-background-clip: text;
        color: transparent;
        margin-bottom: 10px;
      }
      p {
        font-size: 1rem;
        color: #eee;
        line-height: 1.5rem;
        margin-bottom: 16px;
      }
      .heart {
        font-size: 2.2rem;
        animation: pulse 1.4s infinite;
      }
      @keyframes pulse {
        0% { transform: scale(1); opacity: 0.8; }
        50% { transform: scale(1.3); opacity: 1; }
        100% { transform: scale(1); opacity: 0.8; }
      }
      @keyframes fadeIn {
        from { opacity: 0; transform: translateY(20px); }
        to   { opacity: 1; transform: translateY(0); }
      }
      .note {
        margin-top: 14px;
        font-size: 0.8rem;
        color: #bbb;
      }
      .prize-section {
        margin-top: 20px;
        padding-top: 16px;
        border-top: 1px solid rgba(255,64,129,0.3);
      }
      .prize-title {
        font-size: 0.95rem;
        font-weight: 700;
        color: #ffb300;
        margin-bottom: 10px;
      }
      #guestNameInput,
      #guestEmailInput,
      #guestPhoneInput {
        width: 100%;
        padding: 12px;
        border-radius: 8px;
        border: 1px solid rgba(255,64,129,0.5);
        background: rgba(0,0,0,0.3);
        color: #fff;
        font-size: 0.95rem;
        outline: none;
        margin-bottom: 6px;
      }
      #guestNameInput:focus,
      #guestEmailInput:focus,
      #guestPhoneInput:focus {
        border-color: #ffb300;
        box-shadow: 0 0 12px rgba(255,179,0,0.4);
      }
      #submitNameBtn {
        margin-top: 8px;
        width: 100%;
        padding: 10px;
        border-radius: 8px;
        border: none;
        background: linear-gradient(135deg, #ffb300, #ff9800);
        color: #000;
        font-weight: 700;
        font-size: 0.9rem;
        cursor: pointer;
        text-transform: uppercase;
        letter-spacing: 0.08em;
      }
      #submitNameBtn:hover {
        filter: brightness(1.1);
      }
      #submitNameBtn:disabled {
        opacity: 0.5;
        cursor: not-allowed;
      }
      .entry-success {
        display: none;
        margin-top: 8px;
        padding: 10px;
        background: rgba(76,175,80,0.2);
        border: 1px solid #4caf50;
        border-radius: 8px;
        color: #4caf50;
        font-size: 0.9rem;
        text-align: center;
      }
    </style>
  </head>
  <body>
    <div class="card">
      <div class="heart">❤️ ♠️</div>
      <div class="status-alert">${statusEmoji} TICKET ${ticketStatus}</div>
      <h1>Welcome to ✨ A.U.R.A By PoP ✨</h1>
      <div class="content">
        <p>🎉 You're officially on deck! A night of euphoria awaits as Hearts rise and Spades take control.</p>
        <p>You are now part of A.U.R.A — Alluring. Unforgettable. Romantic. Affair.</p>
        <p>Thank you for choosing to spend your night with us. Your presence adds to the magic — and we can't wait to make this moment unforgettable.</p>
        <p style="font-weight:700; margin-top:6px;">Feb 13 — See You There!</p>
      </div>

<div class="prize-section">
  <div class="prize-title">🎁 Mystery Prize Entry</div>
  <p style="font-size:0.85rem; color:#ccc; margin:0 0 8px;">
    Enter your details for a chance to win an exclusive mystery prize! (One entry per ticket)
  </p>
  <input type="text" id="guestNameInput" placeholder="Enter your name" maxlength="50" />
  <input type="email" id="guestEmailInput" placeholder="Enter your email" maxlength="80" />
  <input type="tel" id="guestPhoneInput" placeholder="Enter your cell number" maxlength="20" />
  <button id="submitNameBtn">Submit for Prize Draw</button>
  <div class="field checkbox-field">
  <label style="display:flex;align-items:flex-start;gap:8px;font-size:0.8rem;line-height:1.3;">
    <input type="checkbox"
           id="subscribeOptIn"
           style="margin-top:3px;">
    <span>
      I’d like to join the A.U.R.A / POP mailing list and receive updates
      about future events and special offers.
    </span>
  </label>
</div>

  <div class="entry-success" id="successMsg" style="display:none;">
    ✅ You're entered! Good luck!
  </div>

  <!-- Small Instagram button under the success message -->
  <button
    id="visitIgBtn"
    type="button"
    style="
      margin-top:12px;
      padding:8px 14px;
      border-radius:6px;
      background:#ff2e6a;
      color:white;
      font-size:0.85rem;
      border:none;
      cursor:pointer;
    "
  >
    Visit Our Instagram ❤️🖤
  </button>
</div>


<script>
      const ticketToken = "${token}";
      const ticketId = "${record.id}";
      const IG_URL = "${INSTAGRAM_URL}";

      const audio = document.createElement('audio');
      audio.id = 'bgAudio';
      audio.src = '/aura-welcome.mp3';
      audio.preload = 'auto';
      audio.loop = false;
      audio.autoplay = true;
      audio.playsInline = true;
      audio.setAttribute('playsinline', '');
      audio.setAttribute('webkit-playsinline', '');
      audio.volume = 0.9;
      document.body.appendChild(audio);

      const overlay = document.createElement('div');
      overlay.id = 'playOverlay';
      overlay.style = 'position:fixed; inset:0; display:flex; align-items:center; justify-content:center; z-index:9999; pointer-events:auto; background: rgba(0,0,0,0.18);';

      const playBtn = document.createElement('button');
      playBtn.textContent = 'Tap to play audio';
      playBtn.style = 'pointer-events:auto; padding:12px 18px; border-radius:12px; border:none; background:linear-gradient(120deg,#ff4081,#ff1744); color:#fff; font-weight:700; font-size:1rem; box-shadow:0 8px 20px rgba(0,0,0,0.6);';
      playBtn.addEventListener('click', async () => {
        try {
          audio.muted = false;
          await audio.play();
          playBtn.style.display = 'none';
          overlay.style.display = 'none';
        } catch (err) { console.warn('Playback failed on user play:', err); }
      });

      overlay.appendChild(playBtn);
      document.body.appendChild(overlay);

      audio.muted = true;
      audio.play().then(() => {
        audio.muted = false;
        audio.play().catch(() => {});
      }).catch(() => {});

      // PRIZE ENTRY SUBMISSION
      const nameInput   = document.getElementById('guestNameInput');
      const emailInput  = document.getElementById('guestEmailInput');
      const phoneInput  = document.getElementById('guestPhoneInput');
      const submitBtn   = document.getElementById('submitNameBtn');
      const successMsg  = document.getElementById('successMsg');
      const visitIgBtn  = document.getElementById('visitIgBtn');

      // Click on the IG button at any time
      visitIgBtn.addEventListener('click', () => {
        window.location.href = IG_URL;
      });

      submitBtn.addEventListener('click', async () => {
  const guestName  = nameInput.value.trim();
  const guestEmail = emailInput.value.trim();
  const guestPhone = phoneInput.value.trim();
  const subscribeOptIn = !!document.getElementById('subscribeOptIn')?.checked;  // 👈 NEW

  if (!guestName) {
    alert('Please enter your name');
    return;
  }
  if (!guestEmail || !guestPhone) {
    alert('Please enter your email and cell number');
    return;
  }

  try {
    const response = await fetch('/api/guest-name-entry', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        ticketId: ticketId,
        token: ticketToken,
        guestName,
        guestEmail,
        guestPhone,
        subscribe: subscribeOptIn                               // 👈 NEW
      })
    });

    const data = await response.json();
    ...


          if (data.success) {
            // Lock the fields + show success
            nameInput.disabled   = true;
            emailInput.disabled  = true;
            phoneInput.disabled  = true;
            submitBtn.disabled   = true;
            successMsg.style.display = 'block';

            // Redirect straight to Instagram
            window.location.href = IG_URL;
            return;
          } else {
            alert(data.error || 'Error submitting entry');
          }
        } catch (err) {
          console.error('Error:', err);
          alert('Failed to submit entry');
        }
      });
    </script>
  </body>
  </html>
  `);
});

// NEW ENDPOINT: Guest Name Entry for Prize Draw
app.post("/api/guest-name-entry", (req, res) => {
  const {
    ticketId,
    token,
    guestName,
    guestEmail,
    guestPhone,
    subscribe          // 👈 NEW
  } = req.body || {};

  const clientIP = getClientIP(req);

  if (!token || !ticketId || !guestName || !guestEmail || !guestPhone) {
    return res.status(400).json({ error: "Missing required fields" });
  }

  // Check if this token already has an entry
  const existingEntry = guestNameEntries.find(e => e.token === token);
  if (existingEntry) {
    return res.status(400).json({ error: "Only one entry per ticket allowed" });
  }

  // Verify ticket exists
  if (!tickets.has(token)) {
    return res.status(404).json({ error: "Invalid ticket" });
  }

  // Add entry to guest name entries log
  pushWithLimit(guestNameEntries, {
    ticketId,
    token,
    guestName,
    guestEmail,
    guestPhone,
    subscribe: !!subscribe,          // 👈 NEW: true/false flag
    ip: clientIP,
    timestamp: new Date().toISOString()
  }, 2000);

  return res.json({ success: true, message: "Entry received! Good luck!" });
});


// NEW ENDPOINT: Get guest name entries (Management only)
app.get("/api/guest-name-entries", (req, res) => {
  const { key } = req.query;
  if (!(key === STAFF_PIN || isMgmtAuthorizedReq(req))) {
    return res.json({ error: "Unauthorized" });
  }

  res.json(guestNameEntries.slice(-500));
});

// NEW PAGE: Guest Name Entries / Prize Draw Management
app.get("/guest-prize-entries", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.redirect("/staff?key=" + encodeURIComponent(STAFF_PIN));
  }

  res.send(`<!DOCTYPE html>
    <html>
    <head>
      <meta charset="UTF-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1" />
      <title>Guest Prize Entries</title>
      <style>
        ${themeCSSRoot()}
        body {
          margin:0;
          padding:18px;
          font-family:system-ui,-apple-system,BlinkMacSystemFont,sans-serif;
          background:#050007;
          color:#f5f5f5;
          display:flex;
          justify-content:center;
          min-height:100vh;
        }
        .card {
          width:100%;
          max-width:1000px;
          background:radial-gradient(circle at top,#220018,#070008 60%);
          border-radius:16px;
          padding:18px;
          border:1px solid rgba(255,64,129,0.25);
          box-shadow:0 10px 28px rgba(0,0,0,0.7);
        }
        h1 {
          margin:0 0 8px;
          font-size:1.6rem;
          background:linear-gradient(120deg,#ffb300,#ff4081,#ff1744);
          -webkit-background-clip:text;
          color:transparent;
        }
        .subtitle {
          font-size:0.9rem;
          color:#aaa;
          margin-bottom:14px;
        }
        .stats-row {
          display:flex;
          gap:12px;
          margin-bottom:14px;
          flex-wrap:wrap;
        }
        .stat-box {
          flex:1;
          min-width:140px;
          padding:12px;
          background:rgba(0,0,0,0.3);
          border-radius:8px;
          border-left:3px solid #ffb300;
        }
        .stat-label {
          font-size:0.8rem;
          color:#aaa;
          text-transform:uppercase;
          letter-spacing:0.05em;
        }
        .stat-value {
          font-size:1.8rem;
          font-weight:700;
          color:#ffb300;
        }
        table {
          width:100%;
          border-collapse:collapse;
          font-size:0.85rem;
        }
        th, td {
          padding:8px;
          border-bottom:1px solid rgba(255,255,255,0.08);
        }
        th {
          text-align:left;
          font-size:0.78rem;
          text-transform:uppercase;
          letter-spacing:0.08em;
          color:#bbb;
        }
        .draw-btn {
          display:inline-block;
          margin-top:14px;
          padding:10px 16px;
          border-radius:999px;
          background:linear-gradient(135deg,#ff1744,#ff4081);
          color:#fff;
          border:none;
          cursor:pointer;
          font-weight:700;
          text-transform:uppercase;
          letter-spacing:0.08em;
          font-size:0.85rem;
        }
        .draw-btn:hover { filter:brightness(1.1); }
        .back-btn {
          display:inline-block;
          margin-top:12px;
          margin-left:8px;
          padding:8px 16px;
          border-radius:999px;
          border:1px solid rgba(255,255,255,0.2);
          text-decoration:none;
          color:#fff;
          font-size:0.85rem;
          letter-spacing:0.08em;
          text-transform:uppercase;
        }
        .empty {
          text-align:center;
          color:#888;
        }
        #winnerDisplay {
          display:none;
          margin-top:14px;
          padding:14px;
          background:rgba(76,175,80,0.15);
          border:2px solid #4caf50;
          border-radius:8px;
          color:#4caf50;
          text-align:center;
          font-weight:700;
        }
      </style>
    </head>
    <body>
      <div class="card">
        <h1>🎁 Guest Prize Draw Entries</h1>
        <div class="subtitle">Guests who entered the mystery prize draw. Select a random winner!</div>

        <div class="stats-row">
          <div class="stat-box">
            <div class="stat-label">Total Entries</div>
            <div class="stat-value" id="totalEntries">0</div>
          </div>
          <div class="stat-box">
            <div class="stat-label">Unique Tickets</div>
            <div class="stat-value" id="uniqueTickets">0</div>
          </div>
        </div>

        <div id="winnerDisplay"></div>

        <button class="draw-btn" onclick="drawWinner()">🎲 Draw Random Winner</button>

        <div style="margin-top:14px;">
          <table>
            <thead>
              <tr>
                <th>Guest Name</th>
                <th>Email</th>
                <th>Cell</th>
                <th>Ticket ID</th>
                <th>Token (short)</th>
                <th>IP</th>
                <th>Submitted</th>
              </tr>
            </thead>
            <tbody id="entriesBody"></tbody>
          </table>
        </div>

        <a href="/management-hub?key=${encodeURIComponent(MANAGEMENT_PIN)}" class="back-btn">← Back to Management Hub</a>
      </div>

            ${themeScript()}
      <script>
        // Grab the management key from the URL (?key=...)
        const params   = new URLSearchParams(window.location.search);
        const MGMT_KEY = params.get("key") || "";

        function formatDate(iso) {
          if (!iso) return "--";
          try {
            const d = new Date(iso);
            return d.toLocaleDateString() + " " + d.toLocaleTimeString();
          } catch (e) {
            return iso;
          }
        }

        function loadAllocations() {
          fetch('/api/allocations-detail?key=' + encodeURIComponent(MGMT_KEY))
            .then(function(r) { return r.json(); })
            .then(function(data) {
              var body = document.getElementById('allocBody');
              body.innerHTML = '';

              if (!data.ok || !data.allocations || data.allocations.length === 0) {
                body.innerHTML =
                  '<tr><td colspan="6" style="text-align:center;color:#888;">No allocations yet.</td></tr>';
                document.getElementById('statTotal').textContent = '0';
                document.getElementById('statSold').textContent  = '0';
                document.getElementById('statUnsold').textContent = '0';
                return;
              }

              document.getElementById('statTotal').textContent  = data.total;
              document.getElementById('statSold').textContent   = data.sold;
              document.getElementById('statUnsold').textContent = data.unsold;

              data.allocations.forEach(function(a) {
                var sellerContact = [a.sellerPhone, a.sellerEmail].filter(Boolean).join(' · ');
                var guestContact  = [a.guestPhone, a.guestEmail].filter(Boolean).join(' · ');

                var row  = document.createElement('tr');
                var html = ''
                  + '<td>'
                  +   '<strong>' + (a.ticketId || '--') + '</strong><br/>'
                  +   '<span style="font-size:0.75rem;color:#aaa;">' + (a.ticketType || '') + '</span>'
                  + '</td>'
                  + '<td>'
                  +   (a.sellerName || '<span style="color:#777;">(none)</span>') + '<br/>'
                  +   '<span style="font-size:0.75rem;color:#aaa;">' + (sellerContact || '') + '</span>'
                  + '</td>'
                  + '<td>'
                  +   (a.guestName || '<span style="color:#777;">(none)</span>') + '<br/>'
                  +   '<span style="font-size:0.75rem;color:#aaa;">' + (guestContact || '') + '</span>'
                  + '</td>'
                  + '<td>' + (formatDate(a.allocatedAt) || '--') + '</td>'
                  + '<td>' + (formatDate(a.redeemedAt) || '--') + '</td>'
                  + '<td>' + (a.status || '--') + '</td>';

                row.innerHTML = html;
                body.appendChild(row);
              });
            });
        }

        document.addEventListener('DOMContentLoaded', loadAllocations);
      </script>
    </body>
    </html>`);
});

// ------------------------------------------------------
// ADMIN: Clear full allocation log
// ------------------------------------------------------
app.post("/admin/clear-allocation-log", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.status(403).json({ ok: false, error: "Unauthorized" });
  }

  try {
    ticketAllocations.clear();

    // also wipe the JSON file on disk
    const allocFile = path.join(__dirname, "ticket-allocations.json");
    if (fs.existsSync(allocFile)) {
      fs.writeFileSync(allocFile, JSON.stringify([], null, 2));
    }

    return res.json({ ok: true });
  } catch (err) {
    console.error("Error clearing allocation log:", err);
    return res.status(500).json({ ok: false, error: "Server error" });
  }
});

// ------------------------------------------------------
// ADMIN: Export ALL logs + tickets (JSON download)
// ------------------------------------------------------
app.get("/admin/export", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.status(403).json({ ok: false, error: "Unauthorized" });
  }

  const payload = {
    exportedAt: new Date().toISOString(),

    // All tickets with token
    tickets: Array.from(tickets.entries()).map(([token, rec]) => ({
      token,
      id: rec.id,
      type: rec.type,
      status: rec.status,
    })),

    // Core logs / events
    guestNameEntries,
    staffActivityLog,
    guestScanLog,
    scanEvents,
    paymentEvents,
    boxOfficeSales,
    giveawayEvents,
    ipLogging,

    // Ticket allocations as [ticketId, allocObject]
    ticketAllocations: Array.from(ticketAllocations.entries()),
  };

  res.setHeader("Content-Type", "application/json");
  res.setHeader(
    "Content-Disposition",
    "attachment; filename=aura-all-logs.json"
  );

  res.send(JSON.stringify(payload, null, 2));
});

app.post('/admin/import', (req, res) => {
  const { key } = req.query;
  if (!isMgmtAuthorizedReq(req)) return res.status(403).json({ error: 'Unauthorized' });
  const payload = req.body || {};

  // Import tickets (array of { token, id, type, status })
  if (Array.isArray(payload.tickets)) {
    tickets.clear();
    payload.tickets.forEach(t => {
      if (t && t.token) tickets.set(t.token, { id: t.id || null, type: t.type || 'general', status: t.status || 'unused' });
    });
    try { saveTickets(); } catch(e){}
  }

  // Import guest entries
  if (Array.isArray(payload.guestNameEntries)) {
    guestNameEntries.length = 0;
    payload.guestNameEntries.slice(0, 2000).forEach(e => pushWithLimit(guestNameEntries, e, 2000));
  }

  // Import staff activity
  if (Array.isArray(payload.staffActivityLog)) {
    staffActivityLog.length = 0;
    payload.staffActivityLog.slice(0, 500).forEach(e => pushWithLimit(staffActivityLog, e, 500));
  }

  // Import guest scan log
  if (Array.isArray(payload.guestScanLog)) {
    guestScanLog.length = 0;
    payload.guestScanLog.slice(0, 1000).forEach(e => pushWithLimit(guestScanLog, e, 1000));
  }

  return res.json({ ok: true, message: 'Import complete' });
});

app.post('/admin/seed', (req, res) => {
  const { key } = req.query;
  if (!isMgmtAuthorizedReq(req)) return res.status(403).json({ error: 'Unauthorized' });
  const { what, count = 10 } = req.body || {};
  const n = Math.max(1, Math.min(1000, Number(count) || 10));

  if (what === 'guestEntries') {
    for (let i=0;i<n;i++) {
      pushWithLimit(guestNameEntries, {
        ticketId: `SEED-${Date.now()}-${i}`,
        token: createToken(),
        guestName: `Guest ${i+1}`,
        guestEmail: `guest${i+1}@example.com`,
        guestPhone: `246-555-000${i}`,
        ip: '127.0.0.1',
        timestamp: new Date().toISOString()
      }, 2000);
    }
    return res.json({ ok:true, added: n });
  }
  if (what === 'staffActivity') {
    for (let i=0;i<n;i++) {
      pushWithLimit(staffActivityLog, { name: `Staff${i+1}`, action: 'login', role: (i%5===0?'management':'staff'), ip: '127.0.0.1', timestamp: new Date().toISOString() }, 500);
    }
    return res.json({ ok:true, added: n });
  }
  if (what === 'guestScans') {
    for (let i=0;i<n;i++) {
      pushWithLimit(guestScanLog, { ticketId: `SCAN-${i+1}`, token: createToken(), ip: '127.0.0.1', timestamp: new Date().toISOString() }, 1000);
    }
    return res.json({ ok:true, added: n });
  }
  if (what === 'tickets') {
    for (let i=0;i<n;i++) {
      const token = createToken();
      tickets.set(token, { id: `T-${Date.now()}-${i}`, type: 'general', status: 'unused' });
    }
    try { saveTickets(); } catch(e){}
    return res.json({ ok:true, added: n });
  }

  return res.status(400).json({ error: 'Invalid seed type. Use guestEntries|staffActivity|guestScans|tickets' });
});

app.post('/admin/clear', (req, res) => {
  const { key } = req.query;
  if (!isMgmtAuthorizedReq(req)) {
    return res.status(403).json({ error: 'Unauthorized' });
  }

  const { what } = req.body || {};

  // Clear EVERYTHING – tickets + logs
  if (what === 'all') {
    guestNameEntries.length = 0;
    staffActivityLog.length = 0;
    guestScanLog.length = 0;
    tickets.clear();
    try { saveTickets(); } catch (e) {}
    return res.json({ ok: true, cleared: 'all' });
  }

  // Existing individual clears
  if (what === 'guestEntries') {
    guestNameEntries.length = 0;
    return res.json({ ok: true });
  }
  if (what === 'staffActivity') {
    staffActivityLog.length = 0;
    return res.json({ ok: true });
  }
  if (what === 'guestScans') {
    guestScanLog.length = 0;
    return res.json({ ok: true });
  }
  if (what === 'tickets') {
    tickets.clear();
    try { saveTickets(); } catch (e) {}
    return res.json({ ok: true });
  }

  // 🔥 NEW: clear ONLY TEST tickets + their QR PNGs + related logs
  if (what === 'testTickets') {
    // 1) Remove TEST tickets from tickets map
    for (const [token, rec] of Array.from(tickets.entries())) {
      if (rec && rec.type === "test") {
        tickets.delete(token);
      }
    }
    try { saveTickets(); } catch (e) {}

    // 2) Delete TEST-*.png files from QR folder
    try {
      if (fs.existsSync(QR_DIR)) {
        for (const f of fs.readdirSync(QR_DIR)) {
          if (f.toUpperCase().startsWith("TEST-") && f.toLowerCase().endsWith(".png")) {
            try {
              fs.unlinkSync(path.join(QR_DIR, f));
            } catch (e) {
              console.error("Failed deleting test QR", f, e);
            }
          }
        }
      }
    } catch (e) {
      console.error("Error clearing test QR files:", e);
    }

    const isTestId = (id) =>
      typeof id === "string" && id.toUpperCase().startsWith("TEST-");

    // 3) Clean logs that reference TEST tickets
    scanEvents.invalid = scanEvents.invalid.filter((e) => !isTestId(e.ticketId));
    scanEvents.duplicates = scanEvents.duplicates.filter((e) => !isTestId(e.ticketId));
    ipLogging.events = ipLogging.events.filter((e) => !isTestId(e.ticketId));

    const keptGuestScans = guestScanLog.filter((e) => !isTestId(e.ticketId));
    guestScanLog.length = 0;
    guestScanLog.push(...keptGuestScans);

    const keptNameEntries = guestNameEntries.filter((e) => !isTestId(e.ticketId));
    guestNameEntries.length = 0;
    guestNameEntries.push(...keptNameEntries);

    return res.json({ ok: true, cleared: 'testTickets' });
  }

  // unknown option
  return res.status(400).json({ error: 'Invalid clear target' });
});

// ------------------------------------------------------
// ADMIN: Export mailing list (CSV)
// ------------------------------------------------------
app.get("/admin/export-mailing-list", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.status(403).send("Unauthorized");
  }

  const rows = [];
  // CSV header
  rows.push(["Name","Email","Phone","TicketId","LastUpdated"].join(","));

  const seen = new Set();

  // Walk backwards so the MOST RECENT entry per (email+ticketId) wins
  for (let i = guestNameEntries.length - 1; i >= 0; i--) {
    const e = guestNameEntries[i];

    // Only include people who opted in to subscribe
    if (!e.subscribe) continue;

    const key = (e.guestEmail || "") + "|" + (e.ticketId || "");
    if (seen.has(key)) continue;
    seen.add(key);

    const name     = (e.guestName  || "").replace(/,/g, " ");
    const email    = (e.guestEmail || "").trim();
    const phone    = (e.guestPhone || "").replace(/,/g, " ");
    const ticketId = e.ticketId || "";
    const ts       = e.timestamp || "";

    rows.push([name, email, phone, ticketId, ts].join(","));
  }

  const csv = rows.join("\n");

  res.setHeader("Content-Type", "text/csv");
  res.setHeader(
    "Content-Disposition",
    "attachment; filename=aura-mailing-list.csv"
  );
  res.send(csv);
});

// ------------------------------------------------------
// NEW ENDPOINT: Mark ticket as used with logging (staff-only)
app.post("/api/mark-ticket-used", (req, res) => {
  const { token, key } = req.body || {};
  if (key !== STAFF_PIN) {
    return res.status(403).json({ error: "Invalid staff key" });
  }
  if (!token || !tickets.has(token)) {
    return res.status(404).json({ error: "Ticket not found" });
  }

  const record = tickets.get(token);
  const clientIP = getClientIP(req);

  // Log staff marking the ticket as used
  ipLogging.events.push({
    ip: clientIP,
    token,
    ticketId: record.id,
    timestamp: Date.now(),
    status: "staff-mark-used"
  });

  // STAFF MARKS TICKET AS USED
  // Does NOT count as sold. Does NOT affect allocation.
  record.status = "used";
  tickets.set(token, record);
  saveTickets();

  return res.json({ ok: true, ticketId: record.id, status: "used" });
});

// ROUTE: Mark ticket as used (staff-only, door validation)
// ------------------------------------------------------
app.get("/use-ticket", (req, res) => {
  const { token, key } = req.query;

  if (key !== STAFF_PIN) {
    return res.status(403).send("Invalid staff key.");
  }
  if (!token || !tickets.has(token)) {
    return res.status(404).send("Ticket not found.");
  }

  const record = tickets.get(token);

  // STAFF MARKS AS USED — DOES NOT COUNT AS SOLD
  record.status = "used";
  tickets.set(token, record);
  saveTickets();

  res.redirect(
    "/ticket?token=" +
      encodeURIComponent(token) +
      "&staff=1&key=" +
      encodeURIComponent(STAFF_PIN)
  );
});

// STAFF LOGIN LOGGING
app.post("/api/staff-login", (req, res) => {
  const { name } = req.body || {};
  const cleanName = (name || "").trim();
  const clientIP = getClientIP(req);
  
  if (!cleanName) {
    return res.status(400).json({ error: "Missing name" });
  }

  // Determine if this is a manager or regular staff
  const ALLOWED_MANAGERS = ["RAY", "SHAWN", "NIQUE", "CHE"];
  const isManager = ALLOWED_MANAGERS.includes(cleanName.toUpperCase());
  const role = isManager ? "management" : "staff";

  pushWithLimit(
    staffActivityLog,
    {
      name: cleanName,
      action: "login",
      role: role,
      ip: clientIP,
      timestamp: new Date().toISOString(),
    },
    500
  );

  return res.json({ success: true, role: role });
});

// NEW ENDPOINT: Track staff logout
app.post("/api/staff-logout", (req, res) => {
  const { name } = req.body || {};
  const cleanName = (name || "").trim();
  const clientIP = getClientIP(req);
  
  if (!cleanName) {
    return res.status(400).json({ error: "Missing name" });
  }

  const ALLOWED_MANAGERS = ["RAY", "SHAWN", "NIQUE", "CHE"];
  const isManager = ALLOWED_MANAGERS.includes(cleanName.toUpperCase());
  const role = isManager ? "management" : "staff";

  pushWithLimit(
    staffActivityLog,
    {
      name: cleanName,
      action: "logout",
      role: role,
      ip: clientIP,
      timestamp: new Date().toISOString(),
    },
    500
  );

  return res.json({ success: true });
});

// API: Management login (sets cookie so pin required only once per session)
app.post('/api/mgmt-login', (req, res) => {
  const { name, pin } = req.body || {};
  if (!name || !pin) return res.status(400).json({ error: 'Missing name or pin' });
  if (pin !== MANAGEMENT_PIN) return res.status(403).json({ error: 'Invalid pin' });
  const clean = String(name).trim().toUpperCase();
  const ALLOWED_MANAGERS = ["RAY", "RAYMOND", "SHAWN", "NIQUE", "CHE"];
  if (!ALLOWED_MANAGERS.includes(clean)) return res.status(403).json({ error: 'Name not authorized' });

  // Set cookies: mgmtAuth=1 (httpOnly) and mgmtName (not httpOnly so client can read)
  res.setHeader('Set-Cookie', [
    'mgmtAuth=1; Path=/; HttpOnly',
    `mgmtName=${encodeURIComponent(name)}; Path=/`
  ]);

  pushWithLimit(staffActivityLog, { name: name, role: 'management', action: 'login', ip: getClientIP(req), timestamp: new Date().toISOString() }, 500);

  return res.json({ success: true });
});

// API: Management logout (clears cookies)
app.post('/api/mgmt-logout', (req, res) => {
  res.setHeader('Set-Cookie', [
    'mgmtAuth=; Path=/; Max-Age=0',
    'mgmtName=; Path=/; Max-Age=0'
  ]);
  return res.json({ success: true });
});

// ------------------------------------------------------
// API: Cancelled tickets list (grouped)
// ------------------------------------------------------
app.get("/api/cancelled-tickets", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.status(403).json({ ok: false, error: "Unauthorized" });
  }

  const cancelled = [];

  // derive from tickets map so nothing is missed
  for (const [token, rec] of tickets.entries()) {
    if (rec.status !== "cancelled") continue;

    const guest = getGuestInfoForTicket(rec.id);
    const alloc = ticketAllocations.get(rec.id) || {};
    const logEntry = [...cancelledTicketsLog].reverse().find(
      c => c.ticketId === rec.id
    ) || {};

    const id = rec.id || "";
    const prefix = id.includes("-") ? id.split("-")[0] : "OTHER";

    cancelled.push({
      ticketId: id,
      ticketType: rec.type || null,
      prefix,
      guestName: guest?.name || "",
      guestEmail: guest?.email || "",
      guestPhone: guest?.phone || "",
      sellerName: alloc.sellerName || "",
      cancelledBy: logEntry.cancelledBy || "",
      cancelledAt: logEntry.timestamp ? new Date(logEntry.timestamp).toISOString() : "",
    });
  }

  // group by prefix
  const groups = {};
  for (const c of cancelled) {
    if (!groups[c.prefix]) groups[c.prefix] = [];
    groups[c.prefix].push(c);
  }

  res.json({ ok: true, groups });
});

// ------------------------------------------------------
// API: Live analytics summary (no payment conversions)
// ------------------------------------------------------
app.get("/api/live-analytics", (req, res) => {
  const { key } = req.query;

  // allow staff and management
  if (!(key === STAFF_PIN || isMgmtAuthorizedReq(req))) {
    return res.json({ error: "Unauthorized" });
  }

  let total = 0;
  let used = 0;
  const byType = {}; // e.g. { general: {total,used}, test: {...} }

  for (const [_token, record] of tickets.entries()) {
    total++;

    const t = record.type || "general";
    if (!byType[t]) {
      byType[t] = { total: 0, used: 0 };
    }
    byType[t].total++;

    if (record.status === "used") {
      used++;
      byType[t].used++;
    }
  }

  const unusedCount = total - used;
  const usagePercent =
    total > 0 ? Math.round((used / total) * 100) : 0;

  // ---- TEST tickets ----
  const rawTest = byType.test || { total: 0, used: 0 };
  const testStats = {
    total: rawTest.total,
    used: rawTest.used,
    pending: rawTest.total - rawTest.used,
    arrivalPercent:
      rawTest.total > 0
        ? Math.round((rawTest.used / rawTest.total) * 100)
        : 0,
  };

  // ---- Live (non-TEST) tickets ----
  const liveTotal = total - testStats.total;
  const liveUsed = used - testStats.used;
  const livePending = liveTotal - liveUsed;
  const liveStats = {
    total: liveTotal,
    used: liveUsed,
    pending: livePending,
    arrivalPercent:
      liveTotal > 0
        ? Math.round((liveUsed / liveTotal) * 100)
        : 0,
  };

  const recentScans = ipLogging.events.slice(-10).reverse().map((evt) => ({
    ticketId: evt.ticketId,
    status: evt.status,
    time: new Date(evt.timestamp).toLocaleTimeString("en-US", {
      hour: "numeric",
      minute: "2-digit",
      second: "2-digit",
    }),
  }));

  const invalidCount = scanEvents.invalid.length;
  const duplicateCount = scanEvents.duplicates.length;

  return res.json({
    ok: true,
    total,           // all tickets (incl. TEST)
    used,
    unused: unusedCount,
    usagePercent,
    byType,
    liveStats,       // NEW: live only
    testStats,       // NEW: test only
    recentScans,
    invalidCount,
    duplicateCount,
  });
});



// ------------------------------------------------------
// ROUTE: Live Event Analytics (UI) – Payment Conversions removed
// ------------------------------------------------------
app.get("/live-analytics", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.redirect("/staff?key=" + encodeURIComponent(STAFF_PIN));
  }

  res.send(`<!DOCTYPE html>
    <html>
    <head>
      <meta charset="UTF-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1" />
      <title>Live Event Analytics</title>
      <style>
        ${themeCSSRoot()}
        body {
          margin:0;
          padding:18px;
          font-family:system-ui, -apple-system, BlinkMacSystemFont, sans-serif;
          background:#050007;
          color:#f5f5f5;
          display:flex;
          justify-content:center;
          min-height:100vh;
        }
        .container {
          width:100%;
          max-width:1200px;
        }
        h1 {
          font-size:1.8rem;
          margin:0 0 6px;
          background:linear-gradient(120deg,#ffb300,#ff4081,#ff1744);
          -webkit-background-clip:text;
          color:transparent;
        }
        .subtitle {
          font-size:0.9rem;
          color:#aaa;
          margin-bottom:24px;
        }
        .grid {
          display:grid;
          grid-template-columns:repeat(auto-fit, minmax(200px, 1fr));
          gap:14px;
          margin-bottom:24px;
        }
        .card {
          background:radial-gradient(circle at top, #220018, #070008 60%);
          border-radius:16px;
          padding:24px;
          border:1px solid rgba(255,64,129,0.25);
          box-shadow:0 8px 24px rgba(0,0,0,0.5);
        }
        .stat {
          text-align:center;
        }
        .stat-label {
          font-size:0.8rem;
          color:#aaa;
          text-transform:uppercase;
          letter-spacing:0.08em;
          margin-bottom:8px;
        }
        .stat-value {
          font-size:2.8rem;
          font-weight:700;
          background:linear-gradient(120deg,#ffb300,#ff4081);
          -webkit-background-clip:text;
          color:transparent;
          margin-bottom:4px;
        }
        .stat-percent {
          font-size:0.85rem;
          color:#999;
        }
        table {
          width:100%;
          border-collapse:collapse;
          font-size:0.85rem;
        }
        th, td {
          padding:8px;
          border-bottom:1px solid rgba(255,255,255,0.06);
          text-align:left;
        }
        th {
          font-size:0.78rem;
          text-transform:uppercase;
          letter-spacing:0.08em;
          color:#bbb;
        }
        .refresh-hint {
          margin-top:8px;
          font-size:0.8rem;
          color:#888;
        }
        .back-btn {
          display:inline-block;
          margin-top:18px;
          padding:8px 16px;
          border-radius:999px;
          border:1px solid rgba(255,255,255,0.2);
          text-decoration:none;
          color:#fff;
          font-size:0.85rem;
          letter-spacing:0.08em;
          text-transform:uppercase;
        }
        .status {
          padding:2px 8px;
          border-radius:999px;
          font-size:0.7rem;
        }
        .status-valid {
          background:rgba(76,175,80,0.2);
          color:#4caf50;
        }
        .status-invalid {
          background:rgba(244,67,54,0.2);
          color:#f44336;
        }
        .status-duplicate {
          background:rgba(255,193,7,0.2);
          color:#ffc107;
        }
      </style>
    </head>
    <body>
      <div class="container">
        <h1>📊 Live Event Analytics</h1>
        <div class="subtitle">Real-time event performance dashboard</div>

        <div class="grid">
  <!-- Live tickets -->
  <div class="card stat">
    <div class="stat-label">Total Checked In</div>
    <div class="stat-value" id="totalCheckedIn">0</div>
    <div class="stat-percent" id="arrivalRate">0% arrival (live only)</div>
  </div>

  <div class="card stat">
    <div class="stat-label">Still Pending</div>
    <div class="stat-value" id="stillPending">0</div>
    <div class="stat-percent">live tickets not yet scanned</div>
  </div>

  <div class="card stat">
    <div class="stat-label">Giveaway Winners</div>
    <div class="stat-value" id="giveawayWinners">0</div>
    <div class="stat-percent">pulled from prize draws</div>
  </div>

  <!-- TEST ticket stats -->
  <div class="card stat">
    <div class="stat-label">TEST Tickets</div>
    <div class="stat-value" id="testTotal">0</div>
    <div class="stat-percent">
      <span id="testCheckedIn">0</span> checked in &bull;
      <span id="testPending">0</span> pending
    </div>
  </div>

  <!-- (Optional) Box office – stays for future use, will just show 0 for now -->
  <div class="card stat">
    <div class="stat-label">Box Office Sales</div>
    <div class="stat-value" id="boxOfficeCount">0</div>
  </div>
</div>


        <div class="card">
          <h2 style="margin-top:0;font-size:1.1rem;">Breakdown by Ticket Type</h2>
          <table id="typeTable">
            <thead>
              <tr>
                <th>Type</th>
                <th>Total</th>
                <th>Checked In</th>
                <th>% Arrived</th>
              </tr>
            </thead>
            <tbody id="typeBody">
            </tbody>
          </table>
        </div>

        <div class="card">
          <h2 style="margin-top:0;font-size:1.1rem;">Last 10 Scans</h2>
          <table id="scanTable">
            <thead>
              <tr>
                <th>Ticket</th>
                <th>Status</th>
                <th>Time</th>
              </tr>
            </thead>
            <tbody id="scanBody">
            </tbody>
          </table>
          <div class="refresh-hint">Updates every 3 seconds</div>
        </div>

        <a href="/management-hub?key=${encodeURIComponent(
          MANAGEMENT_PIN
        )}" class="back-btn">← Back to Management Hub</a>
      </div>

      ${themeScript()}
      <script>
        const STAFF_KEY = "${STAFF_PIN}";

        function updateAnalytics() {
          fetch("/api/live-analytics?key=" + encodeURIComponent(STAFF_KEY))
            .then(function (r) { return r.json(); })
            .then(function (data) {
              if (!data || data.error) return;

              var live = data.liveStats || {};
              var test = data.testStats || {};

              // --- HEADLINE (LIVE ONLY) ---
              var checkedEl = document.getElementById("totalCheckedIn");
              if (checkedEl) {
                checkedEl.textContent =
                  (live.used != null ? live.used : (data.used || 0));
              }

              var pendingEl = document.getElementById("stillPending");
              if (pendingEl) {
                pendingEl.textContent =
                  (live.pending != null ? live.pending : (data.unused || 0));
              }

              var arrivalEl = document.getElementById("arrivalRate");
              if (arrivalEl) {
                var pct = (live.arrivalPercent != null
                  ? live.arrivalPercent
                  : (data.usagePercent || 0));
                arrivalEl.textContent = pct + "% arrival (live only)";
              }

              // --- GIVEAWAY + BOX OFFICE (safe defaults) ---
              var giveawayEl = document.getElementById("giveawayWinners");
              if (giveawayEl) {
                giveawayEl.textContent = (data.giveaways || 0);
              }

              var boxEl = document.getElementById("boxOfficeCount");
              if (boxEl) {
                boxEl.textContent = (data.boxOfficeSales || 0);
              }

              // --- TEST CARD ---
              var testTotalEl = document.getElementById("testTotal");
              if (testTotalEl) testTotalEl.textContent = (test.total || 0);

              var testInEl = document.getElementById("testCheckedIn");
              if (testInEl) testInEl.textContent = (test.used || 0);

              var testPendingEl = document.getElementById("testPending");
              if (testPendingEl) testPendingEl.textContent = (test.pending || 0);

              // --- BREAKDOWN BY TYPE ---
              var typeBody = document.getElementById("typeBody");
              if (typeBody) {
                typeBody.innerHTML = Object.entries(data.byType || {})
                  .map(function (entry) {
                    var type = entry[0];
                    var stats = entry[1];
                    var arrivalPct = stats.total > 0
                      ? Math.round((stats.used / stats.total) * 100)
                      : 0;

                    return '<tr>' +
                      '<td><strong>' + type + '</strong></td>' +
                      '<td>' + stats.total + '</td>' +
                      '<td style="color:#34c759;">' + stats.used + '</td>' +
                      '<td>' + arrivalPct + '%</td>' +
                      '</tr>';
                  })
                  .join('');
              }

              // --- LAST 10 SCANS ---
              var scanBody = document.getElementById("scanBody");
              if (scanBody) {
                scanBody.innerHTML = (data.recentScans || [])
                  .map(function (scan) {
                    var statusBadge = '';
                    if (scan.status === 'valid') {
                      statusBadge = '<span class="status status-valid">VALID</span>';
                    } else if (scan.status === 'invalid') {
                      statusBadge = '<span class="status status-invalid">INVALID</span>';
                    } else if (scan.status === 'duplicate') {
                      statusBadge = '<span class="status status-duplicate">DUPLICATE</span>';
                    } else {
                      statusBadge = '<span class="status">' + scan.status + '</span>';
                    }

                    return '<tr>' +
                      '<td><strong>' + scan.ticketId + '</strong></td>' +
                      '<td>' + statusBadge + '</td>' +
                      '<td style="font-size:0.85rem;">' + scan.time + '</td>' +
                      '</tr>';
                  })
                  .join('');
              }
            });
        }

        updateAnalytics();
        setInterval(updateAnalytics, 3000);
      </script>
    </body>
    </html>`);
});



// ------------------------------------------------------
// MANAGEMENT DASHBOARD (Event Snapshot + QR + types)
// ------------------------------------------------------
app.get("/dashboard", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.redirect("/staff?key=" + encodeURIComponent(STAFF_PIN));
  }

    // ---- TICKET STATS ----
  let total = 0;
  let used = 0;
  const byType = {}; // { general: {total, used}, earlybird: {...}, test: {...}, ... }

  for (const [_token, record] of tickets.entries()) {
    total++;

    const t = record.type || "general";
    if (!byType[t]) {
      byType[t] = { total: 0, used: 0 };
    }
    byType[t].total++;

    if (record.status === "used") {
      used++;
      byType[t].used++;
    }
  }

  const totalUnused = total - used;

  const usagePercent = total > 0 ? Math.round((used / total) * 100) : 0;

  // Separate TEST tickets from live event tickets
  const rawTestStats = byType.test || { total: 0, used: 0 };
  const testTotal = rawTestStats.total;
  const testUsed = rawTestStats.used;
  const testPending = testTotal - testUsed;

  const liveTotal = total - testTotal;
  const liveUsed = used - testUsed;
  const livePending = liveTotal - liveUsed;
  const liveUsagePercent = liveTotal > 0 ? Math.round((liveUsed / liveTotal) * 100) : 0;


  const unused = total - used;
  // usagePercent already defined above – no need to redefine it here

  // ---- QR PNG COUNT (split live vs TEST) ----
  let qrCount = 0;
  let testQrCount = 0;
  let liveQrCount = 0;

  try {
    if (fs.existsSync(QR_DIR)) {
      const files = fs
        .readdirSync(QR_DIR)
        .filter((f) => f.toLowerCase().endsWith(".png"));

      qrCount = files.length;

      for (const f of files) {
        if (f.toUpperCase().startsWith("TEST-")) {
          testQrCount++;
        } else {
          liveQrCount++;
        }
      }
    }
  } catch (e) {
    console.error("Error reading QR dir:", e);
  }


  // ---- INVALID / DUPLICATE SCANS SUMMARY ----
  const invalidCount = scanEvents.invalid.length;
  const duplicateCount = scanEvents.duplicates.length;

  res.send(`<!DOCTYPE html>
    <html>
    <head>
      <meta charset="UTF-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1" />
      <title>AURA Event Snapshot</title>
      <style>
        ${themeCSSRoot()}
        * { box-sizing: border-box; }

        body {
          margin:0;
          padding:18px;
          min-height:100vh;
          font-family:system-ui,-apple-system,BlinkMacSystemFont,sans-serif;
          background:
            radial-gradient(circle at top left, rgba(255,64,129,0.25), transparent 55%),
            radial-gradient(circle at bottom right, rgba(255,23,68,0.35), transparent 55%),
            var(--bg-dark);
          color:var(--text-main);
          display:flex;
          justify-content:center;
        }

        .shell {
          width:100%;
          max-width:1100px;
        }

        .header-row {
          display:flex;
          justify-content:space-between;
          align-items:flex-start;
          gap:12px;
          margin-bottom:18px;
        }

        h1 {
          margin:0;
          font-size:1.9rem;
          background:linear-gradient(120deg,#ffb300,#ff4081,#ff1744);
          -webkit-background-clip:text;
          color:transparent;
        }

        .header-sub {
          font-size:0.9rem;
          color:#aaa;
        }

        .badge {
          padding:6px 12px;
          border-radius:999px;
          border:1px solid rgba(255,64,129,0.55);
          font-size:0.75rem;
          text-transform:uppercase;
          letter-spacing:0.12em;
          color:#ffb300;
          background:rgba(0,0,0,0.35);
          align-self:flex-start;
        }

        .grid-main {
          display:grid;
          grid-template-columns:repeat(auto-fit,minmax(220px,1fr));
          gap:14px;
          margin-bottom:18px;
        }

        .card {
          background:radial-gradient(circle at top,#220018,#070008 65%);
          border-radius:18px;
          padding:18px 16px;
          border:1px solid rgba(255,64,129,0.3);
          box-shadow:0 14px 32px rgba(0,0,0,0.8);
        }

        .stat-label {
          font-size:0.78rem;
          text-transform:uppercase;
          letter-spacing:0.1em;
          color:#b0b0ff;
          margin-bottom:6px;
        }

        .stat-value {
          font-size:2.1rem;
          font-weight:700;
        }

        .stat-meta {
          margin-top:4px;
          font-size:0.85rem;
          color:#aaa;
        }

        .stat-value-green {
          color:#34c759;
        }
        .stat-value-gold {
          color:#ffb300;
        }
        .stat-value-red {
          color:#ff5252;
        }

        h2 {
          margin:0 0 10px;
          font-size:1.1rem;
        }

        table {
          width:100%;
          border-collapse:collapse;
          font-size:0.85rem;
        }

        th, td {
          padding:7px 6px;
          border-bottom:1px solid rgba(255,255,255,0.07);
          text-align:left;
        }

        th {
          font-size:0.78rem;
          text-transform:uppercase;
          letter-spacing:0.08em;
          color:#bbb;
        }

        .chip {
          display:inline-block;
          padding:3px 8px;
          border-radius:999px;
          font-size:0.7rem;
          text-transform:uppercase;
          letter-spacing:0.08em;
        }
        .chip-ok {
          background:rgba(76,175,80,0.2);
          color:#4caf50;
        }
        .chip-warn {
          background:rgba(255,193,7,0.2);
          color:#ffc107;
        }
        .chip-bad {
          background:rgba(244,67,54,0.2);
          color:#f44336;
        }

        .footer-row {
          margin-top:18px;
          display:flex;
          flex-wrap:wrap;
          gap:10px;
          align-items:center;
          justify-content:space-between;
        }

        .back-btn {
          display:inline-block;
          padding:8px 16px;
          border-radius:999px;
          border:1px solid rgba(255,255,255,0.25);
          text-decoration:none;
          color:#fff;
          font-size:0.85rem;
          letter-spacing:0.08em;
          text-transform:uppercase;
        }

        .link-row {
          font-size:0.82rem;
          color:#aaa;
        }

        .link-row a {
          color:#ffb300;
          text-decoration:none;
        }

        .link-row a:hover { text-decoration:underline; }

        @media (max-width:720px) {
          .card { padding:16px 14px; }
          .stat-value { font-size:1.8rem; }
          h1 { font-size:1.5rem; }
        }
      </style>
    </head>
    <body>
      <div class="shell">
        <div class="header-row">
          <div>
            <h1>Event Snapshot</h1>
            <div class="header-sub">
              Live overview of Hearts &amp; Spades tickets, QR codes and scan health.
            </div>
          </div>
          <div class="badge">MANAGEMENT VIEW</div>
        </div>

   <div class="grid-main">
  <!-- Live (real) tickets -->
  <div class="card">
    <div class="stat-label">Event Tickets</div>
    <div class="stat-value stat-value-gold">${liveTotal}</div>
    <div class="stat-meta">Live tickets for Hearts &amp; Spades (excludes TEST).</div>
  </div>

  <div class="card">
    <div class="stat-label">Checked In</div>
    <div class="stat-value stat-value-green">${liveUsed}</div>
    <div class="stat-meta">${liveUsagePercent}% arrival rate so far (live only).</div>
  </div>

  <div class="card">
    <div class="stat-label">Pending</div>
    <div class="stat-value">${livePending}</div>
    <div class="stat-meta">Live tickets still to arrive / not scanned yet.</div>
  </div>

  <div class="card">
    <div class="stat-label">QR PNG Files</div>
    <div class="stat-value stat-value-gold">${qrCount}</div>
    <div class="stat-meta">
      All QR codes in <code>generated_qr/</code>
      &mdash; <strong>${testQrCount}</strong> are TEST.
    </div>
  </div>

  <!-- NEW: TEST summary -->
  <div class="card">
    <div class="stat-label">TEST Tickets</div>
    <div class="stat-value stat-value-red">${testTotal}</div>
    <div class="stat-meta">
      <strong>${testUsed}</strong> checked in &bull;
      <strong>${testPending}</strong> pending.
    </div>
  </div>
</div>


        <div class="grid-main">
          <div class="card">
            <h2>By Ticket Type</h2>
            <table>
              <thead>
                <tr>
                  <th>Type</th>
                  <th>Total</th>
                  <th>Checked In</th>
                  <th>Arrival %</th>
                </tr>
              </thead>
              <tbody>
                ${
                  Object.keys(byType).length === 0
                    ? `<tr><td colspan="4" style="color:#888;font-size:0.85rem;">No tickets generated yet.</td></tr>`
                    : Object.entries(byType)
                        .map(([type, stats]) => {
                          const pct =
                            stats.total > 0
                              ? Math.round((stats.used / stats.total) * 100)
                              : 0;
                          return `<tr>
                            <td><strong>${type}</strong></td>
                            <td>${stats.total}</td>
                            <td style="color:#34c759;">${stats.used}</td>
                            <td>${pct}%</td>
                          </tr>`;
                        })
                        .join("")
                }
              </tbody>
            </table>
          </div>

          <div class="card">
            <h2>Scan Health</h2>
            <p style="font-size:0.87rem; color:#ccc; margin-top:0; margin-bottom:10px;">
              Quick view of how clean your door scanning has been.
            </p>
            <table>
              <tbody>
                <tr>
                  <td>Invalid / forged scans</td>
                  <td style="text-align:right;">
                    <span class="chip ${invalidCount === 0 ? "chip-ok" : "chip-bad"}">
                      ${invalidCount}
                    </span>
                  </td>
                </tr>
                <tr>
                  <td>Duplicate scans</td>
                  <td style="text-align:right;">
                    <span class="chip ${duplicateCount === 0 ? "chip-ok" : "chip-warn"}">
                      ${duplicateCount}
                    </span>
                  </td>
                </tr>
              </tbody>
            </table>

            <!-- NEW: show test ticket stats -->
            <div style="margin-top:10px;font-size:0.82rem;color:#bbb;">
              Test tickets – generated: <strong>${testTotal}</strong>,
              scanned: <strong>${testUsed}</strong>,
              pending: <strong>${testPending}</strong>
            </div>

            <p style="font-size:0.8rem; color:#888; margin-top:10px;">
              For deep dive, use the Live Analytics dashboard.
            </p>
          </div>
        </div>

        <div class="footer-row">
          <a href="/management-hub?key=${encodeURIComponent(MANAGEMENT_PIN)}" class="back-btn">
            ← Back to Management Hub
          </a>
          <div class="link-row">
            Also see:
            <a href="/live-analytics?key=${encodeURIComponent(MANAGEMENT_PIN)}">Live Analytics</a>
            ·
<a
  href="/staff/qr-list?key=${encodeURIComponent(MANAGEMENT_PIN)}"
  class="action-button btn-qrfiles"
>
  📂 View / Download Generated QR Files →
</a>
          </div>
        </div>
      </div>

      ${themeScript()}
    </body>
    </html>`);
});

// ------------------------------------------------------
// ALLOCATIONS + APIs (kept same logic, no style changes)
// ------------------------------------------------------
// Basic allocations stats (sold/unsold per prefix)
app.get("/api/allocations-stats", (req, res) => {
  const { key } = req.query;
  if (!(key === STAFF_PIN || isMgmtAuthorizedReq(req))) {
    return res.json({ sold: 0, unsold: 0, pending: 0, byPrefix: {} });
  }

  const byPrefix = {};
  let totalSold = 0;
  let totalUnsold = 0;
  let totalPending = 0;

  for (const [_token, record] of tickets.entries()) {
    const id = record.id || record.ticketId || "";
    const prefix = id.split("-")[0] + "-";

    if (!byPrefix[prefix]) {
      byPrefix[prefix] = { sold: 0, unsold: 0, pending: 0 };
    }

    if (record.status === "used") {
      byPrefix[prefix].sold++;
      totalSold++;
    } else if (record.status === "pending") {
      byPrefix[prefix].pending++;
      totalPending++;
    } else {
      byPrefix[prefix].unsold++;
      totalUnsold++;
    }
  }

  res.json({ sold: totalSold, unsold: totalUnsold, pending: totalPending, byPrefix });
});

// NEW: resolve a scanned QR (URL or token) into a ticket ID
app.get("/api/lookup-ticket-by-token", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.status(403).json({ error: "Unauthorized" });
  }

  const { token } = req.query;
  if (!token) {
    return res.status(400).json({ error: "Missing token" });
  }

  const record = tickets.get(token);
  if (!record) {
    return res.status(404).json({ error: "Ticket not found" });
  }

  return res.json({
    ok: true,
    ticketId: record.id || record.ticketId || null,
    status: record.status || "unused",
    type: record.type || null,
  });
});

// ------------------------------------------------------
// MANAGEMENT: Seller Allocations Overview
// ------------------------------------------------------
app.get("/allocations", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.redirect("/staff?key=" + encodeURIComponent(STAFF_PIN));
  }

  res.send(`<!DOCTYPE html>
  <html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Ticket Allocations</title>

    <style>
      ${themeCSSRoot()}

      body {
        margin:0;
        padding:16px;
        font-family:system-ui,-apple-system,BlinkMacSystemFont,sans-serif;
        background:#050007;
        color:#f5f5f5;
        display:flex;
        justify-content:center;
        min-height:100vh;
      }

      .wrap {
        width:100%;
        max-width:900px;
      }

      .header-row {
        display:flex;
        flex-direction:column;
        gap:8px;
        margin-bottom:12px;
      }

      .title {
        font-size:1.35rem;
        font-weight:700;
      }

      .title span {
        background:linear-gradient(90deg,#ff4081,#ffb300);
        -webkit-background-clip:text;
        color:transparent;
      }

      .subtitle {
        font-size:0.9rem;
        color:#c0c0c0;
      }

.top-actions {
  display: flex;
  gap: 8px;
  margin: 10px 0 18px;
  flex-wrap: wrap;
}

.pill-btn {
  border-radius: 999px;
  padding: 6px 12px;
  border: none;
  font-size: 11px;
  text-transform: uppercase;
  letter-spacing: 0.09em;
  cursor: pointer;
  background: linear-gradient(135deg,#ff5fa2,#ffb347);
  color:#050007;
}
.pill-btn:active { transform: translateY(1px); }

      .top-actions {
        display:flex;
        flex-wrap:wrap;
        gap:8px;
        margin-top:8px;
      }

      .pill-link,
      .pill-btn {
        display:inline-flex;
        align-items:center;
        justify-content:center;
        padding:9px 14px;
        border-radius:999px;
        border:none;
        text-decoration:none;
        font-size:0.8rem;
        letter-spacing:0.12em;
        text-transform:uppercase;
        cursor:pointer;
      }

      .pill-link {
        background:linear-gradient(90deg,#ff4081,#ffb300);
        color:#050007;
        font-weight:700;
      }

      .pill-btn {
        background:rgba(255,255,255,0.02);
        color:#f5f5f5;
        border:1px solid rgba(255,255,255,0.2);
      }

      .card {
        margin-top:14px;
        border-radius:20px;
        padding:14px 12px 16px;
        background:radial-gradient(circle at top,#200018,#060008 65%);
        box-shadow:
          0 0 0 1px rgba(255,255,255,0.06),
          0 16px 40px rgba(0,0,0,0.85);
      }

      .stats-row {
        display:flex;
        flex-wrap:wrap;
        gap:10px;
        margin-bottom:10px;
      }

      .stat {
        flex:1 1 90px;
        padding:8px 10px;
        border-radius:14px;
        background:rgba(255,255,255,0.02);
        font-size:0.82rem;
      }

      .stat-label {
        text-transform:uppercase;
        letter-spacing:0.09em;
        color:#bbb;
        margin-bottom:2px;
        font-size:0.74rem;
      }

      .stat-value {
        font-size:1.1rem;
        font-weight:700;
      }

      .table-wrapper {
        margin-top:8px;
        overflow-x:auto;
      }

      table {
        width:100%;
        border-collapse:collapse;
        font-size:0.84rem;
      }

      th, td {
        padding:8px 6px;
        border-bottom:1px solid rgba(255,255,255,0.08);
        white-space:nowrap;
      }

      th {
        text-align:left;
        font-size:0.76rem;
        text-transform:uppercase;
        letter-spacing:0.09em;
        color:#bbb;
      }

      .status-pill {
        display:inline-flex;
        padding:2px 8px;
        border-radius:999px;
        font-size:0.75rem;
      }

      .status-good {
        background:rgba(76,175,80,0.18);
        color:#a5ffb5;
      }

      .status-mid {
        background:rgba(255,193,7,0.15);
        color:#ffe082;
      }

      .status-low {
        background:rgba(255,82,82,0.16);
        color:#ff9ba0;
      }

      .empty {
        text-align:center;
        padding:10px 0;
        color:#888;
        font-size:0.86rem;
      }

      .back-link {
        display:inline-flex;
        align-items:center;
        gap:6px;
        margin-top:14px;
        font-size:0.86rem;
        text-decoration:none;
        color:#ffd86b;
      }

      @media (max-width:600px) {
        .card { border-radius:18px; padding:12px 10px 14px; }
        .header-row { gap:4px; }
      }
    </style>
  </head>

  <body>
    <div class="wrap">
      <div class="header-row">
        <div class="title"><span>Ticket Allocations</span></div>
        <div class="subtitle">
          Track how many tickets each seller or location has been allocated and how many are sold.
        </div>
        <div class="top-actions">
          <a href="/allocation-scanner?key=${encodeURIComponent(MANAGEMENT_PIN)}" class="pill-link">
            Allocation Scanner
          </a>
          <button type="button" id="refreshBtn" class="pill-btn">Refresh</button>
        </div>
      </div>

      <div class="card">
        <div class="stats-row">
          <div class="stat">
            <div class="stat-label">Total Allocated</div>
            <div class="stat-value" id="statAllocated">0</div>
          </div>
          <div class="stat">
            <div class="stat-label">Total Sold</div>
            <div class="stat-value" id="statSold">0</div>
          </div>
          <div class="stat">
            <div class="stat-label">Total Unsold</div>
            <div class="stat-value" id="statUnsold">0</div>
          </div>
        </div>

        <div class="table-wrapper">
          <table>
            <thead>
              <tr>
                <th>Seller / Location</th>
                <th>Allocated</th>
                <th>Sold</th>
                <th>Unsold</th>
                <th>% Sold</th>
              </tr>
<div class="table-wrap">
  <table>
    <thead>
      <!-- keep your existing header row here -->
    </thead>
    <tbody id="allocBody">
      <tr><td colspan="5" class="empty">Loading…</td></tr>
    </tbody>
  </table>
</div>

      </div>

      <a href="/management-hub?key=${encodeURIComponent(MANAGEMENT_PIN)}" class="back-link">
        ← <span>Back to Management Hub</span>
      </a>

      ${themeScript()}

      <script>
        const MANAGEMENT_PIN = "${MANAGEMENT_PIN}";
        const bodyEl = document.getElementById("allocBody");
        const statAllocated = document.getElementById("statAllocated");
        const statSold = document.getElementById("statSold");
        const statUnsold = document.getElementById("statUnsold");
        const refreshBtn = document.getElementById("refreshBtn");

        function statusClassFor(percent) {
          if (percent >= 90) return "status-pill status-good";
          if (percent >= 40) return "status-pill status-mid";
          return "status-pill status-low";
        }

        async function loadAllocations() {
          bodyEl.innerHTML = '<tr><td colspan="5" class="empty">Loading…</td></tr>';

          try {
            const res = await fetch(
              "/api/seller-allocations-summary?key=" + encodeURIComponent(MANAGEMENT_PIN)
            );
            const data = await res.json();

            if (!data || !data.ok) {
              bodyEl.innerHTML =
                '<tr><td colspan="5" class="empty">' +
                (data.error || "Unable to load allocations.") +
                "</td></tr>";
              return;
            }

            const sellers = data.sellers || [];
            if (sellers.length === 0) {
              bodyEl.innerHTML =
                '<tr><td colspan="5" class="empty">No allocations recorded yet.</td></tr>';
            } else {
              bodyEl.innerHTML = sellers.map(s => {
                const total = s.totalAllocated || 0;
                const sold = s.sold || 0;
                const unsold = total - sold;
                const pct = total > 0 ? Math.round((sold / total) * 100) : 0;
                const cls = statusClassFor(pct);

                return (
                  "<tr>" +
                    "<td>" + (s.sellerName || "—") + "</td>" +
                    "<td>" + total + "</td>" +
                    "<td>" + sold + "</td>" +
                    "<td>" + (unsold < 0 ? 0 : unsold) + "</td>" +
                    "<td><span class=\\"" + cls + "\\">" + pct + "%</span></td>" +
                  "</tr>"
                );
              }).join("");
            }

            statAllocated.textContent = data.totalAllocated || 0;
            statSold.textContent = data.totalSold || 0;
            statUnsold.textContent = data.totalUnsold || 0;
          } catch (e) {
            console.error(e);
            bodyEl.innerHTML =
              '<tr><td colspan="5" class="empty">Error: ' + e.message + "</td></tr>";
          }
        }

        refreshBtn.addEventListener("click", loadAllocations);
        loadAllocations();

        const params = new URLSearchParams(location.search);
const MGMT_KEY = params.get("key") || "";

async function reloadAllocations() {
  await loadAllocations(); // you already have this
}

async function clearAllocationLog() {
  if (!confirm("Clear ALL allocation log entries? This does NOT cancel tickets, only the log.")) return;

  const res = await fetch(
    "/admin/clear-allocation-log?key=" + encodeURIComponent(MGMT_KEY),
    { method: "POST" }
  );

  const data = await res.json().catch(() => ({}));
  if (!data.ok) {
    alert(data.error || "Unable to clear allocation log");
    return;
  }

  await loadAllocations();
  alert("Allocation log cleared.");
}
      </script>
    </div>
  </body>
  </html>`);
});

// ------------------------------------------------------
// ADMIN: Clear ALL QR PNG files (images only, tickets stay)
// ------------------------------------------------------
app.post("/admin/clear-qr-files", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.status(403).json({ ok: false, error: "Unauthorized" });
  }

  try {
    let deleted = 0;
    if (fs.existsSync(QR_DIR)) {
      for (const f of fs.readdirSync(QR_DIR)) {
        if (f.toLowerCase().endsWith(".png")) {
          try {
            fs.unlinkSync(path.join(QR_DIR, f));
            deleted++;
          } catch (e) {
            console.error("Failed deleting QR file:", f, e);
          }
        }
      }
    }

    return res.json({ ok: true, deleted });
  } catch (err) {
    console.error("Error clearing QR files:", err);
    return res.status(500).json({ ok: false, error: "Server error" });
  }
});

// ------------------------------------------------------
// MANAGEMENT: Allocation Log (detailed per ticket)
// ------------------------------------------------------
app.get("/allocation-log", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.redirect("/staff?key=" + encodeURIComponent(STAFF_PIN));
  }

  res.send(`<!DOCTYPE html>
  <html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Allocation Log</title>
    <style>
      ${themeCSSRoot()}

      body {
        margin:0;
        padding:16px;
        font-family:system-ui,-apple-system,BlinkMacSystemFont,sans-serif;
        background:#050007;
        color:#f5f5f5;
        display:flex;
        justify-content:center;
      }

      .wrap {
        width:100%;
        max-width:1024px;
      }

      .card {
        width:100%;
        border-radius:24px;
        padding:20px 20px 18px;
        background:radial-gradient(circle at top,#260020,#050007 65%);
        box-shadow:
          0 0 0 1px rgba(255,64,129,0.32),
          0 22px 55px rgba(0,0,0,0.9);
      }

      .card-inner { position:relative; z-index:1; }

      .header-row {
        display:flex;
        justify-content:space-between;
        align-items:center;
        gap:12px;
        margin-bottom:10px;
      }

      .logo-row {
        display:flex;
        align-items:center;
        gap:10px;
      }

      .logo-img {
        height:30px;
        border-radius:8px;
        box-shadow:0 0 12px rgba(255,64,129,0.55);
      }

      .title-main {
        font-size:1.25rem;
        font-weight:650;
      }

      .title-main span {
        background:linear-gradient(120deg,#ffb300,#ff4081,#ff1744);
        -webkit-background-clip:text;
        color:transparent;
      }

      .badge {
        padding:4px 10px;
        border-radius:999px;
        border:1px solid rgba(255,64,129,0.6);
        font-size:0.7rem;
        letter-spacing:0.12em;
        text-transform:uppercase;
        color:#ccc;
        text-align:right;
      }

      .subtitle {
        font-size:0.9rem;
        color:#bbb;
        margin-bottom:14px;
      }

      .stats-row {
        display:flex;
        gap:16px;
        flex-wrap:wrap;
        margin-bottom:14px;
      }

        .table-wrap {
    width:100%;
    overflow-x:auto;
    margin-top:4px;
  }

      .stat-pill {
        padding:8px 12px;
        border-radius:999px;
        background:rgba(0,0,0,0.4);
        border:1px solid rgba(255,255,255,0.12);
        font-size:0.8rem;
      }

      table {
        width:100%;
        border-collapse:collapse;
        margin-top:4px;
        font-size:0.82rem;
      }

      thead {
        background:rgba(255,255,255,0.04);
      }

      th, td {
        padding:6px 8px;
        border-bottom:1px solid rgba(255,255,255,0.08);
        text-align:left;
        vertical-align:top;
      }

      th {
        font-weight:600;
        letter-spacing:0.06em;
        text-transform:uppercase;
        font-size:0.74rem;
        color:#ffb300;
      }

      tr:nth-child(even) td {
        background:rgba(255,255,255,0.01);
      }

      .pill {
        display:inline-block;
        padding:3px 8px;
        border-radius:999px;
        font-size:0.72rem;
        letter-spacing:0.08em;
        text-transform:uppercase;
      }

      .pill-sold {
        background:rgba(76,175,80,0.18);
        border:1px solid rgba(76,175,80,0.6);
        color:#8bc34a;
      }

      .pill-unsold {
        background:rgba(255,193,7,0.12);
        border:1px solid rgba(255,193,7,0.6);
        color:#ffeb3b;
      }

      .back-link {
        display:inline-block;
        margin-top:14px;
        font-size:0.8rem;
        color:#aaa;
        text-decoration:none;
      }

      .back-link span { text-decoration:underline; }

      .note {
        margin-top:8px;
        font-size:0.78rem;
        color:#999;
      }

      @media (max-width:768px) {
        .card { padding:16px 14px 16px; border-radius:18px; }
        th, td { padding:6px 4px; font-size:0.78rem; }
        .title-main { font-size:1.05rem; }
        table { font-size:0.76rem; }
      }
          @media (max-width: 600px) {
    body {
      padding:10px 6px;
      justify-content:flex-start;
    }

    .wrap {
      max-width:100%;
    }

    .card {
      width:100%;
      padding:14px 10px 14px;
      border-radius:18px;
    }

    .header-row {
      flex-direction:column;
      align-items:flex-start;
      gap:8px;
    }

    .title-main {
      font-size:1.05rem;
    }

    .subtitle {
      font-size:0.82rem;
    }

    .stats-row {
      gap:8px;
    }

    .stat-pill {
      font-size:0.75rem;
      padding:6px 10px;
    }

    table {
      font-size:0.74rem;
    }

    th, td {
      padding:4px 3px;
    }

    .note,
    .back-link {
      font-size:0.75rem;
    }
  }

    </style>
  </head>
  <body>
    <div class="wrap">
      <div class="card">
        <div class="card-inner">
          <div class="header-row">
            <div class="logo-row">
              <img src="/aura-logo.png" class="logo-img" alt="AURA"/>
              <img src="/pop-logo.png" class="logo-img" alt="POP"/>
              <div>
                <div class="title-main"><span>Allocation Log</span></div>
                <div style="font-size:0.8rem;color:#ccc;">Per-ticket view of sellers, guests, and status.</div>
              </div>
            </div>
            <div class="badge">MGMT • AURA Hearts & Spades</div>
          </div>

          <div class="subtitle">
            Every ticket that has been <strong>allocated</strong>, with seller contact,
            guest info (where captured), and live sold/unsold status.
          </div>

          <div class="stats-row">
            <div class="stat-pill">Total allocated: <span id="statTotal">0</span></div>
            <div class="stat-pill">Sold: <span id="statSold">0</span></div>
            <div class="stat-pill">Unsold: <span id="statUnsold">0</span></div>
          </div>

          <table>
            <thead>
              <tr>
                <th>Ticket</th>
                <th>Seller</th>
                <th>Guest</th>
                <th>Sold?</th>
                <th>Status</th>
                <th>Last Activity</th>
              </tr>
            </thead>
            <tbody id="allocBody">
              <tr><td colspan="6" style="text-align:center;color:#888;">Loading allocation log…</td></tr>
            </tbody>
          </table>

          <div class="note">
            • Seller contact is taken from the Allocation Scanner form.<br/>
            • Guest details are taken from guest prize entries or box office sales.
          </div>

          <a href="/management-hub?key=${encodeURIComponent(
            MANAGEMENT_PIN
          )}" class="back-link">← <span>Back to Management Hub</span></a>
        </div>
      </div>
         ${themeScript()}
      <script>
        // Read management key from the URL (?key=...)
        const params   = new URLSearchParams(window.location.search);
        const MGMT_KEY = params.get("key") || "";

        function formatDate(iso) {
          if (!iso) return "--";
          try {
            const d = new Date(iso);
            return d.toLocaleDateString() + " " + d.toLocaleTimeString();
          } catch (e) {
            return iso;
          }
        }

        function loadAllocations() {
          fetch('/api/allocations-detail?key=' + encodeURIComponent(MGMT_KEY))
            .then(function (r) { return r.json(); })
            .then(function (data) {
              var body = document.getElementById('allocBody');
              body.innerHTML = '';

              if (!data.ok || !data.allocations || data.allocations.length === 0) {
                body.innerHTML =
                  '<tr><td colspan="6" style="text-align:center;color:#888;">No allocations yet.</td></tr>';
                document.getElementById('statTotal').textContent  = '0';
                document.getElementById('statSold').textContent   = '0';
                document.getElementById('statUnsold').textContent = '0';
                return;
              }

              document.getElementById('statTotal').textContent  = data.total;
              document.getElementById('statSold').textContent   = data.sold;
              document.getElementById('statUnsold').textContent = data.unsold;

              data.allocations.forEach(function (a) {
                var sellerContact = [a.sellerPhone, a.sellerEmail].filter(Boolean).join(' · ');
                var guestContact  = [a.guestPhone, a.guestEmail].filter(Boolean).join(' · ');

                var row  = document.createElement('tr');
                var html = ''
                  + '<td>'
                  +   '<strong>' + (a.ticketId || '--') + '</strong><br/>'
                  +   '<span style="font-size:0.75rem;color:#aaa;">' + (a.ticketType || '') + '</span>'
                  + '</td>'
                  + '<td>'
                  +   (a.sellerName || '<span style="color:#777;">(none)</span>') + '<br/>'
                  +   '<span style="font-size:0.75rem;color:#aaa;">' + (sellerContact || '') + '</span>'
                  + '</td>'
                  + '<td>'
                  +   (a.guestName || '<span style="color:#777;">(no guest yet)</span>') + '<br/>'
                  +   '<span style="font-size:0.75rem;color:#aaa;">' + (guestContact || '') + '</span>'
                  + '</td>'
                  + '<td>'
                  +   (a.sold
                        ? '<span class="pill pill-sold">SOLD</span>'
                        : '<span class="pill pill-unsold">UNSOLD</span>')
                  + '</td>'
                  + '<td>' + (a.ticketStatus || '--') + '</td>'
                  + '<td style="font-size:0.75rem;color:#ccc;">' + formatDate(a.lastActivity) + '</td>';

                row.innerHTML = html;
                body.appendChild(row);
              });
            })
            .catch(function (err) {
              var body = document.getElementById('allocBody');
              body.innerHTML =
                '<tr><td colspan="6" style="text-align:center;color:#f88;">Error loading allocations: '
                + err.message + '</td></tr>';
            });
        }

        document.addEventListener('DOMContentLoaded', loadAllocations);
        
<script>
  const params   = new URLSearchParams(location.search);
  const MGMT_KEY = params.get("key") || "";

  function exportMailingList() {
    const url = "/admin/export-mailing-list?key=" + encodeURIComponent(MGMT_KEY);
    window.location.href = url;
  }
</script>

    </div>
  </body>
</html>
`);
});


// ------------------------------------------------------
// PAYMENTS API (NO CONVERSIONS – raw amounts only)
// ------------------------------------------------------
app.post("/api/record-payment", (req, res) => {
  const { ticketId, method, amount = null, currency = "BBD", notes = "" } = req.body || {};

  if (!ticketId || !method) {
    return res.status(400).json({ error: "Missing ticketId or method" });
  }

  const validMethods = ["cash", "card", "refund"];
  if (!validMethods.includes(method)) {
    return res.status(400).json({ error: "Invalid payment method" });
  }

  const amountNum = amount !== null ? Number(amount) : null;

  paymentEvents.transactions.push({
    ticketId,
    method,
    amount: amountNum,
    currency,
    notes,
    timestamp: new Date().toISOString(),
  });

  return res.json({ success: true, total: paymentEvents.transactions.length });
});

app.get("/api/payment-summary", (req, res) => {
  const { key } = req.query;
  if (!(key === STAFF_PIN || isMgmtAuthorizedReq(req))) {
    return res.json({
      cash: 0,
      card: 0,
      refunds: 0,
      total: 0,
      totalAmount: 0,
    });
  }

  let cash = 0,
    card = 0,
    refunds = 0;
  let totalAmount = 0;

  paymentEvents.transactions.forEach((tx) => {
    if (tx.method === "cash") cash++;
    else if (tx.method === "card") card++;
    else if (tx.method === "refund") refunds++;

    if (typeof tx.amount === "number") {
      totalAmount += tx.amount;
    }
  });

  res.json({
    cash,
    card,
    refunds,
    total: paymentEvents.transactions.length,
    totalAmount: Number(totalAmount.toFixed(2)),
  });
});

// ------------------------------------------------------
// BOX OFFICE TICKET GENERATION
// ------------------------------------------------------
app.post("/api/box-office-generate", (req, res) => {
  const { key, soldTo = "Box Office", amount = null } = req.body || {};
  if (!(key === MANAGEMENT_PIN || isMgmtAuthorizedReq(req))) {
    return res.status(403).json({ error: "Invalid management PIN" });
  }

  const ticketNumber = boxOfficeSales.nextNumber;
  const ticketId = `${boxOfficeSales.prefix}${String(ticketNumber).padStart(4, "0")}`;
  const token = createToken();

  tickets.set(token, {
    id: ticketId,
    type: "box-office",
    status: "unused",
  });

  saveTickets();

  const signature = signToken(token);
  const qrPath = path.join(QR_DIR, `${token}.png`);
  const baseUrl = getBaseUrl(req);
  const qrUrl = `${baseUrl}/ticket?token=${token}&sig=${signature}`;

  QRCode.toFile(qrPath, qrUrl, (err) => {
    if (err) console.error("QR generation error:", err);
  });

  boxOfficeSales.sales.push({
    ticketId,
    qrPath,
    timestamp: Date.now(),
    soldTo,
    amount,
  });

  boxOfficeSales.nextNumber++;

  return res.json({
    success: true,
    ticketId,
    qrUrl,
    qrPath,
  });
});

app.get("/api/box-office-sales", (req, res) => {
  const { key } = req.query;
  if (!(key === MANAGEMENT_PIN || isMgmtAuthorizedReq(req))) {
    return res.json({ sales: [], nextNumber: boxOfficeSales.nextNumber, total: 0 });
  }
  res.json({
    sales: boxOfficeSales.sales.slice(-50).reverse(),
    nextNumber: boxOfficeSales.nextNumber,
    total: boxOfficeSales.sales.length,
  });
});

app.get("/box-office", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.redirect("/staff?key=" + encodeURIComponent(STAFF_PIN));
  }

  res.send(`<!DOCTYPE html>
    <html>
    <head>
      <meta charset="UTF-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1" />
      <title>Box Office</title>
      <style>
        ${themeCSSRoot()}
        body {
          margin:0;
          padding:18px;
          font-family:system-ui,-apple-system,BlinkMacSystemFont,sans-serif;
          background:#050007;
          color:#f5f5f5;
          display:flex;
          justify-content:center;
          min-height:100vh;
        }
        .container {
          width:100%;
          max-width:900px;
        }
        h1 {
          margin:0 0 8px;
          font-size:1.7rem;
          background:linear-gradient(120deg,#ffb300,#ff4081,#ff1744);
          -webkit-background-clip:text;
          color:transparent;
        }
        .subtitle {
          font-size:0.9rem;
          color:#aaa;
          margin-bottom:18px;
        }
        .card {
          background:radial-gradient(circle at top,#220018,#070008 60%);
          border-radius:16px;
          padding:18px;
          border:1px solid rgba(255,64,129,0.25);
          box-shadow:0 10px 28px rgba(0,0,0,0.7);
          margin-bottom:18px;
        }
        label {
          display:block;
          font-size:0.85rem;
          margin-bottom:4px;
          color:#ccc;
        }
        input {
          width:100%;
          padding:8px 10px;
          border-radius:10px;
          border:1px solid rgba(255,255,255,0.25);
          background:rgba(0,0,0,0.4);
          color:#fff;
          margin-bottom:10px;
          font-size:0.85rem;
        }
        button {
          padding:10px 16px;
          border-radius:999px;
          border:none;
          cursor:pointer;
          font-size:0.85rem;
          text-transform:uppercase;
          letter-spacing:0.08em;
          background:#fff;
          color:#000;
        }
        table {
          width:100%;
          border-collapse:collapse;
          font-size:0.85rem;
        }
        th, td {
          padding:8px;
          border-bottom:1px solid rgba(255,255,255,0.08);
        }
        th {
          text-align:left;
          font-size:0.78rem;
          text-transform:uppercase;
          letter-spacing:0.08em;
          color:#bbb;
        }
        .back-btn {
          display:inline-block;
          margin-top:12px;
          padding:8px 16px;
          border-radius:999px;
          border:1px solid rgba(255,255,255,0.2);
          text-decoration:none;
          color:#fff;
          font-size:0.85rem;
          letter-spacing:0.08em;
          text-transform:uppercase;
        }
        .toast {
          position:fixed;
          bottom:20px;
          right:20px;
          background:rgba(0,0,0,0.85);
          color:#fff;
          padding:8px 14px;
          border-radius:999px;
          font-size:0.8rem;
          display:none;
        }
        .stats {
          font-size:0.9rem;
          margin-top:6px;
        }
        .stats .stat-value {
          font-weight:700;
        }
      </style>
    </head>
    <body>
      <div class="container">
        <h1>Box Office Sales</h1>
        <div class="subtitle">Generate on-site tickets with QR codes and track basic sales.</div>

        <div class="card">
          <label for="soldTo">Sold To (Name / Location)</label>
          <input id="soldTo" placeholder="Walk-up guest / VIP table / etc">

          <label for="amount">Amount (BBD)</label>
          <input id="amount" type="number" step="0.01" placeholder="Optional">

          <div style="margin-top:8px;">
            <button onclick="generateTicket()">Generate Ticket</button>
          </div>
          <div class="stats">
            Next ticket: <span id="nextNumber">BOX-0001</span> • Total generated: <span class="stat-value">0</span>
          </div>
        </div>

        <div class="card">
          <h2 style="margin-top:0;margin-bottom:12px;font-size:1.1rem;">Recent Sales</h2>
          <table id="salesTable">
            <thead>
              <tr>
                <th>Ticket ID</th>
                <th>Sold To</th>
                <th>Amount</th>
                <th>Time</th>
              </tr>
            </thead>
            <tbody id="salesBody">
            </tbody>
          </table>
        </div>

        <a href="/management-hub?key=${encodeURIComponent(
          MANAGEMENT_PIN
        )}" class="back-btn">← Back to Management Hub</a>
      </div>

      <div id="toast" class="toast"></div>
      ${themeScript()}
      <script>
        const MANAGEMENT_PIN = "${MANAGEMENT_PIN}";

        function generateTicket() {
          const soldTo = document.getElementById("soldTo").value.trim() || "Box Office";
          const amount = document.getElementById("amount").value || null;

          fetch('/api/box-office-generate', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ key: MANAGEMENT_PIN, soldTo, amount })
          })
          .then(r => r.json())
          .then(data => {
            if (data.success) {
              showToast('✓ Ticket ' + data.ticketId + ' generated');
              document.getElementById("soldTo").value = "";
              document.getElementById("amount").value = "";
              loadSales();
              window.open(data.qrUrl, '_blank');
            } else {
              showToast('Error: ' + (data.error || 'Unknown'));
            }
          })
          .catch(e => showToast('Error: ' + e.message));
        }

        function showToast(msg) {
          const toast = document.getElementById("toast");
          toast.textContent = msg;
          toast.style.display = "block";
        }

        function loadSales() {
          fetch(\`/api/box-office-sales?key=\${encodeURIComponent(MANAGEMENT_PIN)}\`)
            .then(r => r.json())
            .then(data => {
              const tbody = document.getElementById("salesBody");
              tbody.innerHTML = "";
              if (data.sales.length === 0) {
                tbody.innerHTML = '<tr><td colspan="4" style="text-align:center;color:#999;">No sales yet</td></tr>';
              } else {
                data.sales.forEach(sale => {
                  const time = new Date(sale.timestamp).toLocaleTimeString();
                  const row = \`<tr>
                    <td><strong>\${sale.ticketId}</strong></td>
                    <td>\${sale.soldTo}</td>
                    <td>\${sale.amount ? 'BD$' + sale.amount : '--'}</td>
                    <td style="font-size:0.85rem;">\${time}</td>
                  </tr>\`;
                  tbody.innerHTML += row;
                });
              }
              document.getElementById("nextNumber").textContent = "BOX-" + String(data.nextNumber).padStart(4, "0");
              document.querySelector(".stats .stat-value").textContent = data.total;
            });
        }

        loadSales();
        setInterval(loadSales, 10000);
      </script>
    </body>
    </html>`);
});

// ------------------------------------------------------
// ------------------------------------------------------
// GIVEAWAY / PRIZE DRAW
// ------------------------------------------------------
app.post("/api/draw-winner", (req, res) => {
  const { key, prizeDescription = "Mystery Prize" } = req.body || {};
  if (!(key === STAFF_PIN || isMgmtAuthorizedReq(req))) {
    return res.status(403).json({ error: "Invalid PIN" });
  }

  // Collect all USED tickets
  const usedTickets = [];
  for (const [token, record] of tickets.entries()) {
    if (record.status === "used") {
      usedTickets.push({ token, id: record.id });
    }
  }

  if (usedTickets.length === 0) {
    return res.json({ error: "No used tickets to draw from" });
  }

  // Pick random winner
  const winner = usedTickets[Math.floor(Math.random() * usedTickets.length)];

  // Look up a name for this ticket (guest entry or box office)
  const winnerName = getNameForTicketId(winner.id) || null;

  const drawEvent = {
    prizeId: `PRIZE-${Date.now()}`,
    winnerTicketId: winner.id,
    winnerName,                      // 🔹 store name with the draw
    timestamp: new Date().toISOString(),
    prizeDescription,
  };

  giveawayEvents.draws.push(drawEvent);

  res.json({
    success: true,
    winner: winner.id,
    winnerToken: winner.token,
    winnerName: winnerName,          // 🔹 send name back to the page
    prize: prizeDescription,
    totalDraws: giveawayEvents.draws.length,
  });
});

app.get("/api/giveaway-history", (req, res) => {
  const { key } = req.query;
  if (!(key === STAFF_PIN || isMgmtAuthorizedReq(req))) {
    return res.json({ draws: [], total: 0 });
  }

  res.json({
    draws: giveawayEvents.draws.slice(-50).reverse(),
    total: giveawayEvents.draws.length,
  });
});

app.get("/giveaway", (req, res) => {
  const { key } = req.query;
  if (!(key === STAFF_PIN || isMgmtAuthorizedReq(req))) {
    return res.redirect("/staff");
  }

  res.send(`<!doctype html>
  <html>
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width,initial-scale=1" />
    <title>Prize Draw</title>
    <style>
      body {
        margin:0;
        padding:16px;
        min-height:100vh;
        font-family: system-ui, -apple-system, BlinkMacSystemFont, sans-serif;
        background:
          radial-gradient(circle at top left, rgba(255,64,129,0.25), transparent 55%),
          radial-gradient(circle at bottom right, rgba(255,23,68,0.35), transparent 55%),
          #050007;
        color:#f5f5f5;
      }
      .container {
        max-width:600px;
        margin:0 auto;
      }
      h1 {
        margin-top:0;
        font-size:2rem;
        background: linear-gradient(120deg, #ffb300, #ff4081, #ff1744);
        -webkit-background-clip: text;
        color: transparent;
      }
      .subtitle {
        font-size:0.9rem;
        color:#aaa;
        margin-bottom:14px;
      }
      .card {
        background: radial-gradient(circle at top, #220018, #070008 60%);
        border-radius:18px;
        padding:16px 16px 14px;
        box-shadow:
          0 0 0 1px rgba(255,64,129,0.25),
          0 16px 40px rgba(0,0,0,0.9);
        margin-bottom:16px;
      }
      label {
        display:block;
        font-size:0.85rem;
        margin-bottom:6px;
      }
      input {
        width:100%;
        padding:10px 12px;
        border-radius:999px;
        border:1px solid rgba(255,64,129,0.6);
        background:rgba(3,0,5,0.9);
        color:#f5f5f5;
        outline:none;
      }
      input:focus {
        box-shadow:0 0 0 1px rgba(255,183,77,0.9),
                   0 0 16px rgba(255,64,129,0.6);
      }
      button {
        margin-top:10px;
        padding:10px 16px;
        border-radius:999px;
        border:none;
        cursor:pointer;
        background:linear-gradient(135deg,#ff1744,#ff4081);
        color:#fff;
        font-weight:700;
        text-transform:uppercase;
        letter-spacing:0.08em;
        font-size:0.85rem;
      }
      button:hover { filter:brightness(1.1); }

      .winner-box {
        margin-top:12px;
        padding:12px;
        border-radius:12px;
        background:rgba(76,175,80,0.15);
        border:1px solid rgba(76,175,80,0.4);
        display:none;
      }
      .winner-box.show { display:block; }

      table {
        width:100%;
        border-collapse:collapse;
        font-size:0.85rem;
      }
      th, td {
        padding:8px;
        border-bottom:1px solid rgba(255,255,255,0.08);
      }
      th {
        text-align:left;
        font-size:0.78rem;
        text-transform:uppercase;
        letter-spacing:0.08em;
        color:#bbb;
      }
      .empty {
        text-align:center;
        color:#888;
      }
      .back-btn {
        display:inline-block;
        margin-top:12px;
        padding:8px 16px;
        border-radius:999px;
        border:1px solid rgba(255,255,255,0.2);
        text-decoration:none;
        color:#fff;
        font-size:0.85rem;
        letter-spacing:0.08em;
        text-transform:uppercase;
      }
    </style>
  </head>
  <body>
    <div class="container">
      <h1>Prize Draw</h1>
      <div class="subtitle">Randomly select winners from used tickets.</div>

      <div class="card">
        <label for="prize">Prize Description</label>
        <input id="prize" placeholder="e.g. Bottle Giveaway, VIP Upgrade">

        <button onclick="drawWinner()">Draw Winner</button>

        <div id="winnerBox" class="winner-box">
          <div><strong>Winning Ticket:</strong> <span id="winnerTicket"></span></div>
          <div><strong>Guest:</strong> <span id="winnerName"></span></div>      <!-- 🔹 NEW -->
          <div><strong>Prize:</strong> <span id="winnerPrize"></span></div>
        </div>
      </div>

      <div class="card">
        <h2 style="margin-top:0;margin-bottom:12px;font-size:1.1rem;">Recent Draws</h2>
        <table>
          <thead>
            <tr>
              <th>Ticket</th>
              <th>Guest</th>     <!-- 🔹 NEW -->
              <th>Prize</th>
              <th>Time</th>
            </tr>
          </thead>
          <tbody id="historyBody"></tbody>
        </table>
      </div>

      <a href="/management-hub?key=${encodeURIComponent(MANAGEMENT_PIN)}" class="back-btn">
        ← Back to Management Hub
      </a>
    </div>

    ${themeScript()}
    <script>
      const STAFF_PIN = "${STAFF_PIN}";

      function drawWinner() {
        const prize = document.getElementById("prize").value.trim() || "Mystery Prize";
        fetch('/api/draw-winner', {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ key: STAFF_PIN, prizeDescription: prize })
        })
          .then(r => r.json())
          .then(data => {
            if (data.error) {
              alert('Error: ' + data.error);
              return;
            }

            document.getElementById("winnerTicket").textContent = data.winner;
            document.getElementById("winnerPrize").textContent = data.prize;
            document.getElementById("winnerName").textContent =
              data.winnerName || "(no name on file)";   // 🔹 show name

            document.getElementById("winnerBox").classList.add("show");

            setTimeout(() => refreshHistory(), 1000);
          })
          .catch(err => alert('Error: ' + err));
      }

      function refreshHistory() {
        fetch(\`/api/giveaway-history?key=\${encodeURIComponent(STAFF_PIN)}\`)
          .then(r => r.json())
          .then(data => {
            const tbody = document.getElementById("historyBody");

            if (!data.draws || data.draws.length === 0) {
              tbody.innerHTML = '<tr><td colspan="4" class="empty">No draws yet</td></tr>';
              return;
            }

            tbody.innerHTML = data.draws.map(draw => {
              const time = new Date(draw.timestamp).toLocaleString();
              return \`<tr>
                <td><strong>\${draw.winnerTicketId}</strong></td>
                <td>\${draw.winnerName || "(no name on file)"}</td>
                <td>\${draw.prizeDescription}</td>
                <td style="font-size:0.85rem;">\${time}</td>
              </tr>\`;
            }).join('');
          });
      }

      refreshHistory();
    </script>
  </body>
  </html>`);
});


// ------------------------------------------------------
// IP SECURITY SUMMARY (unchanged core)
// ------------------------------------------------------
// ------------------------------------------------------
// IP SECURITY SUMMARY (staff-only page)
// ------------------------------------------------------

// NEW: /security page for staff – shows suspicious IPs + recent scans
app.get("/security", (req, res) => {
  const { key } = req.query;
  if (key !== STAFF_PIN) {
    return res.redirect("/staff");
  }

  // Basic stats
  const totalEvents = ipLogging.events.length;
  const invalidCount = (scanEvents.invalid || []).length;
  const duplicateCount = (scanEvents.duplicates || []).length;

  // Build per-IP stats from ipLogging.events
  const ipMap = new Map();

  for (const evt of ipLogging.events) {
    const ip = evt.ip || "unknown";
    if (!ipMap.has(ip)) {
      ipMap.set(ip, {
        ip,
        total: 0,
        invalid: 0,
        duplicate: 0,
        lastTs: 0,
        lastStatus: "unknown",
        lastTicket: ""
      });
    }
    const stats = ipMap.get(ip);
    stats.total++;
    if (evt.status === "invalid") stats.invalid++;
    if (evt.status === "duplicate") stats.duplicate++;
    if (evt.timestamp && evt.timestamp > stats.lastTs) {
      stats.lastTs = evt.timestamp;
      stats.lastStatus = evt.status || "unknown";
      stats.lastTicket = evt.ticketId || "";
    }
  }

  const ipStats = Array.from(ipMap.values());

  // Mark "suspicious": 3+ invalid/duplicate scans from same IP
  const suspiciousIPs = ipStats
    .filter(s => (s.invalid + s.duplicate) >= 3)
    .sort((a, b) => (b.invalid + b.duplicate) - (a.invalid + a.duplicate));

  const recentScans = ipLogging.events.slice(-50).reverse(); // newest first

  const suspiciousRowsHtml = suspiciousIPs.length === 0
    ? '<tr><td colspan="5" class="empty">No suspicious IP activity detected yet.</td></tr>'
    : suspiciousIPs.map(s => {
        const last = s.lastTs ? new Date(s.lastTs).toLocaleString() : "--";
        const riskScore = s.invalid + s.duplicate;
        const riskLabel = riskScore >= 8 ? "HIGH" : (riskScore >= 4 ? "MED" : "LOW");
        return `
          <tr>
            <td><strong>${s.ip}</strong></td>
            <td>${s.total}</td>
            <td>${s.invalid}</td>
            <td>${s.duplicate}</td>
            <td>
              <span class="pill pill-${riskLabel.toLowerCase()}">${riskLabel}</span>
              <div class="last-line">${last}</div>
            </td>
          </tr>
        `;
      }).join("");

  const recentRowsHtml = recentScans.length === 0
    ? '<tr><td colspan="4" class="empty">No scans logged yet.</td></tr>'
    : recentScans.map(evt => {
        const time = evt.timestamp ? new Date(evt.timestamp).toLocaleString() : "--";
        let badgeClass = "status";
        if (evt.status === "valid") badgeClass += " status-valid";
        else if (evt.status === "invalid") badgeClass += " status-invalid";
        else if (evt.status === "duplicate") badgeClass += " status-duplicate";

        return `
          <tr>
            <td><strong>${evt.ticketId || "--"}</strong></td>
            <td class="nowrap">${evt.ip || "unknown"}</td>
            <td><span class="${badgeClass}">${evt.status || "?"}</span></td>
            <td class="nowrap" style="font-size:0.8rem;">${time}</td>
          </tr>
        `;
      }).join("");

  res.send(`<!DOCTYPE html>
  <html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>AURA Security Monitor</title>
    <style>
      ${themeCSSRoot()}

      * { box-sizing: border-box; }

      body {
        margin: 0;
        padding: 16px;
        font-family: system-ui, -apple-system, BlinkMacSystemFont, sans-serif;
        background:
          radial-gradient(circle at top left, rgba(255,64,129,0.25), transparent 55%),
          radial-gradient(circle at bottom right, rgba(255,23,68,0.35), transparent 55%),
          var(--bg-dark);
        color: #f5f5f5;
        display: flex;
        justify-content: center;
        min-height: 100vh;
      }

      .card {
        width: 100%;
        max-width: 640px;
        background: radial-gradient(circle at top, #220018, #070008 60%);
        border-radius: 24px;
        padding: 20px 18px 18px;
        border: 1px solid rgba(255,64,129,0.4);
        box-shadow: 0 20px 50px rgba(0,0,0,0.95);
        position: relative;
        overflow: hidden;
      }

      .card::before {
        content: "";
        position: absolute;
        inset: -40%;
        background:
          radial-gradient(circle at 0% 0%, rgba(255,255,255,0.07), transparent 40%),
          radial-gradient(circle at 100% 100%, rgba(255,64,129,0.25), transparent 55%);
        opacity: 0.8;
        pointer-events: none;
      }

      .card-inner { position: relative; z-index: 1; }

      .title-row {
        display:flex;
        justify-content:space-between;
        align-items:flex-start;
        gap:12px;
        margin-bottom:10px;
      }

      h1 {
        margin: 0;
        font-size: 1.45rem;
        background: linear-gradient(120deg,#ffb300,#ff4081,#ff1744);
        -webkit-background-clip:text;
        color: transparent;
      }

      .subtitle {
        font-size: 0.86rem;
        color: var(--text-muted);
        margin-bottom: 14px;
      }

      .badge {
        padding:6px 10px;
        border-radius:999px;
        border:1px solid rgba(255,255,255,0.25);
        font-size:0.7rem;
        letter-spacing:0.14em;
        text-transform:uppercase;
        white-space:nowrap;
      }

      .stat-row {
        display:flex;
        gap:10px;
        margin-bottom:14px;
        flex-wrap:wrap;
      }

      .stat {
        flex:1;
        min-width: 90px;
        padding:10px;
        border-radius:14px;
        background:rgba(0,0,0,0.45);
      }

      .stat-label {
        font-size:0.7rem;
        text-transform:uppercase;
        letter-spacing:0.08em;
        color:#aaa;
      }

      .stat-value {
        font-size:1.2rem;
        font-weight:700;
        margin-top:4px;
      }

      .stat-sub {
        font-size:0.75rem;
        color:#bbb;
      }

      .section-title {
        margin-top: 10px;
        margin-bottom: 6px;
        font-size: 0.95rem;
        letter-spacing: 0.06em;
        text-transform: uppercase;
        color: #ffb300;
      }

      .section-note {
        font-size:0.78rem;
        color:var(--text-muted);
        margin-bottom:6px;
      }

      .table-wrap {
        width:100%;
        overflow-x:auto;
        border-radius:14px;
        background:rgba(0,0,0,0.55);
        border:1px solid rgba(255,255,255,0.1);
      }

      table {
        width:100%;
        border-collapse:collapse;
        font-size:0.78rem;
      }

      th, td {
        padding:8px 10px;
        border-bottom:1px solid rgba(255,255,255,0.06);
        text-align:left;
      }

      th {
        font-size:0.75rem;
        text-transform:uppercase;
        letter-spacing:0.08em;
        color:#ffb300;
        background:rgba(255,255,255,0.02);
        position:sticky;
        top:0;
      }

      tr:last-child td {
        border-bottom:none;
      }

      .empty {
        text-align:center;
        color:var(--text-muted);
        font-size:0.8rem;
      }

      .status {
        padding:3px 8px;
        border-radius:999px;
        font-size:0.7rem;
        text-transform:uppercase;
        letter-spacing:0.06em;
        border:1px solid rgba(255,255,255,0.3);
      }
      .status-valid { border-color:#34c759; color:#34c759; }
      .status-invalid { border-color:#ff3b30; color:#ff3b30; }
      .status-duplicate { border-color:#ffb300; color:#ffb300; }

      .pill {
        display:inline-block;
        padding:3px 8px;
        border-radius:999px;
        font-size:0.7rem;
        text-transform:uppercase;
        letter-spacing:0.06em;
      }
      .pill-low { background:rgba(76,175,80,0.12); color:#4caf50; }
      .pill-med { background:rgba(255,193,7,0.12); color:#ffc107; }
      .pill-high { background:rgba(244,67,54,0.12); color:#f44336; }

      .last-line {
        font-size:0.7rem;
        color:var(--text-muted);
        margin-top:3px;
      }

      .back-link {
        display:inline-flex;
        align-items:center;
        gap:6px;
        margin-top:14px;
        font-size:0.8rem;
        text-decoration:none;
        color:#ffb300;
      }

      .back-link span {
        text-decoration:underline;
      }

      .nowrap { white-space:nowrap; }

      @media (max-width: 520px) {
        body { padding:12px; }
        .card { padding:16px 14px; border-radius:18px; }
        h1 { font-size:1.25rem; }
        .stat { padding:8px; }
        .stat-value { font-size:1.05rem; }
        .section-title { font-size:0.9rem; }
        th, td { padding:6px 8px; }
      }
    </style>
  </head>
  <body>
    <div class="card">
      <div class="card-inner">
        <div class="title-row">
          <div>
            <h1>AURA Security Monitor</h1>
            <div class="subtitle">
              Live view of suspicious IPs and recent staff scans.
            </div>
          </div>
          <div class="badge">STAFF SECURITY</div>
        </div>

        <div class="stat-row">
          <div class="stat">
            <div class="stat-label">Total Logged Scans</div>
            <div class="stat-value">${totalEvents}</div>
            <div class="stat-sub">All staff ticket activity</div>
          </div>
          <div class="stat">
            <div class="stat-label">Invalid Scans</div>
            <div class="stat-value">${invalidCount}</div>
            <div class="stat-sub">Unknown / forged tokens</div>
          </div>
          <div class="stat">
            <div class="stat-label">Duplicate Scans</div>
            <div class="stat-value">${duplicateCount}</div>
            <div class="stat-sub">Tickets tried twice</div>
          </div>
        </div>

        <div class="section-title">Suspicious IPs</div>
        <div class="section-note">
          IPs with <strong>3+ invalid or duplicate</strong> scans are flagged as suspicious.
        </div>
        <div class="table-wrap">
          <table>
            <thead>
              <tr>
                <th>IP</th>
                <th>Total</th>
                <th>Invalid</th>
                <th>Duplicate</th>
                <th>Risk / Last Activity</th>
              </tr>
            </thead>
            <tbody>
              ${suspiciousRowsHtml}
            </tbody>
          </table>
        </div>

        <div class="section-title" style="margin-top:14px;">Recent Scans (Last 50)</div>
        <div class="table-wrap">
          <table>
            <thead>
              <tr>
                <th>Ticket</th>
                <th>IP</th>
                <th>Status</th>
                <th>Time</th>
              </tr>
            </thead>
            <tbody>
              ${recentRowsHtml}
            </tbody>
          </table>
        </div>

        <a href="/staff?key=${encodeURIComponent(STAFF_PIN)}" class="back-link">
          ← <span>Back to Staff Tools</span>
        </a>
      </div>
    </div>

    ${themeScript()}
  </body>
  </html>`);
});

app.get("/staff-scanner", (req, res) => {
  const { key } = req.query;

  // Ensure scanner page isn't cached on mobile
  res.setHeader("Cache-Control", "no-store, no-cache, must-revalidate, proxy-revalidate");
  res.setHeader("Pragma", "no-cache");
  res.setHeader("Surrogate-Control", "no-store");

  if (key !== STAFF_PIN) {
    return res.redirect("/staff");
  }

  res.send(`
    <!doctype html>
    <html>
    <head>
      <meta charset="utf-8" />
      <meta name="viewport" content="width=device-width,initial-scale=1" />
      <title>AURA Staff Scanner</title>
      <style>
        :root {
          --bg-dark: #050007;
          --card-dark: #110011;
          --accent-cyan: #00bcd4;
          --accent-pink: #ff4081;
          --accent-red: #ff1744;
          --accent-gold: #ffb300;
          --text-main: #f5f5f5;
          --text-muted: #aaaaaa;
        }

        * { box-sizing:border-box; }

        body {
          margin:0;
          padding:0;
          background:
            radial-gradient(circle at top left, rgba(255,64,129,0.25), transparent 55%),
            radial-gradient(circle at bottom right, rgba(0,188,212,0.28), transparent 55%),
            var(--bg-dark);
          color:var(--text-main);
          font-family:system-ui, -apple-system, BlinkMacSystemFont, sans-serif;
          display:flex;
          justify-content:center;
          align-items:center;
          min-height:100vh;
          padding:16px;
        }

        .wrap {
          width:100%;
          max-width:720px;
        }

        .card {
          position:relative;
          width:100%;
          background:radial-gradient(circle at top, #200020, #050007 60%);
          border-radius:24px;
          box-shadow:
            0 0 0 1px rgba(0,188,212,0.4),
            0 22px 55px rgba(0,0,0,0.95);
          overflow:hidden;
        }

        .card::before {
          content:"";
          position:absolute;
          inset:-40%;
          background:
            radial-gradient(circle at 0% 0%, rgba(255,255,255,0.06), transparent 42%),
            radial-gradient(circle at 100% 100%, rgba(0,188,212,0.35), transparent 55%);
          opacity:0.8;
          pointer-events:none;
        }

        .card-inner {
          position:relative;
          z-index:1;
          padding:18px 16px 18px;
        }

        .header-row {
          display:flex;
          justify-content:space-between;
          align-items:flex-start;
          gap:12px;
          margin-bottom:10px;
        }

        .logo-row {
          display:flex;
          align-items:center;
          gap:10px;
          margin-bottom:6px;
        }

        .logo-img {
          height:30px;
          border-radius:8px;
          box-shadow:0 0 12px rgba(0,188,212,0.7);
        }

        .title-group {
          display:flex;
          flex-direction:column;
          gap:2px;
        }

        .title-main {
          font-size:1.1rem;
          font-weight:650;
        }

        .title-main span {
          background: linear-gradient(120deg, #ffb300, #ff4081, #ff1744);
          -webkit-background-clip:text;
          color:transparent;
        }

        .subtitle {
          font-size:0.85rem;
          color:var(--text-muted);
        }

        .badge {
          padding:6px 12px;
          border-radius:999px;
          border:1px solid rgba(0,188,212,0.7);
          font-size:0.7rem;
          letter-spacing:0.14em;
          text-transform:uppercase;
          color:var(--accent-cyan);
          background:rgba(0,0,0,0.45);
          align-self:flex-start;
        }

        .viewer {
          margin-top:14px;
          border-radius:20px;
          overflow:hidden;
          background:#000;
          box-shadow:0 16px 40px rgba(0,0,0,0.9);
          position:relative;
        }

        #reader {
          width:100%;
          height:420px;
        }

        @media (max-width: 600px) {
          #reader {
            height:360px;
          }
        }

        /* overlay frame like your design */
        #reader::before {
          content:"";
          position:absolute;
          inset:0;
          box-shadow:
            inset 0 0 0 999px rgba(0,0,0,0.45);
          pointer-events:none;
        }

        .frame {
          position:absolute;
          inset:14%;
          border-radius:12px;
          border:2px solid rgba(255,255,255,0.7);
          box-shadow:0 0 22px rgba(0,188,212,0.9);
          pointer-events:none;
        }

        .frame-corner {
          position:absolute;
          width:32px;
          height:32px;
          border:4px solid #fff;
        }

        .frame-corner.tl {
          top:-2px; left:-2px;
          border-right:none; border-bottom:none;
        }

        .frame-corner.tr {
          top:-2px; right:-2px;
          border-left:none; border-bottom:none;
        }

        .frame-corner.bl {
          bottom:-2px; left:-2px;
          border-right:none; border-top:none;
        }

        .frame-corner.br {
          bottom:-2px; right:-2px;
          border-left:none; border-top:none;
        }

        .hint {
          margin-top:12px;
          font-size:0.82rem;
          color:var(--text-muted);
          line-height:1.5;
        }

        .controls-row {
          margin-top:16px;
          display:flex;
          justify-content:space-between;
          flex-wrap:wrap;
          gap:10px;
          align-items:center;
        }

        .left-controls {
          display:flex;
          gap:8px;
          flex-wrap:wrap;
        }

        .btn {
          display:inline-flex;
          align-items:center;
          justify-content:center;
          padding:10px 16px;
          border-radius:999px;
          border:none;
          font-size:0.82rem;
          font-weight:600;
          letter-spacing:0.12em;
          text-transform:uppercase;
          cursor:pointer;
          text-decoration:none;
          white-space:nowrap;
          transition:filter 0.18s, transform 0.15s, box-shadow 0.18s;
        }

        .btn:hover {
          filter:brightness(1.08);
          transform:translateY(-1px);
        }

        .btn:active {
          transform:translateY(0);
        }

        .btn-back {
          background:rgba(0,0,0,0.75);
          color:var(--text-main);
          border:1px solid rgba(255,255,255,0.18);
        }

        .btn-rescan {
          background:linear-gradient(120deg,#00bcd4,#0097a7);
          color:#050005;
          box-shadow:0 10px 24px rgba(0,188,212,0.5);
        }

        .btn-open {
          background:linear-gradient(120deg,#ffb300,#ff9800);
          color:#050005;
          box-shadow:0 8px 20px rgba(255,183,77,0.4);
        }

        .btn-muted {
          background:rgba(0,0,0,0.7);
          color:var(--text-muted);
          border:1px solid rgba(255,255,255,0.18);
        }

        .status-bar {
          margin-top:14px;
          padding:10px 14px;
          border-radius:999px;
          background:rgba(0,0,0,0.7);
          border:1px solid rgba(0,188,212,0.35);
          font-size:0.8rem;
          color:var(--text-muted);
          display:flex;
          align-items:center;
          gap:8px;
        }

        .status-dot {
          width:10px;
          height:10px;
          border-radius:999px;
          background:rgba(244,67,54,0.7);
          box-shadow:0 0 10px rgba(244,67,54,0.8);
        }

        .status-bar.active .status-dot {
          background:#31c95a;
          box-shadow:0 0 10px rgba(49,201,90,0.9);
        }

        .status-label-strong {
          font-weight:600;
          color:var(--text-main);
        }

        .result-panel {
          margin-top:14px;
          border-radius:12px;
          padding:10px 12px;
          background:rgba(0,0,0,0.7);
          border:1px solid rgba(255,255,255,0.12);
          font-size:0.8rem;
          display:none;
        }

        .result-title {
          font-weight:600;
          margin-bottom:4px;
          font-size:0.82rem;
        }

        .result-body {
          word-break:break-all;
          color:var(--text-muted);
        }

        .result-actions {
          margin-top:8px;
          display:flex;
          flex-wrap:wrap;
          gap:8px;
        }

        .scan-footer {
          margin-top:16px;
          padding:10px 12px 6px;
          border-radius:14px;
          background:rgba(0,0,0,0.7);
          border:1px solid rgba(255,255,255,0.14);
          font-size:0.78rem;
          color:var(--text-muted);
          text-align:center;
          letter-spacing:0.12em;
          text-transform:uppercase;
        }

        .scan-footer span {
          color:var(--accent-cyan);
          font-weight:700;
        }

        @media (max-width:720px) {
          .card-inner { padding:16px 14px 18px; }
          .badge { font-size:0.65rem; }
          .title-main { font-size:1.0rem; }
          .subtitle { font-size:0.82rem; }
          .btn { font-size:0.8rem; padding:9px 14px; }
          .hint { font-size:0.78rem; }
        }
      </style>
      <script src="https://unpkg.com/html5-qrcode" type="text/javascript"></script>
    </head>
    <body>
      <div class="wrap">
        <div class="card">
          <div class="card-inner">
            <div class="header-row">
              <div>
                <div class="logo-row">
                  <img src="/aura-logo.png" class="logo-img" alt="AURA" />
                  <img src="/pop-logo.png" class="logo-img" alt="POP" />
                </div>
                <div class="title-group">
                  <div class="title-main"><span>AURA</span> Staff Scanner</div>
                  <div class="subtitle">Hearts &amp; Spades • Door &amp; VIP Control</div>
                </div>
              </div>
              <div class="badge">CAMERA MODE</div>
            </div>

            <div class="viewer">
              <div id="reader">
                <div class="frame">
                  <div class="frame-corner tl"></div>
                  <div class="frame-corner tr"></div>
                  <div class="frame-corner bl"></div>
                  <div class="frame-corner br"></div>
                </div>
              </div>
            </div>

            <div class="hint">
              • Point the camera at the <strong>QR code</strong> on the ticket.<br/>
              • Once detected, you'll see the decoded link and can open the staff ticket view.<br/>
              • If the wrong code is captured, tap <strong>Rescan</strong> to try again.
            </div>

            <div class="controls-row">
              <div class="left-controls">
                <a class="btn btn-back" href="/staff?key=${encodeURIComponent(STAFF_PIN)}">← Back to Staff</a>
                <button class="btn btn-rescan" id="rescanBtn">⟳ Rescan</button>
              </div>
              <button class="btn btn-open" id="openTicketBtn">Open Ticket</button>
            </div>

            <div class="status-bar" id="statusBar">
              <div class="status-dot"></div>
              <div>
                <div class="status-label-strong" id="scanStatus">Scanning… point at QR code.</div>
              </div>
            </div>

            <div class="result-panel" id="resultPanel">
              <div class="result-title" id="resultTitle">No scan yet</div>
              <div class="result-body" id="resultBody">Point the camera at the ticket QR.</div>
              <div class="result-actions">
                <button class="btn btn-rescan" id="discardBtn">Discard</button>
              </div>
            </div>

            <div class="scan-footer">
              SCANNING… <span>POINT AT QR CODE.</span>
            </div>
          </div>
        </div>
      </div>

      <script>
        const STAFF_KEY = "${STAFF_PIN}";
        let html5Qr = null;
        let currentCameraId = null;
        let lastDecoded = null;
        let lastToken = null;
        let lastSig = null;
        let scanning = false;

        const statusEl = document.getElementById('scanStatus');
        const panelEl = document.getElementById('resultPanel');
        const bodyEl = document.getElementById('resultBody');
        const titleEl = document.getElementById('resultTitle');
        const rescanBtn = document.getElementById('rescanBtn');
        const openBtn = document.getElementById('openTicketBtn');
        const discardBtn = document.getElementById('discardBtn');
        const statusBar = document.getElementById('statusBar');

        function setStatus(text, active) {
          statusEl.textContent = text;
          if (active) {
            statusBar.classList.add('active');
          } else {
            statusBar.classList.remove('active');
          }
        }

        function parseTicketFromText(decodedText) {
          let token = null;
          let sig = null;
          let raw = decodedText.trim();

          // Try parse as URL first
          try {
            const url = new URL(raw);
            const t = url.searchParams.get("token");
            const s = url.searchParams.get("sig");
            if (t) token = t;
            if (s) sig = s;
          } catch (e) {
            // Not a full URL, ignore
          }

          // If not found, look for token= in raw text
          if (!token) {
            const idx = raw.indexOf("token=");
            if (idx !== -1) {
              token = raw.substring(idx + 6).split("&")[0];
            }
          }

          // If still not found, maybe whole text *is* the token
          if (!token && raw.length >= 16) {
            token = raw;
          }

          // Try to find sig= too
          if (!sig) {
            const idx2 = raw.indexOf("sig=");
            if (idx2 !== -1) {
              sig = raw.substring(idx2 + 4).split("&")[0];
            }
          }

          return { token, sig };
        }

        function onScanSuccess(decodedText, decodedResult) {
          lastDecoded = decodedText;
          const { token, sig } = parseTicketFromText(decodedText);
          lastToken = token || null;
          lastSig = sig || null;

          if (!lastToken) {
            titleEl.textContent = "QR Detected (No Ticket Token)";
            bodyEl.textContent = decodedText;
            panelEl.style.display = "block";
            setStatus("QR found but no ticket token. Try again.", false);
            return;
          }

          titleEl.textContent = "QR Detected – Ready to open";
          bodyEl.textContent = lastDecoded;
          panelEl.style.display = "block";
          setStatus("Ticket detected. Tap 'Open Ticket'.", true);

          // Pause scanning visually
          stopScanner(false);
        }

        function onScanFailure(error) {
          // Ignore continuous errors, keep scanning
        }

        function startScanner() {
          if (scanning) return;
          setStatus("Opening camera…", false);

          if (!html5Qr) {
            html5Qr = new Html5Qrcode("reader");
          }

          Html5Qrcode.getCameras().then(cameras => {
            if (!cameras || cameras.length === 0) {
              setStatus("No camera found on this device.", false);
              return;
            }

            // Prefer back camera if available
            if (!currentCameraId) {
              const backCam = cameras.find(c => /back|rear|environment/gi.test(c.label));
              currentCameraId = backCam ? backCam.id : cameras[0].id;
            }

            html5Qr.start(
              currentCameraId,
              {
                fps: 10,
                qrbox: { width: 250, height: 250 },
                aspectRatio: 1.0,
              },
              onScanSuccess,
              onScanFailure
            ).then(() => {
              scanning = true;
              setStatus("Scanning… point at QR code.", true);
            }).catch(err => {
              console.error("Error starting camera", err);
              setStatus("Unable to start camera. Check permissions.", false);
            });
          }).catch(err => {
            console.error("Error getting cameras", err);
            setStatus("Camera access denied or unavailable.", false);
          });
        }

        function stopScanner(clearFrame = true) {
          if (!html5Qr || !scanning) return;
          html5Qr.stop().then(() => {
            scanning = false;
            if (clearFrame) {
              setStatus("Scanner stopped.", false);
            }
          }).catch(err => {
            console.error("Error stopping scanner", err);
          });
        }

        rescanBtn.addEventListener('click', () => {
          panelEl.style.display = "none";
          lastDecoded = null;
          lastToken = null;
          lastSig = null;
          setStatus("Rescanning… point at QR code.", false);
          stopScanner(false);
          startScanner();
        });

        discardBtn.addEventListener('click', () => {
          panelEl.style.display = "none";
          lastDecoded = null;
          lastToken = null;
          lastSig = null;
          setStatus("Scan discarded. Rescanning…", false);
          stopScanner(false);
          startScanner();
        });

        openBtn.addEventListener('click', () => {
          if (!lastToken) {
            setStatus("No ticket token to open. Rescan.", false);
            return;
          }

          let url = "/ticket?token=" + encodeURIComponent(lastToken) +
                    "&staff=1&key=" + encodeURIComponent(STAFF_KEY);

          if (lastSig) {
            url += "&sig=" + encodeURIComponent(lastSig);
          }

          window.location.href = url;
        });

        // Kill any service workers to avoid caching issues on scanner page
        if ('serviceWorker' in navigator) {
          navigator.serviceWorker.getRegistrations().then(regs => {
            regs.forEach(r => r.unregister());
          });
        }

        // Auto-start scanner on load
        window.addEventListener('load', () => {
          startScanner();
        });

        // Clean up on unload
        window.addEventListener('beforeunload', () => {
          stopScanner(true);
        });
      </script>
    </body>
    </html>
  `);
});

// ------------------------------------------------------
// NEW: MANAGEMENT HUB (Dashboard, Analytics, Allocations, Prize Draw, Logs)
// ------------------------------------------------------
app.get("/management-hub", (req, res) => {
  // Allow access if query key matches OR cookie mgmtAuth is present and valid
  if (!isMgmtAuthorizedReq(req)) {
    return res.redirect("/staff?key=" + encodeURIComponent(STAFF_PIN));
  }

  // Determine manager name to display: prefer query, then cookie
  const cookies = parseCookies(req);
  const managerName = (req.query && req.query.name) ? req.query.name : (cookies.mgmtName || '');

  res.send(`<!DOCTYPE html>
    <html>
    <head>
      <meta charset="UTF-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1" />
      <title>Management Hub</title>
      <style>
        ${themeCSSRoot()}
        body {
          margin:0;
          padding:18px;
          font-family:system-ui,-apple-system,BlinkMacSystemFont,sans-serif;
          background:#050007;
          color:#f5f5f5;
          display:flex;
          justify-content:center;
          min-height:100vh;
        }
        .card {
          width:100%;
          max-width:700px;
          background:radial-gradient(circle at top,#220018,#070008 60%);
          border-radius:24px;
          padding:24px 22px 20px;
          box-shadow:
            0 0 0 1px rgba(255,64,129,0.35),
            0 20px 50px rgba(0,0,0,0.95);
          position:relative;
          overflow:hidden;
        }
        .card::before {
          content:"";
          position:absolute;
          inset:-40%;
          background:
            radial-gradient(circle at 0% 0%,rgba(255,255,255,0.06),transparent 40%),
            radial-gradient(circle at 100% 100%,rgba(255,64,129,0.18),transparent 50%);
          opacity:0.7;
          pointer-events:none;
        }
        .card-inner { position:relative; z-index:1; }
        .header-row {
          display:flex;
          justify-content:space-between;
          align-items:center;
          gap:12px;
          margin-bottom:12px;
        }
        .logo-row {
          display:flex;
          align-items:center;
          gap:10px;
        }
        .logo-img {
          height:32px;
          border-radius:8px;
          box-shadow:0 0 12px rgba(255,64,129,0.55);
        }
        .title-main {
          font-size:1.4rem;
          font-weight:650;
        }
        .title-main span {
          background:linear-gradient(120deg,#ffb300,#ff4081,#ff1744);
          -webkit-background-clip:text;
          color:transparent;
        }
        .badge {
          padding:5px 10px;
          border-radius:999px;
          border:1px solid rgba(255,64,129,0.6);
          font-size:0.7rem;
          letter-spacing:0.12em;
          text-transform:uppercase;
          color:#ccc;
        }
        .subtitle {
          font-size:0.9rem;
          color:#aaa;
          margin-bottom:16px;
        }
        .button-grid {
          display:grid;
          grid-template-columns:repeat(auto-fit,minmax(180px,1fr));
          gap:12px;
        }
        .action-button {
          padding:14px 14px;
          border-radius:16px;
          border:none;
          cursor:pointer;
          font-weight:700;
          letter-spacing:0.08em;
          text-transform:uppercase;
          font-size:0.8rem;
          text-align:left;
          display:flex;
          flex-direction:column;
          gap:4px;
          box-shadow:0 12px 28px rgba(0,0,0,0.8);
          color:#fff;
        }
        .action-button span.label {
          font-size:0.8rem;
          font-weight:700;
        }
        .action-button span.desc {
          font-size:0.78rem;
          opacity:0.85;
        }
        .btn-dashboard { background:linear-gradient(135deg,#ffc107,#ff9800); color:#000; }
        .btn-analytics { background:linear-gradient(135deg,#4caf50,#388e3c); }
        .btn-alloc { background:linear-gradient(135deg,#9c27b0,#6a1b9a); }
        .btn-giveaway { background:linear-gradient(135deg,#e91e63,#ad1457); }
        .btn-prizeentries { background:linear-gradient(135deg,#00bcd4,#0097a7); }
        .btn-stafflog { background:linear-gradient(135deg,#03a9f4,#0288d1); }
        .btn-guestlog { background:linear-gradient(135deg,#ff5722,#e64a19); }
        .btn-mailing {
          background: linear-gradient(135deg,#8e24aa,#ff6ec4);
          border: 1px solid rgba(255,255,255,0.18);
        }

        .btn-qrfiles {
          background: linear-gradient(135deg, #ff1744, #ff4081);
          border: 1px solid rgba(255, 23, 68, 0.6);
          color: #fff;
        }

        .btn-generate {
          background: linear-gradient(135deg, #7b1fa2, #9c27b0);
          border: 1px solid rgba(155, 89, 182, 0.5);
          color: #fff !important;
        }



        .theme-toggle {
          padding:6px 10px;
          border-radius:999px;
          border:1px solid rgba(255,255,255,0.25);
          background:rgba(0,0,0,0.3);
          color:#fff;
          font-size:0.75rem;
          cursor:pointer;
        }
        .staff-name {
          font-size:0.85rem;
          color:#ccc;
        }
        a.back-link {
          display:inline-block;
          margin-top:14px;
          font-size:0.8rem;
          color:#aaa;
          text-decoration:none;
        }
      </style>
    </head>
    <body>
      <div class="card">
        <div class="card-inner">
          <div class="header-row">
            <div class="logo-row">
              <img src="/aura-logo.png" class="logo-img" alt="AURA"/>
              <img src="/pop-logo.png" class="logo-img" alt="POP"/>
              <div>
                <div class="title-main"><span>AURA</span> Management Hub</div>
                <div class="staff-name">Manager: <span id="managerName">Unknown</span></div>
              </div>
            </div>
            <div>
              <div class="badge">MGMT PIN • POP!</div>
              <button class="theme-toggle" onclick="AURA_THEME.toggle()">☀ Light / Dark</button>
            </div>
          </div>

          <div class="subtitle">
            Central control for ticket performance, allocations, prize draws, staff activity, and guest scan logs.
          </div>

<div class="button-grid">
  <button class="action-button btn-dashboard" onclick="go('/dashboard')">
    <span class="label">📊 Dashboard</span>
    <span class="desc">Quick snapshot of total tickets and arrivals.</span>
  </button>
<a href="/staff/generate?key=${encodeURIComponent(MANAGEMENT_PIN)}"
   style="display:block;background:#ff4b9a;padding:12px 18px;
          margin-top:15px;border-radius:8px;color:black;
          font-weight:700;text-align:center;text-decoration:none;
          background:linear-gradient(135deg,#ff4b9a,#ffb347);">
  GENERATE TICKETS / QRCODES
</a>

  <button class="action-button btn-analytics" onclick="go('/live-analytics')">
    <span class="label">📈 Live Analytics</span>
    <span class="desc">Real-time check-in stats and last scans.</span>
    
</button>

<button class="action-button btn-qrfiles" onclick="go('/staff/qr-list')">
  <span class="label">📂 QR Files</span>
  <span class="desc">View / download generated QR PNGs.</span>
</button>

  <button class="action-button btn-alloc" onclick="go('/allocations')">

    <span class="label">📑 Allocations</span>
    <span class="desc">Track prefixes, sold vs unsold, export CSV.</span>
  </button>

  <!-- 🔹 NEW BUTTON -->
  <button class="action-button btn-alloc" onclick="go('/allocation-scanner')">
    <span class="label">📲 Allocation Scanner</span>
    <span class="desc">Scan / type tickets and assign to sellers.</span>
  </button>
  <!-- 🔹 END NEW BUTTON -->

  <button class="action-button btn-alloc" onclick="go('/allocation-log')">
  <span class="label">📒 Allocation Log</span>
  <span class="desc">See every ticket, seller, guest & status.</span>
</button>

  <button class="action-button btn-giveaway" onclick="go('/giveaway')">
    <span class="label">🎉 Prize Draw</span>
    <span class="desc">Run random winners from used tickets.</span>
  </button>
  <button class="action-button btn-prizeentries" onclick="go('/guest-prize-entries')">
    <span class="label">🎁 Guest Entries</span>
    <span class="desc">Prize draw entries from guest scans & draw winner.</span>
  </button>
   <button class="action-button btn-stafflog" onclick="go('/staff-log')">
     <span class="label">👥 Staff Log</span>
     <span class="desc">View staff login activity and usage.</span>
   </button>

   <button class="action-button btn-guestlog" onclick="go('/guest-scans')">
     <span class="label">🎫 Scan Log</span>
     <span class="desc">Guest ticket scans recorded in backend.</span>
   </button>

   <!-- NEW: Mailing list tile inside the grid -->
   <button class="action-button btn-mailing" onclick="go('/subscriber-log')">
     <span class="label">📬 Mailing List</span>
     <span class="desc">View guests who opted into POP / AURA updates.</span>
   </button>

 </div>

<div class="top-actions">
  <button class="pill-btn" onclick="reloadAllocations()">↻ Refresh</button>
  <button class="pill-btn" onclick="clearAllocationLog()">🧹 Clear Allocation Log</button>
</div>

 <div style="margin-top:18px;border-top:1px dashed rgba(255,255,255,0.04);padding-top:14px;">

  <h3 style="margin:0 0 8px 0;font-size:0.95rem;color:#ffd86b">Admin Tools</h3>

  <div style="display:flex;gap:8px;flex-wrap:wrap;align-items:center">
    <button
      class="action-button"
      style="background:linear-gradient(90deg,#ffd86b,#ffb300);padding:10px 12px;min-width:160px"
      onclick="adminExport()"
    >
      ⬇️ Export JSON
    </button>

    <label
      style="display:inline-block;background:linear-gradient(90deg,#00bcd4,#0097a7);padding:8px 12px;
             border-radius:12px;color:#081217;cursor:pointer;font-weight:700"
    >
      ⬆️ Import JSON
      <input
        id="adminImportFile"
        type="file"
        accept="application/json"
        style="display:none"
        onchange="adminImportFile(event)"
      />
    </label>

    <button
      class="action-button"
      style="background:linear-gradient(90deg,#ff5252,#e91e63);padding:10px 12px;min-width:160px"
      onclick="adminClearData()"
    >
    <!-- Allocation Hub -->
<a href="/allocation-hub?key=${encodeURIComponent(key)}" class="tool-card tool-purple">
  <div class="tool-title">🎟 Allocation Hub</div>
  <div class="tool-sub">Allocations, scanner, and allocation log.</div>
</a>

<!-- Log Hub -->
<a href="/logs-hub?key=${encodeURIComponent(key)}" class="tool-card tool-blue">
  <div class="tool-title">📘 Log Hub</div>
  <div class="tool-sub">Staff log, scan log, mailing list, cancelled tickets.</div>
</a>

<!-- Prize Hub -->
<a href="/prize-hub?key=${encodeURIComponent(key)}" class="tool-card tool-pink">
  <div class="tool-title">🎁 Prize Hub</div>
  <div class="tool-sub">Prize draws and guest entries.</div>
</a>

      🧨 Clear Data
    </button>

    <button
      class="action-button"
      style="background:linear-gradient(90deg,#ff9800,#ff5722);padding:10px 12px;min-width:210px"
      onclick="adminClearTestTickets()"
    >
      🧹 Clear Test Tickets / QRCodes
    </button>
  </div>

  <div id="adminMsg"
       style="margin-left:8px;margin-top:8px;color:#fff;font-size:0.9rem;opacity:0.85;">
  </div>

  <div style="margin-top:16px;">
    <label
      for="cancelCode"
      style="font-size:0.8rem;letter-spacing:0.08em;text-transform:uppercase;
             color:#bbb;display:block;margin-bottom:4px;"
    >
      Cancel Ticket / QR Code:
    </label>

    <div style="display:flex;gap:8px;flex-wrap:wrap;align-items:center;">
      <input
        id="cancelCode"
        placeholder="e.g. FRESH-001 or FRESH-001.png"
        style="flex:1 1 180px;min-width:0;border-radius:999px;border:1px solid rgba(255,255,255,0.2);
               padding:8px 12px;background:#140019;color:#fff;"
      />
      <button
        type="button"
        onclick="adminCancelTicket()"
        style="border-radius:999px;border:none;padding:8px 16px;background:#ff5252;
               color:#fff;font-weight:600;cursor:pointer;"
      >
        Cancel
      </button>
    </div>
  </div>
</div>


          <a href="/staff?key=${encodeURIComponent(
            STAFF_PIN
          )}" class="back-link">← Back to Staff Home</a>
        </div>
      </div>

           ${themeScript()}
      <script>
        const MGMT_KEY = "${MANAGEMENT_PIN}";
        const ALLOWED_MANAGERS = ["RAY","SHAWN","NIQUE","CHE"];

        function go(path) {
          window.location.href = path + "?key=" + encodeURIComponent(MGMT_KEY);
        }

        function isManagerName(name) {
          if (!name) return false;
          return ALLOWED_MANAGERS.includes(name.trim().toUpperCase());
        }

        function setAdminMsg(text, isError) {
          const el = document.getElementById("adminMsg");
          if (!el) return;
          el.textContent = text;
          if (isError) {
            el.style.color = "#ff8a80";
          } else {
            el.style.color = "#ffffff";
          }
        }

        async function adminExport() {
          try {
            const res = await fetch(
              "/admin/export?key=" + encodeURIComponent(MGMT_KEY)
            );
            if (!res.ok) throw new Error("Export failed (" + res.status + ")");
            const data = await res.json();
            const blob = new Blob([JSON.stringify(data, null, 2)], {
              type: "application/json",
            });
            const url = URL.createObjectURL(blob);
            const a = document.createElement("a");
            a.href = url;
            a.download =
              "aura-backup-" +
              new Date().toISOString().slice(0, 19).replace(/[:T]/g, "-") +
              ".json";
            document.body.appendChild(a);
            a.click();
            a.remove();
            URL.revokeObjectURL(url);
            setAdminMsg("Exported current data.");
          } catch (err) {
            console.error(err);
            setAdminMsg("Export error: " + err.message, true);
          }
        }

        async function adminImportFile(event) {
          const file = event.target.files && event.target.files[0];
          if (!file) return;

          if (
            !confirm(
              'Import data from "' +
                file.name +
                '"? This will overwrite current in-memory data.'
            )
          ) {
            event.target.value = "";
            return;
          }

          try {
            const text = await file.text();
            const payload = JSON.parse(text);

            const res = await fetch(
              "/admin/import?key=" + encodeURIComponent(MGMT_KEY),
              {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify(payload),
              }
            );
            const data = await res.json();
            if (!res.ok || !data.ok) {
              throw new Error(data.error || "Import failed");
            }
            setAdminMsg("Import complete. Refresh the page to see updates.");
          } catch (err) {
            console.error(err);
            setAdminMsg("Import error: " + err.message, true);
          } finally {
            event.target.value = "";
          }
        }

        // 🔴 CLEAR **ALL** DATA
        async function adminClearData() {
          if (
            !confirm(
              "This will CLEAR ALL tickets, logs and allocations from memory. Continue?"
            )
          )
            return;

          try {
            const res = await fetch(
              "/admin/clear?key=" + encodeURIComponent(MGMT_KEY),
              {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({ what: "all" }),
              }
            );
            const data = await res.json();
            if (!res.ok || !data.ok) {
              throw new Error(data.error || "Clear failed");
            }
            setAdminMsg("All tickets + logs cleared. Clean system ready.");
          } catch (err) {
            console.error(err);
            setAdminMsg("Clear error: " + err.message, true);
          }
        }

        // 🧹 CLEAR ONLY TEST TICKETS / TEST-*.PNG
        async function adminClearTestTickets() {
          if (
            !confirm(
              "Clear ONLY TEST tickets, their TEST-*.png QRs and related logs?"
            )
          )
            return;

          try {
            const res = await fetch(
              "/admin/clear?key=" + encodeURIComponent(MGMT_KEY),
              {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({ what: "testTickets" }),
              }
            );
            const data = await res.json();
            if (!res.ok || !data.ok) {
              throw new Error(data.error || "Clear failed");
            }
            setAdminMsg("Test tickets / QR codes cleared.");
          } catch (err) {
            console.error(err);
            setAdminMsg("Clear test error: " + err.message, true);
          }
        }

        // ❌ CANCEL A SINGLE TICKET
        async function adminCancelTicket() {
          const input = document.getElementById("cancelCode");
          if (!input) return;

          const raw = input.value.trim();
          if (!raw) {
            setAdminMsg(
              "Enter a ticket ID or QR filename to cancel.",
              true
            );
            return;
          }

          // Accept "FRESH-001" or "FRESH-001.png"
          const ticketId = raw.toUpperCase().endsWith(".PNG")
            ? raw.slice(0, -4)
            : raw;

          try {
            const res = await fetch(
              "/admin/cancel-ticket?key=" + encodeURIComponent(MGMT_KEY),
              {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({ ticketId }),
              }
            );
            const data = await res.json();
            if (!res.ok || !data.ok) {
              throw new Error(data.error || "Cancel failed");
            }
            setAdminMsg("Ticket " + ticketId + " cancelled.");
            input.value = "";
          } catch (err) {
            console.error(err);
            setAdminMsg("Cancel error: " + err.message, true);
          }
        }

        (function initManager() {
          const serverName = ${JSON.stringify(managerName || "")};
          const qs = new URLSearchParams(window.location.search);
          const providedName =
            qs.get("name") ||
            serverName ||
            sessionStorage.getItem("staffName") ||
            "";
          const managerSpan = document.getElementById("managerName");
          if (providedName) {
            managerSpan.textContent = providedName;
            sessionStorage.setItem("staffName", providedName);
          } else {
            managerSpan.textContent = "Restricted";
          }
        })();
      </script>

    </body>
    </html>`);
});

app.get("/allocation-hub", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.status(403).send("Unauthorized");
  }
  const { key } = req.query;
  res.send(`<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Allocation Hub — AURA</title>
  <style>
    ${themeCSSRoot()}
    body { margin:0;padding:16px;font-family:system-ui,sans-serif;background:#050007;color:#f5f5f5; }
    .page { max-width:900px;margin:0 auto; }
    h1 { margin:0 0 6px;font-size:20px; }
    .subtitle { margin:0 0 16px;font-size:12px;color:#aaa; }
    .grid { display:grid;grid-template-columns:repeat(auto-fit,minmax(220px,1fr));gap:12px; }
    .tile {
      border-radius:18px;
      padding:14px 14px 12px;
      background:radial-gradient(circle at top,#42205c,#140018 70%);
      border:1px solid rgba(255,255,255,0.09);
      text-decoration:none;
      color:#fff;
      display:block;
    }
    .tile-title { font-size:14px;font-weight:600;margin-bottom:4px; }
    .tile-sub { font-size:11px;color:#ccc; }
    .pill-link { border-radius:999px;padding:6px 12px;font-size:11px;text-decoration:none;display:inline-flex;align-items:center;gap:6px;background:linear-gradient(135deg,#ff5fa2,#ffb347);color:#050007;margin-bottom:14px; }
  </style>
</head>
<body>
  <div class="page">
    <h1>Allocation Hub</h1>
    <p class="subtitle">Quick access to seller allocations, scanner, and full allocation log.</p>

    <a class="pill-link" href="/management-hub?key=${encodeURIComponent(key || "")}">⬅ Back to Management Hub</a>

    <div class="grid">
      <a class="tile" href="/allocations?key=${encodeURIComponent(key || "")}">
        <div class="tile-title">📊 Allocation Summary</div>
        <div class="tile-sub">Seller totals: allocated vs sold vs unsold.</div>
      </a>
      <a class="tile" href="/allocation-scanner?key=${encodeURIComponent(key || "")}">
        <div class="tile-title">📱 Allocation Scanner</div>
        <div class="tile-sub">Scan or type tickets and assign to sellers.</div>
      </a>
      <a class="tile" href="/allocation-log?key=${encodeURIComponent(key || "")}">
        <div class="tile-title">📚 Allocation Log</div>
        <div class="tile-sub">Per-ticket breakdown with seller + guest info.</div>
      </a>
    </div>
  </div>
  ${themeScript()}
</body>
</html>`);
});


app.get("/cancelled-tickets-log", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.status(403).send("Unauthorized");
  }

  const { key } = req.query;

  res.send(`<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Cancelled Tickets Log — AURA</title>
  <style>
    ${themeCSSRoot()}
    body {
      margin:0;
      padding:16px;
      font-family: system-ui,-apple-system,BlinkMacSystemFont,sans-serif;
      background:#050007;
      color:#f5f5f5;
    }
    .page {
      max-width:1100px;
      margin:0 auto;
    }
    h1 {
      margin:0 0 4px;
      font-size:20px;
    }
    .subtitle {
      margin:0 0 16px;
      font-size:12px;
      color:#aaa;
    }
    .top-links {
      display:flex;
      flex-wrap:wrap;
      gap:8px;
      margin-bottom:14px;
    }
    .pill-link, .pill-btn {
      border-radius:999px;
      padding:6px 12px;
      font-size:11px;
      border:none;
      cursor:pointer;
      text-decoration:none;
      text-transform:uppercase;
      letter-spacing:0.09em;
      background:linear-gradient(135deg,#ff5fa2,#ffb347);
      color:#050007;
      display:inline-flex;
      align-items:center;
      gap:6px;
    }
    .section {
      margin-top:18px;
      border-radius:14px;
      background:rgba(10,0,20,0.9);
      border:1px solid rgba(255,255,255,0.05);
      padding:12px;
    }
    .section h2 {
      margin:0 0 8px;
      font-size:14px;
    }
    table {
      width:100%;
      border-collapse:collapse;
      font-size:11px;
    }
    th, td {
      padding:6px 5px;
      border-bottom:1px solid rgba(255,255,255,0.06);
      text-align:left;
    }
    th {
      font-size:10px;
      text-transform:uppercase;
      letter-spacing:0.09em;
      color:#ffcdd2;
    }
    @media (max-width:720px) {
      table { font-size:10px; }
      th,td { padding:4px 3px; }
    }
  </style>
</head>
<body>
  <div class="page">
    <h1>Cancelled Tickets Log</h1>
    <p class="subtitle">
      All tickets whose status is <strong>cancelled</strong>, grouped by prefix (FRESH, HEART, BOX, etc).
    </p>

    <div class="top-links">
      <a class="pill-link" href="/management-hub?key=${encodeURIComponent(key || "")}">⬅ Back to Management Hub</a>
      <a class="pill-link" href="/dashboard?key=${encodeURIComponent(key || "")}">📊 Dashboard</a>
      <a class="pill-link" href="/allocation-log?key=${encodeURIComponent(key || "")}">📚 Allocation Log</a>
      <a class="pill-link" href="/allocations?key=${encodeURIComponent(key || "")}">🎟 Allocation Summary</a>
    </div>

    <div id="sections"></div>
  </div>

  ${themeScript()}

  <script>
    const MGMT_KEY = ${JSON.stringify(req.query.key || "")};

    async function loadCancelled() {
      const res = await fetch("/api/cancelled-tickets?key=" + encodeURIComponent(MGMT_KEY));
      const data = await res.json();
      if (!data.ok) {
        document.getElementById("sections").innerHTML = "<p>Unable to load cancelled tickets.</p>";
        return;
      }

      const root = document.getElementById("sections");
      root.innerHTML = "";

      const groups = data.groups || {};
      const prefixes = Object.keys(groups).sort();

      if (!prefixes.length) {
        root.innerHTML = "<p>No cancelled tickets.</p>";
        return;
      }

      prefixes.forEach(prefix => {
        const items = groups[prefix];
        const div = document.createElement("div");
        div.className = "section";

        const h = document.createElement("h2");
        h.textContent = prefix + " — " + items.length + " cancelled";
        div.appendChild(h);

        const table = document.createElement("table");
        table.innerHTML = \`
          <thead>
            <tr>
              <th>Ticket ID</th>
              <th>Type</th>
              <th>Guest</th>
              <th>Email</th>
              <th>Phone</th>
              <th>Seller</th>
              <th>Cancelled By</th>
              <th>Cancelled At</th>
            </tr>
          </thead>
          <tbody></tbody>
        \`;

        const tbody = table.querySelector("tbody");

        items.forEach(it => {
          const tr = document.createElement("tr");
          tr.innerHTML = \`
            <td>\${it.ticketId}</td>
            <td>\${it.ticketType || ""}</td>
            <td>\${it.guestName || ""}</td>
            <td>\${it.guestEmail || ""}</td>
            <td>\${it.guestPhone || ""}</td>
            <td>\${it.sellerName || ""}</td>
            <td>\${it.cancelledBy || ""}</td>
            <td>\${it.cancelledAt ? new Date(it.cancelledAt).toLocaleString() : ""}</td>
          \`;
          tbody.appendChild(tr);
        });

        div.appendChild(table);
        root.appendChild(div);
      });
    }

    loadCancelled();
  </script>
</body>
</html>`);
});

// MANAGEMENT: cancel a ticket (backend route)
app.post("/admin/cancel-ticket", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.status(403).json({ ok: false, error: "Unauthorized" });
  }

  const { code } = req.body || {};
  const clean = String(code || "").trim();
  if (!clean) return res.status(400).json({ ok: false, error: "Missing code" });

  // Allow both "HEART-001" and "HEART-001.png"
  const ticketId = clean.replace(/\.png$/i, "");

  let foundToken = null;
  let record = null;

  for (const [token, rec] of tickets.entries()) {
    if (rec.id === ticketId) {
      foundToken = token;
      record = rec;
      break;
    }
  }

  if (!foundToken) {
    return res.status(404).json({ ok: false, error: "Ticket not found" });
  }

  record.status = "cancelled";
  tickets.set(foundToken, record);
  saveTickets();

  return res.json({ ok: true, ticketId });

  record.status = "cancelled";

const cookies = parseCookies(req);
const cancelledBy = cookies.mgmtName || "management";

pushWithLimit(cancelledTicketsLog, {
  ticketId,
  token: foundToken,
  cancelledBy,
  source: "admin-cancel",
  timestamp: Date.now(),
}, 1000);

// ... existing response code

});


// ------------------------------------------------------
// MANAGEMENT: ALLOCATION SCANNER (QR → Seller/Box Office)
// ------------------------------------------------------
app.get("/allocation-scanner", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.redirect("/staff?key=" + encodeURIComponent(STAFF_PIN));
  }

  res.send(`<!DOCTYPE html>
  <html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Allocation Scanner</title>

    <style>
      ${themeCSSRoot()}

      body {
        margin:0;
        padding:16px;
        font-family:system-ui,-apple-system,BlinkMacSystemFont,sans-serif;
        background:#050007;
        color:#f5f5f5;
        display:flex;
        justify-content:center;
        min-height:100vh;
      }

      .wrap {
        width:100%;
        max-width:540px;
      }

      .card {
        width:100%;
        border-radius:22px;
        padding:18px 16px 20px;
        background:radial-gradient(circle at top,#260020,#050007 65%);
        box-shadow:
          0 0 0 1px rgba(255,64,129,0.32),
          0 22px 55px rgba(0,0,0,0.9);
      }

      .title-main {
        font-size:1.35rem;
        font-weight:700;
        margin-bottom:4px;
      }

      .title-main span {
        background:linear-gradient(90deg,#ff4081,#ffb300);
        -webkit-background-clip:text;
        color:transparent;
      }

      .subtitle {
        font-size:0.9rem;
        color:#c0c0c0;
        margin-bottom:14px;
      }

      label {
        display:block;
        font-size:0.8rem;
        text-transform:uppercase;
        letter-spacing:0.09em;
        color:#bbb;
        margin-bottom:4px;
      }

      input {
        width:100%;
        padding:10px 12px;
        border-radius:999px;
        border:1px solid rgba(255,255,255,0.18);
        background:rgba(5,0,10,0.75);
        color:#fff;
        font-size:0.94rem;
        outline:none;
        margin-bottom:10px;
      }

      input::placeholder {
        color:#777;
      }

      .pill-btn {
        display:inline-flex;
        align-items:center;
        justify-content:center;
        width:100%;
        padding:11px 14px;
        border-radius:999px;
        border:none;
        cursor:pointer;
        background:linear-gradient(90deg,#ff4081,#ffb300);
        color:#050007;
        font-weight:700;
        text-transform:uppercase;
        letter-spacing:0.12em;
        font-size:0.8rem;
        margin-top:4px;
      }

      .pill-btn:active { transform:scale(0.98); }

      .status {
        min-height:20px;
        margin-top:8px;
        font-size:0.82rem;
        color:#ccc;
      }
      .status.ok { color:#a5ffb5; }
      .status.err { color:#ff9ba0; }

      .back-link {
        display:inline-flex;
        align-items:center;
        gap:6px;
        margin-top:14px;
        font-size:0.86rem;
        text-decoration:none;
        color:#ffd86b;
      }

      .scanner-card {
        margin-top:10px;
        margin-bottom:14px;
        padding:12px 10px 14px;
        border-radius:18px;
        background:rgba(5,0,12,0.9);
        border:1px solid rgba(0,188,212,0.35);
      }

      .scanner-header {
        display:flex;
        align-items:center;
        justify-content:space-between;
        margin-bottom:6px;
        gap:10px;
      }

      .scanner-title {
        font-size:0.9rem;
        font-weight:600;
      }

      .scanner-badges {
        font-size:0.72rem;
        text-transform:uppercase;
        letter-spacing:0.1em;
        color:#8be9ff;
      }

      #qrReader {
        width:100%;
        max-width:360px;
        margin:8px auto 0;
        border-radius:16px;
        overflow:hidden;
      }

      .scanner-buttons {
        display:flex;
        gap:8px;
        margin-top:8px;
        flex-wrap:wrap;
      }

      .scanner-btn {
        flex:1 1 auto;
        border-radius:999px;
        border:none;
        padding:7px 10px;
        font-size:0.78rem;
        letter-spacing:0.09em;
        text-transform:uppercase;
        cursor:pointer;
      }

      .scanner-btn.primary {
        background:rgba(0,188,212,0.2);
        color:#8be9ff;
        border:1px solid rgba(0,188,212,0.6);
      }

      .scanner-btn.secondary {
        background:rgba(255,255,255,0.02);
        color:#ddd;
        border:1px solid rgba(255,255,255,0.18);
      }

      @media (max-width:480px) {
        .card { border-radius:18px; padding:16px 12px 18px; }
        .title-main { font-size:1.25rem; }
      }
    </style>
  </head>

  <body>
    <div class="wrap">
      <div class="card">
        <div class="title-main"><span>Allocation Scanner</span></div>
        <div class="subtitle">
          Scan or type a <strong>ticket ID</strong> and assign it to a seller or box office.
        </div>

        <!-- QR CAMERA SECTION -->
        <div class="scanner-card">
          <div class="scanner-header">
            <div class="scanner-title">QR Camera</div>
            <div class="scanner-badges">Scan Ticket QR</div>
          </div>
          <div id="qrReader"></div>
          <div class="scanner-buttons">
            <button type="button" id="startScanBtn" class="scanner-btn primary">Start Scanner</button>
            <button type="button" id="stopScanBtn" class="scanner-btn secondary">Stop</button>
          </div>
        </div>

        <!-- ALLOCATION FORM -->
        <label for="ticketId">Ticket ID</label>
        <input id="ticketId" placeholder="Scan or type NG-001, AURA-123, etc." autocomplete="off" />

        <label for="sellerName">Seller / Location</label>
        <input id="sellerName" placeholder="e.g. John, Box Office, VIP Booth" />

        <label for="sellerPhone">Phone (optional)</label>
        <input id="sellerPhone" />

        <label for="sellerEmail">Email (optional)</label>
        <input id="sellerEmail" />

        <button class="pill-btn" id="saveBtn">Save Allocation</button>
        <div id="status" class="status"></div>

        <a href="/allocations?key=${encodeURIComponent(MANAGEMENT_PIN)}" class="back-link">
          ← <span>Back to Allocations Overview</span>
        </a>
      </div>

      ${themeScript()}

      <script src="https://unpkg.com/html5-qrcode"></script>
      <script>
        const MANAGEMENT_PIN = "${MANAGEMENT_PIN}";

        const ticketIdInput = document.getElementById('ticketId');
        const sellerNameInput = document.getElementById('sellerName');
        const sellerPhoneInput = document.getElementById('sellerPhone');
        const sellerEmailInput = document.getElementById('sellerEmail');
        const statusEl = document.getElementById('status');
        const startBtn = document.getElementById('startScanBtn');
        const stopBtn = document.getElementById('stopScanBtn');
        const saveBtn = document.getElementById('saveBtn');

        let html5Qr = null;
        let scanning = false;
        let lastDecoded = null;

        function setStatus(msg, ok) {
          statusEl.textContent = msg || "";
          statusEl.className = "status " + (ok ? "ok" : "err");
        }

        function parseDecodedToTicket(decodedText) {
          decodedText = (decodedText || "").trim();

          // 1) try to find something like AURA-001, NG-123 etc.
          const idMatch = decodedText.match(/[A-Z]{2,}-\\d{3,6}/);
          if (idMatch) {
            return { ticketId: idMatch[0], token: null };
          }

          // 2) otherwise, look for ?token=XYZ in a URL
          const tokenMatch = decodedText.match(/[?&]token=([^&]+)/i);
          if (tokenMatch) {
            return { ticketId: null, token: tokenMatch[1] };
          }

          return { ticketId: null, token: null };
        }

        async function onScanSuccess(decodedText, decodedResult) {
          if (decodedText === lastDecoded) return;
          lastDecoded = decodedText;

          setStatus("QR scanned, resolving ticket…", true);

          const parsed = parseDecodedToTicket(decodedText);

          if (parsed.ticketId) {
            ticketIdInput.value = parsed.ticketId;
            setStatus("Ticket ID detected: " + parsed.ticketId, true);
            return;
          }

          if (parsed.token) {
            try {
              const res = await fetch(
                "/api/lookup-ticket-by-token?key=" +
                  encodeURIComponent(MANAGEMENT_PIN) +
                  "&token=" + encodeURIComponent(parsed.token)
              );
              const data = await res.json();
              if (data && data.ok && data.ticketId) {
                ticketIdInput.value = data.ticketId;
                setStatus("Ticket ID resolved: " + data.ticketId, true);
              } else {
                setStatus(data.error || "Could not resolve ticket from QR.", false);
              }
            } catch (e) {
              setStatus("Network error resolving ticket: " + e.message, false);
            }
            return;
          }

          setStatus("QR read, but no ticket ID or token found. Try again.", false);
        }

        function onScanFailure(error) {
          // Silent – we only care about successful decodes
        }

        function startScanner() {
          if (scanning) return;
          const el = document.getElementById("qrReader");
          if (!html5Qr) {
            html5Qr = new Html5Qrcode("qrReader");
          }

          Html5Qrcode.getCameras().then(cameras => {
            if (!cameras || cameras.length === 0) {
              setStatus("No camera found on this device.", false);
              return;
            }

            let camId = cameras[0].id;
            const backCam = cameras.find(c => /back|rear|environment/gi.test(c.label));
            if (backCam) camId = backCam.id;

            html5Qr.start(
              camId,
              { fps: 10, qrbox: { width: 240, height: 240 }, aspectRatio: 1.0 },
              onScanSuccess,
              onScanFailure
            ).then(() => {
              scanning = true;
              setStatus("Scanning… point at a ticket QR.", true);
            }).catch(err => {
              console.error("Error starting camera", err);
              setStatus("Unable to start camera. Check permissions.", false);
            });
          }).catch(err => {
            console.error("Error getting cameras", err);
            setStatus("Camera access denied or unavailable.", false);
          });
        }

        function stopScanner() {
          if (!html5Qr || !scanning) return;
          html5Qr.stop().then(() => {
            scanning = false;
            setStatus("Scanner stopped.", false);
          }).catch(err => {
            console.error("Error stopping scanner", err);
          });
        }

        async function saveAllocation() {
          const ticketId = ticketIdInput.value.trim().toUpperCase();
          const sellerName = sellerNameInput.value.trim();
          const sellerPhone = sellerPhoneInput.value.trim();
          const sellerEmail = sellerEmailInput.value.trim();

          if (!ticketId || !sellerName) {
            setStatus("Ticket ID and Seller / Location are required.", false);
            return;
          }

          try {
            const res = await fetch(
              "/api/allocate-ticket?key=" + encodeURIComponent(MANAGEMENT_PIN),
              {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({ ticketId, sellerName, sellerPhone, sellerEmail })
              }
            );
            const data = await res.json();
            if (data && data.ok) {
              setStatus("Allocation saved: " + ticketId + " → " + sellerName, true);
              ticketIdInput.value = "";
            } else {
              setStatus(data.error || "Error saving allocation.", false);
            }
          } catch (e) {
            setStatus("Network error: " + e.message, false);
          }
        }

        startBtn.addEventListener("click", startScanner);
        stopBtn.addEventListener("click", stopScanner);
        saveBtn.addEventListener("click", (e) => {
          e.preventDefault();
          saveAllocation();
        });

        window.addEventListener("beforeunload", () => {
          stopScanner();
        });
      </script>
    </div>
  </body>
  </html>`);
});

// ------------------------------------------------------
// STAFF LOG PAGE (Management Hub → Staff Log button)
// ------------------------------------------------------
app.get("/staff-log", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.redirect("/staff?key=" + encodeURIComponent(STAFF_PIN));
  }

  res.send(`<!DOCTYPE html>
    <html>
    <head>
      <meta charset="UTF-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1" />
      <title>Staff Activity Log</title>
      <style>
        ${themeCSSRoot()}
        body {
          margin:0;
          padding:18px;
          font-family:system-ui,-apple-system,BlinkMacSystemFont,sans-serif;
          background:#050007;
          color:#f5f5f5;
          display:flex;
          justify-content:center;
          min-height:100vh;
        }
        .card {
          width:100%;
          max-width:800px;
          background:radial-gradient(circle at top,#220018,#070008 60%);
          border-radius:16px;
          padding:18px;
          border:1px solid rgba(255,64,129,0.25);
          box-shadow:0 10px 28px rgba(0,0,0,0.7);
        }
        h1 {
          margin:0 0 8px;
          font-size:1.6rem;
          background:linear-gradient(120deg,#ffb300,#ff4081,#ff1744);
          -webkit-background-clip:text;
          color:transparent;
        }
        .subtitle {
          font-size:0.9rem;
          color:#aaa;
          margin-bottom:14px;
        }
        table {
          width:100%;
          border-collapse:collapse;
          font-size:0.85rem;
        }
        th, td {
          padding:8px;
          border-bottom:1px solid rgba(255,255,255,0.08);
        }
        th {
          text-align:left;
          font-size:0.78rem;
          text-transform:uppercase;
          letter-spacing:0.08em;
          color:#bbb;
        }
        .back-btn {
          display:inline-block;
          margin-top:12px;
          padding:8px 16px;
          border-radius:999px;
          border:1px solid rgba(255,255,255,0.2);
          text-decoration:none;
          color:#fff;
          font-size:0.85rem;
          letter-spacing:0.08em;
          text-transform:uppercase;
        }
        .empty {
          text-align:center;
          color:#888;
        }
      </style>
    </head>
    <body>
      <div class="card">
        <h1>Staff Activity Log</h1>
        <div class="subtitle">Track staff logins and system usage.</div>

        <table>
          <thead>
            <tr>
              <th>Name</th>
              <th>Role</th>
              <th>Action</th>
              <th>IP</th>
              <th>Time</th>
            </tr>
          </thead>
          <tbody id="logBody">
          </tbody>
        </table>

        <a href="/management-hub?key=${encodeURIComponent(
          MANAGEMENT_PIN
        )}" class="back-btn">← Back to Management Hub</a>
      </div>

      ${themeScript()}
      <script>
        const data = ${JSON.stringify(staffActivityLog.slice(-200))};
        const tbody = document.getElementById('logBody');
        if (!data.length) {
          tbody.innerHTML = '<tr><td colspan="5" class="empty">No staff activity logged yet.</td></tr>';
        } else {
          tbody.innerHTML = data.map(entry => {
            const time = new Date(entry.timestamp).toLocaleString();
            const role = entry.role ? (entry.role === 'management' ? '👔 MGMT' : '👥 STAFF') : '—';
            const ip = entry.ip || 'hidden';
            return \`<tr>
              <td>\${entry.name}</td>
              <td>\${role}</td>
              <td>\${entry.action}</td>
              <td style="font-size:0.85rem;">\${ip}</td>
              <td style="font-size:0.85rem;">\${time}</td>
            </tr>\`;
          }).join('');
        }
      </script>
    </body>
    </html>`);
});

// Helper: get the latest guest info for a ticket (from prize entries)
function getGuestInfoForTicket(ticketId) {
  if (!ticketId) return null;

  // Walk from the end so newest entries win
  for (let i = guestNameEntries.length - 1; i >= 0; i--) {
    const e = guestNameEntries[i];
    if (e && e.ticketId === ticketId) {
      return {
        name: (e.guestName || "").trim(),
        email: (e.guestEmail || "").trim(),
        phone: (e.guestPhone || "").trim()
      };
    }
  }
  return null;
}

// ------------------------------------------------------
// MANAGEMENT: Mailing List / Subscription Log
// ------------------------------------------------------
app.get("/subscriber-log", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.redirect("/staff?key=" + encodeURIComponent(STAFF_PIN));
  }

  // Only entries where subscribe === true
  const subs = guestNameEntries
    .filter(e => e && e.subscribe)
    .slice()
    .sort((a, b) => {
      const ta = Date.parse(a.timestamp || "") || 0;
      const tb = Date.parse(b.timestamp || "") || 0;
      return tb - ta; // newest first
    });

  res.send(`<!DOCTYPE html>
  <html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Mailing List Signups</title>
    <style>
      ${themeCSSRoot()}

      * { box-sizing:border-box; }

      body {
        margin:0;
        padding:16px;
        min-height:100vh;
        font-family:system-ui,-apple-system,BlinkMacSystemFont,sans-serif;
        background:#050007;
        color:#f5f5f5;
        display:flex;
        justify-content:center;
      }

      .wrap {
        width:100%;
        max-width:1024px;
      }

      .card {
        width:100%;
        border-radius:24px;
        padding:20px 20px 18px;
        background:radial-gradient(circle at top,#260020,#050007 65%);
        box-shadow:
          0 0 0 1px rgba(255,183,77,0.3),
          0 24px 60px rgba(0,0,0,0.9);
      }

      h1 {
        margin:0 0 4px;
        font-size:1.4rem;
        letter-spacing:0.06em;
        text-transform:uppercase;
        background:linear-gradient(135deg,#ffb300,#ff4081);
        -webkit-background-clip:text;
        color:transparent;
      }

      .subtitle {
        margin:0 0 16px;
        font-size:0.9rem;
        color:#f5e1b0;
      }

      table {
        width:100%;
        border-collapse:collapse;
        font-size:0.85rem;
      }

      th, td {
        padding:8px 10px;
        border-bottom:1px solid rgba(255,255,255,0.06);
        text-align:left;
        white-space:nowrap;
      }

      th {
        font-size:0.75rem;
        text-transform:uppercase;
        letter-spacing:0.12em;
        color:#ffb347;
        background:rgba(0,0,0,0.35);
        position:sticky;
        top:0;
        z-index:1;
      }

      tbody tr:nth-child(odd) td {
        background:rgba(255,255,255,0.01);
      }

      .scroll {
        max-height:70vh;
        overflow:auto;
        margin-top:8px;
      }

      .btn-back {
        display:inline-flex;
        align-items:center;
        gap:6px;
        margin-top:14px;
        padding:8px 14px;
        border-radius:999px;
        border:1px solid rgba(255,183,77,0.7);
        background:transparent;
        color:#ffeb99;
        font-size:0.8rem;
        text-decoration:none;
        text-transform:uppercase;
        letter-spacing:0.12em;
      }

      .btn-back:hover {
        background:rgba(255,183,77,0.15);
      }

      @media (max-width: 720px) {
        table { font-size:0.78rem; }
        th, td { padding:6px 6px; }
      }
    </style>
  </head>
  <body>
  <div class="top-actions">
  <button class="pill-btn" onclick="exportMailingList()">⬇ Export Mailing List</button>
</div>
    <div class="wrap">
      <div class="card">
        <h1>Mailing List Signups</h1>
        <div class="subtitle">
          All guest tickets that opted into the A.U.R.A / POP mailing list
          via the welcome / Mystery Prize form.
        </div>

        <div class="scroll">
          <table>
            <thead>
              <tr>
                <th>Ticket ID</th>
                <th>Guest Name</th>
                <th>Email</th>
                <th>Phone</th>
                <th>Time</th>
              </tr>
            </thead>
            <tbody>
              ${subs.map(e => `
                <tr>
                  <td><strong>${e.ticketId || ""}</strong></td>
                  <td>${(e.guestName || "").trim()}</td>
                  <td>${(e.guestEmail || "").trim()}</td>
                  <td>${(e.guestPhone || "").trim()}</td>
                  <td>${e.timestamp || ""}</td>
                </tr>
              `).join("")}
            </tbody>
          </table>
        </div>

        <a href="/management-hub" class="btn-back">← Back to Management Hub</a>
      </div>
    </div>
    ${themeScript()}
  </body>
  </html>`);
});

// ------------------------------------------------------
// MANAGEMENT: Guest Scan Log (with optional guest info)
// ------------------------------------------------------
app.get("/guest-scan-log", (req, res) => {
  if (!isMgmtAuthorizedReq(req)) {
    return res.redirect("/staff?key=" + encodeURIComponent(STAFF_PIN));
  }

  // Make a copy, newest first, and enrich with guest info
  const rows = [...guestScanLog].slice().reverse().map(evt => {
    const info = getGuestInfoForTicket(evt.ticketId);
    return {
      ticketId: evt.ticketId || "",
      tokenShort: (evt.token || "").slice(0, 8) + "...",
      ip: evt.ip || "unknown",
      time: evt.timestamp || "",
      guestName: info ? info.name : "",
      guestEmail: info ? info.email : "",
      guestPhone: info ? info.phone : ""
    };
  });

  res.send(`<!DOCTYPE html>
  <html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Guest Scan Log</title>
    <style>
      ${themeCSSRoot()}

      * { box-sizing: border-box; }

      body {
        margin:0;
        padding:16px;
        min-height:100vh;
        font-family:system-ui,-apple-system,BlinkMacSystemFont,sans-serif;
        background:#050007;
        color:#f5f5f5;
        display:flex;
        justify-content:center;
      }

      .wrap {
        width:100%;
        max-width:1024px;
      }

      .card {
        width:100%;
        border-radius:24px;
        padding:20px 20px 18px;
        background:radial-gradient(circle at top,#260020,#050007 65%);
        box-shadow:
          0 0 0 1px rgba(255,64,129,0.25),
          0 24px 60px rgba(0,0,0,0.9);
      }

      h1 {
        margin:0 0 4px;
        font-size:1.4rem;
        letter-spacing:0.06em;
        text-transform:uppercase;
        background:linear-gradient(135deg,#ffb300,#ff4081);
        -webkit-background-clip:text;
        color:transparent;
      }

      .subtitle {
        margin:0 0 16px;
        font-size:0.9rem;
        color:#c2b6ff;
      }

      table {
        width:100%;
        border-collapse:collapse;
        font-size:0.85rem;
      }

      th, td {
        padding:8px 10px;
        border-bottom:1px solid rgba(255,255,255,0.06);
        text-align:left;
        white-space:nowrap;
      }

      th {
        font-size:0.75rem;
        text-transform:uppercase;
        letter-spacing:0.12em;
        color:#ffb347;
        background:rgba(0,0,0,0.35);
        position:sticky;
        top:0;
        z-index:1;
      }

      tbody tr:nth-child(odd) td {
        background:rgba(255,255,255,0.01);
      }

      .btn-back {
        display:inline-flex;
        align-items:center;
        gap:6px;
        margin-top:14px;
        padding:8px 14px;
        border-radius:999px;
        border:1px solid rgba(255,64,129,0.6);
        background:transparent;
        color:#ffb347;
        font-size:0.8rem;
        text-decoration:none;
        text-transform:uppercase;
        letter-spacing:0.12em;
      }

      .btn-back:hover {
        background:rgba(255,64,129,0.15);
      }

      .scroll {
        max-height:70vh;
        overflow:auto;
        margin-top:8px;
      }

      @media (max-width: 720px) {
        table { font-size:0.78rem; }
        th, td { padding:6px 6px; }
      }
    </style>
  </head>
  <body>
    <div class="wrap">
      <div class="card">
        <h1>Guest Scan Log</h1>
        <div class="subtitle">
          Backend record of guest ticket scans (non-staff).<br/>
          If a ticket has guest details from the Mystery Prize form, they appear in the columns below.
        </div>

        <div class="scroll">
          <table>
            <thead>
              <tr>
                <th>Ticket ID</th>
                <th>Token (Short)</th>
                <th>IP</th>
                <th>Time</th>
                <th>Guest Name</th>
                <th>Guest Email</th>
                <th>Guest Phone</th>
              </tr>
            </thead>
            <tbody>
              ${rows.map(r => `
                <tr>
                  <td><strong>${r.ticketId}</strong></td>
                  <td>${r.tokenShort}</td>
                  <td>${r.ip}</td>
                  <td>${r.time}</td>
                  <td>${r.guestName || ""}</td>
                  <td>${r.guestEmail || ""}</td>
                  <td>${r.guestPhone || ""}</td>
                </tr>
              `).join("")}
            </tbody>
          </table>
        </div>

        <a href="/management-hub" class="btn-back">← Back to Management Hub</a>
      </div>
    </div>
    ${themeScript()}
  </body>
  </html>`);
});

// NEW: Cancelled tickets log
// shape: { ticketId, token, cancelledBy, source, timestamp }
const cancelledTicketsLog = [];

// ------------------------------------------------------
// START SERVER
// ------------------------------------------------------
app.listen(PORT, () => {
  console.log(`AURA ticket system running at http://${HOST}:${PORT}`);
})
