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
- `.github/workflows/docker-image.yml` - build on PR, build and push to Docker Hub on `main`; `.github/dependabot.yml` - weekly pip/docker/actions.
- `docs/assets/` - README images only. `README.md` is the full user guide (config table, CI/CD, deployment caveats).

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
