# Google Maps Data-Driven Styling (DDS) on Satellite Layer

> **Antigravity CLI Prompt & Deployment Blueprint**  
> Autonomous specification package for building and deploying a Google Maps Platform application that demonstrates **Data-Driven Styling (DDS) vector datasets seamlessly overlaid on Google Maps satellite imagery**.

---

## 🎯 Purpose of This Demo

The primary objective of this demo is to demonstrate **Google Maps Data-Driven Styling (DDS) overlaid on a Google Maps satellite layer**.

In standard Google Maps implementations, vector datasets (DDS) cannot be natively rendered on top of raster satellite imagery without being obscured or unsupported. This project solves that challenge using **WebGL2 interleaved rendering** with **destination-over alpha compositing** (`blendFunc: [773, 1, 773, 1]`), allowing DDS vector datasets (USDOT Runways, FAA NOTAMs) to render crisply over satellite tiles alongside real-time 60 FPS ADS-B aircraft telemetry.

---

## 🤖 Antigravity Execution Prompt

To build and run this application from scratch in a clean Google Cloud environment, execute the following steps in order:

### 1. Review Technical Specifications
- [`specs/ARCHITECTURE_SPEC.md`](specs/ARCHITECTURE_SPEC.md): WebGL2 3-pass render pipeline, destination-over alpha compositing, and 60 FPS dead reckoning.
- [`specs/GCP_DEPLOYMENT_SPEC.md`](specs/GCP_DEPLOYMENT_SPEC.md): Automated provisioning script for APIs, Map IDs, CBMS styles, and public dataset ingestion.
- [`specs/APP_SPECIFICATION.md`](specs/APP_SPECIFICATION.md): Implementation details for `server.py`, `app.js`, `index.html`, and `style.css`.
- [`specs/MCP_SERVER_SPEC.md`](specs/MCP_SERVER_SPEC.md): MCP tool definitions for Maps, ADS-B, and NOTAM APIs.

### 2. Provision Cloud Resources
Execute the automated provisioning pipeline in [`specs/GCP_DEPLOYMENT_SPEC.md`](specs/GCP_DEPLOYMENT_SPEC.md):
1. Enable APIs: `maps-backend`, `mapmanagement`, `mapsplatformdatasets`, `tile`.
2. Generate a restricted Google Maps API key.
3. Create 3 CBMS styles (Satellite, Hybrid, Clean Dark with `"variant": "light"`).
4. Create 4 Vector Map IDs (`satellite`, `hybrid`, `roadmap`, `clean`).
5. Ingest public datasets (USDOT Runways, FAA TFR NOTAMs) via Maps Platform Datasets API.
6. Bind datasets and styles to Map IDs and generate `.env`.

### 3. Generate Application Code
Implement the application per [`specs/APP_SPECIFICATION.md`](specs/APP_SPECIFICATION.md):
- **`server.py`**: Python HTTP server with ADS-B caching proxy (3.0s TTL) and NOTAM polling daemon.
- **`app.js`**: Google Maps JS loader, Deck.gl interleaved overlay with destination-over blending, and 60 FPS aircraft animation.
- **`index.html` & `style.css`**: Multi-basemap selector, HUD feature inspector, and dark UI.

### 4. Launch & Verify
```bash
python3 server.py
```
Open `http://localhost:8080/?mapType=satellite` to verify DDS vector layers rendering over satellite imagery.

---

## ⚠️ Critical Rules & Pitfalls

1. **Light Variant Only for DDS**: Always author Cloud StyleConfigs with `"variant": "light"`. Setting `"variant": "dark"` causes the Datasets API to reject the style.
2. **Destination-Over Compositing**: In `app.js`, set `parameters: { blend: true, blendFunc: [773, 1, 773, 1] }` on the satellite `BitmapLayer` so it does not overwrite Pass 1 DDS vector polygons.
3. **Single Import Per Dataset**: Never re-import into an existing dataset. Create a fresh dataset resource per import to prevent `MapContextConfig` binding errors.
4. **No Hardcoded Secrets**: All API keys, Map IDs, and dataset IDs must be loaded dynamically from `.env` via `/api/config`.

---

## 📂 Repository Structure

```
.
├── README.md                  # Master Antigravity prompt
└── specs/
    ├── ARCHITECTURE_SPEC.md   # WebGL2 interleaving & compositing specs
    ├── GCP_DEPLOYMENT_SPEC.md # Automated GCP provisioning & public feeds
    ├── APP_SPECIFICATION.md  # Frontend & backend implementation specs
    └── MCP_SERVER_SPEC.md     # MCP tool definitions
```
