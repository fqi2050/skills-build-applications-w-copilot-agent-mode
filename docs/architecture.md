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

Record your plan here before editing code.

- Presentation tier decision: Keep pages focused on rendering and user interaction; use the shared API client with relative `/api` paths and the Vite proxy. Each capability page handles loading, empty, and error states accessibly, while the service owns HTTP and persistence behavior.
- Logic tier decision: Keep routes thin and delegate validation and data operations to the service layer. Return the response shapes in the API contract, map invalid input and missing resources to deliberate 4xx responses, and never let malformed IDs surface as cast-error 500 responses. Do not add authentication.
- Data tier decision: Make Mongoose schemas enforce required fields and sensible defaults, including an empty `members` array for teams. Keep seeded examples representative and make seeding repeatable without deleting or rewriting unrelated records.
- Activity tracking approach: Validate the request shape, positive finite minutes, and well-formed IDs for an existing user and team before writing. Store the submitted references and activity type, calculate points from minutes at one point per minute, and set `occurredAt` when the activity is created. List activities with their stored references and computed fields.
- Team management approach: Require a non-empty name and mascot; default omitted members to an empty array. Validate member IDs before creating a team, persist submitted name and mascot, and return member IDs consistently from create and list endpoints.
- Leaderboard approach: Sum persisted activity points per team, sort by total descending, and assign ranks starting at 1. Return each row as `teamName`, `totalPoints`, and `rank`.
- Workout suggestions approach: Find the user by email or return 404. Prefer workouts matching the user's fitness goal, using the most recent activity type as a fallback when needed; return a concise reason tied to the goal or activity and put the best match first.

## Test plan

Record the commands you run and what each failure means.

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend` — verifies required model fields, defaults, references, and repeatable non-destructive seeding. Failures indicate a schema or seed-data contract regression.
- API command: `npm run test:api --prefix octofit-tracker/backend` — verifies create/list response shapes, invalid and missing reference handling, leaderboard totals/ranks, and personalized suggestions. Failures indicate incorrect route/service behavior, persistence, or status codes.
- Frontend command: `npm test --prefix octofit-tracker/frontend` — verifies API-driven content, the requested user for suggestions, and accessible loading, empty, and error states on each capability page. Failures indicate a presentation or API-client integration regression.
- Backend build command: `npm run build --prefix octofit-tracker/backend` — type-checks the backend; failures indicate TypeScript errors in models, services, routes, or server wiring.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend` — type-checks and bundles the frontend; failures indicate TypeScript or production-build errors.
- Run the data tests first, then the API tests, then frontend tests; finish with both builds. At the current baseline, the data suite has 4 failing tests (model validation), the API suite has 4 failing tests (unimplemented validation, leaderboard, and suggestion reasoning), and the frontend suite has 15 failing tests (incomplete rendering and async states). Re-run all five commands after implementation and record the results here.
