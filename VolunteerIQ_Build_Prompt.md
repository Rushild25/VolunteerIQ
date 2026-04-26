# VolunteerIQ — Full Build Prompt for Antigravity
### Next.js 14 + Supabase + Google Auth + Claude API

---

## OVERVIEW

Build a full-stack web application called **VolunteerIQ** — an AI-powered volunteer coordination platform for NGOs. The system collects incident reports from multiple sources (WhatsApp bot, Google Forms, and scanned handwritten reports via OCR), plots them on a live map, and uses AI to match the right volunteers to each situation. Coordinators review AI-generated suggestions and dispatch volunteers with one click.

---

## TECH STACK

| Layer | Technology |
|---|---|
| Frontend | Next.js 14 (App Router) |
| Styling | Tailwind CSS + shadcn/ui |
| Backend | Next.js API Routes (fullstack) |
| Database | Supabase (PostgreSQL) |
| Auth | Google OAuth via Supabase Auth |
| Map | Google Maps JavaScript API + Places API |
| AI / LLM | Claude API (Anthropic) — claude-sonnet-4-20250514 |
| WhatsApp | Twilio WhatsApp Sandbox |
| OCR | Claude API vision (image → structured JSON) |
| Real-time | Supabase Realtime (websockets) |
| Notifications | In-app only (future: Twilio WhatsApp notify) |
| Deployment | Vercel |
| Theme | Both light + dark mode with toggle |

---

## ROLES & PERMISSIONS

There are 4 user roles. Role is chosen at signup and approved by Super Admin before access is granted.

### 1. Super Admin
- Approves or rejects new user registrations and role requests
- Can change any user's role at any time
- Has access to all dashboards of all roles
- Manages NGO-level settings (name, logo, coverage area)
- Can view full audit logs of all actions in the system
- Can create / deactivate coordinators

### 2. Coordinator
- Main operational user — sees the live map and manages incidents
- Reviews AI-generated volunteer match suggestions
- Approves or rejects dispatches (manual approval mode)
- Can manually assign volunteers to incidents
- Views analytics dashboard (response times, resolution rates)
- Can add field workers to the system
- Cannot register new coordinators (only super admin can)

### 3. Field Worker
- Submits incident reports via the web app
- Can upload scanned handwritten reports (OCR processed)
- Sees only their own submitted reports and their status
- Gets notified when a report they submitted has been actioned
- No access to the map, volunteer list, or matching engine

### 4. Volunteer
- Self-registers via Google Auth, role request approved by Super Admin
- Has a profile page with: skills, location (city/area), availability toggle, task history
- Receives in-app notifications when assigned to a task
- Can accept or decline a task
- Submits post-task outcome report after completing a task
- Sees only their own assigned tasks

---

## AUTHENTICATION FLOW

1. User lands on `/` — marketing/landing page with "Sign in with Google" button
2. Google OAuth via Supabase Auth
3. After first login → user is taken to `/onboarding`
4. Onboarding page asks:
   - Full name (pre-filled from Google)
   - Phone number
   - Role request: Volunteer / Field Worker (Coordinator and Super Admin are assigned, not self-selected)
   - If Volunteer: skills (multi-select: medical, driving, cooking, logistics, teaching, general), city/area
5. Submission creates a user record with `status: pending_approval`
6. Super Admin sees pending users in their dashboard and approves/rejects
7. On approval, user gets in-app notification and can access their dashboard
8. Coordinators and Super Admins are created directly by existing Super Admin — they do not self-register

---

## DATABASE SCHEMA (Supabase PostgreSQL)

### Table: `users`
```sql
id uuid primary key
email text unique
full_name text
phone text
avatar_url text
role text -- 'super_admin' | 'coordinator' | 'field_worker' | 'volunteer'
status text -- 'pending' | 'active' | 'deactivated'
skills text[] -- for volunteers
location_area text -- city/neighbourhood
availability boolean default true
created_at timestamptz
```

### Table: `incidents`
```sql
id uuid primary key
title text
description text
incident_type text -- 'food' | 'medical' | 'shelter' | 'flood' | 'general'
urgency int -- 1 (today) | 2 (this week) | 3 (ongoing)
latitude float
longitude float
location_text text
source text -- 'web_form' | 'whatsapp' | 'google_form' | 'ocr_scan'
raw_input text -- original message/text before parsing
status text -- 'open' | 'assigned' | 'in_progress' | 'resolved'
reported_by uuid references users(id)
created_at timestamptz
resolved_at timestamptz
```

### Table: `tasks`
```sql
id uuid primary key
incident_id uuid references incidents(id)
volunteer_id uuid references users(id)
assigned_by uuid references users(id)
status text -- 'pending_acceptance' | 'accepted' | 'declined' | 'in_progress' | 'completed'
ai_match_score float
ai_match_reason text
dispatched_at timestamptz
accepted_at timestamptz
completed_at timestamptz
outcome_report text
```

### Table: `volunteer_matches` (AI suggestions per incident)
```sql
id uuid primary key
incident_id uuid references incidents(id)
volunteer_id uuid references users(id)
score float
reason text -- AI explanation
rank int
created_at timestamptz
```

### Table: `notifications`
```sql
id uuid primary key
user_id uuid references users(id)
title text
message text
read boolean default false
link text
created_at timestamptz
```

### Table: `audit_logs`
```sql
id uuid primary key
actor_id uuid references users(id)
action text
target_type text
target_id uuid
metadata jsonb
created_at timestamptz
```

---

## WEBSITE FLOW (PAGE BY PAGE)

---

### PUBLIC PAGES

#### `/` — Landing Page
- Hero section: "The right help, to the right place, in real time"
- Brief explanation of what VolunteerIQ does
- 3 role callouts: For NGOs / For Coordinators / For Volunteers
- "Sign in with Google" CTA button
- No public incident data visible

#### `/onboarding` — First-time setup (post Google login)
- Pre-filled name and avatar from Google
- Phone number input
- Role selector: Volunteer or Field Worker only
- If Volunteer: skills multi-select + area/city input + availability toggle
- Submit → account enters `pending_approval` state
- Page shown: "Your account is pending approval. You'll be notified once approved."

---

### SUPER ADMIN PAGES (`/admin/...`)

#### `/admin/dashboard`
- Stats: total users, pending approvals, active incidents, resolved today
- Pending approval queue — approve/reject with one click + optional note
- Recent audit log feed

#### `/admin/users`
- Full user table with filters (role, status, date joined)
- Click user → see profile, change role, activate/deactivate
- "Add Coordinator" button → creates coordinator directly (no self-signup)

#### `/admin/settings`
- NGO name, logo upload, coverage area (draw on Google Maps)
- Feature toggles: auto-dispatch on/off, WhatsApp integration on/off

---

### COORDINATOR PAGES (`/coordinator/...`)

#### `/coordinator/dashboard`
- **Left panel**: Live Google Maps
  - Pins colored by urgency (red = critical, amber = moderate, blue = ongoing)
  - Marker clustering via `@googlemaps/markerclusterer` — clicking a cluster expands it
  - Realtime updates via Supabase Realtime — new incidents appear instantly
- **Right panel**: Incident feed sorted by urgency
  - Each card: incident type icon, location, urgency badge, source badge (WhatsApp / Form / OCR), time ago
  - Click → opens incident detail sidebar

#### `/coordinator/incidents/[id]` — Incident Detail Sidebar/Page
- Full incident details: description, type, urgency, location, source, raw input (collapsible)
- AI Match Panel:
  - Top 3–5 volunteers ranked by score
  - Each volunteer card: name, photo, distance from incident, skills, availability status
  - AI explanation text: "Matched because: 800m away, has medical skills, available today"
  - "Approve & Dispatch" button per volunteer
  - "Dispatch All Top 3" button
- Manual override: search and assign any volunteer manually
- Incident status controls: mark as in progress / resolved

#### `/coordinator/volunteers`
- Table of all volunteers with filters: skills, availability, area, active task count
- Click volunteer → see full profile + task history

#### `/coordinator/analytics`
- Charts: incidents over time, resolution rate, average response time
- Heatmap by area (which zones get the most incidents)
- Volunteer performance: tasks completed, acceptance rate
- Source breakdown: how many reports from WhatsApp vs Forms vs OCR

#### `/coordinator/reports/new` — Manual Report Entry
- Form: incident type, description, location (map pin drop + text), urgency, source (manual)
- Submit → creates incident, triggers AI matching immediately

---

### FIELD WORKER PAGES (`/field/...`)

#### `/field/dashboard`
- List of their own submitted reports with status badges
- "New Report" button prominent at top
- Notification feed (e.g. "Your report in Sector 12 has been assigned")

#### `/field/report/new` — Submit Incident
- Incident type selector (icon grid: food, medical, shelter, flood, general)
- Description textarea
- Location: "Use my location" button OR type area name
- Urgency: 1 / 2 / 3 selector with plain English labels
- Optional: photo upload
- **OCR Upload tab**: upload a photo of a handwritten report → Claude Vision extracts all fields → pre-fills form → field worker reviews and confirms
- Submit → incident created with `source: web_form`

#### `/field/report/[id]` — Report Status
- Shows submitted report details
- Live status indicator: Open → Assigned → In Progress → Resolved
- Shows which volunteer was assigned (name only, no contact)

---

### VOLUNTEER PAGES (`/volunteer/...`)

#### `/volunteer/dashboard`
- Active task card (if any): location, what to do, coordinator contact
- "Accept" / "Decline" buttons if task is pending acceptance
- Notification feed

#### `/volunteer/tasks`
- All tasks: active, completed, declined
- Each task card: incident type, area, date, status

#### `/volunteer/tasks/[id]` — Task Detail
- Full task details: what to do, location (map), coordinator name
- Accept / Decline (if pending)
- "Mark Complete" button + outcome report textarea (required to close task)

#### `/volunteer/profile`
- Edit: name, phone, photo, skills (multi-select), area/city
- Availability toggle (on/off) — coordinator sees this in real time
- Task history table
- Stats: tasks completed, average response time

---

### SHARED PAGES

#### `/notifications`
- All notifications for the logged-in user
- Mark all as read

#### `/profile`
- Basic profile edit available to all roles
- Shows role badge and account status

---

## INTAKE CHANNELS — TECHNICAL IMPLEMENTATION

### Channel 1: Web Form (Field Workers)
- Direct form submission from `/field/report/new`
- Location captured via browser geolocation or text input → geocoded via Google Maps Geocoding API
- OCR tab: image uploaded → sent to Claude API vision with prompt:
  ```
  Extract from this handwritten report image: incident_type, location, urgency (1-3), description.
  Return only valid JSON. If a field is unclear, return null for that field.
  ```
- Pre-fills form fields — field worker confirms before submitting

### Channel 2: WhatsApp via Twilio
- Twilio WhatsApp Sandbox webhook → POST to `/api/webhooks/whatsapp`
- Incoming message body sent to Claude API with prompt:
  ```
  You are parsing incident reports for an NGO coordination system.
  Extract from this message: incident_type (food/medical/shelter/flood/general),
  location (as specific as possible), urgency (1=today, 2=this week, 3=ongoing),
  description (a clean 1-sentence summary).
  Return ONLY valid JSON. If confidence on any field is below 0.7, return null for that field.
  Also return a "confidence" object with a score per field (0-1).
  Message: "{message}"
  ```
- If any field returns null → Twilio bot replies asking only for the missing field
- Once all fields populated → incident created with `source: whatsapp`
- Bot replies: "✅ Report received. Help is being coordinated. Thank you."

### Channel 3: Google Forms
- Google Form set up with fields matching incident schema
- Google Apps Script sends POST to `/api/webhooks/google-form` on new submission
- API route validates and creates incident with `source: google_form`
- No LLM needed — data already structured

---

## AI MATCHING ENGINE

### Trigger
Runs automatically when a new incident is created (any source). Also runs when coordinator manually triggers re-match.

### API Route: `POST /api/ai/match`

### Algorithm
1. Fetch all volunteers where `status = active` AND `availability = true`
2. For each volunteer, compute a score:
   - **Distance score** (40%): Haversine distance from volunteer's area centroid to incident location. Closer = higher score.
   - **Skill match score** (35%): Compare volunteer skills array to incident type. Medical incident → medical skill = full score. Driving = partial. No match = 0.
   - **Workload score** (15%): Volunteers with fewer active tasks score higher.
   - **History score** (10%): Volunteers who completed similar incident types before score higher.
3. Sort descending. Take top 5.
4. Send top 5 to Claude API for explanation generation:
   ```
   Given this incident: {incident_summary}
   And this volunteer profile: {volunteer_summary}
   Write one short sentence (max 15 words) explaining why this volunteer is a good match.
   Be specific. Mention distance, skills, or availability.
   ```
5. Store results in `volunteer_matches` table
6. Coordinator sees ranked list with explanations in the UI

---

## GOOGLE MAPS IMPLEMENTATION NOTES

### Package
Use `@vis.gl/react-google-maps` (the official React wrapper for Google Maps JS API). Install via:
```bash
npm install @vis.gl/react-google-maps
```

### Map Setup (coordinator dashboard)
```tsx
import { APIProvider, Map, AdvancedMarker, Pin } from '@vis.gl/react-google-maps'

<APIProvider apiKey={process.env.NEXT_PUBLIC_GOOGLE_MAPS_API_KEY}>
  <Map defaultCenter={{ lat: 28.6, lng: 77.2 }} defaultZoom={12} mapId="volunteeriq-map">
    {incidents.map(inc => (
      <AdvancedMarker key={inc.id} position={{ lat: inc.latitude, lng: inc.longitude }}>
        <Pin background={urgencyColor(inc.urgency)} />
      </AdvancedMarker>
    ))}
  </Map>
</APIProvider>
```

### Marker Clustering
```bash
npm install @googlemaps/markerclusterer
```
Wrap markers with `MarkerClusterer` — clusters auto-expand on zoom.

### Geocoding (text → lat/lng)
Use Google Maps Geocoding API via server-side route `/api/geocode`:
```ts
const res = await fetch(
  `https://maps.googleapis.com/maps/api/geocode/json?address=${encodeURIComponent(locationText)}&key=${process.env.GOOGLE_MAPS_API_KEY}`
)
const { results } = await res.json()
const { lat, lng } = results[0].geometry.location
```

### Place Autocomplete (location input fields)
Use `usePlacesAutocomplete` from `@vis.gl/react-google-maps` on all location text inputs — gives users live address suggestions as they type.

### APIs to enable in Google Cloud Console
- Maps JavaScript API
- Geocoding API
- Places API
- (Optional) Maps Embed API

---



- Supabase Realtime subscriptions on `incidents` and `tasks` tables
- Coordinator map updates instantly when new incident is inserted
- Volunteer dashboard updates instantly when new task is assigned
- Notifications table subscribed per user — bell icon badge updates live
- No polling anywhere — pure websocket via Supabase client

---

## KEY COMPONENT STRUCTURE

```
/app
  /(public)
    /page.tsx                  → Landing
    /onboarding/page.tsx       → Role setup
  /(auth)
    /admin/...
    /coordinator/...
    /field/...
    /volunteer/...
    /notifications/page.tsx
    /profile/page.tsx
/components
  /map/MapView.tsx             → Google Maps coordinator map
  /incidents/IncidentCard.tsx
  /incidents/IncidentSidebar.tsx
  /matching/MatchSuggestions.tsx
  /matching/VolunteerCard.tsx
  /reports/ReportForm.tsx
  /reports/OCRUpload.tsx
  /shared/NotificationBell.tsx
  /shared/RoleBadge.tsx
  /shared/ThemeToggle.tsx
/lib
  /supabase/client.ts
  /supabase/server.ts
  /ai/matchVolunteers.ts
  /ai/parseWhatsApp.ts
  /ai/parseOCR.ts
  /google/geocode.ts
/api
  /webhooks/whatsapp/route.ts
  /webhooks/google-form/route.ts
  /ai/match/route.ts
  /incidents/route.ts
  /tasks/route.ts
  /users/route.ts
```

---

## ENVIRONMENT VARIABLES NEEDED

```env
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
NEXT_PUBLIC_GOOGLE_MAPS_API_KEY=
ANTHROPIC_API_KEY=
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_WHATSAPP_NUMBER=
GOOGLE_FORMS_WEBHOOK_SECRET=
```

---

## PAGES SUMMARY TABLE

| Page | Role | Purpose |
|---|---|---|
| `/` | Public | Landing + sign in |
| `/onboarding` | New user | Role setup |
| `/admin/dashboard` | Super Admin | Approvals + overview |
| `/admin/users` | Super Admin | User management |
| `/admin/settings` | Super Admin | NGO settings |
| `/coordinator/dashboard` | Coordinator | Live map + incident feed |
| `/coordinator/incidents/[id]` | Coordinator | AI match + dispatch |
| `/coordinator/volunteers` | Coordinator | Volunteer browser |
| `/coordinator/analytics` | Coordinator | Stats + charts |
| `/coordinator/reports/new` | Coordinator | Manual report entry |
| `/field/dashboard` | Field Worker | My reports |
| `/field/report/new` | Field Worker | Submit incident + OCR |
| `/field/report/[id]` | Field Worker | Report status |
| `/volunteer/dashboard` | Volunteer | Active task |
| `/volunteer/tasks` | Volunteer | Task history |
| `/volunteer/tasks/[id]` | Volunteer | Task detail + accept |
| `/volunteer/profile` | Volunteer | Skills + availability |
| `/notifications` | All | Notification centre |
| `/profile` | All | Basic profile edit |

**Total: 19 pages**

---

## WHAT TO BUILD FIRST (SUGGESTED ORDER)

1. Auth + Supabase setup + Google OAuth + role system
2. Onboarding flow + Super Admin approval queue
3. Field worker report form (web form, no OCR yet)
4. Coordinator dashboard with Google Maps + incident feed
5. AI matching engine + match suggestions panel
6. Dispatch flow + volunteer task acceptance
7. Volunteer profile + availability toggle
8. WhatsApp webhook + Claude parser
9. OCR upload on report form
10. Analytics page
11. Real-time subscriptions (Supabase Realtime)
12. Dark/light mode toggle + polish

---

*Generated for VolunteerIQ hackathon build — Antigravity team*
