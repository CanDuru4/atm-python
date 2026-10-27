# AGENTS.md

Archived beginner project: an ATM simulator from an October 2020 high-school hackathon
(Best Beginner Hack), later given a Flask web front end and a Docker/Kubernetes/Argo CD
pipeline. Public repo. Stack: Python 3.14 (Docker image), Flask 3.1.3, Docker, GitHub Actions.

## Repo map
- `ATM-Python.py` - the original 2020 terminal program (PIN login, balance, deposit, withdraw, bills). Standard library only.
- `app.py` - Flask rewrite of the same account model; state lives in the signed session cookie; one inline `TEMPLATE` string, no templates dir.
- `requirements.txt` - pinned deps for `app.py` only (`Flask==3.1.3`).
- `Dockerfile` - `python:3.14-slim`, runs `python app.py` on port 8080.
- `k8s-deployment.yaml`, `argocd-app.yaml` - Deployment + ClusterIP Service, Argo CD app (automated sync, prune, selfHeal).
- `.github/workflows/docker-image.yml` - build on PR, build and push to Docker Hub on `main`
- `design-assets/banner/` - README banner image only. `README.md` is the full user guide (config table, CI/CD, deployment caveats).

## Commands
- Setup: `python3 -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt`
- Run web app: `python3 app.py` (http://localhost:8080)
- Run CLI: `python3 ATM-Python.py` (PIN `1234`)
- Container: `docker build -t atm-python-app .` then `docker run --rm -p 8080:8080 -e FLASK_SECRET_KEY=... atm-python-app`
- Syntax check (no test suite, no linter config): `python3 -m py_compile app.py ATM-Python.py`
- Deploy: a push to `main` that touches an image path (or a manual `workflow_dispatch`) publishes `canduru04/atm-python-app:latest` and `:<sha>`. Argo CD (`argocd-app.yaml`) syncs only the manifests from `HEAD`; it does not roll pods for a new `:latest` image. Manual path: `kubectl apply -f k8s-deployment.yaml`.

## Conventions and gotchas
- `ATM-Python.py` is kept exactly as written in 2020 (globals, no docblocks, known quirks such as the PIN lockout branch never actually exiting). Do not refactor, restyle, or add docblocks to it; the global docblock rule does not apply to this file.
- `app.py` config comes only from env: `FLASK_SECRET_KEY` (random per-process fallback; required in deployment) and `FLASK_DEBUG` (off by default; must stay off anywhere reachable - the Werkzeug debugger is a remote shell). Never commit a secret key.
- `/paybill` validates the bill name against `BILLS` before using it as a session key; keep that allowlist check.
- Balance checks differ between the two programs: the CLI refuses a withdrawal/bill that would leave exactly 0 (`> 0`), the web app allows it (`>=`).
- CI path filter: only `app.py`, `ATM-Python.py`, `Dockerfile`, `requirements.txt`, and the workflow file trigger builds. Doc-only commits do not build or push.
- Actions are pinned to commit SHAs with the release tag in a trailing comment; keep that form when editing the workflow. PR builds never log in to Docker Hub; the `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` secrets are the owner's and stay as they are.
- The Deployment tracks mutable `:latest` with `imagePullPolicy: IfNotPresent` and has no resources or probes (documented in README "Deployment").

## Validation placement and reuse (owner update 2026-10-26)

- Follow the global Claude/Codex test policy and `~/Developer/personal/mac-mini/validation-economy/README.md` and `COVERAGE.md`. Focus first; full coverage only when the changed inputs require it. Docs-only policy edits need no application suite or deploy.
- Use `~/Developer/personal/mac-mini/validation-economy/remote-run.py` for every check that can run on Linux (owner rule 2026-10-27: "all the things that we can run on pc must be on pc"): tests, lint, typecheck, Node and web checks. The Mini keeps only what needs macOS or a device, each such chunk with its reason written in the gate map; this repo wires the route in its own gate, and Linux-capable work left on the Mini is a bug to fix here. It defaults to `gaming-pc` and handles connection and bounded Wake-on-LAN automatically; call it directly without a separate online check or asking the owner each time. Match the pinned runtime and locked dependencies, bound actual workers, and preserve repo data-export restrictions. Never export credentials, production data, signing keys or uncommitted work.
- The PC pool shares eight CPU slots and 24 GiB across isolated jobs with zero test swap; retain automatic job cleanup, the 8 GiB package-cache budget and 128 GiB virtual-disk cap. Brain is an explicit secondary route (`--runner brain`) when compatible, capped at two CPUs and 4 GiB together. If the PC remains unavailable after bounded wake/connect, report the blocker; never silently start the same work on the Mini or replay an uncertain submitted job.
- For work that must run on the Mini: manual heavy commands use `repo-run.py --repo <absolute checkout> --profile <profile> -- <command>` from the shared helper; choose from python / container for the actual tool. Tiny syntax checks need no reservation. Bound tool workers and reserve separate heavy phases.
- Reuse only supported receipts with matching source, coverage, dependencies, runtime/platform and relevant environment. Preserve passed unaffected batches; count remote results only when the owning gate aggregates them. Receipts do not replace required build artifacts or OS/device/browser proof. Split or offload an impossible reservation instead of waiting indefinitely; only stop your own obsolete jobs. Existing gates need explicit remote routing and matching-result aggregation; this instruction does not automatically move unintegrated commands.
