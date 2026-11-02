# Changelog

This file records the changes we make to CyberDome, newest first. For each fix we write what we saw, what was wrong and how we fixed it. Runtime-generated data is listed separately from intentional source and configuration changes.

## 2026-10-20

### Changed: the trust factors have three named groups

Documentation only. The code and the weights are unchanged.

**What we saw:** Our two documents grouped the trust factors in different ways. `docs/trust-model.md` called login history, device and access frequency the "behaviour" factors (55% of the score), and gave no name to the other four. `docs/policies.md`, in the table "How each policy uses identity and context", used two groups instead: identity (login history, MFA, account age) and context (device, location, time, access frequency). So login history was "behaviour" in one document and "identity" in the other, and device and access frequency were "behaviour" in one and "context" in the other.

**Why it matters:** we explain the weights by group ("behaviour first"), in the documents, the report and the presentation. If the groups change from one document to the next, the explanation is hard to follow, and a reader can't tell which split is right.

**What we changed:**

- `docs/trust-model.md` has a new section, "Three groups of factors": behaviour (login history, device, access frequency, 0.55), context (location, time of access, 0.25) and identity (identity strength, account age, 0.20). It explains each group with an example (Bob, and an attacker with Bob's password), why we use three groups, and why login history is behaviour and not identity.
- `docs/policies.md`: the P4 and P5 rows use the same three groups. The identity column now lists only the identity factors, and the request context column lists the behaviour and context factors. A short note under the table explains why behaviour sits on the context side.

**Example of the old and new P4 row:**

| | Identity or role | Request context |
| --- | --- | --- |
| Before | The admin role and the identity factors (login history, MFA, account age) | The context factors (device, location, time, access frequency) |
| After | The admin role and the identity factors (identity strength, account age) | The behaviour factors (login history, device, access frequency) and context factors (location, time of access) |

## 2026-10-14

### Added: diagrams, and a README that describes CyberDome as it is now

**What we saw:** The only picture of the system was a text diagram in `docs/how-it-works.md`. The README still described the original plans ("will be implemented", Nginx, machine learning, Markov chains), listed only two of the demo users, and didn't mention the tests.

**What we changed:**

- New `docs/diagrams.md` with 8 diagrams in Mermaid (GitHub and VS Code draw them from the text): architecture, logging in, a protected request (sequence), the policies P1 to P6 (flowchart), the trust score, the audit log, a code map, and a table of the data files. We checked that every diagram renders with the Mermaid command-line tool.
- `README.md` is rewritten: what CyberDome does now (the Zero Trust workflow and the five parts), a small diagram, a guide to the documents, the demo users and their roles, setup and start steps (kept from before), how to run the tests, the project structure, limitations, and contribution steps.

### Added: one audit log for every access decision (assignment section 3.5)

Branch: `feat-audit-logging`. The log and the reasons behind it are in `docs/audit-log.md`.

**What we saw:** Only the Policy Engine's decisions were saved, in `data/access_decision.json`. Failed logins, invalid or revoked tokens, visitors who weren't logged in, and requests that failed because the engines were down left no trace. Every request read and rewrote `data/access_decision.json` and `data/access_requests.json` in full, so each request got slower as the files grew: after many test runs (about 1 MB), 45 ms instead of 21 ms.

**What we changed:**

- New `cyberdome/audit.py`: writes audit entries to `data/audit.log`, one JSON object per line, only appended. Each entry has the task's data model fields (`user`, `role`, `resource`, `action`, `device_id`, `source_ip`, `decision`, `trust_score`, `reason`), plus the outcome category (`ALLOW`, `DENY`, `AUTHENTICATION_FAILURE`, `POLICY_VIOLATION`), the policy, who decided, the trust breakdown and the context.
- New `cyberdome/jsonl.py`: appends to one-JSON-object-per-line files with a file lock (the Web UI and the engines are separate processes), hands out the next ID under the same lock, and reads files backwards from the end.
- The Policy Engine writes each decision to the audit log before sending it. It no longer writes `data/access_decision.json`.
- The Web UI logs its own decisions: not logged in (`AUTHENTICATION_FAILURE`), an invalid, expired or revoked token (`AUTHENTICATION_FAILURE`), and Keycloak or the engines not reachable or too slow (`DENY`).
- Failed logins: Keycloak's `LOGIN_ERROR` events are copied into the audit log as `AUTHENTICATION_FAILURE`, when the home page or the `/logging` page syncs events. The event sync now keeps the typed username and Keycloak's error (for example `invalid_user_credentials`). `data/keycloak_audit_state.json` remembers the newest copied event, so none is copied twice.
- The access requests moved to `data/access_requests.jsonl`, also append-only. The access frequency factor reads it from the end and stops at requests older than the window.
- The `/logging` page shows the audit log, newest first, with filters for outcome and user, and the trust breakdown per entry.
- The old `data/access_decision.json` and `data/access_requests.json` stay in `data/` untouched, and are no longer written.
- New `tests/test_audit.py`: 15 tests, for example reading backwards across blocks, unique IDs from 20 threads at once, the data model fields, the filters, and copying failed logins only once. All 15 passed.
- `tests/check_access_control.py` reads decisions from the audit log, and checks the data model fields, the `AUTHENTICATION_FAILURE` entries for a visitor who isn't logged in and for fake and revoked tokens, and the `/logging` page. All 78 checks passed.
- Docs: new `docs/audit-log.md`, and an Audit Log section in `docs/how-it-works.md`.

**Performance:** we measured the median time per request in `tests/check_access_control.py` (10 requests by bob on Resource 1):

| Files | Before (JSON files rewritten) | After (append-only) |
| --- | --- | --- |
| Small (new files) | about 21 ms | about 17 to 19 ms |
| About 1.3 MB | about 45 ms | about 18 ms |

For the 1.3 MB test we filled copies of the new files with entries dated one day earlier, ran the test, and put the real files back. With append-only files the time no longer grows with the amount of data.

### Added: six named policies in the Policy Engine (assignment section 3.4)

Branch: `feat-policy-engine`. The policies and the reasons behind them are in `docs/policies.md`.

**What we saw:** The Policy Engine had one trust threshold per role (Approver 0.9, Policy Administrator 0.6, Security Viewer 0.4) and one sign-in check. Researcher and Administrator had no threshold, so they were never denied for low trust. The Policy Engine didn't know which resource was requested, and it read only the user's first role from `data/user_data.json`. The role table was checked by the Web UI, so requests with the wrong role never reached the Policy Engine and weren't decided or logged there. Decisions said allow or deny, but not which rule decided.

**What we changed:**

- `cyberdome/policy.py` is rewritten with six named policies, checked in order. The first one that denies decides:
  - P1 Invalid authentication (`AUTHENTICATION_FAILURE`)
  - P2 Unauthorized resource (`POLICY_VIOLATION`): the role isn't allowed the resource, or the request changes something (`write`) and the role may only read
  - P3 Risky context, depending on the role: a high-risk country (everyone), a medium-risk country (privileged roles), the night window (privileged roles, or anyone on an admin resource), or too many failed logins
  - P4 Admin resource: needs a trust score of at least `adminMinTrustScore`
  - P5 Low trust: the score is below the resource's `minTrustScore`. The reason names the two weakest trust factors
  - P6 Permitted access
- Every decision has `decision`, `outcome`, `policy_id`, `policy_name` and `reason`. The Policy Engine saves them in `data/access_decision.json`, together with the trust score and breakdown.
- The old per-role function is kept as the commented-out `retired_make_access_decision`, for comparison.
- The Policy Engine uses all roles from the token (in the request record), not the first role from `user_data.json`.
- `web/authz.py`: the Web UI no longer checks the role table (`require_role` is removed). It only checks the login, and the Policy Engine decides the rest. Refused pages show the policy and the reason. Responses have a new `X-CyberDome-Policy` header.
- `policyConfiguration.yml`: each resource has a `minTrustScore` (`resource_1` 0.6, `resource_2` 0.75, `pam_approval` 0.75, `policy_configuration` 0.6, `logging` 0.6) and a `writeRoles` list (only `pam_approval`: Approver, and `policy_configuration`: Policy Administrator). New settings: `adminMinTrustScore` (0.75) and `privilegedRoles` (Administrator, Policy Administrator, Approver). The three role thresholds are removed.
- Separation of duties: an Administrator may view the policy page but not save it. Only a Policy Administrator may change the policies.
- So that each policy uses both identity or role and request context (as the task asks), P2 checks the action and P3 depends on the role. The first version of P2 only looked at the resource, and P3 never looked at the role.
- The policy page shows the minimum per resource and the admin minimum instead of the role thresholds. When saving, the server only changes the minimums, never the allowed roles or the admin flag.
- New `cyberdome/config.py` reads `policyConfiguration.yml`. It used to be in `policy.py`, but `policy.py` and `trust_algorithm.py` now use each other's code, and Python can't import two files that import each other.
- New `tests/test_policies.py`: 29 tests for each policy, write roles, role-dependent context, the order of the policies and the five scenarios of the task. They don't need Keycloak or the engines. All 29 passed.
- `tests/check_access_control.py`: a wrong role, or a write without a write role, is now expected to be decided by the engines and denied by P2. The decisions must have a policy and the trust breakdown, scenario 2 must be saved as `POLICY_VIOLATION`, and separation of duties is checked. Where trust can decide (P4, P5), a deny is accepted, because running the test several times in a row counts as a burst of requests. All 72 checks passed, five runs in a row.
- **Fixed: the decision was sent before it was saved.** The Policy Engine sent each decision to the Web UI and only then rewrote `data/access_decision.json`. A reader (our test) could read the file while it was half written and crash. Now the decision is saved first (so it's logged before access is given), through a temporary file that is swapped in, so the file is never half written.
- **Fixed: the first request after a restart of the Web UI could time out (`504`).** The engines kept the old, dead connection to the Web UI for a moment, and sent the verdict there. Now a node that connects again with the same ID replaces its old connection, and messages go to the newest connection. This bug was there since step 2.
- Docs: new `docs/policies.md`, and the access-decision flow, policy and role sections of `docs/how-it-works.md` are updated.

**Effect:** each request reads and rewrites `data/access_requests.json` and `data/access_decision.json` in full, and the decisions are bigger now (trust breakdown). After many test runs (about 1 MB of decisions) a request took about 45 ms instead of 21 ms. We'll fix this in section 3.5 with a log that only appends.

alice (Approver) is now allowed on the PAM approval pages. Before, she needed 0.9, which nobody reached. bob on Resource 2 is now denied by P2 with a logged reason, instead of by the Web UI without a decision.

### Changed: a new trust model with 7 factors and configurable weights (assignment section 3.3)

Branch: `feat-trust-engine`. The full model and the reasons are in `docs/trust-model.md`.

**What we saw:** All weights were hard-coded. The policy page saved weight settings, but nothing read them. The score used the user's role (admins got a higher score), the authentication type (the same for everyone), and the OS and device type (Linux and phones scored lower, with no reason given). It had no device status or access frequency, which the task names as examples. We also found these bugs:

- The account-age tiers were in the wrong order, so every account older than about 4.5 days got the top score.
- The night window couldn't cross midnight.
- The sign-in value could go up to 1.5, because each user's history started with a made-up 0.
- `sign_in_risk` measured success, not risk.
- `KE` was in both the high-risk and medium-risk lists.
- The config was read once, so changes on the policy page only applied after restarting the engines.
- The policy page reloaded itself every 5 seconds, which wiped what you typed and sent a request through the engines each time. Its fields always started at fixed values, so saving could reset the settings. Without a PAM request, the page crashed.

**What we changed:**

- `cyberdome/trust_algorithm.py` is rewritten. It scores 7 factors: login history (0.20), device (0.20), access frequency (0.15), location (0.15), identity strength (0.15), time of access (0.10) and account age (0.05). The weights come from `trustWeights` in `policyConfiguration.yml`, and the score is divided by their sum. Each factor returns a score and a reason, and `calculate_trust_breakdown` returns both. The config is read for every request.
- The removed factors (role, authentication type, OS, device type) are kept as the commented-out function `retired_factor_scores`, with the reason for each.
- `cyberdome/trust_signals.py`: the sign-in value is a plain success rate over the user's last 10 login attempts, renamed to `sign_in_success_rate`. New helpers: `get_sign_in_success_rate` and `count_recent_access_requests`.
- `cyberdome/networking.py`: the Trust Engine sends the score and the breakdown (`trust_factors`). The Policy Engine saves the breakdown with each decision in `data/access_decision.json`, and uses the new sign-in success rate.
- `cyberdome/policy.py` and `policyConfiguration.yml`: `signInRiskThreshold` is renamed to `signInSuccessThreshold`. New settings: `trustWeights`, `accessFrequencyWindowMinutes` (5) and `accessFrequencyMaxRequests` (30). The four old weight settings, which nothing read, are removed. Risk lists: high `KP`, medium `KE`, low `SE`.
- The policy page shows the 7 weights and the frequency settings, starts every field with the saved value, no longer reloads itself, and works without a PAM request.
- New `tests/test_trust.py`: 21 tests for every factor, the weights, the bug fixes and the example scores. They don't need Keycloak or the engines. All 21 passed.
- Docs: new `docs/trust-model.md`, and a shorter Trust Calculation section in `docs/how-it-works.md` that points to it.

**Effect on scores:** a normal user on a known device now scores about 0.90, and about 0.78 on a new device (before: about 0.57 to 0.70 for everyone). The Policy Engine still uses its old role thresholds until section 3.4. With the new scores, foo (Policy Administrator, 0.6) is allowed, and alice (Approver, 0.9) is still denied.

**Old data:** entries in `data/auth_data.json` still have the old `sign_in_risk` key. Nothing reads it any more, because the success rate is calculated from `auth_status`.

### Added: a real request record with device ID and the user's location (assignment section 3.2)

**What we saw:** The request record still had the **Flask server's** public IP, location and MAC address (`uuid.getnode()`), so every user got the same values, wherever they were. A website can't see a visitor's MAC address anyway, because MAC addresses don't cross routers. The record had no device ID. The location was stored as `City/CC` (`Nairobi/KE`), but the risk lists in `policyConfiguration.yml` hold country codes (`KE`), so they never matched.

**What we changed:**

- `web/request_context.py`:
  - Every browser gets a random device ID in a `cyberdome_device_id` cookie on its first visit (`HttpOnly`, `SameSite=Lax`, one year). An invalid cookie is replaced with a new ID. The record has it as `device_id`.
  - The record has `new_device`: `true` until access was allowed once from this device for this user. Known devices are kept in `data/known_devices.json` (git-ignored, like the rest of `data/`).
  - `location` is now the country code of the user's own `source_ip`, looked up on ipinfo.io (once per IP). Localhost and private addresses give `LOCAL`, and a failed lookup gives `Unknown`.
  - `public_ip_address`, `device_mac` and `device_vendor` are removed.
- `web/app.py` runs the device ID cookie functions on every request. `web/authz.py` remembers the device after an "allow" decision.
- `web/keycloak_functions.py`: removed `get_mac_details`, `get_public_ip` and `get_location`, which only looked up the server's own values.
- `web/templates/logging_and_monitoring.html` shows source IP, device ID and new device instead of the server's public IP, MAC and vendor.
- `cyberdome/networking.py`: the Policy Engine prints the location as it is, without splitting `City/CC`.
- `tests/check_access_control.py` has a new section for the request record: the cookie is set and safe, the record holds the browser's device ID and `LOCAL`, the server fields are gone, the device goes from new to known after an allowed request, and an invalid cookie is replaced.
- Docs: a new "Request Record" section in `docs/how-it-works.md`, with the reasons for each field.

**Not done yet:** the Trust Engine doesn't use `new_device` yet, and `LOCAL` gets the default location score. Both are part of step 6 (section 3.3).

### Added: every protected request goes through the Access Proxy (assignment section 3.2)

**What we saw:** The engines' verdict only told the browser's JavaScript where to go next. The resource pages never asked for a verdict, so opening `/resource-1` directly skipped the Access Proxy, Trust Engine and Policy Engine, and the admin pages never went through the engines at all. The engines only got a user ID, and looked up "the user's latest request" in `data/access_requests.json`, which could pick the wrong request when several came in close together. Each request also started and stopped its own node 4 on port 8004, so two requests at once clashed.

**What we changed:**

- `web/authz.py` has a new `@protected_resource(resource)` check, used on every page in the role table (both resources, policy configuration, logging, PAM approval). It runs the login, token and role checks, then sends the request through Access Proxy, Trust Engine and Policy Engine, and only runs the page if the Policy Engine allows it. A deny gives `403` with the Policy Engine's reason. No verdict in 10 seconds gives `504`, and an unreachable Access Proxy gives `503`. Responses carry `X-CyberDome-Request-Id`, `X-CyberDome-Decision` and `X-CyberDome-Decided-By` headers.
- `web/request_context.py` builds the request record on the server: user, roles, resource, action (`read` or `write`), time, authentication status, source IP, and device type, OS and browser from the `User-Agent` header. Location, public IP and MAC are still the server's own values, now looked up once per process instead of on every request.
- `web/proxy_client.py` keeps one Web UI node (node 4) for all requests, started on the first request and reconnected when needed. Request IDs are given out under a lock. `stop_node` stops it cleanly, so Ctrl+C on the Web UI exits.
- `cyberdome/networking.py`:
  - The Access Proxy checks the record is complete (ID, user, resource, action, authentication status), stamps it as mediated, and forwards the whole record. Incomplete records are rejected with a reason.
  - The Trust Engine scores this request. If scoring fails, it passes the error on instead of crashing.
  - The Policy Engine decides on this request, and saves and returns the decision with resource, action, trust score, reason and time.
- `cyberdome/trust_algorithm.py`: `calculate_overall_trust_score` can take the request record directly.
- The resource selection page links straight to the resources. `POST /receive-access-request` and the browser-side context collection are removed.
- `tests/check_access_control.py` now needs the engines running. It checks that table-allowed requests are mediated and decided by the Policy Engine, that a direct URL is mediated and its decision saved with a reason, that 5 requests at once each get their own verdict, and it measures the time per request. All 59 checks passed. A mediated request took about 18 ms (median, 10 requests).

### Added: role-based access control on every page (assignment section 3.1)

**What we saw:** Roles were only used to grey out options in the browser (`home.html`), and only for three old roles, so Researcher and Administrator saw everything. Most pages had no login check at all. `/testing` and `/viewAccessRequests` had their login check in the wrong order, so it never ran. `/receivePolicyConfigurations`, which saves policy changes, could be called by anyone. `/hidden_resource` let anyone test guesses for the PAM secret, and printed the real secret to the console.

**What we changed:**

- `policyConfiguration.yml` has a new `resources` table: each protected resource with its path, allowed roles, and whether it is an admin resource. Authorization is now driven by this file.

| Resource | Path | Allowed roles | Admin resource |
| --- | --- | --- | --- |
| `resource_1` | `/resource-1` | Researcher, Administrator | No |
| `resource_2` | `/resource-2` | Administrator | No |
| `policy_configuration` | `/configurePolicies`, `/receivePolicyConfigurations` | Administrator, Policy Administrator | Yes |
| `logging` | `/logging` | Administrator, Security Viewer | Yes |
| `pam_approval` | `/testing`, `/approve_request` | Approver | No |

- `cyberdome/access_rules.py` reads the table (`allowed_roles`, `is_role_allowed`, `is_admin_resource`). The Policy Engine will use the same functions.
- `web/authz.py` adds two checks for pages:
  - `@require_authentication`: logged in, and Keycloak says the token is still active (one token introspection per request). Otherwise a login redirect or `401`.
  - `@require_role(resource)`: the same, and one of the user's roles must be allowed for the resource. Otherwise `403`.
  - If Keycloak can't be reached, the answer is `503`. Unknown resources are refused. Both fail closed.
- `web/templates/access_denied.html` explains a `401` or `403` refusal.
- Every page except `/`, `/login`, `/revokeToken` and `/home` (which handle login themselves) now has one of the two checks. This also fixes the decorator order on `/testing` and `/viewAccessRequests`.
- `/hidden_resource` is removed. The secret check is now a plain function (`is_valid_secret_key`) and no longer prints the secret.
- The home menu gets the allowed options from the server, worked out from the same table, instead of guessing from one role name. The page shows all of the user's roles.
- Added `tests/check_access_control.py`. It logs in as alice, bob, charlie and foo through Keycloak and checks every protected page against the table, plus not-logged-in, fake-token and revoked-token cases, and task scenarios 1, 2 and 5. All 54 checks passed.

## 2026-10-13

### Changed: `app.py` split into blueprints

**Why:** `web/app.py` was about 680 lines with every page in one file. Section 3.1 of the assignment needs role checks on the pages, which is easier when each area has its own file.

**What we changed:**

- `web/app.py` now only creates the app, loads its settings, connects the shared objects and registers the pages (about 70 lines).
- `web/extensions.py` creates the shared objects without an app: `oidc`, `db`, `keycloak_admin` and `keycloak_openid`. `app.py` connects them with `init_app`. This way the page files can import them without importing `app.py`, which would be a circular import.
- `web/models.py` holds the PAM database tables.
- `web/routes/` holds the pages, one Flask blueprint per area: `auth`, `access`, `resources`, `policy_config`, `monitoring` and `privileged_access`. The route code is unchanged.
- `url_for` calls use the blueprint names, for example `auth.home`. Templates only link by path, so they didn't change.
- Kept as they were, to be fixed with the role checks: `/testing` and `/viewAccessRequests` have `require_login` above `route`, so they have no login check. They are marked with a comment.
- Before the change we recorded how all 26 routes answer a visitor who isn't logged in, using Flask's test client on a copy of the app. After the change, all 26 answered the same: same status codes and redirects. We also checked that every `url_for` target exists and that no name is undefined.

### Changed: consistent names in the Web UI

**Why:** The Web UI mixed naming styles (`Keycloak_functions.py`, `PAM_Mail_Notification.py`, `apprPage.html`), and one template had spaces in its name (`Fintech Access Tokens.html`).

**What we changed:**

- Renamed the folder `ZeroTrustWebUI/` → `web/`.
- Renamed the Python modules to `snake_case`: `Keycloak_functions.py` → `keycloak_functions.py`, `PAM.py` → `pam.py`, `PAM_Mail_Notification.py` → `pam_mail.py`.
- Renamed the templates to `snake_case`:
  - `apprPage.html` → `approver_page.html`
  - `enterSecretKey.html` → `enter_secret_key.html`
  - `Fintech Access Tokens.html` → `fintech_access_tokens.html`
  - `policyConfiguration.html` → `policy_configuration.html`
  - `privilegedAccessManagement.html` → `privileged_access_management.html`
  - `protectedPage.html` → `protected_page.html`
  - `resourceSelection.html` → `resource_selection.html`
  - `viewAccessRequests.html` → `view_access_requests.html`
- Updated the imports and template names in `app.py`, the folder in `cyberdome/data_paths.py`, `scripts/runWebUI.sh` and `.gitignore`.
- URLs such as `/enterSecretKey` didn't change. Only file names did.
- We checked that every template the code uses exists (except `no_requests.html`, which was already missing), that all 14 templates load, that every linked CSS file exists, and that the app loads with all 25 routes from another folder.

### Changed: Web UI secrets moved to `.env`

**Why:** Secrets were written in the code: the Keycloak client secret (`keycloak_config.py` and `client_secrets.json`), the Keycloak admin password and the Flask session key (`app.py`), and a real Gmail app password belonging to the original author (`email_constants.py`).

**What we changed:**

- Added `cyberdome/settings.py`. It reads `.env` from the repo root if it exists, then takes every setting from environment variables. Variables already set in the environment win over the file.
- Secrets have no default in the code. If `KEYCLOAK_CLIENT_SECRET`, `KEYCLOAK_ADMIN_PASSWORD` or `FLASK_SECRET_KEY` is missing, the Web UI stops at startup with a message that names it.
- Non-secret settings (Keycloak URL, realm, client ID, admin username) keep their old values as defaults.
- `app.py` builds the Flask-OIDC client settings in code, so `ZeroTrustWebUI/client_secrets.json` is deleted. The Web UI no longer depends on the folder it starts from.
- `keycloak_config.py` only builds the Keycloak URLs from the settings.
- The PAM approval email reads `PAM_EMAIL_SENDER` and `PAM_EMAIL_PASSWORD`. Without them, it skips the email and prints why. `ZeroTrustWebUI/email_constants.py` is deleted.
- Added `.env.example` with every variable, and a setup step in the README.
- We tested that the Web UI stops with a clear message without the secrets, and that with them it loads all 25 routes, logs in to the Keycloak admin API, and reaches the client configuration, even when started from another folder. We also tested the `.env` reader with comments, quotes, spaces and invalid lines.
- Not covered yet: the identity stack's local passwords in `compose.yml` and `infra/`.

### Changed: removed dead code

**Why:** Unused imports, unused variables and commented-out code make the files harder to read, and hide which code actually matters.

**What we changed:**

- `cyberdome/networking.py`: the Policy Engine loaded the user's three records twice (once to print, once to use). It now loads them once and prints those values. Removed four lines that worked out the request time and never used it.
- `cyberdome/trust_algorithm.py`: removed the unused `os` import.
- `ZeroTrustWebUI/app.py`:
  - merged the two `from flask import ...` lines
  - replaced `from keycloak_config import *` and `from Keycloak_functions import *` with the names `app.py` uses
  - removed the unused `send_email` import
  - removed two commented-out lines and the unused loop counter in the privileged-access request loop
  - removed an unused database query and a commented-out line in `testApproval()`
- `ZeroTrustWebUI/Keycloak_functions.py`: removed the unused `KeycloakAdmin` import.
- `ZeroTrustWebUI/synthetic_data.py`: removed the unused `string` import.
- We found these with a small checker built on Python's `ast` module, and ran it again afterwards with no findings. A second check confirmed that every name the code uses is defined or imported. `app.py` loads with all 25 routes, and a request through the engines gave the same output and verdict as before.

### Changed: engine launchers moved to `engines/`, leftovers removed

**Why:** The repo root mixed the engine launchers, an old copy of the Access Proxy and an empty `__init__.py` with the project files.

**What we changed:**

- Moved the launchers to `engines/` and renamed them to `snake_case`: `AccessProxy.py` → `engines/access_proxy.py`, `TrustEngine.py` → `engines/trust_engine.py`, `PolicyEngine.py` → `engines/policy_engine.py`.
- The launchers add the repo root to Python's import path from their own location. The old `sys.path.insert(0, '..')` depended on the start folder.
- `scripts/runAll.sh` and the per-engine scripts start the new paths.
- Deleted `node1.py`, an old copy of the Access Proxy launcher.
- Deleted the empty root `__init__.py`. Nothing imports the repo root as a package anymore.
- We ran alice's request through the engines started by `scripts/runAll.sh` again. The Web UI got the verdict for that request, with the same score and verdict as before.

### Changed: one node base class with a subclass per node

**Why:** There were two node classes. `Networking.py` in the root was used by all three engines, and `ZeroTrustWebUI/Networking.py` was an older, smaller copy used by the Web UI. Every engine carried the handlers of every other engine and worked out its job from who sent the message. The two copies had already drifted apart.

**What we changed:**

- Moved `Networking.py` to `cyberdome/networking.py` and removed `ZeroTrustWebUI/Networking.py`.
- The `Networking` base class keeps what every node shares: sending to a node by ID, receiving and dispatching by sender, catching errors, and the connection messages. Its message handlers only print.
- Added one subclass per node. Each one sets its own node ID and overrides only the handler it needs:
  - `AccessProxyNode` (1) handles requests from the Web UI.
  - `TrustEngineNode` (2) handles score requests from the Access Proxy.
  - `PolicyEngineNode` (3) handles scores from the Trust Engine.
  - `WebUINode` (4) handles verdicts from the Policy Engine and has `wait_for_access_decision`.
- The launchers, `app.py`, `node1.py` and `tests/testAP.py` create their own node type, for example `AccessProxyNode("127.0.0.1", 8001)`. The node ID is no longer passed in, so it can't be wrong.
- The engine logic itself didn't change.
- We tested the node IDs, that each class overrides only its own handler, that a crash in a handler keeps the connection working, and that `WebUINode` receives its verdict. We also ran alice's request through the real engines started by `scripts/runAll.sh`. The Web UI got the verdict for that request, with the same score and verdict as before.

### Changed: policy rules moved to `cyberdome/policy.py`

**Why:** The Policy Engine's rules (`make_access_decision`) were a method inside the network class in `Networking.py`, next to the message handling. Section 3.4 of the assignment needs at least five explicit policies, so the rules need a place of their own that is easy to read, test and extend.

**What we changed:**

- Moved `make_access_decision` from `Networking.py` to `cyberdome/policy.py` as a plain function. The rules are the same.
- Added `load_policy_configuration`, which reads `policyConfiguration.yml` on every call, like before.
- `make_access_decision` can take the configuration as an optional argument. Tests can use this instead of changing the file.
- Removed the `yaml` and `POLICY_CONFIG_FILE` imports from `Networking.py`. Only the moved method used them.
- We compared the old method and the new function on 2,772 combinations of role (including unknown roles and no role), trust score and sign-in value. They gave the same verdict every time.

### Changed: shared code moved into a `cyberdome/` package

**Why:** The trust score and the data helpers are used by both the engines and the Web UI, but they lived inside `ZeroTrustWebUI/`. The engines had to import them as `ZeroTrustWebUI.TrustAlgorithm`, and the Web UI imported the same files a different way, which needed a try/except import.

**What we changed:**

- Moved three files into a new `cyberdome/` package and renamed them to Python's usual `snake_case` style:
  - `ZeroTrustWebUI/TrustAlgorithm.py` → `cyberdome/trust_algorithm.py`
  - `ZeroTrustWebUI/trust_signal_collection.py` → `cyberdome/trust_signals.py`
  - `ZeroTrustWebUI/data_paths.py` → `cyberdome/data_paths.py`
- Everything now imports them as `cyberdome.*`. `app.py` and `synthetic_data.py` add the repo root to Python's import path, based on their own location and before their imports. The old `sys.path.insert(0,'..')` in `app.py` ran after the imports and depended on the start folder.
- `Networking.py` imports the three functions it uses by name, and imports `json` and `os` itself. Before, it got them through `import *`.
- Removed an unused `import *` from `PolicyEngine.py`.
- No calculation changed. We checked that alice's trust score stayed at 0.609, that `app.py` loads with all 25 routes, and that a request through all three engines still returns the matching verdict.

### Changed: scripts and test files have their own folders

**Why:** Five shell scripts and two test scripts were spread over the repo root and `ZeroTrustWebUI/`. Each one only worked when started from the folder it lived in.

**What we changed:**

- Moved `runAll.sh`, `runAccessProxy.sh`, `runTrustEngine.sh`, `runPolicyEngine.sh` and `ZeroTrustWebUI/runWebUI.sh` to `scripts/`. Each script now moves to the folder it needs first (the repo root, or `ZeroTrustWebUI/` for the Web UI), so it works from any folder.
- Moved `test.py` and `testAP.py` to `tests/`. They find the project code from their own location. `test.py` now takes the user ID as an argument, because its hard-coded ID belonged to a user that no longer exists.
- We tested this by starting `scripts/runAll.sh` from an unrelated folder (all three engines came up) and running `tests/test.py` from two folders (same score).

### Changed: runtime data now lives in `data/`

**Why:** The generated files were spread over the repo root and `ZeroTrustWebUI/instance/`. Each file was found by a path relative to the folder the process started in, so the Web UI had to start from `ZeroTrustWebUI/` and the engines from the repo root.

**What we changed:**

- Added `ZeroTrustWebUI/data_paths.py`, which builds every data and config path from the code's location.
- Moved `access_requests.json`, `access_decision.json`, `auth_data.json`, `events.json`, `user_data.json` and `privileged_access.db` into `data/`. The folder is created on startup and git-ignored.
- `Networking.py`, `TrustAlgorithm.py`, `trust_signal_collection.py`, `app.py` and `synthetic_data.py` now use these paths. `policyConfiguration.yml` and the demo data stay where they were, but are also found by absolute path.

### Fixed: one error made an engine stop listening

**What we saw:** After the Trust Engine crashed once, it printed nothing for later access requests, even for users with valid data. No verdict came back.

**What was wrong:** `p2pnetwork` reads each connection's messages in its own thread and passes them to our `node_message`. When our code raised an error there, nothing caught it, so the thread ended. The socket stayed open, so the Access Proxy could still send without any error, but the Trust Engine never read another message from it until a restart.

**How we fixed it:** `node_message` in both `Networking.py` and `ZeroTrustWebUI/Networking.py` now catches errors and prints the traceback with the node's name. The connection keeps working for the next message. We tested this with two real nodes: a message that crashes, then a normal one, which still arrived.

### Fixed: access requests failed with "Invalid MAC Address"

**What we saw:** Clicking a resource under **Access Protected Resources** showed "Invalid MAC Address" for every user, and the request never reached the engines.

**What was wrong:** `/resource-selection` reads the MAC address of the machine running Flask and asks macvendors.com who made it. Our Mac uses a private (randomly generated) MAC, which macOS uses for privacy and virtual network adapters also use. Random MACs have no maker, so the lookup returned `404`. The free API also returns `429` when we call it more than about once a second. In both cases `get_mac_details` (`ZeroTrustWebUI/Keycloak_functions.py`) raised an error that nothing caught.

**How we fixed it:** `get_mac_details` now returns `Unknown` when the lookup fails or takes more than 5 seconds, and prints why. The request carries on.

### Fixed: the Trust Engine crashed for users without login data

**What we saw:** The Trust Engine printed `TypeError: 'NoneType' object is not subscriptable` in `calculate_authentication_data_score`, and no verdict came back.

**What was wrong:** The trust score needs each user's login history from `auth_data.json`, and alice had none. Three things caused this:

1. Keycloak didn't save login events, because the realm never turned them on (`eventsEnabled`).
2. `store_keycloak_events` (`ZeroTrustWebUI/trust_signal_collection.py`) only asked Keycloak for events from 2023.
3. `process_events` skipped every event for a user who already had a row in `auth_data.json`, so a user's login history never updated after the first time.

**How we fixed it:**

1. `infra/keycloak/myrealm-realm.json` now sets `eventsEnabled` with `LOGIN` and `LOGIN_ERROR` events. We also turned this on in the running Keycloak through the admin API, so no reset was needed.
2. `store_keycloak_events` fetches recent events without a date range.
3. `process_events` adds every new event, matched by user and time.

## 2026-10-12

### Fixed: the Web UI returned the wrong verdict

**What we saw:** The verdict in the browser didn't always match the decision the Policy Engine printed. The first access request on a fresh setup crashed.

**What was wrong:** After sending the request to the Access Proxy, Flask straight away read the newest entry in `access_decision.json`. It didn't wait for the engines, so it usually returned the previous request's verdict, which could be for a different user. On a fresh setup the file didn't exist yet, so reading it crashed.

**How we fixed it:** The access request `ID` now travels through the Access Proxy, Trust Engine and Policy Engine as `request_id`, and comes back with the verdict. Node 4 (`ZeroTrustWebUI/Networking.py`) waits up to 10 seconds for the verdict with that ID (`ACCESS_DECISION_TIMEOUT` in `app.py`). If none arrives, Flask denies access with `504` and verdict `0`.

## 2026-10-11

### Added

- Added LDAP users `bob`, `charlie`, and `foo` to `infra/openldap/bootstrap.ldif`, with `Researcher`, `Administrator`, and `Policy Administrator` LDAP groups.
- Added `Researcher` and `Administrator` client roles to `infra/keycloak/myrealm-realm.json`.
- Added the `app roles` LDAP role mapper, which maps LDAP groups in `ou=groups` to `ZeroTrustPlatform` client roles of the same name.

### Removed

- Removed the unused 2023 `Keycloak/` and `OpenLDAP/` setups, replaced by `compose.yml` and `infra/`.
- Stopped tracking runtime data: `access_requests.json`, `access_decision.json`, `auth_data.json`, `events.json`, `user_data.json`, and `ZeroTrustWebUI/instance/`. They are now git-ignored and recreated by the app.

### Changed

- `.gitignore` now covers runtime data, `.DS_Store`, `.env`, and `.venv/`, and fixes the editor-backup pattern (`*~`).

### Fixed: `/logging` crashed on a fresh setup

**What was wrong:** The page opened `access_requests.json` without checking that it exists. After we stopped tracking the runtime files in git, a fresh clone has no such file until the first access request.

**How we fixed it:** The page shows an empty list when the file doesn't exist yet.

## 2026-10-05

### Added

- Added `requirements.txt` with the Python runtime dependencies used by the project:
  - Flask;
  - Flask-OIDC;
  - Flask-SQLAlchemy;
  - p2pnetwork;
  - python-keycloak;
  - PyYAML;
  - requests; and
  - tss.
- Added executable `runAll.sh` to start the Trust Engine, Policy Engine, and Access Proxy together.
  - Starts the processes in dependency order.
  - Checks that ports `8001`, `8002`, and `8003` are available.
  - Monitors child processes and stops the full group when one exits or the launcher receives a termination signal.
- Added root `compose.yml` for a portable local identity stack containing:
  - Keycloak 26.4;
  - Postgres 16;
  - OpenLDAP;
  - phpLDAPadmin; and
  - named volumes for database and directory persistence.
- Added `infra/openldap/Dockerfile` so the LDAP bootstrap LDIF is copied into the image instead of mounted from a Colima/macOS host filesystem that prevents ownership changes.
- Added `infra/openldap/bootstrap.ldif` with:
  - the `ou=users` and `ou=groups` directory structure;
  - the local LDAP user `alice`; and
  - the `Approver` LDAP group.
- Added `infra/keycloak/myrealm-realm.json` with:
  - the `myrealm` realm;
  - the confidential `ZeroTrustPlatform` OIDC client;
  - local Flask redirect URIs and web origins;
  - `Approver`, `Security Viewer`, and `Policy Administrator` client roles;
  - the native `demo` user;
  - OpenLDAP user federation; and
  - LDAP username, first-name, last-name, and email attribute mappers.
- Added `docs/how-it-works.md` with architecture, authentication, access-decision, trust-scoring, policy, PAM, persistence, bootstrap, limitation, and troubleshooting documentation.

### Changed

- Updated `AccessProxy.py`, `TrustEngine.py`, and `PolicyEngine.py` to support both interactive and background execution.
  - Interactive runs retain their `exit` prompts.
  - Background runs wait without reading from standard input.
  - `SIGTERM` triggers graceful node shutdown.
  - Trust and Policy nodes now stop in `finally` blocks.
- Updated `ZeroTrustWebUI/keycloak_config.py` so Keycloak URL, realm, client ID, client secret, and optional admin client secret can be supplied through environment variables.
- Updated `ZeroTrustWebUI/app.py` to use the shared Keycloak URL and realm configuration instead of duplicated literals.
- Updated `README.md` with:
  - Python environment setup;
  - Colima and Compose startup commands;
  - local service URLs and development accounts;
  - engine and Web UI startup commands;
  - shutdown and volume-reset commands; and
  - a link to the detailed architecture guide.

### Fixed

- Fixed Keycloak startup failures caused by an invalid realm component export structure.
  - LDAP mappers are now nested beneath the LDAP provider through `subComponents`.
  - Unsupported `parentId` fields were removed from imported mapper representations.
- Fixed LDAP-backed Keycloak authentication returning a server error because no LDAP attribute mappers were configured.
- Added both `localhost` and `127.0.0.1` Flask callback origins to the Keycloak client configuration.
- Fixed the first-login `KeyError: 'createdTimestamp'` in the Flask home route.
  - Added tolerant Keycloak user normalization for fields that may be absent from native or federated user representations.
  - Added defaults for email-verification and TOTP flags.
  - Allowed a missing creation timestamp to be represented as `null`.
- Fixed a second first-login failure risk by handling users with no `ZeroTrustPlatform` client roles instead of indexing an empty role list.
- Completed the Python dependency declaration, replacing the previous single-package requirements file.

### Runtime-Generated State

- Running the login flow appended the local `alice` and `demo` identities to `user_data.json`.
  - This is application-generated state, not a hand-authored seed change.
  - The native `demo` representation currently has no Keycloak creation timestamp, so its persisted `created_timestamp` is `null`.
- Created and exercised the local Colima Compose stack and its named Postgres/OpenLDAP volumes.
- Applied LDAP mapper and client callback updates to the live Keycloak realm while diagnosing startup and login behavior.

### Validation Performed

- Confirmed Python 3.13 resolves every package in `requirements.txt` with pip's dry-run resolver.
- Compiled all changed Python entry points successfully.
- Validated `runAll.sh` Bash syntax, executable mode, occupied-port detection, and preservation of pre-existing processes.
- Validated Compose configuration, Keycloak realm JSON, and LDAP LDIF syntax.
- Confirmed Postgres and OpenLDAP health checks pass under the Colima Docker context.
- Confirmed Keycloak OIDC discovery returns HTTP `200` at the configured realm URL.
- Confirmed phpLDAPadmin returns HTTP `200`.
- Confirmed direct LDAP search returns the seeded `alice` user.
- Confirmed both native `demo` and federated `alice` users can obtain a Keycloak token without exposing tokens in command output.
- Confirmed Flask redirects the login route to the expected Keycloak authorization endpoint.
- Confirmed the Keycloak normalization helper handles the live `alice` and `demo` admin-API payloads.
- Validated the README link, documentation headings, Markdown fences, required sections, and Mermaid block structure.

### Files Changed

#### New Files

- `compose.yml`
- `docs/changelog.md`
- `docs/how-it-works.md`
- `infra/keycloak/myrealm-realm.json`
- `infra/openldap/Dockerfile`
- `infra/openldap/bootstrap.ldif`
- `requirements.txt`
- `runAll.sh`

#### Modified Files

- `AccessProxy.py`
- `PolicyEngine.py`
- `README.md`
- `TrustEngine.py`
- `ZeroTrustWebUI/app.py`
- `ZeroTrustWebUI/keycloak_config.py`
- `user_data.json` (runtime-generated identity records)