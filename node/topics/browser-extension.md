# Topic: Browser Extension (WebExtension MV3)

For Chromium + Firefox extensions. Manifest V3 baseline. Cross-browser
via `webextension-polyfill`.

## 1. Stack baseline

| Layer | Choice |
|---|---|
| Manifest | V3 |
| Build | Vite + `@crxjs/vite-plugin` (Chromium) or `vite-plugin-web-extension` (cross-browser) |
| Language | TypeScript strict |
| UI (if any) | React or no framework — keep popup small |
| Cross-browser | `webextension-polyfill` |
| Testing | Vitest with `happy-dom` for unit, Playwright for E2E browser tests |

Sub-MB-scale extensions can skip Vite and use plain `tsc` + manual
file copying. Vite is worth it once you have a popup, options page,
content scripts, and shared modules.

## 2. Project layout

```
<project>/
├── src/
│   ├── manifest.ts          # generates manifest.json from TS — single source of truth
│   ├── background/
│   │   └── service-worker.ts
│   ├── content/
│   │   └── content-script.ts
│   ├── popup/
│   │   ├── popup.html
│   │   └── popup.tsx
│   ├── options/
│   │   ├── options.html
│   │   └── options.tsx
│   ├── shared/
│   │   ├── messages.ts      # message types (sender ↔ receiver)
│   │   └── storage.ts       # storage helpers
│   └── lib/                 # cross-cutting utilities
├── public/
│   └── icons/               # 16, 32, 48, 128 px
├── test/
│   └── e2e/
├── vite.config.ts
├── package.json
└── tsconfig.json
```

**`manifest.ts` over `manifest.json`** — generates the manifest at build
time from a TypeScript object. Lets you import constants (version,
permissions enum) instead of duplicating them.

## 3. Manifest V3 essentials

```ts
// src/manifest.ts
import type { Manifest } from 'webextension-polyfill';
import pkg from '../package.json' with { type: 'json' };

export const manifest: Manifest.WebExtensionManifest = {
  manifest_version: 3,
  name: 'Project Name',
  version: pkg.version,
  description: pkg.description,
  icons: {
    16: 'icons/icon-16.png',
    32: 'icons/icon-32.png',
    48: 'icons/icon-48.png',
    128: 'icons/icon-128.png',
  },
  background: {
    service_worker: 'src/background/service-worker.ts',
    type: 'module',
  },
  content_scripts: [
    {
      matches: ['<all_urls>'],
      js: ['src/content/content-script.ts'],
      run_at: 'document_idle',
    },
  ],
  action: {
    default_popup: 'src/popup/popup.html',
    default_icon: { 16: 'icons/icon-16.png', 32: 'icons/icon-32.png' },
  },
  options_ui: {
    page: 'src/options/options.html',
    open_in_tab: true,
  },
  permissions: [
    'storage',
    // Add specific permissions; avoid 'tabs' / 'activeTab' unless needed
  ],
  host_permissions: [],
};
```

Rules:
- **Request only the permissions you use.** Each permission you add
  triggers an extra warning at install — costs adoption.
- **Prefer `activeTab` over `<all_urls>`** when you only act on the
  user's current tab. Less invasive, no install-time scariness.
- **`run_at: document_idle`** unless you specifically need
  `document_start` or `document_end`. Idle is least-conflict-prone.

## 4. Service worker rules

MV3 service workers are short-lived. They are **not** persistent like
the old MV2 background pages.

- **Idle timeout: ~30 seconds.** State stored in module-level variables
  is lost.
- **Persist via `chrome.storage`** (sync or local) or `IndexedDB`.
- **Re-register listeners at module load**, every time. The service
  worker re-runs from scratch when it wakes.
- **`chrome.alarms` not `setTimeout`** for delayed work that should
  survive worker death.
- **Keep imports lean.** A heavy import on the first message increases
  cold-start latency.

```ts
// src/background/service-worker.ts
import browser from 'webextension-polyfill';

import { handleMessage, type Message } from '../shared/messages';

browser.runtime.onMessage.addListener(
  (message: unknown, sender) => handleMessage(message as Message, sender),
);

browser.runtime.onInstalled.addListener(({ reason }) => {
  if (reason === 'install') {
    // first-install setup
  }
});
```

## 5. Cross-browser via polyfill

`webextension-polyfill` makes `chrome.*` work as `browser.*` on Firefox
and as a Promise-returning version on Chromium.

```ts
import browser from 'webextension-polyfill';

await browser.storage.local.set({ key: 'value' });
const { key } = await browser.storage.local.get('key');
```

Don't mix `chrome.*` and `browser.*` in the same code path — pick one
(`browser` via polyfill is the standard).

## 6. Storage

| API | Use for | Limits |
|---|---|---|
| `browser.storage.local` | Per-extension, per-browser-profile data | ~5 MB (Chrome), ~10 MB (Firefox) |
| `browser.storage.sync` | Synced across user's devices | ~100 KB total, 8 KB per item |
| `browser.storage.session` | Lives until browser quit, not synced | ~10 MB |
| `IndexedDB` | Large structured data, queries | per-origin quota |
| `localStorage` | **Not in service worker.** Don't use. | — |

Wrap storage in a typed module:

```ts
// src/shared/storage.ts
import browser from 'webextension-polyfill';

interface Settings {
  enabled: boolean;
  apiUrl?: string;
}

const DEFAULTS: Settings = { enabled: true };

export async function getSettings(): Promise<Settings> {
  const stored = await browser.storage.local.get(DEFAULTS) as Settings;
  return stored;
}

export async function setSettings(updates: Partial<Settings>): Promise<void> {
  await browser.storage.local.set(updates);
}
```

## 7. File System Access

For "save to a file on the user's machine" features:

- **Chromium-only API**: `window.showSaveFilePicker()` etc.
- **Firefox fallback**: trigger a download via `browser.downloads.download({ url: dataUrl, ...})` — the user picks the location via the browser's download UI.

```ts
// Chromium path (in popup/options page, NOT in service worker)
async function saveToFile(content: string, suggestedName: string): Promise<void> {
  if ('showSaveFilePicker' in window) {
    const handle = await window.showSaveFilePicker({
      suggestedName,
      types: [{ description: 'Markdown', accept: { 'text/markdown': ['.md'] } }],
    });
    const writable = await handle.createWritable();
    await writable.write(content);
    await writable.close();
  } else {
    // Firefox fallback
    const blob = new Blob([content], { type: 'text/markdown' });
    const url = URL.createObjectURL(blob);
    await browser.downloads.download({ url, filename: suggestedName, saveAs: true });
    URL.revokeObjectURL(url);
  }
}
```

The service worker **cannot** call these — they require a user-gesture
context (popup, options page, or content script).

## 8. Messaging between contexts

Type your messages once, share between contexts:

```ts
// src/shared/messages.ts
export type Message =
  | { type: 'snippet:save'; payload: { text: string; url: string } }
  | { type: 'snippet:list' }
  | { type: 'snippet:export' };

export type Response<M extends Message['type']> =
  M extends 'snippet:list'
    ? { items: SavedSnippet[] }
  : M extends 'snippet:export'
    ? { ok: true }
  : { ok: true };

export async function send<M extends Message>(msg: M): Promise<Response<M['type']>> {
  return browser.runtime.sendMessage(msg) as Promise<Response<M['type']>>;
}
```

**Don't reach across contexts implicitly.** Service workers, content
scripts, popup, and options page are isolated. Communicate via
`runtime.sendMessage` / `tabs.sendMessage`.

## 9. Build & package

`vite.config.ts` (with `@crxjs/vite-plugin`):

```ts
import { defineConfig } from 'vite';
import { crx } from '@crxjs/vite-plugin';
import { manifest } from './src/manifest';

export default defineConfig({
  plugins: [crx({ manifest })],
  build: {
    outDir: 'dist',
    emptyOutDir: true,
    rollupOptions: {
      input: {
        popup: 'src/popup/popup.html',
        options: 'src/options/options.html',
      },
    },
  },
});
```

Build output goes in `dist/`. Load it as an unpacked extension in
Chrome (`chrome://extensions/` → "Load unpacked"). Firefox accepts the
same dir via `about:debugging`.

For distribution: `pnpm build && cd dist && zip -r ../<name>.zip .`
The zip is what you upload to the Chrome Web Store / addons.mozilla.org.

## 10. Permissions / privacy notes

- **Manifest V3 reviews are stricter.** Keep permissions minimal,
  prepare to justify each one in the store listing.
- **No remote-code execution.** Don't `eval` or load scripts from a
  remote URL. Even MV2 disallowed this; MV3 enforces.
- **Privacy policy URL is required** if your extension touches user
  data (which a snippet jar does). Even a one-line "all data is local
  on the user's machine, nothing is sent anywhere" page satisfies it.

## 11. Testing

- **Unit tests** with Vitest + `happy-dom`. Mock `browser.*` with a
  small fake.
- **E2E** with Playwright + `chromium` + `--load-extension=dist/`. Open
  pages, trigger the extension via UI or message, assert.
- **Don't try to test the service worker in isolation** — it's an event
  loop. Test the handlers it calls instead.

## 12. Gotchas — extension-specific

- **Service worker doesn't have `window`.** Don't import code that
  references it (or use a build-time replacement).
- **Content scripts share the page's DOM but not its JS context.** You
  can read DOM, but page-installed globals are invisible.
- **`<all_urls>`** triggers a "Read and change all your data on all
  websites" install warning — kills consumer trust. Scope tightly.
- **Updating the manifest version** doesn't auto-reload the extension
  in Chrome dev — toggle the extension off/on.
- **Promise-returning `browser.*` rejects with an Error**, but
  callback-style `chrome.*` puts the error on `chrome.runtime.lastError`
  silently. Stick to the polyfill / `browser.*` consistently.
