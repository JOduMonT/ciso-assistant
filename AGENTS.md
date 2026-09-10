# AGENTS.md — ciso-assistant

Deployment configuration for [CISO Assistant](https://github.com/intuitem/ciso-assistant-community) on Coolify.
This repo holds compose files and CI only.
It contains no application code, and it must never contain compliance content.

`CLAUDE.md` is a symlink to this file, so any coding agent lands here whatever filename its vendor looks for.
Edit `AGENTS.md`; never replace the symlink with a copy that can drift.

## This repo is public

Two rules follow from that, and neither has an exception.

- **No real domain, IP or other operator PII.**
  Placeholders stay fully generic (`https://sub.domain.tld`), not a TLD swap.
  Everything environment-specific arrives as a Coolify environment variable, declared here as `${VAR:?…}` so a missing value fails the deploy loudly instead of booting misconfigured.
- **No compliance content, ever.**
  The frameworks and control libraries are the operator's intellectual property and live in a separate private repo.
  A public repo has no private parts, and git history keeps whatever was pushed once, so a file deleted in a later commit is still public.
  Content reaches a running instance by import, never by being committed next to the compose file.

## Two compose files, one of them deployed

`docker-compose.coolify.yaml` is what Coolify deploys.
`docker-compose.yaml` is for `docker compose up -d` on a laptop and for the CI smoke test, and is never deployed.

The Coolify variant differs in three ways, each one a rule learned the hard way:

- **No `ports:` block.**
  Publishing a port binds it on the server's public IP in plaintext, bypassing Traefik, TLS and Cloudflare entirely.
  Traefik reaches containers over the external `coolify` network instead, so a host port mapping buys nothing and costs the TLS.
- **No `container_name:`.**
  Fixed names collide when Coolify runs several stacks on one host.
- **caddy serves plain HTTP on `:80`** rather than `tls internal` on `:8443`.
  Traefik terminates TLS in front of it; a second TLS layer behind it breaks routing.

## caddy is load-bearing

It is not decoration and must not be removed.
It path-routes `/api/*` to the backend and everything else to the frontend, and Traefik maps one hostname to one service.
Point Coolify's domain at the `caddy` service.

## Data lives in Postgres

The stack runs its own `db` service; upstream's default is SQLite and this deployment deliberately does not use it.
Every environment gets its own Postgres container, volume and credentials — nothing is shared between `dev` and `prod`.

One asymmetry causes real confusion, so check it before debugging a connection error.
The Django backend reads the database name from `POSTGRES_NAME` and the host from `DB_HOST`/`DB_PORT`, while the `postgres` image initialises itself from `POSTGRES_DB`.
The compose file feeds the same value to both, so set `POSTGRES_NAME` once and let it flow.
Defining `POSTGRES_NAME` is also the switch itself: the backend falls back to SQLite when that variable is absent.

Two Postgres traps this fleet has already paid for are applied here rather than rediscovered.
Postgres 18+ wants a single volume mount at `/var/lib/postgresql`, and the older `/var/lib/postgresql/data` path crash-loops the container even against an empty volume.
Its entrypoint also drops privileges from root on start, which bare `cap_drop: ALL` breaks, so it gets `CHOWN`, `FOWNER`, `DAC_OVERRIDE`, `SETUID` and `SETGID` added back and is not `read_only`.

`qdrant` is a hard dependency rather than an optional extra — the backend and huey both point at it.

## Before changing the hardening block

Every service runs `read_only`, `cap_drop: ALL`, `no-new-privileges` and a tmpfs `/tmp`, plus resource limits and log rotation.
Validate any change against a real container before it reaches a deployed environment.
`cap_drop: ALL` and `read_only` are the two that break things when guessed wrong, and they break at runtime rather than at lint time.
