# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A MERN stack app that serves the classic 2048 game and persists high scores to MongoDB. The Express server serves both the API and static assets from a single port (default 5000).

## Commands

### Server (from `server/`)
```bash
npm install          # install dependencies
npm start            # production: node server.js
npm run dev          # development: nodemon server.js (auto-reload)
```

### Client (from `client/`)
```bash
npm install          # install dependencies
npm start            # dev server on port 3000 (proxies /api/* to port 5000)
npm run build        # build to client/build/ (required before running server in prod)
npm test             # run React tests
```

## Architecture

The app has two distinct frontend layers served by the same Express server:

1. **Vanilla JS game** (`client/public/`): The original 2048 game served as static files. The game logic lives entirely in `client/public/js/` — `game_manager.js` orchestrates everything, `grid.js` handles the board state, `html_actuator.js` renders to the DOM, and `local_storage_manager.js` persists game state client-side. Score submission happens via a form in `client/public/scores.html`.

2. **React app** (`client/src/`): A thin React wrapper built with Create React App. `App.js` sets up routing; the `/game` route embeds the vanilla game in an iframe, and `/highscores` displays the leaderboard by calling `GET /api/scores`.

**Server** (`server/`):
- `server.js`: Express entry point — serves the React build from `client/build/`, registers `/api/scores` routes, exposes Prometheus metrics at `/metrics` (via `express-prom-bundle`, excluded from path-based metrics)
- `routes/scores.js`: `GET /api/scores` returns top 10 scores sorted descending; `POST /api/scores` accepts `{ name, score }` and saves to MongoDB
- `models/Score.js`: Mongoose schema with `name`, `score`, `date` fields

MongoDB reconnects automatically every 10 seconds on failure (`connectTimeoutMS: 5000`).

## Docker

Multi-stage build: compiles the React client in stage 1, then copies the build into the production server image.

```bash
docker build -t 2048-mern .
docker run -p 5000:5000 -e MONGO_URI=mongodb://user:pass@host:27017/dbname 2048-mern
```

`MONGO_URI` must be passed at runtime — it is not baked into the image.

## Environment

The server reads from `server/.env`:
```
PORT=5000
MONGO_URI=mongodb://user:pass@host:27017/dbname
```

`MONGO_URI` must be set for the app to function. The client dev proxy (`"proxy": "http://localhost:5000"` in root `package.json`) routes API calls to the server during development.
