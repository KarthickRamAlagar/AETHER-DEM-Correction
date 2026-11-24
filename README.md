# AETHER GIS: Terrain-Aware DEM Correction and SDG Flood-Exposure Decision Support
 
Machine-learning correction of CartoDEM (30 m) elevation error, validated on ground the model has never seen, with a flood what-if and relief-cost estimate built on the corrected height, and a secured, monitored prototype around it.
 
**Author:** KARTHICKRAMALAGAR R, M.Tech, Amrita School of Computing, Amrita Vishwa Vidyapeetham, Bengaluru
 

 
## 1. The Problem
 
India's national DEM (CartoDEM, 30 m, from Cartosat stereo imagery) is the base for flood mapping, population exposure and relief budgeting. Against 1,221,354 ICESat-2 laser heights it reads **2.97 m too high** (RMSE 6.07 m on the frozen test strip), and the error is largest on rough terrain. Every flooded-area, people and rupee figure built on the map inherits that error.
 
## 2. The approach
 
The model predicts the **error**, not the height (residual learning):
 
```
ΔZ           = Z_ref − Z_raw          (Z_ref = ICESat-2 ATL08 laser height)
Z_corrected  = Z_raw + ΔZ_predicted
```
 
A LightGBM regressor (v2.0.0, Optuna-tuned, 183 trees, learning rate 0.0206, 30 leaves, L2 penalty 27.13) predicts ΔZ from **29 terrain and optical features** (slope, aspect, TRI, TPI, curvature, hillshade, NDVI/NDWI/NDSI/EVI/BSI, Sentinel-2 bands, 3×3 and 5×5 neighbourhood statistics, terrain region class).
 
## 3. Headline results
 
All scores are on **spatially separated test data** (10 km blocks, 5 km buffer, 5 longitude strips). Strip 3 (259,260 points) is frozen: never used for fitting, tuning or model choice, and scored once. The selection rule was written on 13 Oct 2026, before tuning.
 
| Measure (frozen strip) | Raw CartoDEM | Constant shift | Best IDW (k=128) | **Proposed (V2)** |
|---|---|---|---|---|
| RMSE (m) | 6.068 | 5.313 | 4.351 | **4.056** |
| Gain of proposed | −33.2% | −23.7% | −6.8% (95% CI 3.7–9.8%) | — |
 
- 4-strip spatial CV (962,094 rows): V1 default 3.083 m → **V2 tuned 2.982 m**.
- Speed on the same 259,260 points: 1.94 s against 5.96 s for IDW (3.1× faster). Model file 5.04 MB.
- Prediction interval coverage 85.5% against 90% promised (residuals are heavy-tailed; reported as is).
- Radar (Sentinel-1) features add −0.009 m and are 71–90% missing, so they were dropped from the 29-feature set.
- A 3-model stack (Ridge + LightGBM + CatBoost) was tested and is **worse** than the single tuned LightGBM on spatial splits.
## 4. System: three engines
 
| Engine | Role | Main parts |
|---|---|---|
| **1. DEM correction** | Corrects any location's height, with a 90% interval and TreeSHAP reasons | LightGBM model, FastAPI, React UI, PDF reports |
| **2. SDG decision support** | Flood what-if on the corrected DEM → people, cropland, services → relief estimate | HAND rule, Gumbel return levels, quantile GBM, SDG 2 / 11 / 12 pages |
| **3. Security and monitoring** | Gateway, cache, observability, access control | Node + Express gateway :3100, Redis, Prometheus, Grafana, NGINX, PostgreSQL |
 
Flow: raw DEM → Engine 1 corrects → Engine 2 turns the corrected height into exposure and relief → Engine 3 protects and watches both.
 
### Engine 2 results (Basti / Siddharthnagar, 2 m river rise)
 
| Quantity | Raw DEM | Corrected DEM |
|---|---|---|
| People in flood | 32,374 | 50,232 (+55%) |
| Cropland (ha) | 5,874 | 6,773 (+15%) |
| Relief need (INR crore, low–high) | 7.33–12.76 | 9.38–15.82 |
 
Relief need = households × 5,000 + hectares × crop-input subsidy (India SDRF norms 2022–26). These are **sensitivity results and planning estimates**, not proof of better flood prediction and not a release decision. A local-inertial physics solver was built and tested; it failed its gate (won 2 of 6 scenes) and is **not adopted**. The "future prediction" view is a what-if, labelled not operational.
 
### Engine 3 results
 
- 520 checks passed, 0 failed (evidence pack, 25 Oct 2026); gateway tests 19 of 19.
- Helmet, CORS allow-list, Zod input checks, rate limit 300 per minute, GET/POST only, read-only database role, Windows firewall rule.
- Cache effect on this machine: median first call 29.5 ms, repeat 6.0 ms; `/api/dem/comparison/2022` 8,282.6 ms → 3.9 ms.
## 5. Data
 
| Dataset | Provider | Role |
|---|---|---|
| ICESat-2 ATL08 (1,005 granules) | NASA / NSIDC | Laser-measured true height (target) |
| CartoDEM V3 (12 tiles, 30 m) | ISRO / NRSC | Baseline elevation map being corrected |
| Sentinel-2 L2A (6 bands), Sentinel-1 GRD | ESA / Copernicus | Optical and radar features |
| Copernicus GLO-30 | ESA | Global baseline for comparison |
| Copernicus GFM | EODC / Copernicus | Observed flood extent (flood test) |
| WorldPop, VIDA, ESA WorldCover | WorldPop, Google + Microsoft, ESA | People, buildings, cropland |
| CHIRPS, GloFAS | UCSB, ECMWF / Copernicus | Rain and river flow |
| OpenStreetMap, geoBoundaries | OSM, geoBoundaries | Roads, schools, health sites, districts |
| Census 2011, MHA SDRF norms, MoHA Nepal norms | Govt. of India / Nepal | Household size, unit costs |
 
Pipeline: bronze (raw, read-only, SHA-256) → silver (aligned 30 m grid, KD-tree sampling) → gold (**1,221,354 rows × 63 columns**, 29 model inputs 100% filled, duplicate rate 0.021%). 19,123 files hashed; lineage manifest and model registry kept.
 
Study area: Gangetic Plain and foothills, 25–28 N, 80–84 E (validated box), with a separate experiment on the Nepal hills north of 27.5 N.
 
## 6. Repository layout
 
```
server/        FastAPI backend (prediction, EDA, SDG routes)
client/        React + Tailwind frontend (Elevation Correction, New Inputs, SDG pages, Settings, Ops)
model/         Training, validation, prediction and raster-correction code
etl/           Data download, feature extraction, golden dataset build, ops loaders
eda/           Exploratory analysis
gateway/       Node + Express gateway (security, cache, metrics)
ops/           Service configs and RUN_AND_CHECK.md (run and verify all services)
evidence/      Evidence packs (checks, proofs)
```
 
## 7. Tech stack
 
- **Backend:** Python, FastAPI, DuckDB, LightGBM, Optuna, scikit-learn, CatBoost (baselines), SHAP; managed with `uv`
- **Frontend:** React, Tailwind CSS, jsPDF (reports), voice assistant and guided application tour
- **Gateway and ops:** Node.js, Express, Redis (Memurai), PostgreSQL 18, MLflow, Prometheus, Grafana, NGINX, windows_exporter
- **LLM helper:** "Describe-it" auto-fill of feature values from a plain-text description
- **Platform:** developed and tested on Windows (native, no Docker)
## 8. Quick start
 
```bash
# 1. Python environment
uv sync
 
# 2. Backend (port 8000)
uv run uvicorn server.main:app --reload --port 8000
 
# 3. Frontend
cd client
npm install
npm run dev
```
 
Gateway, database, cache and monitoring services: follow `ops/RUN_AND_CHECK.md`. Secrets (admin passwords, API keys) live only in `gateway/.env` and local `.env` files, which are git-ignored. Large data (feature rasters, golden dataset) are not stored in the repository.
 
## 9. Method summary
 
1. **Golden dataset:** ICESat-2 heights joined to CartoDEM and 43 terrain/optical/radar layers; leakage columns removed (reference elevation, signed and absolute error, date proxies).
2. **Selection:** nine feature groups, four tests; F43 → F33 → F31 → **F29** (parity-safe: same inputs available at prediction time).
3. **Models:** 14 baselines, 4 ensembles, 3 searches; Optuna-tuned LightGBM chosen by pooled 4-strip spatial CV; tie band 1%, then lower worst fold, fewer models, lower cost.
4. **Validation:** spatial blocks, buffer, frozen strip scored once; bootstrap 95% CI and Wilcoxon signed-rank for the gain.
5. **Explainability:** TreeSHAP (top drivers: slope_mean_5×5, slope_std_5×5, raw height, terrain region class).
6. **Flood and relief:** HAND rule (flooded if height above nearest drainage ≤ river rise), Gumbel return levels, quantile (pinball) GBM bounds, exposure by district, relief arithmetic with a low/high unit-cost range.
## 10. Limitations (stated openly)
 
- Validated on the Gangetic Plain and foothills; behaviour in high mountains is untested.
- Intervals under-cover (85.5% vs 90%) because errors are heavy-tailed.
- "Cheaper" means compute and storage, not money; no rupee cost was measured.
- Relief figures are estimates from unit-cost ranges (Census 2011 household size; 2015–20 and 2022–26 norms); the low/high range is not statistical uncertainty.
- The physics-based flood solver did not pass its gate and is not used; flood extent skill against Copernicus GFM is unchanged between raw and corrected DEM (IoU 0.0050).
- Prototype is desktop only, runs on one machine, and has no user login by design.
## 11. Status
 
Engines 1–3 built and verified on the local machine; thesis, IEEE TGRS paper and Nature Sustainability drafts prepared from these results. Open items: real-device XR/map checks, remaining road/school/hospital layers.
 
## 12. Citation
 
KARTHICKRAMALAGAR R, *Terrain-Aware Residual Learning for Correcting CartoDEM Elevation Error and SDG Flood-Exposure Decision Support Using Multi-Source Remote Sensing Features*, M.Tech thesis, Amrita School of Computing, Amrita Vishwa Vidyapeetham, Bengaluru, 2026.
 
 
