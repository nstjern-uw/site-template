# Week 6 Study Guide

This guide is shorter than Week 5's — most of the stack was introduced earlier. Focus this week is on **contracts and testing**: the concepts that make Week 6 hard, and the tools that show up in your team's repo and the lecture slides.

If you're tight on time, jump to the [Start here if you're behind](#start-here-if-youre-behind) section at the bottom.

---

## Concepts (the load-bearing reading)

The lecture spent most of its time on these. The deck shows them in motion; the readings below help when something doesn't quite click.

### Contracts as a software engineering pattern

Your team's `CONTRACTS.md` is a contract in the formal SE sense: a written agreement about an interface, enforced by tests. The pattern predates Week 6 by decades — it goes by names like "design by contract" (Bertrand Meyer's term, 1986), "consumer-driven contracts" (modern microservices), or just "interface contracts."

The core idea: **before two components communicate, agree on the shape of the communication, then verify that agreement with tests.** Both sides write to the agreement; neither has to know how the other side is implemented. That's the "black box" thinking the deck talks about — each role's slice is a black box to the others, and the contract is what makes them composable.

**For depth (15-20 min reads):**

- Martin Fowler, [Integration Contract Tests](https://martinfowler.com/bliki/IntegrationContractTest.html) — short, practical, explains the same idea your `CONTRACTS.md` implements at smaller scale.  
- Wikipedia, [Design by contract](https://en.wikipedia.org/wiki/Design_by_contract) — covers the historical term and its philosophical underpinning. Skip the formal-methods sections; the conceptual overview is the useful part.  
- Pact's [Consumer-driven contracts overview](https://docs.pact.io/) — Pact is a tool that does at scale what your team does manually with `CONTRACTS.md`. Good for understanding why this pattern exists in production microservices.

### TDD review (test-driven development)

You've heard the term in earlier weeks. This week it operates: Maya commits failing tests *before* anyone writes implementation, and the failing tests *are* the deliverable that makes the contract executable.

The strict TDD loop is "red → green → refactor": write a failing test, write the minimum code to pass it, then improve the code without changing behavior. Week 6 doesn't ask you to follow that loop literally inside your slice — the tests are pre-committed, you just make them pass — but the **principle** that tests come first is exactly what this week practices at the team level.

**For review:**

- Wikipedia, [Test-driven development](https://en.wikipedia.org/wiki/Test-driven_development) — covers the history and the canonical loop.  
- Kent Beck, *Test-Driven Development by Example* (2002) — the foundational book if you want depth. Chapter 1 alone is enough to get the discipline.

### Truthy fixtures and synthetic-vs-real testing

This is the lesson the live demo was built around. Naming it explicitly so it sticks:

**Truthy fixtures** are synthetic test data that *look* right — fields plausible, shape coherent, test passes — but aren't *verified* against what the real system actually returns. The mock matches what you imagined the API does; the real API might do something different. Tests pass forever; production breaks the first time it talks to the real upstream.

This is **the most common AI testing failure mode.** When you ask an AI to write a test fixture, it generates one from training-data plausibility, not from a real call to the actual service. The fixture is right enough to pass a code review and wrong enough to break in production.

The fix is **end-to-end tests against real services**. Unit tests check your code; e2e tests check whether your assumptions about the world are correct. Both are required. Neither substitutes for the other.

**For depth:**

- Martin Fowler, [Test Double](https://martinfowler.com/bliki/TestDouble.html) — taxonomy of mocks, stubs, fakes, and spies. Useful vocabulary for thinking about what kind of synthetic data you're using and what it doesn't catch.  
- Martin Fowler, [Mocks Aren't Stubs](https://martinfowler.com/articles/mocksArentStubs.html) — the canonical essay on what mocks do and don't verify. Long but worth it; it's the deepest treatment of the truthy-fixtures problem you'll find.  
- Gerald Weinberg, *Perfect Software and Other Illusions about Testing* (2008) — older but still excellent on why tests give false confidence. Out of print; PDFs are findable.

### SDD (specification-driven development) review

You've heard this term too. It's TDD's older cousin: the spec comes before the test, and the test verifies the spec. Your `CONTRACTS.md` is the spec; your test files are TDD applied to that spec.

Most modern web work is SDD-then-TDD: write the contract first (spec), write tests against the contract (TDD), implement to pass the tests. Week 6 makes this sequence explicit at the team level — Maya does the spec work; everyone else does the TDD work.

**For review:**

- Wikipedia, [Specification by example](https://en.wikipedia.org/wiki/Specification_by_example) — closest formal name for what your team is doing.  
- Gojko Adzic, *Specification by Example* (2011) — book-length treatment if you want depth; chapters 1-3 are the essential ones.

### Black-box / interface design

Each role this week works in a black box. Server-side knows the schema (Jamal's slice produces models with these fields), but doesn't know how the migrations were written. Client-side knows the route response shape (Devon's slice returns this JSON), but doesn't know what request library Devon used or what error envelope wraps Nominatim. The contract is what makes black-box composition work.

This is the principle that scales from your team to production microservices to operating systems. **You couple to interfaces, not implementations.** The whole reason `CONTRACTS.md` is the artifact, not "implementation\_guide.md," is that interfaces are stable and implementations change.

**For depth:**

- David Parnas, [On the Criteria To Be Used in Decomposing Systems into Modules](https://www.win.tue.nl/~wstomv/edu/2ip30/references/criteria_for_modularization.pdf) (1972) — the foundational paper. Twelve pages; the second half is what's worth reading. Has aged better than almost any paper of its era.  
- Wikipedia, [Information hiding](https://en.wikipedia.org/wiki/Information_hiding) — the modern restatement of Parnas's principle.

---

## Tools used in code or slides

These are the named libraries and platforms that show up in this week's GitHub repo and in the lecture deck. None of them is taught in depth — the goal here is naming and one-line orientation, with a link if you want more.

### BeautifulSoup

Python library for parsing HTML. Originated as a web-scraping tool — back when most data on the web didn't have an API, you scraped HTML pages instead. Still useful when an API is missing, deprecated, or doesn't expose what you need.

This week the test files use it for a different purpose: parsing the HTML our own Flask app returns, to verify structure in tests. `soup.find("form", attrs={"action": "/cafes/1/rate"})` finds the rate form; `assert form is not None` checks it's there. You don't write BeautifulSoup yourself this week — the test files committed by the coordinator already use it.

**Read more:**

- [Official documentation](https://www.crummy.com/software/BeautifulSoup/bs4/doc/) — the "Quick Start" section is 5 minutes and covers what you need for testing.

### responses (request-mocking library)

Python library that intercepts `requests.get`/`requests.post` calls during tests and returns predetermined responses instead of hitting the real network. Used in `tests/test_cafe_routes.py` to mock Nominatim — tests run offline, deterministically, in any environment.

The pattern: `responses.add(responses.GET, url, json={...})` registers a mock; the wrapped test function then calls Flask routes that internally use `requests`, and `responses` intercepts the outbound HTTP and returns the mock instead.

**Caveat that the live demo demonstrated:** mocked responses are *plausible*, not *verified*. See the truthy-fixtures section above.

**Read more:**

- [responses on PyPI](https://github.com/getsentry/responses) — the README shows the API in 5 minutes.

### Flask-Login

Authentication framework for Flask. Replaces the raw `session["user_id"]` pattern from Week 5 with a structured user-loader, `current_user` global, `@login_required` decorator, and `login_user`/`logout_user` functions.

This week's auth refactor is the db-and-security role's main job. The lecture slides walk the actual diff; the agentic build's commit 2 is the worked example.

**Read more:**

- [Official Flask-Login documentation](https://flask-login.readthedocs.io/) — the "How it Works" and "Your User Class" sections are what you need; the rest is reference material.

### requests (with timeouts and headers)

Python's HTTP-client library. You've seen it before; this week the contract specifies *how* to use it correctly:

- **`timeout=5`** — without a timeout, your route hangs if Nominatim is slow. The default of "no timeout" is one of the most common production bugs in Python.  
- \*\*`headers={"User-Agent": "..."}` \*\* — Nominatim requires this by their usage policy. Many APIs require it; a few will silently rate-limit you to zero if you don't send one.  
- **Catching specific exceptions** — `requests.Timeout` separately from `requests.RequestException`, plus checking `response.status_code == 429` for rate-limiting. The contract specifies an error envelope with three error codes; each maps to a different exception/status.

**Read more:**

- [requests "Quickstart"](https://requests.readthedocs.io/en/latest/user/quickstart/) — re-reading the timeouts and exceptions sections is worth 10 minutes.

### pytest fixtures and conftest

`tests/conftest.py` defines fixtures (test setup helpers) that pytest automatically wires into test functions by name. Week 6's tests use a `client` fixture for an unauthenticated Flask test client and a `logged_in_client` fixture for an authenticated one.

You don't write fixtures yourself unless your role's tests need new ones. The conftest you start with is sufficient.

**Read more:**

- [pytest fixtures](https://docs.pytest.org/en/stable/explanation/fixtures.html) — the conceptual overview is enough; deep customization isn't needed for Week 6\.

### SQLModel JSON columns and `__table_args__`

Two specific SQLModel patterns the contract requires:

- **JSON columns** — for the `tags` field on the `Rating` model. Pattern: `tags: list[str] = Field(sa_column=Column(JSON, nullable=False, default=list))`. SQLModel doesn't have a built-in shortcut; you drop down to SQLAlchemy's `Column(JSON)`.  
- **`__table_args__` UniqueConstraint** — for the (`user_id`, `cafe_id`) uniqueness on `Rating`. Pattern: `__table_args__ = (UniqueConstraint("user_id", "cafe_id", name="uq_ratings_user_cafe"),)`. This enforces "one rating per user per cafe" at the database level, not just the ORM level.

Both are demonstrated in the agentic build's commit 1 (`02cf46f`). Read that commit's diff if the patterns aren't clear.

**Read more:**

- [SQLModel documentation](https://sqlmodel.tiangolo.com/) — covers the basics; for `Column(JSON)` and `__table_args__` you'll fall back to [SQLAlchemy's docs](https://docs.sqlalchemy.org/en/20/core/constraints.html#unique-constraint).

### GitHub Actions and CI/CD

Your repo's `.github/workflows/test.yml` triggers automated test runs on every push and pull request. When you open a PR, the workflow runs, and the result (green check or red X) appears next to the merge button.

This is the broader pattern of **CI/CD** — Continuous Integration (every change runs automated tests) plus Continuous Deployment (passing changes get deployed automatically). Week 6 uses CI; Week 7+ may add CD.

**Branch protection** is the gate that makes CI matter: GitHub refuses to merge a PR if its workflow run failed. Without branch protection, CI is informational. With it, CI is enforcement.

#### Manually triggering a workflow

You don't have to push a commit to run your workflow. Add `workflow_dispatch:` to the workflow's `on:` block:

on:

  push:

    branches: \[main, master\]

  pull\_request:

    branches: \[main, master\]

  workflow\_dispatch:

After you push that change, the Actions tab shows a **"Run workflow"** button on that workflow's page. Click it, pick a branch from the dropdown, and the workflow runs as if a real event triggered it. Useful when you want to test a workflow change without making a fake commit, or when you want to re-run on a feature branch without merging it first.

#### Debugging a failing workflow

Three moves worth knowing:

- **Re-run failed jobs.** From any past run's page, click the "Re-run jobs" dropdown — you can re-run all jobs or just the failed ones. Useful for genuinely flaky tests where you want to retry without changing anything.  
    
- **Re-run with debug logging.** Same dropdown has a "Enable debug logging" checkbox. The re-run produces vastly more verbose output — every step shows the runner's internal `STEP_DEBUG` and `ACTIONS_RUNNER_DEBUG` info. Use this when the normal log is too terse to diagnose what went wrong.  
    
- **The `gh` CLI.** GitHub's command-line tool gives you the same controls from your terminal:  
    
  gh workflow list                           \# see all workflows  
    
  gh workflow run test.yml                   \# trigger a manual run  
    
  gh workflow run test.yml \--ref my-branch   \# trigger on a specific branch  
    
  gh run list \--workflow=test.yml            \# see recent runs  
    
  gh run view \<run-id\> \--log-failed          \# show only the failed step's log  
    
  gh run rerun \<run-id\>                      \# re-run a failed run  
    
  `gh run view --log-failed` is the killer command — it skips ahead to whatever step actually broke without you scrolling through hundreds of lines of successful setup output.

#### Common CI failures and what they mean

- **Workflow doesn't appear in the Actions tab at all.** Either the YAML has a syntax error (GitHub silently fails to register malformed workflows), the workflow's `on:` block doesn't include any event that has occurred yet on this repo (try pushing a commit or adding `workflow_dispatch`), or Actions are disabled in repo Settings → Actions → General.  
- **Workflow runs but fails immediately on `setup-python` or similar action.** Usually a version mismatch — the action version pinned in the YAML may not exist, or the Python version requested isn't available on the current runner image. Check the action's GitHub page for the latest version, and check [GitHub-hosted runner images](https://github.com/actions/runner-images) for what's pre-installed.  
- **Tests pass locally but fail in CI.** Environment difference. Your machine has packages or system dependencies the CI runner doesn't. Common culprits: missing `apt-get` packages, missing `pip` packages not in `requirements.txt`, environment variables your local shell sets but CI doesn't.  
- **Workflow logs are cut off or truncated.** The full log is usually still available — click the gear icon on the run's page and select "View raw logs." For very long logs, GitHub also offers a download.

**Read more:**

- [GitHub Actions documentation, "About workflows"](https://docs.github.com/en/actions/using-workflows/about-workflows) — 10 minutes; covers the YAML structure your `test.yml` uses.  
- [Branch protection rules](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches) — the settings page in your repo configures these; the doc explains what each option does.  
- [Manually running a workflow](https://docs.github.com/en/actions/using-workflows/manually-running-a-workflow) — the official doc on `workflow_dispatch`.  
- [Enabling debug logging](https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/enabling-debug-logging) — when the normal log isn't enough.  
- [GitHub CLI manual: `gh workflow`](https://cli.github.com/manual/gh_workflow) — full CLI reference.  
- For broader CI/CD context, [Martin Fowler on Continuous Integration](https://martinfowler.com/articles/continuousIntegration.html) — long but the canonical introduction.

### Bootstrap (continued use)

You've used Bootstrap in earlier weeks. This week's templates use a few more components: cards in a grid for cafe lists, form-control classes for the rating form, badges for tag displays, alerts for flash messages.

Nothing new to learn — keep using the [Bootstrap 5 documentation](https://getbootstrap.com/docs/5.3/) when you need a class you haven't seen.

---

## Forward-pointers (brief)

Two things this week's code touches that get treated properly later:

### Playwright (Week 7\)

BeautifulSoup tests verify *what the server sends*. They run inside pytest, fast and deterministic, but they cannot verify *what the user experiences* — JavaScript execution, full page renders, multi-step user flows, redirects with cookies.

Week 7 introduces **Playwright** — a tool that drives a real browser (Chromium by default) so tests can exercise actual user flows. We need it because Week 7's OAuth login flow can't be tested any other way; you can't BeautifulSoup your way through "click the GitHub login button, redirect to GitHub, come back authenticated."

Don't read up on Playwright now — Week 7 covers it from scratch. This is just a heads-up that the BeautifulSoup tests this week aren't the whole testing story.

### Continuous Deployment (Week 8+)

Your CI/CD this week is just the CI half — tests run on every push. Continuous Deployment adds the second half: passing changes get deployed automatically, usually to a staging environment first and then production. We'll wire this up in a later week.

For now: GitHub Actions is the runner, your `test.yml` is the workflow, the PR check is the gate. The "deployment" part is manual.

---

## The truthy-fixtures lesson (from the live demo)

If you missed any of the live debugging demo, or want it written down: the demo walked through a real bug surfaced live during preparation.

**The symptom.** Searching the cafe finder for "Vancouver, BC" returned nothing useful — either an empty result or the city of Vancouver presented as a single "cafe." The contract's section 7 (the demo script) said this query should return cafes near Vancouver. The behavior didn't match the contract.

**The diagnosis.** All 24 unit tests passed. The mid-build review passed. By every signal that's supposed to indicate "the build is complete," this build was complete. But the user-visible behavior was broken.

The diagnosis traced back to a *contract gap*. CONTRACTS.md said the server should "for each result that's a cafe (Nominatim category check), insert into cafes table." That sentence presupposed Nominatim returns *mixed results* (some cafes, some other things) for a search to filter. That presupposition is wrong about how Nominatim's `/search` endpoint works. Nominatim is a *geocoder* — it returns whatever the query string names. Querying "Vancouver, BC" returns the city, not cafes near the city.

**Why no test caught it.** The test fixtures mocked Nominatim with cafe-shaped data — a textbook *truthy fixtures* failure. The mock was plausible (right fields, right structure) but unverified against what real Nominatim returns. The mock's existence let the bug propagate from contract to implementation to deploy without anyone confronting the wrong assumption.

**The fix.** Three coordinated changes: (1) revise CONTRACTS.md to specify query construction (`cafes near {user_input}` to trigger Nominatim's special-phrase handling) and a class/type filter on results as a safety belt; (2) add a test fixture with mixed cafe \+ non-cafe data, asserting only the cafe is inserted; (3) patch the implementation to construct the query and filter results. All three landed together as a single commit, with the contract as the source-of-truth that drove the test and code.

**The lesson.** Synthetic test fixtures are *plausible*, not *verified*. Real upstream services return shapes your fixtures didn't anticipate. This is the most common AI testing failure mode, and the only thing that catches it is **end-to-end tests against real services** — the kind specified in your assignment's Part 2 (per-role e2e walks) and Part 3 (whole-system e2e walk in `e2e.md`).

If your team's e2e walk surfaces a contract gap and you fix it at the source — revising the contract, updating tests, applying a code patch — that's the assignment done well, not a sign of failure.

---

## Start here if you're behind

If you're opening this study guide late and panicking, read the following sections in order. Each is 5-10 minutes. Total: \~30 minutes.

1. [**Contracts as a software engineering pattern**](#contracts-as-a-software-engineering-pattern) — what `CONTRACTS.md` is and why it's the load-bearing artifact this week.  
2. [**Truthy fixtures and synthetic-vs-real testing**](#truthy-fixtures-and-synthetic-vs-real-testing) — the lesson the live demo was built around. Not optional; the assignment's Part 2 and Part 3 grading criteria assume you've absorbed this.  
3. [**The truthy-fixtures lesson (from the live demo)**](#the-truthy-fixtures-lesson-from-the-live-demo) — the concrete walkthrough of how the failure mode operates, in case the abstract version didn't land.  
4. Skim the **Tools** section — you don't need to read every entry, just confirm you recognize each name. If a name isn't familiar, read the 2-sentence description and skip the link.  
5. Re-read your role's assignment guidance (in `week6_assignment.md` Part 2\) one more time. Decide what your e2e walk for your slice will look like.

After that, you're functional. The rest of the study guide is depth available when you have time.

---

## What I'm not covering

To be honest about scope:

- **Docker Compose, Postgres, Flask routing fundamentals** — covered in Weeks 4-5 study guides. Refer back if needed.  
- **OAuth, Playwright, advanced CI/CD** — Week 7+ topics; deliberately not pre-loaded here.  
- **Production-scale concerns** like horizontal scaling, observability, secret management — out of scope for the course; relevant after you ship things.

If something in the assignment feels like it requires knowledge that's not covered here or in earlier weeks, that's a signal something's missing — let me know during office hours.  
