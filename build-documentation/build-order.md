# Build Order

## Step 1: Clone and Run the Skeleton

- Clone repo at [https://github.com/nstjern-uw/site-template](https://github.com/nstjern-uw/site-template)
- Connect site in S3 bucket: sync your S3 site into `S3_content/`, run it on your EC2 with Docker Compose, get the login flow working end-to-end, enable the JS sliver, customize at least one Bootstrap thing, push a PR with passing CI.

## Step 2: Determine Roles

- **Server-side** — Flask routes, OAuth, external API integration, business logic, templates
- **Client-side** — JS, CSS, browser interactivity, presentation
- **Db-and-security** — schema, migrations, indexes, input validation, secret hygiene
- **Coordinator** — runs the coordinating-LLM session that produces contracts, owns `CONTRACTS.md`, reviews cross-role PRs, runs integration tests on the shared EC2. The coordinator role often pairs with security work (dependency audits, secret rotation).

## Step 3: Project Planning Details

Any of the following for more detail:

1. **Project name and pitch.** One or two sentences. What does your app do, for whom, and why does it matter?
2. **Team members and roles.** Each member, with their assigned role:
  - Server-side
  - Client-side
  - Db-and-security
  - Coordinator (4-person teams only; for 3-person teams, omit)
3. **Project type.** Which of the seven project types from the Weeks lecture (slides 38–43), or a custom variant (with one sentence describing how yours differs).
4. **Endpoints planned.** A list of 5–8 routes your app will serve, each described in one line. For example:
  ```
   GET  /workouts          List the user's workouts
   POST /workouts          Create a new workout
   GET  /workouts/<id>     Show one workout
   POST /workouts/<id>/delete    Delete a workout
   GET  /stats             Show the user's weekly summary
  ```
   Don't write full method signatures or response schemas. One line per endpoint. The point is that your team has agreed on the app's surface.
5. **Web-based data source.** The external API your app will use. Include:
  - API name and link to docs
  - One specific endpoint your app will hit (e.g., `https://api.openweathermap.org/data/2.5/weather?q=<city>`)
  - Free-tier limits if known (e.g., 60 requests/minute, 1M requests/month)
   This forces you to confirm the API actually exists, has a free tier, and gives you the data your project needs.
6. **Schema sketch.** Tables and columns. No types, no relationships, no indexes — just the structure. For example:
  ```
   Users      (id, email, password_hash, created_at)
   Workouts   (id, user_id, type, duration_minutes, date)
   Goals      (id, user_id, target, deadline)
  ```
   This forces your team to think concretely about persistence.
7. **Link to your team's S3 bucket.** From your team's GitHub org's S3 (set up in Team Assignment 1).
8. **Link to your team's GitHub org.** The org URL.

---

## Step 4: Week 6 Setup

Do this before any role work starts.

### Step 4.1 — Confirm your team's project repo exists

If your team doesn't already have a shared project repo from Week 4's team assignment, your coordinator creates one now:

- From `lhhunghimself/week_5_506_starter` on GitHub, click **Use this template → Create a new repository**
- Name it after your project (e.g., `studyspot`, `book-tracker`, whatever your About page named)
- Owner: your team's GitHub org or the coordinator's account
- Add all teammates as collaborators with write access
- The default branch is `main`

This repo lives for the rest of the course. It's the team's project, not just Week 6's.

### Step 4.2 — Each member clones the team repo to their own EC2

```bash
git clone https://github.com/<team-org>/<project-name>.git
cd <project-name>
docker compose up -d
```

The skeleton from Week 5 is already there — the home page, `/site/`, the auth flow, all of it. You should be able to log in and click around just like Week 5.

### Step 4.3 — Re-enable the CI workflow

The skeleton ships with `.github/workflows/test.yml` set to manual-trigger only. Week 6 turns it back on.

In your team's project repo, edit `.github/workflows/test.yml`. Find this section near the top:

```yaml
"on":
  workflow_dispatch:    # manual trigger only — Week 5
  # Week 6 will switch to:
  # pull_request:
  #   branches: [main, master]
  # push:
  #   branches: [main, master]
```

Replace it with:

```yaml
"on":
  pull_request:
    branches: [main, master]
  push:
    branches: [main, master]
```

Listing both branch names means the workflow fires whether your repo defaults to `main` (GitHub's default since 2020) or `master` (older convention). Use whichever your team's repo actually has — you don't need to rename anything.

Commit this change directly to your default branch as part of your Week 6 setup.

### Step 4.4 — Configure branch protection on your default branch

Once CI is running, lock the merge button behind passing CI. In GitHub:

1. Repo **Settings → Branches → Add branch protection rule**
2. Branch name pattern: your default branch (`main` or `master` — check **Settings → General** to confirm)
3. Check **Require status checks to pass before merging**
4. Search for the test check (the job name from the workflow file) and require it
5. Optionally check **Require a pull request before merging** (recommended)
6. Save

Now any PR to your default branch shows the CI status; if CI is red, the merge button is disabled. The lecture's slide on CI gating covers what this means in detail.

---

## Week 6

### Part 1 — Group: The Coordinator-LLM Session (3 marks)

Before any role-implementation work begins, your team's coordinator runs an LLM session and commits these artifacts to the team repo:

- `CONTRACTS.md` in the repo root — your team's agreed spec
- `coord_session.md` in the repo root — your transcript of the LLM session, lightly cleaned up
- Four test files in `tests/` — one per role, all initially failing

The coordinator does this work in a single PR titled **"Week 6 contracts"** and merges it before role-implementation work starts. Teammates can review the PR, but the coordinator is the author.

> **Note:** The Brew Crew worked-example repo includes `CONTRACTS.md` but does not include a `coord_session.md`. We don't ship a fictional example transcript — simulated AI dialogue would undermine trust. Your transcript is the real artifact; produce one and commit it. I read it as part of grading.

#### What `CONTRACTS.md` must include

Modeled on the Brew Crew example. Required sections:


| Section                        | What goes there                                                                                                                                                                              |
| ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Schema                         | Tables, columns, types, constraints, foreign keys. Include any new tables this week and any changes to skeleton tables.                                                                      |
| Endpoint contracts             | For each new route: HTTP method, path, auth requirement, request shape, response shape, error cases (status codes + envelope).                                                               |
| External API contract          | Your project's data source. Endpoint URL, auth (key or none), rate limits, response shape, what your code does when it fails (timeout, rate-limit, malformed).                               |
| Authorization rules            | Who can read what, who can write what, and what response code a non-authorized user gets. Include the OWASP-style 404-for-not-yours rule if your project has ownership-restricted resources. |
| Role boundaries                | What files each role owns and what they don't touch.                                                                                                                                         |
| Known limitations (deliberate) | Things you're explicitly punting to a later week, with reasons. Required, even if the list is short.                                                                                         |


**Format:** prose with markdown tables. **Length:** probably 100–300 lines. Less than 100 is too sparse; more than 300 means you're over-specifying.

#### What the four test files must do

Modeled on Brew Crew's four test files. The coordinator commits all four, all initially failing.

- `tests/test_<role1>_<purpose>.py` for server-side — assertions on endpoint request/response shapes; mock external API with `responses` library
- `tests/test_<role2>_<purpose>.py` for db-and-security — assertions on schema (tables, columns, FKs, constraints) and auth behavior (login required, ownership rules)
- `tests/test_<role3>_<purpose>.py` for client-side — Flask test client + BeautifulSoup, assertions on form structure and selectors (NOT text content — let the client-side person change copy freely)
- `tests/test_integration.py` for the coordinator — end-to-end flow that exercises every other role's work; passes only when all three teammates have shipped

Each test file's docstring identifies its owner by name and role.

#### What the `coord_session.md` transcript must show

Edit your transcript for clarity (remove typos, off-topic asides) but keep the structure: questions asked, decisions made, places where you pushed back on the LLM, places where you pinged teammates for input. I read this to verify the coordinator wasn't doing all the work themselves.

If your transcript shows the coordinator dictating decisions and the LLM rubber-stamping, you didn't do the assignment. The session is supposed to surface things you didn't know.

#### Group rubric (3 marks)


| Criterion                                                                             | Marks |
| ------------------------------------------------------------------------------------- | ----- |
| All required `CONTRACTS.md` sections present and concrete                             | 1     |
| Four test files committed, all initially failing as expected                          | 1     |
| `coord_session.md` shows real engagement — questions, pushback, teammate consultation | 1     |


---

### Part 2 — Individual: Implement Your Slice + E2E Walk (5 marks)

By the team's submission deadline, your role's tests pass and you've designed and run an end-to-end walk for your slice.

The exact scope depends on which role you signed up for in your team's About page. Below are the four role lanes — find yours.

#### What's an "e2e walk for your slice"?

The lecture's live demo showed why: unit tests pass while user-visible behavior breaks, because synthetic test fixtures don't exercise what real systems actually do. That failure mode is called **truthy fixtures** — the mock looks right but isn't verified against reality.

Your individual e2e walk is the discipline that catches this for your slice. Each role's walk is different because each role exercises different surfaces. Per-role guidance below.

The deliverable for each role: a section in your repo's `e2e.md` (or a per-role file under `e2e/<role>.md`) that has:

- **Definition** — what does end-to-end mean for your slice? (1–2 sentences)
- **Walk** — numbered steps that exercise your slice against real conditions. Each step says exactly what to do and what should happen.
- **Pass criteria** — for each step, what specifically counts as "this worked." Distinguish "appeared to work" from "actually worked correctly."
- **Execution log** — what happened when you ran it. Pass/fail per step. Honest findings score higher than clean runs. If you discover a contract gap or a real bug, document it, fix it, link the fix commit. That's worth more than "all 8 steps passed."

**Format:** markdown, 30–80 lines per role. Less than 30 is too sparse; more than 80 means you're writing prose, not a checklist.

#### Server-side role

Implement the routes in `CONTRACTS.md`.

- New routes go in `app.py` (or a `routes.py` if your team prefers — but get the coordinator's sign-off if so)
- At least one route uses the `requests` library to call your external API
- Handle the failure modes named in the contract (timeout, rate-limit, malformed)
- Pass `tests/test_<your_test_file>.py`

**Your e2e walk:** hit your routes against the deployed app, with all external services live. Use `curl` or a Python script with `requests`, not pytest. Each route gets exercised with at least one realistic input and one error case. For external-API integrations (geocoder, books API, etc.), your walk must hit the real service — that's the whole point.

Example walk steps for a typical project:

1. `POST /register` with realistic data → expect 302 redirect, user row in DB
2. `GET /<your-search-route>?q=<realistic-query>` → expect 200, JSON has results, results match expected shape
3. `GET /<your-search-route>?q=<weird-query-the-API-might-trip-on>` → document what comes back; is it what the contract expects?
4. `GET /<your-detail-route>/<id>` with non-existent id → expect 404

The third bullet is where truthy fixtures get caught. Pick a query you wouldn't have thought to mock for, and see what real API does.

#### Client-side role

Build the templates that consume the routes.

- New templates in `templates/`
- Bootstrap-styled (use the same classes from Week 5)
- Updates to `templates/base.html` if your project's navigation needs new entries
- Forms that POST to the routes match what the server-side role implemented (cross-check with them)
- Pass `tests/test_<your_test_file>.py`

**Your e2e walk:** open your templates in a real browser against the deployed app. Click through every form, every link, every state. Verify the rendered behavior matches what the contract describes. Pytest with BeautifulSoup verifies HTML structure; the browser verifies that humans can actually use it.

Example walk steps:

1. Open `/<your-list-page>` in browser, anonymously. Verify navbar, search form, empty-state message all render.
2. Submit the search form with realistic input. Verify results appear. Click into a result.
3. Log in. Return to the list page. Verify navbar changes (logged-in state).
4. Open a detail page and submit the relevant form. Verify success flash. Verify the page reflects the change.
5. Try the form with invalid input. Verify error flash, form re-renders with values preserved.

Browser testing surfaces things pytest never will: visual layout breaking, JavaScript errors in the console, forms that submit to wrong URLs, flash messages that get swallowed by template logic.

#### DB-and-security role

Land the schema and refactor auth.

- New SQLModel models matching `CONTRACTS.md` schema section
- Refactor auth from raw `session["user_id"]` to Flask-Login (`login_user`, `logout_user`, `current_user`, `@login_required`)
- Set up `LoginManager` and the user-loader callback in `app.py`
- Verify ownership rules in your tests (e.g., 404 when non-owner attempts edit)
- Pass `tests/test_<your_test_file>.py`

The Flask-Login refactor is real work — it's not just "add the import." Lecture slides on Flask-Login walk the actual diff.

**Your e2e walk:** verify the schema and auth behavior in the deployed Postgres, not just in pytest's fixtures. Pytest uses ephemeral state; production has real constraints firing under real load.

Example walk steps:

1. Exec into Postgres: `docker compose exec db psql -U app -d app`. Run `\d <your-tables>`. Verify schema matches `CONTRACTS.md` exactly: column types, NOT NULL constraints, foreign keys, UNIQUE constraints.
2. Try to insert a duplicate via SQL: `INSERT INTO ratings (user_id, cafe_id, ...) VALUES (1, 1, ...) twice`. Verify the UNIQUE constraint actually rejects the second one. Don't trust SQLModel's ORM-level enforcement; verify the database does it.
3. Verify ON DELETE CASCADE: delete a user, verify their ratings disappear too. (Don't actually do this in production data — use test rows.)
4. Browser walk of auth flow: register, log in, log out, log back in, verify Flask-Login's `_user_id` is in the session cookie (use browser dev tools).
5. Direct ownership probe: log in as user A, attempt to edit user B's rating via direct URL. Verify 404 (not 403).

The second bullet is where many bugs hide. ORMs sometimes fudge constraints in ways the database doesn't.

#### Coordinator role

You commit `CONTRACTS.md` and the four test files first, before role-implementation work begins. Once that's merged, your job is:

- Make `tests/test_integration.py` pass (it goes green only when all three teammates' tests pass)
- Help unblock teammates: if a teammate hits a question about whether a route's response shape is right, you're the keeper of "what we agreed."
- If the contract genuinely needs to change mid-week, run a small follow-up LLM session and commit the updated `CONTRACTS.md` + updated tests in a PR titled **"Contract revision: reason"**.
- If you finish early, pick up whichever role is most behind and pair with them.

**Your e2e walk:** the whole-system walk that gets folded into the team's `e2e.md` (Part 3 deliverable). You're the one running `CONTRACTS.md` section 7 end-to-end against the deployed app, with all external services live, before the team submits. Your walk is what catches truthy fixtures across role boundaries — bugs that no individual role's slice can surface alone.

Your `e2e.md` contribution is the team's combined walk, not just your slice's. See Part 3 for the structure.

#### Each role's individual rubric (5 marks)


| Criterion                                                                | Marks |
| ------------------------------------------------------------------------ | ----- |
| Code committed and merged via PR (CI green at merge time)                | 1     |
| Implementation matches `CONTRACTS.md` specification                      | 2     |
| Your role's test file is all-green at end of week                        | 1     |
| E2E walk for your slice — designed, run, documented honestly in `e2e.md` | 1     |


**A note on CI debugging.** If your workflow fails and you want to debug without making fake commits, you can re-run a workflow from the Actions tab — manually, with optional debug logging. Manual re-runs don't appear in your git history; they're a debugging tool, not a deliverable mechanism. The study guide's CI/CD section walks the procedure (manual trigger, debug logging, the `gh` CLI). Use it freely — debugging CI by re-running rather than by spamming commits is the better engineering habit.

The 1 mark for the e2e walk is small but load-bearing. Honest findings score higher than clean runs. If your walk surfaces a contract gap and you fix it (revising `CONTRACTS.md` + tests + code), document it in your execution log and link the fix commit — that's full marks. If your walk reports "all 8 steps passed, no issues" — that's worth investigating before you submit. Real e2e walks against real services almost always surface something. Clean walks usually mean either you didn't actually hit the real service or you're not looking carefully.

---

### Part 3 — Group: Whole-system E2E (2 marks)

By the team's submission deadline, your team commits an `e2e.md` to the repo root (or `e2e/whole_system.md` if you prefer subdirectories). This is the team's end-to-end test definition for the whole project.

**Why we're not having me walk through your demo live:** because watching me walk through it makes the e2e an evaluation exercise, not an engineering exercise. I want you to learn the discipline of defining what to test and running it yourself. The e2e doc is graded; I read it instead of replicating it.

#### What `e2e.md` must include

- **Definition** — one paragraph. What does end-to-end mean for this project? Name the boundaries: browser → Flask → Postgres → which external services. (5–8 sentences)
- **The walk** — numbered list of steps that exercises the system end-to-end. Has to cover your main user flows. Has to hit each external integration at least once with realistic data. Each step says exactly what to do and what should happen.
- **Pass criteria** — for each step, what counts as "this worked." Specific. "User sees results" is not enough; "user sees results that are actually relevant to the query (not unrelated entities), with names, addresses, and other expected fields populated" is.
- **Execution log** — what happened when the team ran it. Pass/fail per step. What did you find? A team that walks their e2e and discovers one or two real issues, fixes them, and documents the fix — that's the assignment done well. A team that reports "all green, no findings" is suspicious by default. I will spot-check by running one step from your walk and comparing what I see to what your log claims.
- **Per-role contributions** — short note showing which role contributed which steps. (The team's e2e is composed of overlapping individual walks; this section just says who owned what.)

A complete worked example of `e2e.md` lives in the demo repo at [https://github.com/lhhunghimself/study_spot_demo/blob/master/e2e.md](https://github.com/lhhunghimself/study_spot_demo/blob/master/e2e.md) — read it to see what a thorough one looks like, including a real finding documented end-to-end. Don't copy it; your project's flows and external services are different.

The template below gives you the skeleton to fill in for your own project. Copy it into a new `e2e.md` at the root of your team's repo and replace the bracketed placeholders.

```markdown
# [Project name] — End-to-End Walk

**Team:** [team name]
**Coordinator:** [name]

## 1. Definition

[One or two paragraphs. What does end-to-end mean for *your* project? Name
the boundaries: browser ↔ Flask ↔ Postgres ↔ [external service, if any].
If your project has no external API, the walk is the full UI flow plus
session lifecycle. Either is fine — be specific about what your system
spans and what your walk has to exercise.]

## 2. The walk

[Numbered steps. 6 to 15 is typical, depending on project complexity.
Cover your main user flows. **Hit each external integration at least once
with realistic input** — that's where truthy fixtures hide. Each step says
exactly what to do and what should happen.]

### Setup

**Step 1.** [e.g., "Fresh state: `docker compose down -v && docker compose up -d`. Wait. Verify pytest passes." — first step is usually environment.]

### [Group of related steps — e.g., "Anonymous user flow"]

**Step 2.** [...]

**Step 3.** [...]

### [Next group — e.g., "Search via [your external API]"]

**Step N.** [The step that hits your external service with a realistic input.
This is your truthy-fixtures-catching moment.]

### [More groups as needed — authenticated flows, edit/delete, error paths, etc.]

## 3. Pass criteria

[One bullet per step from section 2. Be specific. "User sees results"
is too vague; "User sees results that are *actually relevant to the
query* (not unrelated entities), with names, addresses, and other
contract-specified fields populated" is the right level.]

- **Step 1**: [criterion]
- **Step 2**: [criterion]
- ...

## 4. Execution log

[Document what you observed when you ran it, not what you hoped.
Pass/fail per step. If you found and fixed something, document the
fix with commit hashes. If everything genuinely passed, say so with
evidence — and be a little suspicious of yourself, because clean
e2e walks against real services are rare.]

| Step | Result | Notes |
|------|--------|-------|
| 1    | PASS   | [what you observed] |
| 2    | [PASS/FAIL] | [what you observed; if fail, what you did about it] |
| ...  |        |       |

[Below the table, write up any findings. The worked example shows the
shape; here's the skeleton:]

### Finding [N] — [short title]

**Symptom**: [What was broken in user-visible terms.]

**Root cause**: [Where in the contract → test → code chain the gap
lives. Was it a missing field in the contract? A test fixture that
didn't exercise the real condition? A code-level bug?]

**Fix**: [What changed, with commit hashes. Ideally three commits if
the fix touched contract + tests + code, since fixing at the source
means all three follow.]

**Lesson**: [What this teaches about your system's verification gaps.
This is the part that scores the second mark — engineering reflection,
not just bug-tracking.]

## 5. Per-role contributions

[Show which role contributed which steps. Even on a 3-person team,
this section forces explicit role coverage and makes integration
visible.]

| Role | Contribution to this walk |
|------|--------------------------|
| [Coordinator name] | [e.g., "Steps 1, 9, 10 (integration boundaries); composed the whole walk"] |
| [Server-side name] | [steps owned] |
| [Client-side name] | [steps owned] |
| [DB-and-security name] | [steps owned] |

## 6. What we'd do differently next time

[Optional but encouraged. Honest reflection on your team's process —
what would have caught issues earlier, what tools or practices you'd
want for next week. The worked example shows the shape.]

- [Bullet 1]
- [Bullet 2]
- [Bullet 3]
```

#### Notes on filling in the template

- The template is a skeleton, not a script. Add or remove sections if your project genuinely needs different shape. Don't shoehorn a no-external-API project into the "Search via your external API" framing — replace that group with whatever exercises your main user flows.
- Length isn't the criterion. The worked example is ~150 lines; yours might be 80–200 depending on project complexity. A short e2e doc that reflects honest engineering scores better than a long one that pads out the sections.
- You may have zero findings to report. That's possible if your project has no external API and your unit tests already covered the main flows well. If so, your execution log is all PASS, your section 4 says "no findings — here's how we know" with evidence (screenshots of network panel showing real API calls, console output, etc.), and you skip the Finding subsection entirely.
- Per-role contributions don't have to be evenly distributed. Coordinator typically owns the integration-boundary steps; the role with the external API integration typically owns the most steps. Honest distribution beats forced parity.

---

## Step 5: Week 7

### What this week is about

Week 6 ended on the truthy-fixtures lesson: tests against mocks verify mocks, not reality, and end-to-end tests against real services are the only thing that breaks out of the synthetic stack.

Week 7 makes that lesson operational. You will:

- Integrate a service you didn't build — GitHub OAuth — replacing or augmenting the password login from Week 6.
- Verify it with a tool that drives a real browser — Playwright — because BeautifulSoup tests cannot click a "Login with GitHub" button and follow a redirect.
- Harden the session with cookie flags, CSRF protection, and a sensible session lifetime.

The pedagogical thread: OAuth is the canonical case where you cannot honestly mock the service you depend on. The provider's behavior is not in your repo. Unit tests that pretend to know what GitHub returns are exactly the truthy-fixture hazard Week 6 warned about. Playwright against a representative flow is how you escape it.

#### Stack additions


| Tool                         | Purpose                                                                                    | Where it sits                                  |
| ---------------------------- | ------------------------------------------------------------------------------------------ | ---------------------------------------------- |
| Authlib                      | OAuth client for Flask                                                                     | server-side, adds two routes                   |
| Playwright (Python bindings) | Browser-driven E2E tests                                                                   | new `tests/e2e/` directory                     |
| Flask-WTF                    | CSRF token integration                                                                     | wraps forms, integrates with Flask-Login       |
| python-dotenv                | Loads `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET`, `SECRET_KEY` from a gitignored `.env` file | imported once at the top of `app.py`           |
| GitHub OAuth app             | the provider                                                                               | created in your team's GitHub org for the demo |


GitHub is the default provider for this assignment. Google works (Authlib supports it identically); the study guide covers the differences. Pick GitHub unless your team has a specific reason.

---

### Part 1 — Group: Contracts revision for OAuth (2 marks)

The coordinator runs a planning session with the team's LLM-of-choice to revise `CONTRACTS.md` so it specifies the OAuth flow. Good contracts include the tests that enforce them — how you and the LLM decide to capture those tests in the contract is up to you. This contract is harder than the ones you've written so far, because part of it lives on a server you don't control.

**Deliverable:** an updated `CONTRACTS.md` in your repo, plus a short `coord_session.md` capturing the planning session (raw transcript or honest summary).

The revised contract must specify:

- The two new routes (`/login/github`, `/auth/github/callback`) and their inputs/outputs
- What user-info fields you require from the provider and what you do if a field is missing
- The shape of the local user record after a first-time OAuth login
- The link between a local user and an external identity (one user, possibly multiple OAuth identities)
- The session state immediately after a successful callback (what's in the session dict, what cookies are set)
- Logout: what you clear locally, what you do not clear at the provider

What you **cannot** specify in your contract: GitHub's actual behavior. You can specify what your code does with the response, not what the response will be. Note this honestly in the contract — `external_dependency: github.com` — see study guide for representative payload shape.

#### Marking (2 marks total)


| Criterion                                                                                                | Marks |
| -------------------------------------------------------------------------------------------------------- | ----- |
| Contract covers the six items above with concrete types/shapes, not vague prose                          | 1     |
| `coord_session.md` shows real engagement with the cross-role implications, not just a divided to-do list | 1     |


---

### Part 2 — Individual: Role work + one Playwright test (5 marks)

Each team member implements their role's slice of the OAuth + Playwright + hardening work, then writes one Playwright test that exercises their slice end-to-end through a real browser.

Reuse your Week 6 walkthrough where it fits. You designed a test path last week — it doesn't get thrown away. Take your Week 6 walkthrough, insert the OAuth login step at the top (via the test-login backdoor), and the rest of the walkthrough should already exercise the part of the app your role touches. Hand the updated walkthrough to your LLM with the Playwright prompt template and you've got your Part 2 test. If your Week 6 walkthrough doesn't naturally cover your Week 7 slice (you didn't touch the cafe search this week, etc.), then write a small new walkthrough — the per-role test examples below are the minimum each role's test must verify.

Submit a per-person `role_work.md` listing files you touched and one paragraph explaining what your Playwright test verifies (and which Week 6 walkthrough you adapted, if applicable). The test itself lives in `tests/e2e/`.

#### Per-role focus

**Server-side**

- Wire up Authlib with the GitHub provider
- Implement `/login/github` (initiates the flow) and `/auth/github/callback` (handles the return)
- Handle the create-or-link logic: new user vs returning user vs existing local user adding GitHub
- Map missing/null provider fields to sensible defaults; never crash on a partial payload
- **Your Playwright test:** full happy-path login, ending on a page that asserts `Logged in as <username>` is visible

**Client-side**

- "Sign in with GitHub" button on the login page; keep the password form alongside (don't delete it yet — Week 8 may revisit)
- Post-login UX: where does the user land? Make this deliberate, not accidental
- "Remember me" toggle wired to the session lifetime config
- Logout button that actually clears state and lands on a sensible page
- **Your Playwright test:** a logged-out user clicking "Sign in with GitHub", completing login via the test-login backdoor, and seeing their username in the navbar

**DB-and-security**

- Add an `oauth_identity` table (or columns) linking external provider IDs to local users
- Migration script if your project uses migrations; otherwise schema update committed cleanly
- Set cookie flags: `SESSION_COOKIE_SECURE`, `SESSION_COOKIE_HTTPONLY`, `SESSION_COOKIE_SAMESITE='Lax'`
- Configure `PERMANENT_SESSION_LIFETIME` and the "remember me" cookie via Flask-Login
- Wire Flask-WTF CSRF protection to every state-changing form
- **Your Playwright test:** a protected page is inaccessible without login, accessible after login, and inaccessible again after logout — verified through the rendered DOM

**Coordinator**

- Drive Part 1's contracts session and produce `coord_session.md`
- Maintain the integration log: when a teammate's change breaks a contract, note it, surface it to the team, and track the resolution
- Set up the test-login backdoor (see §"The test-login backdoor — code and how to use it" below)
- Set up GitHub OAuth app credentials and document where they live (`.env.example`, never committed `.env`)
- **Your Playwright test:** a smoke test that asserts the app starts, the login page loads, and the GitHub button is present and clickable — the cheapest possible canary

#### What "one Playwright test" means

One test function, scoped to one user-visible behavior. It must:

- Use a real browser context (Playwright launches Chromium by default)
- Drive the UI as a user would (click, type, navigate) — not call your Flask routes directly
- Assert against the rendered DOM (`expect(page.locator(...)).to_be_visible()`), not against your code's internal state
- Run from pytest like the rest of your tests

A template, the backdoor route, and the config wiring are in §"The test-login backdoor — code and how to use it" below.

#### Marking (5 marks total)


| Criterion                                                                                                                              | Marks |
| -------------------------------------------------------------------------------------------------------------------------------------- | ----- |
| Role work is committed, runs, and matches the (possibly-revised) contract                                                              | 1     |
| The Playwright test runs and passes against your local stack                                                                           | 1     |
| The test exercises a real user-visible behavior — clicking real elements, asserting on real rendered output, not bypassing the browser | 1     |
| The test would actually catch a regression in your slice — not a tautological assertion (`assert True` after a navigation)             | 1     |
| `role_work.md` honestly describes what you did, what the test covers, and any known gaps                                               | 1     |


---

### Part 3 — Group: Full-system Playwright suite + team walkthrough (3 marks)

After individual tests land, the team produces (a) a small Playwright suite exercising the full login lifecycle as one user would experience it, and (b) a walkthrough document explaining what the suite verifies. This is the Week 7 analog of Week 6's whole-system narrated walk — now scripted and documented.

The reason this lives in the team part: Playwright is the portable skill from this week. OAuth you'll look up when you need it. Playwright tests you'll write in every web codebase you touch from here on. The walkthrough is how the team demonstrates collective understanding of what its e2e coverage actually buys it.

**Deliverables:** `tests/e2e/test_full_lifecycle.py` plus `team_walkthrough.md` at the repo root.

The suite must include at minimum:

- **First-time OAuth login:** a user with no existing local account logs in via GitHub (through the test-login backdoor), arrives at the post-login page, and has a row in the `oauth_identity` table
- **Returning OAuth login:** the same user, logged out, logs in again; the existing row is reused, not duplicated
- **CSRF protection works:** a POST to a state-changing endpoint without a token is rejected (use Playwright's request context to send a tokenless POST and assert the response)
- **Session expires:** with a short test-only `PERMANENT_SESSION_LIFETIME`, after the lifetime passes (use Playwright's time controls or just sleep in a fast test), the protected page is no longer accessible

`team_walkthrough.md` is the centerpiece. It must:

- Have one section per test, naming what user-visible behavior is exercised and what specific regression the test would catch ("if someone broke the create-or-link branch by always inserting a new row, this test would fail because the second login would find a duplicate oauth_identity")
- Have a **gaps** section: what the suite does not cover, with one-line rationale. "We don't drive the actual GitHub redirect — the test-login backdoor stands in for everything after authorize_redirect" is the right kind of disclosure.
- Read as a coherent walk-through, not a per-test bullet list. A new teammate should be able to understand the team's e2e coverage in ten minutes.

#### Marking (3 marks total)


| Criterion                                                                                                                                                                    | Marks |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----- |
| All four scenarios present, named clearly, pass when run from `pytest tests/e2e/test_full_lifecycle.py`                                                                      | 1     |
| `team_walkthrough.md` walks through each test concretely — the regression it catches and the user-visible behavior it verifies are both named in specifics, not generalities | 1     |
| The walkthrough's gaps section is honest. "We don't test X because Y" is what I'm looking for. "We test everything" is not.                                                  | 1     |


---

### Secrets management with `.env`

OAuth gives you two strings that must never reach GitHub: your `OAUTH_CLIENT_ID` (semi-secret) and your `OAUTH_CLIENT_SECRET` (very secret — anyone who has it can act as your OAuth app). The Flask `SECRET_KEY` that signs session cookies is in the same category. None of these belong in the repo.

This setup is standard boilerplate — the design judgment is in the requirements, not the code. Your team's LLM will produce a correct implementation if you give it the requirements cleanly. This is one of the more honest "humans design, AI implements" moments in the assignment: you're specifying the contract, not typing the eight lines of glue.

#### What your setup must include

- A `.env` file in the repo root holding `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET`, `SECRET_KEY`, and `DATABASE_URL`
- `.env` listed in `.gitignore`, never committed
- `app.py` loads `.env` via `python-dotenv` before any `os.environ` lookups
- All secret reads use `os.environ["KEY"]` (raises on missing) rather than `os.environ.get("KEY")` — a missing secret should crash the app on startup, not surface as a confusing runtime error two minutes later
- A `.env.example` checked into the repo documenting the variable names with placeholder values (this is what I clone before grading — if it's missing or incomplete, I can't run your app, and that counts against you)
- `python-dotenv` added to `requirements.txt`

#### How to get there

Two options. The first is recommended because it lets the LLM adapt the implementation to whatever conventions your team is already using.

**Option A — Prompt your LLM (recommended).** Adapt the requirements above into a prompt. If your team uses a `config.py` class pattern, dependency injection, or an `init_app` factory function, say so in the prompt — the boilerplate should follow your code's conventions, not the assignment's. You can also paste the reference implementation from Option B below into your prompt as a known-good baseline for the LLM to modify ("use this as a starting point, but adapt it to our existing config pattern").

A minimal version of the prompt:

> "In app.py, add startup loading of a .env file via python-dotenv. Load SECRET_KEY, OAUTH_CLIENT_ID, OAUTH_CLIENT_SECRET, and DATABASE_URL. Use os.environ[...] (square brackets, not .get) so missing vars raise on startup. Also: add python-dotenv to requirements.txt, add .env to .gitignore, and produce a .env.example documenting the variable names with placeholder values."

Review the LLM's output against the requirements list. Common things to check: `load_dotenv()` is called before any `os.environ` lookup, the example file is named `.env.example` (not `env.example` or `.example.env`), and `python-dotenv` made it into `requirements.txt` (not just locally installed).

**Option B — Copy the reference implementation directly.** It satisfies the requirements as written. Use this if your team doesn't have its own conventions to adapt, or as the baseline for Option A.

**Reference implementation**

```bash
# .env — gitignored, never committed
OAUTH_CLIENT_ID=Ov23li...
OAUTH_CLIENT_SECRET=ghp_...
SECRET_KEY=<32 bytes of randomness, e.g. python -c 'import secrets; print(secrets.token_hex(32))'>
DATABASE_URL=postgresql://app:app@db:5432/app
```

```gitignore
# .gitignore
.env
```

```python
# top of app.py
from dotenv import load_dotenv
load_dotenv()  # MUST come before any os.environ lookups

import os
app.config["SECRET_KEY"] = os.environ["SECRET_KEY"]
OAUTH_CLIENT_ID = os.environ["OAUTH_CLIENT_ID"]
OAUTH_CLIENT_SECRET = os.environ["OAUTH_CLIENT_SECRET"]
```

```bash
# .env.example — committed; documents required variables, no real values
OAUTH_CLIENT_ID=your-github-client-id
OAUTH_CLIENT_SECRET=your-github-client-secret
SECRET_KEY=generate-a-random-32-byte-hex-string
DATABASE_URL=postgresql://app:app@db:5432/app
```

#### What this does not protect against

Anyone with shell access to your machine, your CI system, or your production server can read `.env`. Secrets management at scale uses secret stores (AWS Secrets Manager, HashiCorp Vault, GitHub Actions secrets, etc.). `.env` is appropriate for development and small deployments. Week 8 talks about handing secrets to a deployed container.

---

### The test-login backdoor — code and how to use it

OAuth is verified through the actual GitHub redirect manually. Everything after the redirect is verified through the test-login backdoor: a route enabled only when `app.config["TESTING"]` is true, which logs in a named user and redirects. Production never sets `TESTING`, so this route is never available outside tests.

This is a mock — and that's fine. Mocking external services is standard practice in any production test suite. The discipline isn't don't mock; it's be explicit about what you've mocked. The "What this does not test" section at the bottom is where we do that naming.

#### Step 1 — add the backdoor route to `app.py`

```python
@app.route("/test/login/<username>")
def test_login(username):
    if not app.config.get("TESTING"):
        abort(404)
    user = db.exec(select(User).where(User.email == f"{username}@test")).first()
    if user is None:
        user = User(email=f"{username}@test", display_name=username)
        db.add(user); db.commit(); db.refresh(user)
    login_user(user)
    return redirect("/dashboard")
```

The 404-when-not-testing guard is the whole safety story. If your production config doesn't set `TESTING`, the route is unreachable in production.

#### Step 2 — enable TESTING in your test fixture

In `tests/e2e/conftest.py`, set `app.config["TESTING"] = True` before yielding the live server (your coordinator handles this). Without this, the backdoor returns 404 and your tests fail at the first navigation.

#### Step 3 — call the backdoor from your Playwright test

Just navigate to the URL. No UI link is needed — `page.goto` is enough.

```python
import pytest
from playwright.sync_api import Page, expect

def test_logged_out_user_sees_login_button(page: Page, live_server):
    page.goto(f"{live_server.url}/")
    expect(page.get_by_role("link", name="Sign in with GitHub")).to_be_visible()

def test_dashboard_requires_login(page: Page, live_server):
    # logged out: protected page redirects to /login
    page.goto(f"{live_server.url}/dashboard")
    expect(page).to_have_url(f"{live_server.url}/login")

    # log in via the backdoor — bypasses GitHub entirely
    page.goto(f"{live_server.url}/test/login/alice")

    # logged in: dashboard reachable, username visible
    page.goto(f"{live_server.url}/dashboard")
    expect(page.get_by_text("Logged in as alice")).to_be_visible()

    # log out: back to the redirect behavior
    page.get_by_role("link", name="Log out").click()
    page.goto(f"{live_server.url}/dashboard")
    expect(page).to_have_url(f"{live_server.url}/login")
```

#### What this approach does not test

The actual GitHub redirect — the part where your `/login/github` route hands off to github.com and the user authorizes there — is not exercised by these tests. That's the documented gap. Verify it manually once when you wire it up; document it in `team_walkthrough.md` as "we don't test the GitHub redirect itself, because the test-login backdoor stands in for everything after the redirect lands back at our callback."

The study guide covers the alternatives (real-provider tests, fake-OAuth-server tests) and why we don't recommend either for this assignment.

---

### A note on the test database

Your team's running app uses Postgres — both in `docker compose up` for development and in production. Your Playwright test fixture uses SQLite, on purpose. The `conftest.py` sets `DATABASE_URL` to a `/tmp/...sqlite` file before importing the app, so the test process gets a hermetic SQLite database while the running container's Postgres is left alone.

This is deliberate. Three reasons:

1. **Hermetic:** each test run drops and recreates its tables, so there's no leftover state from earlier runs. No flaky tests because a previous run inserted a row that the current test wasn't expecting.
2. **No container dependency:** the test process spins up its own DB without needing a Postgres container running alongside. Tests are fast and self-contained — pytest works from a clean checkout with no docker setup.
3. **SQLModel abstracts the SQL:** the ORM emits effectively the same SQL for both backends for the operations this app uses, so most behavior is identical on both.

This is the same "honest mock with a named gap" pattern as the test-login backdoor — we're not pretending SQLite is Postgres, we're using a different DB on purpose, with a known limitation. The limitation: SQLite and Postgres differ in some behaviors (JSON column operations, certain constraint semantics, transaction isolation levels). For features that depend on Postgres-specific behavior, you'd want at least one integration test running against real Postgres — testcontainers is the standard Python tool for that. Not in scope this week, but it's the right pattern when you have a Postgres-specific feature you need to verify.

**Practical consequence for development:** data written by your running app (the Postgres container) and data written by your test runs (SQLite in `/tmp`) don't see each other. If you manually walk a flow in `docker compose up` and see weird state in `/cafes`, that's leftover Postgres data, not test data. Wipe with `docker compose down -v` to start fresh.

---

## Part 6: Week 8 — Production Stack Hardening

### Background

This week we walked through the production stack — nginx in front of gunicorn in front of Flask in front of a database — and treated it as a hardening exercise. Each layer hardens something: nginx hardens the network edge, gunicorn hardens the Python process, the docker network hardens the trust boundary around the database, and the release workflow hardens the path from your laptop to production.

Your project has run on `flask run` until now. This assignment is where it stops doing that.

Week 7 introduced you to hardening at the application layer — cookie flags, CSRF, session lifetime — but those flags were inert on `http://localhost`. This week is the follow-through. The cookie flags from Week 7 actually fire now, because the stack runs over HTTPS. The hardening discipline extends from the app outward to nginx, the deploy pipeline, and the trust boundaries between containers.

The companion Week 8 Study Guide has the canonical configs, the full attack-path test, and the fuller per-role LLM prompts. The assignment points you to specific study guide sections by number (§4, §10, etc.) when you need the worked-out details. Read the study guide section, paste the relevant config into your LLM, and adapt it to your project.

---

### Part A — Team integration (2 marks, team)

Add nginx + gunicorn in front of your project. The Study Guide has the canonical configs you'll adapt — paste them into your LLM and say "adapt this to my project."


| What you need                                      | Where it lives |
| -------------------------------------------------- | -------------- |
| `nginx.conf` (the reverse proxy in front of Flask) | Study Guide §4 |
| `gunicorn.conf.py` (the WSGI server config)        | Study Guide §5 |
| `docker-compose.yml` (the three-container setup)   | Study Guide §6 |
| Dockerfile + self-signed cert generation           | Study Guide §7 |


Each team member does their own hardening work on their own branch and integrates it properly.

#### Branch structure (do this)

1. From your team repo's `main`, create a team branch named `hardening`.
2. Each team member creates a personal branch off `hardening` named `<name>-hardening` (use your first name or GitHub handle — pick one and stick with it).
3. Each member commits their hardening work to their personal branch — frontend's nginx headers, backend's gunicorn config, DB/security's nginx server block and attack-path test, etc.
4. Each personal branch is merged into `hardening` via PR. Conflicts get resolved together.
5. Once the stack runs end-to-end on `hardening`, merge `hardening` into `main`.

Don't delete the personal branches. They're part of the evidence.

The merge structure is the point. It forces each student to actually integrate their work — not just hand it in alongside someone else's. Conflicts surface in the merge; resolving them is the "composition problem" from the coordination questions, lived in your own repo.

#### What "done" looks like

- `docker-compose up` from `main` brings up nginx, your app under gunicorn, and your database.
- nginx listens on 443 with a self-signed cert (generated, not committed).
- `https://localhost` reaches your Flask app via nginx → gunicorn.
- The stack runs from `main`. The `hardening` branch and all `<name>-hardening` branches still exist in the repo.
- Your README has a **"Running the production stack"** section.

#### Evidence (one per team)

- Link to your team repo's `main` branch with the stack committed.
- Link to the merged `hardening` branch.
- The list of `<name>-hardening` branches that were merged into it, with a link to each.

That's it — no screenshots. The repo history is the artifact.

---

### Part B — Common questions (3 marks, per person)

Seven short questions. Everyone on the team answers all seven, individually. Keep answers short, specific, and in your own voice — as long as needed to make the point, no longer. You can ask your LLM about any of these; the questions are designed so that thinking about them produces understanding regardless of where the words come from.

1. What does nginx do that your Flask app shouldn't or can't?
2. What does gunicorn do that `flask run` doesn't?
3. "Hardening" means making something harder to misuse. What's one specific thing your stack is now harder to misuse than it was last week? Point at something concrete.
4. If you wanted to add a load balancer to this picture, where would it go, and what problem would it solve that nginx isn't already solving?
5. What's a single point of failure in your current setup? There's more than one acceptable answer.
6. If someone runs `docker-compose down` on production, what happens to the data in your database? The answer depends on what your team's compose file looks like — go check.
7. What's one thing you learned about your stack from your LLM this week that surprised you, and why?

Submit as `common_questions.md`.

---

### Part C — Role-specific hardening (5 marks, per person)

Each role goes deep on the hardening work in their own layer. Write about your project specifically — generic explanations that could apply to any Flask app earn fewer marks than ones that name your actual routes, models, or quirks.

#### A note on team size and roles

The standard team for this course is three people: frontend, backend, and DB/security. On a small team, the database role and the security role are the same person — because in real small teams, they always are. The DB/security person handles the database layer, the network edge (nginx, attack paths, rate limits), and the deploy pipeline (secrets, CI). All of that is the security surface, and security isn't optional.

On a four-person team, that workload splits: the DB/security role keeps the database and the network edge, and a separate coordination role takes on secrets and the deploy pipeline. Same content, distributed differently based on team size.

Find your role below — DB/security people, note that your section has two paths depending on team size.

#### A note on the LLM probe

Every role's last sub-question is the same shape: use the inline prompt summary in your role section as a starting point (the fuller version with "what to look for in the response" guidance is in the Study Guide §21–§25), adapt it to your project, paste your relevant config or code into it, and run it through your LLM. Then write up what came back.

---

#### Frontend role

You own what the browser sees. Your hardening surface is at the edge between nginx and the user.

1. **Security headers.** The nginx config can set headers that change how the browser treats your site — Strict-Transport-Security, X-Frame-Options, X-Content-Type-Options, Content-Security-Policy, Referrer-Policy. Pick three. For each: what does it do, what attack does it mitigate, and what would break in your project if you set it too strictly?
2. **Static assets.** In `flask run`, static files were served by Flask. In the new stack, nginx can serve them directly. What changes for your project? Which files in your repo move out of Python's hands? What's the performance and security argument for that move?
3. **The cookie flags from Week 7, in real life.** Week 7's `SESSION_COOKIE_SECURE=True` was inert on `http://localhost`. With self-signed HTTPS, it's active. Walk through what changes: when does the session cookie get sent, when doesn't it, what would have broken in dev if we'd set this earlier?
4. **Debugging from the browser side.** A user reports "I keep getting logged out after every redirect." You're the frontend person. What do you check first, and why? What's the LLM useful for here, and what isn't it useful for?
5. **Have the LLM security-test your frontend layer.** Use the short prompt below as a starting point (fuller version with response-handling guidance in Study Guide §21). Adapt it to your project: paste your nginx security headers and CSP, ask the LLM to evaluate what's missing, what's too strict, and what an attacker could still do. Submit the conversation as `llm_probe_frontend.md` (lightly edited for length is fine) and write up what came up — what you'd change, what you'd push back on, what surprised you. Long enough to show what you took away.
  > **Short version of the prompt:** "I'm hardening the frontend of a Flask app. Here's our nginx security header config: [paste]. Here's our CSP: [paste]. Evaluate against best practices: (a) what's missing for our case, (b) what's too strict and would break us, (c) what an attacker could still do. Tell me what you'd change and why."

---

#### Backend role

You own what runs in the Python process. Your hardening surface is at the gunicorn/Flask boundary.

1. **Why gunicorn, concretely.** `flask run` has three production-disqualifying properties. Name them, explain each, and say what gunicorn does differently for each.
2. **Worker model.** gunicorn has several worker classes (sync, gthread, gevent). For your project's traffic pattern (be honest about what that looks like), which class would you pick and why? What would change your mind?
3. **The WSGI contract.** gunicorn imports your Flask app and calls it. What does it mean for Flask to "be a WSGI app"? Why does this contract matter beyond gunicorn specifically? (If your project later switched to a different WSGI server — or to an ASGI framework — what would change?)
4. **ProxyFix and X-Forwarded-Proto.** When nginx terminates TLS, Flask doesn't know the request came in over HTTPS. What breaks if you don't fix this? What does the ProxyFix middleware do, and why is it a Flask concern not an nginx concern?
5. **Have the LLM security-test your backend layer.** Use the short prompt below as a starting point (fuller version in Study Guide §22). Paste your `gunicorn.conf.py` and your Flask production config (debug flag, secret key handling, ProxyFix setup, error handlers) and have the LLM audit it. Submit the conversation as `llm_probe_backend.md` and write up what came up. Long enough to show what you took away.
  > **Short version of the prompt:** "I'm hardening the Python process layer of a Flask app. Here's our gunicorn.conf.py: [paste]. Here's our Flask production config: [paste]. Audit for production readiness: (a) wrong or absent settings, (b) information-leakage paths (tracebacks, error messages, default endpoints), (c) anything that would behave correctly under flask run but break under gunicorn or vice versa. Tell me what you'd change and why."

---

#### DB / Security role

You own what the public internet sees, what touches the data, and (on a 3-person team) the deploy pipeline. Your hardening surface is the widest, and on a small team it extends through to CI and secrets.

- **Standard 3-person team** — answer all of the questions below. This is the canonical version of the role.
- **4-person team variant** — answer questions 1–6 only. Questions 7–9 are covered by your coordination teammate (see the coordination section at the bottom). The LLM probe (question 6) uses the standalone prompt in Study Guide §23 rather than the combined one in §24, since you're not covering the deploy pipeline.

1. **nginx as request filter.** Most traffic to a public-facing app is bots scanning for `/wp-login.php`, `/.env`, `/admin`, etc. Why is it better for nginx to 404 these than for Flask to? What does your Flask access log look like before and after this change?
2. **Run the script-kiddie attack-path test.** Study Guide §10 has the complete `test_attack_paths.py` and `attack_paths.json` — copy them into your repo, run them against your stack, and report what happened. Did everything pass? If anything failed, what did your team's setup let through?
3. **Talk to your LLM about test strategies.** The provided test uses one specific approach — pytest with a parametrized fixture of known-bad paths. Ask your LLM what other strategies exist for the same problem (scanner integration, fuzzers, behavioral tests, whatever it comes up with). For each strategy it names, get an honest read on what it catches that the provided approach doesn't, and what it misses. Then in your own words, explain: what strategies exist, why the provided approach is what it is, what it doesn't catch, what you'd add if security were higher stakes. Include the conversation as `llm_strategies.md`.
4. **The trust boundary.** Your database is on the docker network, not exposed to the public internet. What does that protect against? What does it not protect against? What's still your responsibility at the database layer even with this in place?
5. **Rate limiting on auth endpoints.** The nginx config can rate-limit specific endpoints. In your project, which endpoints would benefit from rate limiting, and at what rate? What goes wrong if you set the rate too low? Too high?
6. **Have the LLM probe your full security surface.** On a 3-person team your security surface is wide — nginx config + attack paths + deploy pipeline. Use the short combined prompt below as a starting point (fuller version with response-handling guidance in Study Guide §24). Paste your nginx server block, the contents of `attack_paths.json`, AND your release workflow + secrets handling. Ask the LLM to walk the full surface in one conversation: what attacks the path-list misses, what leakage paths exist in the deploy, what a hostile collaborator could do. Submit the conversation as `llm_probe_dbsec.md` and write up what came up.
  **4-person team variant:** use the narrower prompt in Study Guide §23 instead — your conversation stops at the nginx/attack-path surface, since your coordination teammate covers the deploy pipeline separately.
  > **Short version of the combined prompt:** "I'm the security person on a small team hardening a Flask app. Here's our nginx config: [paste]. Here's our attack_paths.json: [paste]. Here's our GitHub Actions release workflow and secrets handling: [paste]. Walk the full security surface: (a) attacks the path-list test won't catch, (b) secrets-leakage paths in the deploy, (c) what a hostile collaborator with commit access could do, (d) what's missing from a production-grade deploy. Tell me what you'd change and why."

**The remaining questions are only for 3-person teams.** On a 4-person team, your coordination teammate answers these instead.

1. **Secrets, in three places.** A token used by your deploy needs to live in three places: where it's generated, where it's stored, and where it's used at runtime. What are those three places for one specific secret your project would need (Docker Hub token, AWS SSH key, OAuth client secret, etc.)? What goes wrong if it leaks into a fourth place?
2. **Tag-driven releases.** A release workflow that triggers on `git push --tags` is different from one that triggers on every push to `main`. What's the argument for tag-driven for a project like yours? What's the argument against?
3. **When CI goes red.** A GitHub Actions job has just failed at the deploy step. Walk through your diagnostic posture — what do you look at first, what do you check next, what do you ask your LLM, and what would you NOT ask the LLM because it needs your repo's context?

---

#### Coordination role (4-person teams only)

If your team is four people, one person takes this dedicated coordination role. You own how this gets shipped and what to do when it doesn't. On a 3-person team this role is folded into DB/security — see questions 7–9 above.

1. **Secrets, in three places.** A token used by your deploy needs to live in three places: where it's generated, where it's stored, and where it's used at runtime. What are those three places for one specific secret your project would need? What goes wrong if it leaks into a fourth place?
2. **Tag-driven releases.** A release workflow that triggers on `git push --tags` is different from one that triggers on every push to `main`. What's the argument for tag-driven for a project like yours? What's the argument against?
3. **When CI goes red.** A GitHub Actions job has just failed at the deploy step. Walk through your diagnostic posture — what do you look at first, what do you check next, what do you ask your LLM, and what would you NOT ask the LLM because it needs your repo's context?
4. **The composition problem.** The frontend person's headers, the backend person's gunicorn config, and the DB/security person's nginx rate limits all have to coexist in one running stack. What's an example of these layers conflicting with each other, and how would you (as the coordination role) catch it before it ships?
5. **Have the LLM security-test your deploy pipeline.** Use the short prompt below as a starting point (fuller version in Study Guide §25). Paste your GitHub Actions release workflow and the relevant parts of your docker-compose. Ask the LLM to audit for secrets leakage paths, what a hostile collaborator could do, and what's missing from a production-grade deploy. Submit the conversation as `llm_probe_coord.md` and write up what came up.
  > **Short version of the prompt:** "I'm hardening the deploy pipeline of a small team project. Here's our GitHub Actions release workflow: [paste]. Here's how we handle secrets and our docker-compose: [paste]. Audit for: (a) secrets-leakage paths, (b) what a hostile collaborator with commit access could do, (c) what's missing from a production-grade deploy. Tell me what you'd change and why."

