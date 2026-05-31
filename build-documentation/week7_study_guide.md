# Week 7 Study Guide — OAuth and browser-driven testing

This is the reference document for the Week 7 lecture. It goes deeper than the slides on the conceptual pieces and explains the "why" behind the patterns. The slides are the lecture's spine; this is its body. Read it after lecture (or alongside the recording) to solidify what was covered.

---

## 1\. Why OAuth, and where it sits

### The password problem

Three things make rolling your own username-and-password auth a bad default for a small project:

- **Storing password hashes is a liability.** Hashes need a good algorithm (bcrypt / argon2), a unique salt per user, parameters tuned to your hardware, periodic rehashing as compute power grows, and rotation when an algorithm is deprecated. None of that is hard, but all of it is *yours to maintain* once you store hashes.  
- **Users reuse passwords.** A breach on someone else's site puts your users at risk on your site. You inherit the security posture of every site your users have ever used the same password on.  
- **You are probably not in the identity business.** GitHub, Google, Apple, and Microsoft have thousands of engineer-years of investment in keeping account security tight. Your project doesn't need to compete with them on this dimension.

OAuth replaces "*you store hashes and verify passwords*" with "*you trust a verdict from a provider*."

### OAuth in one sentence

A user proves their identity to a provider, who then tells your app *"yes, this is them"* — without your app ever seeing the password.

The rest is mechanics: how the message travels (the Authorization Code flow), what's in it (an access token and a way to look up user identity), and how you trust it (cryptographic signing of the redirect parameters).

### OAuth2 vs OIDC — the broader landscape

You'll hear both terms; they're not interchangeable.

- **OAuth2** is an *authorization* framework: "this app can access resource X on behalf of user Y." It gives you an access token. It doesn't tell you *who* user Y is — you have to ask the resource server (e.g., GitHub's `/user` endpoint) with the token.  
- **OIDC (OpenID Connect)** is OAuth2 plus a *standardized identity layer*. It returns an `id_token` (a signed JWT) containing standardized claims about the user — `sub` (subject), `email`, `name`, `picture`, etc. Identity is part of the protocol, not inferred from a side call.

Real-world: "Sign in with Google", "Sign in with Microsoft", and "Sign in with Apple" all use OIDC. **GitHub OAuth Apps use OAuth2 alone** — there's no OIDC layer, so you call `/user` after token exchange to get the identity. This assignment uses GitHub, so you're using OAuth2.

The technique you're learning generalizes to any OAuth2 provider; OIDC is the same dance with the identity step standardized instead of being a separate API call.

### The 2FA analogy (the right intuition)

A useful entry-point analogy: OAuth is "the Duo part without the password." When you complete a push-2FA approval on your phone, you're being asked to confirm something on a trusted second device before being granted access. OAuth is that step on its own, as the whole login — you click "Authorize" at the trusted third party, and your app trusts the verdict.

The 2FA analogy captures the *shape* of the dance but not its *purpose*: 2FA strengthens authentication at a single site; OAuth delegates authentication to a different site entirely. A better mental model for the purpose is **the passport at customs**: the customs officer doesn't know you, but they trust your passport because it was issued by a trusted authority. The authorization code is the passport stamp; the scope is the visa type.

---

## 2\. The Authorization Code flow

The flow you're implementing has five visible steps and one invisible-but-critical step.

### The five visible steps

1. **User clicks "Sign in with GitHub"** on your app's login page.  
2. **Your server redirects the user's browser to GitHub** at `https://github.com/login/oauth/authorize?client_id=...&state=...&redirect_uri=...`.  
3. **The user authenticates on GitHub** (entering their password, completing 2FA, etc. — none of which your app sees). GitHub asks the user *"do you want to give \[Your App\] access to X?"* and the user clicks Authorize.  
4. **GitHub redirects the user's browser back** to your app's callback URL: `https://your-app.com/auth/github/callback?code=...&state=...`.  
5. **Your server exchanges the code for an access token** by making a server-to-server request to `https://github.com/login/oauth/access_token`, sending the code, your `client_id`, and (importantly) your `client_secret`. GitHub returns the access token.

After step 5, your server has an access token. It uses that token to call `/user` and get the user's identity (id, login, email, name). At that point you can do the local lookup — find the OAuthIdentity row by `(provider, provider_user_id)`, create a new User if needed, and call `login_user(user)`.

### Why the token exchange is server-to-server

A simpler-looking design would be "GitHub redirects the user back with the access token in the URL." That design existed (it's called the *implicit flow*) and it's deprecated for new applications because:

- The token would appear in the user's browser history, in HTTP referrer headers, and potentially in browser extensions.  
- Any JavaScript on the callback page could read the token from `window.location`.  
- The token couldn't be used to identify the originating app cryptographically (no `client_secret` involved).

The Authorization Code flow solves all three: the *code* is in the URL (a short-lived, single-use ticket), but the *token* is exchanged via a server-to-server HTTPS POST authenticated with `client_secret`. The token never touches the user's browser.

### The state parameter (don't disable it)

When Authlib's `authorize_redirect()` runs in step 2, it generates a random `state` value, stores it in the user's session, and includes it in the redirect URL. When the callback fires in step 4, Authlib verifies that the returned `state` matches the one in the session. If it doesn't, the request is rejected.

This prevents *login CSRF* — an attack where an attacker tricks the user's browser into completing a login as the attacker's identity. The state parameter is one of those "the library does it for you, don't disable it" things. Don't disable it.

### What you receive after the token exchange

{

  "access\_token": "gho\_a1b2c3...",

  "token\_type": "bearer",

  "scope": "read:user user:email"

}

This is the access token. It's an opaque string — you don't parse it; you just present it to GitHub's API on subsequent requests. Don't put it in cookies, don't expose it to JavaScript, don't log it.

To get the user identity, you make a separate API call:

info \= oauth.github.get("user", token=token).json()

\# returns {"id": 4501872, "login": "alice", "email": "...", "name": "..."}

---

## 3\. The lookup contract: which field is the key?

This is the section that costs you the most if you get it wrong.

### The mutability problem

Tempting design: use the username (or email) as the foreign key to look up local users from OAuth identities. *Don't.*

- **Usernames are mutable.** GitHub lets users rename themselves; so does Discord; so does almost every modern platform. If you keyed on `login`, a user who renames themselves can lose access to their account on your app.  
- **Emails are mutable.** Users change them, sometimes lose access to them, sometimes have multiple. They're also not always verified — anyone can put any email in a GitHub profile.  
- **`provider_user_id` is the only stable identifier the provider guarantees.** GitHub's `id` field is an integer assigned at user creation, never reused even if the account is deleted. Google's `sub` claim works the same way. This is what you key on.

### The trust statement

The pattern in one line: **`provider_user_id` is signed by the OAuth flow — it can't be forged. Everything else is user-mutable; store it as a snapshot, never as a key.**

Why "signed by the OAuth flow": the access token you exchanged for in step 5 of the Authorization Code flow is authenticated by your `client_secret`. When you use that token to call `/user`, the `id` field in the response comes from GitHub's authoritative database. The only way an attacker can put a wrong `id` in front of your server is if they've compromised GitHub itself — at which point you have larger problems.

### The data model

Two tables: `User` (your app's notion of a person) and `OAuthIdentity` (the join row between a provider's user and your User).

class User(SQLModel, table=True):

    id: int | None \= Field(primary\_key=True)

    email: str | None \= None

    display\_name: str | None \= None

class OAuthIdentity(SQLModel, table=True):

    id: int | None \= Field(primary\_key=True)

    user\_id: int \= Field(foreign\_key="user.id")

    provider: str          \# "github", "google", ...

    provider\_user\_id: str  \# the id from the provider — stable

    created\_at: datetime

Note: `User` has no password column. There's nothing to verify, because authentication happens at the provider.

Why a separate `OAuthIdentity` table rather than putting `github_id` directly on `User`:

- **Multi-provider.** One person might log in with GitHub today and Google tomorrow. They're the same User, two OAuthIdentity rows pointing at them.  
- **Stability.** A returning OAuth user is one row lookup: `WHERE provider='github' AND provider_user_id='4501872'`. Found → existing User; not found → create both rows.  
- **History.** If you ever need to audit "when did Alice link her Google account?", the OAuthIdentity row has the answer; the User row doesn't.

### The find-or-create logic

identity \= db.exec(

    select(OAuthIdentity).where(

        OAuthIdentity.provider \== "github",

        OAuthIdentity.provider\_user\_id \== str(info\["id"\]),

    )

).first()

if identity:

    user \= db.get(User, identity.user\_id)

    \# optionally refresh display fields:

    \# user.email \= info.get("email"); user.display\_name \= info.get("name")

else:

    user \= User(email=info.get("email"), display\_name=info.get("name"))

    db.add(user); db.commit(); db.refresh(user)

    db.add(OAuthIdentity(

        user\_id=user.id,

        provider="github",

        provider\_user\_id=str(info\["id"\]),

    ))

    db.commit()

login\_user(user)

If you only remember one thing from this section: **`provider_user_id` is the key; `login`, `email`, `name` are display fields refreshed each login.**

### Multi-provider linking (briefly)

If a returning user signs in with GitHub today and Google tomorrow, are they the same User or two? That's a policy decision your app makes:

- **Strict separation:** each `(provider, provider_user_id)` is its own User. The same person has two accounts on your app.  
- **Email-based auto-linking:** if a new provider login's email matches an existing User's email, link them. *Only safe if both emails are verified by their providers* — otherwise it's an account-takeover vector.  
- **Explicit linking:** let the user explicitly connect providers from their settings page. Most secure; standard pattern for production apps.

The Week 7 assignment defers this — each new `(provider, provider_user_id)` is its own User. Week 9+ exercises explore it.

---

## 4\. Login replaced, session preserved (Flask-Login)

The OAuth flow ends with `login_user(user)`. That single line is where OAuth hands off to Flask-Login, and Flask-Login handles everything after.

### Two phases, two roles

**Phase 1 — at the end of the callback**: `login_user(user)` puts `user.id` into a signed cookie called the session cookie. The cookie's value is HMAC-signed with `SECRET_KEY`, so the server can detect tampering. After this point, the browser carries the session cookie on every request to your app.

**Phase 2 — on every subsequent authenticated request**: the browser sends the session cookie; Flask-Login extracts `user.id` from it; Flask-Login calls the `@user_loader` you registered, which fetches the actual `User` row from the database; the result is made available as `current_user` in your views; `@login_required` decorators check that `current_user` is authenticated and let the request through.

@login\_manager.user\_loader

def load\_user(user\_id: str) \-\> User | None:

    return db.get(User, int(user\_id))

This one-function registration is what makes the cookie-to-user lookup work.

### The key insight

**The session cookie is the authenticated credential.** After OAuth completes, there's no password to re-check on each request — just the cookie. Whoever can read or forge that cookie can act as the user.

This is the bridge to the hardening section: the entire purpose of cookie flags, session lifetime, and CSRF protection is to keep that cookie safe and unforgeable.

### Why this matters: same `login_user`, two sources

Flask-Login doesn't know or care how the user authenticated. If you wrote a traditional password form, it would call `login_user(user)` after hash verification. With OAuth, you call `login_user(user)` after the callback create-or-link. Same call, same session machinery. OAuth replaces only the *authentication step*; Flask-Login keeps owning the *session lifecycle*.

---

## 5\. Secrets and the minimum security kit

Three things must be in place before OAuth can work safely. The lecture's "Flask security: minimum kit" slide names them; this section explains why each one earns its place.

### The three concerns

- **Secrets in `.env`.** `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET`, and `SECRET_KEY` all live in a `.env` file that is gitignored. They're loaded by `python-dotenv` at app startup. A `.env.example` is committed to document the variable names without real values.  
- **`SECRET_KEY`.** Flask uses this to sign session cookies. After OAuth, your user identity lives in the session — a weak or shared `SECRET_KEY` means an attacker who knows it can forge session cookies for any user.  
- **CSRF protection.** Flask-WTF tokens every state-changing form against a per-session value. Without it, an attacker can submit forms on a logged-in user's behalf. Detailed in section 7\.

### The `.env` pattern

`.env` (gitignored, never committed):

OAUTH\_CLIENT\_ID=Ov23li...

OAUTH\_CLIENT\_SECRET=ghp\_a...

SECRET\_KEY=\<32 random hex bytes — e.g. python \-c 'import secrets; print(secrets.token\_hex(32))'\>

DATABASE\_URL=postgresql://...

`.env.example` (committed, documents the required variables):

OAUTH\_CLIENT\_ID=your-github-id

OAUTH\_CLIENT\_SECRET=your-github-secret

SECRET\_KEY=random-hex-string

DATABASE\_URL=postgresql://...

Loader at the top of `app.py`:

from dotenv import load\_dotenv

load\_dotenv()        \# MUST come before any os.environ lookups

import os

app.config\["SECRET\_KEY"\] \= os.environ\["SECRET\_KEY"\]

### Two patterns worth internalizing

**`os.environ[KEY]` not `os.environ.get(KEY)`.** Square brackets raise `KeyError` if the variable is missing. `.get()` returns `None` silently. A missing secret should crash your app on startup with a clear error, not turn into a confusing runtime error two minutes later when something tries to use `app.config["SECRET_KEY"] = None`.

**`load_dotenv()` before any `os.environ` access.** Order matters. If you import a module that reads `os.environ["X"]` at import time, and you call `load_dotenv()` after that import, the module will see no `X` in the environment.

### What this doesn't cover

Anyone with shell access to your machine, your CI system, or your production server can read `.env`. Secrets management at scale uses secret stores (AWS Secrets Manager, HashiCorp Vault, GitHub Actions secrets, etc.). `.env` is appropriate for development and small deployments. Week 8 covers handing secrets to a deployed container.

---

## 6\. Session hardening: cookie flags and lifetime

The OAuth flow ends with `login_user(user)`. From that line on, the session cookie is what authenticates every subsequent request (section 4). Flask's default session settings are tuned for development convenience, not production safety. The lecture's hardening section is the small set of configuration changes that turn the default into something safe to ship.

Three areas:

1. **Cookie flags** — what the browser can and can't do with the session cookie  
2. **Session lifetime** — how long the cookie stays valid, how "remember me" extends it  
3. **CSRF tokens** — covered in depth in section 7

This section covers the first two; section 7 covers CSRF.

### Cookie flags (three Flask config lines)

A cookie is just a string the browser stores and sends back with future requests to the same domain. Three flags constrain that behavior, and all three are one-line Flask config settings.

**`SESSION_COOKIE_SECURE = True`** (in production only). The browser refuses to send the cookie over plain HTTP — only HTTPS connections will carry it. This prevents the cookie from being intercepted on the wire by anyone on the same Wi-Fi or in the network path. In development you set this to `False` because `localhost` doesn't have HTTPS; in production it must be `True`, or you've leaked the session to anyone watching the traffic.

*Debug gotcha:* setting `SECURE=True` on localhost (which is HTTP) causes a **silent login loop** — the cookie gets set, the browser refuses to send it back over HTTP, your app sees no session, redirects to login, sets the cookie again, repeat. No browser warning. Use env-driven config so this flag is `False` in dev and `True` in prod.

**`SESSION_COOKIE_HTTPONLY = True`** (always). The browser hides the cookie from JavaScript — `document.cookie` won't return it. This means that an XSS bug on your page (an attacker injecting a `<script>` tag through unescaped user content) can't read the session and steal it. The cookie is still sent with requests automatically; JavaScript just can't see it. XSS is a separate problem you should also avoid having, but this flag is your second line of defense.

**`SESSION_COOKIE_SAMESITE = 'Lax'`** (always). The browser won't include the cookie on cross-site `POST` requests, which is what CSRF attacks rely on (see section 7). It *will* include it on top-level GET navigations like clicking a link to your site, which is what you usually want. The two other valid values: `'Strict'` is more restrictive — even GET navigations from another site don't carry the cookie, which breaks "click this email link and you're already logged in" flows — and `'None'` means no cross-site protection at all (and requires `Secure` to even be allowed).

In short: three config lines, written once in `app.py`. Most of the cookie-flag attack surface is closed by these three changes alone.

### Session lifetime \+ "remember me"

By default, Flask's session cookie is what browsers call a *session cookie* in the literal sense — it lives in browser memory until the user closes the browser. That's fine for a quick login, but not what you want if a user expects to stay logged in across a full workday, much less across reboots.

**`PERMANENT_SESSION_LIFETIME = timedelta(hours=12)`** sets the maximum age of the session cookie. Even if the user keeps their browser open for two days, the session cookie expires after 12 hours and they get bounced back to the login page. Pick the duration based on how sensitive your app is — a banking app might use 30 minutes; a study-spot recommender like Brew Crew can probably use a day.

**Remember-me** is a *separate*, opt-in feature provided by Flask-Login. When the user checks a "remember me" box on the login form, Flask-Login issues an additional cookie (distinct from the session cookie) that survives browser close. On future visits, even after the short-lived session cookie has expired, Flask-Login can use this remember-me cookie to re-establish the user's session. You enable it at login time:

login\_user(user, remember=True, duration=timedelta(days=30))

The pattern in practice: a short-lived session cookie for active use, plus an explicit, opt-in, long-lived remember-me cookie for users who want to stay logged in across days. The user knows — they checked the box. This is a UX choice, not a hardening one, but it lives in the same area of the codebase.

---

## 7\. CSRF in depth

CSRF (Cross-Site Request Forgery) is the one classic attack against cookie-based sessions that you must understand to do session security correctly. The lecture covers the attack in four steps; this section adds the mechanics and the defense rationale.

### The attack

1. Alice is logged into your-app.com. Her session cookie is in her browser.  
2. Alice visits evil.com (a forum, an emailed link, a compromised legitimate site).  
3. evil.com's page contains a hidden `<form action="https://your-app.com/delete-account" method="POST">` that auto-submits via JavaScript.  
4. Alice's browser issues the POST to your-app.com — and includes Alice's session cookie automatically, because the request's *destination* is your-app.com (the cookie's domain) regardless of where the request *originated*.

The server-side observation: an authenticated POST from Alice. Your code can't tell the difference between Alice clicking your own delete button and evil.com weaponizing her session.

### What evil.com does *not* have

The attack often gets misread as cookie theft. It isn't. **evil.com never reads Alice's cookie.** The browser doesn't expose cookies cross-site; same-origin policy prevents JavaScript on evil.com from reading anything about Alice's relationship with your-app.com. evil.com just *triggers* the request; the browser attaches the cookie itself.

This is also not a man-in-the-middle attack. MITM requires the attacker to control the network path between Alice and your-app.com. CSRF requires no network access — just Alice's browser visiting an attacker-controlled page while she's logged in.

The conceptual hook is the **confused deputy problem**: the browser is acting as Alice's deputy, your server trusts the cookie as proof of "Alice did this", and the attacker has tricked the deputy into doing something Alice didn't intend.

### Two complementary defenses

**`SameSite='Lax'` on the cookie**: the browser refuses to attach the cookie at all on a cross-site POST. The forged request goes through to your server, but as anonymous, so the server rejects it.

**CSRF tokens (Flask-WTF)**: every state-changing form on your site embeds a session-bound token. The server verifies the token matches the user's session on submit. evil.com can't fetch your-app.com's pages cross-site (same-origin policy blocks them from reading the response), so they can't extract a valid token to embed in their forged form.

### Why both

It's reasonable to ask: if SameSite blocks the cookie, why also need tokens?

- **GET requests with side effects.** SameSite=Lax permits the cookie on top-level cross-site GET navigations (this is why email login links work). If any of your endpoints change state on GET (`GET /logout`, `GET /unsubscribe?token=...`, `GET /delete?id=42`), they're reachable via `<a href>`, `<img src>`, or `<script src>` — SameSite gives no protection. The "right" answer is "don't have state-changing GETs", but in practice apps slip up. CSRF tokens defend.  
- **The Chrome "Lax+POST" exemption.** Cookies without an explicit SameSite setting get Lax-by-default with a quirk: cross-site POSTs *do* carry them during the first \~2 minutes after the cookie is set. Explicitly setting `SESSION_COOKIE_SAMESITE='Lax'` closes this in browsers that honor the explicit setting strictly. Tokens close it everywhere.  
- **Same-site ≠ same-origin.** SameSite treats `evil.example.com` and `app.example.com` as the same site (they share eTLD+1). If you ever host user-controlled content on a subdomain, an attacker on that subdomain can issue same-site requests with cookies attached. Tokens don't care about site; they care about session signature.  
- **Defense in depth.** Cookie flags depend on correct browser behavior. Browsers change. Configurations get mistakenly relaxed. Tokens are an app-level layer that doesn't depend on browser behavior at all.

The unified mental model: **SameSite stops the cookie from being attached; tokens stop the request from being honored even if the cookie is attached.** Two independent failure points; an attacker has to break both.

### The Flask-WTF fix (the practical implementation)

After the conceptual defenses, the practical implementation is three small changes — set-and-forget once they're in place.

**1\. Wire up CSRFProtect** once in `app.py`:

from flask\_wtf.csrf import CSRFProtect

csrf \= CSRFProtect(app)

**2\. Put `{{ csrf_token() }}` in every state-changing form** that submits via POST/PUT/PATCH/DELETE (GET forms don't change state, so they don't need protection):

\<form method="post" action="/save"\>

  {{ csrf\_token() }}

  \<\!-- form fields \--\>

\</form\>

**3\. For AJAX / `fetch` requests**, include the token as an `X-CSRFToken` header. The token is typically exposed in a `<meta name="csrf">` tag in your base template; JavaScript reads it from there before making the request:

fetch('/api/endpoint', {

  method: 'POST',

  headers: { 'X-CSRFToken': document.querySelector('meta\[name=csrf\]').content }

})

After this setup, `CSRFProtect` refuses any state-changing request that doesn't carry a matching token.

### Hardening summary

Pulling sections 6 and 7 together:

| Concern | Setting / library | What it stops |
| :---- | :---- | :---- |
| Cookie sniffed in transit | `SESSION_COOKIE_SECURE = True` (in prod) | Eavesdropping on plain HTTP |
| Cookie stolen by injected JS | `SESSION_COOKIE_HTTPONLY = True` | XSS-based cookie exfiltration |
| Cookie sent cross-site | `SESSION_COOKIE_SAMESITE = 'Lax'` | Most CSRF, third-party tracking |
| Session lives indefinitely | `PERMANENT_SESSION_LIFETIME = timedelta(hours=...)` | Stale sessions persisting forever |
| Form forgery | `CSRFProtect(app)` \+ `{{ csrf_token() }}` | CSRF (defense in depth with SameSite) |

The full hardening set is **three config flags, one duration setting, and one library call.** Five things, all set-and-forget once they're in. The implementation cost is tiny; the gain is the difference between a Flask app that's safe to put on the public internet and one that isn't.

What this *doesn't* cover, and shouldn't, this week: HTTPS termination (your app trusts the proxy to talk HTTPS), security headers (CSP, HSTS, X-Frame-Options), rate limiting on auth endpoints, secret rotation, and the operational side of running an app in production. Week 8 picks up where this leaves off — nginx as a request filter, defense in depth at the proxy layer, and the deployment-time concerns these settings deliberately skip.

These five settings don't make your app *secure* — they're the floor, the minimum below which it's irresponsible to deploy something that handles user sessions. Everything else is layered on top.

---

## 8\. Testing OAuth with Playwright

You can't drive github.com in your tests, so OAuth testing requires explicit choices about what gets mocked and what gets verified.

### The OAuth-in-tests problem

A naive integration test for OAuth would: launch a browser, navigate to your login page, click "Sign in with GitHub", complete the GitHub auth dance, return to your callback, and verify you're logged in. Three reasons this doesn't work in CI:

- **You'd need real GitHub credentials in your test environment** — both for a GitHub account to log in as, and for an OAuth app whose redirect URI matches your test host. Doable but operationally painful.  
- **GitHub rate-limits OAuth flows**, so a CI build that ran every push would hit limits quickly.  
- **GitHub's UI changes**, breaking your tests for reasons that have nothing to do with your code.

So you have to mock something. The lecture walks through three strategies.

### Three strategies, ranked

**Strategy 1: Run a fake OAuth server in your test process.** Write a tiny Flask app that mimics GitHub's OAuth endpoints (`/authorize` → redirect with a fake code, `/access_token` → return a fake token, `/user` → return a fake user payload). Point your app's OAuth config at this fake server. *Most realistic, most setup overhead.* Useful when the OAuth flow's response shape is what's at risk.

**Strategy 2: Patch Authlib at the boundary.** Use `unittest.mock` to make Authlib's `authorize_access_token()` and `oauth.github.get("user")` return fixed dictionaries. *Less realistic — bypasses the network boundary entirely — but very simple.* Useful for fast unit-test-style coverage of the post-callback logic.

**Strategy 3 (recommended for this assignment): The test-login backdoor.** Add a route to your app that's *only enabled in test mode*. The route directly logs in a named user, bypassing OAuth entirely. Your Playwright tests hit this route instead of the real OAuth flow.

@app.route("/test/login/\<username\>")

def test\_login(username):

    if not app.config.get("TESTING"):

        abort(404)

    user \= db.exec(select(User).where(User.email \== f"{username}@test")).first()

    if user is None:

        user \= User(email=f"{username}@test", display\_name=username)

        db.add(user); db.commit(); db.refresh(user)

    login\_user(user)

    return redirect("/dashboard")

The 404 guard ensures the route is invisible in production (where `TESTING` is never set). The route accepts a username, finds or creates a User, and logs them in via Flask-Login — the same `login_user(user)` call your real callback makes after find-or-create.

### Why the backdoor is the right default

**Speed.** Your tests don't hit the network. They run in seconds.

**Determinism.** No GitHub rate limits, no flakiness from real auth provider UI changes.

**Honesty.** The backdoor is *named as a mock*. It's not pretending to test OAuth — it's testing everything *after* OAuth. The actual OAuth flow gets verified manually (you click "Sign in with GitHub" in a browser at least once and confirm it works), and the post-OAuth behavior gets verified in CI via Playwright \+ the backdoor.

This is the "honest mock with a named gap" pattern from Week 6's truthy-fixtures lesson. The backdoor mocks. Your `team_walkthrough.md` lists the gap explicitly: *"we do not test the actual GitHub redirect; we test only what happens after our app receives a valid user."* That sentence in the walkthrough is what turns a hidden mock into an honest one.

### What the Part 3 tests look like

The four full-system scenarios:

1. **First-time OAuth login** — new user, no existing OAuthIdentity row, hits the backdoor, lands on the post-login page, OAuthIdentity row exists after.  
2. **Returning OAuth login** — same user, logs out, hits the backdoor again, existing row reused (not duplicated).  
3. **CSRF rejection** — POST to a state-changing endpoint without a token; request rejected.  
4. **Session expires** — with a short test-only `PERMANENT_SESSION_LIFETIME`, after expiry the protected page is no longer accessible.

Notice that none of these "test OAuth" in the strict sense. They test the *post-OAuth contract* — what happens once your app has accepted a verdict from somewhere. That's the right scope for browser-driven testing of an OAuth integration.

---

## 9\. Contracts across an external boundary

Week 6's `CONTRACTS.md` described every interface *inside* your repo — every function called, every endpoint hit, every shape passed. Week 7 introduces something new: part of the contract lives on a server you don't control.

### What you can and cannot specify

| What | Specify in CONTRACTS.md | Document only |
| :---- | :---- | :---- |
| Your `/login/github` and `/auth/github/callback` routes | Yes — they're yours | — |
| What user-info fields you require from the provider | Yes — what you depend on | — |
| Your behavior when a field is missing | Yes — defaults, errors | — |
| GitHub's actual JSON shape | — | Yes — note as external |
| Session state after callback | Yes — session dict contents, cookies set | — |
| Database rows touched on first OAuth login | Yes — `User` \+ `OAuthIdentity` | — |
| GitHub's rate limits | — | Yes — operational note |

The point: you *cannot* specify GitHub's behavior. You *can* specify what your code does in response to GitHub's behavior, including what happens when GitHub returns something unexpected. Note external dependencies honestly in the contract: `external_dependency: github.com — see study guide for representative payload shape`.

### Tests are part of contracts, not separate

The pattern you've been building toward all course: a contract isn't complete until you've named the tests that enforce it. By Week 7, this should feel natural — contracts include the tests that enforce them, not as a separate document but as part of the spec.

How you and the LLM decide to capture those tests in `CONTRACTS.md` is up to you. Some teams write them inline next to each contract item ("*Tested by:* `test_dashboard_requires_login`"). Some teams keep them as a flat list at the bottom. The format isn't graded; the existence of test scenarios mapped to contract items is.

### The six items the Part 1 contract revision must cover

1. **The two new routes** — `/login/github` and `/auth/github/callback`, their inputs and outputs.  
2. **Required provider fields** — what you depend on from the user-info payload; what your code does if a field is missing.  
3. **Local user record shape** — fields and types of a User the first time they sign in via OAuth.  
4. **External identity link** — how `OAuthIdentity` rows map to `User` rows; one user, possibly multiple providers.  
5. **Post-callback session state** — what's in the session dict, which cookies are set with what flags.  
6. **Logout** — what you clear locally; what you don't touch at the provider, and why.

For each, the coordinator's planning session with the LLM produces both contract language and (at the team's discretion) the test scenario that verifies it.

---

## 10\. Common pitfalls and debug aids

Things that go wrong, what they look like, what to check first.

### Redirect URI mismatch

**Symptom:** GitHub returns an error page saying `redirect_uri_mismatch` after the user clicks Authorize, or simply fails silently.

**Cause:** The `redirect_uri` registered on your GitHub OAuth App (set when you created the app at github.com/settings/developers) doesn't exactly match the URL your app is using as the callback. GitHub does an exact string comparison: a trailing slash matters, http vs https matters, port number matters, `localhost` vs `127.0.0.1` matters.

**Fix:** Make sure the GitHub OAuth App's "Authorization callback URL" is *exactly* `http://localhost:5000/auth/github/callback` (or whatever your dev URL is) and that your app uses *exactly* the same URL when building the redirect.

### `SECURE=True` on localhost → silent login loop

**Symptom:** User clicks Sign in with GitHub, completes the GitHub flow, redirected back to your app, and is immediately redirected to login again. No error message.

**Cause:** `SESSION_COOKIE_SECURE = True` makes the browser refuse to send the cookie over HTTP. Localhost is HTTP. So the cookie gets set after the callback but the browser refuses to attach it on the next request, your app sees no session, and redirects to login.

**Fix:** Use env-driven config so `SECURE = False` in dev and `True` in prod. Never hardcode `True`.

### Email-as-foreign-key

**Symptom:** A user changes their GitHub email (or you start using a different provider that has the same email), and they're treated as a new user — losing their account history.

**Cause:** Your `OAuthIdentity` table (or wherever you map external → local) used `email` as the join key.

**Fix:** Use `(provider, provider_user_id)`. Always. Email is a display field, refreshed each login.

### Tautological tests

**Symptom:** Your test passes. Always. Including when you delete the entire feature it's supposed to be testing.

**Cause:** The assertion doesn't depend on anything the code did. `assert True`. `assert page.url` (which is always truthy). `expect(page).not_to_be_visible()` on an element that never existed.

**Fix:** Before you trust a test, *break the code it's supposed to be testing* and confirm the test fails. A test that can't fail isn't a test.

### Don't hit real GitHub in tests

**Symptom:** Your CI build occasionally fails because GitHub rate-limited your test runner, or because GitHub temporarily had a service blip, or because GitHub changed something about their UI.

**Cause:** Your test is actually completing the OAuth dance against real GitHub instead of using the test-login backdoor.

**Fix:** Use the backdoor. The actual GitHub redirect gets one manual smoke test (the "I clicked it in a browser" gap explicitly named in `team_walkthrough.md`); CI uses the backdoor exclusively.

### A missing secret crashing on use instead of startup

**Symptom:** Your app starts fine, runs fine, until a specific code path tries to use `SECRET_KEY` (or any other env var) and you get a confusing runtime error from deep in Flask's session machinery.

**Cause:** You used `os.environ.get("SECRET_KEY")` (returns `None` silently) instead of `os.environ["SECRET_KEY"]` (raises `KeyError` immediately).

**Fix:** Use `os.environ[KEY]` for required secrets. A missing required secret should crash your app on startup with an obvious error, not turn into a debugging session two minutes in.

---

## A short reading list

If you want to go deeper on any of this:

- **OAuth 2.0 Authorization Code Grant** — the spec (RFC 6749 §4.1). Dense but short. Read the abstract and the flow diagram.  
- **Authlib for Flask** docs — the library you're using. Section on "Flask Client" covers what `OAuth(app)`, `register()`, `authorize_redirect()`, and `authorize_access_token()` do.  
- **OWASP CSRF Prevention Cheat Sheet** — the canonical reference. The "Token-Based Mitigation" section is the one to read.  
- **Playwright Python — Locators** — the page you'll come back to most when writing tests.

