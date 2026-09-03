# Aegis Orbital

> **“See the collision risk before the close approach.”**

Aegis Orbital is a near-real-time satellite conjunction and collision-risk visualization dashboard designed for Space Situational Awareness (SSA). It ingests publicly available Two-Line Element (TLE) orbital data, propagates satellite and orbital debris trajectories using analytical SGP4 mechanics, screens for close approaches across a configurable prediction window, evaluates a transparent simplified collision-risk heuristic, and visualizes the complete orbital environment in an interactive Three.js 3D mission-control interface.

---

## 1. Overview & Hackathon Judge Pitch

In low Earth orbit (LEO), over 30,000 tracked objects travel at speeds exceeding 7.5 km/s (27,000 km/h). At these orbital encounter velocities, even a millimeter-sized paint fleck carries the kinetic energy of a bowling ball at highway speeds, while a collision between two satellites can generate thousands of fragments, triggering a cascading Kessler syndrome.

**Aegis Orbital** provides space operators and mission analysts with an accessible, high-performance monitoring tool that answers live:
- What orbital objects are currently being tracked around Earth?
- Where are satellites and debris clusters located right now in 3D?
- Which objects are converging on crossing trajectories?
- When and where will the Time of Closest Approach (TCA) occur?
- What is the refined miss distance and relative encounter velocity?
- Why is a specific conjunction classified as LOW, GUARDED, ELEVATED, HIGH, or CRITICAL?

---

## 2. Architecture

```mermaid
flowchart TD
    subgraph Data Sources
        CT[CelesTrak GP/TLE Public API]
        ST[Space-Track API - Optional Credentials]
        DemoData[(Frozen Offline Demo Catalog)]
    end

    subgraph Backend Core (Python / FastAPI)
        Ingest[Ingestion Service & TLE Checksum Validator]
        Cache[(Local SQLite Cache: data/aegis_orbital.sqlite)]
        SGP4[Vallado SGP4 Propagation Engine]
        Frames[Coordinate Transformations: TEME -> ECEF -> WGS-84]
        Screening[Spatial & Altitude Pre-filter + Coarse Scan]
        Brent[Brent's Numerical TCA Refinement]
        RiskEngine[Transparent Collision-Risk Model]
        FastAPI[FastAPI REST API Layer]
        Scheduler[Background Screening Worker Task]
    end

    subgraph Frontend Mission Control (React / Three.js / TypeScript)
        Store[Zustand Simulation State Store]
        Canvas[Three.js / React Three Fiber 3D Viewport]
        Earth[Realistic Earth & Atmospheric Fresnel Glow]
        Instanced[InstancedMesh 60fps Bulk Satellite Markers]
        Trails[3D Keplerian Orbit Trajectories & Encounter Beacons]
        UI[Mission Control HUD: Metrics, Conjunction Table, Telemetry Panels, Timeline Scrubber]
    end

    CT --> Ingest
    ST -.-> Ingest
    DemoData --> Cache
    Ingest --> Cache
    Cache --> SGP4
    SGP4 --> Frames
    Frames --> Screening
    Screening --> Brent
    Brent --> RiskEngine
    RiskEngine --> FastAPI
    Scheduler --> Screening
    FastAPI --> Store
    Store --> Canvas
    Canvas --> Earth
    Canvas --> Instanced
    Canvas --> Trails
    Store --> UI
```

---

## 3. Technology Stack

### Backend
| Package | Version | Purpose |
|---|---|---|
| Python | 3.11+ / 3.12+ | Core runtime environment |
| FastAPI | ^0.110.0 | High-performance asynchronous REST API framework |
| Pydantic v2 | ^2.6.0 | Strict data validation and schema serialization |
| sgp4 | ^2.23 (Vallado) | Industry-standard SGP4 analytical orbital propagator |
| NumPy | ^1.26.0 | Vectorized orbital state mathematics |
| SciPy | ^1.12.0 | Numerical scalar minimization (Brent's method) for TCA refinement |
| Pandas | ^2.2.0 | Tabular dataset manipulation and batch processing |
| httpx | ^0.27.0 | Async HTTP client for resilient TLE ingestion |
| SQLite | 3.x | Persistent local caching for offline resilience |
| Pytest | ^8.0.0 | Automated unit and integration test suite |

### Frontend
| Package | Version | Purpose |
|---|---|---|
| React | ^18.3.1 | Declarative user interface framework |
| TypeScript | ^5.4.2 | Strict compile-time type safety |
| Vite | ^5.1.6 | Next-generation frontend bundler and HMR dev server |
| Three.js | ^0.160.0 | Core WebGL 3D rendering engine |
| @react-three/fiber | ^8.16.0 | React declarative wrapper for Three.js |
| @react-three/drei | ^9.102.0 | Camera controls, stars, and 3D visual helpers |
| Tailwind CSS | ^3.4.1 | Dark aerospace mission-control styling tokens |
| Zustand | ^4.5.2 | Lightweight reactive state management |
| Lucide React | ^0.359.0 | Mission-control telemetry iconography |

---

## 4. Data Sources & Limitations

1. **CelesTrak General Perturbations (GP) Elements**:
   - Publicly accessible, no authentication required.
   - Provides orbital elements for Space Stations, Active Payloads, Starlink constellations, GPS constellations, Weather satellites, and major debris fields (Cosmos 1408, Cosmos 2251, Iridium 33, Fengyun 1C).
2. **Space-Track.org (Optional)**:
   - Feature-flagged behind `SPACETRACK_IDENTITY` and `SPACETRACK_PASSWORD` environment variables.
3. **Frozen Offline Demo Dataset**:
   - Packaged inside `backend/app/data/demo/dataset.json` for deterministic offline demonstrations.

---

## 5. Orbital Mechanics & Coordinate Reference Frames

In accordance with strict aerospace engineering discipline, Aegis Orbital explicitly models three coordinate reference frames:

1. **TEME (True Equator, Mean Equinox)**:
   - Quasi-inertial coordinate system produced directly by the Vallado SGP4 analytical model.
   - All orbital mechanics equations, relative position vectors $\vec{r}_{\text{rel}} = \vec{r}_2 - \vec{r}_1$, and relative velocity vectors $\vec{v}_{\text{rel}} = \vec{v}_2 - \vec{v}_1$ are computed in TEME.
2. **ECEF (Earth-Centered, Earth-Fixed)**:
   - Rotating terrestrial reference frame tied to Earth's geoid.
   - Converted from TEME via Greenwich Mean Sidereal Time (GMST) angle $\theta_{\text{GMST}}$ rotation:
     $$\begin{bmatrix} X_{\text{ECEF}} \\ Y_{\text{ECEF}} \\ Z_{\text{ECEF}} \end{bmatrix} = \begin{bmatrix} \cos\theta & \sin\theta & 0 \\ -\sin\theta & \cos\theta & 0 \\ 0 & 0 & 1 \end{bmatrix} \begin{bmatrix} X_{\text{TEME}} \\ Y_{\text{TEME}} \\ Z_{\text{TEME}} \end{bmatrix}$$
   - Used for 3D Three.js Earth-relative scene rendering.
3. **WGS-84 Geodetic Coordinates**:
   - Closed-form conversion via Bowring's method using equatorial radius $a = 6378.137\text{ km}$ and flattening $f = 1/298.257223563$.
   - Yields geodetic Latitude ($\phi$), Longitude ($\lambda$), and Ellipsoidal Altitude ($h\text{ km}$).

---

## 6. Conjunction Screening & Numerical Refinement

1. **Altitude Envelope Pre-filter**:
   - Rejects non-overlapping orbits (e.g. LEO vs GEO) based on radial apogee and perigee bounds:
     $$[r_{\text{perigee}, 1}, r_{\text{apogee}, 1}] \cap [r_{\text{perigee}, 2}, r_{\text{apogee}, 2}] \neq \emptyset$$
2. **Coarse Temporal Screening**:
   - Candidate pairs are sampled at $\Delta t_{\text{coarse}} = 60\text{ s}$ over a 24-hour prediction window.
   - Approaches entering within the $R_{\text{screening}} = 50\text{ km}$ screening radius are flagged.
3. **Bounded Numerical Brent Minimization**:
   - For flagged candidate pairs, Brent's scalar minimization (`scipy.optimize.minimize_scalar`) finds the exact minimum distance $d_{\text{min}}$ and Time of Closest Approach (TCA) to $\le 1.0\text{ s}$ numerical tolerance:
     $$\text{TCA} = \arg\min_{t \in [t_{\text{coarse}} - \Delta t, \, t_{\text{coarse}} + \Delta t]} \|\vec{r}_2(t) - \vec{r}_1(t)\|$$

---

## 7. Simplified Collision-Risk Model

### Formula
$$S = \min\left(100, \; S_d(d_{\text{min}}, R_{\text{HBR}}, R_{\text{screen}}) + S_v(v_{\text{rel}}) + S_t(\Delta t_{\text{TCA}})\right)$$
Where:
- **$S_d$ (0 to 80 points)**: Exponential proximity score scaled by miss distance relative to Hard-Body Radius (HBR: ~50m for stations, ~5m for payloads, ~0.5m for debris).
- **$S_v$ (0 to 15 points)**: Severity scaling based on encounter velocity magnitude (0 to 15+ km/s).
- **$S_t$ (0 to 5 points)**: Time urgency scaling for encounters within $< 6\text{ hours}$.

### Risk Categorization Bands
- `0 – 20`: **LOW** (Nominal clearance)
- `20 – 50`: **GUARDED** (Routine conjunction advisory)
- `50 – 75`: **ELEVATED** (Approaching alert threshold)
- `75 – 90`: **HIGH** (Close proximity requiring collision avoidance planning)
- `90 – 100`: **CRITICAL** (Direct collision corridor near-miss)

### Mandatory Scientific Disclaimer
> **"This is a Simplified Collision Risk Estimate, not an operational collision probability. True conjunction probability requires orbital covariance/uncertainty information that is not contained in a standard TLE."**

---

## 8. Installation & Setup

### Prerequisites
- Python 3.11+ / 3.12+ (or `uv`)
- Node.js 18+ / 20+ and `npm`

### 1. Clone & Environment Configuration
```bash
git clone <repository_url> spacetech
cd spacetech
cp .env.example .env
```

### 2. Backend Setup
```bash
# Using uv (fastest):
uv venv .venv
# On Windows PowerShell:
.venv\Scripts\activate
# On Linux/macOS:
source .venv/bin/activate

uv pip install -r backend/requirements.txt
```

### 3. Frontend Setup
```bash
cd frontend
npm install
```

---

## 9. Running Locally

### Option A: Standard Dev Mode
**Terminal 1 — Backend:**
```bash
# From repository root
.\.venv\Scripts\python -m uvicorn app.main:app --app-dir backend --host 0.0.0.0 --port 8000 --reload
```
API Documentation will be live at: [http://localhost:8000/docs](http://localhost:8000/docs)

**Terminal 2 — Frontend:**
```bash
cd frontend
npm run dev
```
Mission Control Dashboard will be live at: [http://localhost:5173](http://localhost:5173)

### Option B: Docker Compose
```bash
docker-compose up --build
```

---

## 10. API Reference

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/health` | Service health, active data source (LIVE/CACHED/DEMO), and cache stats |
| `GET` | `/api/objects` | Paginated catalog of tracked satellites with orbital elements |
| `GET` | `/api/objects/{norad_id}` | Detailed orbital state vector, geodetic position, speed, and TLE |
| `GET` | `/api/orbit/{norad_id}` | Discretized 3D trajectory samples for orbital trail visualization |
| `GET` | `/api/conjunctions` | Screened close approaches with risk scores and filter parameters |
| `GET` | `/api/conjunctions/{event_id}` | Detailed conjunction encounter metrics, target summaries, and rationale |
| `GET` | `/api/stats` | Top-level dashboard summary metrics and countdown to next TCA |
| `POST` | `/api/propagate` | On-demand batch propagation of satellite coordinates to target UTC |

---

## 11. Running Tests

Run the complete 27-test automated test suite covering orbital transformations, SGP4 propagation, numerical screening, collision-risk scoring, and FastAPI endpoints:

```bash
# Set PYTHONPATH and execute pytest
$env:PYTHONPATH = "backend"
.\.venv\Scripts\python -m pytest backend/tests -v
```

---

## 12. Limitations (Scientific Honesty)

1. **SGP4 Analytical Fidelity**: SGP4 models mean orbital elements with approximate atmospheric drag and low-order gravitational harmonics ($J_2, J_3, J_4$). Typical LEO error grows by ~1 to 3 km per day.
2. **Absence of Covariance in TLEs**: Standard TLE sets contain no 3D positional covariance error ellipsoids. True operational collision probability ($P_c$) requires covariance matrices typically available only via Conjunction Data Messages (CDMs).
3. **Hard-Body Radius Assumptions**: Satellite physical dimensions are modeled based on general category approximations (~50m stations, ~5m payloads, ~0.5m debris).
4. **Screening Radius Configuration**: The 50 km screening radius is a configurable operational threshold, not a physical boundary.

---

## 13. Future Improvements

1. **Covariance Estimation & 2D Encounter Plane $P_c$**: Foster / Akella-Alfriend collision probability calculations using synthetic or CDM-derived covariance matrices.
2. **High-Precision Orbit Propagation (HPOP)**: Numerical integration (Cowell's method) with high-order Earth geopotential (EGM96/EGM2008), solar radiation pressure (SRP), and atmospheric density models (NRLMSISE-00).
3. **Space-Track API Auto-Sync**: Automated ingestion of verified Space-Track CDMs.
4. **Collision Avoidance Maneuver Planning**: Optimal impulsive $\Delta v$ thrust calculation to maximize miss distance while minimizing propellant expenditure.
