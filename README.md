# Version Control System — GitHub Clone 🐙

> A full-stack GitHub-inspired version control platform featuring a custom CLI with Git-like commands (init, add, commit, push, pull, revert), AWS S3-backed remote storage, a REST API with JWT authentication, real-time Socket.IO events, and a React frontend built with GitHub's own Primer design system.

[![React](https://img.shields.io/badge/React-18.3-61DAFB?logo=react&logoColor=white)](https://reactjs.org/)
[![Vite](https://img.shields.io/badge/Vite-5.3-646CFF?logo=vite&logoColor=white)](https://vitejs.dev/)
[![Node.js](https://img.shields.io/badge/Node.js-Express-339933?logo=node.js&logoColor=white)](https://nodejs.org/)
[![MongoDB](https://img.shields.io/badge/MongoDB-Mongoose_8-47A248?logo=mongodb&logoColor=white)](https://www.mongodb.com/)
[![AWS S3](https://img.shields.io/badge/AWS-S3-FF9900?logo=amazon-aws&logoColor=white)](https://aws.amazon.com/s3/)
[![Socket.IO](https://img.shields.io/badge/Socket.IO-4.7-010101?logo=socket.io&logoColor=white)](https://socket.io/)
[![Primer](https://img.shields.io/badge/GitHub_Primer-React-6e40c9)](https://primer.style/react/)
[![License: ISC](https://img.shields.io/badge/License-ISC-blue.svg)](https://opensource.org/licenses/ISC)

---

## Table of Contents

- [Overview](#overview)
- [Live Demo](#live-demo)
- [Features](#features)
- [System Architecture](#system-architecture)
- [Tech Stack](#tech-stack)
- [CLI Reference](#cli-reference)
- [How the VCS Works Internally](#how-the-vcs-works-internally)
- [Data Models](#data-models)
- [API Reference](#api-reference)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [Environment Variables](#environment-variables)
- [Key Engineering Decisions](#key-engineering-decisions)
- [Future Improvements](#future-improvements)

---

## Overview

This project is a ground-up implementation of a version control system and developer collaboration platform — inspired by GitHub. It has two distinct layers working in tandem:

**1. A custom CLI tool** (powered by `yargs`) that mimics core Git commands. Running `node index.js init` creates a local `.apnaGit/` directory with a staging area and commits folder. Files can be staged (`add`), snapshotted with a UUID-based commit ID (`commit`), pushed to AWS S3 (`push`), pulled back from S3 (`pull`), and reverted to any previous commit (`revert`).

**2. A full-stack web platform** with a React + Vite frontend (styled with GitHub's official Primer design system) and an Express REST API backend. Users can sign up, log in with JWT authentication, create and manage repositories, track issues, and view a GitHub-style contribution heatmap on their profile.

Both layers share the same Node.js codebase — the entry point (`index.js`) parses CLI arguments via `yargs` and either starts the HTTP server or executes a VCS command directly.

---

## Live Demo

> _Deployment link here (e.g., frontend on Vercel, backend on Render)_

---

## Features

### Custom Version Control CLI
- `init` — Initializes a `.apnaGit/` repository in the current directory with a `commits/` folder and `config.json` (stores S3 bucket name)
- `add <file>` — Copies a file into `.apnaGit/staging/` (staging area)
- `commit <message>` — Creates a UUID-named commit directory, snapshots all staged files into it, and writes a `commit.json` with the message and ISO timestamp
- `push` — Iterates all local commits and uploads every file to AWS S3 under `commits/<commitID>/`
- `pull` — Lists all objects in S3 under the `commits/` prefix and downloads them locally, reconstructing the commit history
- `revert <commitID>` — Reads the specified commit directory and copies all its files back into the working directory, effectively restoring that snapshot

### Web Platform
- **JWT Authentication** — Signup hashes passwords with `bcryptjs` (salted, 10 rounds) and returns a signed JWT; login verifies the hash and re-issues the token; token stored in `localStorage`
- **Protected Routes** — `Routes.jsx` checks `localStorage` for a valid `userId` on every navigation; unauthenticated users are redirected to `/auth`, authenticated users are bounced away from `/auth`
- **React Context for Auth State** — `AuthContext` persists `currentUser` across the app without prop drilling, initialized from `localStorage` on mount
- **Repository Management** — Create, read, update, delete repositories; toggle public/private visibility; search your repos by name (client-side, real-time filtering)
- **Issue Tracking** — Create, update (title, description, status), and delete issues tied to repositories; issues have `open` / `closed` enum status
- **User Profiles** — View username, follower/following counts, and a dynamically generated contribution heatmap
- **GitHub-style Heatmap** — Activity heatmap generated with `@uiw/react-heat-map`, with dynamically computed panel colors scaling from black → green based on the max daily activity count
- **Real-time via Socket.IO** — Users join a Socket.IO room by their `userID` on connection, enabling server-to-client push notifications (e.g., new issues, repo updates)
- **GitHub Primer UI** — Auth pages use `@primer/react` components (`PageHeader`, `Box`, `Button`, `UnderlineNav`, Octicon icons) — the same design system that powers GitHub.com

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Developer (Terminal)                         │
│                                                                  │
│   node index.js init / add / commit / push / pull / revert      │
└──────────────────────────────┬──────────────────────────────────┘
                               │ yargs CLI dispatch
                               │
          ┌────────────────────▼────────────────────┐
          │       Local Filesystem (.apnaGit/)        │
          │   staging/  │  commits/<uuid>/            │
          │             │    ├── file.txt              │
          │             │    └── commit.json           │
          └────────────────────┬────────────────────┘
                               │ push / pull
                               ▼
                    ┌──────────────────┐
                    │    AWS S3 Bucket  │
                    │  commits/<uuid>/ │
                    └──────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                      Browser (Client)                            │
│                                                                  │
│   React + Vite Frontend  (Primer UI, Axios, Socket.IO client)   │
│   /  Dashboard  |  /auth  Login  |  /signup  |  /profile        │
└──────────────────────────────┬──────────────────────────────────┘
                               │ HTTP REST + WebSocket
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│             Express.js + Socket.IO Server  (Port 3002)           │
│                                                                  │
│   ┌─────────────┐  ┌──────────────┐  ┌──────────────────────┐   │
│   │ user.router │  │ repo.router  │  │   issue.router       │   │
│   └──────┬──────┘  └──────┬───────┘  └──────────┬───────────┘   │
│          │                │                      │               │
│   ┌──────▼────────────────▼──────────────────────▼───────────┐   │
│   │                   Controllers                             │   │
│   │  userController | repoController | issueController        │   │
│   └──────────────────────────┬────────────────────────────────┘   │
└─────────────────────────────┼───────────────────────────────────┘
                              │ Mongoose ODM
┌─────────────────────────────▼───────────────────────────────────┐
│                        MongoDB Atlas                             │
│              users  |  repositories  |  issues                   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| CLI | yargs 17 | Git-like command parsing and dispatch |
| VCS Storage (local) | Node.js `fs` + `uuid` | Staging area, UUID commit snapshots |
| VCS Storage (remote) | AWS SDK v2 + S3 | Remote push/pull of commit objects |
| Backend | Node.js + Express.js | REST API server |
| Real-time | Socket.IO 4.7 | WebSocket rooms per user for live events |
| Auth | JWT + bcryptjs | Stateless token auth, salted password hashing |
| Database | MongoDB + Mongoose 8 | Document store with populated references |
| Frontend | React 18 + Vite 5 | Fast SPA with HMR in development |
| Routing | React Router v6 | Client-side routing with auth guards |
| Auth State | React Context API | Global user state without Redux |
| UI Library | GitHub Primer React | GitHub's official design system |
| Icons | Primer Octicons | GitHub's official icon set |
| Heatmap | @uiw/react-heat-map | GitHub-style contribution activity graph |
| HTTP Client | Axios | REST API calls from frontend |
| Config | dotenv | Environment variable management |

---

## CLI Reference

All commands are run from the project root via `node index.js <command>`.

```bash
# Initialize a new local repository
node index.js init
# → Creates .apnaGit/commits/ and .apnaGit/config.json

# Stage a file
node index.js add <filepath>
# → Copies <filepath> into .apnaGit/staging/

# Commit staged files with a message
node index.js commit "your commit message"
# → Creates .apnaGit/commits/<uuid>/ with all staged files + commit.json

# Push all local commits to AWS S3
node index.js push
# → Uploads every file in every commit dir to S3: commits/<uuid>/<file>

# Pull all commits from AWS S3
node index.js pull
# → Lists S3 objects under commits/ prefix and downloads them locally

# Revert working directory to a specific commit
node index.js revert <commitID>
# → Copies all files from .apnaGit/commits/<commitID>/ back to parent dir

# Start the Express + Socket.IO web server
node index.js start
```

---

## How the VCS Works Internally

### Init
Creates the hidden `.apnaGit/` directory (analogous to `.git/`) with two subdirectories — `staging/` and `commits/` — and writes a `config.json` storing the S3 bucket name from the environment.

### Add → Staging Area
`addRepo(filePath)` uses `fs.copyFile` to copy the target file into `.apnaGit/staging/`. Multiple files can be staged sequentially before committing.

### Commit → UUID Snapshot
`commitRepo(message)` generates a UUID v4 as the commit ID, creates `.apnaGit/commits/<uuid>/`, copies every file from `staging/` into that directory, and writes a `commit.json`:
```json
{
  "message": "your commit message",
  "date": "2024-01-15T10:30:00.000Z"
}
```

### Push → AWS S3
`pushRepo()` reads all commit directories, then for each file uploads it to S3 with the key `commits/<commitID>/<filename>`. This mirrors how Git pushes object packs to a remote.

### Pull → Restore from S3
`pullRepo()` calls `s3.listObjectsV2` with the prefix `commits/`, then downloads each object and writes it to the local `.apnaGit/` directory, rebuilding the commit history from the remote.

### Revert → Time Travel
`revertRepo(commitID)` reads `.apnaGit/commits/<commitID>/` and copies every file back into the parent working directory, restoring the project to that snapshot's exact state.

---

## Data Models

### User
```js
{
  username:     String (required, unique),
  email:        String (required, unique),
  password:     String,                        // bcrypt hash
  repositories: [ObjectId → Repository],
  followedUsers:[ObjectId → User],
  starRepos:    [ObjectId → Repository]
}
```

### Repository
```js
{
  name:        String (required, unique),
  description: String,
  content:     [String],                       // array of content entries
  visibility:  Boolean,                        // true = public, false = private
  owner:       ObjectId → User (required),
  issues:      [ObjectId → Issue]
}
```

### Issue
```js
{
  title:       String (required),
  description: String (required),
  status:      String (enum: ["open", "closed"], default: "open"),
  repository:  ObjectId → Repository (required)
}
```

---

## API Reference

Base URL: `http://localhost:3002`

### User Routes

| Method | Endpoint | Auth | Description |
|--------|----------|:----:|-------------|
| POST | `/signup` | — | Register user, returns JWT + userId |
| POST | `/login` | — | Authenticate, returns JWT + userId |
| GET | `/allUsers` | — | Fetch all users |
| GET | `/userProfile/:id` | — | Get user profile by ID |
| PUT | `/updateProfile/:id` | — | Update email and/or password |
| DELETE | `/deleteProfile/:id` | — | Delete user account |

### Repository Routes

| Method | Endpoint | Auth | Description |
|--------|----------|:----:|-------------|
| POST | `/repo/create` | ✅ | Create a new repository |
| GET | `/repo/all` | — | Get all repositories (populated) |
| GET | `/repo/:id` | — | Get repository by ID |
| GET | `/repo/name/:name` | — | Get repository by name |
| GET | `/repo/user/:userID` | ✅ | Get all repos for a user |
| PUT | `/repo/update/:id` | ✅ | Update content and description |
| PATCH | `/repo/toggle/:id` | ✅ | Toggle public/private visibility |
| DELETE | `/repo/delete/:id` | ✅ | Delete repository |

### Issue Routes

| Method | Endpoint | Auth | Description |
|--------|----------|:----:|-------------|
| POST | `/issue/create` | ✅ | Create an issue on a repository |
| GET | `/issue/all` | — | Get all issues |
| GET | `/issue/:id` | — | Get issue by ID |
| PUT | `/issue/update/:id` | ✅ | Update title, description, status |
| DELETE | `/issue/delete/:id` | ✅ | Delete an issue |

---

## Project Structure

```
Github-main/
│
├── backend-main/
│   ├── index.js                  # Entry point: yargs CLI dispatch + Express server
│   ├── commit.json               # Sample commit artifact
│   │
│   ├── config/
│   │   └── aws-config.js         # AWS SDK S3 client + bucket config
│   │
│   ├── controllers/
│   │   ├── init.js               # Creates .apnaGit/ + staging + config.json
│   │   ├── add.js                # Copies file to .apnaGit/staging/
│   │   ├── commit.js             # UUID commit snapshot with commit.json
│   │   ├── push.js               # Uploads commits to AWS S3
│   │   ├── pull.js               # Downloads commits from AWS S3
│   │   ├── revert.js             # Restores working dir to a commit snapshot
│   │   ├── userController.js     # Signup, login, CRUD user profiles (JWT + bcrypt)
│   │   ├── repoController.js     # Repository CRUD + visibility toggle
│   │   └── issueController.js    # Issue CRUD with open/closed status
│   │
│   ├── models/
│   │   ├── userModel.js          # User schema: repos, followedUsers, starRepos refs
│   │   ├── repoModel.js          # Repository schema: owner, issues, visibility
│   │   └── issueModel.js         # Issue schema: status enum, repo ref
│   │
│   ├── routes/
│   │   ├── main.router.js        # Aggregates all sub-routers
│   │   ├── user.router.js        # /signup /login /userProfile/:id
│   │   ├── repo.router.js        # /repo/create /repo/all /repo/toggle/:id
│   │   └── issue.router.js       # /issue/create /issue/update/:id
│   │
│   ├── middleware/
│   │   ├── authMiddleware.js     # JWT verification middleware (scaffold)
│   │   └── authorizeMiddleware.js# Authorization middleware (scaffold)
│   │
│   └── package.json
│
└── frontend-main/
    ├── index.html
    ├── package.json
    └── src/
        ├── main.jsx              # React root: wraps app in AuthProvider + BrowserRouter
        ├── App.jsx               # Root component
        ├── Routes.jsx            # Route definitions + auth guard (useEffect + navigate)
        ├── authContext.jsx       # AuthContext: currentUser state, localStorage sync
        │
        └── components/
            ├── Navbar.jsx        # Top nav with links to Dashboard and Profile
            ├── auth/
            │   ├── Login.jsx     # Login form using Primer Box/Button/PageHeader
            │   └── Signup.jsx    # Signup form using Primer components
            ├── dashboard/
            │   └── Dashboard.jsx # Repo list + search + suggested repos sidebar
            └── user/
                ├── Profile.jsx   # User profile with Primer UnderlineNav + logout
                └── HeatMap.jsx   # Contribution heatmap with dynamic color scaling
```

---

## Getting Started

### Prerequisites

- Node.js ≥ 18
- MongoDB (local or Atlas)
- AWS account with an S3 bucket configured

### 1. Clone the Repository

```bash
git clone https://github.com/Vishal985057/Version-control-system.git
cd Version-control-system/Github-main/Github-main
```

### 2. Start the Backend Server

```bash
cd backend-main
npm install
# Create .env file (see Environment Variables)
node index.js start
# Server runs on http://localhost:3002
```

### 3. Use the CLI

```bash
# In any project directory:
node /path/to/backend-main/index.js init
node /path/to/backend-main/index.js add myfile.txt
node /path/to/backend-main/index.js commit "first commit"
node /path/to/backend-main/index.js push
node /path/to/backend-main/index.js pull
node /path/to/backend-main/index.js revert <commitID>
```

### 4. Start the Frontend

```bash
cd frontend-main
npm install
npm run dev
# Runs on http://localhost:5173
```

---

## Environment Variables

Create a `.env` file inside `backend-main/`:

```env
# MongoDB
MONGODB_URI=mongodb+srv://<user>:<password>@cluster.mongodb.net/githubclone

# JWT
JWT_SECRET_KEY=your_jwt_secret_here

# AWS S3
S3_BUCKET=your-s3-bucket-name
AWS_ACCESS_KEY_ID=your_aws_access_key
AWS_SECRET_ACCESS_KEY=your_aws_secret_key
AWS_REGION=ap-south-1

# Server
PORT=3002
```

---

## Key Engineering Decisions

**Why `yargs` for the CLI instead of a separate CLI package?**
Using `yargs` inside the same `index.js` that runs the Express server lets a single codebase serve both purposes. The CLI commands (`init`, `add`, `commit`, etc.) import the exact same controller functions that could also be wired to API endpoints. This avoids duplicating business logic and keeps the project as a single deployable unit.

**Why UUID for commit IDs instead of content hashing (like Git's SHA-1)?**
Git uses SHA-1 hashes of content to enable deduplication — identical file contents produce the same object. UUID v4 is simpler to implement and avoids the need to implement a content-addressable storage layer from scratch. This is an intentional scope trade-off for a learning project; content hashing would be the production-correct approach.

**Why AWS S3 as the remote instead of a database?**
Commit objects are fundamentally binary blobs (arbitrary files). S3 is purpose-built for this — it scales infinitely, is cheap per GB, and its key-prefix structure (`commits/<uuid>/file`) maps directly to the commit directory structure. Storing binary file content in MongoDB would be an antipattern.

**Why GitHub Primer React for the UI?**
Building a GitHub clone with GitHub's own design system creates an authentic visual experience and demonstrates ability to integrate third-party component libraries. Primer's `PageHeader`, `UnderlineNav`, `Button`, and Octicon icons are production components used on GitHub.com itself.

**Why React Context over Redux for auth state?**
Auth state is a single `currentUser` value shared between `Routes.jsx`, `Navbar.jsx`, `Profile.jsx`, and the auth pages. Context's overhead is appropriate for this scope. The pattern — initializing from `localStorage` in a `useEffect`, exposing a setter through context — correctly mirrors how production apps handle token-based session persistence without a server-side session store.

**Why Socket.IO room-per-user instead of a broadcast model?**
Joining each user to their own `userID` room allows the server to send targeted notifications (e.g., "someone opened an issue on your repo") without broadcasting to all connected clients. This pattern scales correctly and mirrors how GitHub delivers per-user notifications in real time.

---

## Future Improvements

- [ ] Implement content-addressable storage (SHA-256 hashing) for true deduplication of commit objects
- [ ] Wire `authMiddleware.js` to protect all write routes with JWT verification
- [ ] Add branch support: branch creation, switching, and merging
- [ ] Implement diff computation between commits (Myers diff algorithm)
- [ ] Add pull requests with merge conflict detection
- [ ] Connect the heatmap to real commit data from the user's repositories
- [ ] Add follow/unfollow and star repository functionality (models already support it)
- [ ] Write integration tests for CLI commands (Jest + tmp directory fixtures)
- [ ] Containerize with Docker and add a `docker-compose.yml` for one-command local setup

---

_Built with Node.js, Express, MongoDB, AWS S3, Socket.IO, React, Vite, and GitHub Primer._
