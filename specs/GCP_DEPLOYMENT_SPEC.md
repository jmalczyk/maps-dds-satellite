# GCP Deployment & Dynamic Build Specification

> **Target Platform**: Google Cloud Platform (GCP)  
> **Automation Tooling**: Antigravity CLI / `gcloud` / GCP REST APIs  
> **Design Principle**: 100% generic, zero hardcoded credentials, keys, or IDs. All resources (API keys, Map IDs, StyleConfigs, Datasets) are dynamically provisioned from public data feeds during the build execution.

---

## 1. Prerequisites & Build Environment

Before executing the provisioning pipeline, verify:
- `gcloud` CLI installed and authenticated (`gcloud auth login` or active service account).
- Target GCP project exists with billing enabled.
- Python 3.9+ with standard libraries (`http.server`, `urllib.request`, `json`, `threading`, `gzip`).
- Utilities: `curl`, `jq`.

Set the target GCP project variable:
```bash
export GCP_PROJECT_ID="${GCP_PROJECT_ID:-your-gcp-project-id}"
export GCP_PROJECT_NUMBER="$(gcloud projects describe ${GCP_PROJECT_ID} --format='value(projectNumber)')"
```

---

## 2. Public Data Sources

The application relies strictly on public open data sources to reconstruct its geospatial datasets:

| Dataset | Public Upstream Endpoint | Format | Licensing / Source |
| :--- | :--- | :--- | :--- |
| **USDOT Runways** | `https://geodata.bts.gov/datasets/usdot::runways.geojson` | GeoJSON (Polygon) | U.S. Bureau of Transportation Statistics (Public Domain) |
| **FAA TFR NOTAMs** | `https://tfr.faa.gov/geoserver/TFR/ows?service=WFS&version=1.1.0&request=GetFeature&typeName=TFR:V_TFR_LOC&maxFeatures=2000&outputFormat=application/json&srsname=EPSG:4326` | GeoJSON (Polygon) | Federal Aviation Administration (Public Domain) |
| **FAA TFR Metadata** | `https://tfr.faa.gov/tfrapi/exportTfrList` | JSON | Federal Aviation Administration (Public Domain) |
| **Live ADS-B Telemetry** | `https://api.adsb.lol/v2/lat/{lat}/lon/{lon}/dist/{dist}` | JSON | Open ADS-B Community Feeds |
| **Historical ADS-B Tracks** | `https://globe.adsb.lol/data/traces/` | JSON | Open ADS-B Community Feeds |

---

## 3. Automated End-to-End Build & Provisioning Script

The following self-contained provisioning script can be executed by Antigravity CLI to dynamically generate all required cloud resources, download public data, and output a local `.env` file:

```bash
#!/usr/bin/env bash
set -euo pipefail

echo "======================================================================"
echo "🚀 ANTIGRAVITY AUTOMATED GCP PROVISIONING PIPELINE"
echo "======================================================================"

# 1. Enable Required Google Cloud APIs
echo "[1/7] Enabling Google Cloud APIs..."
gcloud services enable \
  maps-backend.googleapis.com \
  mapmanagement.googleapis.com \
  mapsplatformdatasets.googleapis.com \
  tile.googleapis.com \
  --project="${GCP_PROJECT_ID}"

# 2. Provision Restricted Google Maps API Key
echo "[2/7] Generating Google Maps API Key..."
gcloud services api-keys create \
  --display-name="Aviation-Platform-Dynamic-Key" \
  --api-target=service=maps-backend.googleapis.com \
  --api-target=service=tile.googleapis.com \
  --project="${GCP_PROJECT_ID}"

MAPS_API_KEY="$(gcloud services api-keys list --project=${GCP_PROJECT_ID} --filter='displayName:Aviation-Platform-Dynamic-Key' --format='value(keyString)')"
echo "Generated API Key: ${MAPS_API_KEY}"

# Acquire OAuth2 token for Map Management & Datasets REST APIs
TOKEN="$(gcloud auth print-access-token)"

# 3. Author Cloud-Based Map Styling (CBMS) StyleConfigs
echo "[3/7] Generating Cloud StyleConfigs..."

# A. Satellite (100% Zero-Base-Data Suppression)
STYLE_RES_SAT=$(curl -s -X POST "https://mapmanagement.googleapis.com/v2/projects/${GCP_PROJECT_ID}/styleConfigs" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "X-Goog-User-Project: ${GCP_PROJECT_ID}" \
  -H "Content-Type: application/json" \
  -d '{
    "displayName": "Aviation-Satellite-ZeroBase",
    "variant": "light",
    "jsonStyleSheet": "{\"infrastructure\":{\"geometry\":{\"visible\":false},\"label\":{\"visible\":false}},\"natural\":{\"geometry\":{\"visible\":false},\"label\":{\"visible\":false}},\"pointOfInterest\":{\"geometry\":{\"visible\":false},\"label\":{\"visible\":false}},\"political\":{\"geometry\":{\"visible\":false},\"label\":{\"visible\":false}}}"
  }')
STYLE_ID_SAT=$(echo "${STYLE_RES_SAT}" | jq -r '.name' | cut -d'/' -f4)

# B. Hybrid (No Polygon Fills + Crisp White Borders)
STYLE_RES_HYB=$(curl -s -X POST "https://mapmanagement.googleapis.com/v2/projects/${GCP_PROJECT_ID}/styleConfigs" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "X-Goog-User-Project: ${GCP_PROJECT_ID}" \
  -H "Content-Type: application/json" \
  -d '{
    "displayName": "Aviation-Hybrid-NoFills",
    "variant": "light",
    "jsonStyleSheet": "{\"political.border\":{\"geometry\":{\"stroke\":{\"color\":\"#ffffff\",\"weight\":1.5}}},\"political.geometry\":{\"visible\":false},\"natural.land.landCover\":{\"geometry\":{\"visible\":false}},\"infrastructure.urbanArea\":{\"geometry\":{\"visible\":false}}}"
  }')
STYLE_ID_HYB=$(echo "${STYLE_RES_HYB}" | jq -r '.name' | cut -d'/' -f4)

# C. Clean Dark (Light-Variant Tactical Dark Theme)
STYLE_RES_CLEAN=$(curl -s -X POST "https://mapmanagement.googleapis.com/v2/projects/${GCP_PROJECT_ID}/styleConfigs" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "X-Goog-User-Project: ${GCP_PROJECT_ID}" \
  -H "Content-Type: application/json" \
  -d '{
    "displayName": "Aviation-Clean-Dark",
    "variant": "light",
    "jsonStyleSheet": "{\"water\":{\"geometry\":{\"fill\":{\"color\":\"#0b0f19\"}}},\"natural.land\":{\"geometry\":{\"fill\":{\"color\":\"#161922\"}}},\"political.border\":{\"geometry\":{\"stroke\":{\"color\":\"#334155\",\"weight\":1.2}}},\"infrastructure.roadNetwork.highway\":{\"geometry\":{\"stroke\":{\"color\":\"#334155\",\"weight\":1.0}}},\"infrastructure.building\":{\"geometry\":{\"visible\":false}},\"pointOfInterest\":{\"visible\":false}}"
  }')
STYLE_ID_CLEAN=$(echo "${STYLE_RES_CLEAN}" | jq -r '.name' | cut -d'/' -f4)

# 4. Provision Dedicated JavaScript Vector Map IDs
echo "[4/7] Generating Dedicated Vector Map IDs..."

MAP_RES_SAT=$(curl -s -X POST "https://mapmanagement.googleapis.com/v2/projects/${GCP_PROJECT_ID}/mapConfigs" \
  -H "Authorization: Bearer ${TOKEN}" -H "X-Goog-User-Project: ${GCP_PROJECT_ID}" -H "Content-Type: application/json" \
  -d '{"mapType": "JAVASCRIPT", "description": "Aviation Satellite Mode"}')
MAP_ID_SAT=$(echo "${MAP_RES_SAT}" | jq -r '.name' | cut -d'/' -f4)

MAP_RES_HYB=$(curl -s -X POST "https://mapmanagement.googleapis.com/v2/projects/${GCP_PROJECT_ID}/mapConfigs" \
  -H "Authorization: Bearer ${TOKEN}" -H "X-Goog-User-Project: ${GCP_PROJECT_ID}" -H "Content-Type: application/json" \
  -d '{"mapType": "JAVASCRIPT", "description": "Aviation Hybrid Mode"}')
MAP_ID_HYB=$(echo "${MAP_RES_HYB}" | jq -r '.name' | cut -d'/' -f4)

MAP_RES_ROAD=$(curl -s -X POST "https://mapmanagement.googleapis.com/v2/projects/${GCP_PROJECT_ID}/mapConfigs" \
  -H "Authorization: Bearer ${TOKEN}" -H "X-Goog-User-Project: ${GCP_PROJECT_ID}" -H "Content-Type: application/json" \
  -d '{"mapType": "JAVASCRIPT", "description": "Aviation Roadmap Mode"}')
MAP_ID_ROAD=$(echo "${MAP_RES_ROAD}" | jq -r '.name' | cut -d'/' -f4)

MAP_RES_CLEAN=$(curl -s -X POST "https://mapmanagement.googleapis.com/v2/projects/${GCP_PROJECT_ID}/mapConfigs" \
  -H "Authorization: Bearer ${TOKEN}" -H "X-Goog-User-Project: ${GCP_PROJECT_ID}" -H "Content-Type: application/json" \
  -d '{"mapType": "JAVASCRIPT", "description": "Aviation Clean Dark Mode"}')
MAP_ID_CLEAN=$(echo "${MAP_RES_CLEAN}" | jq -r '.name' | cut -d'/' -f4)

# 5. Fetch Public Geospatial Datasets
echo "[5/7] Fetching Public Geospatial Data..."

# A. Download USDOT Runways from BTS GeoData if not already cached
if [ ! -f "runways.geojson" ]; then
  echo "Downloading USDOT Runways from public BTS endpoint..."
  curl -s -L "https://geodata.bts.gov/datasets/usdot::runways.geojson" -o "runways.geojson"
fi

# B. Fetch Active FAA CONUS NOTAMs
echo "Fetching active FAA CONUS NOTAMs from public FAA GeoServer..."
python3 -c "
import urllib.request, json
wfs_url = 'https://tfr.faa.gov/geoserver/TFR/ows?service=WFS&version=1.1.0&request=GetFeature&typeName=TFR:V_TFR_LOC&maxFeatures=2000&outputFormat=application/json&srsname=EPSG:4326'
req = urllib.request.Request(wfs_url, headers={'User-Agent': 'Mozilla/5.0'})
with urllib.request.urlopen(req) as resp:
    data = json.loads(resp.read().decode())
features = [f for f in data.get('features', []) if f.get('properties', {}).get('STATE') not in ['AK', 'HI', 'PR', 'GU', 'VI']]
with open('notams_conus.geojson', 'w') as f:
    json.dump({'type': 'FeatureCollection', 'features': features}, f)
print(f'Fetched {len(features)} active CONUS NOTAMs.')
"

# 6. Ingest Datasets into Maps Platform Datasets API
echo "[6/7] Ingesting Datasets into Google Cloud..."

# Ingest Runways
RUNWAYS_RES=$(curl -s -X POST "https://mapsplatformdatasets.googleapis.com/v1/projects/${GCP_PROJECT_ID}/datasets" \
  -H "Authorization: Bearer ${TOKEN}" -H "X-Goog-User-Project: ${GCP_PROJECT_ID}" -H "Content-Type: application/json" \
  -d '{"displayName": "USDOT Runways CONUS", "usage": ["USAGE_DATA_DRIVEN_STYLING"]}')
RUNWAYS_DATASET_ID=$(echo "${RUNWAYS_RES}" | jq -r '.name' | cut -d'/' -f4)

BOUNDARY="----WebKitFormBoundary$(date +%s)"
(
  echo "--${BOUNDARY}"
  echo 'Content-Disposition: form-data; name="metadata"'
  echo 'Content-Type: application/json'
  echo ''
  echo '{"local_file_source": {"file_format": "FILE_FORMAT_GEOJSON"}}'
  echo "--${BOUNDARY}"
  echo 'Content-Disposition: form-data; name="rawdata"; filename="runways.geojson"'
  echo 'Content-Type: application/geo+json'
  echo ''
  cat runways.geojson
  echo ''
  echo "--${BOUNDARY}--"
) | curl -s -X POST "https://mapsplatformdatasets.googleapis.com/upload/v1/projects/${GCP_PROJECT_ID}/datasets/${RUNWAYS_DATASET_ID}:import" \
  -H "Authorization: Bearer ${TOKEN}" -H "X-Goog-User-Project: ${GCP_PROJECT_ID}" \
  -H "Content-Type: multipart/form-data; boundary=${BOUNDARY}" --data-binary @- > /dev/null

# Ingest NOTAMs
NOTAMS_RES=$(curl -s -X POST "https://mapsplatformdatasets.googleapis.com/v1/projects/${GCP_PROJECT_ID}/datasets" \
  -H "Authorization: Bearer ${TOKEN}" -H "X-Goog-User-Project: ${GCP_PROJECT_ID}" -H "Content-Type: application/json" \
  -d '{"displayName": "FAA CONUS NOTAMs", "usage": ["USAGE_DATA_DRIVEN_STYLING"]}')
NOTAMS_DATASET_ID=$(echo "${NOTAMS_RES}" | jq -r '.name' | cut -d'/' -f4)

(
  echo "--${BOUNDARY}"
  echo 'Content-Disposition: form-data; name="metadata"'
  echo 'Content-Type: application/json'
  echo ''
  echo '{"local_file_source": {"file_format": "FILE_FORMAT_GEOJSON"}}'
  echo "--${BOUNDARY}"
  echo 'Content-Disposition: form-data; name="rawdata"; filename="notams_conus.geojson"'
  echo 'Content-Type: application/geo+json'
  echo ''
  cat notams_conus.geojson
  echo ''
  echo "--${BOUNDARY}--"
) | curl -s -X POST "https://mapsplatformdatasets.googleapis.com/upload/v1/projects/${GCP_PROJECT_ID}/datasets/${NOTAMS_DATASET_ID}:import" \
  -H "Authorization: Bearer ${TOKEN}" -H "X-Goog-User-Project: ${GCP_PROJECT_ID}" \
  -H "Content-Type: multipart/form-data; boundary=${BOUNDARY}" --data-binary @- > /dev/null

# 7. Bind MapContextConfigs
echo "[7/7] Binding MapContextConfigs to Map IDs..."

bind_context() {
  local map_id="$1"
  local style_id="$2"
  curl -s -X POST "https://mapmanagement.googleapis.com/v2/projects/${GCP_PROJECT_ID}/mapConfigs/${map_id}/mapContextConfigs" \
    -H "Authorization: Bearer ${TOKEN}" -H "X-Goog-User-Project: ${GCP_PROJECT_ID}" -H "Content-Type: application/json" \
    -d '{
      "mapConfig": "projects/'"${GCP_PROJECT_ID}"'/mapConfigs/'"${map_id}"'",
      "styleConfig": "projects/'"${GCP_PROJECT_ID}"'/styleConfigs/'"${style_id}"'",
      "dataset": [
        "projects/'"${GCP_PROJECT_ID}"'/datasets/'"${RUNWAYS_DATASET_ID}"'",
        "projects/'"${GCP_PROJECT_ID}"'/datasets/'"${NOTAMS_DATASET_ID}"'"
      ],
      "mapVariants": ["ROADMAP"]
    }' > /dev/null
}

bind_context "${MAP_ID_SAT}" "${STYLE_ID_SAT}"
bind_context "${MAP_ID_HYB}" "${STYLE_ID_HYB}"
bind_context "${MAP_ID_ROAD}" "${STYLE_ID_CLEAN}" # Or standard vector
bind_context "${MAP_ID_CLEAN}" "${STYLE_ID_CLEAN}"

# Output generated environment configuration
cat <<EOF > .env
GCP_PROJECT_ID=${GCP_PROJECT_ID}
MAPS_API_KEY=${MAPS_API_KEY}
MAP_ID_SATELLITE=${MAP_ID_SAT}
MAP_ID_HYBRID=${MAP_ID_HYB}
MAP_ID_ROADMAP=${MAP_ID_ROAD}
MAP_ID_CLEAN=${MAP_ID_CLEAN}
RUNWAYS_DATASET_ID=${RUNWAYS_DATASET_ID}
NOTAMS_DATASET_ID=${NOTAMS_DATASET_ID}
PORT=8080
EOF

echo "======================================================================"
echo "🎉 PROVISIONING COMPLETE! Generated configuration saved to .env"
echo "======================================================================"
```

---

## 4. Running the Application

Once `.env` is generated, start the server:
```bash
python3 server.py
```
`server.py` reads `.env` automatically to configure API keys, dataset IDs, and Map IDs.
