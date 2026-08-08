# MythicPanel

Full-stack World of Warcraft character dashboard that combines profile, Mythic+, raid, PvP, realm, season, and progression-history data in one interface.

I built the project as a split-stack application: a Java and Spring Boot API owns every external integration, while a Next.js frontend focuses on server-rendered presentation and interaction.

[Source code](https://github.com/rmb34/wowMythicPanel) · [LinkedIn](https://linkedin.com/in/lucas-da-silva-santos-a46879285)

> MythicPanel is an independent project and is not affiliated with or endorsed by Blizzard Entertainment or Raider.IO.

---

## Product Overview

Character progression data is distributed across different services. Blizzard provides the official character, equipment, raid, realm, season, and PvP data, while Raider.IO provides the Mythic+ scoring and run information players commonly use.

MythicPanel aggregates those sources into a single character page. The frontend sends one request to the application backend and receives a unified response instead of coordinating multiple third-party APIs in the browser.

The result is a focused dashboard for inspecting a character's current state and recent progression without exposing integration credentials to the client.

---

## Main Features

### Character Search and Profile

Users can search for a character by realm and name. Realm autocomplete is populated through the Blizzard API and cached by the backend.

The profile presents:

- Character name and realm.
- Region.
- Class and active specialization.
- Faction and race.
- Guild.
- Character level.
- Average and equipped item level.
- Official character artwork.

The visual theme adapts to the character's faction while preserving the same component structure.

### Mythic+

The Mythic+ area combines Raider.IO data with the current Blizzard season:

- Current rating when available.
- Highest completed key.
- Best runs.
- Dungeon and key level.
- Completion time and timer result.
- Run score.
- Active affixes.
- Historical season scores.
- Role-specific rating breakdown.

When Raider.IO has runs but has not yet published an aggregate season score, the interface falls back to the available run and highest-key information rather than hiding the entire section.

### Raid Progression

Raid data comes from Blizzard and is restricted to the latest expansion returned by the API.

The dashboard displays:

- Raid instances.
- Progress by difficulty.
- Completed and total encounter counts.
- Individual bosses.
- Boss-level kill counts.

### PvP

The PvP section covers:

- 2v2 arena.
- 3v3 arena.
- Rated Battlegrounds.
- Solo Shuffle.
- Rating and tier label.
- Matches played, won, and lost.
- Honor level and honorable kills.

### Progress History

Successful character lookups can create PostgreSQL snapshots containing:

- Mythic+ rating.
- Highest key completed.
- Equipped item level.
- Average item level.
- Capture timestamp.

Snapshots are limited to one record per character within a six-hour interval. A scheduled cleanup removes records older than 30 days. The frontend turns the stored history into timeline charts.

### Multi-Region API

The backend accepts the `us`, `eu`, `kr`, and `tw` regions and validates region, realm slug, and character name at the controller boundary.

The main search experience currently targets US realms, while the routing and API contracts already support the other configured regions.

---

## Engineering Highlights

### Backend-for-Frontend Aggregation

The Spring Boot service is the single integration boundary for Blizzard and Raider.IO.

```text
Browser
   |
   v
Next.js Server Components
   |
   v
Spring Boot API
   |
   +---- Blizzard OAuth token service
   +---- Blizzard profile, media, raid, PvP, realm, and season clients
   +---- Raider.IO Mythic+ client
   +---- PostgreSQL character cache and snapshots
```

The frontend receives one `CharacterDashboardResponse`. It never calls the external providers directly and never receives Blizzard client credentials.

This boundary also centralizes validation, error mapping, rate limiting, caching, and provider-specific behavior.

### Typed Integration Layer

Each provider has its own client, DTOs, configuration, and exception type.

Controllers return a shared API envelope with data, metadata, or a sanitized error. A global exception handler translates validation errors, missing characters, upstream failures, and unexpected exceptions into stable HTTP responses.

Services return response DTOs rather than JPA entities, keeping persistence details out of the public API contract.

### Graceful Degradation

The two upstream integrations fail differently and are handled accordingly.

- A Raider.IO `400` or `404` means the character is not indexed there. Mythic+ is omitted while the Blizzard-backed profile can still render.
- Other Raider.IO failures become a typed upstream error.
- If Blizzard is unavailable and the character has already been stored, the backend can return its cached basic profile.
- If neither the API nor the local cache has the character, the request returns a not-found response.

The fallback is intentionally described as a cached profile, not as a complete cached dashboard: Mythic+, raid, and PvP data are not persisted in the same form.

### OAuth and Reference-Data Caching

Blizzard access tokens are cached per region until shortly before expiration, avoiding an OAuth request for every character lookup.

Realm and season reference data use Caffeine caches. Character profile data and history use PostgreSQL because they must survive application restarts and support timeline queries.

### Snapshot Retention

The snapshot service checks the most recent record before writing. A new snapshot is allowed only after the configured interval has elapsed.

A daily scheduled job deletes snapshots outside the retention window, keeping the history useful without allowing unbounded growth.

### Proxy-Aware Rate Limiting

The backend uses Bucket4j token buckets with a stricter policy for character endpoints.

Client identification checks `X-Forwarded-For`, then `X-Real-IP`, and finally the servlet remote address. This accommodates deployment behind a reverse proxy while keeping the policy inside the API layer.

---

## Architecture

```text
Next.js frontend
      |
      | server-side HTTP calls
      v
Spring Boot REST API
      |
      +---- Controller validation
      +---- Application services
      +---- Provider clients
      +---- Exception translation
      +---- Rate limiting
      |
      +-----------> Blizzard API
      |                 |
      |                 +---- OAuth 2.0 client credentials
      |                 +---- profile and media
      |                 +---- raid and PvP
      |                 +---- realms and seasons
      |
      +-----------> Raider.IO API
      |                 |
      |                 +---- Mythic+ scores and best runs
      |
      v
Spring Data JPA
      |
      v
PostgreSQL
      |
      +---- cached character profiles
      +---- progression snapshots
```

The frontend uses Server Components by default. Client Components are reserved for browser-side interactions such as tabs, season selection, and charts.

---

## Technology Stack

| Layer | Technology |
|---|---|
| Backend | Java 21, Spring Boot 3 |
| HTTP integrations | Spring WebFlux `WebClient` |
| Persistence | Spring Data JPA, PostgreSQL 16 |
| Database migrations | Flyway |
| Caching | Caffeine |
| Rate limiting | Bucket4j |
| Frontend | Next.js 16, React 19, TypeScript |
| UI | Tailwind CSS 4, shadcn/ui, Recharts, Framer Motion |
| Backend testing | JUnit 5, MockMvc, Mockito, MockWebServer, H2 |
| Frontend testing | Vitest, React Testing Library, MSW |
| CI | GitHub Actions |
| Deployment | Docker, Render, Vercel |

---

## Security and Reliability

- Blizzard credentials are read from environment variables and remain in the backend.
- The frontend calls the backend from the server rather than exposing the provider origin as a browser integration.
- Region, realm, and character parameters are constrained at the controller boundary.
- Production startup fails when the allowed CORS origins are not configured.
- Character and general API endpoints use separate token buckets.
- Provider exceptions are converted to sanitized public messages.
- Security headers are applied by both the Spring Boot API and Next.js frontend.
- The frontend Content Security Policy restricts scripts, connections, frames, fonts, and image origins.
- Only the health actuator endpoint is exposed without internal actuator details.
- Database changes are versioned through Flyway migrations.

---

## Testing and Continuous Integration

The backend suite covers:

- Character response aggregation.
- Provider fallbacks.
- Blizzard OAuth token reuse.
- Blizzard and Raider.IO HTTP behavior.
- DTO deserialization from representative payloads.
- Controller validation and error envelopes.
- Realm and season services.
- Snapshot interval and history queries.
- Repository behavior.
- Rate limiting and proxy-address resolution.
- CORS and security headers.
- Actuator exposure.
- End-to-end dashboard aggregation at the service boundary.

The frontend suite covers:

- API-client responses and failures.
- Character profile rendering.
- Mythic+, raid, PvP, and history components.
- Season switching and empty states.
- Source and last-updated indicators.
- Search and navigation behavior.

GitHub Actions runs two independent jobs on pull requests and pushes to `main`:

```text
Backend:  tests -> build
Frontend: install -> lint -> tests -> production build
```

---

## Running Locally

Requirements:

- Java 21.
- Node.js 20 or newer.
- Docker.
- Blizzard API credentials.

```bash
docker compose up -d

cp backend/.env.example backend/.env
cd backend && ./gradlew bootRun

cd frontend
npm install
npm run dev
```

The repository also provides `./dev.sh` to start the local services together.

Flyway applies the database migrations when the backend starts.

---

## Current Limitations

- The main search interface is currently centered on US realms, although the backend supports four regions.
- The Raider.IO request includes explicit historical season slugs. New expansions can require a code update until season discovery is made dynamic.
- The cached fallback preserves the basic character profile, not every external dashboard section.
- Snapshot history has a rolling 30-day retention window.

---

## Author

**Lucas da Silva Santos** — Full Stack Developer

[LinkedIn](https://linkedin.com/in/lucas-da-silva-santos-a46879285)

> World of Warcraft and Blizzard Entertainment are trademarks or registered trademarks of Blizzard Entertainment, Inc. Raider.IO is a separate third-party service.
