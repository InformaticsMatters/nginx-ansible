# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

An Ansible playbook and single role (`nginx`) that deploys a placeholder
NGINX website to a Kubernetes (or OpenShift) cluster. It is designed to be
run from an [AWX] server, which injects the cluster credentials. The site is
typically an "out of service" holding page served from a ConfigMap.

## Commands

Dependencies are managed with [uv]; `pyproject.toml` is the single source of
truth and `.python-version` pins the interpreter (Python 3.14). `uv sync`
creates `.venv/` and installs everything (including the bundled
`kubernetes.core` collection via the `ansible` package).

Lint (mirrors the CI in `.github/workflows/lint.yaml`):

```bash
uv sync
uv run yamllint .
find . -type f -name '*.yaml.j2' -exec uv run yamllint {} +
```

`ansible-lint` is available as a dev dependency (config in `.ansible-lint`)
but is not run by CI:

```bash
uv run ansible-lint
```

Run the playbook (discouraged outside AWX — see Authentication below):

```bash
uv run ansible-playbook site.yaml
```

There is no test suite; CI only runs `yamllint` on push.

## Architecture

`site.yaml` is the entry playbook; it runs against `localhost` (see
`inventory.yaml`, `ansible.cfg`) and just includes the `nginx` role. All real
logic lives in `roles/nginx/`:

- `tasks/main.yaml` — orchestrator. Runs `prep.yaml`, asserts auth, then
  branches to `deploy.yaml` (when `nx_state == 'present'`) or `undeploy.yaml`
  (when `nx_state == 'absent'`). Cluster `host`/`api_key` are passed to all
  `k8s` modules via `module_defaults` for `group/k8s`.
- `tasks/prep.yaml` — loads `K8S_AUTH_HOST` / `K8S_AUTH_API_KEY` env vars into
  the `k8s_auth_host` / `k8s_auth_api_key` facts.
- `tasks/deploy.yaml` — applies templates in order: namespace, optional
  DockerHub pull secret, serviceaccount, then configmap, deployment, service,
  ingress.
- `tasks/undeploy.yaml` — deletes the namespace (which removes everything in
  it).
- `templates/*.yaml.j2` — the Kubernetes manifests, rendered with role
  variables.

### Variables

- `defaults/main.yaml` — user-facing controls: `nx_state`, `nx_namespace`,
  `nx_hostname` (must be set; the deploy asserts it is not the placeholder
  `SetMe`), `nx_content` (the served `index.html`), `nx_dockerhub_pullsecret`.
- `vars/main.yaml` — less-commonly-changed values: `nx_image_tag`, CPU/memory
  requests and limits, `nx_cert_issuer`, `wait_timeout`.

### Key behaviours

- The Deployment injects `ANSIBLE_DATE_TIME_ISO8601_MICRO` as an env var so
  every playbook run forces a pod redeployment ("bounces" NGINX).
- The DockerHub pull secret is only created when `nx_dockerhub_pullsecret` is
  non-empty, and the ServiceAccount only references it conditionally.
- The Ingress expects an `nginx` IngressClass and uses a cert-manager
  cluster-issuer named `letsencrypt-nginx-<nx_cert_issuer>`.

## Authentication

The role requires `k8s_auth_host` and `k8s_auth_api_key` to be set (asserted
in `tasks/main.yaml`). Under AWX these come from the injected `K8S_AUTH_HOST`
and `K8S_AUTH_API_KEY` environment variables. Running outside AWX requires
providing these yourself.

## Conventions

- `yamllint` config (`.yamllint`) sets `indent-sequences: false`, so YAML
  lists (`-`) align with their parent key rather than being indented. The
  `.github/` dir and a couple of upstream-copied templates are excluded.
- `ansible-lint` (`.ansible-lint`) skips the 160-char line rule and warns on
  the role-name and nested-jinja rules.

[awx]: https://github.com/ansible/awx
[uv]: https://docs.astral.sh/uv/
