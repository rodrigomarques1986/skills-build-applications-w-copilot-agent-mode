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

## Responsibilities by tier

- **Presentation tier (React + Vite)**
  - Owns the UX for the four main capabilities: activities, teams, leaderboard, and workout suggestions.
  - Uses React Router to navigate between routes and fetches data through the shared API client.
  - Handles the user-visible states required by the contract: loading, empty, and error.
  - The browser consumes relative `/api` URLs; Vite proxies these requests to the Express server on port `8000`.

- **Logic tier (Express + TypeScript)**
  - Exposes the application contract under `/api/*`.
  - Validates request bodies, user/team references, and missing or malformed data before writing to the database.
  - Performs domain logic such as activity scoring, team aggregation, and workout recommendation based on recent user patterns.
  - Returns deliberate 4xx errors for bad input and 2xx or 201 responses for successful operations.

- **Data tier (MongoDB + Mongoose)**
  - Persists `User`, `Team`, `Activity`, and `Workout` records.
  - Enforces schema validation and required relationships so invalid documents are rejected before they reach the service layer.
  - Seeds representative sample data idempotently so the API and UI can be exercised reliably in tests.

## Required data validation rules

The data contract is enforced at the model layer and validated by Mongoose before records are saved.

- **User**
  - `name` is required.
  - `email` is required.
  - `fitnessGoal` is required.
  - `team` is optional for creation but must reference an existing team when provided.

- **Team**
  - `name` is required.
  - `mascot` is required.
  - `members` defaults to an empty array.
  - `members` stores `ObjectId` references to users.

- **Activity**
  - `user` is required.
  - `team` is required.
  - `type` is required.
  - `minutes` is required and must be a valid positive number.
  - `points` is required and computed from activity type/minutes logic.
  - `occurredAt` is required and must be a valid date.
  - Invalid `userId` or `teamId` values must fail with a deliberate 4xx in the API layer rather than causing a Mongo cast error.

- **Workout**
  - `title` is required for the seeded record.
  - `focus` is required.
  - `minutes` is required.
  - `difficulty` is required.

## API behavior and expected HTTP responses

| Capability | Endpoint | Expected behavior |
| --- | --- | --- |
| List activities | `GET /api/activities` | Returns `{ activities: [...] }` with 200. |
| Create activity | `POST /api/activities` | Accepts `userId`, `teamId`, `type`, and `minutes`; validates references; creates activity with computed `points` and `occurredAt`; returns `{ activity: ... }` with 201. |
| Reject invalid activity input | `POST /api/activities` | Missing fields, malformed IDs, or references to non-existent users/teams must return a 4xx; never a 500 due to cast errors. |
| List teams | `GET /api/teams` | Returns `{ teams: [...] }` with 200. |
| Create team | `POST /api/teams` | Accepts `name`, `mascot`, and optional `members`; rejects missing `name` with 400; returns `{ team: ... }` with 201. |
| Leaderboard | `GET /api/leaderboard` | Returns `{ leaderboard: [{ teamName, totalPoints, rank }, ...] }` ordered by total points descending with 1-based ranks. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Locates the matching user by email, derives suggestions from recent activity patterns or matching goal, and returns `{ suggestions: [...] }` with 200. Unknown emails return a deliberate 4xx. |

## Frontend loading, empty, and error states

The React tier must behave as a proper presentation layer and expose accessible states:

- **Loading state**
  - Each route should show a visible loading indicator while the request is pending.
  - The contract expects `role="status"` with text matching `/loading/i`.

- **Empty state**
  - If the API returns an empty list, the UI should show a meaningful empty message.
  - Examples: `No activities`, `No teams`, `No leaderboard entries`, `No workout suggestions`.

- **Error state**
  - If a request fails or the backend responds with an error payload, the page should render an accessible alert.
  - The expected element is `role="alert"` and the content should mention an error or unavailable state.

- **Successful data rendering**
  - Activities display record type/minutes/points.
  - Teams display name and team members.
  - Leaderboard displays team totals and a rank.
  - Workout suggestions show title, focus, and a reason string.

## Implementation order (required)

1. **MongoDB / Mongoose data models and validation**
   - Define the Mongoose schemas for `User`, `Team`, `Activity`, and `Workout`.
   - Add required validation for all contract fields.
   - Ensure `members` defaults to an empty array and team/user references are consistent.
   - Confirm the seed script can populate representative data idempotently without duplication.
   - Acceptance criteria: the data-tier contract tests pass, and invalid document shapes fail validation as expected.

2. **Express services and API business logic**
   - Implement the service functions in the backend that create and list activities, create and list teams, compute leaderboard totals, and suggest workouts.
   - Keep validation in the API or service layer so malformed IDs and missing fields return deliberate 4xx responses, not database cast errors.
   - Compute `points` and `occurredAt` for new activities.
   - Build team aggregation and workout matching logic based on real user activity data.
   - Acceptance criteria: the API returns the correct payloads and HTTP status codes per contract.

3. **Backend API contract validation**
   - Run the backend contract suite to verify request and response shapes.
   - Confirm that invalid input is rejected with 4xx, valid creation endpoints return 201, and lookup endpoints return the correct JSON payloads.
   - Validate leaderboard ordering and workout suggestion relevance against seeded data patterns.
   - Acceptance criteria: `npm run test:api --prefix octofit-tracker/backend` passes without contract mismatches.

4. **React frontend integration and presentation states**
   - Wire the UI to the `/api` endpoints, using the shared client and proper route-level fetch patterns.
   - Replace placeholder cards with real list rendering for activities, teams, leaderboard, and workout suggestions.
   - Implement loading, empty, and error states in a way that matches the frontend contract.
   - Acceptance criteria: the frontend contract tests pass and each route renders the expected accessible content.

## Test plan and acceptance criteria

The architecture is intentionally validated in a tiered sequence: data contract first, API contract second, then frontend contract last. This preserves separation of responsibility and prevents UI work from masking backend failures.

### 1. Data-tier validation
- Command: `npm run test:data --prefix octofit-tracker/backend`
- Purpose: verifies required schema fields, defaults, and seeded data behavior.
- Acceptance criteria:
  - `User` validation rejects missing `name`, `email`, and `fitnessGoal`.
  - `Team` validation rejects missing `name` and `mascot` and ensures `members` exists as an array.
  - `Activity` validation rejects missing required fields such as `user`, `team`, `type`, `minutes`, `points`, and `occurredAt`.
  - `Workout` validation rejects missing `focus`, `minutes`, and `difficulty`.
  - Seed data creates representative records idempotently without duplicate corruption.

### 2. API contract validation
- Command: `npm run test:api --prefix octofit-tracker/backend`
- Purpose: verifies HTTP behavior and business logic across the `/api` contract.
- Acceptance criteria:
  - `POST /api/activities` with missing fields or malformed IDs returns a 4xx.
  - Valid activity creation returns 201 and includes computed `points` and `occurredAt`.
  - `GET /api/activities` lists persisted activities.
  - `POST /api/teams` rejects missing names with 400 and returns 201 otherwise.
  - `GET /api/teams` lists team records with members.
  - `GET /api/leaderboard` returns descending totals with rank numbers starting at 1.
  - `GET /api/workouts/suggestions/:userEmail` returns matching suggestions; unknown users return a 4xx.

### 3. Frontend presentation validation
- Command: `npm test --prefix octofit-tracker/frontend`
- Purpose: verifies that the React tier consumes the API contract correctly and exposes accessible states.
- Acceptance criteria:
  - Activities route renders API data and not the placeholder card.
  - Teams route renders teams and members.
  - Leaderboard route renders numeric totals and ranks.
  - Workouts route requests suggestions for the seeded user email and renders the first suggestion.
  - Loading state is announced via `role="status"` and loading text.
  - Empty state is displayed for empty payloads.
  - Error state is displayed as an accessible alert.

### 4. Type-check and build validation
- Backend build command: `npm run build --prefix octofit-tracker/backend`
- Frontend build command: `npm run build --prefix octofit-tracker/frontend`
- Purpose: ensures TypeScript compiles cleanly across the project after the tiered implementation is completed.
- Acceptance criteria:
  - No TypeScript errors in either backend or frontend.
  - The project builds successfully after the contract-driven implementation finishes.

## Summary

The architecture requires a strict sequence of implementation: validate the data model first, then implement business logic in Express, then verify the API contract, and only then finish the React presentation layer. This ensures the responsibilities stay separated while the application can be validated in a repeatable, test-driven workflow.
