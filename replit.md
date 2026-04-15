# Uptime Kuma

A self-hosted monitoring tool that tracks the uptime of websites, APIs, and services. It features a real-time dashboard, 90+ notification providers, and status pages.

## Architecture

- **Frontend**: Vue 3 + Vite, communicates via Socket.io WebSocket
- **Backend**: Node.js + Express + Socket.io, serves the built frontend
- **Database**: SQLite (stored in `data/` directory), with Knex.js migrations
- **Port**: Backend runs on port 5000 (via `UPTIME_KUMA_PORT=5000`)

## Running the App

The app is configured with a single workflow:
- **Start application**: `UPTIME_KUMA_PORT=5000 node server/server.js`
- The backend serves the built frontend from the `dist/` directory

## Development Notes

- On first run, Uptime Kuma shows a database setup wizard to configure the SQLite database
- After setup, users create an admin account and can start adding monitors
- Vite is configured with `allowedHosts: true` and `host: "0.0.0.0"` for Replit proxy compatibility
- Frontend is pre-built into `dist/` — run `npm run build` after frontend changes

## Key Files

- `server/server.js` - Main backend entry point
- `server/config.js` - Port/host configuration
- `config/vite.config.js` - Vite build configuration
- `src/` - Vue 3 frontend source
- `data/` - Runtime data directory (SQLite DB, uploaded files)
- `db/knex_migrations/` - Database migrations

## Deployment

- Target: VM (needs persistent state for SQLite and WebSocket)
- Build command: `npm run build`
- Run command: `node server/server.js` (uses PORT env var for port 5000)
