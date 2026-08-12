---
name: Pull the attendance report for a webinar Session
description: >-
  Pull the participant and attendance report for one Demio webinar Session through the
  Public Demio API, resolving the Session ID correctly and handling the personal data it
  returns.
api: openapi/demio-openapi.yml
provider: demio
operations:
  - pingViaHeaders
  - listEvents
  - getEvent
  - getEventSession
  - listSessionParticipants
generated: '2026-08-12'
method: generated
---

# Pull the attendance report for a webinar Session

Base URL: `https://my.demio.com/api/v1`

## The identifier that trips everyone up

Attendance is reported **per Session, not per Event**. The Session identifier is
`date_id` — what the Demio UI and Help Center call the *Session ID*. `id` (the Event ID)
and `date_id` are both bare integers with no prefix, so passing the wrong one does not
raise a type error; it returns `404 {"messages":["Event Date not found"]}` or, worse,
silently reports on a different Session. Resolve `date_id` explicitly every time.

## Before you start

Send the account key pair as headers from Demio **Settings > API**:

```
Api-Key: <key>
Api-Secret: <secret>
```

Never the `api_key`/`api_secret` query-string form — it leaks the secret into logs.

## Steps

1. **Verify the credential.** Call `pingViaHeaders` (`GET /ping`). `200` with
   `{"pong": true, ...}` means you are authorized. `401` = bad credentials, `403` =
   inactive Demio account.

2. **Find the Event.** Call `listEvents` (`GET /events`). Add `?type=past` when you are
   reporting on a webinar that has already run — that is the common case for attendance.
   The whole collection comes back at once; there is no pagination.

3. **Resolve the Session ID.** Call `getEvent` (`GET /event/{id}`) and read the `dates[]`
   array. Each entry is `{date_id, status, timestamp, datetime, zone}`. For attendance,
   take the Date whose `status` is `finished`. For a Series Event there will be several —
   pick by `timestamp`, not by position. Optionally confirm with `getEventSession`
   (`GET /event/{id}/date/{date_id}`).

4. **Pull the report.** Call `listSessionParticipants`
   (`GET /report/{date_id}/participants`). Filter with `?status=` using exactly one of the
   published values — `attended`, `did not attend`, `completed`, `left early`, `banned`
   (note the spaces; these are not slugs). Omit it to get everyone.

   The response is:

   ```json
   {"participants": [
     {"email": "...", "name": "...",
      "custom_fields": [{"id": "last_name", "name": "Last Name", "value": "Doe"}],
      "attended": true, "status": "completed"}
   ]}
   ```

   `attended` is a boolean and `status` is the finer-grained outcome; a `completed`
   participant and a `left early` participant both have `attended: true`. Do not compute
   attendance rate from `status` alone.

5. **Handle the empty case.** A Session with no participants returns
   `{"participants": []}` with a `200`. That is a valid answer, not an error — report it
   as zero, do not retry.

## This response is personal data

Every row carries a real person's name, email address and whatever they typed into the
registration form. Treat the payload as PII: do not write it to logs, do not paste it into
a chat transcript, and return only the fields the operator asked for. The API offers no
read-by-person, no update and no delete, so nothing here can be used to satisfy a data
subject request — that goes through Demio support.

## Rate limits and response size

180 requests/minute; 100 calls/day on a free trial, 5,000/day for a paying customer,
resetting 00:00 UTC and **shared with the account's Zapier usage**. There is no `429`
documented and no `RateLimit-*`/`Retry-After` header, so you cannot see how much quota is
left from a response.

The participant list is unpaginated with no published ceiling, so a 3,000-seat webinar
returns 3,000 records in one body. Budget for that, and note that building a cross-webinar
attendance history means Events → Dates → participants, one call per Session, against that
same daily quota.

## Errors

| Status | Meaning |
|---|---|
| 401 | `Authorization failed` — wrong key/secret |
| 403 | `Account is not active` |
| 404 | `Event not found` — wrong Event ID |
| 404 | `Event Date not found` — wrong `date_id`, or you passed an Event ID |

Errors are `{"messages": ["..."]}` with no stable machine-readable code. Branch on the
status.
