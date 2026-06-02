# Eskom App — Full Deployment Architecture Reference

## Overview

This project uses a **monorepo, full-stack deployment** pattern where a single Express.js server (`server.js`) is responsible for **both serving the frontend (HTML/CSS/JS pages) AND exposing the REST API endpoints**. There is no separate frontend deployment. Both Vercel and Render fully support this pattern.

---

## Project Structure

```
eskom-app/
│
├── server.js              ← Express server: serves frontend + all API routes
├── package.json           ← Dependencies + start script
├── vercel.json            ← Vercel deployment config
│
├── index.html             ← Landing page (root)
│
├── pages/                 ← All app HTML pages
│   ├── login.html
│   ├── dashboard.html
│   ├── admin.html
│   ├── cases.html
│   ├── caseList.html
│   ├── assign.html
│   ├── record.html
│   ├── map.html
│   ├── report.html
│   ├── commander.html
│   ├── resolved.html
│   ├── evaluations.html
│   └── user-dashboard.html
│
├── css/                   ← Stylesheets
│   ├── landing.css
│   ├── dashboard.css
│   ├── assign.css
│   ├── caseList.css
│   ├── report.css
│   ├── resolved.css
│   └── style.css
│
└── js/                    ← Client-side scripts
    ├── auth.js
    ├── dashboard.js
    ├── cases.js
    ├── assign.js
    ├── record.js
    ├── map.js
    ├── report.js
    ├── commander.js
    ├── evaluations.js
    ├── landing.js
    ├── main.js
    └── record.js
```

---

## How the Server Works (server.js)

### 1. Static File Serving

```js
const publicRoot = path.join(__dirname);

// Block sensitive files from being served
app.use((req, res, next) => {
    const blockedPaths = ['/server.js', '/package.json', '/package-lock.json', '/.env', '/env'];
    if (blockedPaths.includes(req.path)) return res.status(404).end();
    next();
});

// Serve everything in the project root (css/, js/, pages/, index.html) as static files
app.use(express.static(publicRoot, { dotfiles: 'ignore', index: false }));

// Root route explicitly serves the landing page
app.get('/', (req, res) => res.sendFile(path.join(publicRoot, 'index.html')));
```

**What this means:**
- `/css/dashboard.css` → served directly from `css/dashboard.css`
- `/js/auth.js` → served directly from `js/auth.js`
- `/pages/login.html` → served directly from `pages/login.html`
- `/` → serves `index.html`

The frontend never needs to call a separate CDN or static file host — Express handles it all.

### 2. API Endpoints

All API routes are prefixed with `/api/` and come after the static middleware:

| Method | Route | Auth | Role |
|--------|-------|------|------|
| POST | `/api/auth/login` | None | Public |
| GET | `/api/cases` | JWT | All |
| POST | `/api/cases` | JWT | admin, commander |
| PUT | `/api/cases/:id` | JWT | All (scoped by role) |
| DELETE | `/api/cases/:id` | JWT | admin |
| POST | `/api/cases/:id/reopen` | JWT | admin |
| GET | `/api/investigators` | JWT | All |
| POST | `/api/investigators` | JWT | admin |
| GET | `/api/users` | JWT | admin |
| POST | `/api/users` | JWT | admin |
| PUT | `/api/users/:id/status` | JWT | admin |
| PUT | `/api/users/:id/role` | JWT | admin |
| DELETE | `/api/users/:id` | JWT | admin |
| GET | `/api/properties` | JWT | All |
| POST | `/api/upload` | JWT | admin |
| POST | `/api/tips` | None | Public |
| GET | `/api/tips` | JWT | All |
| GET | `/api/dashboard/stats` | JWT | All (scoped by role) |
| GET | `/api/commander/stats` | JWT | commander, admin |
| GET | `/api/evaluations` | JWT | admin, commander |
| POST | `/api/evaluations` | JWT | admin, commander |
| GET | `/api/reports/:caseId` | JWT | All (scoped by role) |

### 3. Authentication & RBAC

**JWT flow:**
1. Client POSTs `{ email, password, role }` to `/api/auth/login`
2. Server validates against Supabase `users` table (bcrypt password compare)
3. Server returns `{ token, role, userId }`
4. Client stores token in `localStorage`
5. Every subsequent API call sends `Authorization: Bearer <token>` header
6. `authenticateToken` middleware verifies the token on every protected route
7. `requireRole(['admin', 'commander'])` middleware enforces role-based access

**Roles:**
- `admin` — full access to all endpoints
- `commander` — case management, investigator oversight, evaluations
- `investigator` — own cases only (filtered at query level, not just middleware)

---

## HTML Asset Path Rules

This is critical. Because pages live inside `/pages/` subdirectory:

### Pages inside `/pages/*.html`
```html
<!-- CSS: go up one level with ../ -->
<link rel="stylesheet" href="../css/dashboard.css">
<link rel="stylesheet" href="../css/assign.css">

<!-- JS: go up one level with ../ -->
<script src="../js/auth.js"></script>
<script src="../js/dashboard.js"></script>

<!-- Internal page links: same directory, no prefix needed -->
<a href="dashboard.html">Dashboard</a>
<a href="login.html">Login</a>
```

### Root index.html
```html
<!-- CSS: no prefix, files are in same level -->
<link rel="stylesheet" href="css/landing.css">

<!-- JS: no prefix -->
<script src="js/landing.js"></script>

<!-- Link into pages/ -->
<a href="pages/login.html">Get Started</a>
```

### API calls in JS files (`js/*.js`)
```js
// BASE_URL is always empty string — relative to current origin
const BASE_URL = '';

// This resolves to /api/cases on whatever domain the app is running on
fetch(`${BASE_URL}/api/cases`, { headers: authHeaders });
```

Using `BASE_URL = ''` means the same code works:
- Locally at `http://localhost:3000`
- On Vercel at `https://eskom-app.vercel.app`
- On Render at `https://eskom-app.onrender.com`
- On any other domain

**Never hardcode `http://localhost:3000`** in fetch calls — it will break in production.

---

## Environment Variables

Create a `.env` file locally (never commit this):

```env
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_KEY=your-service-role-key
JWT_SECRET=your-long-random-secret-string
PORT=3000
```

In production (Vercel or Render), set these same variables in the platform's environment variable settings panel.

---

## Local Development

```bash
# Install dependencies
npm install

# Start the server
npm start
# or
node server.js
```

Then open `http://localhost:3000` in your browser.

The server listens on `process.env.PORT || 3000`, so it works both locally and in production where the platform sets PORT automatically.

```js
// At the bottom of server.js
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
```

---

## Deploying on Vercel

### The Problem Vercel Has By Default

Vercel's `@vercel/node` builder packages `server.js` into a serverless function. By default it only includes JavaScript files and `node_modules`. Your `css/`, `js/`, `pages/`, and `index.html` are **not bundled**, so `express.static()` can't find them at runtime and every static file request returns 404.

### The Fix: `vercel.json` with `includeFiles`

```json
{
  "version": 2,
  "builds": [
    {
      "src": "server.js",
      "use": "@vercel/node",
      "config": {
        "includeFiles": [
          "index.html",
          "css/**",
          "js/**",
          "pages/**"
        ]
      }
    }
  ],
  "routes": [
    { "src": "/(.*)", "dest": "/server.js" }
  ]
}
```

**What each part does:**

| Config | Purpose |
|--------|---------|
| `"src": "server.js"` | Entry point for the serverless function |
| `"use": "@vercel/node"` | Tells Vercel to use the Node.js runtime |
| `"includeFiles"` | Bundles all static assets into the function so `__dirname` paths resolve correctly |
| `"routes": [{ "src": "/(.*)", "dest": "/server.js" }]` | Routes ALL requests (static files + API) through Express |

Because Express handles routing internally — static files first via `express.static()`, then API routes, then the `404` fallback — Vercel just needs to forward everything to `server.js`.

### Deploy to Vercel

1. Push your code to GitHub
2. Import the repo at [vercel.com](https://vercel.com)
3. Add your environment variables in Project Settings → Environment Variables
4. Deploy — Vercel auto-detects `vercel.json`

---

## Deploying on Render

Render is a traditional server host (not serverless), which means it runs your server as a **persistent process**. This is actually simpler — `express.static()` works exactly the same as locally because the filesystem is a real filesystem.

### render.yaml (create this file in your project root)

```yaml
services:
  - type: web
    name: eskom-app
    runtime: node
    buildCommand: npm install
    startCommand: npm start
    envVars:
      - key: SUPABASE_URL
        sync: false
      - key: SUPABASE_KEY
        sync: false
      - key: JWT_SECRET
        sync: false
      - key: NODE_ENV
        value: production
```

### Deploy to Render

1. Push your code to GitHub
2. Create a new **Web Service** at [render.com](https://render.com)
3. Connect your GitHub repo
4. Set:
   - **Build Command:** `npm install`
   - **Start Command:** `npm start`
5. Add your environment variables in the Environment tab
6. Deploy

**No `vercel.json` is used on Render.** Render just runs `npm start` which runs `node server.js` directly — the server boots up, `express.static()` serves the files from the real filesystem, and everything works exactly as it does locally.

### Vercel vs Render — Key Differences

| | Vercel | Render |
|--|--------|--------|
| Execution model | Serverless (function spins up per request) | Persistent server process |
| Static files | Must use `includeFiles` in vercel.json | Works automatically from filesystem |
| Cold starts | Yes (first request after idle can be slow) | No (server stays running) |
| Config file needed | `vercel.json` required | `render.yaml` optional |
| Free tier | Generous, no sleep | Sleeps after 15min inactivity on free tier |
| Best for | High-traffic, auto-scale apps | Apps needing persistent state or WebSockets |

---

## Complete Request Flow (Both Platforms)

```
Browser requests /pages/dashboard.html
       │
       ▼
Vercel/Render routes to server.js
       │
       ▼
Express security middleware
  (blocks /server.js, /.env, etc.)
       │
       ▼
express.static(publicRoot)
  ─ finds pages/dashboard.html in bundle/filesystem
  ─ sends the file
       │
       ▼
Browser receives HTML, parses it
  ─ requests ../css/dashboard.css → Express serves it
  ─ requests ../js/dashboard.js  → Express serves it
       │
       ▼
dashboard.js runs, calls:
  fetch('/api/dashboard/stats', { headers: { Authorization: 'Bearer ...' }})
       │
       ▼
Express routes to app.get('/api/dashboard/stats', authenticateToken, ...)
  ─ verifies JWT
  ─ queries Supabase
  ─ returns JSON
       │
       ▼
Dashboard renders data
```

---

## Summary Checklist for Any New Project Using This Pattern

- [ ] `server.js` uses `express.static(path.join(__dirname))` to serve all frontend files
- [ ] Root route `app.get('/')` explicitly sends `index.html`
- [ ] All pages in subdirectories use `../css/` and `../js/` for asset paths
- [ ] All `fetch()` calls use `BASE_URL = ''` (relative URLs, never hardcoded localhost)
- [ ] `package.json` has `"start": "node server.js"` script
- [ ] Server listens on `process.env.PORT || 3000`
- [ ] Sensitive files are blocked before `express.static()`
- [ ] For **Vercel**: `vercel.json` includes `includeFiles` with all static asset globs
- [ ] For **Render**: `render.yaml` or manual service config with `npm start`
- [ ] Environment variables set in platform dashboard (never committed to git)
- [ ] `.env` is in `.gitignore`
