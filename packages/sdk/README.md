# budabit-sdk

SDK for building [Budabit](https://budabit.dev) Smart Widget extensions.

## Install

```bash
npm install budabit-sdk@^0.3.0
```

## Usage

### Bridge (iframe ↔ host communication)

```ts
import { createWidgetBridge, type WidgetBridge } from 'budabit-sdk';

const bridge = createWidgetBridge({
  targetWindow: window.parent,
  targetOrigin: '*',
});

// Publish a Nostr event
const result = await bridge.request('nostr:publish', {
  kind: 1,
  content: 'Hello from my widget!',
  tags: [],
  created_at: Math.floor(Date.now() / 1000),
});

// Receive initial host context.
bridge.onEvent('widget:init', (payload) => {
  console.log('Host version:', payload.hostVersion);
});

// Receive repository-scoped context changes.
bridge.onEvent('context:repoUpdate', (repo) => {
  console.log('Repository:', repo?.repoName);
});

// Call once after registering initial handlers.
bridge.signalReady();

// Subscription events use the host-assigned ID returned by subscribe().
const subscription = await bridge.subscribe({
  subscriptionId: 'client-label',
  relays: ['wss://relay.example.com'],
  filter: { kinds: [1], limit: 20 },
});
bridge.onEvent('nostr:subscription:event', ({ subscriptionId, event }) => {
  if (subscriptionId === subscription.subscriptionId) console.log(event);
});

// Cleanup also unsubscribes any still-open subscriptions.
bridge.destroy();
```

### Manifest CLI

Generate a Smart Widget event (kind 30033):

```bash
npx budabit-generate \
  --title 'My Widget' \
  --type tool \
  --app-url 'https://cdn.example.com/my-widget/index.html' \
  --icon 'https://cdn.example.com/my-widget/icon.png' \
  --image 'https://cdn.example.com/my-widget/preview.png' \
  --permissions 'nostr:publish,nostr:query,nostr:subscribe,ui:toast' \
  --nostr-kinds '30301,30302'
```

### Programmatic manifest generation

```ts
import { generateSmartWidgetEvent } from 'budabit-sdk/manifest';

const event = generateSmartWidgetEvent({
  title: 'My Widget',
  widgetType: 'tool',
  appUrl: 'https://cdn.example.com/my-widget/index.html',
  iconUrl: 'https://cdn.example.com/my-widget/icon.png',
  imageUrl: 'https://cdn.example.com/my-widget/preview.png',
  permissions: ['nostr:publish', 'ui:toast'],
  nostrKinds: [30301, 30302],
});
```

### Testing

```ts
import { createMockWidgetBridge } from 'budabit-sdk/testing';

const mock = createMockWidgetBridge();

// Simulate widget requesting a publish
const promise = mock.request('nostr:publish', { kind: 1, content: 'test', tags: [], created_at: 0 });

// Simulate host responding
const msg = mock.sentMessages[0];
mock.respondTo(msg.id!, { status: 'ok' });

const result = await promise; // { status: 'ok' }
```

### Worker bridge

```ts
import { createWorkerBridge } from 'budabit-sdk/worker';

const bridge = createWorkerBridge((msg) => self.postMessage(msg));
self.addEventListener('message', (e) => bridge.handleMessage(e.data));
```

### Host theme

Widgets are hosted inside an iframe, and the Budabit host communicates its UI
theme via `widget:init` (initial `theme`/`themeBackground` fields) and
`widget:themeChanged` events. `watchHostTheme` wires this up in one call: it
seeds a `prefers-color-scheme` fallback so there's no light-mode flash before
the host connects, then applies every subsequent theme change.

```ts
import { createWidgetBridge, watchHostTheme } from 'budabit-sdk';

const bridge = createWidgetBridge({ targetWindow: window.parent, targetOrigin: '*' });
const unwatch = watchHostTheme(bridge); // seeds + applies data-theme/color-scheme/--host-background

bridge.signalReady();

// later, on teardown
unwatch();
```

`watchHostTheme` sets `document.documentElement.dataset.theme` to `"light"` or
`"dark"` (style with `[data-theme='dark'] { ... }`), syncs `color-scheme` so
native form controls/scrollbars match, and exposes the host's effective
background as the `--host-background` CSS custom property. Use
`applyHostTheme`/`seedHostThemeFallback` directly if you need more control
than the bridge-wired helper provides.

## Typed host actions

Version 0.2.0 added typed requests for Nostr publish/query/sign/NIP-44 encryption/subscriptions, storage, repository context, toast/resize/navigation, plus string overloads for host-specific actions. Version 0.3.0 adds the host theme helpers (`applyHostTheme`, `seedHostThemeFallback`, `watchHostTheme`).

## Subpath Exports

| Import | Contents |
|--------|----------|
| `budabit-sdk` | Types, WidgetBridge, signaling helpers, host theme helpers |
| `budabit-sdk/manifest` | Event generator, CLI utilities |
| `budabit-sdk/testing` | MockWidgetBridge, test helpers |
| `budabit-sdk/worker` | Worker bridge |

## Scaffold a new widget

```bash
npx create-budabit-widget my-widget
```

## License

MIT
