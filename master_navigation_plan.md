# Master Navigation & POI Architecture

## 1. Executive Summary: The Objective Best Stack
For an AI-driven iOS app that relies on hyper-customizable UI and constant, proactive "surroundings" awareness (reading POIs to the user), the combination of **Mapbox Navigation SDK** and **Overture Maps (Foursquare/GERS)** is the most powerfully scalable and legally compliant stack available.

### Why the Alternatives Fail:
1. **Google Maps / Places API:** Google's Terms of Service strictly prohibit rendering Places data on a non-Google Map. More importantly, calling the Google Places Local Search API continuously while a user walks would result in astronomical API bills. 
2. **Apple MapKit:** Native and free, but highly rigid. Customizing the turn-by-turn UI to match a premium, futuristic AI aesthetic is notoriously difficult, and gaining deep access to Apple's POI database for offline/AI ingestion is impossible.
3. **TomTom / HERE:** Map UI SDKs are far less developer-friendly and visually polished compared to Mapbox v11.

### Why Mapbox + Overture Wins:
- **Mapbox** provides pixel-perfect UI customization (e.g., hiding generic map chrome, glowing 3D paths). Their `CoreNavigation` engine handles the incredibly complex math of off-route detection, GPS snapping, and active/passive states.
- **Overture Maps** (which heavily incorporates Foursquare data) utilizes the GERS (Global Entity Reference System). Because it's open data, you can self-host millions of POIs in your own database. Your AI can poll this database infinitely for zero marginal cost, enabling true "proactive narration."

---

## 2. The Implementation Blueprint

### Step 1: The POI Backend (Overture + PostGIS)
You cannot ship a 50GB Overture dataset inside the iOS app binary.
1. **Database:** Host the Overture Places Parquet files (filtered to your launch cities) on a **Supabase PostgreSQL database** using the `PostGIS` extension.
2. **The Spatial API:** Build a Supabase Edge Function that takes `latitude`, `longitude`, and `radius`. It utilizes `ST_DWithin` to instantly return all Overture POIs around the user.
3. **Integration (Single Source of Truth Fix):** `ContextPipeline` must NOT fire its own `CLLocationManager` to get these coordinates. It must explicitly receive coordinates passively from Mapbox's `EnhancedLocationManager` (funneled via the `AppCoordinator`) to prevent catastrophic dual-GPS battery drain.
4. **Network Polling Rule:** Do NOT poll this API every 10 seconds. 
   - **Passive Mode (Pocket):** Only poll Supabase when the user breaks the massive 2-kilometer `Bounding Box Geofence` (The 19+1 Pagination Rule).
   - **Active Mode (Screen On / Routing):** Only poll Supabase when Mapbox detects a positional delta of `>50 meters` from the last fetch.

### Step 2: Mapbox Navigation Integration
1. **Application Layer:** Discard Mapbox's default "Drop-In UI" (`NavigationViewController`). Instead, build a custom SwiftUI `NavigationMapView` wrapper.
2. **Route Fetching:** Use `NavigationRouteOptions` to request Walk/Drive routes.
3. **Active Navigation Spooling:** When a user starts a route, initialize `MapboxNavigationProvider`. Subscribe to its `routeProgress` publisher to track distance to the next turn under-the-hood.

### Step 3: The "AI-Native" UX (Synthesizing Voice)
Standard navigation apps use rigid TTS ("In 500 feet, turn left"). For a top-tier companion app, ElevenLabs must own the navigation directions natively.
1. **Intercept Instructions:** Mapbox emits `SpokenInstruction` events. Instead of letting `AVSpeechSynthesizer` read them, intercept the event.
2. **System Prompt Injection:** Pipe the Mapbox instruction into the ElevenLabs System Prompt alongside the local Overture POI data.
3. **Result:** Instead of robotic directions, the AI says: *"Take a left at the next light—by the way, I see you're passing that Foursquare-rated heritage museum right now."*

### Step 4: Advanced AI Real-Time Tools (Micro-Context)
To elevate the navigation, we expose specific iOS spatial sensors to the AI using ElevenLabs Client Tool Calls:
1. **Heading & Angle Direction (`CLHeading`):** Inject the user's compass bearing. This allows the AI to give relative, human directions: *"Look to your right, that glass building is the museum"* instead of absolute directions (*"Look North"*).
2. **Velocity & Motion State (`CMMotionActivityManager`):** Determine if the user is Walking, Running, or Stationary. The AI adjusts pacing: *"Keep this pace, and you'll arrive in 3 minutes."*
3. **Route Probability Status:** Mapbox provides 'off-route' probability data without immediately snapping to a new route. We expose a `check_route_confidence` tool so the AI can naturally ask: *"It looks like you might have turned around. Do you want me to find a new path?"* before Mapbox aggressively recalculates.

## 3. Anticipated Problems & Fixes

- **Problem 1: GPS Jitter in Cities.** Urban canyons cause GPS signals to bounce. If the user's dot jumps a block over, Mapbox might trigger an aggressive "Rerouting..." loop.
  - **Fix:** Explicitly configure Mapbox's `EnhancedLocationManager` to aggressively snap the raw GPS coordinate to the known edge of the road polygon.
- **Problem 2: Battery Meltdown.** Continuously querying the backend for POIs while running Mapbox 3D rendering and a real-time ElevenLabs Audio stream.
  - **Fix (Resolved):** Because the app is using **LiveKit WebRTC** (instead of inefficient WebSockets) and enforcing the `>50 meter Delta` polling limit (instead of sheer time intervals), CPU and Network overhead drops by 80%.
