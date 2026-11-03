# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

- Presentation tier: Keep route components focused on rendering. Use the shared API client for requests and provide accessible loading, error, and empty states; keep business rules in the backend.
- Logic tier: Keep Express routes thin and put validation, persistence workflows, point calculation, aggregation, and recommendation selection in `octofitService.ts`.
- Data tier: Use Mongoose schemas to require meaningful fields, model references, and default team members to an empty array. Keep seeding idempotent and preserve unrelated records.
- Activity tracking: Validate required fields, ID format, positive minutes, and that both referenced records exist before writing. Store the selected user and team, calculate points as one point per minute, and set `occurredAt` on the server. Return deliberate 4xx responses for invalid input or references.
- Team management: Require a non-empty name, preserve the submitted name and mascot, default omitted members to an empty list, and validate member IDs before saving. Keep user/team membership references consistent when members are supplied.
- Leaderboard: Sum activity points by team, include teams with no activity at zero points, sort by total points descending with team name as a deterministic tie-breaker, and assign 1-based ranks.
- Workout suggestions: Find the user by email and use the fitness goal and recent activity to rank matching workouts. Include a short reason with each recommendation and return a deliberate 4xx response when the user does not exist.

## Test plan

- `npm run test:data --prefix octofit-tracker/backend`: Schema and seed contracts; failures indicate missing persistence constraints, reference/default behavior, or seed regressions. Baseline: 4 failed and 1 passed; the four required-field checks fail, while seed idempotency passes.
- `npm run test:api --prefix octofit-tracker/backend`: HTTP behavior against an in-memory MongoDB; failures indicate route/service validation, persistence, aggregation, or recommendation mismatches. Baseline: 4 failed; activity and team creation return 501 for invalid requests, the leaderboard is empty, and workout suggestions do not match the user's activity/goal.
- `npm test --prefix octofit-tracker/frontend`: API-driven rendering and accessible loading, empty, and error states. Failures indicate presentation/API-client behavior or accessibility regressions. Baseline: 15 failed; feature views remain at the implementation-pending state.
- `npm run build --prefix octofit-tracker/backend`: TypeScript compile check; failures indicate backend type or module-contract errors. Baseline: passed.
- `npm run build --prefix octofit-tracker/frontend`: TypeScript and production Vite build; failures indicate frontend type, module, or bundling errors. Baseline: passed.
