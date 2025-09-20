# communityconnect
# CommunityConnect — Open-Source Community Volunteer Platform

> Owner: @JatoriJenkinsSE
>  repo: communityconnect

A friendly, lightweight volunteer management app for community groups and nonprofits.

**Highlights**

* Event & shift scheduling (create events, add time‑boxed shifts)
* Volunteer sign‑up + check‑in / check‑out
* Hours tracking & export (CSV)
* Simple JWT auth (demo) with roles: admin, volunteer
* Rust API (Axum + sqlx + SQLite) + React frontend (Vite + TS)
* Docker Compose for local dev; GitHub Actions CI; optional GHCR images

---

## Monorepo Layout

```
communityconnect/
├─ README.md
├─ LICENSE
├─ docker-compose.yml
├─ .env
├─ .github/workflows/ci.yml
├─ Cargo.toml                      # workspace
├─ crates/common/src/lib.rs
├─ services/
│  ├─ api/                         # Rust API
│  │  ├─ Cargo.toml
│  │  ├─ Dockerfile
│  │  ├─ migrations/
│  │  │  ├─ 2025-09-20-000001_init.sql
│  │  │  └─ 2025-09-20-000002_seed.sql (optional)
│  │  └─ src/
│  │     ├─ main.rs
│  │     ├─ auth.rs
│  │     ├─ events.rs
│  │     ├─ shifts.rs
│  │     ├─ volunteers.rs
│  │     └─ reports.rs
│  └─ gateway/ (optional proxy, omitted by default)
└─ web/
   ├─ index.html
   ├─ vite.config.ts
   ├─ package.json
   └─ src/
      ├─ main.tsx
      ├─ App.tsx
      ├─ api.ts
      └─ components/{EventList.tsx,ShiftBoard.tsx,VolunteerLogin.tsx}
```

---

## Workspace `Cargo.toml`

```toml
[workspace]
members = [
  "crates/common",
  "services/api",
]
resolver = "2"

[workspace.package]
version =  0.1.0
edition =  2021
author = JatoriJenkinsSE
repository = https://github.com/JatoriJenkinsSE/communityconnect
license = MIT OR Apache-2.0

[workspace.dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sqlx = { version = "0.7", features = ["runtime-tokio", "sqlite", "time", "macros"] }
jsonwebtoken = "9"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "fmt"] }
tower-http = { version = "0.5", features = ["trace", "cors"] }
thiserror = "1"
anyhow = "1"
chrono = { version = "0.4", features = ["serde"] }
uuid = { version = "1", features = ["v4", "serde"] }
```

---

## `crates/common/src/lib.rs`

```rust
use serde::{Deserialize, Serialize};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

pub fn init_tracing(service: &str) {
    let filter = std::env::var("RUST_LOG").unwrap_or_else(|_| "info,tower_http=info".into());
    tracing_subscriber::registry()
        .with(tracing_subscriber::EnvFilter::new(filter))
        .with(tracing_subscriber::fmt::layer().with_target(false).compact())
        .init();
    tracing::info!(%service, "tracing initialized");
}

pub fn must_env(k: &str) -> String {
    std::env::var(k).unwrap_or_else(|_| panic!("Missing env: {}", k))
}

#[derive(Debug, Serialize, Deserialize)]
pub struct ApiError { pub message: String }
impl ApiError { pub fn new(m: impl Into<String>) -> Self { Self { message: m.into() } } }

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct Claims { pub sub: String, pub role: String, pub exp: usize }

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct Event { pub id: String, pub title: String, pub date: String, pub location: String }

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct Shift { pub id: String, pub event_id: String, pub start: String, pub end: String, pub capacity: i64 }

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct Volunteer { pub id: String, pub name: String, pub email: String }
```

---

## API Service `services/api/Cargo.toml`

```toml
[package]
name = "communityconnect-api"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = { workspace = true }
tokio = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
sqlx = { workspace = true }
jsonwebtoken = { workspace = true }
tracing = { workspace = true }
tracing-subscriber = { workspace = true }
tower-http = { workspace = true }
thiserror = { workspace = true }
anyhow = { workspace = true }
uuid = { workspace = true }
chrono = { workspace = true }
common = { path = "../../crates/common" }
```

---

## DB Migrations

`services/api/migrations/2025-09-20-000001_init.sql`

```sql
CREATE TABLE volunteers (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  email TEXT UNIQUE NOT NULL
);

CREATE TABLE events (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  date TEXT NOT NULL,        -- ISO date
  location TEXT NOT NULL
);

CREATE TABLE shifts (
  id TEXT PRIMARY KEY,
  event_id TEXT NOT NULL REFERENCES events(id) ON DELETE CASCADE,
  start TEXT NOT NULL,       -- ISO datetime
  end TEXT NOT NULL,         -- ISO datetime
  capacity INTEGER NOT NULL
);

CREATE TABLE signups (
  volunteer_id TEXT NOT NULL REFERENCES volunteers(id) ON DELETE CASCADE,
  shift_id TEXT NOT NULL REFERENCES shifts(id) ON DELETE CASCADE,
  PRIMARY KEY (volunteer_id, shift_id)
);

CREATE TABLE hours (
  volunteer_id TEXT NOT NULL REFERENCES volunteers(id) ON DELETE CASCADE,
  shift_id TEXT NOT NULL REFERENCES shifts(id) ON DELETE CASCADE,
  check_in TEXT,  -- ISO datetime
  check_out TEXT, -- ISO datetime
  PRIMARY KEY (volunteer_id, shift_id)
);
```

*(Optional seed in `2025-09-20-000002_seed.sql`)*

```sql
INSERT INTO volunteers (id,name,email) VALUES ('vol-alice','Alice Johnson','alice@example.com');
INSERT INTO events (id,title,date,location) VALUES ('evt-park','Park Cleanup','2025-10-01','Lincoln Park');
INSERT INTO shifts (id,event_id,start,end,capacity) VALUES
  ('s1','evt-park','2025-10-01T09:00:00Z','2025-10-01T12:00:00Z',10),
  ('s2','evt-park','2025-10-01T12:30:00Z','2025-10-01T15:00:00Z',8);
```

---

## API Implementation (selected)

`services/api/src/main.rs`

```rust
mod auth; mod events; mod shifts; mod volunteers; mod reports;
use axum::{routing::{get, post}, Router};
use tower_http::{trace::TraceLayer, cors::CorsLayer};
use sqlx::SqlitePool;
use common::{init_tracing, must_env};

#[derive(Clone)]
pub struct Cfg { pub db: SqlitePool, pub jwt_secret: String }

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    dotenvy::dotenv().ok();
    init_tracing("communityconnect-api");
    let db = SqlitePool::connect(&must_env("DATABASE_URL")).await?;

    let cfg = Cfg { db, jwt_secret: must_env("AUTH_SECRET") };

    let app = Router::new()
        .merge(auth::routes())
        .merge(events::routes())
        .merge(shifts::routes())
        .merge(volunteers::routes())
        .merge(reports::routes())
        .layer(CorsLayer::permissive())
        .layer(TraceLayer::new_for_http())
        .with_state(cfg);

    let port: u16 = std::env::var("PORT").ok().and_then(|s| s.parse().ok()).unwrap_or(7000);
    axum::Server::bind(&([0,0,0,0], port).into()).serve(app.into_make_service()).await?;
    Ok(())
}
```

`services/api/src/auth.rs`

```rust
use axum::{routing::{post}, Json, Router, extract::{State}, http::{HeaderMap, StatusCode}};
use common::{ApiError, Claims};
use jsonwebtoken::{encode, decode, Algorithm, EncodingKey, DecodingKey, Header, Validation};
use serde::{Deserialize, Serialize};
use crate::Cfg;
use std::time::{SystemTime, UNIX_EPOCH};

#[derive(Deserialize)]
struct LoginReq { email: String, name: String, role: Option<String> }
#[derive(Serialize)]
struct LoginRes { token: String }

pub fn routes() -> Router<Cfg> {
    Router::new().route("/login", post(login)).route("/whoami", post(whoami))
}

async fn login(State(cfg): State<Cfg>, Json(body): Json<LoginReq>) -> Result<Json<LoginRes>, (StatusCode, Json<ApiError>)> {
    if body.email.is_empty() { return Err((StatusCode::BAD_REQUEST, Json(ApiError::new("email required")))) }
    let now = SystemTime::now().duration_since(UNIX_EPOCH).unwrap().as_secs() as usize;
    let role = body.role.unwrap_or_else(|| "volunteer".to_string());
    let claims = Claims { sub: body.email, role, exp: now + 3600 };
    let jwt = encode(&Header::new(Algorithm::HS256), &claims, &EncodingKey::from_secret(cfg.jwt_secret.as_bytes()))
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, Json(ApiError::new(e.to_string()))))?;
    Ok(Json(LoginRes { token: jwt }))
}

async fn whoami(State(cfg): State<Cfg>, headers: HeaderMap) -> Result<Json<Claims>, (StatusCode, Json<ApiError>)> {
    let token = headers.get(axum::http::header::AUTHORIZATION)
        .and_then(|v| v.to_str().ok()).and_then(|s| s.strip_prefix("Bearer "))
        .ok_or((StatusCode::UNAUTHORIZED, Json(ApiError::new("missing token"))))?;
    let data = decode::<Claims>(token, &DecodingKey::from_secret(cfg.jwt_secret.as_bytes()), &Validation::new(Algorithm::HS256))
        .map_err(|e| (StatusCode::UNAUTHORIZED, Json(ApiError::new(format!("invalid token: {}", e)))))?;
    Ok(Json(data.claims))
}
```

`services/api/src/events.rs`

```rust
use axum::{routing::{get, post, delete}, Json, Router, extract::{State, Path}, http::StatusCode};
use serde::{Deserialize, Serialize};
use sqlx::SqlitePool;
use uuid::Uuid;
use crate::Cfg;
use common::{ApiError};

#[derive(Deserialize)]
struct EventCreate { title: String, date: String, location: String }
#[derive(Serialize)]
struct EventRes { id: String, title: String, date: String, location: String }

pub fn routes() -> Router<Cfg> { Router::new().route("/events", get(list).post(create)).route("/events/:id", delete(delete_one)) }

async fn list(State(cfg): State<Cfg>) -> Result<Json<Vec<EventRes>>, (StatusCode, Json<ApiError>)> {
    let rows = sqlx::query!("SELECT id, title, date, location FROM events ORDER BY date DESC").fetch_all(&cfg.db).await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, Json(ApiError::new(e.to_string()))))?;
    Ok(Json(rows.into_iter().map(|r| EventRes { id: r.id, title: r.title, date: r.date, location: r.location }).collect()))
}

async fn create(State(cfg): State<Cfg>, Json(b): Json<EventCreate>) -> Result<(StatusCode, Json<EventRes>), (StatusCode, Json<ApiError>)> {
    let id = format!("evt-{}", Uuid::new_v4());
    sqlx::query!("INSERT INTO events (id,title,date,location) VALUES (?,?,?,?)", id, b.title, b.date, b.location)
        .execute(&cfg.db).await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, Json(ApiError::new(e.to_string()))))?;
    Ok((StatusCode::CREATED, Json(EventRes { id, title: b.title, date: b.date, location: b.location })))
}

async fn delete_one(State(cfg): State<Cfg>, Path(id): Path<String>) -> Result<StatusCode, (StatusCode, Json<ApiError>)> {
    sqlx::query!("DELETE FROM events WHERE id = ?", id).execute(&cfg.db).await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, Json(ApiError::new(e.to_string()))))?;
    Ok(StatusCode::NO_CONTENT)
}
```

`services/api/src/shifts.rs`

```rust
use axum::{routing::{get, post, delete}, Json, Router, extract::{State, Path}, http::StatusCode};
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use crate::Cfg;
use common::ApiError;

#[derive(Deserialize)]
struct ShiftCreate { event_id: String, start: String, end: String, capacity: i64 }
#[derive(Serialize)]
struct ShiftRes { id: String, event_id: String, start: String, end: String, capacity: i64 }

pub fn routes() -> Router<Cfg> { Router::new().route("/shifts", get(list).post(create)).route("/shifts/:id", delete(delete_one)) }

async fn list(State(cfg): State<Cfg>) -> Result<Json<Vec<ShiftRes>>, (StatusCode, Json<ApiError>)> {
    let rows = sqlx::query!("SELECT id, event_id, start, end, capacity FROM shifts ORDER BY start ASC").fetch_all(&cfg.db).await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, Json(ApiError::new(e.to_string()))))?;
    Ok(Json(rows.into_iter().map(|r| ShiftRes { id: r.id, event_id: r.event_id, start: r.start, end: r.end, capacity: r.capacity }).collect()))
}

async fn create(State(cfg): State<Cfg>, Json(b): Json<ShiftCreate>) -> Result<(StatusCode, Json<ShiftRes>), (StatusCode, Json<ApiError>)> {
    let id = format!("sh-{}", Uuid::new_v4());
    sqlx::query!("INSERT INTO shifts (id, event_id, start, end, capacity) VALUES (?,?,?,?,?)", id, b.event_id, b.start, b.end, b.capacity)
        .execute(&cfg.db).await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, Json(ApiError::new(e.to_string()))))?;
    Ok((StatusCode::CREATED, Json(ShiftRes { id, event_id: b.event_id, start: b.start, end: b.end, capacity: b.capacity })))
}

async fn delete_one(State(cfg): State<Cfg>, Path(id): Path<String>) -> Result<StatusCode, (StatusCode, Json<ApiError>)> {
    sqlx::query!("DELETE FROM shifts WHERE id = ?", id).execute(&cfg.db).await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, Json(ApiError::new(e.to_string()))))?;
    Ok(StatusCode::NO_CONTENT)
}
```

`services/api/src/volunteers.rs`

```rust
use axum::{routing::{get, post}, Json, Router, extract::State, http::StatusCode};
use serde::{Deserialize, Serialize};
use uuid::Uuid;
use crate::Cfg;
use common::ApiError;

#[derive(Deserialize)]
struct VolunteerCreate { name: String, email: String }
#[derive(Serialize)]
struct VolunteerRes { id: String, name: String, email: String }

pub fn routes() -> Router<Cfg> { Router::new().route("/volunteers", get(list).post(create)) }

async fn list(State(cfg): State<Cfg>) -> Result<Json<Vec<VolunteerRes>>, (StatusCode, Json<ApiError>)> {
    let rows = sqlx::query!("SELECT id, name, email FROM volunteers ORDER BY name").fetch_all(&cfg.db).await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, Json(ApiError::new(e.to_string()))))?;
    Ok(Json(rows.into_iter().map(|r| VolunteerRes { id: r.id, name: r.name, email: r.email }).collect()))
}

async fn create(State(cfg): State<Cfg>, Json(b): Json<VolunteerCreate>) -> Result<(StatusCode, Json<VolunteerRes>), (StatusCode, Json<ApiError>)> {
    let id = format!("vol-{}", Uuid::new_v4());
    sqlx::query!("INSERT INTO volunteers (id,name,email) VALUES (?,?,?)", id, b.name, b.email)
        .execute(&cfg.db).await
        .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, Json(ApiError::new(e.to_string()))))?;
    Ok((StatusCode::CREATED, Json(VolunteerRes { id, name: b.name, email: b.email })))
}
```

`services/api/src/reports.rs`

```rust
use axum::{routing::get, Json, Router, extract::State, http::StatusCode};
use serde::Serialize;
use crate::Cfg; use common::ApiError;

#[derive(Serialize)]
struct HoursRow { volunteer: String, total_hours: f64 }

pub fn routes() -> Router<Cfg> { Router::new().route("/reports/hours", get(hours)) }

async fn hours(State(cfg): State<Cfg>) -> Result<Json<Vec<HoursRow>>, (StatusCode, Json<ApiError>)> {
    // Simplified: check_out - check_in in hours
    let rows = sqlx::query!(
        r#"SELECT v.name as volunteer,
                   SUM((julianday(h.check_out) - julianday(h.check_in))*24.0) as total
            FROM hours h
            JOIN volunteers v ON v.id = h.volunteer_id
            WHERE h.check_in IS NOT NULL AND h.check_out IS NOT NULL
            GROUP BY v.name
            ORDER BY total DESC"#
    ).fetch_all(&cfg.db).await
     .map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, Json(ApiError::new(e.to_string()))))?;
    Ok(Json(rows.into_iter().map(|r| HoursRow { volunteer: r.volunteer.unwrap_or_default(), total_hours: r.total.unwrap_or(0.0) }).collect()))
}
```

---

## Frontend (Vite + React)

`web/package.json`

```json
{
  "name": "communityconnect-web",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "axios": "^1.7.2"
  },
  "devDependencies": {
    "typescript": "^5.5.4",
    "vite": "^5.0.0",
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0"
  }
}
```

`web/src/api.ts`

```ts
import axios from "axios";
export const api = axios.create({ baseURL: import.meta.env.VITE_API_URL || "http://localhost:7000" });
export const listEvents = async () => (await api.get("/events")).data;
export const listShifts = async () => (await api.get("/shifts")).data;
export const createVolunteer = async (name:string,email:string) => (await api.post("/volunteers", {name, email})).data;
```

`web/src/App.tsx`

```tsx
import { useEffect, useState } from "react";
import { listEvents, listShifts, createVolunteer } from "./api";

type Event = { id: string; title: string; date: string; location: string };

type Shift = { id: string; event_id: string; start: string; end: string; capacity: number };

export default function App() {
  const [events, setEvents] = useState<Event[]>([]);
  const [shifts, setShifts] = useState<Shift[]>([]);
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");

  useEffect(() => { (async () => {
    setEvents(await listEvents());
    setShifts(await listShifts());
  })(); }, []);

  const signup = async (e: React.FormEvent) => {
    e.preventDefault();
    await createVolunteer(name, email);
    setName(""); setEmail("");
    alert("Thanks for signing up! We'll be in touch.");
  }

  return (
    <div style={{ maxWidth: 900, margin: "2rem auto", fontFamily: "Inter, system-ui, sans-serif" }}>
      <h1>CommunityConnect</h1>
      <h2>Upcoming Events</h2>
      <ul>
        {events.map(ev => (
          <li key={ev.id}><strong>{ev.title}</strong> — {ev.date} @ {ev.location}</li>
        ))}
      </ul>

      <h2>Shifts</h2>
      <ul>
        {shifts.map(s => (
          <li key={s.id}>{s.event_id}: {s.start} → {s.end} (cap {s.capacity})</li>
        ))}
      </ul>

      <h2>Volunteer Sign‑Up</h2>
      <form onSubmit={signup}>
        <input placeholder="Your name" value={name} onChange={e=>setName(e.target.value)} />
        <input placeholder="Your email" value={email} onChange={e=>setEmail(e.target.value)} />
        <button type="submit">Sign up</button>
      </form>
    </div>
  );
}
```

---

## `.env` (dev)

```env
RUST_LOG=info
DATABASE_URL=sqlite://communityconnect.db
AUTH_SECRET=dev_change_me
PORT=7000
VITE_API_URL=http://localhost:7000
```

---

## Docker Compose

`docker-compose.yml`

```yaml
version: "3.9"
services:
  api:
    build: ./services/api
    environment:
      - RUST_LOG=${RUST_LOG}
      - DATABASE_URL=${DATABASE_URL}
      - AUTH_SECRET=${AUTH_SECRET}
      - PORT=${PORT}
    volumes:
      - ./.data:/data
    ports: ["7000:7000"]
  web:
    build: ./web
    environment:
      - VITE_API_URL=${VITE_API_URL}
    ports: ["5173:5173"]
```

`services/api/Dockerfile`

```dockerfile
FROM rust:1.80 as builder
WORKDIR /app
COPY . ../../
RUN cargo build --release -p communityconnect-api

FROM gcr.io/distroless/cc-debian12
WORKDIR /app
COPY --from=builder /app/target/release/communityconnect-api /usr/local/bin/api
ENV RUST_LOG=info
EXPOSE 7000
CMD ["/usr/local/bin/api"]
```

`web/Dockerfile`

```dockerfile
FROM node:22-alpine as build
WORKDIR /app
COPY web/package*.json ./
RUN npm ci
COPY web .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
```

---

## GitHub Actions (CI)

`.github/workflows/ci.yml`

```yaml
name: CI
on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

jobs:
  rust:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - name: Build
        run: cargo build --workspace --all-targets --locked
      - name: Test
        run: cargo test --workspace --locked
  docker:
    needs: rust
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: JatoriJenkinsSE
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build & Push API
        uses: docker/build-push-action@v6
        with:
          context: .
          file: services/api/Dockerfile
          push: true
          tags: ghcr.io/jatorijenkinsse/communityconnect-api:latest
```

---

## Quickstart (Tailored for **@JatoriJenkinsSE**)

```bash
# 1) Repo init
mkdir communityconnect && cd communityconnect
git init
# add files as above

git add .
git commit -m "feat: communityconnect volunteer platform"
git branch -M main
git remote add origin https://github.com/JatoriJenkinsSE/communityconnect.git
git push -u origin main

# 2) Run API (dev)
export DATABASE_URL=sqlite://communityconnect.db
sqlx database create || true
sqlx migrate run
RUST_LOG=info AUTH_SECRET=dev PORT=7000 cargo run -p communityconnect-api

# 3) Run Web (dev)
(cd web && npm i && npm run dev)
```

---

## Roadmap

* Add email confirmations + ICS calendar export for shifts
* Add QR code check‑in at events (mobile friendly)
* Role‑based routes in frontend, admin dashboard
* Hours CSV export endpoint `/reports/hours.csv`
* Multi‑tenant orgs with custom subdomains
* Deploy Helm chart + GHCR images for k8s
}
