# CABANA Finance — Architecture

ESG Investment Dashboard. Diagrams below are traced from the source in `Project Repo/`, not from the
layer summary in the README.

**Layers as declared:**

| Layer | Stack | Location |
|---|---|---|
| Application | Next.js 15.2.2, React 19 | `Project Repo/frontend/cabana` |
| Server / API | RESTful, Python 3.12, Flask | `Project Repo/flask-api` |
| Persistence | AWS RDS (PostgreSQL) | accessed via `db.py` |
| Deployment | Docker Hub, AWS EC2, GitHub Actions | `docker-compose.yml`, `.github/workflows/` |

The single most important architectural fact, which shapes every diagram: **there are no Next.js API
routes and no server-side data fetching.** Every page carries `'use client'`, and the only outbound
call is at `frontend/cabana/app/(pages)/get-demo/page.jsx:76` — the *browser* calls Flask directly.

---

## 1. Layered Architecture

```mermaid
flowchart TB
    subgraph APP["1 · APPLICATION LAYER — Next.js 15.2.2 · React 19 · frontend/cabana"]
        direction TB
        L["app/layout.jsx<br/>RootLayout · Geist fonts · title CABANA Finance"]
        subgraph PAGES["App Router pages — all 'use client'"]
            H["app/page.jsx<br/>Home"]
            GD["app/(pages)/get-demo/page.jsx<br/>ONLY API caller"]
            AB["app/(pages)/about-us/page.jsx"]
            TR["app/(pages)/trending/page.js<br/>EMPTY — 0 bytes"]
        end
        subgraph CMP["components/"]
            HD["Header.jsx"]
            PN["Panel.jsx — text prop"]
            SB["Searchbar.jsx<br/>local useState only<br/>NOT wired to API"]
            FT["Footer.jsx"]
            FC["FlipCard.jsx"]
            DD["DataDisplay.jsx<br/>MUI table renderer"]
        end
        SD["public/data.js<br/>static esgDesc1 / esgDesc2"]
    end

    subgraph API["2 · SERVER / API LAYER — Flask · flask-cors · Python 3.12"]
        direction TB
        IDX["index.py — routes<br/>/ · /dummy · /hello<br/>/get · /slowget<br/>/getIndustry · /getCompanies"]
        EF["esg_functions.py<br/>valid_category · valid_columns<br/>create_sql_query · create_column_array<br/>create_adage_data_model"]
        DB["db.py<br/>run_sql → list of dicts<br/>run_sql_raw → list of tuples"]
    end

    subgraph PER["3 · PERSISTENCE LAYER — AWS RDS PostgreSQL"]
        direction TB
        T1[("7 category tables<br/>environmental_risk · environmental_opportunity<br/>social_* · governance_* · esg")]
        T2[("industry<br/>company · industry")]
    end

    ENV[".env → DB_HOST · DB_PORT<br/>DB_NAME · DB_USER · DB_PASSWORD"]

    L --> H & GD & AB & TR
    H --> HD & PN & SB & FT
    GD --> DD
    AB --> FC
    AB -.reads.-> SD

    GD ==>|"fetch GET http://127.0.0.1:5000/get<br/>browser to Flask, cross-origin"| IDX
    IDX --> EF
    IDX --> DB
    EF --> DB
    DB ==>|"psycopg2 · new connection per call"| T1
    DB ==> T2
    ENV -.->|"load_dotenv on every query"| DB

    style GD stroke:#d97706,stroke-width:3px
    style TR stroke:#dc2626,stroke-width:2px,stroke-dasharray: 5 5
    style SB stroke:#dc2626,stroke-width:2px,stroke-dasharray: 5 5
```

Next.js here is a client-rendered shell, not a server. `'use client'` on every page plus a
browser-side `fetch` to a different origin means none of the App Router's server-fetch benefits
apply, and `CORS(app)` at `index.py:9` becomes load-bearing rather than incidental.

The dashed red nodes are real gaps: `Searchbar.jsx` holds six pieces of filter state
(`searchQuery`, `startDate`, `endDate`, `country`, `priceRange`, `category`) that never reach the
API, and `trending/page.js` is a zero-byte file, so `/trending` routes but has no default export.

---

## 2. Module Dependency Graph

```mermaid
flowchart LR
    subgraph FE["frontend/cabana"]
        GDP["get-demo/page.jsx"]
        DDC["DataDisplay.jsx"]
        MUI["@mui/material · @mui/joy<br/>@emotion · tailwind"]
        GDP -->|"import"| DDC
        GDP --> MUI
        DDC --> MUI
    end

    subgraph BE["flask-api"]
        I["index.py"]
        E["esg_functions.py"]
        D["db.py"]
        T["test.py<br/>7 asserts + 3 sanity checks<br/>hits 127.0.0.1:5000 live"]
        I -->|"create_sql_query, get_industry,<br/>get_companies, valid_category,<br/>valid_columns, ALLOWED_COLUMNS,<br/>create_column_array,<br/>create_adage_data_model"| E
        I -->|"run_sql, run_sql_raw"| D
        E -->|"run_sql — imported, never called"| D
        T -.->|"requests.get"| I
    end

    GDP ==>|"HTTP · JSON"| I
    D ==>|"psycopg2"| RDS[("AWS RDS")]

    style E stroke:#d97706,stroke-width:2px
```

`esg_functions.py:1` imports `run_sql` but never calls it — the module is pure query-string
construction, which is why it sits beside `db.py` rather than above it.

---

## 3. Persistence Model

Reconstructed from `ALLOWED_CATEGORIES`, `ALLOWED_COLUMNS`, and the two `industry` queries.

```mermaid
erDiagram
    ESG_CATEGORY_TABLE {
        string company_name
        string perm_id
        string data_type
        string disclosure
        string metric_description
        string metric_name
        string metric_unit
        string metric_value
        string metric_year
        string nb_points_of_observations
        string metric_period
        string provider_name
        string reported_date
        string pillar
        string headquarter_country
        string category
    }
    INDUSTRY {
        string company
        string industry
    }
    ESG_CATEGORY_TABLE }o..o{ INDUSTRY : "joined only in app code, by company name string"
```

`ESG_CATEGORY_TABLE` stands for all seven tables in `ALLOWED_CATEGORIES` — they share one identical
16-column shape, which is why `create_sql_query(table, columns, conditions)` can treat the category
name as an interchangeable table name. No foreign key is declared anywhere in the code;
`industry.company` and `*.company_name` are never joined in SQL, only queried separately.

---

## 4. Primary Data Flow — the `/get` happy path

The one path a user can drive end to end.

```mermaid
sequenceDiagram
    autonumber
    actor U as Analyst
    participant P as get-demo/page.jsx<br/>browser · 'use client'
    participant DD as DataDisplay.jsx
    participant F as Flask index.py<br/>/get
    participant EF as esg_functions.py
    participant DB as db.py · run_sql
    participant R as AWS RDS

    U->>P: select category · tick columns · type conditions
    Note over P: useState: category, columns[],<br/>conditions{}, data, loading, error, fetched
    U->>P: click "Fetch Data"
    activate P
    P->>P: setLoading(true) · setError(null) · setData(null)
    P->>P: build URLSearchParams<br/>category + columns + per-column conditions

    P->>+F: GET /get?category=environmental_risk&columns=...&company_name=Tervita+Corp
    Note over P,F: cross-origin — allowed by CORS(app)

    F->>F: request.args to category, columns,<br/>conditions = args.to_dict()
    F->>+EF: valid_category(category)
    EF-->>-F: True
    F->>F: conditions.pop("category")
    F->>+EF: valid_columns(columns)
    EF->>EF: create_column_array — strip spaces, split on comma
    EF-->>-F: True
    F->>F: conditions.pop("columns")

    F->>+EF: create_sql_query(category, columns, conditions)
    EF-->>-F: SELECT company_name,metric_name,metric_value<br/>FROM environmental_risk<br/>WHERE company_name = 'Tervita Corp'

    F->>+DB: run_sql(sql, create_column_array(columns))
    DB->>DB: load_dotenv(find_dotenv())
    DB->>+R: psycopg2.connect(host, port, db, user, password)
    R-->>-DB: connection
    DB->>+R: cursor.execute(sql)
    loop fetchone() until None
        R-->>DB: row tuple
        DB->>DB: rows.append(dict(zip(columns, row)))
    end
    deactivate R
    DB->>DB: cursor.close() · conn.close()
    DB-->>-F: list of dicts — company_name, metric_name, metric_value

    F->>EF: create_adage_data_model(res)
    Note over EF: wraps in ADAGE envelope:<br/>data_source, dataset_type, dataset_id,<br/>time_object, events[]<br/>returns json.dumps — a STRING
    F->>F: print(create_adage_data_model(res))
    Note right of F: envelope built twice —<br/>once to print, once to return
    F-->>-P: 200 · jsonify(json.dumps(...)) — double-encoded JSON

    P->>P: result = await response.json() — a string
    P->>P: JSON.parse(result) — an object
    Note over P: line 81 — this second parse exists only<br/>because the API double-encodes
    P->>P: setData(data.events)
    P->>P: finally — setLoading(false) · setFetched(true)
    deactivate P

    P->>+DD: DataDisplay data={events}
    DD->>DD: Object.keys(data[0]) for column headers
    DD->>DD: underscores to spaces · null to em-dash
    DD-->>-U: MUI table
```

Steps 26–29 are one bug wearing two costumes. `create_adage_data_model` already returns
`json.dumps(...)`, then `index.py:64` wraps that string in `jsonify`. The result is a JSON *string
containing* JSON, forcing `JSON.parse(result)` on the client. Returning the dict instead would let
frontend line 81 be deleted — a two-line fix spanning two layers.

`db.py` also calls `load_dotenv(find_dotenv())` and opens a fresh `psycopg2` connection on every
query. Against RDS that is a TLS handshake per request.

---

## 5. Failure and Rejection Paths

Traced from the actual `try` / `except` structure. All five branches are reachable from the current
UI or from `test.py`.

```mermaid
sequenceDiagram
    autonumber
    participant P as get-demo page
    participant F as Flask /get
    participant EF as esg_functions
    participant DB as db.run_sql
    participant R as AWS RDS

    P->>F: GET /get?...

    alt category present but not in ALLOWED_CATEGORIES
        F->>EF: valid_category("invalid_category")
        EF-->>F: False
        F-->>P: 200 OK · body "Error 400: Invalid category"
        Note over F,P: HTTP status is 200, not 400 —<br/>the error lives in the body as a bare string
        P->>P: JSON.parse gives a string, .events undefined
        P->>P: DataDisplay — Array.isArray(undefined) is false
        P-->>P: renders "Error: Invalid response format"

    else category parameter omitted entirely
        F->>F: if category and not valid_category(...) is False
        F->>F: else branch — conditions.pop("category")
        Note over F: KeyError — the else branch pops a key<br/>that was never in the query string
        F-->>P: 500 Internal Server Error

    else a requested column not in ALLOWED_COLUMNS
        F->>EF: valid_columns("invalid_column")
        EF-->>F: False
        F-->>P: 200 OK · body "Error 400: Invalid columns"

    else RDS unreachable or SQL error
        F->>DB: run_sql(sql, cols)
        DB->>R: connect / execute
        R--xDB: exception
        DB->>DB: except — print("Error:", e)
        DB-->>F: None
        Note over DB,F: swallowed — the caller cannot distinguish<br/>"no rows" from "database down"
        F->>EF: create_adage_data_model(None)
        EF-->>F: envelope with events set to null
        F-->>P: 200 OK · valid-looking envelope, null payload
        P-->>P: renders "Error: Invalid response format"

    else exception raised inside the route body
        F->>F: except Exception as e — jsonify(e)
        Note over F: Exception objects are not JSON-serializable,<br/>so this raises TypeError and returns 500,<br/>masking the real error — see TODO at index.py:66
    end
```

The repo's own tests encode the status-code quirk: `test_get_invalid_category` asserts
`status_code == 200` and then checks for the error string.

---

## 6. The Other Routes — divergent response shapes

Three routes, three different response contracts, which is why the frontend cannot reuse one parser.

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant F as Flask
    participant EF as esg_functions
    participant DB as db.py
    participant R as AWS RDS

    Note over C,R: /getIndustry — dict rows, NO ADAGE envelope
    C->>F: GET /getIndustry?company=PrimeCity+Investment+PLC
    F->>EF: get_industry(company)
    EF-->>F: SELECT industry FROM industry WHERE company = '...'
    F->>DB: run_sql(sql, ["industry"])
    DB->>R: execute · fetchone loop
    R-->>DB: rows
    DB-->>F: [ industry = Real Estate ]
    F-->>C: jsonify(res) — single-encoded, no envelope
    Note over C: no JSON.parse needed — unlike /get

    Note over C,R: /getCompanies — raw tuples, column names lost
    C->>F: GET /getCompanies?industry=Real+Estate
    F->>EF: get_companies(industry)
    EF-->>F: SELECT company FROM industry WHERE industry = '...'
    F->>DB: run_sql_raw(sql)
    DB->>R: execute · fetchall
    R-->>DB: list of 1-tuples
    DB-->>F: list of tuples
    F-->>C: return res — array of arrays
    Note over C: keys discarded — see TODO at index.py:104

    Note over C,R: /slowget — full esg table, no category narrowing
    C->>F: GET /slowget?columns=...
    F->>EF: create_sql_query("esg", columns, conditions)
    F->>DB: run_sql
    DB->>R: SELECT over the whole esg table
    Note over R: 70,000 companies — the repo disabled its own<br/>test_get_no_columns for exactly this reason
    R-->>DB: large result set
    DB-->>F: rows
    F-->>C: ADAGE envelope, double-encoded
```

---

## 7. Deployment Layer — CI/CD flow

From `ci.yml`, `cd.yml`, and `lint.yml`.

```mermaid
sequenceDiagram
    autonumber
    actor D as Developer
    participant GH as GitHub · main
    participant LI as Linting<br/>ubuntu-latest
    participant CI as CI Pipeline<br/>ubuntu-latest
    participant DH as Docker Hub<br/>bpetselis01
    participant CD as CD Pipeline<br/>self-hosted runner on EC2
    participant EC2 as AWS EC2 Docker daemon

    D->>GH: git push origin main

    par all three trigger on push to main
        GH->>+LI: on.push branches main
        LI->>LI: checkout@v4 · setup-node@v4 node 20<br/>yarn cache from frontend/cabana/yarn.lock
        LI->>LI: yarn install --frozen-lockfile · yarn add -D typescript
        LI->>-LI: yarn lint
        Note over LI: advisory only — nothing gates CI on this

    and
        GH->>+CI: on.push branches main
        CI->>CI: checkout@v3
        CI->>DH: docker login with DOCKER_USERNAME / DOCKER_PASSWORD secrets
        CI->>CI: docker build -t bpetselis01/flask-api<br/>-f flask-api/Dockerfile flask-api
        CI->>DH: docker push bpetselis01/flask-api:latest
        CI->>CI: docker build -t bpetselis01/nextjs<br/>-f frontend/Dockerfile .
        Note over CI: context is repo root, but that Dockerfile does<br/>COPY package*.json ./ and no such file exists at root.<br/>flask-api was fixed to a narrow context, nextjs was not.
        CI->>-DH: docker push bpetselis01/nextjs:latest

    and
        GH->>CD: on.push branches main
        Note over CD: RACE — CD also fires directly on push,<br/>in parallel with CI, so it can pull<br/>the PREVIOUS :latest images
    end

    CI-->>CD: on.workflow_run "CI Pipeline" types completed
    Note over CD: completed includes FAILED runs —<br/>there is no conclusion == success guard

    activate CD
    CD->>DH: sudo docker pull bpetselis01/flask-api:latest
    CD->>DH: sudo docker pull bpetselis01/nextjs:latest
    CD->>EC2: sudo docker rm -f flask-api-container · errors ignored
    CD->>EC2: sudo docker rm -f nextjs-container · errors ignored
    Note over EC2: containers removed before new ones start —<br/>hard downtime window, no health check, no rollback
    CD->>EC2: docker run -d -p 5900:5000 --name flask-api-container
    CD->>EC2: docker run -d -p 3000:3000 --name nextjs-container
    deactivate CD
    Note over EC2: no --env-file passed, so DB_* vars are absent<br/>in the deployed Flask container.<br/>Only docker-compose supplies ./.env
```

---

## 8. Runtime Topology — local vs deployed

Where the layers stop lining up.

```mermaid
flowchart TB
    subgraph LOCAL["LOCAL — docker-compose up --build"]
        direction TB
        LB["Browser<br/>localhost:3000"]
        subgraph LC["docker-compose network"]
            LN["nextjs<br/>3000:3000<br/>volume ./frontend/cabana:/cabana"]
            LF["flask-api<br/>5000:5000<br/>env_file ./.env<br/>FLASK_APP=index.py"]
        end
        LB -->|"serves UI"| LN
        LB ==>|"fetch 127.0.0.1:5000 — matches"| LF
        LF ==>|"DB_* from .env — present"| LR[("AWS RDS")]
    end

    subgraph PROD["DEPLOYED — AWS EC2 via cd.yml"]
        direction TB
        PB["Browser<br/>EC2_HOST:3000"]
        subgraph PE["EC2 host · plain docker run"]
            PN["nextjs-container<br/>3000:3000"]
            PF["flask-api-container<br/>5900:5000 — note 5900"]
        end
        PB -->|"serves UI"| PN
        PB x--x|"fetch http://127.0.0.1:5000<br/>resolves to the USER's own machine,<br/>and the published port is 5900, not 5000"| PF
        PF x--x|"no --env-file, so DB_HOST is unset"| PR[("AWS RDS")]
    end

    style PB stroke:#dc2626,stroke-width:2px
    style PF stroke:#dc2626,stroke-width:2px
```

`http://127.0.0.1:5000` hardcoded at `get-demo/page.jsx:76` is the seam where the four layers come
apart. Because the fetch runs in the browser, `127.0.0.1` means the visitor's own machine — never
the EC2 host. It works locally only because that machine happens to run both containers. The port
compounds it: compose publishes Flask on `5000`, `cd.yml` publishes `5900:5000`. Fixing the host
alone would still not connect. The usual resolution is a build-time `NEXT_PUBLIC_API_URL`, which
also requires CI's `docker build` to pass a matching `--build-arg`.

---

## Findings

| # | Finding | Location |
|---|---|---|
| 1 | Hardcoded `127.0.0.1:5000` — deployed frontend cannot reach the API | `get-demo/page.jsx:76` |
| 2 | Port mismatch: compose `5000`, EC2 `5900` | `docker-compose.yml` vs `cd.yml` |
| 3 | `docker run` passes no `--env-file`, so `DB_*` are unset in prod | `cd.yml` |
| 4 | **SQL injection**: condition *values* interpolated with `.format()`; `get_industry` / `get_companies` fully interpolated. Column and table names are allowlisted, values are not | `esg_functions.py:22,46,53` |
| 5 | `jsonify(json.dumps(...))` double-encodes, forcing a client-side `JSON.parse` | `index.py:64` and `page.jsx:81` |
| 6 | `conditions.pop("category")` raises `KeyError` when `category` is omitted | `index.py:49` |
| 7 | `jsonify(e)` on an Exception object raises `TypeError` and returns 500 | `index.py:66,86,96,107` |
| 8 | DB errors swallowed to `None` — "down" is indistinguishable from "no rows" | `db.py:38,68` |
| 9 | Validation failures return HTTP 200 with an error string body | `index.py:47,53` |
| 10 | CD triggers on `push` *and* `workflow_run`, racing CI — can deploy stale images | `cd.yml` |
| 11 | `workflow_run` `types: [completed]` has no success guard — deploys after failed CI | `cd.yml` |
| 12 | `rm -f` before `run` — downtime window, no health check or rollback | `cd.yml` |
| 13 | nextjs image built with root context but Dockerfile does `COPY package*.json ./` | `ci.yml` |
| 14 | New connection plus `load_dotenv` on every query; no pooling | `db.py:13,17,48,52` |
| 15 | ADAGE envelope constructed twice per `/get` request | `index.py:63-64` |
| 16 | `trending/page.js` is 0 bytes; `Searchbar` state never reaches the API | frontend |
| 17 | Prod images run `flask run` and `yarn dev` — dev servers, not production builds | both Dockerfiles |

Items 1–3 are why the deployed data flow breaks in three independent places. Item 4 is the one worth
fixing first on merit: `psycopg2` parameterization via `cursor.execute(sql, params)` is already
reachable, since `db.py` passes the query straight through.
