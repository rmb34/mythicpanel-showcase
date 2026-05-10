# MythicPanel

> A World of Warcraft character dashboard built as a full-stack project — aggregating live data from Blizzard's API and raider.io into a single, fast profile view.

**By Lucas da Silva Santos — Full Stack Developer | Co-founder at [Repetz](https://repetz.com.br)**

---

Most WoW character lookup tools make you visit three or four different websites to get a complete picture of a player. Blizzard's own armory for gear and raid progression. raider.io for Mythic+ scores. Another tool for PvP ratings. And none of them talk to each other.

MythicPanel aggregates all of it into a single dashboard: Mythic+ rating with historical season comparison, raid progression by difficulty, PvP bracket ratings and tier labels, item level, guild — pulled live and unified in one response.

The project was also a deliberate exercise in building a clean split-stack architecture: a Spring Boot backend as the single source of truth for all external API communication, and a Next.js frontend that never touches Blizzard or raider.io directly.

> 🔒 The source repository is private. This README documents the architecture, stack, and technical decisions behind the project.

---

## What It Does

**Mythic+ Dashboard**

Displays the character's current season rating pulled from raider.io, with a fallback chain for seasons where the aggregate isn't yet computed (new expansion launches). Shows the ten best runs with dungeon, key level, time, affixes, and individual run scores. Historical season scores with a switcher to compare across seasons.

**Raid Progression**

Raid history from the Blizzard API, filtered to the latest expansion. Shows progression per instance, per difficulty (LFR / Normal / Heroic / Mythic), with boss-level kill counts.

**PvP Brackets**

Arena (2v2, 3v3), Rated Battlegrounds, and Solo Shuffle ratings. Tier labels (Combatant through Gladiator) with win/loss records and win rate per bracket, pulled via the Blizzard PvP API.

**Character Profile**

Class, spec, faction, race, guild, item level (average and equipped), avatar image. Faction-themed UI — Alliance gets blue tones, Horde gets red.

**Snapshot History**

Periodic snapshots of rating, item level, and highest key completed, stored in the database and surfaced as a history chart. The system only snapshots when data meaningfully changes, so the history is signal rather than noise.

**Graceful Degradation**

If Blizzard's API is down, the backend falls back to the last cached character data. If raider.io returns a 400 or 404 (character not indexed), the M+ section gracefully hides rather than breaking the page. The UI always renders something useful.

---

## Architecture

```
Browser → Next.js (SSR) → Spring Boot Backend → Blizzard API
                                              → raider.io API
                                              → PostgreSQL (cache + snapshots)
```

The backend is the only layer that communicates with external APIs. The frontend calls one endpoint and gets a single unified `CharacterDashboardResponse`. This keeps credentials server-side, centralizes rate limiting, and makes the frontend a pure presentation layer.

```
Controller → Service → Repository
              ↓
          BlizzardClient   (OAuth2 token + REST calls)
          RaiderIoClient   (public API, no auth)
```

Services return DTOs, never JPA entities. Controllers delegate and return. No business logic in either layer boundary.

On the frontend, pages are Server Components by default. Client Components are used only where state or browser APIs are required — the season switcher, tab navigation. The API client lives in `lib/api.ts` and is the only place that talks to the backend.

---

## Tech Stack

**Backend**
- Java 21 / Spring Boot 3
- Spring Data JPA + Flyway migrations
- Spring WebFlux WebClient (reactive HTTP client for external APIs)
- PostgreSQL (Supabase in production)
- Bucket4j for rate limiting
- Blizzard OAuth2 client credentials flow

**Frontend**
- Next.js 14 (App Router)
- TypeScript
- Tailwind CSS + shadcn/ui
- OKLCH color system for faction theming

**Infrastructure**
- Backend: Render (Docker)
- Frontend: Vercel

---

## Interesting Problems

**raider.io doesn't have current season data at expansion launch**

When a new WoW expansion starts, raider.io's `mythic_plus_scores_by_season` takes time to aggregate per-season totals. The API returns an empty array even for characters with completed runs.

The fix: request `mythic_plus_scores_by_season:current:previous:season-tww-2:...` with explicit season slugs instead of the undocumented `:all` alias, which stopped working at the Midnight expansion launch. This gets the correct aggregate when available. When `seasonScores` is empty but `bestRuns` has data, the UI falls back to showing highest key completed and individual run scores instead of hiding the M+ section entirely.

**Blizzard API returns realm as a slug in URLs, but the armory uses display names**

The API path uses slugs (`frostmourne`) and the display uses formatted names ("Frostmourne"). The backend stores and uses slugs throughout, but serializes them as `realm` in the JSON response for consistency with the URL convention the frontend already uses in routing.

**Rate limiting behind a reverse proxy**

The rate limiter initially read `request.getRemoteAddr()` — which behind Render's proxy always returns the proxy's IP, not the client's. Fixed by reading `X-Forwarded-For` first (splitting on comma for multi-hop chains), then `X-Real-IP`, with `getRemoteAddr()` as final fallback.

**Snapshot frequency control**

The snapshot service runs on every successful API call, but only commits a new record if enough time has passed since the last one and if the data has actually changed meaningfully (rating or item level delta above a threshold). Otherwise, a character looked up ten times an hour would generate ten identical history points.

---

## Security

- Input validation on all path parameters (`region`, `realm`, `name`) with `@Pattern` constraints at the controller layer
- Rate limiting per client IP (Bucket4j, token bucket algorithm)
- Blizzard credentials stored as environment variables, never committed
- Security headers on both layers: CSP, `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`
- All external API exceptions are typed (`BlizzardApiException`, `RaiderIoException`) and handled in a single `@ControllerAdvice` that sanitizes error messages before sending them to the client

---

## Testing

Backend: JUnit 5 + Mockito. Unit tests on Services with mocked external clients. `@WebMvcTest` integration tests on Controllers with MockMvc. MockWebServer for testing HTTP client behavior (400/404/500 from raider.io).

Frontend: Vitest + React Testing Library. Component tests with mocked data. `vi.stubGlobal("fetch", ...)` for API client error handling tests.

Test conventions: names follow `should_<result>_when_<condition>()` on the backend and `should <result> when <condition>` on the frontend.

---

## What I'd Do Differently

The raider.io season logic currently hardcodes specific season slugs in the query string. A better approach would be to fetch the current season slug from raider.io's static data endpoint and build the query dynamically, so it doesn't require a code change each expansion.

The snapshot system stores history per character but doesn't yet expose a useful visualization beyond a raw line chart. Rating progression tied to patch dates would be more informative — showing when a key week happened, when the expansion launched, when the character went on a break.

---

*Built with Java 21 · Spring Boot 3 · Next.js 14 · PostgreSQL · Blizzard API · raider.io API*
