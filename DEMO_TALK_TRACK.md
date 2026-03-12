# WireMock Live Demo — Click & Talk Track

**Total runtime:** ~15 minutes  
**Format:** Three sequential GitHub Actions workflow runs, each building on the last

---

## Pre-demo setup checklist

Do all of this the morning of the event, not 5 minutes before.

### WireMock Cloud

**Important:** `wiremock environments create` creates **empty** mock APIs in WireMock Cloud.
It provisions the cloud resources and writes the profile file — it does **not** copy or inherit
stubs from the base mock APIs. You must seed the CI environment stubs separately (step 4 below).

- [ ] WML-IP API (`mkg1g`) exists and has the London stub
- [ ] WML-Weather API (`49o3r`) exists and has the dynamic London weather stub
- [ ] Pull the base mock APIs locally (required before creating the environment):
  ```bash
  wiremock pull mock-api mkg1g --into ip-api
  wiremock pull mock-api 49o3r --into weather-api
  ```
- [ ] Create the CI environment:
  ```bash
  wiremock environments create --profile ci
  ```
  This creates "WML-IP API [ci]" and "WML-Weather API [ci]" as **empty** mock APIs in WireMock
  Cloud, and writes their IDs into `.wiremock/wiremock-ci.yaml`. Commit that file.
- [ ] Seed the CI environment mock APIs with stubs. Read the new IDs from `wiremock-ci.yaml`
  then push the locally-pulled stubs up to each CI mock API:
  ```bash
  # Replace <CI_IP_API_ID> and <CI_WEATHER_API_ID> with the cloud_id values
  # from the generated .wiremock/wiremock-ci.yaml
  wiremock push mock-api ip-api --to=cloud:<CI_IP_API_ID>
  wiremock push mock-api weather-api --to=cloud:<CI_WEATHER_API_ID>
  ```
  After this, Stage 2 will work immediately. Stage 3 will overwrite these seeds with
  freshly recorded stubs on every run — that's intentional.

### GitHub repo
- [ ] Secret `WIREMOCK_API_TOKEN` set (Settings → Secrets and variables → Actions → **Secrets**)
- [ ] Secret `WEATHER_API_KEY` set — real WeatherAPI.com key, needed for Stage 3 recording
- [ ] Variable `WIREMOCK_PROFILE` = `ci` (Settings → Secrets and variables → Actions → **Variables**)
- [ ] All three workflow files pushed to `main`

### Your machine, day-of
- [ ] `npm run dev` running — app shows London weather card via WireMock Cloud
- [ ] Font size increased in IDE and terminal for the back row

### Browser tabs, in order
1. `http://localhost:3000` — the running app
2. GitHub repo → Code → `src/app/lib/` — for `api.ts` and `api.test.ts`
3. GitHub repo → Code → `.wiremock/` — for `wiremock.yaml` and `wiremock-ci.yaml`
4. GitHub Actions — all workflows list
5. WireMock Cloud — mock APIs list (`app.wiremock.cloud/mock-apis`)

---

## Opening — 0:00–1:15

**Click:** Switch to Tab 1 (running app). The London weather card is visible.

> "This is the weather app we're using today — a Next.js app that calls two external APIs on every page load. First, an IP geolocation API to figure out where you are. Then a weather API to fetch conditions for that location. Today it's showing London because we're already using WireMock to mock both those APIs in development."

**Click:** Switch to Tab 2 (GitHub, `src/app/lib/api.ts`).

> "The key thing to notice in the API layer is here — these two base URLs are configurable via environment variables. In dev, they point at our WireMock Cloud mock APIs. In production they'd point at the real endpoints. The fallback values are hardcoded, but the overrides mean we can redirect the app anywhere — including to a WireMock server running locally inside a CI runner."

**Click:** Open `api.test.ts` in the same directory.

> "We have three integration tests. They call the same functions the UI uses. Nothing fancy — they just verify the responses have the shape we expect. These tests need to pass in CI, and that's what today is about. How do you run them without hitting live external APIs on every commit?"

---

## Stage 1: Baseline Playback — 1:15–5:30

**Click:** Switch to Tab 4 (GitHub Actions). Click on **"Stage 1 – Baseline Playback"** in the workflow list.

> "Let me walk through this first workflow before I run it."

**Click:** Click into the YAML view. Scroll slowly through it as you speak.

> "The runner starts fresh on every job — no state, no installed tools. So the first thing we do after checking out code is install the WireMock CLI globally. Same as you'd do locally — it's just an npm package."

Point to the `Authenticate with WireMock Cloud` step:

> "Then we authenticate. The CLI stores a config token that all subsequent commands use. The token comes from a GitHub Actions secret — it never appears in logs. And yes, because the runner is ephemeral, this config step runs every single time. That's intentional."

Point to the `Pull mock APIs from WireMock Cloud` step:

> "Now the interesting bit. We pull two specific mock APIs from WireMock Cloud by their IDs — `mkg1g` and `49o3r`. The `--into` flag maps each one into a named service slot defined in our `.wiremock/wiremock.yaml` file, which locks in the local ports. Port 8080 for ip-api, 8081 for the weather API."

Point to the `Start WireMock local playback` step:

> "`wiremock run` starts both mock servers locally — in the background. We then wait until both ports are actually accepting connections before moving on. No fixed sleep, no race condition."

Point to the `Run tests against WireMock playback` step:

> "And then we run our tests with the base URLs pointed at localhost. Any non-empty string works for the API key — the stub doesn't match on it. No credentials, no network calls to external services."

**Click:** Click the **"Run workflow"** button → **"Run workflow"** to trigger it.

> "Let me kick that off."

**Click:** Switch to Tab 5 (WireMock Cloud, mock APIs list). Click into **WML-IP API**.

> "While that's running — these are the mocks that the runner is about to pull down. The IP geolocation stub returns a London response. Static data."

**Click:** Click into **WML-Weather API** → Stubs.

> "The weather stub is slightly more interesting — `last_updated` is dynamic. It uses WireMock's response templating to output the current London time, rounded to the nearest hour. So when this stub is served at 3pm, it says 3pm. At 4pm, 4pm. The rest of the weather data is static — but it's realistic enough to test against."

**Click:** Switch back to Tab 4, click into the running workflow job to watch step-by-step progress.

Wait for it to complete — it should finish green.

> "There it is. Tests pass. Against our mocks. The runner never called ip-api.com or WeatherAPI.com."

---

## Transition — 5:30–6:00

> "Now — hardcoded IDs in a workflow YAML. `mkg1g`, `49o3r`. That works fine for one environment. But in practice you have dev, staging, CI, sometimes per-team environments. If IDs are hardcoded in the YAML, you end up with multiple workflow files, or everyone editing the same one. Neither is great."

---

## Stage 2: Environment-Based Playback — 6:00–10:00

**Click:** Switch to Tab 3 (GitHub, `.wiremock/` directory). Open `wiremock.yaml`.

> "We have a `wiremock.yaml` checked into the repo. It's the base environment config — it defines our two services, their names, local ports, and the cloud IDs for the production mock APIs. This file is what makes ports deterministic across all three stages."

**Click:** Open `wiremock-ci.yaml` in the same folder.

> "And this is the CI environment profile. It's intentionally thin — it only overrides the cloud IDs. Everything else — ports, paths, service names — is inherited from the base file. This was generated by running `wiremock environments create --profile ci`, which creates dedicated mock APIs in WireMock Cloud for CI and writes this file for you."

**Click:** Switch to Tab 4. Click on **"Stage 2 – Environment-Based Playback"**.

Scroll to the pull step:

> "Here's the change from Stage 1. One line, instead of two. No IDs anywhere in the workflow."

Read it aloud:

> `wiremock pull mock-api --all --profile ${{ vars.WIREMOCK_PROFILE }}`

> "The CLI reads `WIREMOCK_PROFILE` — a GitHub repo variable — maps it to the profile overlay file in `.wiremock/`, and uses those cloud IDs to know what to pull. The workflow YAML is completely decoupled from which specific mock APIs are used."

Scroll up to point out `vars.WIREMOCK_PROFILE` vs `secrets.*`:

> "Notice it's `vars.` not `secrets.` — the profile name isn't sensitive, so it lives as a plain variable. Anyone on the team can see it and change it without touching the workflow file itself."

**Click:** Go to GitHub repo → Settings → Secrets and variables → Actions → **Variables** tab to show `WIREMOCK_PROFILE = ci`.

> "There it is. `ci`. To switch to a staging environment, you'd change this to `staging`, create a staging profile, and that's it. The workflow runs unchanged."

**Click:** Go back to Actions. Click **"Run workflow"** on Stage 2 → **"Run workflow"**.

While it runs:

> "Same test outcome, different mechanism. The value here shows up when your team grows — environment management stops being a workflow YAML problem and becomes a WireMock Cloud configuration problem. Much better place for it to live."

Wait for completion.

> "Green again."

---

## Transition — 10:00–10:30

> "There's still one problem neither of these stages solves. Stale mocks. If ip-api.com adds a field, or WeatherAPI.com changes a response format, our tests keep passing — against stubs that no longer reflect reality. We're testing against a lie. We need a way to keep the mocks fresh automatically."

---

## Stage 3: Live Re-Recording + Environment Playback — 10:30–14:30

**Click:** Switch to Tab 4. Click on **"Stage 3 – Re-Record + Environment Playback"**.

Scroll to the PHASE 1 comment block:

> "Stage 3 splits the job into two phases. Recording, then playback."

Point to the `Start recording proxies` step:

> "`wiremock record-many` reads our `wiremock.yaml` and starts a recording proxy on each service port — 8080 proxying to ip-api.com, 8081 proxying to WeatherAPI.com. It starts in the background, same SIGTERM pattern the docs recommend for non-interactive CI use."

> "The `--profile ci` flag tells it where to save the recordings. Not to the production mock APIs — to the CI environment ones."

Point to `Run tests through recording proxies`:

> "Here's my favourite part. We run the same test suite through the recording proxies. The tests call `localhost:8080` and `localhost:8081` — the proxies forward those requests to the real APIs, get real responses back, and record everything as stubs. The tests that will later validate our mocks are also what generates them. One test run, two jobs."

> "The real API key only exists here, in this phase. Only this step needs it."

Point to `Stop recording and upload stubs to WireMock Cloud`:

> "We send SIGTERM. WireMock CLI's graceful shutdown flushes all captured traffic as stubs to WireMock Cloud — into the CI environment. The log output from that process prints here so we can see exactly what was saved."

Scroll to PHASE 2:

> "Phase 2 is exactly Stage 2. Pull those freshly recorded stubs, start playback, run the tests again. Except now the stubs are seconds old."

**Click:** Click **"Run workflow"** on Stage 3 → **"Run workflow"**.

**Click:** Switch to Tab 5 (WireMock Cloud). Navigate to the CI environment mock APIs — "WML-IP API [ci]" and "WML-Weather API [ci]".

> "If we watch these while the recording phase runs, we should see the stubs update in real time as they're flushed up from the runner."

Refresh the stubs list after 30–40 seconds — you should see updated timestamps on the stubs.

**Click:** Switch back to Tab 4. Watch the job progress. When both test steps are visible, click into **"Run tests through recording proxies"** to show its timing.

> "Recording run — slightly slower, going through the proxy to live APIs."

**Click:** Click into **"Run tests against WireMock playback (validates fresh stubs)"**:

> "Playback run — faster. Pure mock. Every single CI run, fresh stubs, validated immediately."

Wait for the job to finish green.

> "Clean. Every commit now re-records from the real APIs and validates against the recording. Mocks that can't go stale."

---

## Close — 14:30–15:00

> "Three stages, three patterns."

> "Pull by ID — explicit, simple, great for getting started."

> "Pull by environment — scalable, the workflow YAML stops caring about which specific mock APIs it's using."

> "Record then play — mocks that stay honest. The recording and the validation are the same test run."

> "All of this runs in a standard GitHub Actions runner with no external infrastructure beyond WireMock Cloud. The CLI installs in seconds, authenticates with a token, and the runner tears everything down when the job ends. Nothing to maintain."

---

## If things go wrong

| Situation | What to do |
|-----------|-----------|
| Stage 1 fails on the pull step | Check that `WIREMOCK_API_TOKEN` secret is set and hasn't expired. Show `wiremock whoami` working locally as proof of concept while you investigate. |
| Stage 2 fails because CI environment has no stubs | The Stage 3 rehearsal run wasn't done. Skip forward to Stage 3 live, then re-run Stage 2 after it completes — the recording will have populated the CI environment. |
| Stage 3 fails on the recording run (live API unreachable from runner) | The runner IP may be rate-limited or blocked by ip-api.com. Fall back to re-running Stage 2 and explain Stage 3 conceptually — "this is what the recording phase does, and here's the result you'd see." |
| Workflow takes longer than expected | Use the wait time in WireMock Cloud — show the request journal, the stub detail view, walk through the dynamic templating on the weather stub. There's always more to show. |
| Any workflow is already green from an earlier run | Click the three-dot menu → **"Re-run all jobs"** to get a fresh run on screen. |
