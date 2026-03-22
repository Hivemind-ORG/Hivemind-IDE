# Hivemind Plugin API Reference

Complete API reference for the Hivemind IDE plugin system. This document covers every hook, bridge API, action call, and backend runtime method available to plugin developers.

> **Status badges:** Every method in this document is marked with a status badge. All APIs are currently ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red).

---

## Table of Contents

- [Overview](#overview)
- [Plugin Architecture](#plugin-architecture)
- [1. Hooks (Backend Runtime)](#1-hooks-backend-runtime)
  - [Application Hooks](#application-hooks)
  - [Agent Hooks](#agent-hooks)
  - [Git Hooks](#git-hooks)
  - [Pull Request Hooks](#pull-request-hooks)
  - [Repository Hooks](#repository-hooks)
  - [Plugin Lifecycle Hooks](#plugin-lifecycle-hooks)
  - [File System Hooks](#file-system-hooks)
  - [Settings Hooks](#settings-hooks)
  - [System Hooks](#system-hooks)
- [2. UI Bridge APIs (Read Operations)](#2-ui-bridge-apis-read-operations)
  - [Storage](#bridge-storage)
  - [Events](#bridge-events)
  - [Toast](#bridge-toast)
  - [System](#bridge-system)
  - [Repositories](#bridge-repositories)
  - [Agents](#bridge-agents)
  - [Git](#bridge-git)
  - [Files](#bridge-files)
  - [Pull Requests](#bridge-pull-requests)
  - [Settings](#bridge-settings)
  - [Macros](#bridge-macros)
  - [Repository Groups](#bridge-repository-groups)
  - [Plugins](#bridge-plugins)
  - [AI](#bridge-ai)
  - [View](#bridge-view)
  - [Clipboard](#bridge-clipboard)
- [3. UI Action Calls (Write Operations)](#3-ui-action-calls-write-operations)
  - [Agent Actions](#action-agents)
  - [Git Actions](#action-git)
  - [File Actions](#action-files)
  - [Pull Request Actions](#action-pull-requests)
  - [Settings Actions](#action-settings)
  - [Macro Actions](#action-macros)
  - [Terminal Actions](#action-terminal)
  - [Navigation Actions](#action-navigation)
  - [Clipboard Actions](#action-clipboard)
- [4. Backend Runtime API (QuickJS)](#4-backend-runtime-api-quickjs)
- [5. Type Reference](#5-type-reference)

---

## Overview

Hivemind plugins extend the IDE through two runtimes:

1. **Backend plugins** (`main.js`) run in a sandboxed QuickJS context on the Rust backend. They can register hook listeners, define HTTP routes for agents, emit events to the frontend, and inject custom instructions into agent prompts.

2. **UI plugins** (`ui/sidebar.html` or `ui/panel.html`) run in a sandboxed `<iframe>` in the frontend. They access the IDE through the `window.hivemind` bridge object, which provides read APIs, action calls, storage, event listeners, and navigation.

Both runtimes communicate through the Tauri event system. Backend hooks fire when the IDE emits events (agent lifecycle, git operations, etc.). UI bridge calls are routed through `postMessage` to the host window, which delegates to Tauri IPC commands.

## Plugin Architecture

```
+---------------------+       Tauri Events       +---------------------+
|  Backend Runtime    |  <------------------->    |  Frontend (React)   |
|  (QuickJS)          |                           |                     |
|  - main.js          |       postMessage         |  +---------------+  |
|  - hivemind.on()    |  <------------------->    |  | Plugin iframe |  |
|  - hivemind.emit()  |                           |  | sidebar.html  |  |
|  - registerRoute()  |                           |  | window.hivemind|  |
+---------------------+                           |  +---------------+  |
        |                                         +---------------------+
        |  HTTP (host.docker.internal)                     |
        v                                                  v
+---------------------+                           Tauri IPC (invoke)
|  Agent in Docker    |
|  - shell scripts    |
|  - plugin routes    |
+---------------------+
```

---

## 1. Hooks (Backend Runtime)

Hooks are events fired by the Hivemind backend. Backend plugins subscribe to them via `hivemind.on(event, callback)` in `main.js`. The callback receives a single data object with the fields described below.

Hooks must be declared in your plugin's `manifest.json` under the `"hooks"` array.

---

### Application Hooks

#### `app:started`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when Hivemind has finished initializing and is ready to use.

| Field | Type | Description |
|-------|------|-------------|
| *(none)* | | Empty payload |

```js
hivemind.on("app:started", function(data) {
  hivemind.log("Hivemind is ready!");
});
```

---

### Agent Hooks

#### `agent:launched`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when an agent session has been launched and its Docker container is running.

| Field | Type | Description |
|-------|------|-------------|
| `agent_id` | `string` | Unique agent session ID |
| `repo_id` | `string` | Repository the agent is working on |
| `branch` | `string` | Branch name the agent is working on |
| `mode` | `string` | Agent mode (e.g., `"coder"`, `"bug-fixer"`, `"code-reviewer"`) |

```js
hivemind.on("agent:launched", function(data) {
  hivemind.log("Agent " + data.agent_id + " launched on branch " + data.branch);
});
```

---

#### `agent:completed`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when an agent has finished its task successfully.

| Field | Type | Description |
|-------|------|-------------|
| `agent_id` | `string` | Unique agent session ID |
| `summary` | `string` | Summary of what the agent accomplished |

```js
hivemind.on("agent:completed", function(data) {
  hivemind.log("Agent " + data.agent_id + " completed: " + data.summary);
});
```

---

#### `agent:stopped`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when an agent has been stopped (by user request or system).

| Field | Type | Description |
|-------|------|-------------|
| `agent_id` | `string` | Unique agent session ID |
| `status` | `string` | Final status of the agent |

```js
hivemind.on("agent:stopped", function(data) {
  hivemind.log("Agent " + data.agent_id + " stopped with status: " + data.status);
});
```

---

#### `agent:error`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when an agent encounters an error.

| Field | Type | Description |
|-------|------|-------------|
| `agent_id` | `string` | Unique agent session ID |
| `error` | `string` | Error message |

```js
hivemind.on("agent:error", function(data) {
  hivemind.log("Agent " + data.agent_id + " error: " + data.error);
  hivemind.toast("Agent error: " + data.error, "error");
});
```

---

#### `agent:restarted`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when an agent session is restarted.

| Field | Type | Description |
|-------|------|-------------|
| `agent_id` | `string` | Unique agent session ID |
| `prompt` | `string` | The prompt the agent was restarted with |

```js
hivemind.on("agent:restarted", function(data) {
  hivemind.log("Agent " + data.agent_id + " restarted with prompt: " + data.prompt);
});
```

---

### Git Hooks

#### `git:commit`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when an agent makes a git commit.

| Field | Type | Description |
|-------|------|-------------|
| `agent_id` | `string` | Agent that made the commit |
| `sha` | `string` | Commit SHA |
| `files_changed` | `number` | Number of files changed |

```js
hivemind.on("git:commit", function(data) {
  hivemind.log("Commit " + data.sha + " by agent " + data.agent_id + " (" + data.files_changed + " files)");
});
```

---

#### `git:push`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when an agent pushes a branch to the remote.

| Field | Type | Description |
|-------|------|-------------|
| `agent_id` | `string` | Agent that pushed |
| `force` | `boolean` | Whether the push was a force push |

```js
hivemind.on("git:push", function(data) {
  if (data.force) {
    hivemind.log("WARNING: Force push by agent " + data.agent_id);
  }
});
```

---

#### `git:pull`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when an agent pulls a branch from the remote.

| Field | Type | Description |
|-------|------|-------------|
| `agent_id` | `string` | Agent that pulled |
| `branch` | `string` | Branch that was pulled |

```js
hivemind.on("git:pull", function(data) {
  hivemind.log("Agent " + data.agent_id + " pulled branch " + data.branch);
});
```

---

#### `git:merge`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when an agent completes a merge operation.

| Field | Type | Description |
|-------|------|-------------|
| `agent_id` | `string` | Agent that performed the merge |
| `success` | `boolean` | Whether the merge succeeded |
| `has_conflicts` | `boolean` | Whether merge conflicts exist |

```js
hivemind.on("git:merge", function(data) {
  if (data.has_conflicts) {
    hivemind.toast("Merge conflicts detected", "error");
  }
});
```

---

#### `git:rebase`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when an agent completes a rebase operation.

| Field | Type | Description |
|-------|------|-------------|
| `agent_id` | `string` | Agent that performed the rebase |
| `success` | `boolean` | Whether the rebase succeeded |
| `has_conflicts` | `boolean` | Whether rebase conflicts exist |

```js
hivemind.on("git:rebase", function(data) {
  if (!data.success) {
    hivemind.log("Rebase failed for agent " + data.agent_id);
  }
});
```

---

#### `git:pr_created`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when an agent creates a pull request.

| Field | Type | Description |
|-------|------|-------------|
| `agent_id` | `string` | Agent that created the PR |
| `url` | `string` | URL of the created PR |
| `number` | `number` | PR number |

```js
hivemind.on("git:pr_created", function(data) {
  hivemind.log("PR #" + data.number + " created: " + data.url);
  hivemind.toast("PR created: #" + data.number, "success");
});
```

---

### Pull Request Hooks

#### `pr:review_submitted`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when a PR review is submitted by an agent.

| Field | Type | Description |
|-------|------|-------------|
| `agent_id` | `string` | Agent that submitted the review |
| `verdict` | `string` | Review verdict (e.g., `"approve"`, `"request_changes"`, `"comment"`) |

```js
hivemind.on("pr:review_submitted", function(data) {
  hivemind.log("Agent " + data.agent_id + " submitted review: " + data.verdict);
});
```

---

#### `pr:comment_added`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when a PR comment is added by an agent.

| Field | Type | Description |
|-------|------|-------------|
| `agent_id` | `string` | Agent that added the comment |
| `body` | `string` | Comment body text |
| `file` | `string` | File the comment is on |
| `line` | `number` | Line number the comment is on |

```js
hivemind.on("pr:comment_added", function(data) {
  hivemind.log("Comment on " + data.file + ":" + data.line + " — " + data.body);
});
```

---

### Repository Hooks

#### `repo:added`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when a repository is added to Hivemind.

| Field | Type | Description |
|-------|------|-------------|
| `repo_id` | `string` | Unique repository ID |
| `name` | `string` | Repository name |
| `path` | `string` | Local filesystem path |

```js
hivemind.on("repo:added", function(data) {
  hivemind.log("Repository added: " + data.name + " at " + data.path);
});
```

---

#### `repo:removed`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when a repository is removed from Hivemind.

| Field | Type | Description |
|-------|------|-------------|
| `repo_id` | `string` | Unique repository ID |

```js
hivemind.on("repo:removed", function(data) {
  hivemind.log("Repository removed: " + data.repo_id);
});
```

---

### Plugin Lifecycle Hooks

#### `plugin:installed`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when a plugin is installed.

| Field | Type | Description |
|-------|------|-------------|
| `plugin_id` | `string` | ID of the installed plugin |
| `version` | `string` | Version of the installed plugin |

```js
hivemind.on("plugin:installed", function(data) {
  hivemind.log("Plugin installed: " + data.plugin_id + " v" + data.version);
});
```

---

#### `plugin:uninstalled`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when a plugin is uninstalled.

| Field | Type | Description |
|-------|------|-------------|
| `plugin_id` | `string` | ID of the uninstalled plugin |

```js
hivemind.on("plugin:uninstalled", function(data) {
  hivemind.log("Plugin uninstalled: " + data.plugin_id);
});
```

---

#### `plugin:enabled`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when a plugin is enabled.

| Field | Type | Description |
|-------|------|-------------|
| `plugin_id` | `string` | ID of the enabled plugin |

```js
hivemind.on("plugin:enabled", function(data) {
  hivemind.log("Plugin enabled: " + data.plugin_id);
});
```

---

#### `plugin:disabled`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when a plugin is disabled.

| Field | Type | Description |
|-------|------|-------------|
| `plugin_id` | `string` | ID of the disabled plugin |

```js
hivemind.on("plugin:disabled", function(data) {
  hivemind.log("Plugin disabled: " + data.plugin_id);
});
```

---

### File System Hooks

#### `file:changed`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when a file changes in a watched directory.

| Field | Type | Description |
|-------|------|-------------|
| `path` | `string` | Absolute path to the changed file |
| `kind` | `string` | Type of change: `"create"`, `"modify"`, or `"delete"` |

```js
hivemind.on("file:changed", function(data) {
  if (data.kind === "delete") {
    hivemind.log("File deleted: " + data.path);
  }
});
```

---

### Settings Hooks

#### `settings:changed`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires when a setting is changed.

| Field | Type | Description |
|-------|------|-------------|
| `key` | `string` | Setting key that changed |
| `value` | `string` | New value of the setting |

```js
hivemind.on("settings:changed", function(data) {
  if (data.key === "appearance.theme") {
    hivemind.log("Theme changed to: " + data.value);
  }
});
```

---

### System Hooks

#### `log`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fires for system log events.

| Field | Type | Description |
|-------|------|-------------|
| `message` | `string` | Log message |
| `level` | `string` | Log level (e.g., `"info"`, `"warn"`, `"error"`) |

```js
hivemind.on("log", function(data) {
  if (data.level === "error") {
    hivemind.log("System error: " + data.message);
  }
});
```

---

## 2. UI Bridge APIs (Read Operations)

UI plugins access the IDE through `window.hivemind` inside their sandboxed iframe. All async methods return Promises that resolve when the host responds via `postMessage`.

---

<a id="bridge-storage"></a>
### Storage

Per-plugin key-value storage persisted in the SQLite database.

#### `storage.get(key)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Retrieve a value from plugin-scoped storage.

| Parameter | Type | Description |
|-----------|------|-------------|
| `key` | `string` | Storage key |

**Returns:** `Promise<string | null>` -- The stored value, or `null` if not found.

```js
const lastRun = await hivemind.storage.get("last_run_timestamp");
if (lastRun) {
  console.log("Last run:", lastRun);
}
```

---

#### `storage.set(key, value)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Set a value in plugin-scoped storage. Fire-and-forget (no return value).

| Parameter | Type | Description |
|-----------|------|-------------|
| `key` | `string` | Storage key |
| `value` | `string` | Value to store (must be a string; use `JSON.stringify()` for objects) |

**Returns:** `void`

```js
hivemind.storage.set("last_run_timestamp", new Date().toISOString());
hivemind.storage.set("config", JSON.stringify({ theme: "dark", verbose: true }));
```

---

#### `storage.delete(key)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Delete a value from plugin-scoped storage. Fire-and-forget.

| Parameter | Type | Description |
|-----------|------|-------------|
| `key` | `string` | Storage key to delete |

**Returns:** `void`

```js
hivemind.storage.delete("temporary_cache");
```

---

<a id="bridge-events"></a>
### Events

#### `on(event, callback)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Listen for events emitted by the backend or other plugins via `hivemind.emit()`.

| Parameter | Type | Description |
|-----------|------|-------------|
| `event` | `string` | Event name to listen for |
| `callback` | `function(data)` | Callback invoked with the event data |

**Returns:** `void`

```js
hivemind.on("new-event", function(entry) {
  console.log("Event:", entry.event, entry.data);
});

// Listen for plugin-emitted custom events
hivemind.on("my-plugin:status-update", function(data) {
  document.getElementById("status").textContent = data.message;
});
```

---

<a id="bridge-toast"></a>
### Toast

#### `toast(message, type?)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Show a toast notification in the IDE.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `message` | `string` | | Notification message |
| `type` | `string` | `"info"` | Toast type: `"info"`, `"success"`, or `"error"` |

**Returns:** `void`

```js
hivemind.toast("Operation completed successfully", "success");
hivemind.toast("Something went wrong", "error");
hivemind.toast("Processing...");  // defaults to "info"
```

---

<a id="bridge-system"></a>
### System

#### `pluginId`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

The current plugin's ID. This is a property, not a function.

**Type:** `string`

```js
console.log("My plugin ID:", hivemind.pluginId);
// => "my-plugin"
```

---

#### `getSystemInfo()`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Get system information about the running Hivemind instance.

**Returns:** `Promise<{ api_port: number, platform: string, user_agent: string }>`

| Response Field | Type | Description |
|----------------|------|-------------|
| `api_port` | `number` | Port the local API server is running on |
| `platform` | `string` | Platform identifier (e.g., `"MacIntel"`, `"Win32"`) |
| `user_agent` | `string` | Browser user agent string |

```js
const info = await hivemind.getSystemInfo();
console.log("API running on port:", info.api_port);
console.log("Platform:", info.platform);
```

---

#### `system.getStats()`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Get system resource statistics.

**Returns:** `Promise<{ cpu_usage_percent: number, memory_used: number, memory_total: number } | null>`

| Response Field | Type | Description |
|----------------|------|-------------|
| `cpu_usage_percent` | `number` | CPU usage as a percentage (0-100) |
| `memory_used` | `number` | Memory used in bytes |
| `memory_total` | `number` | Total memory in bytes |

Returns `null` if stats are unavailable.

```js
const stats = await hivemind.system.getStats();
if (stats) {
  const memPercent = ((stats.memory_used / stats.memory_total) * 100).toFixed(1);
  console.log("CPU:", stats.cpu_usage_percent + "%", "Memory:", memPercent + "%");
}
```

---

<a id="bridge-repositories"></a>
### Repositories

#### `repositories.list()`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

List all repositories registered in Hivemind.

**Returns:** `Promise<Repository[]>`

See [Repository](#repository) type.

```js
const repos = await hivemind.repositories.list();
repos.forEach(function(repo) {
  console.log(repo.name, repo.local_path);
});
```

---

#### `repositories.get(repoId)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Get a single repository by ID.

| Parameter | Type | Description |
|-----------|------|-------------|
| `repoId` | `string` | Repository UUID |

**Returns:** `Promise<Repository | null>`

Returns `null` if not found.

```js
const repo = await hivemind.repositories.get("abc-123-def");
if (repo) {
  console.log("Found:", repo.name, "at", repo.local_path);
}
```

---

#### `repositories.getBranches(repoId, repoPath)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

List all branches (local and remote) for a repository.

| Parameter | Type | Description |
|-----------|------|-------------|
| `repoId` | `string` | Repository UUID |
| `repoPath` | `string` | Absolute path to the repository on disk |

**Returns:** `Promise<BranchInfo[]>`

See [BranchInfo](#branchinfo) type.

```js
const repo = await hivemind.repositories.get("abc-123-def");
if (repo && repo.local_path) {
  const branches = await hivemind.repositories.getBranches(repo.id, repo.local_path);
  branches.forEach(function(b) {
    var label = b.is_head ? " (HEAD)" : "";
    console.log(b.name + label + " — " + b.commit_sha.substring(0, 7));
  });
}
```

---

<a id="bridge-agents"></a>
### Agents

#### `agents.list()`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

List all agent sessions (including completed and stopped ones).

**Returns:** `Promise<AgentSession[]>`

See [AgentSession](#agentsession) type.

```js
const sessions = await hivemind.agents.list();
var running = sessions.filter(function(s) { return s.status === "running"; });
console.log(running.length + " agents currently running");
```

---

#### `agents.get(agentId)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Get a specific agent session by ID.

| Parameter | Type | Description |
|-----------|------|-------------|
| `agentId` | `string` | Agent session UUID |

**Returns:** `Promise<AgentSession | null>`

Returns `null` if not found.

```js
const session = await hivemind.agents.get("agent-abc-123");
if (session) {
  console.log("Agent status:", session.status, "on branch:", session.branch);
}
```

---

<a id="bridge-git"></a>
### Git

#### `git.getUncommittedFiles(worktreePath)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Get the list of uncommitted files in a worktree.

| Parameter | Type | Description |
|-----------|------|-------------|
| `worktreePath` | `string` | Absolute path to the git worktree |

**Returns:** `Promise<FileStatusEntry[]>`

See [FileStatusEntry](#filestatusentry) type.

```js
const files = await hivemind.git.getUncommittedFiles("/path/to/worktree");
files.forEach(function(f) {
  console.log(f.status + ": " + f.path);
});
```

---

#### `git.getDiff(repoPath, branch)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Get the diff between a branch and its origin branch.

| Parameter | Type | Description |
|-----------|------|-------------|
| `repoPath` | `string` | Absolute path to the repository or worktree |
| `branch` | `string` | Origin/base branch to diff against |

**Returns:** `Promise<FileDiff[]>`

See [FileDiff](#filediff) type.

```js
const diffs = await hivemind.git.getDiff("/path/to/repo", "main");
diffs.forEach(function(file) {
  console.log(file.status + " " + file.path + " (" + file.hunks.length + " hunks)");
});
```

---

#### `git.getAheadBehind(repoPath, branch)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Get the number of commits a branch is ahead/behind its origin.

| Parameter | Type | Description |
|-----------|------|-------------|
| `repoPath` | `string` | Absolute path to the repository or worktree |
| `branch` | `string` | Origin/base branch to compare against |

**Returns:** `Promise<{ ahead: number, behind: number }>`

| Response Field | Type | Description |
|----------------|------|-------------|
| `ahead` | `number` | Number of commits ahead of origin |
| `behind` | `number` | Number of commits behind origin |

```js
const status = await hivemind.git.getAheadBehind("/path/to/repo", "main");
console.log("Ahead: " + status.ahead + ", Behind: " + status.behind);
```

---

#### `git.blame(repoPath, filePath, line)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Get git blame information for a specific line of a file.

| Parameter | Type | Description |
|-----------|------|-------------|
| `repoPath` | `string` | Absolute path to the repository or worktree |
| `filePath` | `string` | Relative path to the file within the repo |
| `line` | `number` | Line number (1-based) |

**Returns:** `Promise<BlameLineInfo | null>`

See [BlameLineInfo](#blamelineinfo) type. Returns `null` on failure.

```js
const blame = await hivemind.git.blame("/path/to/repo", "src/main.ts", 42);
if (blame) {
  console.log("Line 42 last modified by " + blame.author + " in " + blame.short_sha);
}
```

---

#### `git.log(repoPath, branch, limit)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Get the commit history for a repository.

| Parameter | Type | Description |
|-----------|------|-------------|
| `repoPath` | `string` | Absolute path to the repository |
| `branch` | `string` | Branch name (currently unused; log reads from HEAD) |
| `limit` | `number` | Maximum number of commits to return |

**Returns:** `Promise<CommitLogEntry[]>`

See [CommitLogEntry](#commitlogentry) type.

```js
const commits = await hivemind.git.log("/path/to/repo", "main", 20);
commits.forEach(function(c) {
  console.log(c.sha.substring(0, 7) + " " + c.author + " — " + c.message);
});
```

---

<a id="bridge-files"></a>
### Files

#### `files.read(filePath)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Read the contents of a file.

| Parameter | Type | Description |
|-----------|------|-------------|
| `filePath` | `string` | Absolute path to the file |

**Returns:** `Promise<string | null>`

Returns `null` if the file cannot be read.

```js
const content = await hivemind.files.read("/path/to/repo/package.json");
if (content) {
  var pkg = JSON.parse(content);
  console.log("Project:", pkg.name, "v" + pkg.version);
}
```

---

#### `files.list(dirPath)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

List the contents of a directory.

| Parameter | Type | Description |
|-----------|------|-------------|
| `dirPath` | `string` | Absolute path to the directory |

**Returns:** `Promise<DirEntry[]>`

See [DirEntry](#direntry) type.

```js
const entries = await hivemind.files.list("/path/to/repo/src");
entries.forEach(function(e) {
  var icon = e.is_dir ? "[DIR]" : "[FILE]";
  console.log(icon + " " + e.name + (e.is_dir ? "" : " (" + e.size + " bytes)"));
});
```

---

#### `files.exists(filePath)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Check if a file or directory exists.

| Parameter | Type | Description |
|-----------|------|-------------|
| `filePath` | `string` | Absolute path to check |

**Returns:** `Promise<boolean>`

```js
const hasConfig = await hivemind.files.exists("/path/to/repo/.eslintrc.js");
if (!hasConfig) {
  console.log("No ESLint config found");
}
```

---

#### `files.search(dirPath, query)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Search for files by name within a directory tree.

| Parameter | Type | Description |
|-----------|------|-------------|
| `dirPath` | `string` | Root directory to search from |
| `query` | `string` | Search query (matches against file names and content) |

**Returns:** `Promise<SearchResult[]>`

See [SearchResult](#searchresult) type.

```js
const results = await hivemind.files.search("/path/to/repo", "TODO");
results.forEach(function(r) {
  var loc = r.line_number ? ":" + r.line_number : "";
  console.log(r.file_name + loc + " — " + (r.snippet || ""));
});
```

---

<a id="bridge-pull-requests"></a>
### Pull Requests

#### `pr.list(repoId)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

List open pull requests for a repository.

| Parameter | Type | Description |
|-----------|------|-------------|
| `repoId` | `string` | Repository UUID |

**Returns:** `Promise<PullRequestInfo[]>`

See [PullRequestInfo](#pullrequestinfo) type.

```js
const prs = await hivemind.pr.list("repo-abc-123");
prs.forEach(function(pr) {
  var draft = pr.draft ? " [DRAFT]" : "";
  console.log("#" + pr.number + draft + " " + pr.title + " (" + pr.head_branch + " -> " + pr.base_branch + ")");
});
```

---

#### `pr.get(repoId, prNumber)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Get detailed information about a specific pull request.

| Parameter | Type | Description |
|-----------|------|-------------|
| `repoId` | `string` | Repository UUID |
| `prNumber` | `number` | Pull request number |

**Returns:** `Promise<PullRequestDetail | null>`

See [PullRequestDetail](#pullrequestdetail) type. Returns `null` if not found.

```js
const pr = await hivemind.pr.get("repo-abc-123", 42);
if (pr) {
  console.log("#" + pr.number + " " + pr.title);
  console.log("  +" + pr.additions + " -" + pr.deletions + " across " + pr.changed_files + " files");
  console.log("  Mergeable:", pr.mergeable);
}
```

---

#### `pr.getFiles(repoId, prNumber)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Get the file changes (diffs) for a pull request.

| Parameter | Type | Description |
|-----------|------|-------------|
| `repoId` | `string` | Repository UUID |
| `prNumber` | `number` | Pull request number |

**Returns:** `Promise<FileDiff[]>`

See [FileDiff](#filediff) type.

```js
const files = await hivemind.pr.getFiles("repo-abc-123", 42);
files.forEach(function(f) {
  console.log(f.status + " " + f.path);
  f.hunks.forEach(function(h) {
    console.log("  " + h.header);
  });
});
```

---

<a id="bridge-settings"></a>
### Settings

#### `settings.get(key)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Get a single setting value.

| Parameter | Type | Description |
|-----------|------|-------------|
| `key` | `string` | Setting key (e.g., `"appearance.theme"`) |

**Returns:** `Promise<string | null>`

```js
const theme = await hivemind.settings.get("appearance.theme");
console.log("Current theme:", theme);
```

---

#### `settings.getAll()`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Get all settings as a key-value map.

**Returns:** `Promise<Record<string, string>>`

```js
const all = await hivemind.settings.getAll();
for (var key in all) {
  console.log(key + " = " + all[key]);
}
```

---

#### `settings.getTheme()`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Get the current theme setting. Convenience method equivalent to `settings.get("appearance.theme")`.

**Returns:** `Promise<string | null>`

```js
const theme = await hivemind.settings.getTheme();
document.body.className = theme === "light" ? "theme-light" : "theme-dark";
```

---

<a id="bridge-macros"></a>
### Macros

#### `macros.list(repoId)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

List all run macros for a repository.

| Parameter | Type | Description |
|-----------|------|-------------|
| `repoId` | `string` | Repository UUID |

**Returns:** `Promise<RunMacro[]>`

See [RunMacro](#runmacro) type.

```js
const macros = await hivemind.macros.list("repo-abc-123");
macros.forEach(function(m) {
  console.log(m.name + " (" + m.lanes.length + " lanes)");
});
```

---

<a id="bridge-repository-groups"></a>
### Repository Groups

#### `repoGroups.list()`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

List all repository groups.

**Returns:** `Promise<RepoGroup[]>`

See [RepositoryGroup](#repositorygroup) type.

```js
const groups = await hivemind.repoGroups.list();
groups.forEach(function(g) {
  console.log("Group: " + g.name + " (ID: " + g.id + ")");
});
```

---

<a id="bridge-plugins"></a>
### Plugins

#### `plugins.list()`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

List all installed plugins.

**Returns:** `Promise<Plugin[]>`

See [Plugin](#plugin) type.

```js
const plugins = await hivemind.plugins.list();
plugins.forEach(function(p) {
  var status = p.enabled ? "enabled" : "disabled";
  console.log(p.name + " v" + p.version + " [" + status + "]");
});
```

---

<a id="bridge-ai"></a>
### AI

#### `ai.getProvider()`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Get the currently active AI provider configuration.

**Returns:** `Promise<AiProviderConfig | null>`

See [AiProviderConfig](#aiproviderconfig) type. Returns `null` if no provider is configured.

```js
const provider = await hivemind.ai.getProvider();
if (provider) {
  console.log("AI Provider:", provider.provider_id, "via", provider.auth_method);
} else {
  console.log("No AI provider configured");
}
```

---

<a id="bridge-view"></a>
### View

#### `view.getFocusState()`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Get the current focus state of the IDE. When in "focus mode" (viewing an agent worktree), returns the repository and branch context. Otherwise returns the active tab.

**Returns:** `Promise<FocusState>`

When focused:
| Response Field | Type | Description |
|----------------|------|-------------|
| `isFocusMode` | `true` | Indicates focus mode is active |
| `repoId` | `string` | Repository UUID |
| `branch` | `string` | Branch name |
| `repoPath` | `string` | Worktree path |

When not focused:
| Response Field | Type | Description |
|----------------|------|-------------|
| `isFocusMode` | `false` | Indicates focus mode is not active |
| `repoId` | `null` | |
| `branch` | `null` | |
| `repoPath` | `null` | |
| `activeTab` | `string` | Currently active main tab ID |

```js
const focus = await hivemind.view.getFocusState();
if (focus.isFocusMode) {
  console.log("Focused on:", focus.branch, "in repo", focus.repoId);
  console.log("Worktree path:", focus.repoPath);
} else {
  console.log("Not in focus mode. Active tab:", focus.activeTab);
}
```

---

<a id="bridge-clipboard"></a>
### Clipboard

#### `clipboard.read()`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Read text from the system clipboard.

**Returns:** `Promise<string>`

Returns an empty string if the clipboard is empty or cannot be read.

```js
const text = await hivemind.clipboard.read();
if (text) {
  console.log("Clipboard contains:", text.substring(0, 100));
}
```

---

## 3. UI Action Calls (Write Operations)

Action calls perform mutations: launching agents, writing files, creating branches, etc. Most return Promises to indicate success/failure.

---

<a id="action-agents"></a>
### Agent Actions

#### `agents.launch(opts)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Launch a new agent session. Fire-and-forget -- the launch is handled asynchronously by the backend.

| Parameter | Type | Description |
|-----------|------|-------------|
| `opts` | `object` | Launch options |
| `opts.repoId` | `string` | Repository UUID |
| `opts.branch` | `string` | Branch name for the agent to work on |
| `opts.originBranch` | `string` | *(optional)* Base branch to branch from |
| `opts.prompt` | `string` | Prompt / task description for the agent |

**Returns:** `void`

```js
hivemind.agents.launch({
  repoId: "repo-abc-123",
  branch: "feature/add-dark-mode",
  originBranch: "main",
  prompt: "Add a dark mode toggle to the settings page"
});
```

---

#### `agents.stop(branch)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Stop an agent by its branch name. Fire-and-forget.

| Parameter | Type | Description |
|-----------|------|-------------|
| `branch` | `string` | Branch name the agent is working on |

**Returns:** `void`

```js
hivemind.agents.stop("feature/add-dark-mode");
```

---

#### `agents.stopById(agentId)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Stop an agent by its session ID. Fire-and-forget.

| Parameter | Type | Description |
|-----------|------|-------------|
| `agentId` | `string` | Agent session UUID |

**Returns:** `void`

```js
hivemind.agents.stopById("agent-abc-123");
```

---

#### `agents.destroy(agentId)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Destroy an agent session, removing its Docker container and worktree. Fire-and-forget.

| Parameter | Type | Description |
|-----------|------|-------------|
| `agentId` | `string` | Agent session UUID |

**Returns:** `void`

```js
hivemind.agents.destroy("agent-abc-123");
```

---

#### `agents.restartById(agentId, withPrompt?)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Restart an agent, optionally re-using its existing prompt. Fire-and-forget.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `agentId` | `string` | | Agent session UUID |
| `withPrompt` | `boolean` | `true` | Whether to re-send the prompt on restart |

**Returns:** `void`

```js
// Restart with prompt
hivemind.agents.restartById("agent-abc-123", true);

// Restart without re-sending the prompt
hivemind.agents.restartById("agent-abc-123", false);
```

---

#### `agents.restartWithPrompt(agentId, prompt)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Update an agent's prompt and restart it. The agent's stored prompt is updated before restarting.

| Parameter | Type | Description |
|-----------|------|-------------|
| `agentId` | `string` | Agent session UUID |
| `prompt` | `string` | New prompt to set before restarting |

**Returns:** `Promise<boolean>` -- `true` if the restart was initiated.

```js
const ok = await hivemind.agents.restartWithPrompt(
  "agent-abc-123",
  "Focus on the header component only. Use Tailwind CSS classes."
);
if (ok) {
  hivemind.toast("Agent restarted with new prompt", "success");
}
```

---

<a id="action-git"></a>
### Git Actions

#### `git.commit(worktreePath, message)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Stage all changes and create a commit in a worktree.

| Parameter | Type | Description |
|-----------|------|-------------|
| `worktreePath` | `string` | Absolute path to the git worktree |
| `message` | `string` | Commit message |

**Returns:** `Promise<CommitResult | null>`

See [CommitResult](#commitresult) type. Returns `null` on failure.

```js
const result = await hivemind.git.commit("/path/to/worktree", "feat: add dark mode toggle");
if (result) {
  hivemind.toast("Committed " + result.sha.substring(0, 7) + " (" + result.files_changed + " files)", "success");
}
```

---

#### `git.push(repoPath, branch)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Push a branch to the remote.

| Parameter | Type | Description |
|-----------|------|-------------|
| `repoPath` | `string` | Absolute path to the repository |
| `branch` | `string` | Branch name to push |

**Returns:** `Promise<boolean>` -- `true` if the push was initiated.

```js
const ok = await hivemind.git.push("/path/to/repo", "feature/dark-mode");
if (ok) {
  hivemind.toast("Branch pushed", "success");
}
```

---

#### `git.pull(repoPath, branch)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Pull a branch from the remote.

| Parameter | Type | Description |
|-----------|------|-------------|
| `repoPath` | `string` | Absolute path to the repository |
| `branch` | `string` | Branch name to pull |

**Returns:** `Promise<PullResult | null>`

See [PullResult](#pullresult) type. Returns `null` on failure.

```js
const result = await hivemind.git.pull("/path/to/repo", "main");
if (result) {
  if (result.up_to_date) {
    console.log("Already up to date");
  } else {
    console.log("Pulled " + result.commits_pulled + " commits. New HEAD: " + result.new_head);
  }
}
```

---

#### `git.createBranch(repoPath, name, from?)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Create a new branch.

| Parameter | Type | Description |
|-----------|------|-------------|
| `repoPath` | `string` | Absolute path to the repository |
| `name` | `string` | Name for the new branch |
| `from` | `string` | *(optional)* Source branch or commit to branch from. Defaults to HEAD. |

**Returns:** `Promise<BranchInfo | null>`

See [BranchInfo](#branchinfo) type. Returns `null` on failure.

```js
const branch = await hivemind.git.createBranch("/path/to/repo", "feature/new-widget", "main");
if (branch) {
  console.log("Created branch:", branch.name, "at", branch.commit_sha.substring(0, 7));
}
```

---

#### `git.checkout(repoPath, branch)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Checkout a branch.

| Parameter | Type | Description |
|-----------|------|-------------|
| `repoPath` | `string` | Absolute path to the repository |
| `branch` | `string` | Branch name to checkout |

**Returns:** `Promise<boolean>` -- `true` if the checkout was initiated.

```js
const ok = await hivemind.git.checkout("/path/to/repo", "main");
```

---

#### `git.fetch(repoPath)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Fetch from the remote.

| Parameter | Type | Description |
|-----------|------|-------------|
| `repoPath` | `string` | Absolute path to the repository |

**Returns:** `Promise<boolean>` -- `true` if the fetch was initiated.

```js
await hivemind.git.fetch("/path/to/repo");
console.log("Remote data fetched");
```

---

#### `git.revert(worktreePath)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Revert all uncommitted changes in a worktree (equivalent to `git checkout -- .`).

| Parameter | Type | Description |
|-----------|------|-------------|
| `worktreePath` | `string` | Absolute path to the git worktree |

**Returns:** `Promise<boolean>` -- `true` if the revert was initiated.

```js
if (confirm("Revert all changes?")) {
  await hivemind.git.revert("/path/to/worktree");
  hivemind.toast("Changes reverted", "info");
}
```

---

<a id="action-files"></a>
### File Actions

#### `files.write(filePath, content)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Write content to a file, creating it if it does not exist and overwriting if it does.

| Parameter | Type | Description |
|-----------|------|-------------|
| `filePath` | `string` | Absolute path to the file |
| `content` | `string` | File content to write |

**Returns:** `Promise<boolean>` -- `true` if the write was initiated.

```js
await hivemind.files.write(
  "/path/to/repo/.env.example",
  "DATABASE_URL=postgres://localhost:5432/mydb\nPORT=3000\n"
);
```

---

#### `files.delete(filePath)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Delete a file or directory.

| Parameter | Type | Description |
|-----------|------|-------------|
| `filePath` | `string` | Absolute path to the file or directory to delete |

**Returns:** `Promise<boolean>` -- `true` if the delete was initiated.

```js
await hivemind.files.delete("/path/to/repo/tmp/old-cache.json");
```

---

#### `files.create(filePath)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Create an empty file.

| Parameter | Type | Description |
|-----------|------|-------------|
| `filePath` | `string` | Absolute path for the new file |

**Returns:** `Promise<boolean>` -- `true` if the creation was initiated.

```js
await hivemind.files.create("/path/to/repo/src/components/NewWidget.tsx");
```

---

#### `files.mkdir(dirPath)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Create a directory (including parent directories if needed).

| Parameter | Type | Description |
|-----------|------|-------------|
| `dirPath` | `string` | Absolute path for the new directory |

**Returns:** `Promise<boolean>` -- `true` if the creation was initiated.

```js
await hivemind.files.mkdir("/path/to/repo/src/features/dark-mode");
```

---

#### `files.rename(from, to)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Rename a file or directory.

| Parameter | Type | Description |
|-----------|------|-------------|
| `from` | `string` | Absolute path of the existing entry |
| `to` | `string` | New name (not a full path -- just the new file/directory name) |

**Returns:** `Promise<string | null>` -- The new path if successful, or `null` on failure.

```js
const newPath = await hivemind.files.rename(
  "/path/to/repo/src/utils/helpers.js",
  "utils.js"
);
if (newPath) {
  console.log("Renamed to:", newPath);
  // => "/path/to/repo/src/utils/utils.js"
}
```

---

<a id="action-pull-requests"></a>
### Pull Request Actions

#### `pr.create(repoId, title, body, branch, target)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Create a new pull request.

| Parameter | Type | Description |
|-----------|------|-------------|
| `repoId` | `string` | Repository UUID |
| `title` | `string` | PR title |
| `body` | `string` | PR description (Markdown) |
| `branch` | `string` | Head branch (source) |
| `target` | `string` | Base branch (target) |

**Returns:** `Promise<{ url: string, number: number } | null>`

Returns `null` on failure.

```js
const pr = await hivemind.pr.create(
  "repo-abc-123",
  "feat: add dark mode toggle",
  "## Changes\n- Added dark mode toggle to settings\n- Persists preference in localStorage",
  "feature/dark-mode",
  "main"
);
if (pr) {
  hivemind.toast("PR #" + pr.number + " created!", "success");
  console.log("PR URL:", pr.url);
}
```

---

<a id="action-settings"></a>
### Settings Actions

#### `settings.set(key, value)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Set a setting value.

| Parameter | Type | Description |
|-----------|------|-------------|
| `key` | `string` | Setting key |
| `value` | `string` | Setting value |

**Returns:** `Promise<boolean>` -- `true` if the setting was saved.

```js
await hivemind.settings.set("appearance.theme", "dark");
```

---

<a id="action-macros"></a>
### Macro Actions

#### `macros.save(repoId, name, lanesJson)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Save a new run macro or update an existing one.

| Parameter | Type | Description |
|-----------|------|-------------|
| `repoId` | `string` | Repository UUID |
| `name` | `string` | Macro name |
| `lanesJson` | `string` | JSON string of the lanes array (see [MacroLane](#macrolane)) |

**Returns:** `Promise<RunMacro | null>`

See [RunMacro](#runmacro) type. Returns `null` on failure.

```js
const lanes = [
  {
    id: "lane-1",
    name: "Backend",
    commands: "npm run dev:server",
    dependsOn: [],
    readySignal: "Server listening on port 3000"
  },
  {
    id: "lane-2",
    name: "Frontend",
    commands: "npm run dev",
    dependsOn: ["lane-1"],
    readySignal: "ready in"
  }
];
const macro = await hivemind.macros.save("repo-abc-123", "Dev Stack", JSON.stringify(lanes));
if (macro) {
  console.log("Macro saved:", macro.id);
}
```

---

#### `macros.delete(macroId)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Delete a run macro.

| Parameter | Type | Description |
|-----------|------|-------------|
| `macroId` | `string` | Macro UUID |

**Returns:** `Promise<boolean>` -- `true` if the macro was deleted.

```js
const ok = await hivemind.macros.delete("macro-abc-123");
if (ok) {
  hivemind.toast("Macro deleted", "info");
}
```

---

<a id="action-terminal"></a>
### Terminal Actions

#### `terminal.spawn(cwd)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Spawn a new terminal session with a PTY.

| Parameter | Type | Description |
|-----------|------|-------------|
| `cwd` | `string` | Working directory for the terminal |

**Returns:** `Promise<string>` -- The terminal session ID (prefixed with `"plugin-"`).

```js
const termId = await hivemind.terminal.spawn("/path/to/repo");
console.log("Terminal spawned:", termId);
```

---

#### `terminal.send(sessionId, input)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Send input text to a terminal session.

| Parameter | Type | Description |
|-----------|------|-------------|
| `sessionId` | `string` | Terminal session ID (from `terminal.spawn()`) |
| `input` | `string` | Input to send (include `\n` for enter) |

**Returns:** `Promise<boolean>` -- `true` if the input was sent.

```js
const termId = await hivemind.terminal.spawn("/path/to/repo");
await hivemind.terminal.send(termId, "npm test\n");
```

---

#### `terminal.kill(sessionId)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Kill a terminal session.

| Parameter | Type | Description |
|-----------|------|-------------|
| `sessionId` | `string` | Terminal session ID |

**Returns:** `Promise<boolean>` -- `true` if the kill was initiated.

```js
await hivemind.terminal.kill(termId);
console.log("Terminal killed");
```

---

<a id="action-navigation"></a>
### Navigation Actions

#### `navigate.toSettings(section?)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Navigate the IDE to the settings panel.

| Parameter | Type | Description |
|-----------|------|-------------|
| `section` | `string` | *(optional)* Settings section to open |

**Returns:** `void`

```js
hivemind.navigate.toSettings("appearance");
hivemind.navigate.toSettings();  // opens settings without a specific section
```

---

#### `navigate.toRepo(repoId)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Navigate the IDE to a specific repository tab.

| Parameter | Type | Description |
|-----------|------|-------------|
| `repoId` | `string` | Repository UUID |

**Returns:** `void`

```js
hivemind.navigate.toRepo("repo-abc-123");
```

---

<a id="action-clipboard"></a>
### Clipboard Actions

#### `clipboard.write(text)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Write text to the system clipboard. Fire-and-forget.

| Parameter | Type | Description |
|-----------|------|-------------|
| `text` | `string` | Text to write to clipboard |

**Returns:** `void`

```js
hivemind.clipboard.write("Copied from my plugin!");
hivemind.toast("Copied to clipboard", "success");
```

---

## 4. Backend Runtime API (QuickJS)

These methods are available on the global `hivemind` object inside `main.js`, which runs in a sandboxed QuickJS context on the Rust backend.

---

#### `hivemind.log(message)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Log a message. The message is printed to stdout with the plugin ID prefix and emitted as a `log` event to the frontend.

| Parameter | Type | Description |
|-----------|------|-------------|
| `message` | `string` | Message to log |

**Returns:** `void`

```js
hivemind.log("Plugin initialized successfully");
// => [Plugin:my-plugin] Plugin initialized successfully
```

---

#### `hivemind.on(event, callback)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Register a hook listener. The callback is invoked when the specified event fires. Multiple callbacks can be registered for the same event. Callbacks are called synchronously in registration order.

| Parameter | Type | Description |
|-----------|------|-------------|
| `event` | `string` | Hook event name (see [Hooks](#1-hooks-backend-runtime)) |
| `callback` | `function(data)` | Callback receiving the event payload |

**Returns:** `void`

```js
hivemind.on("agent:launched", function(data) {
  hivemind.log("Agent launched: " + data.agent_id + " on " + data.branch);
});

hivemind.on("git:commit", function(data) {
  hivemind.log("Commit: " + data.sha);
});
```

---

#### `hivemind.registerRoute(path, handler)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Register an HTTP route handler. Agents in Docker containers can call these routes via the local API server. Routes support glob-style matching with `*` at the end.

| Parameter | Type | Description |
|-----------|------|-------------|
| `path` | `string` | Route path pattern (e.g., `"/plugins/my-plugin/*"`) |
| `handler` | `function(request)` | Handler function that receives a request object |

The `request` object contains:
| Field | Type | Description |
|-------|------|-------------|
| `method` | `string` | HTTP method (e.g., `"POST"`, `"GET"`) |
| `path` | `string` | Full request path |
| `body` | `object` | Parsed JSON body |
| `headers` | `object` | Request headers as key-value pairs |

The handler must return a response object:
| Field | Type | Description |
|-------|------|-------------|
| `status` | `number` | HTTP status code |
| `body` | `object` | Response body (will be serialized to JSON) |
| `headers` | `object` | *(optional)* Response headers |

**Returns:** `void`

```js
hivemind.registerRoute("/plugins/my-plugin/*", function(req) {
  if (req.path === "/plugins/my-plugin/status") {
    return { status: 200, body: { running: true, version: "1.0.0" } };
  }

  if (req.path === "/plugins/my-plugin/log" && req.method === "POST") {
    var message = req.body.message || "(no message)";
    hivemind.log("Agent says: " + message);
    return { status: 200, body: { ok: true } };
  }

  return { status: 404, body: { error: "Not found: " + req.path } };
});
```

---

#### `hivemind.emit(event, data)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Emit a custom event to the frontend. UI plugins can listen for these events using `hivemind.on(event, callback)` in their iframe.

| Parameter | Type | Description |
|-----------|------|-------------|
| `event` | `string` | Event name (convention: prefix with your plugin ID) |
| `data` | `object` | Event payload |

**Returns:** `void`

```js
// In main.js (backend)
hivemind.on("agent:completed", function(data) {
  hivemind.emit("my-plugin:agent-done", {
    agent_id: data.agent_id,
    summary: data.summary,
    timestamp: new Date().toISOString()
  });
});

// In sidebar.html (frontend), the UI plugin can listen:
// hivemind.on("my-plugin:agent-done", function(data) { ... });
```

---

#### `hivemind.storage.get(key)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Get a value from plugin-scoped storage. This is a **synchronous** call in the backend runtime.

| Parameter | Type | Description |
|-----------|------|-------------|
| `key` | `string` | Storage key |

**Returns:** `string | null`

```js
var raw = hivemind.storage.get("event_log");
var log;
try { log = raw ? JSON.parse(raw) : []; } catch(e) { log = []; }
```

---

#### `hivemind.storage.set(key, value)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Set a value in plugin-scoped storage. This is a **synchronous** call in the backend runtime.

| Parameter | Type | Description |
|-----------|------|-------------|
| `key` | `string` | Storage key |
| `value` | `string` | Value to store |

**Returns:** `void`

```js
hivemind.storage.set("event_log", JSON.stringify(log));
```

---

#### `hivemind.agents.launch(opts)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Launch an agent from the backend runtime. Emits a `request_agent_launch` event that the frontend handles.

| Parameter | Type | Description |
|-----------|------|-------------|
| `opts` | `object` | Launch options |
| `opts.repoId` | `string` | Repository UUID (required) |
| `opts.branch` | `string` | Branch name (required) |
| `opts.originBranch` | `string` | *(optional)* Base branch |
| `opts.prompt` | `string` | Agent prompt (required) |

**Returns:** `void`

```js
hivemind.agents.launch({
  repoId: "repo-abc-123",
  branch: "fix/auto-patch",
  originBranch: "main",
  prompt: "Fix the failing test in src/utils.test.ts"
});
```

---

#### `hivemind.agents.stop(branch)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Stop an agent by branch name from the backend runtime. Emits a `plugin_stop_agent` event.

| Parameter | Type | Description |
|-----------|------|-------------|
| `branch` | `string` | Branch name the agent is working on |

**Returns:** `void`

```js
hivemind.agents.stop("fix/auto-patch");
```

---

#### `hivemind.toast(message, type?)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Show a toast notification from the backend runtime. Emits a `plugin_toast` event.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `message` | `string` | | Notification message |
| `type` | `string` | `"info"` | Toast type: `"info"`, `"success"`, or `"error"` |

**Returns:** `void`

```js
hivemind.toast("Build completed!", "success");
hivemind.toast("Something went wrong", "error");
```

---

#### `hivemind.registerInstructions(text)`

> ![UNTESTED](https://img.shields.io/badge/status-UNTESTED-red)

Register custom instructions that will be injected into every agent's prompt. Useful for teaching agents about your plugin's capabilities (e.g., available shell scripts or API routes).

| Parameter | Type | Description |
|-----------|------|-------------|
| `text` | `string` | Instruction text (plain text, will be appended to agent prompts) |

**Returns:** `void`

```js
hivemind.registerInstructions(
  "The My Plugin is installed.\n" +
  "You can send status updates to the plugin:\n" +
  "  curl -X POST http://host.docker.internal:$HIVEMIND_PORT/hivemind/v1/plugins/my-plugin/status \\\n" +
  "    -H 'Content-Type: application/json' \\\n" +
  "    -d '{\"message\": \"Working on task...\"}'\n"
);
```

---

## 5. Type Reference

This section defines the data types returned by bridge APIs and action calls. All types are serialized as JSON over the `postMessage` bridge.

---

<a id="repository"></a>
### Repository

Represents a repository registered in Hivemind.

```typescript
interface Repository {
  id: string;                    // UUID
  provider_id: string | null;    // Git provider ID (null for local-only repos)
  name: string;                  // Repository name
  full_name: string;             // Full name (e.g., "owner/repo")
  clone_url: string;             // Git clone URL
  local_path: string | null;     // Absolute path on disk (null if not cloned)
  is_local_only: boolean;        // True if not linked to a remote provider
  default_branch: string;        // Default branch name (e.g., "main")
  group_id: string | null;       // Repository group ID
  sort_order: number;            // Sort position within group
  created_at: string;            // ISO 8601 timestamp
}
```

---

<a id="repositorygroup"></a>
### RepositoryGroup

Represents a group of repositories for organizational purposes.

```typescript
interface RepositoryGroup {
  id: string;           // UUID
  name: string;         // Group name
  sort_order: number;   // Sort position
}
```

---

<a id="agentsession"></a>
### AgentSession

Represents an agent session (running, stopped, or completed).

```typescript
interface AgentSession {
  id: string;                      // UUID
  repo_id: string;                 // Repository UUID
  branch: string;                  // Branch name the agent works on
  status: string;                  // "running" | "idle" | "stopped" | "paused" | "error" | "destroyed"
  provider: string;                // AI provider ID
  model: string | null;            // AI model name
  instructions: string | null;     // Custom instructions
  mode: string;                    // "coder" | "bug-fixer" | "code-reviewer" | etc.
  prompt: string;                  // Task prompt
  worktree_path: string | null;    // Path to agent's git worktree
  attached: boolean;               // Whether agent is attached to a terminal
  auto_accept: boolean;            // Whether agent auto-accepts tool calls
  origin_branch: string | null;    // Base branch the agent branched from
  allocated_ports: string | null;  // JSON string of allocated ports
  api_token: string | null;        // Agent's API token
  created_at: string;              // ISO 8601 timestamp
  updated_at: string;              // ISO 8601 timestamp
}
```

---

<a id="branchinfo"></a>
### BranchInfo

Represents a git branch.

```typescript
interface BranchInfo {
  name: string;              // Branch name (e.g., "main", "feature/foo")
  is_remote: boolean;        // True if this is a remote-tracking branch
  is_head: boolean;          // True if this is the currently checked-out branch
  commit_sha: string;        // Full SHA of the branch tip commit
  commit_message: string;    // Message of the tip commit
  commit_date: number;       // Unix timestamp of the tip commit
  upstream: string | null;   // Upstream branch name (e.g., "origin/main")
  ahead: number;             // Commits ahead of upstream
  behind: number;            // Commits behind upstream
}
```

---

<a id="commitlogentry"></a>
### CommitLogEntry

Represents a commit in the log history.

```typescript
interface CommitLogEntry {
  sha: string;                 // Full commit SHA
  author: string;              // Author name
  date: string;                // ISO 8601 date string
  message: string;             // Commit message
  files: CommitFileEntry[];    // Files changed in this commit
}
```

---

<a id="commitfileentry"></a>
### CommitFileEntry

Represents a file changed in a commit.

```typescript
interface CommitFileEntry {
  path: string;       // File path relative to repo root
  status: string;     // "Added" | "Modified" | "Deleted" | "Renamed"
  additions: number;  // Lines added
  deletions: number;  // Lines deleted
}
```

---

<a id="blamelineinfo"></a>
### BlameLineInfo

Git blame information for a single line.

```typescript
interface BlameLineInfo {
  author: string;         // Author name
  author_email: string;   // Author email
  sha: string;            // Full commit SHA
  short_sha: string;      // Short SHA (7 chars)
  timestamp: number;      // Unix timestamp
  committed: boolean;     // True if the line has been committed (not uncommitted)
}
```

---

<a id="filestatusentry"></a>
### FileStatusEntry

An uncommitted file change in a worktree.

```typescript
interface FileStatusEntry {
  path: string;     // File path relative to worktree root
  status: string;   // "Added" | "Modified" | "Deleted" | "Renamed"
}
```

---

<a id="commitresult"></a>
### CommitResult

Result of a commit operation.

```typescript
interface CommitResult {
  sha: string;           // Commit SHA
  files_changed: number; // Number of files changed
}
```

---

<a id="pullresult"></a>
### PullResult

Result of a pull operation.

```typescript
interface PullResult {
  up_to_date: boolean;     // True if already up to date
  commits_pulled: number;  // Number of new commits pulled
  new_head: string;        // SHA of the new HEAD after pull
}
```

---

<a id="filediff"></a>
### FileDiff

A file-level diff containing one or more hunks.

```typescript
interface FileDiff {
  path: string;                   // File path
  old_path: string | null;        // Previous path (for renames)
  status: FileStatus;             // "Added" | "Modified" | "Deleted" | "Renamed"
  hunks: DiffHunk[];              // Diff hunks
  binary: boolean;                // True if file is binary
  truncated: boolean;             // True if diff was truncated
}
```

---

### DiffHunk

A single hunk within a file diff.

```typescript
interface DiffHunk {
  header: string;        // Hunk header (e.g., "@@ -10,5 +10,7 @@")
  lines: DiffLine[];     // Lines in the hunk
}
```

---

### DiffLine

A single line within a diff hunk.

```typescript
interface DiffLine {
  kind: DiffLineKind;            // "Context" | "Addition" | "Deletion"
  content: string;               // Line content
  old_line_no: number | null;    // Line number in the old file
  new_line_no: number | null;    // Line number in the new file
}
```

---

<a id="direntry"></a>
### DirEntry

A directory entry (file or subdirectory).

```typescript
interface DirEntry {
  name: string;                 // File/directory name
  path: string;                 // Absolute path
  is_dir: boolean;              // True if directory
  size: number;                 // Size in bytes (0 for directories)
  extension: string | null;     // File extension (e.g., "ts", "rs")
}
```

---

<a id="searchresult"></a>
### SearchResult

A search result from file search.

```typescript
interface SearchResult {
  path: string;                   // Absolute path to the file
  file_name: string;              // File name
  match_type: string;             // "filename" | "content"
  line_number: number | null;     // Line number for content matches
  snippet: string | null;         // Matching snippet for content matches
}
```

---

<a id="pullrequestinfo"></a>
### PullRequestInfo

Summary information about a pull request.

```typescript
interface PullRequestInfo {
  number: number;                       // PR number
  title: string;                        // PR title
  url: string;                          // PR URL
  head_branch: string;                  // Source branch
  base_branch: string;                  // Target branch
  author: string;                       // Author username
  author_avatar_url: string | null;     // Author avatar URL
  draft: boolean;                       // True if draft PR
}
```

---

<a id="pullrequestdetail"></a>
### PullRequestDetail

Detailed information about a pull request.

```typescript
interface PullRequestDetail {
  number: number;                       // PR number
  title: string;                        // PR title
  body: string;                         // PR description (Markdown)
  url: string;                          // PR URL
  head_branch: string;                  // Source branch
  base_branch: string;                  // Target branch
  author: string;                       // Author username
  author_avatar_url: string | null;     // Author avatar URL
  draft: boolean;                       // True if draft PR
  additions: number;                    // Total lines added
  deletions: number;                    // Total lines deleted
  changed_files: number;                // Number of files changed
  head_sha: string;                     // SHA of the head commit
  mergeable: boolean | null;            // Merge status (null if unknown)
}
```

---

<a id="runmacro"></a>
### RunMacro

A saved run macro with terminal lanes.

```typescript
interface RunMacro {
  id: string;                  // UUID
  repo_id: string;             // Repository UUID
  name: string;                // Macro name
  lanes: MacroLane[];          // Terminal lanes
  sort_order: number;          // Sort position
  created_at: string;          // ISO 8601 timestamp
  updated_at: string;          // ISO 8601 timestamp
}
```

---

<a id="macrolane"></a>
### MacroLane

A single lane within a run macro.

```typescript
interface MacroLane {
  id: string;                         // Lane ID
  name: string;                       // Lane display name
  commands: string;                   // Shell commands to run
  dependsOn: string[];                // IDs of lanes this depends on
  readySignal?: string;               // Output pattern indicating lane is ready
  teardownCommands?: string;          // Commands to run on shutdown
  killDelay?: number;                 // Delay (ms) before force-killing
}
```

---

<a id="plugin"></a>
### Plugin

Represents an installed plugin.

```typescript
interface Plugin {
  id: string;                // Plugin ID (e.g., "hivemind-debug")
  name: string;              // Display name
  version: string;           // Semver version
  description: string;       // Plugin description
  author: string;            // Author name
  enabled: boolean;          // Whether the plugin is enabled
  manifest_json: string;     // Raw manifest.json content
  installed_at: string;      // ISO 8601 timestamp
  updated_at: string;        // ISO 8601 timestamp
  source: string;            // Installation source
}
```

---

<a id="aiproviderconfig"></a>
### AiProviderConfig

Active AI provider configuration.

```typescript
interface AiProviderConfig {
  id: string;                // Config UUID
  provider_id: string;       // Provider ID (e.g., "claude-code", "codex")
  auth_method: string;       // Authentication method used
  api_key_stored: boolean;   // Whether an API key is stored
  is_default: boolean;       // Whether this is the default provider
  created_at: string;        // ISO 8601 timestamp
}
```
