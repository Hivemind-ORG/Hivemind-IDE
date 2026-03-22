# Hivemind Plugin Development Guide

This guide is for third-party developers who want to create, test, and distribute plugins for the Hivemind IDE. You do not need the Hivemind source code to build plugins -- everything here works with the released desktop app.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Quick Start](#2-quick-start)
3. [Plugin Manifest](#3-plugin-manifest)
4. [Building a UI Plugin](#4-building-a-ui-plugin)
5. [Building a Backend Plugin](#5-building-a-backend-plugin)
6. [Creating Agent Scripts](#6-creating-agent-scripts)
7. [Testing Your Plugin](#7-testing-your-plugin)
8. [Importing Plugins](#8-importing-plugins)
9. [Publishing to the Marketplace](#9-publishing-to-the-marketplace)
10. [Tips and Best Practices](#10-tips-and-best-practices)
11. [References](#11-references)

---

## 1. Introduction

Hivemind plugins extend the IDE with custom functionality. A plugin is a directory containing a `manifest.json` and any combination of three types of extensions:

| Type | What it does | File(s) |
|------|-------------|---------|
| **UI** | Adds a sidebar panel or bottom panel rendered as HTML in an iframe | `ui/sidebar.html`, `ui/panel.html` |
| **Backend** | Runs JavaScript in a sandboxed QuickJS runtime on the Rust backend -- listens to hooks, registers HTTP routes, emits events | `main.js` |
| **Scripts** | Shell scripts that agents can call from inside their Docker containers | `scripts/*` |

You can use any combination of these in a single plugin. A plugin that only adds a sidebar is valid. A plugin that only registers backend hooks is valid. Mix and match as you like.

---

## 2. Quick Start

### Minimum viable plugin

Create a directory and add two files:

```
my-hello-plugin/
  manifest.json
  ui/
    sidebar.html
```

**manifest.json**
```json
{
  "id": "my-hello-plugin",
  "name": "Hello Plugin",
  "version": "0.1.0",
  "description": "A minimal plugin that says hello.",
  "author": "Your Name",
  "hivemind_version": ">=0.9.0",
  "ui": {
    "sidebar": "ui/sidebar.html"
  },
  "sidebar_icon": "waving_hand",
  "sidebar_label": "Hello"
}
```

**ui/sidebar.html**
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <style>
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', system-ui, sans-serif;
      background: transparent;
      color: #d4d4d8;
      font-size: 12px;
      padding: 12px;
    }
    h2 { margin: 0 0 8px; font-size: 14px; }
    .info { color: #71717a; }
  </style>
</head>
<body>
  <h2>Hello from my plugin!</h2>
  <p class="info">Hivemind version loading...</p>

  <script>
    hivemind.getSystemInfo().then(function(info) {
      document.querySelector('.info').textContent =
        'Running on ' + info.platform + ', API port ' + info.api_port;
    });
  </script>
</body>
</html>
```

That is it. Zip it up, import it, and you will see a hand-wave icon in the sidebar.

### Full working example

Here is a more complete plugin that combines all three extension types. It tracks agent launches, exposes an HTTP route for agents, and shows a count in the sidebar.

```
agent-tracker/
  manifest.json
  main.js
  ui/
    sidebar.html
  scripts/
    track-event
```

**manifest.json**
```json
{
  "id": "agent-tracker",
  "name": "Agent Tracker",
  "version": "1.0.0",
  "description": "Tracks agent launches and displays a counter in the sidebar.",
  "author": "Your Name",
  "hivemind_version": ">=0.9.0",
  "main": "main.js",
  "ui": {
    "sidebar": "ui/sidebar.html"
  },
  "sidebar_icon": "monitoring",
  "sidebar_label": "Tracker",
  "scripts": [
    "scripts/track-event"
  ],
  "hooks": [
    "agent:launched",
    "agent:completed"
  ],
  "routes": [
    "/plugins/agent-tracker/*"
  ]
}
```

**main.js**
```js
hivemind.log("Agent Tracker loaded");

// Tell agents how to use our script
hivemind.registerInstructions(
  "The Agent Tracker plugin is installed.\n" +
  "You can report custom events:\n" +
  "  hivemind-plugin-agent-tracker-track-event EVENT_NAME\n"
);

// Count agent launches
hivemind.on("agent:launched", function(data) {
  var raw = hivemind.storage.get("launch_count");
  var count = raw ? parseInt(raw, 10) : 0;
  count++;
  hivemind.storage.set("launch_count", String(count));
  hivemind.emit("count-updated", { count: count });
});

hivemind.on("agent:completed", function(data) {
  hivemind.emit("agent-done", { agent_id: data.agent_id });
});

// HTTP route for agents
hivemind.registerRoute("/plugins/agent-tracker/*", function(req) {
  if (req.path === "/plugins/agent-tracker/count") {
    var raw = hivemind.storage.get("launch_count");
    return { status: 200, body: { count: raw ? parseInt(raw, 10) : 0 } };
  }
  return { status: 404, body: { error: "Unknown route" } };
});
```

**ui/sidebar.html**
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <style>
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', system-ui, sans-serif;
      background: transparent;
      color: #d4d4d8;
      font-size: 12px;
      padding: 12px;
    }
    .count {
      font-size: 36px;
      font-weight: bold;
      color: #a78bfa;
      margin: 16px 0;
    }
    .label { color: #71717a; font-size: 10px; text-transform: uppercase; }
  </style>
</head>
<body>
  <div class="label">Agent Launches</div>
  <div class="count" id="count">0</div>

  <script>
    // Load initial count from storage
    hivemind.storage.get("launch_count").then(function(raw) {
      if (raw) document.getElementById("count").textContent = raw;
    });

    // Update in real-time when backend emits
    hivemind.on("count-updated", function(data) {
      document.getElementById("count").textContent = data.count;
    });
  </script>
</body>
</html>
```

**scripts/track-event** (make this file executable)
```sh
#!/bin/sh
# hivemind-plugin-agent-tracker-track-event — Report a custom event
#
# Usage:
#   hivemind-plugin-agent-tracker-track-event "build_complete"

set -e

EVENT="${1:?Usage: hivemind-plugin-agent-tracker-track-event EVENT_NAME}"

curl -sf -X POST "${HIVEMIND_API_URL}/plugins/agent-tracker/log" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${HIVEMIND_API_TOKEN}" \
  -d "{\"event\":\"${EVENT}\",\"agent_id\":\"${HIVEMIND_AGENT_ID}\"}"
```

---

## 3. Plugin Manifest

The `manifest.json` file is the only required file. It tells Hivemind everything about your plugin.

### All fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `id` | Yes | `string` | Unique plugin identifier. Must be **lowercase kebab-case** (e.g. `my-cool-plugin`). No underscores, no uppercase, no spaces. |
| `name` | Yes | `string` | Human-readable display name (e.g. `My Cool Plugin`). |
| `version` | Yes | `string` | Semver version string (e.g. `1.0.0`, `0.2.3`). |
| `description` | No | `string` | Short summary of what the plugin does. Shown in the plugin list. |
| `author` | No | `string` | Your name or organization. |
| `hivemind_version` | No | `string` | Minimum Hivemind version required (e.g. `>=0.9.0`). |
| `main` | No | `string` | Path to the backend JavaScript entry point (e.g. `main.js`). Runs in QuickJS. |
| `ui` | No | `object` | UI entry points. |
| `ui.sidebar` | No | `string` | Path to the sidebar HTML file (e.g. `ui/sidebar.html`). |
| `ui.panel` | No | `string` | Path to the panel HTML file (e.g. `ui/panel.html`). |
| `sidebar_icon` | No | `string` | A [Material Symbols](https://fonts.google.com/icons?icon.set=Material+Symbols) icon name (e.g. `bug_report`, `monitoring`, `rocket_launch`). |
| `sidebar_label` | No | `string` | Short label shown under the sidebar icon (e.g. `Debug`). |
| `scripts` | No | `string[]` | List of paths to agent scripts (e.g. `["scripts/my-tool"]`). |
| `hooks` | No | `string[]` | List of hook events this plugin listens to. |
| `routes` | No | `string[]` | List of HTTP route patterns this plugin handles. |

### ID format rules

- Must be lowercase
- Must use hyphens to separate words (kebab-case)
- Must not start with `hivemind-` (reserved for official plugins)
- Good: `my-plugin`, `code-stats`, `pr-notifier`
- Bad: `MyPlugin`, `my_plugin`, `hivemind-custom`

### Version format

Use semantic versioning: `MAJOR.MINOR.PATCH` (e.g. `1.0.0`).

- Bump `PATCH` for bug fixes
- Bump `MINOR` for new features (backward compatible)
- Bump `MAJOR` for breaking changes

### Available hooks

These are the hook event names you can listen to:

| Hook | Fires when |
|------|-----------|
| `app:started` | Hivemind IDE finishes loading |
| `agent:launched` | An agent starts running |
| `agent:completed` | An agent finishes successfully |
| `agent:stopped` | An agent is manually stopped |
| `agent:error` | An agent encounters an error |
| `git:commit` | A commit is made (by agent or user) |
| `git:push` | A branch is pushed |
| `git:merge` | A merge is performed |
| `git:rebase` | A rebase is performed |
| `git:pr_created` | A pull request is created |
| `log` | A log message is emitted |

---

## 4. Building a UI Plugin

UI plugins render HTML inside a sandboxed iframe in the Hivemind sidebar or bottom panel. The iframe has access to the `window.hivemind` bridge API, which lets you interact with the IDE.

### File placement

- **Sidebar**: `ui/sidebar.html` -- appears in the left sidebar when the user clicks your plugin icon
- **Panel**: `ui/panel.html` -- appears in the bottom panel area

You can have both if you want.

### The `window.hivemind` bridge API

Inside your HTML, a global `hivemind` object is automatically injected. You do not need to import anything. All methods are available immediately.

#### Core

```js
// Show a toast notification
hivemind.toast("Hello!", "info");      // types: "info", "success", "error"

// Get system info
hivemind.getSystemInfo().then(function(info) {
  // info.api_port, info.platform, info.user_agent
});

// Listen for events emitted by your backend (main.js)
hivemind.on("my-event", function(data) {
  console.log("Received:", data);
});
```

#### Storage (persistent key-value)

```js
// Save data (scoped to your plugin)
hivemind.storage.set("my-key", "my-value");

// Read data
hivemind.storage.get("my-key").then(function(value) {
  console.log(value); // "my-value"
});

// Delete a key
hivemind.storage.delete("my-key");
```

#### Repositories

```js
// List all repositories
hivemind.repositories.list().then(function(repos) {
  // repos = [{ id, name, local_path, ... }, ...]
});

// Get a specific repository
hivemind.repositories.get(repoId).then(function(repo) { ... });

// List branches for a repo
hivemind.repositories.getBranches(repoId, repoPath).then(function(branches) { ... });
```

#### Agents

```js
// List all agent sessions
hivemind.agents.list().then(function(sessions) {
  // sessions = [{ id, branch, status, provider, ... }, ...]
});

// Get a specific agent
hivemind.agents.get(agentId).then(function(session) { ... });

// Launch an agent
hivemind.agents.launch({
  repoId: "repo-uuid",
  branch: "feature/my-branch",
  originBranch: "main",        // optional
  prompt: "Fix the login bug"
});

// Stop an agent by branch name
hivemind.agents.stop("feature/my-branch");

// Stop, destroy, or restart by agent ID
hivemind.agents.stopById(agentId);
hivemind.agents.destroy(agentId);
hivemind.agents.restartById(agentId, true);
hivemind.agents.restartWithPrompt(agentId, "New instructions").then(function() { ... });
```

#### Git

```js
hivemind.git.log(repoPath, branch, 20).then(function(commits) { ... });
hivemind.git.getDiff(repoPath, branch).then(function(diff) { ... });
hivemind.git.getAheadBehind(repoPath, branch).then(function(result) {
  // result = { ahead: 3, behind: 1 }
});
hivemind.git.getUncommittedFiles(worktreePath).then(function(files) { ... });
hivemind.git.blame(repoPath, filePath, lineNumber).then(function(info) { ... });
hivemind.git.commit(worktreePath, "commit message").then(function(result) { ... });
hivemind.git.push(repoPath, branch).then(function() { ... });
hivemind.git.pull(repoPath, branch).then(function() { ... });
hivemind.git.createBranch(repoPath, "new-branch", "main").then(function() { ... });
hivemind.git.checkout(repoPath, branch).then(function() { ... });
hivemind.git.fetch(repoPath).then(function() { ... });
hivemind.git.revert(worktreePath).then(function() { ... });
```

#### Files

```js
hivemind.files.read(filePath).then(function(content) { ... });
hivemind.files.list(dirPath).then(function(entries) { ... });
hivemind.files.exists(filePath).then(function(exists) { ... });
hivemind.files.search(dirPath, "query").then(function(results) { ... });
hivemind.files.write(filePath, content).then(function() { ... });
hivemind.files.delete(filePath).then(function() { ... });
hivemind.files.create(filePath).then(function() { ... });
hivemind.files.mkdir(dirPath).then(function() { ... });
hivemind.files.rename(oldPath, newName).then(function() { ... });
```

#### Pull Requests

```js
hivemind.pr.list(repoId).then(function(prs) { ... });
hivemind.pr.get(repoId, prNumber).then(function(pr) { ... });
hivemind.pr.getFiles(repoId, prNumber).then(function(files) { ... });
hivemind.pr.create(repoId, "PR title", "PR body", "my-branch", "main").then(function(pr) { ... });
```

#### Other APIs

```js
// Settings
hivemind.settings.get("key").then(function(value) { ... });
hivemind.settings.getAll().then(function(all) { ... });
hivemind.settings.getTheme().then(function(theme) { ... });

// Clipboard
hivemind.clipboard.read().then(function(text) { ... });
hivemind.clipboard.write("copied text");

// Navigation
hivemind.navigate.toSettings("plugins");
hivemind.navigate.toRepo(repoId);

// View state
hivemind.view.getFocusState().then(function(state) {
  // state = { isFocusMode, repoId, branch, repoPath, activeTab }
});

// Terminal
hivemind.terminal.spawn("/path/to/dir").then(function(sessionId) { ... });
hivemind.terminal.send(sessionId, "ls -la\n").then(function() { ... });
hivemind.terminal.kill(sessionId).then(function() { ... });

// Other read-only APIs
hivemind.system.getStats().then(function(stats) { ... });
hivemind.macros.list(repoId).then(function(macros) { ... });
hivemind.repoGroups.list().then(function(groups) { ... });
hivemind.plugins.list().then(function(plugins) { ... });
hivemind.ai.getProvider().then(function(config) { ... });
```

### Styling for dark theme

The Hivemind IDE uses a dark theme. Your iframe should match it. Here are the recommended CSS patterns:

```css
body {
  /* Transparent background so it blends with the IDE */
  background: transparent;

  /* Standard text colors matching the IDE palette */
  color: #d4d4d8;

  /* System font stack */
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', system-ui, sans-serif;

  /* Match the IDE's compact sizing */
  font-size: 11px;
  line-height: 1.5;
}

/* Muted text for labels and secondary info */
.muted { color: #71717a; }

/* Borders */
.border { border: 1px solid rgba(255, 255, 255, 0.08); }

/* Buttons */
.btn {
  background: none;
  border: 1px solid rgba(255, 255, 255, 0.1);
  color: #a1a1aa;
  font-size: 10px;
  padding: 3px 8px;
  border-radius: 4px;
  cursor: pointer;
}
.btn:hover {
  color: #fff;
  border-color: rgba(255, 255, 255, 0.25);
}

/* Accent color (purple) */
.accent { color: #a78bfa; }

/* Status colors */
.success { color: #4ade80; }
.error   { color: #f87171; }
.warning { color: #fbbf24; }
```

### Example: a dashboard that lists repos and agents

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <style>
    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', system-ui, sans-serif;
      background: transparent;
      color: #d4d4d8;
      font-size: 11px;
      padding: 12px;
    }
    h3 { font-size: 11px; color: #71717a; text-transform: uppercase; margin: 12px 0 4px; }
    .item { padding: 4px 0; border-bottom: 1px solid rgba(255,255,255,0.05); }
    .name { font-weight: 500; }
    .meta { font-size: 9px; color: #52525b; font-family: monospace; }
    .dot {
      display: inline-block; width: 6px; height: 6px;
      border-radius: 50%; margin-right: 4px; vertical-align: middle;
    }
    .dot-running { background: #22c55e; }
    .dot-idle { background: #60a5fa; }
    .dot-stopped { background: #52525b; }
    .dot-error { background: #ef4444; }
    .empty { color: #3f3f46; padding: 8px 0; }
  </style>
</head>
<body>

  <h3>Repositories</h3>
  <div id="repos"><div class="empty">Loading...</div></div>

  <h3>Active Agents</h3>
  <div id="agents"><div class="empty">Loading...</div></div>

  <script>
    // Load repositories
    hivemind.repositories.list().then(function(repos) {
      var el = document.getElementById("repos");
      if (!repos.length) { el.innerHTML = '<div class="empty">No repositories</div>'; return; }
      el.innerHTML = "";
      repos.forEach(function(repo) {
        var div = document.createElement("div");
        div.className = "item";
        div.innerHTML =
          '<div class="name">' + repo.name + '</div>' +
          '<div class="meta">' + (repo.local_path || repo.id) + '</div>';
        el.appendChild(div);
      });
    });

    // Load agents
    hivemind.agents.list().then(function(sessions) {
      var el = document.getElementById("agents");
      var active = sessions.filter(function(s) { return s.status !== "destroyed"; });
      if (!active.length) { el.innerHTML = '<div class="empty">No active agents</div>'; return; }
      el.innerHTML = "";
      active.forEach(function(s) {
        var div = document.createElement("div");
        div.className = "item";
        div.innerHTML =
          '<span class="dot dot-' + s.status + '"></span>' +
          '<span class="name">' + (s.branch || s.id) + '</span>' +
          '<div class="meta">' + s.status + ' &middot; ' + (s.provider || '') + '</div>';
        el.appendChild(div);
      });
    });
  </script>
</body>
</html>
```

---

## 5. Building a Backend Plugin

Backend plugins run JavaScript in a QuickJS sandbox on the Rust side. They execute when Hivemind starts (or when the plugin is enabled) and stay active for the lifetime of the app session.

### File setup

Create a `main.js` at the path specified in your manifest's `main` field:

```json
{
  "main": "main.js"
}
```

### Available APIs in `main.js`

Inside the QuickJS context, a global `hivemind` object is available with these APIs:

#### `hivemind.log(message)`

Write a log message. This prints to the Hivemind console and emits a `log` event that UI plugins can listen to.

```js
hivemind.log("Plugin initialized");
hivemind.log("Processing " + items.length + " items");
```

#### `hivemind.on(event, callback)`

Register a callback for a hook event. The callback receives a `data` object with event-specific fields.

```js
hivemind.on("agent:launched", function(data) {
  hivemind.log("Agent launched: " + JSON.stringify(data));
});

hivemind.on("git:commit", function(data) {
  hivemind.log("New commit detected");
});
```

Your manifest's `hooks` array must include each event you listen to.

#### `hivemind.registerRoute(pathPattern, handler)`

Register an HTTP route handler. Agents (and any HTTP client) can call these routes via the Hivemind API server.

```js
hivemind.registerRoute("/plugins/my-plugin/*", function(req) {
  // req = { method, path, body, headers }

  if (req.path === "/plugins/my-plugin/status") {
    return { status: 200, body: { ok: true, message: "Running" } };
  }

  if (req.path === "/plugins/my-plugin/data" && req.method === "POST") {
    var name = req.body.name || "unknown";
    return { status: 200, body: { received: name } };
  }

  return { status: 404, body: { error: "Not found" } };
});
```

Route patterns use simple glob matching:
- `/plugins/my-plugin/*` matches any path starting with `/plugins/my-plugin/`
- `/plugins/my-plugin/exact` matches only that exact path

Your manifest's `routes` array must include each pattern you register. Always scope routes under `/plugins/<your-plugin-id>/` to avoid collisions.

#### `hivemind.emit(event, data)`

Emit an event that UI iframes can listen to via `hivemind.on()`. Use this to push real-time updates from the backend to your sidebar or panel.

```js
// In main.js (backend)
hivemind.emit("status-changed", { active: true, count: 42 });

// In sidebar.html (frontend) -- the UI receives it
hivemind.on("status-changed", function(data) {
  document.getElementById("count").textContent = data.count;
});
```

#### `hivemind.storage`

Persistent key-value storage, scoped to your plugin.

```js
// Synchronous in the backend context
var value = hivemind.storage.get("my-key");
hivemind.storage.set("my-key", "my-value");
hivemind.storage.delete("my-key");
```

**Note**: In the UI bridge, `storage.get()` returns a Promise. In the backend QuickJS context, it returns the value directly.

#### `hivemind.registerInstructions(text)`

Register custom instructions that will be injected into agent prompts. Use this to tell agents about tools your plugin provides.

```js
hivemind.registerInstructions(
  "The My Plugin extension is installed.\n" +
  "You can use the following command:\n" +
  "  hivemind-plugin-my-plugin-do-thing ARG1 ARG2\n" +
  "\n" +
  "Call this when you need to do the thing.\n"
);
```

#### `hivemind.toast(message, type)`

Show a toast notification in the IDE. Types: `"info"`, `"success"`, `"error"`.

```js
hivemind.toast("Processing complete", "success");
```

#### `hivemind.agents.launch(opts)` / `hivemind.agents.stop(branch)`

Launch or stop agents from the backend.

```js
hivemind.agents.launch({
  repoId: "uuid-here",
  branch: "feature/auto-fix",
  originBranch: "main",
  prompt: "Fix all linting errors"
});

hivemind.agents.stop("feature/auto-fix");
```

### Example: a hook listener that logs events

```js
hivemind.log("Event Logger loaded");

hivemind.on("app:started", function(data) {
  hivemind.log("App started");
  hivemind.toast("Event Logger is active", "info");
});

hivemind.on("agent:launched", function(data) {
  var msg = "Agent launched";
  if (data && data.branch) msg += " on " + data.branch;
  hivemind.log(msg);
});

hivemind.on("git:push", function(data) {
  hivemind.log("Push detected");
  hivemind.emit("push-event", data);
});
```

### Important notes about backend JavaScript

- The QuickJS runtime is **ES5-compatible** with some ES6 features. Use `var` instead of `let`/`const` to be safe. Arrow functions are not supported; use `function()` syntax.
- There is no `setTimeout`, `setInterval`, `fetch`, or Node.js APIs. Your code runs synchronously.
- Memory limit: 64 MB per plugin. Stack limit: 1 MB.
- Errors in hook callbacks are caught and logged automatically. They will not crash other plugins.

---

## 6. Creating Agent Scripts

Agent scripts are shell commands that AI agents can call from inside their Docker containers. They are the bridge between your plugin and the agent runtime.

### File placement

Put scripts in a `scripts/` directory inside your plugin. List them in the manifest:

```json
{
  "scripts": [
    "scripts/my-tool",
    "scripts/my-other-tool"
  ]
}
```

### Naming convention

When installed, scripts become available to agents with the name pattern:

```
hivemind-plugin-<plugin-id>-<script-name>
```

For example, if your plugin ID is `code-quality` and you have `scripts/run-lint`, the agent can call:

```
hivemind-plugin-code-quality-run-lint
```

### Requirements

1. **Shebang line**: Every script must start with `#!/bin/sh` or `#!/bin/bash`
2. **LF line endings**: Scripts must use Unix-style line endings (`\n`), not Windows-style (`\r\n`)
3. **Executable permissions**: Scripts must be executable (`chmod +x scripts/my-tool`)

### Environment variables available to scripts

Inside the Docker container, these environment variables are set:

| Variable | Description |
|----------|-------------|
| `HIVEMIND_API_URL` | Base URL of the Hivemind API (e.g. `http://host.docker.internal:12345/hivemind/v1`) |
| `HIVEMIND_API_TOKEN` | Authentication token for API calls |
| `HIVEMIND_AGENT_ID` | ID of the current agent session |

### Example: a simple logging script

**scripts/send-status**
```sh
#!/bin/sh
# hivemind-plugin-my-plugin-send-status — Report status to the plugin
#
# Usage:
#   hivemind-plugin-my-plugin-send-status "Step completed" info
#   hivemind-plugin-my-plugin-send-status "Something failed" error
#
# Arguments:
#   $1  Status message (required)
#   $2  Level: info (default), warn, error

set -e

MESSAGE="${1:?Usage: hivemind-plugin-my-plugin-send-status MESSAGE [LEVEL]}"
LEVEL="${2:-info}"

# Escape for JSON
ESC_MSG=$(printf '%s' "$MESSAGE" | sed 's/\\/\\\\/g; s/"/\\"/g')

curl -sf -X POST "${HIVEMIND_API_URL}/plugins/my-plugin/status" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${HIVEMIND_API_TOKEN}" \
  -d "{\"message\":\"${ESC_MSG}\",\"level\":\"${LEVEL}\",\"agent_id\":\"${HIVEMIND_AGENT_ID}\"}"
```

To handle this in your backend, register a matching route in `main.js`:

```js
hivemind.registerRoute("/plugins/my-plugin/*", function(req) {
  if (req.path === "/plugins/my-plugin/status") {
    var message = (req.body && req.body.message) || "(no message)";
    var level = (req.body && req.body.level) || "info";
    hivemind.log("[" + level + "] " + message);
    hivemind.emit("status-update", { message: message, level: level });
    return { status: 200, body: { ok: true } };
  }
  return { status: 404, body: { error: "Unknown route" } };
});
```

---

## 7. Testing Your Plugin

### Option A: Use the packaging script

If you have cloned the Hivemind repository, you can use the included script:

```bash
cd plugins/
./package-plugin.sh my-plugin
```

This creates `my-plugin.zip` in the `plugins/` directory.

### Option B: Manually create a zip

If you do not have the Hivemind repo, create the zip yourself. The key requirement is that the zip must contain your plugin as a **top-level directory** (not loose files).

**macOS / Linux:**
```bash
cd /path/to/parent-of-plugin
zip -r my-plugin.zip my-plugin/ -x "*/.DS_Store" "*/._*"
```

**Windows (PowerShell):**
```powershell
Compress-Archive -Path .\my-plugin -DestinationPath .\my-plugin.zip
```

The resulting zip structure should look like:

```
my-plugin.zip
  my-plugin/
    manifest.json
    main.js
    ui/
      sidebar.html
    scripts/
      my-tool
```

Not like this (wrong -- loose files):

```
my-plugin.zip
  manifest.json     <-- BAD: no wrapping directory
  main.js
```

### Importing for testing

1. Open Hivemind IDE
2. Go to **Settings > Plugins**
3. Click **Import Plugin**
4. Select your `.zip` file
5. A trust dialog will appear showing the plugin name, author, and description. Review and click **Trust & Install**

### Checking for errors

- After installing, verify the plugin appears in the plugin list with the correct name, version, and icon
- Enable the plugin if it is not already enabled
- If your plugin has a sidebar, click its icon in the left sidebar to check the UI renders
- Check the Hivemind console for any error messages (look for `[Plugin:your-plugin-id]` entries)
- Install the official **Hivemind Debug** plugin to see hook events firing in real-time -- this is very useful for verifying your hooks work correctly

### Quick test cycle

When iterating on your plugin:

1. Make your changes
2. Re-zip the plugin directory
3. In Hivemind, uninstall the old version (Settings > Plugins, click your plugin, Uninstall)
4. Import the new zip
5. Test

---

## 8. Importing Plugins

### From a local file

1. Go to **Settings > Plugins**
2. Click **Import Plugin**
3. Select a `.zip` file from your filesystem
4. Review the trust dialog and confirm

### From the Marketplace

1. Go to **Settings > Plugins**
2. Browse the **Marketplace** section
3. Click **Install** on any plugin you want
4. The plugin will be downloaded, verified, and installed automatically

### Updating plugins

When a newer version of an installed plugin is available in the marketplace, Hivemind will show an **Update** button next to it. Click it to download and install the new version.

For manually imported plugins, uninstall the current version and import the new zip.

---

## 9. Publishing to the Marketplace

The Hivemind plugin marketplace is powered by a public GitHub registry at:

```
https://github.com/Hivemind-ORG/Hivemind-Plugins
```

### How to submit your plugin

1. **Host your plugin zip** somewhere publicly accessible. GitHub Releases are recommended:
   - Create a GitHub repository for your plugin
   - Create a release (e.g. `v1.0.0`)
   - Upload your `.zip` file as a release asset
   - Copy the direct download URL for the zip

2. **Fork the registry repo**:
   - Go to `https://github.com/Hivemind-ORG/Hivemind-Plugins`
   - Click Fork

3. **Add your plugin to `registry.json`**:

   The registry file has this structure:

   ```json
   {
     "version": 1,
     "plugins": [
       {
         "id": "your-plugin-id",
         "name": "Your Plugin Name",
         "version": "1.0.0",
         "description": "A short description of what your plugin does.",
         "author": "Your Name",
         "icon": "rocket_launch",
         "download_url": "https://github.com/you/your-plugin/releases/download/v1.0.0/your-plugin-id.zip"
       }
     ]
   }
   ```

   Fields:

   | Field | Required | Description |
   |-------|----------|-------------|
   | `id` | Yes | Must match the `id` in your plugin's `manifest.json` |
   | `name` | Yes | Display name |
   | `version` | Yes | Must match the version in your plugin's `manifest.json` |
   | `description` | Yes | Short description for the marketplace listing |
   | `author` | Yes | Your name or organization |
   | `icon` | No | Material Symbols icon name for the marketplace listing |
   | `download_url` | Yes | Direct URL to download the `.zip` file |

4. **Submit a pull request** to the registry repository with your addition.

### Updating your listing

When you release a new version:

1. Upload the new zip to your hosting (e.g., create a new GitHub release)
2. Update `version` and `download_url` in your registry entry
3. Submit a PR to the registry repo

---

## 10. Tips and Best Practices

### Keep plugins small and focused

A plugin that does one thing well is better than a plugin that tries to do everything. Small plugins load faster and are easier to debug.

### Use storage for persistent data

The `hivemind.storage` API is your database. It persists across app restarts and is scoped to your plugin so you cannot accidentally overwrite another plugin's data.

Store complex data as JSON strings:

```js
// Save
var data = { count: 42, items: ["a", "b"] };
hivemind.storage.set("my-data", JSON.stringify(data));

// Load
var raw = hivemind.storage.get("my-data");
var data = raw ? JSON.parse(raw) : { count: 0, items: [] };
```

### Handle errors gracefully

The bridge catches errors, but your UI should show useful information when things go wrong:

```js
hivemind.repositories.list().then(function(repos) {
  renderRepos(repos);
}).catch(function(err) {
  document.getElementById("status").textContent = "Failed to load repositories";
});
```

In backend hooks, wrap risky logic:

```js
hivemind.on("agent:launched", function(data) {
  try {
    // Your logic here
  } catch (e) {
    hivemind.log("Error handling agent launch: " + e);
  }
});
```

### Test with the debug plugin

Install the official **Hivemind Debug** plugin. It shows every hook event in real-time, which is invaluable when developing your own plugin. You can see exactly what data each hook receives.

### Use transparent backgrounds for UI

Always set `background: transparent` on your `<body>` element. This ensures your plugin blends with whatever theme Hivemind uses, rather than clashing with a solid white or black background.

### Scope your routes

Always put your HTTP routes under `/plugins/<your-plugin-id>/` to avoid colliding with other plugins or the core Hivemind API.

### Write clean script headers

Agent scripts should include a usage comment so the agent (and human developers) know how to call them:

```sh
#!/bin/sh
# hivemind-plugin-my-plugin-my-tool — Short description of what it does
#
# Usage:
#   hivemind-plugin-my-plugin-my-tool ARG1 [ARG2]
#
# Arguments:
#   $1  Description of first argument (required)
#   $2  Description of second argument (optional, default: "value")
```

### Use `registerInstructions` to teach agents

If your plugin provides agent scripts, always call `hivemind.registerInstructions()` in your `main.js` to tell agents what commands are available and how to use them. Without this, agents will not know your scripts exist.

### Mind the JavaScript limitations

Backend code runs in QuickJS, not Node.js or a browser. Stick to:
- `var` declarations (not `let`/`const`)
- `function()` expressions (not arrow functions `=>`)
- `JSON.parse()` / `JSON.stringify()` for data
- No `async`/`await`, no `fetch`, no `setTimeout`

---

## 11. References

- [PLUGIN_API.md](./PLUGIN_API.md) -- Complete API reference for all backend and bridge APIs
- [PLUGIN_READY_CHECKLIST.md](./PLUGIN_READY_CHECKLIST.md) -- Pre-release checklist to validate your plugin before publishing
- [Material Symbols icons](https://fonts.google.com/icons?icon.set=Material+Symbols) -- Browse icons for your `sidebar_icon`
- [Hivemind-Plugins registry](https://github.com/Hivemind-ORG/Hivemind-Plugins) -- The marketplace registry repository
