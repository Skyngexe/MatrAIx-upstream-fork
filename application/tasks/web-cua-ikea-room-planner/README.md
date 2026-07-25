# IKEA Room Planner (CUA — Docker Linux desktop)

MatrAIx **CUA** (computer-use) web task on a live public retail site: a persona
designs a room with IKEA's Room Planner / Home Design tool. A real **headed
Chromium** window runs in a Docker Linux desktop (Xvfb + XFCE) and
`persona-computer-1` drives it from **screenshots** (navigate / click / scroll /
type via xdotool), finishing with a **done** action after writing
`/app/output/room_plan.json` from the desktop terminal.

- URL: `https://www.ikea.com/us/en/home-design/room/?roomType=generic#1d9a5bb8-08b5-43aa-ab0c-ff91d92c95f9/0943b0b9-198c-4e74-b287-171db3f4ad35`
  (the `#<design-id>/<scene-id>` fragment is **required** — see [Notes](#notes))
- Output: `/app/output/room_plan.json`
- Environment: `application/shared-web-cua-linux`

## Why a CUA variant

IKEA's Room Planner is a heavy 3D/WebGL drag-and-drop canvas behind bot
protection. A headed desktop browser under Xvfb sends a normal Chrome
user-agent natively (clearing UA/automation-fingerprint blocks with no patch)
and the screenshot loop is closer to real end-user behaviour than DOM-only
browsing — the trade-off is that CUA runs are slower and costlier
(screenshot → model → xdotool, many steps). See
[web-interaction.md](../../web-interaction.md) § CUA for the mode comparison.

## Suggested setup (non-binding)

| Field | Value |
|-------|-------|
| Agent | `persona-computer-1` |
| Environment | `docker` (Linux Xvfb, `network_mode = "public"`) |
| Persona | `persona/datasets/bench-dev-sample/persona_0042.yaml` |
| API key | `ANTHROPIC_API_KEY` (or Bedrock: `AWS_BEARER_TOKEN_BEDROCK` + `AWS_REGION`) |

Anthropic API:

```bash
uv sync --extra computer-1
export ANTHROPIC_API_KEY=...
uv run harbor run \
  -a persona-computer-1 \
  -m anthropic/claude-sonnet-4-6 \
  --ak persona_path=persona/datasets/bench-dev-sample/persona_0042.yaml \
  -p application/tasks/web-cua-ikea-room-planner
```

Bedrock (this repo's host default — Sonnet 4.5 computer-use over a Bedrock
bearer token):

```bash
export AWS_BEARER_TOKEN_BEDROCK=...   # Bedrock API key
export AWS_REGION=us-east-1
uv run harbor run -c configs/jobs/example-job-recipe/appSim-web-cua-ikea-room-planner-bedrock.yaml
```

Oracle (reference submission; no live desktop):

```bash
uv run harbor run -p application/tasks/web-cua-ikea-room-planner -a oracle
```

## Notes

- The verifier checks the **submission schema** (≥3 products with names/prices,
  ≥1 IKEA series, valid budget/room/fit enums, ≥1 modification, ≥1 safety-
  guidance entry, a professional-boundary field, and a written reason) — not
  semantic match to live inventory, which changes over time. On success it
  emits `structured_output.json` with `task_outcome`, `web_artifact`,
  `decision`, `personalization`, `safety_guidance`, and `user_feedback`
  contexts, which `reporting.json` `contextRules` aggregate into the batch
  metrics: design personalization, budget / lifestyle fit, and safety +
  professional-boundary quality.
- CUA writes `room_plan.json` itself from the desktop terminal before finishing
  with a **done** action (no `cua_submission_profile` is needed for this custom
  schema — the profile materializers only cover the fixed decision schemas).
- **The URL fragment is required.** `?roomType=generic` on its own (no `#`)
  leaves the planner stuck on "Preparing your room ..." indefinitely — verified
  in headed Chromium: still spinning at t+45s, while the fragment URL reaches an
  interactive canvas in ~5s from a cold profile. `/home-design/room/` with no
  query behaves the same way. The fragment names the design/scene to load, so it
  must be kept verbatim wherever this task's URL appears (`instruction.md`,
  `solution/solve.sh`, the Playground registry entry).
- Known limitation: driving IKEA's 3D canvas by screenshot is demanding and
  slow (~20+ min, many steps). The oracle path emits a schema-valid reference
  submission for a completed job + batch report without a live desktop.
- Known limitation (**WebGL2**): the Computer1 runtime starts Chromium with
  `--disable-gpu` on a GPU-less Xvfb desktop, so IKEA's shaders report
  "require WebGL2, which isn't supported on this device". The persona still
  clears bot protection and can browse the live catalog for real product names
  and prices — which is what the verifier scores — but in-canvas 3D layout needs
  a GPU-backed desktop. Expect catalog-driven furnishing, not 3D manipulation,
  from live CUA runs.
