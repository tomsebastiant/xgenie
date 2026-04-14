# Build Plan — xgenie

> Personal reference for building this project 1 hour/day with daily GitHub commits.
> Goal: a deployed, interview-ready portfolio project that mirrors production AI agent patterns from Intuit work.

---

## Why this project

Every component maps directly to production experience:

| This project | Production equivalent (Intuit) |
|---|---|
| LangGraph supervisor + parallel analyst nodes | 7-node parallel RCA graph + supervisor-orchestrated agent |
| AnalysisCache with `asyncio.Event` | SkillCache across concurrent agent invocations |
| Java data service as tool provider | Go MCP server with 19 diagnostic tools |
| Langfuse tracing on all LLM calls | Langfuse across all Intuit agent workflows |
| Multi-LLM config (cheap nodes + quality synthesis) | 12-LLM config pattern |
| Prompt files per node (`.md`) | 40+ YAML-managed prompt store |

**Interview framing:** *"I built a football analytics agent using the same LangGraph supervisor pattern I use at Intuit for incident RCA — parallel analysis nodes, shared cache to prevent duplicate API calls, multi-LLM routing between investigator and synthesis roles. The domain is different, the architecture is identical."*

---

## Free tools stack

| Tool | Purpose | Cost |
|---|---|---|
| VS Code | IDE for Java + Python | Free |
| start.spring.io | Spring Boot project generator | Free |
| Spring Boot | **4.0.5** (latest stable — no SNAPSHOT/M4) | — |
| PostgreSQL 16 | Local dev DB via Docker | Free |
| Google Gemini API | LLM for all agent nodes (15 req/min) | Free at aistudio.google.com |
| Langfuse Cloud | LLM call tracing + observability | Free (50k obs/month) |
| Render free tier | Deploy Java service + Python agent | Free (spins down after 15min idle) |
| Supabase | PostgreSQL for prod (500MB) | Free |
| Vercel | Deploy React frontend | Free |
| Claude.ai | Code review, debugging, prompt iteration | Free tier |

**Local PostgreSQL via Docker (run once, leave it running):**
```powershell
docker run --name xgenie-db -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=xgenie -p 5432:5432 -d postgres:16
```

**When actively interviewing:** upgrade Java + Python services to Railway ($5/month) for always-on, no cold starts.

---

## Commit message convention

Use conventional commits on every commit — interviewers scroll the history.

```
feat: add Match and Competition JPA entities
fix: planner routing logic for player question type
docs: agent README with architecture and sample output
test: loader service and match controller unit tests
refactor: extract event aggregation into PlayerStatsService
```

---

## Phase 1 — Java Spring Boot data service

**Duration:** Weeks 1–3 (~15 hours)  
**Goal:** Working REST API over real StatsBomb data, with Swagger docs, tests, and a Dockerfile.  
**Key interview talking point:** Spring Boot 4.x, JPA, PostgreSQL (local Docker for dev, Supabase for prod), proper layering.

---

### Week 1 — Project skeleton + data loading

**Day 1 (1hr)**
- Create GitHub repo named `xgenie`
- Generate Spring Boot project at [start.spring.io](https://start.spring.io):
  - Spring Boot: **4.0.5**
  - Group: `com.footballintel`, Artifact: `data-service`, Java: **21**
  - Dependencies: Spring Web, Spring Data JPA, **PostgreSQL Driver**, Lombok
- Move generated folder into repo as `data-service/`
- Clone StatsBomb open-data as a git submodule: `git submodule add https://github.com/statsbomb/open-data statsbomb-data`

```
git commit: feat: init spring boot project + statsbomb submodule
```

**Day 2 (1hr)**
- Create JPA entity classes: `Match`, `Competition`
- Fields needed: id, name, date, home team, away team, competition id, season id
- Add `MatchRepository`, `CompetitionRepository` (Spring Data interfaces — no implementation needed)

```
git commit: feat: add Match and Competition JPA entities
```

**Day 3 (1hr)**
- Write `StatsBombLoaderService` with `@PostConstruct`
- Reads `competitions.json` and `matches/{competition_id}/{season_id}.json` at startup
- Parse with Jackson, insert into PostgreSQL
- Log counts: "Loaded 380 matches across 1 competition"
- Don't try to load everything yet — La Liga only

```
git commit: feat: StatsBombLoaderService parses competitions and matches
```

**Day 4 (1hr)**
- Add `CompetitionController`: `GET /api/competitions`
- Add `MatchController`: `GET /api/competitions/{id}/matches`
- Test with curl or Postman
- Fix Jackson parsing bugs (StatsBomb JSON has nested objects — use `@JsonIgnoreProperties(ignoreUnknown = true)`)

```
git commit: feat: competitions and matches REST endpoints
```

**Day 5 (1hr)**
- Add `application-dev.yml` — PostgreSQL config pointing to local Docker instance:
  ```yaml
  spring:
    datasource:
      url: jdbc:postgresql://localhost:5432/xgenie
      username: postgres
      password: postgres
    jpa:
      hibernate:
        ddl-auto: update
  statsbomb:
    data-path: ../statsbomb-data/data
  ```
- Add `application-prod.yml` — PostgreSQL config via env vars (Supabase in prod):
  ```yaml
  spring:
    datasource:
      url: ${SPRING_DATASOURCE_URL}
      username: ${SPRING_DATASOURCE_USERNAME}
      password: ${SPRING_DATASOURCE_PASSWORD}
    jpa:
      hibernate:
        ddl-auto: validate
  ```
- Write `data-service/README.md` — what it does, how to run, example curl commands
- This documentation is what interviewers read first

> **Note on springdoc-openapi (Week 3):** Spring Boot 4.x requires this specific version:
> ```xml
> <dependency>
>   <groupId>org.springdoc</groupId>
>   <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
>   <version>2.8.8</version>
> </dependency>
> ```

```
git commit: feat: multi-profile config + data-service README
```

---

### Week 2 — Event data + player stats

**Day 1 (1hr)**
- Add `Event` JPA entity
- Fields: id, matchId, type (pass/shot/pressure/carry), minute, second, x, y, playerId, playerName, teamId, xg (nullable)
- Load **only La Liga 2015/16 events** — full dataset is large, start scoped

```
git commit: feat: Event entity + load La Liga 2015/16 events
```

**Day 2 (1hr)**
- Add `EventController`:
  - `GET /api/matches/{matchId}/events` with optional `?type=` filter
  - `GET /api/matches/{matchId}/events/summary` — returns shot count, xG total, pass count, pressure count per team

```
git commit: feat: event endpoints with type filter and summary
```

**Day 3 (1hr)**
- Add `PlayerStatsService` — aggregates Event table into per-player stats
- Stats: goals (shots with outcome=goal), total shots, xG sum, key passes
- Use JPQL queries or Java stream aggregation — no new entity needed
- Return as a DTO, not a raw entity

```
git commit: feat: PlayerStatsService aggregates goals, shots, xG per player
```

**Day 4 (1hr)**
- Add `PlayerController`: `GET /api/players/{playerId}/stats?competitionId=&seasonId=`
- Test that Messi's stats return correctly for La Liga 2015/16
- Fix aggregation bugs — there will be some

```
git commit: feat: player stats endpoint with competition and season filter
```

**Day 5 (1hr)**
- Write JUnit tests:
  - `StatsBombLoaderServiceTest` — verify match count after load
  - `MatchControllerTest` — `@SpringBootTest` + MockMvc, verify 200 response and JSON structure
- Add `src/test/resources/application-test.yml` pointing to the local Docker PostgreSQL
- 2–3 tests is enough — the point is showing test discipline

```
git commit: test: loader service and match controller integration tests
```

---

### Week 3 — Team metrics + API polish

**Day 1 (1hr)**
- Add `GET /api/teams/{teamId}/tactical-profile`
- Compute from existing Event data:
  - Formation: from lineup events
  - Pressing intensity: pressure events per 90 minutes
  - Possession %: pass events by team / total pass events
- Return as DTO

```
git commit: feat: team tactical profile endpoint
```

**Day 2 (1hr)**
- Add `GET /api/matches/{matchId}/timeline`
- Returns key events in order: goals (with scorer), red cards, substitutions (with minute)
- Sorted by minute — clean, simple, very useful for the agent's report writer

```
git commit: feat: match timeline endpoint (goals, cards, substitutions)
```

**Day 3 (1hr)**
- Add `@ControllerAdvice` for global error handling — proper 404s for missing matches/players
- Add springdoc-openapi dependency to pom.xml
- Swagger UI auto-available at `/swagger-ui.html` — no extra code
- Test the Swagger page renders all endpoints correctly

```
git commit: feat: global error handling + OpenAPI docs via springdoc
```

**Day 4 (1hr)**
- Write `data-service/Dockerfile`
- Write top-level `docker-compose.yml` that starts both the Java service **and a PostgreSQL container**:
  ```yaml
  services:
    db:
      image: postgres:16
      environment:
        POSTGRES_DB: xgenie
        POSTGRES_PASSWORD: postgres
      ports:
        - "5432:5432"
    data-service:
      build: ./data-service
      ports:
        - "8080:8080"
      environment:
        SPRING_PROFILES_ACTIVE: dev
      depends_on:
        - db
  ```
- Test `docker-compose up --build` brings up both services
- Fix any Docker build issues (common: StatsBomb data path inside container)

```
git commit: feat: Dockerfile + docker-compose for data service
```

**Day 5 (1hr)**
- Update root `README.md`:
  - Architecture ASCII diagram (copy from spec)
  - Java service setup instructions
  - Example curl commands for every endpoint
  - Make it scannable in 90 seconds — headers, code blocks, no walls of text

```
git commit: docs: root README with architecture diagram and API examples
```

**Phase 1 checkpoint:** You now have a deployable Spring Boot backend over real football data. The Swagger page at `/swagger-ui.html` is a standalone interview demo — show it before the agent is even built.

---

## Phase 2 — LangGraph agent MVP

**Duration:** Weeks 4–6 (~15 hours)  
**Goal:** Full agent with all three analyst nodes, AnalysisCache, Langfuse tracing, and a FastAPI endpoint.  
**Key interview talking points:** LangGraph supervisor pattern, `asyncio.Event` deduplication (same as Intuit SkillCache), Langfuse trace showing parallel execution.

---

### Week 4 — Agent skeleton + planner

**Day 1 (1hr)**
- Set up Python project in `agent/` folder
- `pyproject.toml` with Poetry, or `requirements.txt` if simpler
- Dependencies: `langgraph`, `langchain`, `langchain-google-genai`, `fastapi`, `uvicorn`, `httpx`, `langfuse`
- Create folder structure: `app/graph/nodes/`, `app/tools/`, `app/prompts/`

```
git commit: feat: python agent project setup with pyproject.toml
```

**Day 2 (1hr)**
- Define `FootballAgentState` TypedDict in `state.py` (exact structure from README)
- Write `DataClient` in `tools/data_client.py` — async httpx client
- Methods: `get_matches()`, `get_events()`, `get_event_summary()`, `get_player_stats()`, `get_tactical_profile()`, `get_timeline()`
- All methods take match/player/team IDs as params

```
git commit: feat: agent state definition and Java API HTTP client
```

**Day 3 (1hr)**
- Write `agent/app/prompts/planner.md` — instructions for deciding which nodes to invoke
- Write `planner_node` in `nodes/planner.py`
- Calls Gemini, parses response into `list[str]` of analysis tasks
- Tasks should be: `"event_analysis"`, `"player_analysis"`, `"tactical_analysis"`

```
git commit: feat: planner node with Gemini and prompt file
```

**Day 4 (1hr)**
- Write `graph.py` — `StateGraph` with planner node only
- Add conditional edge: `planner → route_to_analysts()`
- Add report_writer stub that returns analysis tasks as a string
- Run the graph with a test question — get it to execute without errors

```
git commit: feat: LangGraph graph definition with planner wired to stub writer
```

**Day 5 (1hr) — Langfuse from day one**
- Sign up at [cloud.langfuse.com](https://cloud.langfuse.com) — free
- Add `CallbackHandler` setup in `config.py`
- Pass handler to every graph invocation via `config={"callbacks": [langfuse_handler]}`
- Verify a test run appears in the Langfuse dashboard
- **Do not skip this step** — the Langfuse trace screenshot is a README artefact

```
git commit: feat: Langfuse tracing integrated on all LLM calls
```

---

### Week 5 — Event analyst + report writer

**Day 1 (1hr)**
- Write `event_analyst_node` in `nodes/event_analyst.py`
- Fetches events from Java service via DataClient
- Passes event summary to Gemini with `event_analyst.md` prompt
- Writes findings string to `state.event_analysis`
- Write `agent/app/prompts/event_analyst.md`

```
git commit: feat: event analyst node fetches Java data and analyses with Gemini
```

**Day 2 (1hr) — The most important implementation**
- Write `AnalysisCache` in `graph/cache.py`
- Pattern: `asyncio.Event` prevents duplicate in-flight API calls
- If node A is fetching match events and node B needs the same data, B waits on the Event
- Add detailed comments explaining the pattern — you will be asked about this in interviews
- Wire the cache into `FootballAgentState`

```python
# The key pattern — explain this in interviews:
# Same as SkillCache used in production Intuit agent
async def get_or_fetch(self, key: str, fetch_fn):
    if key in self._cache:
        return self._cache[key]
    if key in self._events:
        await self._events[key].wait()  # wait, don't fire duplicate request
        return self._cache.get(key)
    event = asyncio.Event()
    self._events[key] = event
    try:
        result = await fetch_fn()
        self._cache[key] = result
        return result
    finally:
        event.set()  # unblock all waiters
```

```
git commit: feat: AnalysisCache with asyncio.Event deduplication pattern
```

**Day 3 (1hr)**
- Write `report_writer_node` in `nodes/report_writer.py`
- Takes `event_analysis`, `player_analysis`, `tactical_analysis` from state
- Synthesises into a markdown report using Gemini (use a larger model here if you have quota)
- Write `agent/app/prompts/report_writer.md` — this prompt drives output quality, iterate on it
- The report should have sections: Summary, Key Events, Analysis, Tactical Notes

```
git commit: feat: report writer synthesises all node findings into markdown report
```

**Day 4 (1hr)**
- Wire the full graph: `planner → event_analyst → report_writer`
- Test end-to-end: `"How did Barcelona perform in their first La Liga 2015/16 match?"`
- Fix chain errors — there will be state key mismatches or async issues
- Verify the report output is readable and analytical, not just raw data

```
git commit: feat: full graph runs end-to-end for match analysis question
```

**Day 5 (1hr)**
- Write `server.py` — FastAPI app with `POST /query`
- Request body: `{ "question": str, "competition_id": int | null }`
- Response: `{ "report": str, "analysis_tasks": list[str] }`
- Test with curl: `curl -X POST localhost:8000/query -d '{"question": "..."}'`

```
git commit: feat: FastAPI /query endpoint wraps agent graph
```

---

### Week 6 — Player + tactical analysts

**Day 1 (1hr)**
- Add `player_analyst_node` — fetches player stats, analyses performance
- Wire into graph as a parallel node alongside `event_analyst`
- Update `planner.md` prompt to route player questions here
- Planner should NOT invoke player_analyst for pure match questions

```
git commit: feat: player analyst node with parallel graph wiring
```

**Day 2 (1hr)**
- Add `tactical_analyst_node` — uses team tactical profile endpoint
- Implement `asyncio.gather()` for true parallel execution across all three nodes
- The gather pattern: all three nodes fire simultaneously, report_writer waits for all

```python
# Parallel execution pattern in graph.py
results = await asyncio.gather(
    event_analyst_node(state),
    player_analyst_node(state),
    tactical_analyst_node(state),
)
```

```
git commit: feat: tactical analyst + asyncio.gather parallel execution
```

**Day 3 (1hr)**
- Test 5 different question types from the README examples
- Verify planner routes correctly — a player question should not invoke tactical analyst
- Fix routing logic in the conditional edge function
- This is the hardest debugging session in the whole project

```
git commit: fix: planner conditional routing for all 5 question types
```

**Day 4 (1hr)**
- Write `agent/Dockerfile`
- Update `docker-compose.yml` to start both Java service and Python agent
- Test `docker-compose up` brings up the full stack
- Test a curl to the agent on localhost returns a real report

```
git commit: feat: agent Dockerfile + full stack docker-compose
```

**Day 5 (1hr)**
- Write `agent/README.md`:
  - Graph architecture diagram (ASCII is fine)
  - Node responsibilities
  - AnalysisCache explanation
  - Sample question + paste the actual report output it generated
- The sample output is the most important part — shows the project actually works

```
git commit: docs: agent README with node architecture and real sample output
```

**Phase 2 checkpoint:** You have a working AI agent. Take two screenshots for the README: (1) the Langfuse trace showing all nodes firing in parallel, (2) a sample report output. These are the artefacts that close interviews.

---

## Phase 3 — Deploy + minimal frontend

**Duration:** Weeks 7–8 (~8 hours)  
**Goal:** Shareable URL on your resume, demo-ready React frontend, demo video recorded.

---

### Week 7 — Deploy to Render free tier

**Day 1 (1hr) — Java service deployment**
- Push repo to GitHub if not already
- Go to [render.com](https://render.com) → New Web Service → connect repo
- Root directory: `data-service`
- Build command: `./mvnw clean package -DskipTests`
- Start command: `java -jar target/data-service-*.jar`
- Env var: `SPRING_PROFILES_ACTIVE=prod`
- First deploy will fail — read the logs, fix it

```
git commit: feat: render.yaml deployment config for data service
```

**Day 2 (1hr) — PostgreSQL on Supabase**
- Create free project at [supabase.com](https://supabase.com)
- Get the JDBC connection string from Settings → Database
- Add to Render env vars: `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, `SPRING_DATASOURCE_PASSWORD`
- Verify StatsBombLoaderService runs at startup on deployed service (check Render logs)

```
git commit: feat: supabase postgresql wired for production deployment
```

**Day 3 (1hr) — Python agent deployment**
- Render → New Web Service → same repo
- Root directory: `agent`
- Build command: `pip install poetry && poetry install`
- Start command: `uvicorn app.server:app --host 0.0.0.0 --port $PORT`
- Env vars: `GEMINI_API_KEY`, `DATA_SERVICE_URL` (Render URL of Java service), Langfuse keys
- Test the live `/query` endpoint

```
git commit: feat: render deployment config for python agent
```

**Day 4 (1hr) — React frontend**
- `cd frontend && npm create vite@latest . -- --template react`
- Build `QueryInput.jsx` — text input + submit button
- Build `ReportDisplay.jsx` — renders markdown (`npm install react-markdown`)
- Wire to agent API: `VITE_API_URL` env var pointing to Render agent URL

```
git commit: feat: react frontend with query input and markdown report display
```

**Day 5 (1hr) — Vercel deploy**
- Push frontend to GitHub
- [vercel.com](https://vercel.com) → New Project → connect repo
- Root directory: `frontend`, Framework: Vite
- Add env var: `VITE_API_URL=https://your-agent.onrender.com`
- Get your `*.vercel.app` URL — **this goes on your resume**

```
git commit: feat: vercel deployment config for react frontend
```

---

### Week 8 — Demo prep

**Day 1 (1hr) — Demo video**
- Record 90-second video (Loom free, or OBS)
- Script: type a question → show Langfuse trace with nodes firing → show final report
- Upload to YouTube (unlisted) or Loom
- Add the link to root README under a "Demo" heading

```
git commit: docs: add demo video link and live URL to README
```

**Day 2 (1hr) — Architecture write-up**
- Write the "Why this architecture" section in README in your own words
- Explain: why Java owns data and Python owns reasoning
- Explain: why the AnalysisCache pattern prevents redundant API calls
- Add the ASCII architecture diagram from the spec
- This section is what senior engineers read when evaluating the repo

```
git commit: docs: architecture rationale and design decisions section
```

**Day 3 (1hr) — Technical write-up**
- Write `TECHNICAL_NOTES.md` or a dev.to post on the AnalysisCache pattern
- 300–400 words: what problem it solves, how asyncio.Event works, how it mirrors the Intuit SkillCache
- Link from README
- Writing about code signals senior-level thinking to interviewers

```
git commit: docs: AnalysisCache technical write-up
```

**Day 4 (1hr) — Resume update**
- Update resume with the project entry:

> *xgenie — LangGraph agent with parallel analyst nodes, Java Spring Boot 4.x data service over StatsBomb open data, AnalysisCache (asyncio.Event deduplication), Langfuse tracing. Same supervisor/cache patterns used in production at Intuit. [live link] [github link]*

```
git commit: docs: add live URL and project completion to README
```

**Phase 3 checkpoint:** You have a deployed project with a shareable URL, a demo video, and a resume line. This is the submission state.

---

## Phase 4 — Polish (add only when you can explain it confidently)

Do these in order. Stop the week before any interview — half-finished features hurt more than they help.

**LLM-as-Judge eval**
- Score 10 sampled reports against a rubric: completeness (0–10), accuracy (0–10), reasoning quality (0–10)
- Log scores to Langfuse as metadata
- Shows you think about quality measurement, not just shipping
```
git commit: feat: LLM-as-Judge eval scores report quality against rubric
```

**Railway upgrade (when actively interviewing)**
- Migrate Java + Python services to Railway $5/month
- Always-on, no cold starts — responsive for live demos
- Built-in PostgreSQL and Redis included in plan
```
git commit: feat: railway deployment config replaces render
```

**Redis cross-session cache**
- Extends AnalysisCache pattern to persist between agent runs
- Repeated questions (same match, different phrasing) hit Redis instead of Java API
- Good interview conversation: "I extended the in-run cache to persist across requests"
```
git commit: feat: redis cross-session cache for java API responses
```

**Additional competitions**
- `git submodule update` to pull latest StatsBomb data
- StatsBombLoaderService picks up new competitions automatically at next restart
- No code changes needed — just restart the service
```
git commit: data: add UCL and World Cup 2022 to loaded competitions
```

---

## Daily habit checklist

Before closing the laptop each session:

- [ ] Code compiles / tests pass
- [ ] At least one commit pushed with a proper `feat:/fix:/docs:/test:` prefix  
- [ ] No TODO comments left in new code (file an issue instead)
- [ ] If something took longer than expected, note why below

---

## Blockers and notes

> Add notes here as you build. What was harder than expected? What shortcuts did you take? This becomes context for future debugging sessions and for explaining the project in interviews.

---

*Last updated: April 2026 — Spring Boot 4.0.5, PostgreSQL throughout (Docker locally, Supabase in prod). Keep this file updated as the build evolves.*
