# Trust Model

How the Trust Engine scores each access request (assignment section 3.3), and why we chose these factors and weights.

The code is in `cyberdome/trust_algorithm.py`. The weights and settings are in `policyConfiguration.yml`, and can be changed on the policy page.

## How the score works

Each request gets a trust score between 0 (no trust) and 1 (full trust). The Trust Engine scores 7 factors, each between 0 and 1, and adds them up with their weights:

```text
trust score = sum(weight of factor * score of factor) / sum(weights)
```

Dividing by the sum of the weights means the weights don't have to add up to exactly 1. If an admin sets them to 2, 2, 1, ..., the score is still between 0 and 1.

The Trust Engine also sends a **breakdown** to the Policy Engine: each factor's score, weight and a short reason. So when a request is denied for low trust, we can see which factors pulled the score down.

The configuration is read again for every request, so changes on the policy page apply straight away.

## The 7 factors

| Factor | Config key | What it measures | Scores | Weight |
| --- | --- | --- | --- | --- |
| Login history | `loginHistory` | Successful logins out of the user's last 10 login attempts (Keycloak `LOGIN` and `LOGIN_ERROR` events) | The success rate, 0 to 1. No login history: 0.5 | 0.20 |
| Device | `device` | Whether access was allowed from this device before (`new_device` in the request record) | Known: 1.0. New: 0.4 | 0.20 |
| Access frequency | `accessFrequency` | The user's requests in the last `accessFrequencyWindowMinutes` minutes (default 5) | Up to `accessFrequencyMaxRequests` (default 30): 1.0. More: 0.3 | 0.15 |
| Location | `location` | The country of the source IP, against the risk lists | Low-risk country or `LOCAL`: 1.0. Not in any list: 0.7. Medium-risk or `Unknown`: 0.5. High-risk: 0.1 | 0.15 |
| Identity strength | `identityStrength` | A verified email and MFA (TOTP) set up in Keycloak | 0.5 for each. Both: 1.0 | 0.15 |
| Time of access | `accessTime` | Whether the request is inside the night window (`periodStartInput` to `periodEndInput`) | Outside: 1.0. Inside: 0.5 | 0.10 |
| Account age | `accountAge` | How long ago the Keycloak account was created | Under 1 month: 0.4. 1 to 6 months: 0.7. Over 6 months: 1.0 | 0.05 |

The night window can cross midnight, for example 22:00 to 06:00.

`LOCAL` (localhost and private networks) counts as low risk, because in our lab that is where the users are.

## Three groups of factors

We sort the 7 factors into three groups. Each group answers a different question about the request:

| Group | Factors | Weight together | The question it answers |
| --- | --- | --- | --- |
| **Behaviour** | Login history, device, access frequency | 0.55 | How is this user acting right now? |
| **Context** | Location, time of access | 0.25 | Where and when does the request come from? |
| **Identity** | Identity strength, account age | 0.20 | How strong and how established is this account? |

**Example.** Bob logs in with his password, from his usual laptop, in the daytime, from the lab network. Behaviour: his recent logins worked, the device is known, and he made a normal number of requests. Context: the location is `LOCAL` and it's not night. Identity: his email is verified but he has no MFA, and his account is new. Now an attacker logs in with Bob's stolen password. Identity looks exactly the same, because it describes the account and not the person using it. Context can often be faked, for example with a VPN. But the attacker is on a new device, and a script makes many requests quickly, so the behaviour factors drop.

**Why three groups:**

- The names make the weights easy to explain. Behaviour gets the most weight (see "Behaviour first" below), because it is the hardest part for an attacker to fake.
- The groups follow common Zero Trust terms. NIST SP 800-207 describes the inputs to a trust decision as the subject's identity and attributes, the device's state, and environmental factors such as location and time. The ARB-ZTA paper adds behaviour, such as access frequency and authentication failures.
- One name for each factor's group, used in every document. Before this, `docs/policies.md` counted login history as an identity factor and device and access frequency as context factors, while this document counted all three as behaviour. See "Changed: the trust factors have three named groups" in `changelog.md`.

**Login history is behaviour, not identity.** It is about the account, but it measures what has happened recently (logins that worked and logins that failed), not a fixed property of the account. A burst of failed logins can change it within minutes, like the other behaviour factors.

## Why these factors and weights

We based the weights on the ARB-ZTA paper (Nahar and Andersson), which scores risk from access frequency (0.25), session duration (0.20), authentication failures (0.25), time of day (0.15) and reaction to MFA challenges (0.15). Our reasons per factor:

- **Login history (0.20).** Failed logins are a classic sign of password guessing. The paper gives authentication failures one of its two highest weights. We look at the last 10 attempts, so old failures stop counting after a while.
- **Device (0.20).** A login from a device we have never seen is the most common sign of a stolen account. An attacker with a stolen password usually doesn't have the user's device. We weight it as high as login history.
- **Access frequency (0.15).** Many requests in a short time look like a script or a scraping attack, not a person. The paper gives it one of its two highest weights. We weight it a bit lower, because our limit is simple (one window, one maximum) and could catch a fast user.
- **Location (0.15).** Requests from high-risk countries are more likely to be attacks. But an IP lookup is approximate and a VPN can change it, so we don't weight it higher.
- **Identity strength (0.15).** A verified email and MFA make it harder to take over the account. MFA is close to the paper's "reaction to MFA challenges".
- **Time of access (0.10).** Access at night, outside normal working hours, is less usual. The paper gives time of day 0.15. We use 0.10, because researchers do sometimes work late.
- **Account age (0.05).** A brand-new account has no history, so we trust it a little less. Old accounts can still be stolen, so this factor has the lowest weight.

**Behaviour first:** login history, device and access frequency together make up 55% of the score. They describe how this user behaves right now, and an attacker with stolen login details can't easily fake them.

## Example scores

| Situation | Score |
| --- | --- |
| Known device, daytime, `LOCAL`, all logins successful, verified email, no MFA, new account | about 0.90 |
| The same on a new device | about 0.78 |
| A new device, at night, from a high-risk country | about 0.59 |

The Policy Engine decides what score is enough for each resource (section 3.4).

## Factors we removed

The original CyberDome used four more inputs. We took them out of the score, and kept the old code as a commented-out function (`retired_factor_scores` in `cyberdome/trust_algorithm.py`), in case we want to bring one back.

| Removed factor | Old scores | Why we removed it |
| --- | --- | --- |
| Role (part of identity) | Policy Administrator 0.9, Approver 0.7, others 0.5 | A role says **what** a user may do, not how much we trust this request. Giving admins a higher score means a stolen admin account is trusted more, which is the opposite of what we want. Roles are used by the Policy Engine instead (section 3.4). |
| Authentication type | `code` 0.7, others 0.5 | Every browser login through Keycloak has the type `code`, so every user got the same value. It didn't tell users apart. |
| Operating system | Linux 0.5, others 0.8 | There was no reason given for treating Linux as riskier. It just penalised Linux users. |
| Device type | Mobile 0.5, desktop 0.8 | Same problem: a phone is not riskier by itself. Whether the device is **known** says much more, so the device factor uses that instead. |

## Bugs fixed in the old model

| Bug | Effect | Fix |
| --- | --- | --- |
| Account-age tiers in the wrong order | Every account older than about 4.5 days got the top score, and the middle tier was never used | The tiers are checked from the oldest down |
| The night window couldn't cross midnight | A window like 22:00 to 06:00 never matched | When the start is after the end, the check is "after the start or before the end" |
| The sign-in value could go above 1 | Each user's history started with a made-up 0, so the "trend" added up to 0.5 extra | The value is a plain success rate over the last 10 attempts, always between 0 and 1. See "Why we replaced the Markov chain sign-in score" below |
| `sign_in_risk` measured success, not risk | A higher value meant fewer failures, the opposite of the name | Renamed to `sign_in_success_rate` (and `signInRiskThreshold` to `signInSuccessThreshold`) |
| Location never matched | The record had `City/CC`, the lists had country codes. `KE` was also in both the high-risk and medium-risk lists. | The record has the country code (step 3), and each country is in one list |
| The config was read once | Changes on the policy page only applied after restarting the engines | The config is read for every request |
| Hard-coded weights | The weight settings on the policy page were saved but never used | The weights come from `policyConfiguration.yml` |

## Why we replaced the "Markov chain" sign-in score

The original CyberDome calculated the sign-in value with code it called a Markov chain (`calculate_sign_in_risk` and `predict_sign_in_risk` in `cyberdome/trust_signals.py`). We replaced it with a plain success rate over the last 10 login attempts. This section explains what the old code did, why we replaced it, and what we use now.

### What the old code did

It ran each time new Keycloak login events were processed (`process_events`). For each user it:

1. Built a list of success rates over time. The list started with a made-up `0`. After each login attempt, it added "successful logins so far divided by all attempts so far".
2. Took the trend: the last value minus the one before it.
3. Predicted the next value: the current rate plus the trend.
4. Stored the average of the current rate and the prediction as `sign_in_risk`.

Example: a user logs in successfully twice, then fails once.

| After | Rate so far | List |
| --- | --- | --- |
| start | | `[0]` (made up) |
| success | 1/1 = 1.0 | `[0, 1.0]` |
| success | 2/2 = 1.0 | `[0, 1.0, 1.0]` |
| failure | 2/3 = 0.67 | `[0, 1.0, 1.0, 0.67]` |

- Trend: 0.67 - 1.0 = -0.33 (getting worse)
- Prediction: 0.67 + (-0.33) = 0.33
- Stored value: (0.67 + 0.33) / 2 = 0.5

So one failed login dropped the value from 0.67 to 0.5. A falling trend counted extra.

### What it was meant to do

The idea is good. If a user's logins are getting worse, for example because someone is guessing their password right now, the system should react before the overall rate drops a lot. The code tried to look ahead instead of only looking back.

### Why it wasn't really a Markov chain

A Markov chain has states and probabilities of moving between them. For logins, the states could be "success" and "failure". From the history we would count, for example, how often a failure is followed by another failure. A high value is typical for password guessing.

The old code had no states and no probabilities. It only took the difference between the last two values and added it on top. The variable was called `transition_prob`, but it was a difference between two numbers, not a probability.

### Why we replaced it

1. **The value could go above 1.** Because the list started with a made-up `0`, a user's first successful login looked like a jump from 0 to 1.0. Trend: +1.0. Prediction: 1.0 + 1.0 = 2.0. Stored value: (1.0 + 2.0) / 2 = 1.5. A share of successful logins can't be more than 100%.
2. **The history was wrong.** Every entry for a user got the user's final value, not the value at that moment. So `data/auth_data.json` didn't show how the rate changed over time.
3. **The name was backwards.** `sign_in_risk` went up when logins succeeded, so a higher "risk" meant less risk.
4. **It was hard to explain and test.** The result depended on the order and number of all past events, and it didn't do what its name said.

### What we use now

The share of successful logins over the user's last 10 attempts, called `sign_in_success_rate` (`success_rate` and `add_sign_in_success_rates` in `cyberdome/trust_signals.py`).

- **Recent failures still count a lot,** which keeps the useful part of the old idea. One failure in the last 10 attempts gives 0.9 straight away.
- **Old failures stop counting** after 10 newer attempts, so a user isn't punished forever for a mistake long ago.
- **It's always between 0 and 1,** and easy to explain and test (`tests/test_trust.py`).

### Future work

A real Markov chain over "success" and "failure" would be a sound improvement. The probability that the next attempt fails, given that the last one failed, is a good sign of password guessing, and it could become its own factor or replace the login history factor.

## Limits

- The device factor relies on a cookie, which the user can delete or copy (see the Request Record section in `how-it-works.md`).
- On localhost, the location is always `LOCAL`. Tests for other countries fake the location in the test code.
- The scores per level (for example 0.4 for a new device) are our own choices. We haven't tuned them with real data.
- The access frequency limit is one fixed number for everyone. A better version would compare with each user's normal behaviour.
