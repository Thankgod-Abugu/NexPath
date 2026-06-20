# NexPath — System Architecture & Tech Stack Plan

---

## Overview

NexPath is a two-system platform: a **Web Platform** (destination search, route generation, deep linking) and a **Mobile AR App** (real-world navigation overlay). They communicate via a shared backend and a deep-link handoff mechanism.

---

## System Architecture Diagram (Text)

```
┌─────────────────────────────────────────────────────────────────────┐
│                          USER JOURNEY                               │
│                                                                     │
│  [User opens Web App] → [Searches destination] → [Route generated] │
│       → [Taps "Navigate in AR"] → [Opens Mobile AR App]            │
│              → [AR arrows + waypoints overlay real world]           │
└─────────────────────────────────────────────────────────────────────┘

┌──────────────────────────┐        ┌────────────────────────────────┐
│     WEB PLATFORM         │        │       MOBILE AR APP            │
│  (Browser / React.js)    │        │  (Unity + ARCore/ARKit)        │
│                          │        │                                │
│  - Search UI             │        │  - Camera feed                 │
│  - Map display           │◄──────►│  - AR arrow overlays           │
│  - Route preview         │        │  - Waypoint markers            │
│  - Deep link generator   │        │  - Destination label           │
│                          │        │  - GPS + Gyroscope + Compass   │
└──────────┬───────────────┘        └──────────────┬─────────────────┘
           │                                       │
           ▼                                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                        BACKEND (Node.js / Express)               │
│                                                                  │
│  - Campus Location Database (PostgreSQL + PostGIS)               │
│  - Route API endpoint (/api/route)                               │
│  - Location search endpoint (/api/locations)                     │
│  - Deep link token generator (/api/session)                      │
│  - Google Maps / OSM API proxy                                   │
└──────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────┐
│     EXTERNAL APIs               │
│                                 │
│  - Google Maps Geocoding API    │
│  - Google Maps Directions API   │
│  - OpenStreetMap / OSRM         │
└─────────────────────────────────┘
```

---

## Layer-by-Layer Breakdown

### 1. Web Platform

**Purpose:** Let the user search for a destination, preview the route on a map, then hand off to the AR app.

| Concern | Recommended Choice | Why |
|---|---|---|
| Framework | React.js (Vite) | Fast, component-based, good ecosystem |
| Map display | Leaflet.js + OSM tiles | Free, lightweight, customizable |
| Geocoding | Google Maps Geocoding API | Most accurate for real addresses |
| Route display | Google Maps Directions API or OSRM | Turn-by-turn waypoints |
| Styling | Tailwind CSS | Rapid, consistent UI |
| Deep linking | Custom URL scheme `nexpath://navigate?data=...` | Passes route to AR app |
| Hosting | Vercel or Netlify (free tier) | Simple deployment |

**Key screens:**
- Home / search screen
- Map preview with route drawn
- "Open in AR" button (triggers deep link)
- Admin panel for adding custom campus locations

---

### 2. Backend API

**Purpose:** Central coordinator — handles location data, routing logic, session tokens, and API proxying.

| Concern | Recommended Choice | Why |
|---|---|---|
| Runtime | Node.js + Express | JavaScript consistency with web frontend |
| Database | PostgreSQL + PostGIS extension | Stores lat/lng, supports geospatial queries |
| ORM | Prisma | Clean schema management |
| Authentication | JWT (for admin panel only) | Lightweight, stateless |
| Hosting | Railway or Render (free tier) | Easy Node + Postgres deployment |
| API proxy | Wrap Google Maps calls server-side | Keeps your API key hidden |

**Key endpoints:**
```
GET  /api/locations?q=computer+science     → search campus locations
GET  /api/locations/:id                    → get exact coordinates
POST /api/route                            → generate route between two points
POST /api/session                          → create deep-link session token
GET  /api/session/:token                   → AR app fetches route data by token
```

**Campus Location DB Schema:**
```sql
CREATE TABLE locations (
  id          SERIAL PRIMARY KEY,
  name        TEXT NOT NULL,           -- "Computer Science Dept Building"
  aliases     TEXT[],                  -- ["CS Building", "Department"]
  category    TEXT,                    -- "academic" | "admin" | "facility"
  lat         DOUBLE PRECISION,
  lng         DOUBLE PRECISION,
  description TEXT,
  is_mapped   BOOLEAN DEFAULT false    -- true if on Google Maps already
);
```

---

### 3. Deep Link Handoff (Web → AR App)

This is the bridge between the two systems. When the user taps "Navigate in AR":

**Step-by-step flow:**
```
1. Web app sends route data to backend → gets back a session token (e.g. "abc123")
2. Web app opens: nexpath://navigate?token=abc123
3. Mobile OS intercepts this URL → opens NexPath AR app
4. AR app reads token → calls GET /api/session/abc123
5. AR app receives: { waypoints: [{lat, lng}, ...], destination: "CS Building" }
6. AR app begins navigation
```

**Fallback:** If the AR app isn't installed, the deep link falls back to a QR code the user can scan, or a web-based compass-guide view.

---

### 4. Mobile AR App

**Purpose:** Overlay navigation arrows and waypoints onto the real-world camera feed.

| Concern | Recommended Choice | Why |
|---|---|---|
| Engine | Unity (recommended over Unreal) | Better ARCore/ARKit support, lighter builds, more campus nav examples |
| AR Framework | AR Foundation (Unity) | Single codebase for both Android (ARCore) + iOS (ARKit) |
| Language | C# | Unity's native language |
| GPS + Sensors | Unity LocationService + Input.compass | Built-in device sensor access |
| Waypoint rendering | World-space 3D arrows (Unity GameObjects) | Placed at real GPS coordinates |
| Backend comms | UnityWebRequest (REST calls to your Node.js API) | Fetch route data on launch |
| Deep link handling | Application.deepLinkActivated (Unity event) | Catches `nexpath://` URL |

**Navigation logic:**
```
1. App receives waypoints (list of lat/lng pairs)
2. Converts each waypoint from GPS coords → Unity world-space position
   using Haversine formula relative to user's current GPS location
3. Places 3D arrow GameObjects at those world positions
4. As user moves, recalculates positions in real time using GPS updates
5. Arrow nearest to user pulses/animates to draw attention
6. On arrival (within ~5m of destination), shows a "You have arrived" overlay
```

---

### 5. Campus Location Database (Custom Layer)

This is NexPath's competitive advantage over just using Google Maps.

**What to store:**
- Buildings not yet on Google Maps (e.g. new CS building)
- Campus-specific nicknames (e.g. "Container" cybercafe)
- Internal landmarks (specific lecture halls, labs within buildings)
- Pedestrian shortcuts and internal pathways

**How to populate it:**
- Manual entry via admin web panel
- Walk around campus with phone GPS to record coordinates
- Cross-reference with OpenStreetMap campus data exports

---

## Recommended Pathfinding Logic

```
If destination exists in Google Maps:
    → Use Google Maps Directions API (walking mode) for waypoints

Else if destination is in campus custom DB:
    → Use stored lat/lng as destination endpoint
    → Generate route using OSRM (open source) or A* on OSM graph

For campus shortcuts (internal paths not on any map):
    → Store as custom path segments in DB
    → Apply Dijkstra's algorithm to find shortest walkable path
```

---

## Technology Stack Summary

```
┌─────────────────┬───────────────────────────────────────────┐
│ Layer           │ Technology                                │
├─────────────────┼───────────────────────────────────────────┤
│ Web Frontend    │ React.js + Vite + Tailwind CSS            │
│ Map Display     │ Leaflet.js + OpenStreetMap tiles          │
│ Geocoding       │ Google Maps Geocoding API                 │
│ Routing         │ Google Maps Directions API / OSRM         │
│ Backend API     │ Node.js + Express                         │
│ Database        │ PostgreSQL + PostGIS                      │
│ ORM             │ Prisma                                    │
│ Auth (admin)    │ JWT                                       │
│ Mobile Engine   │ Unity 2022 LTS                            │
│ AR Framework    │ AR Foundation (ARCore + ARKit)            │
│ Deep Linking    │ Custom URI scheme (nexpath://)            │
│ Deployment      │ Vercel (web) + Railway (backend)         │
│ Version Control │ Git + GitHub                              │
└─────────────────┴───────────────────────────────────────────┘
```

---

## Development Phases (Suggested Order)

### Phase 1 — Foundation (Weeks 1–3)
- Set up PostgreSQL database + seed it with 10–15 UNN campus locations
- Build backend API (location search, route endpoint, session token)
- Test with Postman

### Phase 2 — Web Platform (Weeks 4–6)
- Build React frontend: search UI, Leaflet map, route preview
- Integrate Google Maps Geocoding + Directions API
- Implement deep link generation + QR fallback

### Phase 3 — Mobile AR App (Weeks 7–11)
- Set up Unity project with AR Foundation
- Implement GPS → world-space coordinate conversion
- Render arrows and waypoint markers in AR
- Implement deep link receiver + API call to fetch route

### Phase 4 — Integration & Testing (Weeks 12–14)
- End-to-end test: web → deep link → AR navigation
- Test on campus with real GPS
- Accuracy testing, edge cases (tall buildings, poor signal)
- User testing with students

### Phase 5 — Polish & Submission (Weeks 15–16)
- Admin panel for managing campus locations
- UI polish on both platforms
- Documentation + project writeup

---

## Key Risks & Mitigations

| Risk | Mitigation |
|---|---|
| GPS inaccuracy near tall buildings | Show compass bearing as fallback; future: Bluetooth beacons |
| Google Maps API cost | Cap daily requests; use OSM/OSRM as free fallback |
| AR app not installed when user clicks deep link | Fallback to QR code + web compass view |
| New buildings not in Google Maps | Custom campus DB solves this by design |
| Deep link fails on some Android versions | Use App Links (HTTPS-based) as more reliable alternative |
