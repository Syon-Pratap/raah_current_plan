# Master Architecture Migration Summary: The V2 Pivot

**Purpose:** This document summarizes the exact architectural shifts the engineering team is making to prepare the RAAH codebase for a production-grade launch. It outlines what we are changing from the MVP across all core domains, why we are changing it, and how it dramatically increases speed, battery life, and system stability.

---

## 1. Navigation (Solving the Dual-GPS Flaw)
* **The Old Way:** The app used generic Apple Maps APIs and simultaneously ran multiple manual location trackers, which confused the system and severely drained the battery.
* **The New Change:** We are standardizing exclusively on the **Mapbox CoreNavigation SDK**. Crucially, Mapbox is established as the absolute single source of truth for all GPS telemetry. 
* **The Impact:** Prevents rogue location listeners from melting the battery. It provides a stunning, customizable 3D UI, and allows the AI to naturally read turn-by-turn guidance dynamically instead of using standard robotic map voices.

## 2. Database & POI Engine (Overture on Supabase)
* **The Old Way:** Attempting to execute static math formulas on the iPhone using raw Google Places API limits, which is illegal under Google's Terms of Service for non-Google Maps and astronomically expensive.
* **The New Change:** We are heavily adopting massive open-source **Overture Maps** (Foursquare) datasets and self-hosting them inside a master **Supabase PostgreSQL / PostGIS** database.
* **The Impact:** Zero marginal API costs for local places extraction. RAAH can pull limitless data about nearby cafes, ratings, and landmarks privately and instantly.

## 3. Auto-Narration (The 19+1 Geofence Engine)
* **The Old Way:** To proactively narrate a walking tour, the MVP constantly polled the GPS every few seconds to see what was nearby, which forces the app to crash or die in the background.
* **The New Change:** We implemented the **19+1 Geofence Pagination Rule**. The app downloads 19 POIs from Supabase and registers them natively as ultra-low-power iOS "Geofence Bubbles", surrounded by one massive 2-kilometer wide Bounding Box. The app then goes completely to sleep.
* **The Impact:** When a user walks near a Geofenced POI, iOS wakes the app up for just 10 seconds to read the fact to them. When they leave the 2km Box, the app gracefully fetches the next chunk of the city. Absolute zero continuous battery drain.

## 4. Long-Term Memory (RAG Vectorization)
* **The Old Way:** The iPhone attempted to manage user memory locally, taking up massive storage and struggling to search dynamic text quickly.
* **The New Change:** We moved all historical memory (e.g., allergies, art tastes, conversational habits) into a cloud vector database (**Supabase pgvector**). When the AI needs to know your tastes, it executes an instantaneous Cloud Tool Call (`search_user_memory`).
* **The Impact:** Infinite memory scale. The system calculates a dynamic formula combining Proximity, Recency, and Relevance entirely on the server. It injects exactly what you like into the AI instantly, ensuring RAAH remembers you flawlessly without bloating the phone.

## 5. Hyperlocal Web Search (Tavily Router)
* **The Old Way:** The iPhone directly pinged standard search engines, exposing our private API keys and resulting in 5-8 second delays as it tried to scrape web pages locally.
* **The New Change:** The iOS app now acts as a mediator, pushing search requests through a secure Supabase Edge Function connected to **Tavily**, an AI-native search engine.
* **The Impact:** Tavily instantly scrapes and summarizes top Reddit forums and local blogs into clean Markdown in **~1.5 seconds**, giving the AI true local wisdom without exposing our security keys or making the user wait in silence.

## 6. The Voice Engine (Zero Latency WebRTC)
* **The Old Way:** The app relied on traditional custom WebSockets to stream audio to ElevenLabs, which struggled with IP handovers (switching from WiFi to Cellular) and caused awkward robotic delays.
* **The New Change:** We are replacing the entire voice layer with the **LiveKit WebRTC Swift SDK**.
* **The Impact:** WebRTC is the gold standard for live video calls. This drops AI response latency to under 1 second, makes the voice connection highly resilient in subways/tunnels, and allows the user to naturally interrupt the AI mid-sentence without breaking the app.

## 7. UI Biological Reactivity (OrbView)
* **The Old Way:** The glowing "Orb" avatar on the screen ran on a hardcoded, fake 3-second math loop to pretend it was talking.
* **The New Change:** We bound the new WebRTC audio feed's LiveKit `audioLevel` directly into iOS scaling math. 
* **The Impact:** The Orb now physically grows, shrinks, and flares in exact real-time synchronization with the volume and syllables of the AI's actual voice. It creates a hyper-realistic, biologically reactive aesthetic that hovers over a gorgeous frosted-glass 3D Mapbox Map, distinguishing the product as truly premium.

## 8. Fixing the Split-Brain Syndrome (Centralizing AI)
* **The Old Way:** The MVP codebase was heavily fractured, attempting to run OpenAI Vision (the Snap and Ask camera feature) alongside ElevenLabs.
* **The New Change:** We completely removed the legacy AI camera code because attempting to constantly invoke the iOS camera lens while the phone is asleep causes fatal iOS background crashes.
* **The Impact:** RAAH is now a pure-audio, bulletproof companion. The iPhone operates as a lightweight, passive sensory organ feeding audio strictly to ElevenLabs, ensuring 100% stability in the user's pocket.

---
**Executive Conclusion:** 
By making these structural changes across all 8 major domains, we move RAAH from a "heavy, local MVP" to a "lightweight, edge-cloud application." This pivot guarantees that users can keep the app running in their pocket all day without their battery dying, experiencing flawless, uninterrupted voice companionship.
