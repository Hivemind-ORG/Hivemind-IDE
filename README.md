# Hivemind

### Your AI agents. Your code. One app.

Hivemind is a desktop app that puts AI coding agents to work on your repositories. Give an agent a task — fix a bug, build a feature, review a pull request — and watch it work in real time. Each agent gets its own isolated workspace, so multiple agents can work on different tasks simultaneously without interfering with each other.

---

## What can Hivemind do?

**Let AI agents write code for you**
Launch an AI agent, describe what you want, and let it do the work. The agent can read your code, run commands, make changes, commit, and push — all on its own.

**Review pull requests automatically**
Point an agent at a pull request and it will review the changes, leave inline comments, and submit its review — just like a human teammate would.

**Work on multiple tasks at once**
Each agent works in its own isolated branch. You can have several agents running in parallel — one fixing a bug, another building a feature, another reviewing code — all at the same time.

**Watch everything in real time**
Every agent has a live terminal stream so you can see exactly what it's doing. You also get a built-in code editor and diff viewer to inspect changes as they happen.

**Connect your Git provider**
Hivemind integrates with GitHub, GitLab, and Bitbucket. Agents can create pull requests, push branches, and interact with your repositories directly.

**Set up dev macros**
Define multi-step command sequences (start the server, run tests, lint) that your agents or you can trigger with a keyboard shortcut.

---

## Getting Started

1. **Download Hivemind** from the [latest release](https://github.com/Hivemind-ORG/Hivemind-IDE/releases/latest)
2. Install and open the app
3. Connect your Git provider (GitHub, GitLab, or Bitbucket)
4. Add a repository
5. Launch an agent and give it a task

That's it. The agent takes it from there.

---

## Requirements

- **macOS** (Apple Silicon or Intel) / **Windows** / **Linux**
- **Claude Code CLI** installed and authenticated (Hivemind uses Claude Code as the AI agent)
- A Git provider account (GitHub, GitLab, or Bitbucket) for push/PR features

---

## How It Works

When you launch an agent in Hivemind, here's what happens behind the scenes:

1. Hivemind creates a **separate git worktree** (an isolated copy of your repo on its own branch)
2. The AI agent starts up in that workspace with access to a terminal and git commands
3. The agent reads your code, makes changes, runs commands, and commits — all autonomously
4. You watch the agent's progress through a live terminal and diff view
5. When it's done, the agent can push its branch and open a pull request

You stay in control. You can watch, intervene, or let it run in the background.

---

## Features at a Glance

| Feature | Description |
|---------|-------------|
| AI Agent Launcher | Launch agents with a prompt, pick a branch, and let them work |
| Live Terminal | Watch agent activity in real time |
| Code Editor | Built-in editor with syntax highlighting |
| Diff Viewer | See exactly what changed, side by side |
| PR Reviews | Agents can review PRs with inline comments |
| Git Integration | Full git support — commit, push, pull, branch, merge |
| Multi-Provider | GitHub, GitLab, and Bitbucket |
| Macros | Multi-step command automation |
| Themes | Light, Dark, Dim, and Catppuccin Mocha |
| Auto-Updates | Stay up to date with in-app updates |

---

## Links

- [Download latest release](https://github.com/Hivemind-ORG/Hivemind-IDE/releases/latest)
- [GitHub Repository](https://github.com/Hivemind-ORG/Hivemind-IDE)
- [Report an issue](https://github.com/Hivemind-ORG/Hivemind-IDE/issues)

---

Made by Erik de Jager.
