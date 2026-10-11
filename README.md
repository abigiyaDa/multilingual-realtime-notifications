# Real-time multilingual notifications across multiple backend workers

A write-up of a notification system I built during my backend internship.
The production code is in a private company repository, so this is a
generic description of the design, with no company code or data.

## Context
A support platform needed in-app and push notifications in English,
Amharic and Afaan Oromo, delivered in real time to individuals, groups
and everyone. Stack: FastAPI, SQLAlchemy, MySQL, Redis, WebSockets.

## Design overview
- A notification is stored once as an **event with an audience rule**
  (who it is for), not as a list of recipients.
- Per-user state (read, dismissed) is stored separately from the event.
- Text is stored in a base language plus one translation per active
  language. It is resolved once per event, not once per recipient, and
  falls back to English if a translation is missing.
- The database is the source of truth. WebSocket and push are
  best-effort delivery on top of it: a client that misses a live message
  still sees it in its feed on the next load.

## Problem 1: WebSockets live on one worker
A WebSocket connection exists only on the worker that accepted it. When
a notification was created on a different worker, it never reached the
user.

**Fix:** Redis pub/sub. Each worker runs a listener on a shared channel.
The code that creates a notification publishes the recipient IDs and the
event (or a broadcast flag), and every worker's listener delivers it to
the connections that worker holds locally. Sync request code schedules
the publish onto the app's event loop, so callers don't need to be async.

    creating worker --publish--> Redis --> every worker's listener --> local sockets --> user

## Problem 2: one row per user doesn't scale
My first design wrote one notification row per recipient. For broadcasts
this meant thousands of writes for one event.

**Fix:** shared events plus separate per-user state. Creating a
notification is now one write per event, however large the audience.
Read and dismiss state is tracked per user in its own table.

**Which notifications belong to which user?** Instead of storing
recipients, each event carries a small audience rule (an individual, a
group, a role, or everyone). A user's feed is built by matching those
rules against the user's attributes. The same rule is used to find who
should receive the live message when the event is created.

## Problem 3: duplicate alerts
Users got a push notification and an in-app alert at the same time.

**Fix:** a presence check before sending push. If the user is online,
the push is skipped. For some events the check goes one step further: a
connected user who is not looking at the relevant screen still gets a
push.

## Trade-offs
- **Simple first, migration later.** I shipped the per-user version
  first so the team could test early. It cost me a database migration
  later. I would make the same choice, but plan the migration path
  from day one.
- **Rules instead of per-user rows.** Writes are cheap, but reads do
  more work, and the audience is evaluated when a user loads their
  feed, not frozen at send time. I gave up an exact record of who a
  notification was sent to.
- **Best-effort real-time delivery.** There is no acknowledgement or
  retry for WebSocket or push, and the publish is fire-and-forget. I
  accepted that because the stored feed covers anything missed, and
  guaranteed delivery would have added a lot of complexity.
- **Per-worker online check.** The "is this user online" check looks at
  the connections held by the current worker, so a user connected to a
  different worker can still get a redundant push. A shared presence
  store would fix it; I left it out to keep the first version simple.
- **Push sent inline.** Recipients are resolved with a query and pushes
  go out in a loop with a presence lookup per recipient, inside the
  request. That is fine at the current audience sizes. At larger scale I
  would move push to a background queue and batch the presence lookups.
- **Storing every language.** Translations are stored per event and
  resolved once, so clients pick the language and the server does no
  per-user rendering. The cost is extra storage, and a missing
  translation silently falls back to English.

## Testing
I tested locally by running the backend with 4 uvicorn workers
(`--workers 4`) and connecting three WebSocket listener scripts, one per
test user, each authenticated with its own token and printing every event
it received. I then triggered notifications through the API and checked
that each listener received exactly the events meant for it, no matter
which worker handled the request. Afterwards I wrote setup documentation
for the frontend team covering how to connect, authenticate and handle
the events.

## Workload
About 30 hours a week during the internship semester, alongside classes.
I owned the schema, API, WebSocket and push layers, plus deployment debugging.
