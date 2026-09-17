# ATM Python

<p align="center">
  <a href="https://github.com/CanDuru4/atm-python/actions/workflows/docker-image.yml"><img src="https://img.shields.io/github/actions/workflow/status/CanDuru4/atm-python/docker-image.yml?branch=main&label=docker%20image" alt="Docker image build status"></a>
  <img src="https://img.shields.io/badge/python-3.14-blue" alt="Python 3.14">
  <img src="https://img.shields.io/badge/flask-3.1.3-black" alt="Flask 3.1.3">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-green" alt="MIT license"></a>
</p>

A beginner ATM simulator written in October 2020 for a high-school computer
science hackathon, and later retrofitted with a small web front end and a
container pipeline. Two programs sit side by side in this repository: the
original menu-driven terminal program (`ATM-Python.py`), kept as it was
written, and a Flask rewrite of the same account model (`app.py`) that is built
into a Docker image by GitHub Actions and deployed to Kubernetes through
Argo CD. It is meant for people learning Python, and for anyone who wants a
tiny self-contained app to practise a CI/CD pipeline on.

Nothing here talks to a real bank. Balances and bills are hardcoded starting
values that live in memory (CLI) or in the browser session (web app).

> **Context:** High-school computer science hackathon, October 2020, Best Beginner Hack.

<p align="center">
  <a href="https://canduru.net">
    <img src="docs/assets/canduru-banner.png" alt="Can Duru - one line of code at a time" width="221" height="90">
  </a>
</p>

## Features

**Terminal program - `ATM-Python.py`**

- PIN login with three attempts before the account locks
- View balance
- Deposit cash
- Withdraw cash, refused when the balance would not cover it
- Pay electric, gas and water bills from a repeating bill menu

**Web app - `app.py`**

- Single-page ATM served by Flask, with the balance and the three bills held in
  a signed session cookie
- Deposit and withdraw forms, with insufficient-balance handling
- Bill payment for the same three utilities, with the selection validated
  server-side
- No login screen: the web app is the transaction menu only

**Delivery**

- `Dockerfile` producing a `python:3.14-slim` image that listens on port 8080
- Kubernetes `Deployment` + `Service` manifests and an Argo CD `Application`
- GitHub Actions pipeline that builds on every pull request and publishes to
  Docker Hub from `main`
- Dependabot watching pip, Docker and GitHub Actions dependencies

## Tech stack

Python 3.14 · Flask 3.1.3 · Docker · Kubernetes · Argo CD · GitHub Actions ·
Dependabot

## Getting started

### Prerequisites

- Python 3.11 or newer for a local run. The container image and CI build on
  Python 3.14, which is the version the app is verified against. Run the
  project with Python 3, **not** Python 2.7 - the code uses Python 3 syntax
  and will not run on 2.7.
- Docker, only if you want to build or run the container image.

### Installation

#### Run the web app locally

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python3 app.py
```

Then open <http://localhost:8080>.

#### Run the web app in Docker

```bash
docker build -t atm-python-app .
docker run --rm -p 8080:8080 -e FLASK_SECRET_KEY="$(python3 -c 'import secrets; print(secrets.token_hex(32))')" atm-python-app
```

### Configuration

| Name | Required | Purpose |
| --- | --- | --- |
| `FLASK_SECRET_KEY` | No locally, yes in deployment | Signs the session cookie. When it is unset a random key is generated per process, so sessions reset on restart and do not work across replicas. |
| `FLASK_DEBUG` | No | Set to `1` to enable the Werkzeug debugger. Off by default, and it must stay off anywhere the port is reachable, because the debugger is a remote shell. |

## Usage

Run the original terminal program:

```bash
python3 ATM-Python.py
```

The PIN is `1234`. This login exists in the terminal program only.

The web app at <http://localhost:8080> is the transaction menu only: deposit, withdraw,
and pay the electric, gas or water bill.

## Project structure

```
.
├── ATM-Python.py              # 2020 hackathon program, terminal menu
├── app.py                     # Flask rewrite of the same account model
├── requirements.txt           # Python dependencies for app.py (pinned)
├── Dockerfile                 # python:3.14-slim image, serves on :8080
├── k8s-deployment.yaml        # Kubernetes Deployment + ClusterIP Service
├── argocd-app.yaml            # Argo CD Application, automated sync
├── docs/assets/               # README images
└── .github/
    ├── dependabot.yml         # weekly pip / docker / github-actions updates
    └── workflows/
        └── docker-image.yml   # build on PR, build and push on main
```

## Deployment

```bash
kubectl apply -f k8s-deployment.yaml
```

`argocd-app.yaml` registers the repository with Argo CD and syncs the manifests
automatically, with `prune` and `selfHeal` enabled.

Two things to know before pointing this at a real cluster: the Deployment
tracks the mutable `:latest` tag with `imagePullPolicy: IfNotPresent`, so a node
that already holds an older `:latest` will not pick up a new build - use the
commit-SHA tag for anything you want to roll deterministically. And the
Deployment sets no resource requests, limits, or health probes.

### CI/CD

The pipeline lives in `.github/workflows/docker-image.yml`.

- **Triggers.** Pushes to `main`, pull requests targeting `main`, and manual
  `workflow_dispatch` runs. The push and pull request triggers are filtered to
  the paths that can change the image: `app.py`, `ATM-Python.py`, `Dockerfile`,
  `requirements.txt` and the workflow file itself.
- **Pull requests build only.** The Docker Hub login step is skipped and
  `push` is `false`, so a pull request - including one from a fork - never
  reaches the registry credentials. A broken `Dockerfile` still fails the check.
- **`main` builds and pushes.** The image is published as
  `canduru04/atm-python-app:latest` and `canduru04/atm-python-app:<commit sha>`,
  so every build is also addressable by the commit that produced it.
- **Actions are pinned to commit SHAs**, with the release tag in a trailing
  comment. A retagged upstream release cannot silently change what runs in CI.
  Dependabot's `github-actions` ecosystem opens a pull request when a new
  release appears, and rewrites both the SHA and the comment.
- **Least privilege.** The workflow declares `permissions: contents: read` at
  the top level, so the automatic `GITHUB_TOKEN` cannot write to the repository.

Repository secrets used by the pipeline, by name: `DOCKERHUB_USERNAME` and
`DOCKERHUB_TOKEN`. They are only read on `main` and manual runs.

## License

MIT. See [LICENSE](LICENSE). Copyright (c) 2020-2026 Can Duru.

## Author

Can Duru — [canduru.net](https://canduru.net)
