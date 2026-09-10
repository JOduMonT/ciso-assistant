# ciso-assistant

Deployment configuration for [CISO Assistant](https://github.com/intuitem/ciso-assistant-community), a GRC platform, running on [Coolify](https://coolify.io).

This repo holds compose files and CI.
There is no application code here — the images come from upstream — and there is no compliance content, which lives elsewhere and is imported into a running instance.

## Run it locally

```bash
cp .env.example .env
docker compose up -d
```

Then open the URL in `CISO_ASSISTANT_URL`, `https://localhost:8443` by default.
Caddy serves a self-signed certificate locally, so expect a browser warning.

The first boot creates the database and runs migrations, which can take a couple of minutes before the backend reports healthy; the frontend and worker wait for it.

## How it deploys

Coolify deploys `docker-compose.coolify.yaml` and auto-deploys on push to `main` through its native GitHub webhook.
Point the application's domain at the **`caddy`** service: it path-routes `/api/*` to the backend and everything else to the frontend, and the reverse proxy in front maps one hostname to one service.

Required environment variables are declared as `${VAR:?…}`, so a missing value fails the deploy loudly rather than starting a misconfigured container.
See `.env.example` for what each one is for.

## Services

- **backend** — Django API, and the only writer to the database
- **huey** — background worker, the same image with a different entrypoint
- **frontend** — SvelteKit UI
- **db** — Postgres, one instance per environment with its own volume and credentials
- **qdrant** — vector store, a hard dependency of the backend rather than an optional extra
- **caddy** — the stack's single entry point, path-routing between frontend and backend

Image tags are pinned rather than floating, so a redeploy never silently changes what runs and Renovate has something to diff against.
`check-release.yml` opens a bump PR when upstream publishes a release.

## Contributing

`AGENTS.md` documents the rules that are easy to break and expensive to discover, including why the deployed compose file has no `ports:` block and why `caddy` cannot be removed.
Read it before changing either compose file.
