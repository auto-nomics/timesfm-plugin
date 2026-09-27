# timesfm plugin

Migrated from the legacy `timesfm_forecast_container` wrapper in nodes-io.
One directory = one plugin family = one git-able unit.

## Layout

- `manifest.toml` — node kind `timesfm_forecast`: params, ports, image
  provenance, resources
- `Dockerfile` — image build provenance (moved verbatim from
  `containers/timesfm/`; build + push still via GHCR). Bakes the Apache-2.0
  `google/timesfm-2.5-200m-pytorch` checkpoint into
  `/opt/timesfm/checkpoint` and sha256-verifies `model.safetensors` at
  build time
- `app/` — the `timesfm_service` Python package (FastAPI HTTP skeleton plus
  the `timesfm_service.runner` CLI entrypoint the manifest invokes); the
  Dockerfile `COPY app /app`s it, so it is part of the image build
- `requirements.txt` / `requirements-dev.txt` — pinned wheels (CPU torch)
- `fixtures/forecast_request.json` — 64-point sample request; consumed by
  the ignored nodes-io E2E test (`crates/node-bundles/nodes-io/tests/
  timesfm_container.rs`), which resolves it through `NODE_PLUGINS_ROOT`
- `test_timesfm_service.sh` — image smoke test (build + `/healthz`
  contract, no checkpoint download); `root=` repointed to this directory
- `container_README.md` — the original `containers/timesfm/README.md`
  (paths and the local build tag repointed; see the staleness note below)

There is no `scripts/` directory: timesfm is a baked-runner family. The
entry point is the `timesfm_service.runner` module inside the image, so the
manifest declares only `interpreter = "python"` plus fixed argv, exactly
like the deseq2 migration.

## Offline guarantee

The wrapper's defining property is preserved unchanged: the pinned image
carries the TimesFM 2.5 checkpoint, the manifest keeps the hardened
defaults (`network` unset → isolated, `read_only_rootfs` default true,
`pull_policy` unset → missing), and the compiled env always sets
`TIMESFM_LOCAL_FILES_ONLY=true`. The node never needs network egress.

## Provenance

- Image:
  `ghcr.io/auto-nomics/autonomics/timesfm@sha256:9fa439cb6b84df08bf686e4e9e69993ec9223b2f713a6b205c2dce373ca33991`,
  tag `2.5-200m-r1`, from `Dockerfile` (base `python:3.11.11-slim`).
- Upstream: [google-research/timesfm](https://github.com/google-research/timesfm)
  (`timesfm[torch]==3.0.2`) with the Apache-2.0
  `google/timesfm-2.5-200m-pytorch` checkpoint at revision
  `1d952420fba87f3c6dee4f240de0f1a0fbc790e3`. TimesFM 3.x weights are
  NOT redistributable and are not in this image.

## Migration parity

The golden test (`crates/container-plugin/tests/timesfm_migration.rs`)
compares the compiled `ContainerCommandSpec` against the legacy Rust
wrapper (`nodes-io/src/timesfm_container.rs`): image, command, outputs,
resources, and the full `TIMESFM_*` env channel are byte-equal. Deliberate
deltas:

- **Kind rename**: `timesfm_forecast_container` → `timesfm_forecast`; the
  artifact prefix follows the kind (`/artifacts/timesfm_forecast_container`
  → `/artifacts/timesfm_forecast`), the same rule the mrpresso/deseq2
  migrations applied. DAG specs referencing the old kind must be
  regenerated.
- **Baked runner**: the legacy wrapper exec'd
  `python -m timesfm_service.runner` directly; the manifest reproduces that
  byte-exactly as `interpreter = "python"` + `argv = ["-m",
  "timesfm_service.runner"]` with no script. Unlike ldsc/mrpresso there is
  no wrapper-owned script to adapt.
- **`timeout_secs` (1800 s) and `artifact_prefix` are node-level
  constants** instead of per-instance spec params; the legacy spec accepted
  per-node overrides, the plugin DSL pins them per kind.
- **Schema surface delta**: the legacy JSON schema advertised
  `artifact_prefix` and `timeout_secs` as spec properties (harness
  metadata, never read by the runner). The plugin schema carries only
  `horizon` and `max_context`.
- **Validation**: legacy `validate()` bounds map 1:1 onto param bounds —
  `horizon` 1..=256 → `min = 1.0, max = 256.0`, `max_context` 1..=16128 →
  `min = 1.0, max = 16128.0` (both inclusive, matching the legacy
  comparisons). `timeout_secs > 0` and the absolute `artifact_prefix` check
  are loader-level node validations.
- **Static env**: `TIMESFM_CHECKPOINT`, `TIMESFM_REVISION`, and
  `TIMESFM_LOCAL_FILES_ONLY` were constants in the legacy
  `container_spec`; they appear verbatim in `[nodes.command.env]`.
  `TIMESFM_HORIZON` / `TIMESFM_MAX_CONTEXT` template the two params; the
  runner's own fallbacks (`horizon` from the request JSON, context 1024)
  are never reached because the env is always set — same as legacy.

## container_README.md staleness note

The moved container-era README predates the baked-checkpoint rework: its
prose still describes the 3.0 skeleton ("checkpoint downloaded on first
model load", `google/timesfm-3.0-pytorch` licensing guidance) while the
Dockerfile now bakes the 2.5 checkpoint at build time. The HTTP API,
multivariate request shapes, and `TIMESFM_*` service environment it
documents remain accurate for `uvicorn timesfm_service.main:app`
deployments; the DAG node path (`python -m timesfm_service.runner`) does
not use the HTTP server. Kept verbatim (modulo path/tag spellings) as
provenance.
