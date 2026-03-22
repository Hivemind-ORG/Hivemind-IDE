# Hivemind Plugin Development Guide

This guide covers everything you need to create, package, and distribute plugins for Hivemind IDE.

## Overview

Plugins extend Hivemind through:

- **Hooks** — Run JavaScript code when events happen (agent launched, git commit, etc.)
- **Routes** — Add custom API endpoints that agents can call
- **UI Panels** — Add sidebar panels and full-content tabs rendered as HTML in iframes
- **Agent Scripts** — Add shell commands that agents can call from the terminal

Plugins run trusted JavaScript via an embedded QuickJS runtime on the backend. UI panels run in sandboxed iframes with a bridge API for communication with Hivemind.

---

## Plugin Package Structure

A plugin is distributed as a `.zip` archive containing a single root directory:

```
my-plugin/
  manifest.json          # Required — plugin metadata
  main.js                # Optional — backend JavaScript (hooks, routes)
  ui/
    sidebar.html         # Optional — sidebar panel HTML
    panel.html           # Optional — main content tab HTML
  scripts/
    hivemind-my-plugin-do-thing   # Optional — agent CLI scripts
```

---

## manifest.json

The manifest declares your plugin's metadata and capabilities.

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "version": "1.0.0",
  "description": "A brief description of what this plugin does",
  "author": "Your Name",
  "hivemind_version": ">=0.9.0",
  "main": "main.js",
  "ui": {
    "sidebar": "ui/sidebar.html",
    "panel": "ui/panel.html"
  },
  "sidebar_icon": "extension",
  "sidebar_label": "My Plugin",
  "scripts": [
    "scripts/hivemind-my-plugin-do-thing"
  ],
  "hooks": [
    "agent:launched",
    "agent:completed",
    "git:commit"
  ],
  "routes": [
    "/plugins/my-plugin/*"
  ]
}
```

### Required Fields

| Field | Description |
|-------|-------------|
| `id` | Unique plugin identifier. Must be lowercase kebab-case (`^[a-z][a-z0-9-]*$`). |
| `name` | Human-readable plugin name. |
| `version` | Semantic version string (e.g., `"1.0.0"`). |

### Optional Fields

| Field | Description |
|-------|-------------|
| `description` | Brief description shown in the plugin list. |
| `author` | Author name shown in the plugin list. |
| `hivemind_version` | Minimum Hivemind version required. |
| `main` | Path to the backend JavaScript file (relative to plugin root). |
| `ui.sidebar` | Path to the sidebar panel HTML file. |
| `ui.panel` | Path to the main content panel HTML file. |
| `sidebar_icon` | [Material Symbols](https://fonts.google.com/icons) icon name for the sidebar button. Default: `"extension"`. |
| `sidebar_label` | Label shown in the sidebar header when this plugin's panel is active. |
| `scripts` | Array of shell script paths (relative to plugin root). |
| `hooks` | Array of hook event names this plugin listens to (informational). |
| `routes` | Array of route patterns this plugin handles (informational). |

---

## Backend JavaScript API (main.js)

Your `main.js` runs in QuickJS on the Rust backend. It has access to the `hivemind` global object.

### hivemind.on(event, callback)

Register a hook listener. The callback receives the event data as a plain object.

```javascript
hivemind.on('agent:launched', function(data) {
  hivemind.log('Agent ' + data.agent_id + ' launched on branch ' + data.branch);
});

hivemind.on('git:commit', function(data) {
  hivemind.log('Commit ' + data.sha + ' by agent ' + data.agent_id);
});
```

### hivemind.registerRoute(path, handler)

Register an HTTP route handler. When an agent makes a request to an unregistered route, Hivemind checks plugin handlers. The handler receives a request object and must return a response object.

```javascript
hivemind.registerRoute('/plugins/my-plugin/*', function(req) {
  // req = { method: "POST", path: "/plugins/my-plugin/action", body: {...}, headers: {...} }

  if (req.path === '/plugins/my-plugin/greet') {
    return {
      status: 200,
      body: { message: 'Hello from my plugin!' }
    };
  }

  return { status: 404, body: { error: 'Unknown action' } };
});
```

**Request object:**

| Field | Type | Description |
|-------|------|-------------|
| `method` | string | HTTP method (GET, POST, etc.) |
| `path` | string | Full request path |
| `body` | object | Parsed JSON body |
| `headers` | object | Request headers |

**Response object:**

| Field | Type | Description |
|-------|------|-------------|
| `status` | number | HTTP status code |
| `body` | object | Response body (will be JSON-serialized) |

### hivemind.storage

Key-value storage persisted across restarts. Each plugin gets its own isolated namespace.

```javascript
hivemind.storage.set('counter', '42');
var value = hivemind.storage.get('counter');  // "42" or null
hivemind.storage.delete('counter');
```

> Note: All values are stored as strings. Serialize objects with `JSON.stringify()`.

### hivemind.emit(event, data)

Emit a custom event to the frontend. Events are namespaced as `plugin:{pluginId}:{event}`.

```javascript
hivemind.emit('status-update', { progress: 75 });
```

Frontend plugin UIs can listen for these events via the bridge API (see UI section below).

### hivemind.log(message)

Log a message to stdout with the plugin's name as prefix.

```javascript
hivemind.log('Plugin initialized');
// Output: [Plugin:my-plugin] Plugin initialized
```

### hivemind.agents.launch(options)

Launch an agent on a branch. If the branch doesn't exist, it will be created from `originBranch`.

```javascript
hivemind.agents.launch({
  repoId: 'repository-uuid',
  branch: 'fix/my-bug',
  originBranch: 'main',
  prompt: 'Fix the authentication bug in login.ts'
});
```

| Option | Type | Required | Description |
|--------|------|----------|-------------|
| `repoId` | string | Yes | Repository UUID |
| `branch` | string | Yes | Branch name to work on (created if it doesn't exist) |
| `originBranch` | string | No | Branch to create from (defaults to `branch`) |
| `prompt` | string | Yes | Instructions for the agent |

### hivemind.agents.stop(branch)

Stop the running agent on the specified branch.

```javascript
hivemind.agents.stop('fix/my-bug');
```

### hivemind.toast(message, type?)

Show a toast notification to the user.

```javascript
hivemind.toast('Deployment complete!', 'success');
hivemind.toast('Check the logs', 'info');
hivemind.toast('Build failed', 'error');
```

| Argument | Type | Description |
|----------|------|-------------|
| `message` | string | Toast message text |
| `type` | string | `"info"` (default), `"success"`, or `"error"` |

---

## Available Hook Events

### Agent Lifecycle

| Event | Payload | When |
|-------|---------|------|
| `agent:launched` | `{ agent_id, repo_id, branch, mode }` | Agent session created and process started |
| `agent:completed` | `{ agent_id, summary }` | Agent reports task completion |
| `agent:stopped` | `{ agent_id, status }` | Agent stopped (user action or normal exit) |
| `agent:error` | `{ agent_id, status }` | Agent stopped due to error |
| `app:started` | `{}` | All plugins loaded and app ready |

### Git Operations

| Event | Payload | When |
|-------|---------|------|
| `git:commit` | `{ agent_id, sha }` | Agent committed changes |
| `git:push` | `{ agent_id, force }` | Agent pushed branch |
| `git:merge` | `{ agent_id, success }` | Agent merged a branch |
| `git:rebase` | `{ agent_id, success }` | Agent rebased onto a branch |
| `git:pr_created` | `{ agent_id, url, number }` | Agent created a pull request |

### UI Events (Frontend-Triggered)

UI events can be fired from the frontend via `firePluginHook()`:

| Event | Payload |
|-------|---------|
| `app:settings_opened` | `{}` |
| `sidebar:tab_changed` | `{ tab }` |

---

## UI Panels

Plugins can provide HTML-based UI panels that render in sandboxed iframes.

### Sidebar Panel

Shown in the left sidebar when the user clicks your plugin's icon in the bottom bar. Defined by `ui.sidebar` in the manifest.

### Main Content Panel

A full-width tab in the main content area. Defined by `ui.panel` in the manifest. Opened via `hivemind.openTab()` from the bridge API.

### HTML Structure

Plugin HTML files are standard HTML documents. Hivemind injects a bridge script that provides `window.hivemind` in the iframe.

```html
<!DOCTYPE html>
<html>
<head>
  <style>
    body {
      font-family: system-ui, sans-serif;
      margin: 0;
      padding: 12px;
      color: #e0e0e0;
      background: transparent;
    }
  </style>
</head>
<body>
  <h3>My Plugin</h3>
  <button id="open-panel">Open Full Panel</button>
  <div id="status"></div>

  <script>
    document.getElementById('open-panel').addEventListener('click', function() {
      hivemind.openTab();
    });

    // Listen for backend events
    hivemind.on('status-update', function(data) {
      document.getElementById('status').textContent = 'Progress: ' + data.progress + '%';
    });

    // Use storage
    hivemind.storage.get('lastVisit').then(function(value) {
      if (value) {
        console.log('Last visit: ' + value);
      }
      hivemind.storage.set('lastVisit', new Date().toISOString());
    });
  </script>
</body>
</html>
```

### Frontend Bridge API (window.hivemind)

Available inside plugin iframes:

| Method | Description |
|--------|-------------|
| `hivemind.openTab()` | Open this plugin's main content tab |
| `hivemind.storage.get(key)` | Get a stored value (returns Promise) |
| `hivemind.storage.set(key, value)` | Store a value |
| `hivemind.storage.delete(key)` | Delete a stored value |
| `hivemind.on(event, callback)` | Listen for events from the backend |
| `hivemind.pluginId` | This plugin's ID (string) |

> Note: `storage.get()` is asynchronous in the iframe (returns a Promise) because it communicates with the host via postMessage.

---

## Agent CLI Scripts

Plugins can provide shell scripts that agents call from the terminal. Scripts must follow the naming convention:

```
hivemind-{pluginId}-{command}
```

For example, a plugin with ID `docker` might provide:
- `hivemind-docker-build`
- `hivemind-docker-run`
- `hivemind-docker-stop`

### Script Environment

When an agent runs, these environment variables are available:

| Variable | Description |
|----------|-------------|
| `HIVEMIND_API_URL` | Base URL of the Hivemind API (e.g., `http://localhost:20042`) |
| `HIVEMIND_AGENT_ID` | The agent's unique ID |
| `HIVEMIND_API_TOKEN` | Bearer token for API authentication |

### Example Script

```bash
#!/bin/sh
# hivemind-my-plugin-greet — Call the plugin's custom API endpoint

MESSAGE="${1:?Usage: hivemind-my-plugin-greet MESSAGE}"

curl -sf -X POST "${HIVEMIND_API_URL}/plugins/my-plugin/greet" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${HIVEMIND_API_TOKEN}" \
  -d "{\"message\":\"${MESSAGE}\"}"
```

### How It Works

1. Plugin scripts are placed in the `scripts/` directory of your plugin
2. On install, Hivemind copies them to the `hivemind-api/bin/` directory
3. This directory is on the agent's PATH, so agents can call them directly
4. Scripts typically use `curl` to call your plugin's registered routes

---

## Tutorial: Building a Simple Plugin

Let's build a "hello-world" plugin that logs agent completions and provides a sidebar panel.

### Step 1: Create the directory structure

```
hello-world/
  manifest.json
  main.js
  ui/
    sidebar.html
```

### Step 2: Write manifest.json

```json
{
  "id": "hello-world",
  "name": "Hello World",
  "version": "1.0.0",
  "description": "A simple example plugin",
  "author": "You",
  "main": "main.js",
  "ui": {
    "sidebar": "ui/sidebar.html"
  },
  "sidebar_icon": "waving_hand",
  "sidebar_label": "Hello",
  "hooks": ["agent:completed"]
}
```

### Step 3: Write main.js

```javascript
hivemind.log('Hello World plugin loaded!');

// Track completions
hivemind.on('agent:completed', function(data) {
  var count = parseInt(hivemind.storage.get('completions') || '0', 10);
  count++;
  hivemind.storage.set('completions', String(count));
  hivemind.log('Agent ' + data.agent_id + ' completed! Total: ' + count);

  // Notify frontend
  hivemind.emit('completion-count', { count: count });
});
```

### Step 4: Write ui/sidebar.html

```html
<!DOCTYPE html>
<html>
<head>
  <style>
    body {
      font-family: system-ui, sans-serif;
      margin: 0;
      padding: 16px;
      color: #d4d4d8;
      background: transparent;
    }
    .count {
      font-size: 48px;
      font-weight: bold;
      text-align: center;
      margin: 32px 0;
      color: #a78bfa;
    }
    .label {
      text-align: center;
      font-size: 12px;
      color: #71717a;
    }
  </style>
</head>
<body>
  <div class="count" id="count">-</div>
  <div class="label">Agent Completions</div>

  <script>
    // Load initial count
    hivemind.storage.get('completions').then(function(val) {
      document.getElementById('count').textContent = val || '0';
    });

    // Listen for updates
    hivemind.on('completion-count', function(data) {
      document.getElementById('count').textContent = String(data.count);
    });
  </script>
</body>
</html>
```

### Step 5: Package and install

```bash
# Create the zip (from the parent directory of hello-world/)
zip -r hello-world.zip hello-world/

# Install via Hivemind:
# Settings → Plugins → Import Plugin → select hello-world.zip
```

---

## Packaging and Distribution

### Creating a Plugin Archive

1. Ensure your plugin directory matches the `id` in manifest.json
2. Zip the directory from its parent:

```bash
cd /path/to/plugins
zip -r my-plugin.zip my-plugin/
```

3. The resulting `my-plugin.zip` should contain:
   ```
   my-plugin/manifest.json
   my-plugin/main.js
   my-plugin/ui/sidebar.html
   ...
   ```

### Installing a Plugin

1. Open Hivemind Settings (gear icon in sidebar)
2. Go to the **Plugins** section
3. Click **Import Plugin**
4. Select the `.zip` file
5. Confirm the trust dialog
6. The plugin loads immediately

### Plugin Storage Location

Installed plugins are stored at:
- **macOS/Linux:** `~/.config/com.hivemind.ide/plugins/{pluginId}/`
- **Windows:** `%APPDATA%/com.hivemind.ide/plugins/{pluginId}/`

### Managing Plugins

- **Enable/Disable:** Toggle the switch next to each plugin in Settings → Plugins
- **Uninstall:** Click the delete icon next to a plugin

When disabled, a plugin's hooks stop firing, routes stop responding, and sidebar icons are hidden. The plugin files remain on disk until uninstalled.

---

## API Reference Summary

### Backend (main.js — QuickJS)

| API | Description |
|-----|-------------|
| `hivemind.on(event, callback)` | Register hook listener |
| `hivemind.registerRoute(path, handler)` | Register HTTP route handler |
| `hivemind.storage.get(key)` | Read from KV store (synchronous) |
| `hivemind.storage.set(key, value)` | Write to KV store |
| `hivemind.storage.delete(key)` | Delete from KV store |
| `hivemind.emit(event, data)` | Emit event to frontend |
| `hivemind.log(message)` | Log with plugin prefix |
| `hivemind.agents.launch(opts)` | Launch an agent on a branch |
| `hivemind.agents.stop(branch)` | Stop agent on a branch |
| `hivemind.toast(message, type?)` | Show a toast notification |
| `hivemind.pluginId` | This plugin's ID |

### Frontend (iframe — window.hivemind)

| API | Description |
|-----|-------------|
| `hivemind.openTab()` | Open main content tab |
| `hivemind.storage.get(key)` | Read from KV store (async, returns Promise) |
| `hivemind.storage.set(key, value)` | Write to KV store |
| `hivemind.storage.delete(key)` | Delete from KV store |
| `hivemind.on(event, callback)` | Listen for backend events |
| `hivemind.agents.launch(opts)` | Launch an agent on a branch |
| `hivemind.agents.stop(branch)` | Stop agent on a branch |
| `hivemind.toast(message, type?)` | Show a toast notification |
| `hivemind.pluginId` | This plugin's ID |
