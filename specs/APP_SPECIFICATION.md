# Application Codebase Specification: Frontend & Backend Components

> **Application**: Real-Time Aviation Tracking & Geospatial Intelligence  
> **Source Directory**: `maps_dds_satellite_demo/`  
> **Target Framework**: Vanilla JavaScript (ES2022), WebGL2, Deck.gl v8.9+, Google Maps JavaScript API v3, Python 3.9+.

---

## 1. Directory Structure

```
maps_dds_satellite_demo/
├── index.html                 # Main single-page application interface
├── app.js                     # Core frontend application logic & WebGL interleaving
├── style.css                  # Futuristic tactical glass HUD styling
├── server.py                  # Python backend HTTP server, proxy, and FAA daemon
├── runways.geojson            # USDOT Runways GeoJSON (8,796 polygon features)
├── notams_conus.geojson       # FAA CONUS Active NOTAMs GeoJSON
├── deck.gl.min.js             # Standalone Deck.gl bundle
├── specs/                     # System and deployment specifications
│   ├── ARCHITECTURE_SPEC.md   # System architecture and WebGL compositing rules
│   ├── GCP_DEPLOYMENT_SPEC.md # Step-by-step GCP / Antigravity provisioning
│   ├── MCP_SERVER_SPEC.md     # Model Context Protocol server definitions
│   └── APP_SPECIFICATION.md  # This codebase specification
└── skills/                    # Antigravity CLI skills
    └── google-maps-dds-aviation/
        └── SKILL.md           # Antigravity Skill playbook
```

---

## 2. Frontend: `index.html`

### Key DOM Containers & Elements
1. **Multi-Map Containers**:
   - `<div id="map-satellite" class="map-layer"></div>`: Satellite container.
   - `<div id="map-hybrid" class="map-layer"></div>`: Hybrid container.
   - `<div id="map-roadmap" class="map-layer"></div>`: Roadmap container.
   - `<div id="map-clean" class="map-layer"></div>`: Clean Dark container.
2. **Tactical Control Panel** (`.control-panel`):
   - **Base MapType Switcher**: Radio buttons/tabs for `Satellite`, `Hybrid`, `Roadmap`, `Clean`.
   - **Datasets & Layers Section**:
     - `FAA CONUS NOTAMs`: Toggle checkbox, opacity slider, manual refresh button, status badge (`#notams-badge`).
     - `USDOT Runways`: Toggle checkbox, opacity slider, count badge (`8,796 Runways`).
     - `Live Aircraft (Air & Ground)`: Toggle checkbox, opacity slider, active count badge (`#aircraft-badge`), streaming status indicator (`#aircraft-poll-status`).
3. **Feature Inspection Floating HUD** (`#feature-inspector`):
   - Header with `#inspector-tag`, `#inspector-title`, and `#close-inspector-btn`.
   - Body with `#inspector-attributes` holding table cells tagged with IDs (`#ac-meta-alt`, `#ac-meta-speed`, `#ac-meta-heading`, `#ac-meta-vert`, `#ac-meta-status`, `#ac-meta-coords`, `#ac-meta-seq`).
4. **Configuration Modal** (`#config-modal`):
   - Inputs for API Key, Map IDs (`satellite`, `hybrid`, `roadmap`, `clean`), and Dataset IDs (`runways`, `notams`).

---

## 3. Frontend: `app.js`

### Key Modules & Functions

#### 1. Canvas 2D Sprite Atlas Generator
- `generateAircraftAtlas()`: Creates an offscreen 448x64 canvas rendering 7 distinct aircraft silhouettes:
  - Heavy Widebody (`B77W`, `B748`, `A388`, `A359`).
  - Narrowbody Airliner (`B738`, `A321`, `A320`).
  - Regional Jet (`CRJ9`, `E175`).
  - Business Jet (`GLF6`, `C680`).
  - General Aviation Prop (`C172`, `SR22`).
  - Rotorcraft / Helicopter (`UH60`, `EC45`, `B407`).
  - Military Tactical (`F18`, `F16`, `F35`, `B52`).

#### 2. Buffered Viewport Footprint Calculation
- `getMapBufferedFootprint()`:
  - Retrieves current `google.maps.LatLngBounds`.
  - Expands bounds by `+30%` ($\Delta\text{lat} \times 0.30$, $\Delta\text{lon} \times 0.30$).
  - Derives radius in nautical miles ($35\text{ nm} \le R \le 1200\text{ nm}$).
  - Returns `{ lat, lon, dist, south, north, west, east }`.

#### 3. Continuous ADS-B Streaming & Forward Projection
- `fetchLiveAircraftData()`:
  - Appends `getMapBufferedFootprint()` query parameters to `/api/aircraft`.
  - Calculates latency: $\Delta t = (\text{serverNow} - \text{posTimestamp}) + \Delta t_{\text{network}}$.
  - Projects coordinates forward along heading vector:
    $$\Delta\text{lat} = \frac{v \cdot \Delta t \cdot \cos(\theta)}{111139}, \quad \Delta\text{lon} = \frac{v \cdot \Delta t \cdot \sin(\theta)}{111139 \cdot \cos(\text{lat})}$$
  - Calls `updateSelectedAircraftMetadata(selAc, res.sequence)` if an aircraft is currently selected.
- `continuousAircraftPoll()`: Chained async loop (`setTimeout(continuousAircraftPoll, 1000)` in `finally`).

#### 4. 60 FPS Dead-Reckoning Animation Loop
- `startAircraftAnimationLoop()`:
  - Runs inside `requestAnimationFrame`.
  - Advances aircraft coordinates continuously along velocity vector:
    $$\text{coord}[0] += \frac{\text{speedDegSec} \cdot \sin(\theta)}{\cos(\text{lat})} \cdot dt, \quad \text{coord}[1] += \text{speedDegSec} \cdot \cos(\theta) \cdot dt$$
  - Decoupled exponential decay of drift corrections.
  - Updates `#ac-meta-coords` and `selectedMapInfoWindow.setPosition()` at 60 FPS.
  - Invokes `updateDeckLayers(true)` to trigger WebGL2 redraw.

#### 5. Deck.gl Layer Interleaving
- `getDeckLayers(mode)`:
  - In Satellite/Hybrid: `TileLayer` streaming 2D satellite tiles with destination-over blending: `parameters: { blend: true, blendFunc: [773, 1, 773, 1] }`.
  - `PathLayer`: Selected aircraft complete flight path (`depthTest: false`).
  - `ScatterplotLayer`: Discrete luminous dots marking reported GPS fixes along path.
  - `IconLayer`: Aircraft silhouettes rendered from sprite atlas with `updateTriggers: { getPosition: animFrameCounter, getAngle: animFrameCounter }`.

#### 6. Live Infowindow & HUD Telemetry Synchronization
- `handleAircraftFeatureClick(info)`: Populates `#feature-inspector`, opens `#selectedMapInfoWindow`, and triggers `fetchCompleteFlightTrack(d.hex)`.
- `updateSelectedAircraftMetadata(d, seq)`: Updates `#ac-meta-alt`, `#ac-meta-speed`, `#ac-meta-heading`, `#ac-meta-vert`, `#ac-meta-coords`, `#ac-meta-seq` and triggers `.meta-value-updated` keyframe pulse.

---

## 4. Frontend: `style.css`

### Key Visual Styling & Animations
1. **Glassmorphic Tactical Theme**:
   - Translucent background: `rgba(15, 23, 42, 0.85)` with `backdrop-filter: blur(12px)`.
   - Electric accents: Cyan (`#06b6d4`), Sky Blue (`#38bdf8`), Gold (`#f59e0b`), Green (`#22c55e`), Crimson (`#ef4444`).
2. **Telemetry Pulse Animation**:
   ```css
   @keyframes telemetry-pulse {
     0% { background-color: rgba(34, 197, 94, 0.45); color: #4ade80; text-shadow: 0 0 8px rgba(74, 222, 128, 0.6); }
     100% { background-color: transparent; color: inherit; text-shadow: none; }
   }
   .meta-value-updated {
     animation: telemetry-pulse 1.2s ease-out;
     border-radius: 4px;
     padding: 1px 4px;
     display: inline-block;
   }
   ```
3. **Dark Google Maps InfoWindow**:
   ```css
   .gm-style .gm-style-iw-c {
     background: rgba(15, 23, 42, 0.95) !important;
     border: 1px solid rgba(14, 165, 233, 0.4) !important;
     box-shadow: 0 10px 25px -5px rgba(0, 0, 0, 0.6), 0 0 15px rgba(14, 165, 233, 0.25) !important;
     border-radius: 8px !important;
     color: #f1f5f9 !important;
     backdrop-filter: blur(12px) !important;
   }
   ```

---

## 5. Backend: `server.py`

### Key Modules & Threads
1. **Footprint Aircraft Fetcher** (`fetch_aircraft_for_footprint`):
   - Queries `https://api.adsb.lol/v2/lat/{lat}/lon/{lon}/dist/{dist}`.
   - Bounding box filtering: `south <= lat <= north`, `west <= lon <= east`.
   - In-memory cache with 3.0-second TTL per footprint (`cache_key = f"{lat}_{lon}_{dist}"`).
   - Graceful fallback to CONUS baseline cache on upstream errors/timeouts.
2. **Background Footprint Refresher** (`background_aircraft_worker`):
   - Periodically refreshes `_active_footprint` every 4.0 seconds in the background.
3. **FAA NOTAM Sync Pipeline** (`sync_notams_pipeline`):
   - Fetches WFS GeoServer (`TFR:V_TFR_LOC`) and metadata (`tfrapi/exportTfrList`).
   - Filters to CONUS active restrictions.
   - Computes SHA-256 fingerprint; only imports to GCP Datasets API when a delta occurs.
4. **HTTP Request Handler** (`Handler`):
   - `GET /api/aircraft`: Parses `lat, lon, dist, south, north, west, east`.
   - `GET /api/aircraft/{hex}/track`: Returns full historical track coordinates and fixes.
   - `GET /api/notams/status`: Returns sync status and active count.
   - `POST /api/notams/refresh`: Forces immediate NOTAM sync.
   - `GET /api/tiles-session`: Creates 2D Map Tiles session.
