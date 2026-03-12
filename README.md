<h1 align="center">Hivemind IDE</h1>

<p align="center">
  An AI-agent-centric desktop IDE that lets AI agents autonomously write code, fix bugs, and review pull requests — all inside sandboxed Docker containers with real-time terminal streaming and full Git integration.
</p>

<p align="center">
  <a href="https://github.com/Hivemind-ORG/Hivemind-IDE/releases/latest">Download</a>
</p>

---

## Features

### AI Agent Workspace

- **Launch AI agents** that work autonomously inside isolated Docker containers
- **Real-time terminal streaming** — watch agents think, code, and debug live via xterm.js
- **Built-in agent templates** — Bug Fixer, Code Reviewer, and Feature Builder, each with tailored system prompts
- **Custom templates** — create and save your own agent instructions
- **Macro system** — record and replay agent workflows with a single shortcut

### Code Editor

- Full-featured editor powered by **CodeMirror 6** with syntax highlighting for 20+ languages (TypeScript, Python, Rust, Go, Java, C++, and more)
- Integrated file tree with drag-and-drop support
- Theme customization with preset system

### Git Integration

- Complete Git workflow powered by **libgit2** — branches, commits, diffs, and merge conflict resolution
- **GitHub and GitLab** support with OAuth token management and secure credential storage via OS keyring
- Pull request creation, review, and auto-approval workflows
- Each agent gets its own **git worktree**, enabling safe parallel work without conflicts

### Docker Sandboxing

- Every agent runs in its own Docker container built from a custom base image (Ubuntu 22.04 + Node 20 + Git + Python 3)
- Dynamically allocated ports (10000–19999) and dedicated PTY streams per agent
- Agents communicate back to the host via a local HTTP API and WebSocket connection

## Architecture

Hivemind IDE is built on three communication layers:

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Frontend ↔ Backend** | Tauri v2 IPC | React UI invokes Rust commands, receives typed events |
| **Agents ↔ Backend** | Axum HTTP API + WebSocket | Agents in Docker call back to host for git ops, status updates, PR reviews |
| **Backend ↔ Git Providers** | GitHub / GitLab REST APIs | OAuth-authenticated remote operations |

## Tech Stack

| Component | Technologies |
|-----------|-------------|
| **Frontend** | React 18, TypeScript (strict), Vite, Tailwind CSS, Zustand, CodeMirror 6, xterm.js |
| **Backend** | Rust, Tauri v2, Axum 0.7, Tokio, SQLite (WAL mode), Bollard (Docker), git2 |
| **Containers** | Docker — custom Ubuntu 22.04 base image with Node 20, Git, Python 3 |
| **Security** | OS-native keyring (Apple Keychain / Windows Credential Manager / Linux Secret Service) |

## Supported Platforms

| Platform | Architecture |
|----------|-------------|
| **macOS** | Apple Silicon (arm64), Intel (x86_64) |
| **Linux** | x86_64 (Ubuntu 22.04+) |
| **Windows** | x86_64 (Windows 10+) |

All platforms support automatic updates via GitHub Releases.

## Prerequisites

- [Node.js](https://nodejs.org/) 18+
- [Rust](https://rustup.rs/) (latest stable)
- [Docker](https://www.docker.com/) (for running agents)
- Tauri v2 system dependencies ([see platform-specific requirements](https://v2.tauri.app/start/prerequisites/))
