# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

For detailed architecture and project-structure diagrams, see [the OctoFit diagrams](./diagramas.md).

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

Implement from the data tier upward, with one contract-tested capability at a time.

- **Presentation tier decision:** Keep API calls in the shared client and have each route component use it for its own capability. Use relative `/api` paths by default and honor `VITE_API_BASE_URL`; components must visibly handle loading, empty, success, and error states with accessible status and alert roles.
- **Logic tier decision:** Keep Express routes focused on HTTP input/output and put validation and business rules in the service layer. Return deliberate 4xx responses for invalid input or missing records; do not let malformed IDs become cast-error 500s or replace failures with successful empty results.
- **Data tier decision:** Define required fields, relationships, and useful defaults in the Mongoose schemas. Seed representative users, teams, activities, and workouts with stable keys and idempotent upserts; do not delete or overwrite unrelated records.
- **Activity tracking approach:** Accept existing user/team IDs, validate their format and existence before creating an activity, and return a 400 for missing required input. Store the references, activity type, and minutes; use minutes as the initial deterministic points rule and set `occurredAt` when creating the record.
- **Team management approach:** Require a name, preserve the submitted name and mascot, and default members to an empty array. Store member references and return them when listing teams.
- **Leaderboard approach:** Aggregate stored activity points by team, include each team's name, sort totals highest-first, and assign ranks starting at 1. Return entries as `{ teamName, totalPoints, rank }`.
- **Workout suggestions approach:** Find the user by email, use their fitness goal and recent activities to select matching workouts, and include a brief reason for the first suggestion. Return a deliberate 4xx when no user matches.

## Test plan

Run the focused contract tests in dependency order, then build both tiers:

1. `npm run test:data --prefix octofit-tracker/backend` — verifies required model fields, defaults, references, and a representative seed that is idempotent and preserves unrelated records. A failure points to schema or seed behavior.
2. `npm run test:api --prefix octofit-tracker/backend` — verifies activity/team creation and validation, leaderboard ordering, and user-specific workout suggestions through HTTP. A failure points to route/service behavior or response shape.
3. `npm test --prefix octofit-tracker/frontend` — verifies API-backed rendering and accessible loading, empty, and error states for each route. A failure points to client integration or presentation behavior.
4. `npm run build --prefix octofit-tracker/backend` — checks backend TypeScript types; a failure indicates a compile-time issue not necessarily covered by runtime contracts.
5. `npm run build --prefix octofit-tracker/frontend` — type-checks and builds the Vite app; a failure indicates a frontend compile or bundling issue.
6. `node --test tests/exercise-validation.test.mjs` — checks the exercise metadata and grading safeguards. A failure points to repository-level exercise configuration, not an OctoFit tier contract.

Baseline captured before implementation: the data tests had 4 failures (required model validation) and 1 pass (seed behavior); the API tests had 4 failures (invalid input returned 501, leaderboard was empty, and a suggestion lacked its expected reason); the frontend had 15 failures across rendering and route states. The exercise validator had 4 failures out of 40; the metadata check reports a README badge/template-owner mismatch, which also prevents three other validator tests from reaching their intended assertions. Re-run each command after its corresponding changes; this baseline is not a substitute for a passing final run.
