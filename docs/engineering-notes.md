# Engineering notes

Longer write-ups of production problems in PrepParity: what I saw, what I suspected, what the evidence showed, and what I changed. Where my first guess was wrong I've said so, because how I got to the cause matters as much as the fix.

---

## 1. One upload pinned the whole server

**Symptom.** A single PDF upload held about 23 vCPUs for more than three minutes. Two uploads at once slowed each other to a crawl.

**Cause.** Two things compounded:

- The step that classifies a document's headings against the syllabus called the embedding model once per heading, inside a loop. A document with a hundred headings meant a hundred separate forward passes.
- PyTorch and the BLAS libraries default to one thread per core. On a host with roughly 23 cores, each of those tiny passes tried to use every core. For short strings that's slower than a few threads, and it meant one upload took the whole machine.

**Fix.**

- One batched `encode()` call before the loop (batch size 32) instead of one call per heading. I compared the new embeddings with the old ones and they matched to within about 1e-7.
- Inference threads capped at 4, set through the thread environment variables before torch is imported and again with `torch.set_num_threads()` after the model loads. The second call is needed because the environment variables only take effect if torch reads them at import time.

**Lesson.** On a many-core host, "use all the cores" is the wrong default for small inference jobs.

---

## 2. The same PDF gave 77, then 29, then 10 topics

**Symptom.** Uploading one PDF several times produced different topic counts. The summary said "5 topics and 72 more" while only 10 topic cards appeared.

**First hypothesis (wrong).** I suspected the AI fallback path that runs when a document has no usable table of contents, since an AI step can vary between runs.

**What the logs showed.** Production logs had `Done via toc: 77 topics` on every run. Extraction was deterministic, so the problem had to be later in the pipeline.

**Cause.** Persistence. Each topic was written with its own insert, and a failed insert was caught and silently ignored. The summary line was built from the full extracted list, while the topic count and the cards were read back from what had been saved. Under load, some inserts failed and the two numbers drifted apart.

**Fix.**

- Batched inserts of 12 rows, with retries, then a row-by-row fallback when a whole batch fails.
- Saved rows are matched back to their topic by segment index rather than by position, so it doesn't matter what order the database returns them in.
- A reconciliation step rebuilds the summary from what was actually saved, so the text and the cards always agree.

That took a typical upload from about 77 database round trips to about 7.

**Lesson.** Trust production logs over theory, and never swallow a write error without at least counting it.

---

## 3. Production was much slower than local

**Symptom.** Pages that loaded instantly on my laptop were slow for real users, with CPU nearly idle.

**Cause.**

- The Supabase Python client is synchronous. Calling it from inside `async def` handlers blocked the event loop, so on one worker, every user waited behind whichever query was running. Locally there's only one user, so it never showed.
- Authentication made a network call to Supabase Auth on every request to validate the token, which added a round trip to everything.

**Fix.**

- JWTs are now verified locally against the project's signing secret, with a network fallback for asymmetric-key tokens.
- Blocking database calls on the hot paths run in a thread pool (`asyncio.to_thread`), and that pool was raised from 40 to 64 threads.
- The user's plan is cached for 10 seconds per user, and every write that changes a subscription clears the entry so upgrades show up straight away.
- The HTTP connection pool was widened and kept warm over HTTP/1.1. HTTP/2 multiplexes every query onto one connection, so when Supabase closed a stale connection, every in-flight request failed together.

**Lesson.** An async framework with a sync driver is a sync server, and you only notice under concurrency.

---

## 4. Uploads: what the fix could and couldn't do

**Symptom.** Large uploads took around a minute and sometimes failed and re-sent from the start.

**What was going on.** Users are in India and the backend is in Singapore. The browser sent the whole file to the API, the API buffered it, then processed it. While a large file arrived, a worker sat busy receiving it, and long transfers could hit the proxy timeout, which made the client send everything again.

**Fix.** The browser asks the API for a signed URL and uploads directly to storage. The API then gets only the file's path and downloads it within Singapore. If the direct path fails, the client automatically falls back to the old upload.

**What it doesn't fix.** The India-to-Singapore transfer itself. Storage takes the file in its home region, so the user's upload bandwidth still sets the floor. The real gains were freeing the worker, ending the double-send, and making the second hop local. I wrote it down this way so nobody expects the change to make uploads faster on a slow connection.

**Privacy.** The PDF is deleted when the student leaves the study flow, not after a fixed timer, because the reading view still needs it while they're studying. A page refresh never deletes it, since the browser can't tell a refresh from a close. An hourly job removes anything still in storage after 24 hours, which covers closed tabs and crashes.

---

## 5. Cron jobs that would have run several times, at the wrong hour

**The scheduler runs inside the web process,** which keeps operations simple: no Redis, no separate worker. But with several worker processes, every one of them would register and fire the same jobs, sending duplicate emails and racing on writes.

**Fix.** At startup each worker tries to take an exclusive, non-blocking file lock. Exactly one gets it and owns the schedule, and the others skip scheduling. Separate replicas don't share a filesystem, so there's also a per-replica switch to turn scheduling off.

**A second bug found on the way.** The scheduler was configured with the IST timezone, but the triggers didn't inherit it, so on a UTC container every job fired five and a half hours off. Every trigger now gets the timezone explicitly.

---

## 6. Going free without leaving AI spend open

The original model used monthly AI credits and locked some features behind paid tiers. The switch was to make everything free with per-feature daily limits, plus one paid plan that only raises those limits.

**The limits engine.** One table keyed by user, feature and IST date, and one Postgres function that does an atomic `INSERT ... ON CONFLICT DO UPDATE`. Because the date is part of the key, limits reset at midnight with no job. Each metered request reserves its slot before the expensive work starts, and reads fail open, so a database hiccup never locks out a paying user.

**The cost audit.** Removing the old credit gate could have left a model call with no limit at all. I traced every call site in the codebase:

- Uploads, photo OCR, answer evaluation, mock tests, AI questioning and study-material builds each sit behind their own daily limit.
- The scoring and extraction steps inside an upload are covered by the upload limit.
- Highlights and cross-paper links are generated once per topic and then cached, so their cost is bounded by uploads.
- AI questioning is capped at three rounds per session, so a session costs a known number of calls.

**One prompt, no per-page wiring.** A single interceptor wraps `fetch` on the client. Any response with the daily-limit code opens the same upgrade prompt, and other 429s (like the 30-second upload cooldown) show their own message instead.

**A gap I found while writing this.** The slot is reserved before the work and isn't refunded if the work fails, so a failed upload still uses up one of the student's daily uploads. Refunding on failure is on the list.

---

## 7. Subscriptions on UPI AutoPay

Most Indian users pay with UPI AutoPay, not cards. Razorpay's in-place subscription update only supports cards, so it couldn't be used for upgrades or plan switches.

**Fix.** A plan change creates a new subscription whose first recurring charge is deferred to the existing renewal date. Any prorated difference is charged now as an add-on, and the old subscription is cancelled once the new one is authorised. This works on UPI AutoPay, cards and e-mandates. Switching from monthly to annual costs nothing up front. The student just authorises the new mandate.

**Promo codes** had the same problem: a Razorpay offer applies to one payment instrument only. Codes are resolved in Postgres to a percentage, and the discounted first cycle is charged as an add-on before the full-price plan starts. That works the same way on every payment method. Each account can redeem one code, ever.

**Activation is recorded twice, on purpose.** The browser sends the payment ids with Razorpay's HMAC signature, which the API verifies before granting access, so access is instant. The webhook does the same grant as a backup for when the browser closes before that call lands. Both paths are idempotent, and a status check makes sure the welcome email goes out exactly once.

---

## 8. Idle memory and the hosting bill

**Symptom.** Railway bills RAM per GB-minute. With almost no traffic, the bill kept climbing, and the usage graph sat at about 7.5 GB with CPU near zero.

**What I checked first.** I didn't assume a leak. I loaded the full app locally (every router, the model, a 256-sentence encode) and measured each stage. One process came to roughly 0.3 GB. The background-task sets remove their tasks when they finish, the one document cache is capped at 4 entries, and nothing polls in a loop. The code alone didn't add up to 7.5 GB.

**What I found.**

- The build log showed the **CUDA build of PyTorch**. With torch unpinned, Linux resolves the default wheel, which pulled roughly 3 GB of NVIDIA and Triton libraries onto a host with no GPU.
- The start command defaulted to **4 workers** when its environment variable was missing, and each worker loads its own copy of the model.
- With a 64-thread pool, glibc's default of up to 8 malloc arenas per core means memory freed after a heavy upload can stay held in per-thread arenas instead of going back to the OS. That would explain a graph that steps up after work and never comes back down.

**Fix.** CPU-only PyTorch pinned on Linux; a default of 1 worker; glibc arenas capped at 2 before any thread starts; a job every 15 minutes that returns freed heap to the OS; and diagnostics that report process memory separately from page cache, so the next reading shows where the memory actually is.

**Honest status.** I haven't measured the savings yet, so I'm not putting a number on them. The dashboard figure was also workspace-wide, so idle extra environments may account for part of it. The new diagnostics are there to answer that.

---

## 9. An outage that wasn't

**Report.** "Production is down." Uploads failed with `Failed to fetch`, side panels didn't load, and the API's health URL wouldn't open in the browser.

**How I narrowed it down.**

1. Railway showed the service online, so it hadn't simply crashed.
2. The deploy logs showed a clean boot and the model loaded, and the platform's own health check got a `200`.
3. The only request in the logs was that internal health check. If browsers were reaching the API, even a request blocked by CORS would have shown up. So requests were never arriving.
4. `dig @8.8.8.8` resolved the API hostname to an IP, while the machine's own resolver returned nothing.
5. `curl --resolve <host>:443:<ip>` forced the connection to that IP and got a `200` with the real health payload from the Singapore edge.

**Conclusion.** The server and the DNS record were both fine. The local network's resolver wasn't resolving that one hostname. Nothing in production needed to change, and three restarts and a redeploy had been aimed at the wrong layer.

**Lesson.** Rule out one layer at a time: process, platform health check, request logs, DNS, then a direct connection. Logs that show *no* requests are evidence too.

---

## What I'd do next

- **An automated test suite.** Verification today is type checks, scripted checks of specific paths, and k6 load tests. Unit and integration tests around the limits engine, payments and the upload pipeline would be the first ones.
- **An async database client,** so blocking calls don't need to go to a thread pool.
- **Refund a daily-limit slot when the work fails** (see section 6).
- **Measure the memory changes** and decide from the data whether the embedding model should load lazily.
