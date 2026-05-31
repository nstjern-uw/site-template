# Week 5 Study Guide: Foundations of Web Application Architecture

A companion to the lecture and the Week 5 assignment. Keep it open while you work.

## The one-line summary

**This week is the core principles of web development — at the level you need to design the app yourself, with AI as your aid, not the other way around.**

When AI builds you a Flask app, you should be able to read what it built and say *"yes, that's the route I asked for, and the session line is doing what I expected, and the password hashing is correct."* That's today.

---

## The architecture in one diagram

   BROWSER  ←→  SERVER  ←→  DATABASE

  (frontend)   (your code)  (durable state)

       ↑           ↑

      HTTP        SQL

Every concept this week sits somewhere in this diagram. Get this picture in your head.

---

## The three forms of memory

You'll confuse these. Everyone does. The trick is knowing which one is right for which job.

| Memory | Where it lives | What it holds | How long it lasts |
| :---- | :---- | :---- | :---- |
| **Session** | On the server, in memory or session store | "Who is logged in right now" | Until logout, server restart, or expiry |
| **Cookie** | In the browser, sent on every request | The session ID (a small key) | Until cleared or expired |
| **Database** | On the server, durable | Who Alice IS — her account, history, content | Until manually deleted |

The dance, in one sentence: **cookie identifies the session, session identifies the user, database holds the truth.**

---

## The three engines

Every page load uses three different engines, each in a different place:

1. **Server-side templating** — Python \+ Jinja2 produce an HTML string on the server. By the time the network hands the response off, Jinja2 is *done*. Some people call this "server-side rendering" — same idea, different word.  
2. **Browser HTML/CSS engine** — receives the HTML, parses the DOM, applies CSS, paints pixels. This is what people usually mean when they say a page "rendered."  
3. **Browser JavaScript engine** — runs your `<script>` blocks; can modify the DOM after initial paint.

The most common student confusion is "when does Jinja get rendered?" Answer: **before the network**, on the server. The browser never sees `{{ }}` — it sees the substituted HTML.

---

## REST in three principles

You'll see "RESTful" in every API doc you ever read. These are the three things it means:

| Principle | What it means |
| :---- | :---- |
| **Resources at URLs** | Every "thing" the app knows about lives at a URL: `/users/42`, `/posts/123` |
| **Verbs as actions** | HTTP methods describe what to do: `GET` (read), `POST` (create), `DELETE` (remove) |
| **Stateless** | Each request stands alone; the server has no memory between requests |

The third principle is *why* sessions and cookies exist. HTTP itself has amnesia.

---

## HTTP methods reference

Every HTTP request uses one of seven methods. You'll write GET and POST 95% of the time.

| Method | Purpose | When you'll use it |
| :---- | :---- | :---- |
| **GET** | Read a resource. Shouldn't change server state. Can carry simple inputs in the URL. | Every page load. Address-bar URLs. Links. Search/filter forms. |
| **POST** | Create something or submit data. | Every form submission. Login, register, "create new X." |
| **PUT** | Replace an existing resource. | REST APIs. Rare in HTML forms. |
| **PATCH** | Partially update an existing resource. | REST APIs. Rare in HTML forms. |
| **DELETE** | Remove a resource. | REST APIs. Often modeled as POST in HTML. |
| **HEAD** | Like GET but headers only — no body. | Rarely. |
| **OPTIONS** | Ask the server what methods it supports. | Browsers send these for CORS. |

**GET vs POST in detail:**

|  | GET | POST |
| :---- | :---- | :---- |
| Where inputs go | URL query string: `/search?q=cats&page=2` | Request body |
| Visible in address bar & browser history | Yes | No |
| Cacheable | Yes | No |
| Should be idempotent (safe to retry) | Yes | No |
| Use for | Reads; simple filters/flags; pagination | Creates, submits, anything that changes state |

GET *can* carry data in the URL — that's how search and pagination work — but it's brittle for anything more than simple filters: URLs have length limits (\~2K characters in practice), values need URL-encoding (spaces, special characters), and everything is visible in the address bar, browser history, and server logs. Composing complex queries by hand is error-prone. For anything beyond a few key-value pairs, use POST.

The skeleton's login route (`@app.route("/login", methods=["GET", "POST"])`) handles both: GET returns the form, POST processes it. That's the standard form-handling pattern.

---

## HTTP status codes reference

Every HTTP response starts with a three-digit number. Six you'll see all the time:

| Code | Family | Meaning |
| :---- | :---- | :---- |
| **200 OK** | 2xx success | Worked. Here's the page/data. |
| **302 Found** | 3xx redirect | Look over there instead. Browser follows. |
| **400 Bad Request** | 4xx client error | You sent bad data. |
| **401 Unauthorized** | 4xx client error | You need to log in to see this. |
| **404 Not Found** | 4xx client error | That URL doesn't exist. |
| **500 Server Error** | 5xx server error | Server crashed. Check the logs. |

Memory aid: **2xx \= good, 3xx \= follow, 4xx \= your fault, 5xx \= server's fault.**

302 is what login uses — after a successful POST, the server sends 302 → /, browser follows to the home page. That's the redirect-after-POST pattern.

---

## Inspecting HTTP — three tools (named for now, depth in Week 6\)

When you need to see what's actually on the wire:

| Tool | When to use | What it gives you |
| :---- | :---- | :---- |
| **Browser dev tools** (F12 → Network tab) | Your week 5 tool | Every request the page makes; click for headers, body, status |
| **curl** | Command-line testing | Reproducible. `curl http://localhost:5000/`. Already on EC2. |
| **Postman** | Repeated GUI testing | Save requests in collections. Good for testing your own API as you build it. |

For Week 5, the browser's Network tab is enough. The demo and assignment both use it directly. Week 6 covers curl and Postman in depth.

---

## MVC: it's the pattern from your maze project

You used Model-View-Controller in your maze project last term. The same pattern shows up in Flask:

| Maze project | Flask |
| :---- | :---- |
| Model — game state | Database (SQLModel classes, the `users` table) |
| View — the UI grid | Templates (Jinja2 HTML files) |
| Controller — input handler | Route handlers (`@app.route(...)` functions in `app.py`) |

If you ever feel lost in Flask, ask: **"Is this code a Model, View, or Controller?"** That tells you where it belongs.

---

## The login flow: one feature, every concept

Login is a microcosm of the entire stack. Walking through it once means you've seen the system in motion.

1\.  Browser → GET /login

2\.  Server  → render login.html, return HTTP 200 \+ HTML

3\.  User submits form

4\.  Browser → POST /login   username=alice, password=...

5\.  Server  → SELECT user FROM users WHERE username=alice

6\.  Database → user row \+ password\_hash

7\.  Server  → check\_password() ✓

              session\["user\_id"\] \= 42        ← THE LOAD-BEARING LINE

8\.  Server  → HTTP 302 → /home

              Set-Cookie: session\_id=abc123

9\.  Browser → GET /home   Cookie: session\_id=abc123

10\. Server  → reads cookie, looks up session, finds user\_id=42

              SELECT user FROM users WHERE id=42

              renders home.html with username

              HTTP 200 \+ HTML

Concepts this touches: database, form rendering, validation, sessions, cookies, password hashing, signed cookies, redirects (302), status codes (200/302/...), error messaging, future OAuth (same shape, externalized).

If you understand login, you understand most of what your team will build.

---

## The single most important line

session\["user\_id"\] \= user.id

Read it again. This is what login *does*. Every other line of the login route is supporting machinery — validation, redirects, password hashing. The actual moment of "this user is logged in now" is this one line.

What it does:

1. Server creates a session entry, keyed to a random ID like `abc123`, with `{user_id: 42}` inside.  
2. Server sends `Set-Cookie: session_id=abc123` in the response headers.  
3. Browser stores the cookie, sends it back on every subsequent request.  
4. Next request: server reads cookie, looks up session `abc123`, finds `user_id=42`, knows it's Alice.

**Why signing matters.** Flask signs the cookie with your `SECRET_KEY`. If a user tries to forge `session_id=foo, user_id=1` (the admin), the signature won't match — Flask rejects it.

---

## The frontend: where JavaScript earns its keep

The login form *works* without any JavaScript. But there's friction:

- **Double submission** — user clicks Submit twice in 50ms; two POST requests fire; two database writes happen  
- **Late validation** — bad password? user waits 200ms+ for the server round-trip before seeing the error  
- **No live feedback** — user typing in the username field has no idea it's already taken until they submit

Pure server-side can't fix any of these. By the time the request reaches the server, the friction has already happened. **Anything that needs to happen *now* — not after a server round-trip — lives in JavaScript.**

### The JS sliver in your skeleton

The fix has two parts: a small attribute on the HTML form, and a tiny bit of JavaScript that watches for those forms.

**The HTML side** (already in `templates/login.html` and `register.html`):

\<form method="POST" action="/login" data-disable-on-submit\>

    ...

    \<button type="submit"\>Log in\</button\>

\</form\>

The `data-disable-on-submit` attribute is a custom marker. HTML lets you put any `data-*` attribute on any element; they don't do anything by themselves but JavaScript can find elements by them. Here we use it to opt forms in to the disable-on-submit behavior — login and register want it; some other future form might not.

**The JS side** (in `static/js/forms.js`):

document.querySelectorAll("form\[data-disable-on-submit\]").forEach(form \=\> {

    form.addEventListener("submit", () \=\> {

        const button \= form.querySelector("button\[type='submit'\]");

        button.disabled \= true;

    });

});

Line by line:

- `document.querySelectorAll("form[data-disable-on-submit]")` — search the page for every `<form>` element that has the `data-disable-on-submit` attribute. Returns a list (technically a NodeList).  
- `.forEach(form => { ... })` — for each form found, run this setup code.  
- `form.addEventListener("submit", () => { ... })` — register a callback that fires when the form is submitted. The callback runs in the browser, **before the request leaves**. This is the key timing detail: the JavaScript runs *between* the user's click and the network request. That's the gap the server can't see.  
- `form.querySelector("button[type='submit']")` — find the submit button inside this specific form.  
- `button.disabled = true` — set the button's disabled state. Browsers won't fire click events on a disabled button. The second click is silently ignored. The first request continues normally.

**A natural follow-up question:** *if the button is disabled, when does it become re-enabled?* For our login flow, the answer is: **never on this page.** Login redirects either way — success → `/`, failure → `/login` with a flash error. When the new page loads, the disabled button is gone with the old DOM, and the new page has its own fresh button. The disable-and-forget pattern works exactly because we redirect on every outcome.

This wouldn't work for a form that *doesn't* navigate after submission — for example, an AJAX form that updates part of the page without reloading. There you'd need to re-enable the button explicitly when the response comes back. We don't have that case in the skeleton, but you'll see it in Week 6+ when forms start interacting with APIs without full page navigation.

That's it. Five lines of real work, plus the boilerplate that finds the right forms.

### Are there other ways to fix the timing bug?

Yes, several. Each is right for a different scope.

| Approach | How it works | When it's right |
| :---- | :---- | :---- |
| **Disable the button** (your skeleton) | Client-side. Button-disable in browser. | Single-tab UX friction. Fast, simple, two lines. |
| **Idempotency tokens** | Server gives each form a unique token; rejects duplicate tokens. | Bulletproof. Handles two-tab submits, network retries, anywhere correctness is critical (payments, account creation). |
| **Database UNIQUE constraints** | `username UNIQUE` etc. — second INSERT fails. | Free defense in depth when you can express the rule in the schema. Doesn't help for non-uniqueable side effects (sending money, sending emails). |
| **Rate limiting** | Reject more than N requests/sec from same IP/user. | Wrong granularity for two clicks 50ms apart, but the right tool for spam/abuse. |
| **Framework state (`isSubmitting`)** | React/Vue/Svelte component holds an `isSubmitting` boolean; renders the button as `disabled={isSubmitting}`. | The framework version of the same idea. Idiomatic in modern frontends — that's where Week 6 goes. |

The button-disable fix is right for *this* — a login form where the bug is "user clicked twice." For higher-stakes operations like payment, you'd layer multiple defenses: client-side disable \+ server-side idempotency token \+ database constraint. Each one covers a failure mode the others don't.

---

That's the principle: **server-side for state and durability, client-side for instant feedback**. When you have a lot of client-side code, you organize it with a framework. That's what React is. We'll get there next week.

---

## Stack reference

Your skeleton and your team's project will use this stack:

| Layer | Tool | Why |
| :---- | :---- | :---- |
| Server framework | Flask | Simple, widely used, battle-tested |
| Template engine | Jinja2 (only `{{ var }}` and `{% extends %}`) | Built into Flask; we use the minimum |
| Database | Postgres | Industry standard; concurrency, scalability, security |
| ORM | SQLModel | Modern Python idiom; what you used last term |
| Password hashing | Werkzeug's `generate_password_hash` | Built into Flask; good enough for this course |
| Frontend styling | Bootstrap 5 (via CDN) | Professional UI without writing CSS |
| Frontend interactivity | Vanilla JavaScript | The JS sliver — when client-side is needed |
| Container orchestration | Docker Compose | One command to bring up the app \+ db |
| CI | GitHub Actions workflow \+ pytest | Workflow file ships in skeleton; auto-trigger disabled this week, enabled in Week 6 |

---

## Advanced topics named (but not used)

You'll see these in real-world Flask code. We don't need them this week. Pick them up from docs when your team's project actually requires them.

| Tool | What it does | When you'll need it |
| :---- | :---- | :---- |
| **Full Jinja2** (`{% if %}`, `{% for %}`, filters, macros) | Logic in templates | When templates get repetitive — but try to keep logic in Python instead |
| **Blueprints** | Split routes across files | When `app.py` grows past \~15 routes |
| **Alembic** | Schema migrations | When you need to evolve the schema over time (Week 6+) |
| **Gunicorn** | Production WSGI server | When you deploy for real (Week 8+) |

If you find yourself reaching for these in Week 5, you're over-engineering. The skeleton uses one `app.py`, `create_all()`, and Flask's dev server, and that's intentional.

---

## Docker Compose: the five commands

Compose orchestrates multiple containers as one app. Your skeleton has two services: `app` (Flask) and `db` (Postgres). You'll learn to add a third in Week 6+.

| Command | What it does |
| :---- | :---- |
| `docker compose up` | Start everything. The default. Add `-d` to run detached. |
| `docker compose down` | Stop everything. Add `-v` to also wipe volumes (destroys the database). |
| `docker compose ps` | What's running right now? |
| `docker compose logs -f app` | Tail one service's output. Best debugging tool. |
| `docker compose exec db psql -U app -d app` | Run a command inside a running container. |

The two-container model is intentional: the database is its own thing. It runs separately, restarts separately, and in production usually lives on a different machine. Compose lets you treat them as one app for development.

`docker-compose.yml` is the recipe. Read it like one — each service is one container.

---

## Template idioms: four things to recognize

The Flask templates in your skeleton (login.html, register.html, base.html) use Jinja2. Four constructs cover almost everything you'll see:

| Idiom | What it does |
| :---- | :---- |
| `{{ variable }}` | Substitute a value from Python into the HTML. |
| `{% extends "base.html" %}` | "This template fills in blocks of base.html." Layout sharing. |
| `{% block content %} ... {% endblock %}` | The holes in the parent that children fill. |
| `{% if user %} ... {% endif %}` | Conditional rendering. Used in the navbar to show different links when logged in. |

If you find yourself reaching for `{% for %}` loops or filters or macros, look at the slide on advanced topics — those exist but you don't need them this week. Try to keep the logic in Python instead.

---

## The `S3_content/` pattern

Your S3 site lives at `/site/` in the running app. You populate the `S3_content/` folder by syncing your S3 bucket:

aws s3 sync s3://\<your-bucket\>/ S3\_content/

How it works in the skeleton:

1. The home route (`/`) is Flask-rendered — `render_template("home.html")`. It has a navbar with links to Login, Register, About, and **My Site**.  
2. The `/site/` route calls `send_from_directory(S3_CONTENT, "index.html")` — Flask reads your S3 index and returns it.  
3. The `/site/<path:filename>` route does the same for any other file in `S3_content/` — CSS, JS, images, nested subdirectories.  
4. The Flask routes (`/login`, `/register`, `/about`, `/logout`) are entirely separate from `/site/`. There's no overlap, no priority logic — just different URL spaces.

Why this design: putting your S3 site at `/site/` keeps the Flask navbar always reachable. The home page is *the entry point* — it has navigation, login state, and a clear link to your S3 content. Each student gets a different-looking site at `/site/` because each student has different bucket contents. And it sets up Week 6 — *Flask is the orchestrator, S3 is one thing it serves*; next week your team adds dynamic routes that interact with external APIs.

You can re-run `aws s3 sync` any time you update your S3 bucket.

---

## What "done by Saturday" looks like

For your individual half (7 marks):

- [ ] Skeleton runs on your EC2: `docker compose up -d` works, browser sees the home page at `/`  
- [ ] You ran `aws s3 sync` and clicking "My Site" in the navbar shows your own files served by Flask  
- [ ] You can register an account, log out, log in again, and see your username in the navbar  
- [ ] You uncommented the JS sliver and verified that double-clicking submit no longer fires two requests  
- [ ] You made at least one Bootstrap styling change visible to a TA  
- [ ] Tests pass locally (`docker compose exec app pytest -v` returns all 7 green)  
- [ ] You opened a PR with your changes and merged it

For your team's group half (3 marks):

- [ ] Your team has met and agreed on all 8 About-page sections  
- [ ] Your About page is concrete: real endpoints, real schema, real API choice  
- [ ] Every team member's repo has the same About page content

Then Week 6 starts: your team's coordinator (4-person teams) runs the LLM session that produces `CONTRACTS.md` from your About page. Each role starts implementing against the contracts.

---

## A few common stumbles

**"Docker compose says port 5000 is already in use."** Some other container is running. `docker compose down` to stop, or `docker ps` to see what's running and `docker stop <id>` it.

**"My EC2's IP changed and now nothing works."** Same Week 4 gotcha. Update `<EC2-IP>` everywhere. Easiest workaround: leave your EC2 running 24/7 (Free Tier covers this).

**"`docker compose up` runs but the app can't connect to the database."** The Compose file's healthcheck waits for Postgres to be ready before starting Flask. If you see `psycopg2.OperationalError`, the healthcheck didn't fire correctly — try `docker compose down -v && docker compose up`.

**"My JS sliver isn't working."** Check the browser console for errors. The script is at `static/js/forms.js` and is loaded by `templates/base.html`. Make sure you uncommented the *whole* block (between `/*` and `*/`), and refresh the browser hard (Ctrl+Shift+R) so the new JS loads.

**"Tests pass locally with one DB but might fail with another."** The pytest tests use SQLite in-memory; if you run the app with Postgres locally and write Postgres-specific SQL in tests, those tests will pass under Postgres but not under SQLite. Stick to standard SQL in tests.

**"My S3 site loads but the styling/images/JavaScript don't appear."** Almost always a URL pathing issue. Open your browser's Network tab (F12) and look for 404s — those are the assets that didn't load. The fix is usually one line: open your `S3_content/index.html` and find paths like `<link href="/styles.css">` or `<script src="/lightbox.js">` (with a *leading slash*). When S3 served them, the leading slash was relative to the bucket root. Now Flask serves them under `/site/`, and a leading slash means "relative to the Flask app root" — which is *not* `/site/` — so `/styles.css` ends up trying to hit Flask's root. Easiest fix: drop the leading slashes — `<link href="styles.css">`, `<script src="js/lightbox.js">`. Save, refresh.

---

## When in doubt

Ask: *"is this a Model, View, or Controller?"* That tells you which file to look in.

Ask: *"does this happen on the server or in the browser?"* That tells you whether it's Python or JavaScript.

Ask: *"is this short-term, durable, or in-the-cookie?"* That tells you which form of memory to use.

Three questions. Answer those and most Week 5 confusion clears up.  
