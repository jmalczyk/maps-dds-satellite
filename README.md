# Google Maps Data-Driven Styling (DDS) Real-Time Aviation Platform

> **Antigravity CLI Prompt & Deployment Blueprint**  
> This repository contains the complete architectural, deployment, and codebase specifications to reconstruct and deploy the **Google Maps Platform Data-Driven Styling (DDS) Real-Time Aviation Platform** in a fresh Google Cloud environment.

---

## 🎯 Purpose of This Repository

The purpose of this repository is to serve as the master specification and prompt package for **Antigravity CLI** to autonomously build, configure, and launch a high-performance geospatial intelligence web application.

The application unifies:
1. **Google Maps Platform Data-Driven Styling (DDS)**: Vector datasets (USDOT Runways and FAA CONUS Temporary Flight Restrictions) rendered natively inside Google Maps WebGL vector engine with client-side declarative styling.
2. **WebGL2 Interleaved Overlays**: 2D Satellite/Aerial raster imagery and thousands of moving aircraft rendered inside Google Maps' own `WebGLOverlayView.onDrawVector()` pass via `deck.GoogleMapsOverlay({ interleaved: true })` with zero lag and zero camera shearing.
3. **Continuous 60 FPS ADS-B Aircraft Tracking**: Dynamic $+30\%$ buffered spatial queries streaming real-time ADS-B transponder telemetry from public open feeds, forward-projected upon arrival, and animated smoothly at 60 FPS with decoupled drift reconciliation.
4. **Live Tactical HUD & Anchored InfoWindow**: Real-time attribute inspection (Altitude, Ground Speed, Heading, Vertical Rate, GPS Coordinates, Telemetry Sequence) with visual pulse feedback on telemetry updates.

All resources (API keys, Map IDs, StyleConfigs, and Datasets) are designed to be dynamically provisioned from **public open data sources** during the build process, with zero hardcoded credentials or project-specific IDs.

---

## 🤖 Antigravity Execution Instructions

When using this repository as an autonomous prompt for **Antigravity CLI**, follow these execution phases in sequential order:

### Phase 1: Review Specification Documents
Before generating code or executing cloud provisioning commands, thoroughly inspect all files in the `specs/` directory:
- [`specs/ARCHITECTURE_SPEC.md`](specs/ARCHITECTURE_SPEC.md): Read this first to understand the WebGL2 3-pass render pipeline, destination-over alpha compositing (`blendFunc: [773, 1, 773, 1]`), 60 FPS dead-reckoning mathematics, and multi-dataset integration.
- [`specs/GCP_DEPLOYMENT_SPEC.md`](specs/GCP_DEPLOYMENT_SPEC.md): Detailed provisioning commands for enabling Google Cloud APIs, generating API keys, authoring Cloud StyleConfigs, creating 4 dedicated Vector Map IDs, downloading public data, and importing datasets.
- [`specs/APP_SPECIFICATION.md`](specs/APP_SPECIFICATION.md): File-by-file technical breakdown of `index.html`, `app.js`, `style.css`, and `server.py`.
- [`specs/MCP_SERVER_SPEC.md`](specs/MCP_SERVER_SPEC.md): Tool schemas and configuration for Google Maps Platform, ADS-B telemetry, and FAA NOTAM MCP servers.

---

### Phase 2: Automated Cloud Provisioning
Execute the automated provisioning pipeline described in [`specs/GCP_DEPLOYMENT_SPEC.md`](specs/GCP_DEPLOYMENT_SPEC.md):

1. **Set Environment Variables**:
   ```bash
   export GCP_PROJECT_ID="your-target-project-id"
   ```
2. **Enable Required Google Cloud APIs**:
   - `maps-backend.googleapis.com` (Maps JavaScript API)
   - `mapmanagement.googleapis.com` (Map Management API v2)
   - `mapsplatformdatasets.googleapis.com` (Maps Platform Datasets API v1)
   - `tile.googleapis.com` (2D Map Tiles API)
3. **Provision Google Maps API Key**:
   - Create a restricted browser key and export as `MAPS_API_KEY`.
4. **Author 3 Cloud-Based Map Styling (CBMS) StyleConfigs**:
   - **Satellite**: 100% suppression of all 135 base features.
   - **Hybrid**: Suppress 60 polygon fills; retain white borders (`#ffffff`) and road lines.
   - **Clean Dark**: Author with `"variant": "light"` and midnight dark palette (`#0b0f19` water, `#161922` land, `#334155` borders/roads).
5. **Provision 4 Dedicated Vector Map IDs**:
   - Create Map IDs for `satellite`, `hybrid`, `roadmap`, and `clean` modes.
6. **Fetch Public Datasets**:
   - Download USDOT Runways from Bureau of Transportation Statistics:  
     `https://geodata.bts.gov/datasets/usdot::runways.geojson`
   - Fetch active FAA CONUS NOTAMs from FAA TFR WFS GeoServer:  
     `https://tfr.faa.gov/geoserver/TFR/ows?service=WFS&version=1.1.0&request=GetFeature&typeName=TFR:V_TFR_LOC&maxFeatures=2000&outputFormat=application/json&srsname=EPSG:4326`
7. **Ingest Datasets via Maps Platform Datasets API**:
   - Create dataset resources with `"usage": ["USAGE_DATA_DRIVEN_STYLING"]`.
   - Upload GeoJSON files via multipart `:import`.
8. **Bind MapContextConfigs**:
   - Link both datasets and the respective StyleConfig to each Map ID.
9. **Generate `.env` File**:
   - Write all generated IDs, keys, and ports to a local `.env` configuration file.

---

### Phase 3: Application Code Generation
Generate the application codebase following [`specs/APP_SPECIFICATION.md`](specs/APP_SPECIFICATION.md):

1. **`server.py`**:
   - Python HTTP server loading `.env` on startup and exposing `/api/config`.
   - Upstream ADS-B proxy (`api.adsb.lol`) with a 3.0-second TTL in-memory cache and bounding box spatial filtering.
   - Background 60-second FAA NOTAM sync daemon with SHA-256 delta detection.
   - 2D Map Tiles session proxy (`/api/tiles-session`).
2. **`index.html`**:
   - Multi-map container divs (`#map-satellite`, `#map-hybrid`, `#map-roadmap`, `#map-clean`).
   - Glassmorphic control panel with layer toggles and opacity sliders.
   - Floating HUD Feature Inspector (`#feature-inspector`) with tagged attribute cells.
3. **`app.js`**:
   - Offscreen 2D canvas sprite atlas generator for 7 aircraft classifications.
   - `getMapBufferedFootprint()` calculating $+30\%$ spatial buffer on current viewport.
   - Chained continuous polling loop (`fetchLiveAircraftData()`).
   - 60 FPS dead-reckoning loop advancing aircraft along velocity vectors with decoupled drift decay.
   - Deck.gl interleaved overlay with destination-over blending for satellite tiles (`blendFunc: [773, 1, 773, 1]`) and `updateTriggers` for GPU vertex re-upload.
   - Live synchronization of `#feature-inspector` and anchored `google.maps.InfoWindow`.
4. **`style.css`**:
   - Dark glassmorphism theme (`backdrop-filter: blur(12px)`).
   - `.meta-value-updated` keyframe pulse animation.
   - Custom dark styling for `google.maps.InfoWindow`.

---

### Phase 4: Launch & Verification
1. **Start the Application Server**:
   ```bash
   python3 server.py
   ```
2. **Verify in Browser**:
   Open `http://localhost:8080/?mapType=clean` to verify basemaps, datasets, and aircraft streaming.
3. **Execute Automated Verification Suite**:
   Run headless Chrome CDP test script to validate buffered footprint querying, aircraft selection, and real-time infowindow telemetry updates.

---

## ⚠️ Critical Technical Pitfalls to Avoid

1. **Clean Dark Map Variant Rejection**:
   - *Problem*: Setting `"variant": "dark"` in CBMS causes the Datasets API to reject the style (`Datasets are not supported with dark map variants`).
   - *Solution*: Always set `"variant": "light"` and style geometries dark (`#0b0f19` water, `#161922` land).
2. **Edith MapLayers Multi-Version Invalidation Bug**:
   - *Problem*: Re-importing onto a dataset creates multiple `MapLayer` rows, causing `MapContextConfig` creation to fail with `InvalidArgumentError: All datasets must belong to the same project as the MapView`.
   - *Solution*: Always create a fresh dataset resource (`POST /datasets`) with a single completed import before binding to `MapContextConfig`.
3. **Satellite Overwriting Vector Polygons**:
   - *Problem*: Deck.gl raster satellite tiles in Pass 2 overwrite Pass 1 ground polygons (Runways and NOTAMs).
   - *Solution*: Use destination-over alpha compositing on satellite `BitmapLayer`: `parameters: { blend: true, blendFunc: [773, 1, 773, 1] }`.
4. **Upstream ADS-B HTTP 429 Rate Limiting**:
   - *Problem*: Polling `api.adsb.lol` every second causes nginx rate limiting (`HTTP 429`).
   - *Solution*: Cache responses in `server.py` per footprint with a 3.0s TTL and filter by bounding box.
5. **Frozen Aircraft Between Polling Cycles**:
   - *Problem*: Deck.gl shallow equality check ignores in-place coordinate mutations.
   - *Solution*: Add `updateTriggers: { getPosition: animFrameCounter, getAngle: animFrameCounter }` to force GPU vertex buffer re-uploads on every frame.

---

## 📂 Repository Contents

```
.
├── README.md                  # This file (Antigravity CLI prompt & blueprint)
└── specs/
    ├── ARCHITECTURE_SPEC.md   # WebGL2 interleaving, compositing & math specs
    ├── GCP_DEPLOYMENT_SPEC.md # Automated GCP provisioning script & public feeds
    ├── MCP_SERVER_SPEC.md     # Model Context Protocol v1.0 tool definitions
    └── APP_SPECIFICATION.md  # Detailed frontend and backend codebase breakdown
```
