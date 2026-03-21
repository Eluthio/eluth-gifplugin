# Eluth Plugin Developer Guide

This document explains how to build, submit, and distribute plugins for Eluth community servers.

---

## Plugin Tiers

Plugins are assigned a trust tier that determines what they can access and how they're distributed.

| Tier | Name | How distributed | What it can access |
|------|------|----------------|--------------------|
| 1 | **Official** | Bundled with community server | Full zone access, backend hooks, Central WSS |
| 2 | **Approved** | GitHub release zip, installed via server settings | Declared scopes, reviewed per release |
| 3 | **Unofficial** | Pasted URL, iframe sandbox only | Read-only display, no auth data |

---

## `plugin.json` — The Manifest

Every plugin must include a `plugin.json` at the root of its repository.

```json
{
  "name": "My Plugin",
  "slug": "my-plugin",
  "version": "1.0.0",
  "tier": "approved",
  "author": "Your Name",
  "description": "A short description shown in the Plugin Store.",
  "homepage": "https://github.com/yourname/my-plugin",
  "zones": ["input"],
  "data_access": [],
  "backend_hooks": [],
  "wss_scopes": [],
  "settings": [
    {
      "key": "api_key",
      "label": "API Key",
      "type": "text",
      "required": false,
      "placeholder": "Your API key"
    }
  ],
  "entry": "src/MyPlugin.vue",
  "eluth_key": "your-approved-key-goes-here"
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | ✓ | Display name (max 80 chars) |
| `slug` | ✓ | Unique identifier, lowercase letters and hyphens only (e.g. `gif-picker`) |
| `version` | ✓ | Semver (e.g. `1.2.3`) |
| `tier` | ✓ | `official`, `approved`, or `unofficial` |
| `author` | ✓ | Your name or organisation |
| `description` | ✓ | Shown in Plugin Store (max 300 chars) |
| `zones` | ✓ | Array of zones the plugin targets (see below) |
| `data_access` | ✓ | Data the plugin reads (declare all; shown to server admin at install) |
| `backend_hooks` | ✓ | Backend events to hook (Tier 2 only, reviewed per release) |
| `wss_scopes` | ✓ | WebSocket scopes needed (Tier 2 only) |
| `settings` | — | Configuration fields shown in the server admin plugin settings panel |
| `entry` | ✓ | Path to the main Vue component (relative to repo root) |
| `eluth_key` | Tier 2 | Issued by Eluth after code review — ties this key to this exact manifest |

---

## Zones

Zones are areas of the community server UI where your plugin can render content. Declare all zones you use in `plugin.json`.

| Zone | What it is | Tier 3 | Tier 2 | Tier 1 |
|------|-----------|--------|--------|--------|
| `chat` | Message list area | Read-only widget | React to messages, send via API | Full |
| `input` | Chat input bar | ✗ | Declared scope | Full |
| `sidebar` | Left channel sidebar | Read-only items | Click handlers, badge counts | Full |
| `member_panel` | Member list | Usernames only | User IDs, roles | Full |
| `settings` | Server settings | ✗ | Own settings page only | Full |

> **DMs are not a plugin zone.** DMs are a central server concern. Plugins cannot access DM data.

---

## Plugin Component API

Your plugin is a Vue 3 single-file component. It receives `settings` as a prop (the key/value pairs configured by the server admin) and can emit events.

### Input Zone

```vue
<script setup>
const props = defineProps({
    settings: { type: Object, default: () => ({}) }
})
const emit = defineEmits(['insert'])

// Call emit('insert', 'some text') to insert text into the chat input
</script>
```

### Chat Zone (read-only, Tier 3)

```vue
<script setup>
const props = defineProps({
    messages: { type: Array, default: () => [] }  // recent messages (read-only)
})
</script>
```

### postMessage Bridge (Tier 3 Unofficial)

Tier 3 plugins run in a sandboxed `<iframe>` and communicate via `postMessage`. The host exposes a controlled API surface — your plugin cannot access the parent DOM, localStorage, auth tokens, or cookies.

```js
// Inside your Tier 3 plugin iframe
window.parent.postMessage({ type: 'plugin:ready' }, '*')

window.addEventListener('message', (e) => {
    if (e.data.type === 'eluth:messages') {
        // Array of recent messages (username + content only, no IDs)
        console.log(e.data.messages)
    }
})
```

---

## Settings

Settings you declare in `plugin.json` are shown to server admins in the Plugin Settings panel. The values are passed to your component as the `settings` prop.

```json
"settings": [
    { "key": "api_key", "label": "API Key", "type": "text", "required": false }
]
```

Supported types: `text`, `password`, `number`, `toggle` (boolean), `select` (add `options: [{value, label}]`)

---

## Getting a Tier 2 Key

Tier 2 plugins require a key issued by Eluth that is cryptographically tied to your approved codebase.

1. **Create an account** at [eluth.io](https://eluth.io)
2. **Submit your plugin** via the Plugin Registry in your account dashboard:
   - Public GitHub repository URL
   - Your `plugin.json` (copied in)
   - A description of what your plugin does and why each declared scope is needed
3. **Eluth reviews your code** — we check the repo at the version you submitted
4. **On approval** you receive a `plugin_key`. Add it to `plugin.json` under `"eluth_key"`.
5. **Each new release** requires a new submission if you change declared scopes or `plugin.json`. Patch releases (bug fixes, no manifest changes) can be auto-approved via diff review.

> Keys are tied to a hash of your approved `plugin.json`. If you change the manifest without re-submitting, community servers will reject the plugin at install time.

---

## Distributing Your Plugin

### Tier 2 — GitHub Release

1. Tag your release: `git tag v1.2.3 && git push --tags`
2. Create a GitHub release from the tag
3. The release zip is what community servers download and install
4. Server admins install by pasting your GitHub release URL in Settings → Plugins → Install

### Tier 3 — Any URL

Tier 3 plugins can be installed by pasting any public URL pointing to a zip containing `plugin.json` and your entry file. Users will see a prominent warning that the plugin is unreviewed.

---

## Plugin Store

| Category | Listed? | Badge |
|----------|---------|-------|
| Official (Tier 1) | Auto-listed | ✅ Official |
| Approved (Tier 2) | Listed after review | 🔍 Reviewed |
| Unofficial (Tier 3) | Not listed | ⚠️ Unreviewed |

---

## Reference Implementation

The [GIF Picker](https://github.com/Eluthio/eluth-gifplugin) is an Official Tier 1 plugin and serves as the reference implementation for input zone plugins.

---

## Plugin API Reference

### Central Server API (Tier 2 only, declared `wss_scopes`)

Available endpoints for Tier 2 plugins with approved scopes — documented at [eluth.io/developers/api](https://eluth.io/developers/api).

---

## Submitting via API

You can also submit directly via the central API:

```
POST https://sol.eluth.io/api/plugins/apply
Authorization: Bearer {your_jwt}
Content-Type: application/json

{
  "plugin_name": "My Plugin",
  "slug": "my-plugin",
  "repo_url": "https://github.com/yourname/my-plugin",
  "version": "1.0.0",
  "description": "What it does.",
  "manifest": { ...your plugin.json content... }
}
```

Check your applications:
```
GET https://sol.eluth.io/api/plugins/my-applications
Authorization: Bearer {your_jwt}
```
