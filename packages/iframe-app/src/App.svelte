<script lang="ts">
  import {
    WidgetBridge,
    createWidgetBridge,
    createTextNote,
    watchHostTheme,
    type UnsignedEvent,
    type WidgetInitPayload,
    type RepoContext,
  } from 'budabit-sdk';
  import { onMount } from 'svelte';

  // Bridge + host-provided context
  let bridge = $state<WidgetBridge | null>(null);
  let initPayload = $state<WidgetInitPayload | null>(null);
  let repoContext = $state<RepoContext | null>(null);

  // UI state
  let note = $state('');
  let status = $state('Initializing Smart Widget...');
  let lastPublishResult = $state<string | null>(null);
  let lastError = $state<string | null>(null);

  // Initialize bridge and set up handlers
  onMount(() => {
    const b = createWidgetBridge({
      targetWindow: window.parent,
      targetOrigin: '*',
      timeoutMs: 15000,
    });

    bridge = b;
    status = 'Ready. Waiting for host context...';

    // Match the host application's theme (light/dark + background)
    const offTheme = watchHostTheme(b);
    lastPublishResult = null;
    lastError = null;

    // Listen for widget:init (new lifecycle event)
    const offInit = b.onEvent('widget:init', (payload) => {
      initPayload = payload as WidgetInitPayload | null;
      const ver = initPayload?.hostVersion ?? 'unknown';
      status = `Connected (host v${ver})`;
    });

    // Listen for repo context updates (for repo-scoped extensions)
    const offRepoUpdate = b.onEvent('context:repoUpdate', (ctx) => {
      repoContext = ctx as RepoContext | null;
      if (repoContext) {
        status = `Connected — repo: ${repoContext.repoName}`;
      }
    });

    // Legacy fallback for hosts older than the current lifecycle API.
    // Uncomment this block (and its cleanup below) only when supporting those hosts.
    // `context:update` is deprecated and will be removed in v2.0.
    // const offContextUpdate = b.onEvent('context:update', (ctx) => {
    //   if (!repoContext && ctx && !('repoPubkey' in ctx)) {
    //     status = 'Connected (context received via deprecated event)';
    //   }
    // });

    // Signal to the host that we're ready
    b.signalReady();

    return () => {
      offTheme();
      offInit();
      offRepoUpdate();
      // offContextUpdate(); // Uncomment with the legacy fallback above.
      b.destroy();
      bridge = null;
    };
  });

  function buildNoteEvent(content: string): UnsignedEvent {
    const tags: string[][] = [];

    if (repoContext?.repoNaddr) {
      tags.push(['a', repoContext.repoNaddr]);
    }

    return createTextNote(content, tags);
  }

  async function publishNote(): Promise<void> {
    if (!bridge) return;

    const content = note.trim();
    if (!content) return;

    lastPublishResult = null;
    lastError = null;
    status = 'Publishing note via host (nostr:publish)...';

    const event = buildNoteEvent(content);

    try {
      const res = await bridge.request('nostr:publish', event);

      if (res && typeof res === 'object' && 'error' in res && typeof res.error === 'string') {
        lastError = res.error;
        status = `Publish failed: ${res.error}`;
        return;
      }

      lastPublishResult = 'ok';
      status = 'Published successfully';
      note = '';
    } catch (err) {
      const msg = err instanceof Error ? err.message : String(err);
      lastError = msg;
      status = `Publish failed: ${msg}`;
    }
  }

  async function showToast(): Promise<void> {
    if (!bridge) return;

    lastError = null;

    const message = repoContext
      ? `Hello from Smart Widget (repo: ${repoContext.repoName})`
      : 'Hello from Smart Widget';

    try {
      const res = await bridge.request('ui:toast', { message, type: 'info' });

      if (res && typeof res === 'object' && 'error' in res && typeof res.error === 'string') {
        lastError = res.error;
        status = `Toast failed: ${res.error}`;
        return;
      }

      status = 'Toast requested';
    } catch (err) {
      const msg = err instanceof Error ? err.message : String(err);
      lastError = msg;
      status = `Toast failed: ${msg}`;
    }
  }

  async function requestResize(): Promise<void> {
    if (!bridge) return;
    try {
      await bridge.request('ui:resize', { height: 400 });
    } catch {
      // Resize is best-effort
    }
  }
</script>

<div class="container">
  <header>
    <h1>Budabit Smart Widget Template (Tool)</h1>
    <p class="status" class:ready={!!initPayload}>{status}</p>
  </header>

  {#if initPayload}
    <section class="context">
      <h2>Host Context</h2>
      <dl>
        {#if initPayload.pubkey}
          <dt>User Pubkey:</dt>
          <dd class="pubkey">{String(initPayload.pubkey)}</dd>
        {/if}

        {#if initPayload.hostVersion}
          <dt>Host Version:</dt>
          <dd>{initPayload.hostVersion}</dd>
        {/if}

        {#if Array.isArray(initPayload.relays) && initPayload.relays.length > 0}
          <dt>Relays:</dt>
          <dd>{initPayload.relays.join(', ')}</dd>
        {/if}
      </dl>

      {#if repoContext}
        <h3>Repo Context</h3>
        <dl>
          <dt>Repo:</dt>
          <dd>{repoContext.repoName}</dd>
          <dt>Owner:</dt>
          <dd class="pubkey">{repoContext.repoPubkey?.slice(0, 16) ?? 'unknown'}...</dd>
          <dt>Relays:</dt>
          <dd>{repoContext.repoRelays.join(', ')}</dd>
        </dl>
      {/if}

      <details class="context-raw">
        <summary>Raw init payload</summary>
        <pre>{JSON.stringify(initPayload, null, 2)}</pre>
      </details>
    </section>
  {:else}
    <section class="waiting">
      <h2>Waiting for host context</h2>
      <p>
        This Smart Widget works without context, but receives it via
        <code>widget:init</code> and optionally <code>context:repoUpdate</code>.
      </p>
    </section>
  {/if}

  <section class="actions">
    <h2>Actions</h2>

    <div class="action-group">
      <h3>Publish a note (nostr:publish)</h3>
      <div class="input-group">
        <input
          type="text"
          bind:value={note}
          placeholder="Type a note to publish..."
          onkeydown={(e) => e.key === 'Enter' && publishNote()}
        />
        <button onclick={publishNote} disabled={!bridge || !note.trim()}>
          Publish
        </button>
      </div>

      {#if lastPublishResult}
        <p class="result">Last publish result: {lastPublishResult}</p>
      {/if}
    </div>

    <div class="action-group">
      <h3>UI Actions</h3>
      <div class="button-group">
        <button onclick={showToast} disabled={!bridge}>
          Show Toast (ui:toast)
        </button>
        <button onclick={requestResize} disabled={!bridge}>
          Resize to 400px (ui:resize)
        </button>
      </div>
    </div>

    {#if lastError}
      <div class="error">
        <strong>Error:</strong> {lastError}
      </div>
    {/if}
  </section>
</div>

<style>
  /* Theme tokens — the budabit-sdk theme helpers set `data-theme` on <html>
     from the host's widget:init / widget:themeChanged events. */
  :global(:root) {
    color-scheme: light;
    --ext-bg: #f5f5f5;
    --ext-surface: #ffffff;
    --ext-surface-2: #f8f9fa;
    --ext-text: #333333;
    --ext-text-muted: #666666;
    --ext-border: #dddddd;
    --ext-border-subtle: #eeeeee;
    --ext-shadow: rgba(0, 0, 0, 0.1);
    --ext-accent: #007bff;
    --ext-accent-hover: #0056b3;
    --ext-accent-text: #ffffff;
    --ext-disabled: #cccccc;
    --ext-warning-bg: #fff3cd;
    --ext-warning-border: #ffc107;
    --ext-warning-text: #856404;
    --ext-success-bg: #d4edda;
    --ext-success-border: #28a745;
    --ext-success-text: #155724;
    --ext-danger-bg: #f8d7da;
    --ext-danger-border: #dc3545;
    --ext-danger-text: #721c24;
  }

  :global([data-theme='dark']) {
    color-scheme: dark;
    --ext-bg: #151c23;
    --ext-surface: #1e2831;
    --ext-surface-2: #18212a;
    --ext-text: #e6ebf0;
    --ext-text-muted: #98a6b3;
    --ext-border: #374757;
    --ext-border-subtle: #2c3947;
    --ext-shadow: rgba(0, 0, 0, 0.5);
    --ext-accent: #3b96ff;
    --ext-accent-hover: #63abff;
    --ext-accent-text: #ffffff;
    --ext-disabled: #3d4a56;
    --ext-warning-bg: #3f3520;
    --ext-warning-border: #b58a1f;
    --ext-warning-text: #e8c869;
    --ext-success-bg: #14321f;
    --ext-success-border: #2f9e57;
    --ext-success-text: #7fd6a0;
    --ext-danger-bg: #3b1d21;
    --ext-danger-border: #e05260;
    --ext-danger-text: #f1a7ad;
  }

  :global(body) {
    margin: 0;
    padding: 0;
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell,
      sans-serif;
    background: var(--host-background, var(--ext-bg));
    color: var(--ext-text);
  }

  .container {
    max-width: 800px;
    margin: 0 auto;
    padding: 2rem;
  }

  header {
    text-align: center;
    margin-bottom: 2rem;
  }

  h1 {
    margin: 0 0 0.5rem 0;
    color: var(--ext-text);
  }

  .status {
    padding: 0.5rem 1rem;
    background: var(--ext-warning-bg);
    border: 1px solid var(--ext-warning-border);
    border-radius: 4px;
    color: var(--ext-warning-text);
    font-size: 0.9rem;
  }

  .status.ready {
    background: var(--ext-success-bg);
    border-color: var(--ext-success-border);
    color: var(--ext-success-text);
  }

  section {
    background: var(--ext-surface);
    border-radius: 8px;
    padding: 1.5rem;
    margin-bottom: 1.5rem;
    box-shadow: 0 2px 4px var(--ext-shadow);
  }

  h2 {
    margin: 0 0 1rem 0;
    color: var(--ext-text);
    font-size: 1.25rem;
  }

  h3 {
    margin: 0 0 0.75rem 0;
    color: var(--ext-text-muted);
    font-size: 1rem;
  }

  dl {
    display: grid;
    grid-template-columns: auto 1fr;
    gap: 0.5rem 1rem;
    margin: 0;
  }

  dt {
    font-weight: 600;
    color: var(--ext-text-muted);
  }

  dd {
    margin: 0;
    color: var(--ext-text);
  }

  .pubkey {
    font-family: monospace;
    font-size: 0.85rem;
    word-break: break-all;
  }

  .context-raw {
    margin-top: 1rem;
  }

  .context-raw pre {
    margin: 0.75rem 0 0 0;
    padding: 0.75rem;
    background: var(--ext-surface-2);
    border-radius: 6px;
    border: 1px solid var(--ext-border-subtle);
    overflow: auto;
    font-size: 0.85rem;
  }

  .action-group {
    margin-bottom: 1.5rem;
  }

  .action-group:last-child {
    margin-bottom: 0;
  }

  .input-group {
    display: flex;
    gap: 0.5rem;
  }

  input[type='text'] {
    flex: 1;
    padding: 0.5rem;
    border: 1px solid var(--ext-border);
    border-radius: 4px;
    font-size: 1rem;
    background: var(--ext-surface);
    color: var(--ext-text);
  }

  .button-group {
    display: flex;
    gap: 0.5rem;
    flex-wrap: wrap;
  }

  button {
    padding: 0.5rem 1rem;
    background: var(--ext-accent);
    color: var(--ext-accent-text);
    border: none;
    border-radius: 4px;
    font-size: 1rem;
    cursor: pointer;
    transition: background 0.2s;
  }

  button:hover:not(:disabled) {
    background: var(--ext-accent-hover);
  }

  button:disabled {
    background: var(--ext-disabled);
    cursor: not-allowed;
  }

  .result {
    margin: 0.75rem 0 0 0;
    color: var(--ext-text);
    font-size: 0.95rem;
  }

  .error {
    margin-top: 1rem;
    padding: 0.75rem;
    border: 1px solid var(--ext-danger-border);
    background: var(--ext-danger-bg);
    border-radius: 6px;
    color: var(--ext-danger-text);
  }

  .waiting {
    text-align: center;
  }

  .waiting p {
    margin: 0.5rem 0;
    color: var(--ext-text-muted);
  }
</style>
