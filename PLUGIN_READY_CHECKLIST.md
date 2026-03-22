# Plugin Ready Checklist

Use this checklist before submitting or releasing a plugin to verify it meets all technical requirements.

## Manifest (`manifest.json`)

- [ ] File exists at the root of the plugin directory
- [ ] `id` is lowercase kebab-case (e.g. `my-plugin`, not `MyPlugin` or `my_plugin`)
- [ ] `name` is set and non-empty
- [ ] `version` is set in semver format (`X.Y.Z`)
- [ ] `description` provides a short summary of what the plugin does
- [ ] `author` is set
- [ ] `hivemind_version` specifies the minimum compatible version (e.g. `>=0.9.0`)

## Entry Points

- [ ] If `main` is set, the referenced `.js` file exists at the specified path
- [ ] If `ui.sidebar` is set, the referenced `.html` file exists
- [ ] If `ui.panel` is set, the referenced `.html` file exists

## Scripts

- [ ] Each path listed in `scripts` points to an existing file in the plugin directory
- [ ] Script files start with a valid shebang (e.g. `#!/bin/sh`)
- [ ] Script files use LF line endings (not CRLF)
- [ ] Script files are executable (`chmod +x`)

## Hooks & Routes

- [ ] Each entry in `hooks` is a valid event name (e.g. `app:started`, `agent:launched`, `git:commit`)
- [ ] Each entry in `routes` uses a valid path pattern (e.g. `/plugins/{plugin-id}/*`)
- [ ] Route patterns are scoped to your plugin ID to avoid collisions

## Packaging

- [ ] Plugin directory name matches the manifest `id`
- [ ] Running `./package-plugin.sh <plugin-dir>` completes without errors
- [ ] The output `.zip` is a reasonable size (no build artifacts, `node_modules`, or junk files)
- [ ] The zip contains the plugin directory as the root folder (not loose files)

## Install Test

- [ ] Plugin installs successfully via Settings > Plugins > Import Plugin
- [ ] Plugin appears in the installed list with correct name, version, and icon
- [ ] Plugin can be enabled and disabled without errors
- [ ] Plugin can be uninstalled cleanly
- [ ] Reinstalling after uninstall works without issues

## Runtime (if plugin has `main.js`)

- [ ] No errors in the Hivemind console on plugin load
- [ ] Registered hooks fire correctly when their events occur
- [ ] Registered routes respond with valid status codes
- [ ] Plugin storage (`hivemind.storage`) read/write works
- [ ] `hivemind.emit()` events reach the UI iframe (if applicable)

## UI (if plugin has sidebar/panel HTML)

- [ ] Sidebar or panel renders correctly
- [ ] Bridge API calls (`window.hivemind.*`) work from the iframe
- [ ] UI handles loading and error states gracefully
