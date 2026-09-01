# NeighbourFit

> Discover Melbourne suburbs that match your lifestyle.

NeighbourFit generates a personalised **Fit Score** (0-100) for Melbourne suburbs, weighted by what the user actually cares about across public transport, food and groceries, healthcare, education, and parks and recreation. Users pick a lifestyle persona, adjust preference weights, search a suburb, and get a data-driven score alongside an ABS Census community profile.

Built March to June 2026 across three shipped iterations by a student team in the **Monash FIT5120 Industry Experience Studio**, a postgraduate industry-experience unit, working under an industry mentor from **APA Group**.

**Winner, Monash PG Industry Experience Expo Award (S1 2026).**

---

## What I built

I owned the **AI layer**, the **REST API backend and scoring engine**, and the **Vue frontend**.

### The AI layer

Three LLM-backed features on **Llama 3.3 70B via the Groq API**, chosen for inference speed fast enough for real-time interaction.

**1. Voice-to-Preferences.** Users describe their lifestyle in natural language instead of moving sliders - *"a quiet place with good parks for my dog and decent coffee nearby"*. Browser `SpeechRecognition` captures the transcript in `PersonaliseSection.vue`, the backend prompts Llama 3.3 to act as a structured extraction engine, and the response comes back as **enforced JSON** - persona, five category weights, and a `reasoning` string. The frontend selects the persona card, sets the sliders, and shows the reasoning back to the user as an explanation for what just changed.

**2. Persona-aware suburb summaries.** On each search, `AiSummaryCard.vue` fires an async request that does not block the deterministic scores from rendering. The backend packages the persona, the five category scores and the ABS Census demographics into a prompt constrained to exactly three sentences: overall impression, one specific data-backed strength or weakness, then a practical verdict.

**3. Hybrid recommendation re-ranking.** The interesting one architecturally. The backend first computes Fit Scores for every suburb using the persona's weights and takes the top 15 - **deterministic, auditable, no model involved**. Only then does Llama 3.3 re-rank those 15 down to 3, judging soft factors the maths cannot see (noise, lifestyle, demographic fit), returning enforced JSON with a one-sentence justification per suburb.

**Design decisions worth naming:**

- **Structured output over parsing.** Every LLM call enforces a JSON schema, so the frontend never regex-scrapes prose. This is the difference between a demo and something you can build a UI on.
- **Explainability by default.** Both the voice engine and the recommender return their reasoning, and it is surfaced to the user rather than hidden. People do not trust a slider that moves on its own.
- **Graceful degradation everywhere.** If Groq times out, errors, or the key is missing: summaries fail silently and hide the card, voice shows a friendly error and falls back to manual sliders, and recommendations skip the re-ranking and return the top 3 by maths alone. **The deterministic core works 100% of the time regardless of model availability.** The AI is a layer on top of a system that stands up without it, not a dependency.

### Backend and data

- **The REST API backend**, with `UNION ALL` queries spanning multiple spatial data tables to combine point-of-interest data with PTV stop data in a single pass
- **The weighted scoring algorithm** that ranks suburbs against user preferences, wired through `backend/app.py`, `backend/scoring.py` and the `analyze_suburb` / `search_suburb` Lambda handlers
- **Diagnosed and fixed a production serialisation failure** where PostgreSQL `RealDictRow` and `Decimal` types would not serialise inside AWS Lambda. It worked locally under Flask and failed only once deployed, which is the most annoying class of bug there is and the reason the local server was built to mirror the Lambda logic in the first place.

### Frontend

- **Frontend views** - the Explore page (persona picker, search, live score, demographics), suburb Compare with radar chart and per-category breakdown, the Rankings leaderboard, the Landing page, and the Methodology explainer
- **Components** - the animated Fit Score ring, persona-aware preference sliders, the category breakdown chart, the Leaflet map with amenity markers, and suburb search with autocomplete
- **`services/api.js`** - the Axios client layer between the Vue app and the API, environment-switched between the local Flask server and the deployed API Gateway endpoint

Built in a team, as the unit requires. The above is the work I owned.

---

## What it does

| Feature | Description |
|---|---|
| **Persona selector** | Six lifestyle profiles (Student, New to Melbourne, Young Professional, Family, Retiree, Outdoor Enthusiast) that pre-set the scoring weights |
| **Personalised Fit Score** | 0-100, weighted across five categories, recalculated live as the user moves the sliders |
| **ABS Census demographics** | A separate community profile score that adapts to the chosen persona |
| **Suburb comparison** | Two suburbs side by side, with animated score rings, a radar chart, and an auto-generated verdict |
| **Rankings** | Leaderboard of Melbourne suburbs by equal-weight Fit Score |
| **Voice to preferences** | Users describe their lifestyle in natural language and the preference weights are set automatically |
| **AI suburb summary** | A persona-aware plain-English explanation of why a suburb fits, generated with Llama 3.3 |
| **Top 3 recommendations** | Deterministic scoring shortlists 15 suburbs; the LLM re-ranks to 3 on soft factors and explains each pick |

---

## Architecture

```
Vue 3 (Vite) ── Axios ──► API Gateway ──► AWS Lambda ──► PostgreSQL (RDS)
     │                                          │
  Leaflet map                              Groq / Llama 3.3
     │                                    (summaries, voice parsing)
  AWS Amplify
```

| Layer | Technology |
|---|---|
| Frontend | Vue 3, Vite, Vue Router, Axios, Tailwind |
| Maps | Leaflet.js with OpenStreetMap tiles and marker clustering |
| Backend (local dev) | Python Flask |
| Backend (production) | AWS Lambda behind API Gateway |
| LLM | Groq API, Llama 3.3 70B |
| Database | PostgreSQL on AWS RDS |
| Hosting | AWS Amplify, branch-per-environment |

The local Flask server deliberately mirrors the Lambda logic, so scoring changes could be developed and tested locally without deploying.

---

## How the scoring works

**Fit Score** is a weighted average of five category scores, each derived from point-of-interest density within the suburb:

```
Fit Score = Σ (category_score × user_weight) / Σ user_weights
```

Users set each weight from 0 to 5, and the score recalculates in real time.

**Demographics Score** is computed separately from ABS Census data and is deliberately not affected by the preference weights:

```
Demographics Score = (young_adult_ratio + diversity_score + student_population_ratio) / 3 × 100
diversity_score    = (overseas_born_ratio + non_english_home_ratio) / 2
```

Keeping these two scores separate was a deliberate design decision: the Fit Score answers "does this suburb serve my daily needs", and the demographics profile answers "will I find people like me here". Collapsing them into one number would have hidden the distinction that the whole product exists to surface.

---

## Data sources

| Category | Source |
|---|---|
| Transport stops | Public Transport Victoria GTFS feed |
| Food and grocery venues | OpenStreetMap (Overpass API) |
| Healthcare facilities | OpenStreetMap and AIHW datasets |
| Education | Australian Government ACARA |
| Parks and open spaces | OpenStreetMap |
| Demographics | ABS Census 2021 |

---

## What I would do differently

- **The scoring is density-based, not accessibility-based.** Counting amenities inside a suburb boundary rewards large suburbs and ignores whether anything is actually walkable. Isochrone-based scoring, or at least distance decay from the population centroid, would measure what users think they are being told.
- **Secrets management was inconsistent.** The Lambda functions read credentials from the environment; one local helper script hardcoded them. It should have been enforced from the first commit rather than fixed later.
- **The LLM summaries had no caching layer.** Every summary was a live Groq call, which is slow and needlessly expensive for a result that changes only when the underlying scores change. Keying a cache on suburb plus persona plus score-vector would have cut nearly all of it.
- **I never evaluated the LLM outputs systematically.** The prompts were tuned by reading results and adjusting, which is how most people do it and is not good enough. A small labelled set and a rubric - is the summary factually consistent with the scores it was given, does the re-ranking beat the raw maths - would have told us whether the AI layer was actually adding value or just adding latency.
- **The suburb boundary data drove a lot of edge cases** we handled reactively rather than modelling up front.

---

## Status

Semester 1 2026, Monash FIT5120 Industry Experience Studio. This repository is documentation.

The application ran on AWS Amplify with an API Gateway, Lambda and RDS backend for the duration of the unit, covering 290+ Melbourne suburbs. The Amplify frontend is still served, but the backend was decommissioned after the semester, so searches return no data and the deployed URL is deliberately not linked here.

The application code lives in the team's repository and is not mine alone to publish.
