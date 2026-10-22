# Relay

A workflow builder. Draw a few steps on a canvas, point a webhook or a schedule at them, and Relay runs them on its own and keeps a record of what every step did.

**Live:** https://relay-ufbk.onrender.com (the free server sleeps when idle, so the first load can take a minute)

A workflow starts with a webhook or a cron schedule and can use these steps: HTTP request, filter, transform (JSONata) and delay. Steps can branch and merge. Every run keeps the request each step sent and what came back, and the run page updates while it is running.

React, TypeScript, React Flow and Tailwind on the front. Node, Express and TypeScript on the back. PostgreSQL stores everything, including the job queue.

## Run it locally

```
npm install
npm run dev
```

Open http://localhost:5173 and make an account. With no `DATABASE_URL` set, the server starts its own Postgres in `.data/pg`, so there is nothing else to install. Try the "Order alert" template: turn it on, then copy the curl command from the webhook step.

With Docker, `docker compose up --build` starts the app, Postgres and Redis, and serves everything on http://localhost:3001.

## How it works

**The queue is Postgres.** Each step that is ready to run is one row in a `jobs` table. Workers claim a row with `SELECT ... FOR UPDATE SKIP LOCKED`, so two workers never take the same one and never wait on each other. The tests run eight workers across two engines and check that no step is ever run twice.

**Redis wakes the workers.** An idle worker used to poll the table every 250 ms, which on a free database adds up to gigabytes of traffic a month. Now `enqueue` publishes on a Redis channel and idle workers sleep until it fires. They still poll every 2 seconds (`POLL_MS`), because a delayed or retried job becomes due without anyone publishing anything. Redis is optional: without `REDIS_URL` the same wake-up happens inside the process, and with it a second instance would be woken too. If Redis is down, workers fall back to the 2 second poll. Redis only carries the nudge. The jobs stay in Postgres.

**A delay does not sleep.** The first time a worker sees a delay step it puts the job back with a `run_at` in the future and moves on. A restart in the middle of a three-day wait loses nothing, and a hundred waiting runs cost no workers.

**Crashes.** A claimed job carries a lock time. If the worker dies, the lock expires and another worker picks the job up. A worker that was only slow, and lost its lock, finds that its result is not wanted and drops it. Steps are therefore at least once: an HTTP call can repeat after a crash.

**Branches.** When a step finishes, the run row is locked, the graph is planned again, and the steps that became ready are queued. A merge step waits for every branch into it. A filter that says no skips everything behind it, and a merge still runs if another branch reached it. Removing that lock makes the tests fail, which I checked.

**Retries.** Network errors, 429 and 5xx are retried with a wait that doubles each time. A 4xx is treated as an answer: the step succeeds with `ok: false`, so a filter can react to it.

**Expressions run in worker threads.** JSONata cannot touch the host, but a recursive expression can spin forever, and it never yields to timers, so a timeout on the main thread never fires. The first version of this hung my test run for good. Expressions now run in a small pool of worker threads, and a worker that takes too long is terminated and replaced.

**SSRF.** The HTTP step lets a user type a URL, which is a way to reach things only the server can reach. Connections go through a DNS lookup that refuses loopback, private, link-local and cloud metadata addresses, so a hostname that resolves to one is caught too. Redirects are checked on every hop, and odd spellings like `http://2130706433/` are normalised first. One trap I hit: adding `::ffff:0:0/96` to Node's `BlockList` blocks every IPv4 address, so mapped addresses are unwrapped by hand.

Passwords are hashed with bcrypt, the session is an httpOnly cookie, and login, register and webhooks are rate limited. Workflows are scoped to their owner, and another user's id gets a 404.

## Limits

Run one instance. The scheduler is safe to run twice, but that has only been tested inside one process. There is no secret store, so a token typed into an HTTP header is saved in the workflow, and secret-looking headers are only masked in the run view. Steps are at least once. Run history keeps the newest 100 runs per workflow. No teams or sharing.

## Deploy

1. Create a free Postgres database on Neon (or any host) and copy the connection string.
2. In Render, create a Blueprint from this repo. It reads `render.yaml`.
3. Set `DATABASE_URL` to the connection string. `JWT_SECRET` is generated for you.
4. Optional: add a Render Key Value instance and put its internal URL in `REDIS_URL`.

The server runs migrations on start, and saves nothing in memory that matters, so a deploy is safe. The free web service sleeps when idle, and schedules do not fire while it sleeps, so point an uptime monitor at `/healthz`.

## Tests

```
npm test        # shared logic, the engine and the API against a real Postgres
npm run build
npm run e2e     # Firefox: sign up, build a workflow, run it, webhook, live updates
```

The first e2e run needs `npx playwright install firefox`.
