# Policies

How the Policy Engine decides each access request (assignment section 3.4), and why the policies look the way they do.

The code is in `cyberdome/policy.py`. The settings are in `policyConfiguration.yml` and on the policy page.

## What the Policy Engine gets

For every request to a protected page, the Policy Engine gets:

- **the request record** from the Web UI (see "Request Record" in `how-it-works.md`): user, roles from the token, resource, action, authentication status, location, device ID, `new_device`, time
- **the trust score and its breakdown** from the Trust Engine (see `trust-model.md`)
- **the user's sign-in success rate**: the share of successful logins over their last 10 attempts

## The 6 policies

The policies are checked in this order. The first one that denies decides. If none denies, P6 allows.

| ID | Policy | Rule | Result | Outcome in the log |
| --- | --- | --- | --- | --- |
| P1 | Invalid authentication | The record says the user is not authenticated, or no login is recorded for the user | Deny | `AUTHENTICATION_FAILURE` |
| P2 | Unauthorized resource | The resource is not in the resource table, or none of the user's roles is allowed it, or the request changes something (`write`) and none of the user's roles may write | Deny | `POLICY_VIOLATION` |
| P3 | Risky context | A high-risk country (everyone), a medium-risk country (privileged roles), the night window (privileged roles, or anyone on an admin resource), or too many failed logins | Deny | `DENY` |
| P4 | Admin resource | An admin resource needs a trust score of at least `adminMinTrustScore` (0.75) | Deny if lower | `DENY` |
| P5 | Low trust | The trust score is below the resource's `minTrustScore`, or there is no trust score | Deny | `DENY` |
| P6 | Permitted access | None of the policies above denied | Allow | `ALLOW` |

Every decision says which policy decided and why. For example:

```text
DENY   P2  Role Researcher may not access resource_2 (allowed: Administrator)
DENY   P2  Role Administrator may view policy_configuration but not change it (may change: Policy Administrator)
DENY   P3  The request comes from KE, a medium-risk location, and Administrator is a privileged role
DENY   P5  Trust score 0.67 is below the minimum of 0.75 for resource_2. Lowest factors: device 0.4 (new device), accessFrequency 0.3 (45 requests in the last 5 minutes (more than 30))
ALLOW  P6  Role Researcher may access resource_1, and trust score 0.90 meets the minimum of 0.6
```

### How each policy uses identity and context

The task says "Policies must consider both identity/role and request context". We made sure that each policy does, not only the policies together:

| Policy | Identity or role | Request context |
| --- | --- | --- |
| P1 | The user and their login history | The authentication status of this request |
| P2 | The user's roles | The resource, and the action (`read` or `write`) |
| P3 | Whether the user has a privileged role, and their recent logins | Location, time of day, and whether the resource is an admin resource |
| P4 | The admin role (P2 has checked it) and the identity factors of the trust score (identity strength, account age) | The behaviour factors (login history, device, access frequency) and context factors (location, time of access) of the trust score |
| P5 | The identity factors of the trust score | The behaviour and context factors of the trust score, and the resource's minimum |
| P6 | All of the above | All of the above |

The factor groups (behaviour, context, identity) are explained in "Three groups of factors" in `trust-model.md`. The behaviour factors sit in the request context column, because they describe what is happening around this request right now: which device sends it, how many requests came just before it, and whether the recent logins worked. The identity factors describe the account itself.

In the first version of the policies, P2 only looked at the resource, and P3 never looked at the role. "Why P2 checks the action" and "Why P3 depends on the role" below explain how we changed that.

## Settings

Per resource, in the `resources` table of `policyConfiguration.yml`:

| Resource | Allowed roles (may read) | `writeRoles` (may change) | Admin resource | `minTrustScore` |
| --- | --- | --- | --- | --- |
| `resource_1` (transaction data) | Researcher, Administrator | nobody | No | 0.6 |
| `resource_2` (access tokens, sensitive) | Administrator | nobody | No | 0.75 |
| `pam_approval` | Approver | Approver | No | 0.75 |
| `policy_configuration` | Administrator, Policy Administrator | Policy Administrator | Yes | 0.6 |
| `logging` | Administrator, Security Viewer | nobody | Yes | 0.6 |

A resource without `writeRoles` can't be changed by anyone (fail closed).

Global settings:

| Setting | Default | Used by |
| --- | --- | --- |
| `adminMinTrustScore` | 0.75 | P4 |
| `highRiskLocations` | `KP` | P3 (everyone) |
| `mediumRiskLocations` | `KE` | P3 (privileged roles) |
| `privilegedRoles` | Administrator, Policy Administrator, Approver | P3 |
| `periodStartInput`, `periodEndInput` (night window) | 00:00 to 06:00 | P3 |
| `signInSuccessThreshold` | 0.5 | P3 |

The minimum scores and `adminMinTrustScore` can be changed on the policy page. The allowed roles, `writeRoles` and `privilegedRoles` can only be changed in the file, so the page can't be used to give a role more access.

## Why the policies look like this

### Why the Policy Engine makes every decision

Before this change, the Web UI checked the role table itself (`require_role`), and only sent allowed requests to the engines. So a request like bob on Resource 2 was refused by the Web UI and never reached the Policy Engine, and it wasn't logged as a decision.

Now the Web UI only checks the login (it needs a valid token to build the request record), and the Policy Engine decides everything else. This follows the Zero Trust architecture in NIST SP 800-207: the Policy Engine is the one place that decides, and the Web UI only enforces the decision. It also means every refused request has a decision with a policy ID and a reason, which the audit log (section 3.5) needs.

### Why P2 checks the action (read or write)

**Before:** P2 only asked "may this role access this resource?". Viewing the policy page and saving new policies were the same thing to P2. Anyone who could see the policies could also change them.

**Now:** each resource has two lists. `allowedRoles` may read it, and `writeRoles` may change it. A `write` request (an HTTP POST, for example saving the policy page or approving a PAM request) needs a role in `writeRoles`.

**Why:**

- **The action is part of the request context.** With it, P2 uses the role and the request (what the user wants to do), not only the role and the page.
- **Least privilege.** The task and NIST SP 800-207 both ask for the least access needed. Reading and changing are different powers, and most people only need to read.
- **Separation of duties.** An Administrator runs the system and may view the policies, for example to understand why a request was denied. But only a Policy Administrator may change them. So one stolen Administrator account can't both use the system and turn off its protections.
- **Fail closed.** A resource without `writeRoles` can't be changed by anyone. A new resource is read-only until someone decides who may change it.

**Example:** charlie (Administrator) opens the policy page: allowed (P6). charlie saves new policies: denied by P2. foo (Policy Administrator) saves them: allowed.

### Why P3 depends on the role

**Before:** P3 denied the same risky contexts for everyone: a high-risk country, an admin page at night, or many failed logins. The user's role made no difference, so P3 was only about context.

**Now:** some rules only apply to privileged roles (`privilegedRoles`: Administrator, Policy Administrator, Approver):

| Context | Normal roles (for example Researcher) | Privileged roles |
| --- | --- | --- |
| High-risk country (`KP`) | Denied | Denied |
| Medium-risk country (`KE`) | Allowed (the trust score is lower) | Denied |
| Night window, normal resource | Allowed (the trust score is lower) | Denied |
| Night window, admin resource | Denied | Denied |
| Too many failed logins | Denied | Denied |

**Why:**

- **The risk depends on who is asking.** A stolen privileged account can do much more damage: change policies, approve privileged access, read access tokens. So a context that is only "a bit unusual" for a researcher is a reason to stop an administrator.
- **This is what "identity and context together" means.** The same context gives a different answer for a different role. For example, bob (Researcher) from `KE` is allowed, and charlie (Administrator) from `KE` is denied.
- **Normal users aren't blocked for small risks.** For them, a medium-risk country or night-time only lowers the trust score, and P5 decides.

**Which roles are privileged:** the roles that can change the system or approve access for others. Security Viewer only reads the logs, so it isn't privileged. Admin pages are still denied at night for everyone, because they hold the policies and the audit log.

### Why the minimum trust is per resource, not per role

The original Policy Engine had one threshold per role: Approver 0.9, Policy Administrator 0.6, Security Viewer 0.4. Researcher and Administrator had none, so they were never denied for low trust.

We moved the minimum to each resource:

- **Zero Trust ties access to how sensitive the resource is** (NIST SP 800-207). A user who is fine for low-risk data shouldn't automatically get sensitive data. With per-role thresholds, an Administrator needed the same score for every page.
- **One table answers both questions:** who may access each resource, and how much trust it needs.
- **No role is forgotten.** Every resource has a minimum, so every request is checked against one.

The old function is kept as a commented-out function (`retired_make_access_decision` in `cyberdome/policy.py`) for comparison.

### Why this order

Rules that don't depend on the trust score come first (P1 to P3), then the rules that do (P4 and P5). That way the reason is always the clearest one. For example, a request from a high-risk country is denied by P3 with "KP is a high-risk location", not by P5 with "trust score too low".

### Why P3 denies outright, while the trust score only lowers the score

Some risks are clear enough to deny on their own: a high-risk country, admin work at night, or many failed logins. Others are only a bit unusual, like a new device. Small risks lower the trust score (section 3.3), and the score decides. Clear risks deny through P3, whatever the score.

### Why the admin minimum is 0.75, and a new device alone still passes

A device only becomes "known" after access was allowed from it once. If admin pages refused every new device, a Policy Administrator (who may only open admin pages) could never make a device known, and would be locked out for good.

With 0.75, a normal admin on a new device (about 0.78) still gets in. A new device plus one more risk (for example failed logins or a burst of requests) drops below 0.75 and is denied. A real system would ask for MFA in this case instead (step-up authentication). See Future work.

### Why "no login recorded" counts as invalid authentication

The trust score and P3 use the user's recent logins from Keycloak's events. If there is no login at all for a user who has a token, something is wrong (for example Keycloak events are switched off, or the token came from somewhere else). We can't check the login history, so we deny.

## Scenarios from the task

| Scenario | Example | Decided by |
| --- | --- | --- |
| 1. Valid request | bob (Researcher) opens Resource 1 on a known device | ALLOW, P6 |
| 2. Unauthorized resource | bob (Researcher) opens Resource 2 | DENY, P2 (`POLICY_VIOLATION`) |
| 3. Invalid or expired authentication | A revoked or fake token | `401` from the Web UI before the engines. P1 if the record says unauthenticated |
| 4. Low trust | bob opens Resource 1 after a burst of requests from a new device, or with Resource 1's minimum raised on the policy page | DENY, P5 (or P3 for a high-risk country) |
| 5. Admin on an admin resource | charlie (Administrator) opens the policy page, foo (Policy Administrator) saves it | ALLOW, P6 (after P4 passes). charlie saving is denied by P2 |

## Limits and future work

- **Step-up MFA.** For medium risk, a real system would ask for a second factor (for example an OTP in Keycloak) instead of denying. This is the adaptive authentication our charter is about. It would fit between P4 and P5.
- **MFA for admins.** Requiring MFA for admin resources would be stronger than a minimum score. Our demo users don't have MFA set up.
- **The device rule is weak.** A device becomes known after one allowed request. A real system would register devices, for example through device management (MDM).
- **Invalid tokens never reach the Policy Engine.** The Web UI refuses them with `401`, because it can't build a request record without a valid token. The audit log (section 3.5) records these as `AUTHENTICATION_FAILURE` from the Web UI.
