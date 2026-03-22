# Hivemind

### Your AI agents in one clear overview.

Hivemind is a desktop app that puts AI coding agents to work on your repositories — and keeps you in the loop while they do it. Launch multiple agents at once, each tackling a different task in its own isolated workspace, and see all of their progress from a single screen. Live terminals, real-time diffs, branch comparisons, and auto-refreshing file trees give you a complete picture of everything that's happening across your codebase at a glance.

---

## What can Hivemind do?

**Let AI agents write code for you**
Launch an AI agent, describe what you want, and let it do the work. The agent can read your code, run commands, make changes, commit, and push — all on its own.

**Review pull requests automatically**
Point an agent at a pull request and it will review the changes, leave inline comments, and submit its review — just like a human teammate would.

**Run agents in parallel — and actually keep track of them**
Each agent works in its own isolated branch. You can have several agents running at once — one fixing a bug, another building a feature, another reviewing code — and Hivemind gives you a clear overview of all of them. Switch between agent workspaces with tabs, compare branches side by side, and never lose track of what's happening where.

**Watch everything in real time**
Every agent has a live terminal stream so you can follow exactly what it's doing. File trees auto-refresh as agents make changes, diffs update on the fly, and toast notifications let you know when tasks finish — even while you're focused on another agent.

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
| Parallel Agent Overview | See all running agents and their progress from one screen |
| Worktree Tabs | Switch between agent workspaces instantly |
| Live Terminal | Watch each agent's activity in real time |
| Auto-Refreshing File Trees | Files update automatically as agents make changes |
| Diff Viewer | See exactly what changed, side by side |
| Branch Compare | Compare any two branches to see the full picture |
| AI Agent Launcher | Launch agents with a prompt, pick a branch, and let them work |
| Code Editor | Built-in editor with syntax highlighting and inline search |
| PR Reviews | Agents can review PRs with inline comments |
| Global Search | Search across repos and branches to understand impact |
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
