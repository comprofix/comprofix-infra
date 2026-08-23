# comprofix-infra

Ansible playbook that provisions and deploys the Docker-based services running
on Comprofix's infrastructure: one lab host (`docker.comprofix.xyz`) and two
VPS hosts (`vps01.comprofix.com`, `vps02.comprofix.com`).

## What it does

Running `main.yml` against a host will:

- Update/dist-upgrade apt packages and install a base package set
  (`tasks/base.yml`)
- Configure `fail2ban` and install Docker (`grzegorzfranus.fail2ban`,
  `geerlingguy.docker` roles)
- Deploy and configure:
  - **Traefik** — reverse proxy / TLS termination via Cloudflare DNS-01
    (`tasks/traefik.yml`)
  - **Dozzle** — Docker log viewer, exposed behind Traefik
    (`tasks/dozzle.yml`)
  - **Portainer** — Docker/container management UI, exposed behind Traefik
    (`tasks/portainer.yml`)
- Configure and enable `unattended-upgrades`

## Inventory

Defined in `hosts.ini`:

| Group | Host(s) | Domain |
|---|---|---|
| `lab` | `docker.comprofix.xyz` | `comprofix.xyz` |
| `vps` | `vps01.comprofix.com`, `vps02.comprofix.com` | `comprofix.com` |

Group-specific vars (e.g. `traefik_domain`, and a fixed `traefik_host` for
`lab`) live in `group_vars/`.

## Requirements

Install Galaxy dependencies before running:

```bash
ansible-galaxy install -r requirements.yml
```

## Secrets / local vars

Copy `vars.yml.example` to `vars.yml` and fill in real values. `vars.yml` is
git-ignored and must never be committed — it holds:

- `traefik_api_password` — htpasswd-style `user:hash` string for the Traefik
  dashboard / basic-auth middleware
- `CF_API_EMAIL` / `CF_DNS_API_TOKEN` — Cloudflare credentials for Traefik's
  DNS-01 challenge

Do **not** set `traefik_host` in `vars.yml` — it's derived per group in
`group_vars/lab.yml` and `group_vars/vps.yml`.

## Running locally

```bash
ansible-playbook main.yml -e "@vars.yml" --limit lab   # or --limit vps
```

## CI/CD

`.github/workflows/deploy.yml` runs on every push to `main` that touches
playbook-relevant paths (`main.yml`, `hosts.ini`, `requirements.yml`,
`ansible.cfg`, `tasks/**`, `templates/**`, `group_vars/**`, or the workflow
itself):

- `deploy-lab` — runs on a self-hosted runner, targets `--limit lab`
- `deploy-vps` — runs on `ubuntu-latest`, targets `--limit vps`

Both jobs run inside the `quay.io/ansible/creator-ee` container, install
Galaxy roles, generate `vars.yml` from GitHub Actions secrets
(`TRAEFIK_API_PASSWORD`, `CF_API_EMAIL`, `CF_DNS_API_TOKEN`), run the
playbook, then delete `vars.yml`.

Required repo secrets: `SSH_PRIVATE_KEY`, `SSH_PUBLIC_KEY`,
`SSH_KNOWN_HOSTS`, `TRAEFIK_API_PASSWORD`, `CF_API_EMAIL`,
`CF_DNS_API_TOKEN`.

## Dependency updates

`renovate.json` keeps Galaxy roles/collections and container image tags
up to date automatically.
