# AI-Powered Multimodal Construction Intelligence Platform

React.js frontend + FastAPI backend + multimodal LLM. Seven modules share one pipeline:

```
React form (text + optional image)
   -> POST /api/<module>  (multipart: JSON `payload` + `image`)
   -> Pydantic request validation
   -> optional live-data enrichment (e.g. weather forecast)
   -> multimodal LLM call, forced to return JSON
   -> Pydantic response validation (one automatic retry on mismatch)
   -> typed JSON back to React
```

## Modules

| Route | Input | Output contract |
|---|---|---|
| `POST /api/safety` | site photo (required), notes | `SafetyReport`: score, violations with severity and corrective action |
| `POST /api/cracks` | crack photo (required), element, notes | `CrackReport`: patterns, causes, structural risk, engineer-inspection flag |
| `POST /api/valuation` | property details, photo optional | `ValuationReport`: estimate, range, price per sq ft, assumptions |
| `POST /api/planning` | project brief | `ProjectPlan`: phases, resources, cost split, risks |
| `POST /api/assistant` | question, context, photo optional | `AssistantReply`: answer, action items, follow-ups |
| `POST /api/weather` | latitude, longitude, activities | `WeatherAdvisory`: per-activity go / caution / postpone (live Open-Meteo data) |
| `POST /api/knowledge` | question, depth | `KnowledgeAnswer`: answer, key points, standards, caveats |

`GET /api/contracts/<module>` returns the JSON Schemas for each module. `GET /api/health` lists modules.

## Run it

```bash
# backend (Python 3.10+)
cd backend
python -m venv .venv && source .venv/bin/activate     # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env                                   # then add ANTHROPIC_API_KEY
uvicorn app.main:app --reload --port 8000
python -m pytest                                       # contract tests, no API key needed

# frontend (Node 18+), in a second terminal
cd frontend
npm install
npm run dev                                            # http://localhost:5173
```

## Where to change things

- **Add or edit a module:** `backend/app/schemas.py` (contracts) and `backend/app/modules.py` (prompt + image policy), then add an entry to `MODULES` in `frontend/src/App.jsx`. The panel and result view are generic.
- **Swap the model:** set `LLM_MODEL` in `backend/.env`. Only `backend/app/llm.py` talks to the provider.
- **Production hardening:** auth, rate limiting, image storage, audit log of AI outputs, and a human-review step for safety and structural findings.

## Limits to state in your report

Outputs are advisory. Crack, safety and valuation results come from a single photo or form and are not a substitute for a structural engineer, site safety officer or certified valuer.

## Host on Streamlit

`backend/streamlit_app.py` is a Streamlit version of the whole UI. It imports the same prompts and
JSON contracts as the FastAPI app, so it runs on its own with no API server and no React build.

```bash
cd backend
pip install -r requirements.txt
cp .streamlit/secrets.toml.example .streamlit/secrets.toml   # add your key
streamlit run streamlit_app.py                                # http://localhost:8501
```

**Deploy on Streamlit Community Cloud**
1. Push this project to a GitHub repo (`.env` and `secrets.toml` are git-ignored).
2. On share.streamlit.io choose "Create app", pick the repo, and set the main file path to `backend/streamlit_app.py`.
3. Open Advanced settings, choose Python 3.11 or newer, and paste `ANTHROPIC_API_KEY = "..."` into Secrets.
4. Deploy. Every push to the branch redeploys.

Community Cloud runs Streamlit apps only. To keep the React + FastAPI version, host the backend on
Render or Railway (start command: `uvicorn app.main:app --host 0.0.0.0 --port $PORT`) and the frontend
on Vercel or Netlify, then point the frontend's `/api` calls at the backend URL.
