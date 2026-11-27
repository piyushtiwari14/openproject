# Vercel deployment assessment

> Status: architecture assessment only. No Vercel deployment configuration is active yet.
> Reviewed against the `dev` branch repository layout.

## Why the suggested three-service configuration is not directly applicable

OpenProject is a Ruby on Rails application with an Angular asset bundle, a Node.js Hocuspocus collaboration server, a GoodJob worker, PostgreSQL, and cache dependencies.

The Angular build is not an independent frontend application: `frontend/angular.json` writes compiled files to `public/assets/frontend`, and the Rails application serves the application pages and assets. Deploying Angular as a catch-all public service would not replace the Rails server.

The Hocuspocus service listens on port 1234 and uses `OPENPROJECT_URL` to call back into Rails for collaborative document operations. Its current README describes an `OPENPROJECT_URL` override and the requirement to configure the corresponding allowed host on the Rails side.

The Docker Compose development stack includes backend, worker, frontend, PostgreSQL, Hocuspocus, and Memcached. Production runtime configuration uses PostgreSQL, and Action Cable's production configuration uses Redis. Local disk storage is configured in `config/storage.yml`; production uploads therefore require an explicitly chosen durable storage service before a stateless deployment.

## Candidate service map (requires owner confirmation)

| Candidate name | Source root | Proposed role | Proposed exposure |
|---|---|---|---|
| `backend` | repository root | Rails HTTP application, serving Rails pages and compiled Angular assets | Public at `/` |
| `hocuspocus` | `extensions/op-blocknote-hocuspocus` | Hocuspocus collaboration HTTP/WebSocket server | Public WebSocket endpoint; exact route/prefix must be verified |
| `worker` | repository root (dedicated worker image/command required) | GoodJob background worker | Internal daemon, no public route |
| Angular build | `frontend` | Build step that emits assets for Rails | Not an independently routed service |

PostgreSQL, Redis, Memcached and durable file storage are backing services, not Vercel HTTP services. They need managed external equivalents or separate infrastructure. Database migrations and first-run initialization need an explicit deployment/release process.

## Candidate service bindings

- `hocuspocus` -> `backend`: Hocuspocus uses `OPENPROJECT_URL` for callbacks into Rails. A Vercel service binding could inject an internal backend URL, but the app's public host/allowed-host/HTTPS behavior must be configured consistently.
- `worker` -> external PostgreSQL and cache/queue services: these are connection strings, not Vercel service bindings unless the backing services are themselves deployed as Vercel services.
- `backend` -> external PostgreSQL, Redis (if Action Cable is enabled), Memcached (if configured), and object storage: these should be explicit project environment variables/managed credentials, not hardcoded hosts.

## Deployment blockers / decisions

1. Confirm whether this repo is intended to run the complete OpenProject product on Vercel, or only selected HTTP components.
2. Confirm the public collaboration path and verify that Hocuspocus WebSocket upgrades are supported through the selected routing design.
3. Select managed PostgreSQL with pgvector, Redis, cache, and durable object storage. Local container filesystems/volumes are not durable deployment storage.
4. Define Rails production image/build, runtime command, health check, secrets, migrations and release procedure. The existing production Dockerfile has multiple targets and its all-in-one target bundles PostgreSQL/Caddy/Hocuspocus, which should not be used as a stateless service without deliberate changes.
5. Define a separate GoodJob daemon image/command and queue configuration. A worker should not be exposed through public rewrites.
6. Validate Vercel plan/region/quotas, container limits, WebSocket behavior and persistent daemon availability before production use.

## Why no vercel.json was added yet

The provided example assumes Vite/Angular independent service roots and a simple public frontend catch-all. That does not match this repository's runtime topology. Adding a guessed config would expose an incomplete deployment and could silently route all application traffic away from Rails.

After confirmation of service names, public routes, backing services and deployment scope, add the Vercel config and change only the application URLs that need internal service bindings.
