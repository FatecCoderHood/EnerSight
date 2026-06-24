# EnerSight

Analytical platform centralizing public ANEEL data (DEC/FEC continuity indicators).

## Stack
- **frontend/**: Vue 3 + TypeScript, Vite, vue-router, vue-i18n, axios, Chart.js, Leaflet.
- **backend/enersight-api**: Java 17, Spring Boot 4 (Web MVC, Data JPA, Flyway), PostgreSQL/PostGIS, Lombok, Maven.
- **backend/enersight-worker**: Python, SQLAlchemy, psycopg2, pydantic — ANEEL data ingestion jobs.
- **backend/enersight-temporal-series**: Python, pandas, Prophet, pymongo; FastAPI + motor + uvicorn for the API service.
- Data stores: PostgreSQL/PostGIS (relational), MongoDB (time series), via `backend/docker/docker-compose.yaml`.

## Commands
- Frontend: `cd frontend && npm install`, `npm run dev`, `npm run build` (runs `vue-tsc -b`), `npm run preview`.
- API: `cd backend/enersight-api && ./mvnw spring-boot:run`, tests with `./mvnw test`.
- Worker: `cd backend/enersight-worker && pip install -r requirements.txt`, run via `python -m app.worker` or `python -m app.ingestion.job_producer`.
- Temporal series: `cd backend/enersight-temporal-series && pip install -r requirements.txt -r requirements_api.txt`, API via `Dockerfile.api` (uvicorn on port 8000).
- Full stack (db + mongo + workers): `docker compose -f backend/docker/docker-compose.yaml up`.

## Architecture
- `frontend/src`: `pages/`, `components/`, `routes/` (vue-router), `composables/`, `service/` and `api/` (axios clients), `i18n.ts` + `locales/`.
- `backend/enersight-api/src/main/java/com/enersight`: `controller/`, `service/`, `repository/`, `dto/`; Flyway migrations in `resources/db/migration`.
- `backend/enersight-worker/app`: `ingestion/` (downloader, extractor, transformer, loader, pipeline), `jobs/`, `db/` (SQLAlchemy repositories), `models/`.
- `backend/enersight-temporal-series/app`: `data/` (loader, treatment), `training/` (Prophet trainer), `storage/` (writer), `db/mongo.py`, `api/main.py` (FastAPI).
- PostgreSQL is the source of truth for relational/ingestion data; MongoDB stores derived time-series/forecast outputs.

## Constraints
- `cmdstanpy` is pinned to `1.2.4` — newer versions break Prophet's bundled CmdStan (see comment in `requirements.txt`).
- Do not hand-edit Flyway migration files that have already been applied; add a new migration instead.
- Mongo and Postgres credentials in `docker-compose.yaml` are dev-only defaults, not for production use.

## Definition of Done
- Frontend changes build cleanly (`npm run build`) with no TypeScript errors.
- Backend API changes pass `./mvnw test` and any new/changed Flyway migration runs cleanly on a fresh DB.
- Python services run without errors against the dockerized Postgres/Mongo stack.
- New env vars or config are reflected in `docker-compose.yaml`.
