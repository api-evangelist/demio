---
name: Register an attendee and return their unique join link
description: >-
  Register a person for a Demio webinar Event through the Public Demio API and return the
  unique join link that Demio issues for them, without creating duplicate registrations.
api: openapi/demio-openapi.yml
provider: demio
operations:
  - pingViaHeaders
  - listEvents
  - getEvent
  - registerForEvent
generated: '2026-08-12'
method: generated
---

# Register an attendee and return their unique join link

Base URL: `https://my.demio.com/api/v1`

## Before you start

Authorization is an account-wide key/secret pair from Demio **Settings > API**
(`https://my.demio.com/manage/api-details`). Send them as headers:

```
Api-Key: <key>
Api-Secret: <secret>
```

Do **not** use the `api_key`/`api_secret` query-string form. It works, and it puts the
account secret into proxy logs, server access logs and browser history.

There are no scopes. The key carries full account authority — reading every Event and
every registrant's personal data, and creating registrations that email real people.

## Steps

1. **Verify the credential and find out whether it is a sandbox key.** Call
   `pingViaHeaders` (`GET /ping`). A `200` returns `{"pong": true, "sandbox": <bool>}`.
   Read `sandbox`. If it is `false` you are about to write to production and email real
   people — stop and confirm with the operator before continuing. A `401` means the
   credentials are wrong; a `403` means the Demio account is not active and no API call
   will fix it.

2. **Resolve the Event.** If you were given an Event ID, skip to step 3. Otherwise call
   `listEvents` (`GET /events?type=upcoming`) and match on `name`. The response is the
   whole collection — there is no pagination and no total count, so do not attempt to
   page it. Each entry carries `id`, `date_id` (the next Session), `status`, `timestamp`,
   `zone` and `registration_url`.

3. **Pick the Session.** Call `getEvent` (`GET /event/{id}?active=true`) to read the
   `dates[]` array and choose the `date_id` you want. If you omit `date_id` at
   registration, Demio uses the nearest active Date — acceptable for a single-session
   Event, wrong for a series where the person asked for a specific date. A `404`
   (`{"messages":["Event not found"]}`) means the Event ID is wrong or is not visible to
   this key.

4. **Check for a duplicate before you write.** `registerForEvent` has **no idempotency
   key**, and Demio does not document whether re-registering the same email returns the
   original join link or creates a second registrant. Before registering, call
   `listSessionParticipants` (`GET /report/{date_id}/participants`) and look for the
   email. If it is already there, return the existing record and do not register again.

5. **Register.** Call `registerForEvent` (`PUT /event/register`) with
   `Content-Type: application/json`:

   ```json
   {
     "id": 1,
     "date_id": 35,
     "name": "John",
     "email": "john.doe@example.com",
     "last_name": "Doe"
   }
   ```

   Use `ref_url` (the Event registration page URL) instead of `id` when you have the link
   but not the ID. Predefined optional fields are `last_name`, `company`, `website`,
   `phone_number`, `gdpr`. Custom form fields are passed as additional properties keyed by
   the field's Unique Identifier from the Event's Registration block — those identifiers
   live in the Demio UI, not in the contract, so ask the operator for them rather than
   guessing.

6. **Persist the result.** A `201` returns:

   ```json
   {"hash": "...", "join_link": "https://event.demio.com/join/..."}
   ```

   The `hash` is the only handle on that registrant and it is returned **once**. There is
   no read-by-email endpoint, so if you do not store it the join link is unrecoverable
   through the API. Write it to the caller's system before reporting success.

## Errors

Errors are `{"messages": ["..."]}` — human-readable strings with no stable code. Branch on
the HTTP status, not the text.

| Status | Meaning | What to do |
|---|---|---|
| 400 | Validation failed — e.g. `Name must be more than 2 symbols`, `Wrong Email format` | Fix the fields. All failed rules come back together. |
| 400 | `Bad Request syntax. Try to check request Body data JSON format.` | Your JSON is malformed. |
| 401 | `Authorization failed` | Wrong key/secret. |
| 403 | `Account is not active` | Billing/account state. Not fixable via the API. |
| 404 | `Event not found` | Wrong Event ID. |

## Rate limits

180 requests/minute, and a daily account quota of 100 calls on a free trial or 5,000 for a
paying customer, resetting at 00:00 UTC. That quota is **shared with the account's Zapier
usage**. Demio publishes no `429`, no `Retry-After` and no `RateLimit-*` headers, so you
cannot read remaining quota from a response — budget your calls, back off on any
unexplained error, and do not poll.

## Do not

- Retry a failed `PUT /event/register` blindly. Without an idempotency key, a retry can
  create a duplicate registrant and send the person a second confirmation email.
- Register anyone without the operator's explicit confirmation. This operation contacts a
  third party.
