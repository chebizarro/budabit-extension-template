# budabit-sdk

SDK for building [Budabit](https://budabit.dev) Smart Widget extensions.

## Install

```bash
npm install budabit-sdk@^0.2.0
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

## Typed host actions

Version 0.2.0 includes typed requests for Nostr publish/query/sign/NIP-44 encryption/subscriptions, storage, repository context, toast/resize/navigation, plus string overloads for host-specific actions.

## Subpath Exports

| Import | Contents |
|--------|----------|
| `budabit-sdk` | Types, WidgetBridge, signaling helpers |
| `budabit-sdk/manifest` | Event generator, CLI utilities |
| `budabit-sdk/testing` | MockWidgetBridge, test helpers |
| `budabit-sdk/worker` | Worker bridge |

## Scaffold a new widget

```bash
npx create-budabit-widget my-widget
```

## License

MIT
