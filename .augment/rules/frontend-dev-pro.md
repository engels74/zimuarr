---
type: "agent_requested"
description: "Bun + Svelte 5 + SvelteKit 2 + UnoCSS + shadcn-svelte coding guidelines"
---
# Idiomatic Bun + Svelte 5 + SvelteKit 2 + UnoCSS (presetWind4) + shadcn-svelte

This stack is compiler-first, signal-based, and single-binary. Svelte 5 compiles reactive intent (runes) to fine-grained DOM updates with no virtual DOM; SvelteKit 2 wraps it with file-based routing, load functions, form actions, and (2.27+) remote functions; Bun is the runtime, package manager, bundler, and test runner in one native binary; UnoCSS `presetWind4` is a Tailwind-v4-aligned on-demand atomic engine driven from `uno.config.ts`; and shadcn-svelte gives you copy-in, runes-native components built on Bits UI that you own and edit. Optimize for: explicit reactivity via runes, server/client separation via file naming, and letting the compiler do the work.

Agents write wrong-but-plausible code here by importing habits from adjacent ecosystems. The five biggest traps: (1) **Svelte 4 idioms** — `export let`, `$:`, stores, `on:click`, slots, `createEventDispatcher` — none are correct in a new Svelte 5 codebase. (2) **React habits** — reaching for `useEffect`-style `$effect` to compute derived values (use `$derived`), or `new Component()` (use `mount`). (3) **Node habits in Bun** — `express`, `dotenv`, `ts-node`, `jest`, `npm`/`pnpm` commands, `fs.readFile` where `Bun.file` is better. (4) **Tailwind-config habits** — creating `tailwind.config.js`, PostCSS setup, or `@tailwind` directives instead of `uno.config.ts` + `presetWind4`. (5) **`$app/stores`** instead of `$app/state`. Every section below shows the modern way once and well.

## Stack snapshot

- **Research date:** July 1, 2026
- **Research basis:** current official docs, release notes, specifications, changelogs, and primary repositories.

| Tool | Target version | Notes |
|---|---|---|
| Bun | 1.3.14 (shipped May 12, 2026) | runtime + PM + bundler + test runner |
| Svelte | 5.x (5.56+) | runes mode is default for new code |
| SvelteKit | 2.x (2.68+) | remote functions since 2.27 (experimental) |
| `@sveltejs/vite-plugin-svelte` | 7.x | requires Vite 8 |
| Vite | 8.x | |
| UnoCSS + `@unocss/preset-wind4` | 66.x | presetWind4 is Tailwind-v4-aligned |
| bits-ui | 2.x | headless primitives under shadcn-svelte |
| shadcn-svelte | latest (Svelte 5 + Tailwind v4 native) | copy-in components you own |
| sveltekit-superforms | 2.x | ecosystem-standard for complex forms |

Runes, `$app/state`, and `{@attach}` require **Svelte 5** and are the assumed floor. Where a feature has a higher floor it is annotated inline.

## Reactivity: the runes system

Runes are `$`-prefixed compiler symbols. They are **not imported** and only have meaning inside `.svelte`, `.svelte.js`, and `.svelte.ts` files. A file enters runes mode the moment it uses any rune. The four you write constantly are `$state`, `$derived`, `$effect`, `$props`; the rest are for specific cases.

### `$state` and deep reactivity

```svelte
<script lang="ts">
  let count = $state(0);
  let user = $state({ name: 'Ada', tags: ['math'] });

  function addTag() {
    user.tags.push('logic'); // mutation triggers updates — no reassignment needed
  }
</script>

<button onclick={() => count++}>clicks: {count}</button>
```

Objects and arrays passed to `$state` become **deeply reactive proxies** — mutating a nested property triggers fine-grained updates. This is the sharp break from Svelte 4, where you had to reassign (`user = user`) to force an update. The trade-off: proxying has overhead. For large objects you only ever *replace* (API responses, immutable data), use `$state.raw` — not proxied, only reassignment triggers updates:

```svelte
<script lang="ts">
  let rows = $state.raw<Row[]>([]);
  async function load() {
    rows = await fetch('/api/rows').then(r => r.json()); // replace, don't mutate
  }
</script>
```

`$state.snapshot(value)` returns a detached, plain (non-proxy) copy of deep state — pass it to non-Svelte libraries that choke on proxies, or log it (browser devtools show the proxy target, not the current value; prefer `$inspect` or `$state.snapshot` when logging).

Gotcha: `$state(...)` creates a proxy with a **different identity** than the value passed in, so `value === proxy` is always false.

### Reactive classes and `.svelte.ts` modules

Class fields can be `$state`, which is how you build reusable reactive logic. Runes work outside components in `.svelte.ts`/`.svelte.js` modules — this **replaces stores** for almost all shared-state cases:

```ts
// counter.svelte.ts
export function createCounter(initial = 0) {
  let count = $state(initial);
  const doubled = $derived(count * 2);
  return {
    get count() { return count; },
    get doubled() { return doubled; },
    increment() { count++; },
    reset() { count = initial; }
  };
}
```

```ts
// theme.svelte.ts — app-wide singleton reactive state
class Theme {
  accent = $state('violet');
  dense = $state(false);
}
export const theme = new Theme();
```

The `get` accessors matter: they keep `counter.count` referring to the *current* value, not a snapshot at call time. For SSR-safe per-request state, never use a module-level singleton (it leaks across requests on a long-lived server) — use SvelteKit's `event.locals` and load data through `+page.server.ts`.

`writable`/`readable`/`derived` from `svelte/store` still ship and work, but reach for them only for RxJS/observable interop, third-party libraries that export stores, or fine-grained subscription lifecycle control. Otherwise use runes.

### `$derived` and `$derived.by`

Use `$derived` for computed values — never `$effect` to keep one piece of state in sync with another. `$derived(expr)` takes an expression; `$derived.by(() => {...})` takes a function body for multi-step computation:

```svelte
<script lang="ts">
  let items = $state<Item[]>([]);
  let filter = $state('');

  let visible = $derived(items.filter(i => i.name.includes(filter)));
  let stats = $derived.by(() => {
    const total = visible.reduce((s, i) => s + i.price, 0);
    return { count: visible.length, total };
  });
</script>
```

Never mutate a `$derived` value, and never write `let color = type === 'danger' ? 'red' : 'green'` expecting reactivity — that computes once. Derived values recompute lazily and consistently.

### `$effect`, `$effect.pre`, `$effect.root`, `$effect.tracking`

`$effect` runs after the DOM updates, re-running when its read dependencies change, with a cleanup return. Use it as an escape hatch for side effects — network calls, third-party DOM libraries, canvas — not for deriving state.

```svelte
<script lang="ts">
  let color = $state('#f00');
  let size = $state(50);
  let canvas: HTMLCanvasElement;

  $effect(() => {
    const ctx = canvas.getContext('2d')!;
    ctx.fillStyle = color;
    ctx.fillRect(0, 0, size, size);
  });
</script>
<canvas bind:this={canvas} width={size} height={size}></canvas>
```

Never set a `$state` inside an `$effect` that reads the same state without a guard — that is an infinite loop and Svelte throws `effect_update_depth_exceeded`. `$effect.pre` runs *before* DOM updates (rare: measuring layout, syncing scroll). `$effect.root` creates a manually-controlled effect scope outside the component lifecycle (returns a cleanup). `$effect.tracking()` returns whether code is running inside a reactive context. Effects do not run during SSR.

### `$props`, `$bindable`, `$host`

Declare all props in one `$props()` call; put defaults in the destructure; type with a local `Props` interface:

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';

  interface Props {
    id: string;
    count?: number;
    items?: string[];
    onSelect: (item: string) => void; // callback prop replaces createEventDispatcher
    children?: Snippet;
  }
  let { id, count = 0, items = [], onSelect, children }: Props = $props();
</script>
```

`export let`, `$$props`, and `$$restProps` are all wrong in runes mode. For two-way binding, the child must mark the prop `$bindable()`:

```svelte
<!-- TextInput.svelte -->
<script lang="ts">
  let { value = $bindable('') }: { value?: string } = $props();
</script>
<input bind:value />
```

```svelte
<!-- parent -->
<script lang="ts">
  import TextInput from './TextInput.svelte';
  let name = $state('');
</script>
<TextInput bind:value={name} />
```

`$host()` returns the host element and is only valid inside custom-element components (`<svelte:options customElement="my-el" />`), typically to dispatch `CustomEvent`s.

### `$inspect` and `$inspect.trace`

`$inspect(a, b)` re-logs whenever tracked values change (deeply); `.with((type, ...values) => {})` swaps the handler (`type` is `'init'` or `'update'`). `$inspect.trace(label)` (Svelte 5.14+) must be the first statement in an `$effect`/`$derived.by` body and prints which reactive dependency triggered a re-run. Both are dev-only and stripped from production builds.

```svelte
<script lang="ts">
  let count = $state(0);
  $inspect(count).with((type) => { if (type === 'update') debugger; });

  $effect(() => {
    $inspect.trace('sync'); // first line — traces what fired this effect
    syncToServer(count);
  });
</script>
```

## Templates: events, snippets, attachments

### Event attributes, not directives

Svelte 5 uses plain attributes: `onclick`, `oninput`, `onsubmit` — **not** `on:click`. Handlers are just functions/props. There is no `createEventDispatcher`; components communicate upward via callback props.

```svelte
<button onclick={() => count++}>+</button>
<input oninput={(e) => (query = e.currentTarget.value)} />
<form onsubmit={(e) => { e.preventDefault(); save(); }}>…</form>
```

### Snippets replace slots

`{#snippet}` / `{@render}` replace slots entirely (slots are deprecated). Content between component tags that isn't a snippet declaration becomes the `children` snippet (the default-slot equivalent). Snippets are first-class values, take typed parameters, and can be passed as props.

```svelte
<!-- DataTable.svelte -->
<script lang="ts" generics="T">
  import type { Snippet } from 'svelte';
  let { data, header, row }: {
    data: T[];
    header: Snippet;
    row: Snippet<[T]>; // snippet taking one arg of type T
  } = $props();
</script>

<table>
  <thead><tr>{@render header()}</tr></thead>
  <tbody>
    {#each data as item}
      <tr>{@render row(item)}</tr>
    {/each}
  </tbody>
</table>
```

```svelte
<!-- usage -->
<script lang="ts">
  import DataTable from './DataTable.svelte';
  const fruits = [{ name: 'apple', qty: 5 }, { name: 'pear', qty: 3 }];
</script>

<DataTable data={fruits}>
  {#snippet header()}<th>Fruit</th><th>Qty</th>{/snippet}
  {#snippet row(f)}<td>{f.name}</td><td>{f.qty}</td>{/snippet}
</DataTable>
```

Render an optional snippet with optional chaining: `{@render children?.()}`. For fallback content, use `{#if children}{@render children()}{:else}…{/if}`. Snippets are always truthy, can recurse, and (Svelte 5.5+) can be exported from `<script module>` if they don't reference non-module declarations. Type `children` as `Snippet` imported from `svelte`.

### `{@attach}` replaces `use:` actions

Attachments (Svelte 5.29+) are the modern element-lifecycle primitive, superseding `use:` actions for most cases. An attachment is a function receiving the DOM node, optionally returning cleanup. Unlike actions, attachments are **fully reactive** — they re-run when any state read inside them changes — and they are spreadable through components as props.

```svelte
<script lang="ts">
  import tippy from 'tippy.js';
  import type { Attachment } from 'svelte/attachments';

  let content = $state('Hello!');

  function tooltip(text: string): Attachment {
    return (element) => {
      const instance = tippy(element, { content: text });
      return instance.destroy; // cleanup
    };
  }
</script>

<input bind:value={content} />
<button {@attach tooltip(content)}>Hover me</button>
```

To scope expensive setup, put the reactive part in a nested `$effect` inside the attachment so the outer setup runs once. For existing library actions, `fromAction(action, () => arg)` from `svelte/attachments` converts an action to an attachment. Falsy values (`false`/`undefined`) are treated as no attachment, enabling conditional use.

### Mounting components

Components are no longer classes. Use `mount`/`hydrate`/`unmount` from `svelte` — never `new Component()`:

```ts
import { mount, unmount } from 'svelte';
import App from './App.svelte';

const app = mount(App, { target: document.body, props: { title: 'Hi' } });
// later
unmount(app, { outro: true }); // play out transitions before removal (5.13+)
```

In a SvelteKit app you rarely call these directly — the framework mounts for you.

## SvelteKit 2: routing and files

Routing is file-based under `src/routes`. Each file kind has one job:

| File | Runs on | Purpose |
|---|---|---|
| `+page.svelte` | client + SSR | page UI; receives `data` and `form` props |
| `+page.ts` | server + client | universal `load` (runs both places) |
| `+page.server.ts` | server only | server `load` + form `actions`; DB/secrets safe |
| `+layout.svelte` | client + SSR | shared shell; renders `{@render children()}` |
| `+layout.ts` / `+layout.server.ts` | as above | layout-level load |
| `+server.ts` | server only | API endpoint (`GET`/`POST`/…) returning `Response` |
| `+error.svelte` | client + SSR | error boundary for the subtree |

Navigate with plain `<a href>` — there is no framework `<Link>`.

### Load functions and typing

Universal `load` (`+page.ts`) runs on server then client and can return non-serializable values on the client. Server `load` (`+page.server.ts`) runs only on the server, can touch databases and private env, and must return devalue-serializable data. Types come from the generated `./$types`.

```ts
// src/routes/blog/[slug]/+page.server.ts
import type { PageServerLoad } from './$types';
import { error } from '@sveltejs/kit';
import * as db from '$lib/server/database';

export const load: PageServerLoad = async ({ params }) => {
  const post = await db.getPost(params.slug);
  if (!post) error(404, 'Not found'); // just call it — no `throw` needed in SvelteKit 2
  return { post };
};
```

```svelte
<!-- +page.svelte -->
<script lang="ts">
  import type { PageProps } from './$types';
  let { data }: PageProps = $props(); // PageProps includes data + form + params (2.24+)
</script>
<h1>{data.post.title}</h1>
```

Decision: use `+page.ts` when the load is safe to run in the browser (public APIs, client-only libs) and you want the load to re-run client-side on navigation without a server round-trip; use `+page.server.ts` whenever you access secrets, a database, or private env. Leaking server-only imports into `+page.ts` is a top mistake — put them behind `$lib/server`.

### Streaming promises from load

In SvelteKit 2, returning a promise from `load` streams it — the page renders immediately and the data resolves in place. To *block*, `await` explicitly (use `Promise.all` to avoid waterfalls):

```ts
export const load: PageServerLoad = async () => {
  return {
    hero: await getHero(),          // awaited: blocks render
    comments: getComments()          // promise: streams in later
  };
};
```

```svelte
{#await data.comments}
  <p>Loading comments…</p>
{:then comments}
  {#each comments as c}<Comment {c} />{/each}
{/await}
```

### Invalidation

`depends('app:posts')` in a load registers a custom dependency; `invalidate('app:posts')` or `invalidate(url)` re-runs matching loads; `invalidateAll()` re-runs everything. `refreshAll()` (with remote functions) re-runs active remote functions plus current-page loads.

## Form actions and progressive enhancement

For server mutations tied to a `<form>`, use form actions in `+page.server.ts`. They work without JavaScript; `use:enhance` upgrades them to no-reload submissions.

```ts
// +page.server.ts
import type { Actions } from './$types';
import { fail } from '@sveltejs/kit';

export const actions: Actions = {
  create: async ({ request, locals }) => {
    const data = await request.formData();
    const title = String(data.get('title') ?? '');
    if (!title) return fail(400, { title, missing: true });
    await locals.db.posts.create({ title });
    return { success: true };
  }
} satisfies Actions;
```

```svelte
<!-- +page.svelte -->
<script lang="ts">
  import { enhance } from '$app/forms';
  import type { PageProps } from './$types';
  let { form }: PageProps = $props(); // ActionData is on `form`
  let saving = $state(false);
</script>

<form method="POST" action="?/create" use:enhance={() => {
  saving = true;
  return async ({ update }) => { await update(); saving = false; };
}}>
  <input name="title" aria-invalid={form?.missing ? 'true' : undefined} />
  <button disabled={saving}>{saving ? 'Saving…' : 'Create'}</button>
</form>
```

For complex/nested forms with schema validation, **superforms** (`sveltekit-superforms`, v2) is the ecosystem standard — it works with any Standard Schema validator (Zod, Valibot, etc.), merges `PageData`/`ActionData`, coerces `FormData`, and handles nested data and files. Define the schema at module top level so its adapter is cached:

```ts
// +page.server.ts
import { superValidate, message } from 'sveltekit-superforms';
import { valibot } from 'sveltekit-superforms/adapters';
import { fail } from '@sveltejs/kit';
import * as v from 'valibot';

const schema = v.object({
  email: v.pipe(v.string(), v.email()),
  name: v.pipe(v.string(), v.minLength(2))
});

export const load = async () => ({ form: await superValidate(valibot(schema)) });

export const actions = {
  default: async ({ request }) => {
    const form = await superValidate(request, valibot(schema));
    if (!form.valid) return fail(400, { form });
    return message(form, 'Saved');
  }
};
```

```svelte
<script lang="ts">
  import { superForm } from 'sveltekit-superforms';
  let { data } = $props();
  const { form, errors, enhance } = superForm(data.form);
</script>
<form method="POST" use:enhance>
  <input name="email" bind:value={$form.email} />
  {#if $errors.email}<span>{$errors.email}</span>{/if}
</form>
```

## App state, navigation, shallow routing

Use `$app/state` (SvelteKit 2.12+), a runes-based API — **not** `$app/stores` (deprecated, slated for removal in SvelteKit 3). Import `page`, `navigating`, `updated` and read them as plain objects (no `$` prefix). `page` is fine-grained: `page.state` updates don't invalidate `page.data`.

```svelte
<script lang="ts">
  import { page, navigating } from '$app/state';
  const id = $derived(page.params.id); // reactive
</script>

<nav>
  <a href="/" aria-current={page.url.pathname === '/' ? 'page' : undefined}>Home</a>
  {#if navigating.to}<span>Navigating to {navigating.to.url.pathname}…</span>{/if}
</nav>
```

Never write `$: x = page.params.id` — it will not update. On the server, `$app/state` values are readable only during rendering.

`$app/navigation` provides `goto`, `invalidate`, `invalidateAll`, `preloadData`, `pushState`, `replaceState`, `afterNavigate`, `beforeNavigate`. Shallow routing (`pushState`/`replaceState`) associates state with a history entry without a full navigation — ideal for modals:

```svelte
<script lang="ts">
  import { preloadData, pushState, goto } from '$app/navigation';
  import { page } from '$app/state';
  import Modal from './Modal.svelte';
  import PhotoPage from './[id]/+page.svelte';
  let { data } = $props();
</script>

{#each data.thumbnails as thumb}
  <a href="/photos/{thumb.id}" onclick={async (e) => {
    if (e.metaKey || e.ctrlKey || innerWidth < 640) return;
    e.preventDefault();
    const result = await preloadData(`/photos/${thumb.id}`);
    if (result.type === 'loaded' && result.status === 200) {
      pushState(`/photos/${thumb.id}`, { selected: result.data });
    } else { goto(`/photos/${thumb.id}`); }
  }}>
    <img src={thumb.src} alt={thumb.alt} />
  </a>
{/each}

{#if page.state.selected}
  <Modal onclose={() => history.back()}>
    <PhotoPage data={page.state.selected} />
  </Modal>
{/if}
```

Type page state via `App.PageState` in `src/app.d.ts`.

## Server-only modules, env, hooks

Anything under `$lib/server` is server-only and a build error if imported client-side. Environment access is split four ways:

| Import | Contents | When |
|---|---|---|
| `$env/static/private` | build-time private vars, inlined | secrets known at build |
| `$env/dynamic/private` | runtime private vars | secrets from runtime env |
| `$env/static/public` | build-time `PUBLIC_`-prefixed | client-safe, build-time |
| `$env/dynamic/public` | runtime `PUBLIC_`-prefixed | client-safe, runtime |

Only `PUBLIC_`-prefixed vars may reach the client. Private env is never importable from universal load or components.

Hooks live in `src/hooks.server.ts`, `src/hooks.client.ts`, and the shared `src/hooks.ts`.

```ts
// src/hooks.server.ts
import type { Handle, HandleFetch, HandleServerError } from '@sveltejs/kit';
import { sequence } from '@sveltejs/kit/hooks';

const auth: Handle = async ({ event, resolve }) => {
  const session = event.cookies.get('session');
  event.locals.user = session ? await getUser(session) : null;
  return resolve(event);
};

const securityHeaders: Handle = async ({ event, resolve }) => {
  const res = await resolve(event);
  res.headers.set('X-Frame-Options', 'DENY');
  return res;
};

export const handle = sequence(auth, securityHeaders);

export const handleFetch: HandleFetch = async ({ request, fetch }) => fetch(request);

export const handleError: HandleServerError = ({ error, event }) => {
  console.error(error); // log full error server-side
  return { message: 'Something went wrong' }; // safe message to client
};
```

Type `event.locals` in `src/app.d.ts`. The shared `src/hooks.ts` exports `reroute` (rewrite paths before routing) and `transport` (custom-type serialization across the server/client boundary).

### `transport`: custom types across the boundary (2.11+)

SvelteKit serializes load data with **devalue** (richer than JSON: Dates, Maps, Sets, BigInt, undefined, cyclical refs). For your own classes, register a transporter — each has `encode` (server; returns a falsy value for non-instances, else an array/object encoding) and `decode` (client; rebuilds the instance):

```ts
// src/hooks.ts
import type { Transport } from '@sveltejs/kit';
import { Vector } from '$lib/math';

export const transport: Transport = {
  Vector: {
    encode: (value) => value instanceof Vector && [value.x, value.y],
    decode: ([x, y]) => new Vector(x, y)
  }
};
```

The `encode` type guard is required because devalue runs it recursively on every value. Superforms integrates with this via a `transport` option to `superValidate`/`superForm`.

## Remote functions (2.27+, experimental)

Remote functions collapse the client/server API boundary into typed function calls. They live in `*.remote.ts` files and come in four kinds: `query` (reads), `form` (progressively-enhanced submissions), `command` (JS-driven mutations), `prerender` (build-time data). Per the official docs they are **available since 2.27 and currently experimental** (subject to change without notice) — enable `kit.experimental.remoteFunctions` and `compilerOptions.experimental.async` in `svelte.config.js`, and pin your SvelteKit version. The API is still tightening across minor releases: `query.batch` landed in `@sveltejs/kit@2.38.0`, tree-shaking "lazy discovery" of remote functions in `2.39.0`, and the enhanced form schema/`fields`/`issues` API in `2.42.0`; the `query`→`hydratable` change made `compilerOptions.experimental.async` mandatory for `.run()`. Prefer remote functions for internal app-to-itself traffic; keep `+server.ts` for public APIs.

```ts
// src/routes/blog/data.remote.ts
import { query } from '$app/server';
import * as v from 'valibot';
import * as db from '$lib/server/database';

export const getPost = query(v.string(), async (slug) => {
  const [post] = await db.sql`SELECT * FROM post WHERE slug = ${slug}`;
  return post;
});
```

```svelte
<script lang="ts">
  import { getPost } from './data.remote';
  let { slug } = $props();
  const post = getPost(slug);
</script>
{#if post.loading}…{:else}<h1>{post.current.title}</h1>{/if}
```

Arguments cross an HTTP boundary, so they must be validated by a Standard Schema validator. Inside a `command`/`form` you can call `getPost(slug).refresh()` on the server so refreshed data rides back in the same response (single-flight mutation) — no invalidate-everything.

## Adapters and page options

The adapter converts the build for a deployment target, set in `svelte.config.js`. `@sveltejs/adapter-auto` detects common platforms but supports none based on Bun.

| Adapter | Use when |
|---|---|
| `@sveltejs/adapter-node` | generic Node/long-lived server; most robust, actively maintained |
| `@sveltejs/adapter-static` | fully prerendered SSG |
| `@sveltejs/adapter-vercel` / `-cloudflare` / `-netlify` | those platforms |
| `svelte-adapter-bun` (community) | standalone Bun server; use for non-form-heavy apps |

Reality check on Bun: the dev server always runs under Vite regardless of adapter. `svelte-adapter-bun` produces a `Bun.serve`-based standalone server (`bun ./build/index.js`) with WebSocket support, but it is based on an older `adapter-node` and has known friction with SvelteKit's CSRF/origin protection for form actions. If forms misbehave in production, switching back is a one-line change to `@sveltejs/adapter-node`. Choose the Bun adapter for API/WebSocket-centric apps; default to `adapter-node` when in doubt.

Page options are exported from `+page`/`+layout` files:

```ts
export const prerender = true;   // render at build time
export const ssr = false;         // disable server render (SPA-style route)
export const csr = true;          // keep client hydration
```

Configure CSRF and CSP in `svelte.config.js` under `kit.csrf` and `kit.csp`.

## Bun as the toolchain

Bun is runtime, package manager, bundler, and test runner. Use Bun commands, not npm/pnpm/yarn.

### Package management

```bash
bun install                 # install; writes text-based bun.lock (default since 1.2)
bun add svelte @sveltejs/kit
bun add -d vite @sveltejs/vite-plugin-svelte  # dev dep
bun install --frozen-lockfile   # CI: fail if lockfile would change
bun update --interactive        # selective updates
bun why <pkg>                    # explain dependency chain
```

Commit `bun.lock` (text, diffs cleanly). For workspaces/monorepos, Bun 1.3 makes **isolated installs the default for workspaces** — per InfoQ's Bun 1.3 coverage, "Isolated installs are now the default for workspaces, preventing packages from accessing undeclared dependencies" — governed by a `configVersion` field in the lockfile. `bunfig.toml` centralizes config:

```toml
[install]
# supply-chain hardening: only install versions published ≥3 days ago
minimumReleaseAge = 259200
minimumReleaseAgeExcludes = ["@types/node", "typescript"]
```

Never mix package managers — conflicting lockfiles cause subtle version mismatches. Do not reach for `dotenv` (Bun auto-loads `.env`, `.env.local`, etc. and exposes `Bun.env`), `ts-node` (Bun runs TS directly, no build step), or `jest` (use `bun test`).

### Running scripts and the `--bun` flag

`bun run dev` runs `package.json` scripts. Framework CLIs like Vite still spin up Node by default; force Bun as the engine with `--bun`:

```bash
bun --bun run dev     # runs `vite dev` under the Bun engine
bun --bun run build
```

Bun only actually *becomes* the runtime when `Bun.serve()` starts or `--bun` is passed; otherwise Node runs in the background even in a Bun-installed project.

### Built-in APIs

Prefer Bun's native APIs over Node equivalents:

```ts
// File I/O — faster and simpler than node:fs
const text = await Bun.file('./data.json').text();
await Bun.write('./out.txt', 'hello');

// Password hashing (Argon2 by default)
const hash = await Bun.password.hash(password);
const ok = await Bun.password.verify(password, hash);

// Shell
import { $ } from 'bun';
await $`echo ${name} > greeting.txt`;

// Postgres (Bun.SQL) — no external driver
import { sql } from 'bun';
const users = await sql`SELECT * FROM users LIMIT 10`;

// SQLite
import { Database } from 'bun:sqlite';
const db = new Database('app.db');
```

`Bun.serve` with the `routes` object gives a fast native HTTP server (useful for standalone services alongside a SvelteKit app):

```ts
import { serve, sql } from 'bun';

serve({
  port: 3000,
  routes: {
    '/health': new Response('OK'),                    // zero-alloc static route
    '/api/users': {
      GET: async () => Response.json(await sql`SELECT * FROM users LIMIT 10`),
      POST: async (req) => {
        const { name } = await req.json();
        const [u] = await sql`INSERT INTO users ${sql({ name })} RETURNING *`;
        return Response.json(u, { status: 201 });
      }
    },
    '/api/users/:id': async (req) => {
      const [u] = await sql`SELECT * FROM users WHERE id = ${req.params.id}`;
      return u ? Response.json(u) : new Response('Not found', { status: 404 });
    }
  },
  fetch: () => new Response('Not found', { status: 404 })
});
```

Per Bun's official routing docs, static `Response` routes are cached for the lifetime of the server object and you can "generally expect at least a 15% performance improvement over manually returning a Response object."

### Testing with `bun:test`

`bun test` auto-discovers `*.test.ts`/`*.spec.ts`. The API is Jest-compatible: `describe`, `test`/`it`, `expect`, `mock`, `spyOn`, lifecycle hooks, snapshots. Bun's own site advertises it as "10-30x faster" than Jest.

```ts
import { describe, test, expect, spyOn } from 'bun:test';
import { createCounter } from '../src/counter.svelte';

describe('counter', () => {
  test('increments', () => {
    const c = createCounter(0);
    c.increment();
    expect(c.count).toBe(1);
    expect(c.doubled).toBe(2);
  });

  test('mock + spy', () => {
    const obj = { fetchIt: () => 1 };
    const spy = spyOn(obj, 'fetchIt');
    obj.fetchIt();
    expect(spy).toHaveBeenCalledTimes(1);
  });
});
```

For DOM/component tests, Bun supports **happy-dom** via `@happy-dom/global-registrator`. Register it in a preload file:

```ts
// happydom.ts
import { GlobalRegistrator } from '@happy-dom/global-registrator';
GlobalRegistrator.register();
```

```toml
# bunfig.toml
[test]
preload = ["./happydom.ts"]
```

Then use `@testing-library/svelte` to render. `bun test --parallel`, `--isolate`, `--shard`, and `--changed` (shipped in Bun 1.3.13, May 2026) tune execution. Note: Svelte component tests run through Vitest in much of the ecosystem; Bun's runner is excellent for logic in `.svelte.ts` modules and server code.

## UnoCSS with presetWind4

`presetWind4` (UnoCSS 66.x) is the Tailwind-CSS-v4-aligned preset — the current one, superseding `presetWind3`/`presetUno`/`presetWind`. There is **no `tailwind.config.js`, no PostCSS, no `@tailwind` directives**. Everything lives in `uno.config.ts`; theme tokens are emitted as CSS variables. presetWind4 uses the OKLCH color model, `@property`, `color-mix()`, and cascade layers — so it targets modern browsers only.

Key differences from presetWind3: the reset is **built in** (`presetWind4({ preflights: { reset: true } })` — no separate `@unocss/reset` needed); `presetRemToPx` is internal (use `createRemToPxResolver()` via `utilityResolver`); theme output goes to `theme`/`properties` cascade layers; it is *not* compatible with `presetLegacyCompat`, and `transformerDirectives` has known rough edges with it (use with caution).

### Vite/SvelteKit integration

Two integration modes. **Global mode** (`unocss/vite`) generates one global stylesheet — the default choice. **Scoped mode** (`@unocss/svelte-scoped/vite`) inlines each component's styles into its `<style>` block and is for library authors or very large apps fighting global-sheet growth (it does not support attributify/tagify presets and needs a hooks-based placeholder). Use global unless you have a specific reason.

```ts
// vite.config.ts
import { sveltekit } from '@sveltejs/kit/vite';
import UnoCSS from 'unocss/vite';
import { defineConfig } from 'vite';

export default defineConfig({
  plugins: [UnoCSS(), sveltekit()] // UnoCSS before sveltekit
});
```

```ts
// src/routes/+layout.svelte or app entry
import 'virtual:uno.css';
```

For `class:foo={bar}` directive extraction, add the Svelte extractor:

```ts
import extractorSvelte from '@unocss/extractor-svelte';
// ...
UnoCSS({ extractors: [extractorSvelte()] })
```

### `uno.config.ts` structure

```ts
import {
  defineConfig,
  presetIcons,
  presetTypography,
  transformerVariantGroup,
  transformerDirectives
} from 'unocss';
import presetWind4 from '@unocss/preset-wind4';

export default defineConfig({
  presets: [
    presetWind4({ preflights: { reset: true } }),
    presetIcons({ scale: 1.2, warn: true }),
    presetTypography()
  ],
  transformers: [
    transformerVariantGroup(),   // hover:(bg-blue text-white)
    transformerDirectives()      // @apply / --at-apply in CSS
  ],
  shortcuts: {
    'btn': 'px-4 py-2 rounded bg-blue-600 text-white hover:bg-blue-700',
    'card': 'rounded-lg border border-gray-200 p-4 shadow-sm'
  },
  rules: [
    ['text-balance', { 'text-wrap': 'balance' }]
  ],
  theme: {
    colors: { brand: { DEFAULT: '#7c3aed', muted: '#a78bfa' } }
  }
});
```

`presetIcons` uses on-demand icon sets (`i-lucide-house`, `i-mdi-account`); install the icon collection (`@iconify-json/lucide`). `transformerVariantGroup` enables `hover:(bg-x text-y)` grouping; `transformerDirectives` enables `@apply`/`--at-apply` inside CSS. Attributify mode (`presetAttributify`) lets you write `<button bg="blue-600" text="white">`. Do not create a `tailwind.config.js` for styling purposes and do not use `@tailwind base/components/utilities`.

## shadcn-svelte

shadcn-svelte is a runes-native community port of shadcn/ui: copy-in components built on **Bits UI** (unstyled, accessible primitives) that you own and edit — not an installed dependency. Adding a component copies its source into `$lib/components/ui`. The stock target is Tailwind CSS v4 + Svelte 5; components ship using `$props`, snippets, and `onclick`, each element carrying a `data-slot` attribute. It uses `tailwind-variants` for variants and a `cn()` util (`clsx` + `tailwind-merge`). Current dependency companions: `bits-ui`, `@lucide/svelte`, `tailwind-variants`, `tailwind-merge`, `clsx`, `svelte-sonner`, `mode-watcher`, `formsnap`. `cmdk-sv` and `tailwindcss-animate` are superseded (by Bits UI's `Command` and `tw-animate-css` respectively).

### The UnoCSS presetWind4 nuance — honest guidance

shadcn-svelte is designed around Tailwind. This stack uses UnoCSS presetWind4, which is Tailwind-v4-*compatible* but not identical. There are two viable paths; **path A (community preset) is the current best practice**:

**Path A — `unocss-preset-shadcn` (recommended).** The community package `unocss-preset-shadcn` (unocss-community, maintainer hyoban) supports presetWind4 by default since its v1.0. It generates shadcn's theme tokens, animations, and utilities under UnoCSS. Setup:

1. Install the reset, UnoCSS, and the presets: `bun add -d unocss @unocss/reset unocss-preset-animations unocss-preset-shadcn`, plus component deps `bun add lucide-svelte tailwind-variants clsx tailwind-merge`.
2. Do **not** run `shadcn-svelte init`. Instead create `components.json` and the `cn()` util manually, then add components with `bunx shadcn-svelte@latest add <component>`.
3. Create an **empty `tailwind.config.js`** in the project root solely to satisfy the shadcn CLI (it is not used for styling).
4. Configure `uno.config.ts`:

```ts
import { defineConfig } from 'unocss';
import presetWind4 from '@unocss/preset-wind4';
import presetAnimations from 'unocss-preset-animations';
import { presetShadcn } from 'unocss-preset-shadcn';

export default defineConfig({
  presets: [
    presetWind4(),
    presetAnimations(),
    presetShadcn({ color: 'zinc' })
  ],
  // presetWind4/shadcn tokens live in .ts/.js too — widen the scan pipeline
  content: {
    pipeline: {
      include: [
        /\.(svelte|[jt]sx?|html)($|\?)/,
        'src/**/*.{js,ts}'
      ]
    }
  }
});
```

5. Import the Tailwind reset and `uno.css` in your root layout, and add a small theme-sync so dark mode toggling updates the CSS variables.

The `cn()` util is the standard clsx + tailwind-merge combination (tailwind-merge v3 supports Tailwind v4 class syntax, which is what presetWind4 emits, so class-conflict resolution works correctly):

```ts
// src/lib/utils.ts
import { type ClassValue, clsx } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

**Path B — stock Tailwind v4.** If you don't need UnoCSS's engine features, run shadcn-svelte on its native Tailwind v4 setup (`@tailwindcss/vite`, `@import "tailwindcss"`, `@import "tw-animate-css"`, OKLCH variables under `@theme inline`). This is the smoothest shadcn path but means running Tailwind alongside/instead of UnoCSS — a choice against this stack's UnoCSS premise.

Honest friction points with Path A: the community preset's README example still literally shows `presetWind3` even though the preset defaults to presetWind4 (swap in `presetWind4()` as above); you must widen the content pipeline to scan `.ts`/`.js` for the generated theme classes; and dark-mode requires the theme-sync + `mode-watcher`, not a Tailwind config.

### Theming and dark mode

Theme tokens are CSS variables (`--background`, `--primary`, …) in `:root`/`.dark`. Dark mode uses `mode-watcher`, which must be mounted in the **root layout** so it runs before paint — putting theme logic in `onMount` causes a light/dark flash:

```svelte
<!-- src/routes/+layout.svelte -->
<script lang="ts">
  import '../app.css';
  import { ModeWatcher } from 'mode-watcher';
  let { children } = $props();
</script>

<ModeWatcher />
{@render children()}
```

```svelte
<!-- toggle -->
<script lang="ts">
  import { toggleMode } from 'mode-watcher';
  import { Button } from '$lib/components/ui/button';
</script>
<Button onclick={toggleMode}>Toggle theme</Button>
```

Because you own the component files, there is no `npm update` for them — when upstream fixes a bug, re-run `add` for that component and reconcile.

## Project layout & tooling config

```
my-app/
├─ src/
│  ├─ routes/            # file-based routing
│  ├─ lib/
│  │  ├─ server/         # server-only (build error if imported client-side)
│  │  └─ components/ui/  # shadcn-svelte copy-in components
│  ├─ hooks.server.ts
│  ├─ hooks.client.ts
│  ├─ hooks.ts           # reroute + transport
│  ├─ app.d.ts           # App.Locals / PageData / Error / Platform / PageState
│  ├─ app.css
│  └─ app.html
├─ static/
├─ uno.config.ts
├─ svelte.config.js
├─ vite.config.ts
├─ tsconfig.json
├─ bunfig.toml
├─ components.json
├─ bun.lock
└─ package.json
```

```ts
// src/app.d.ts
declare global {
  namespace App {
    interface Locals { user: { id: string; name: string } | null; }
    interface PageData {}
    interface PageState { selected?: unknown; }
    interface Error { message: string; code?: string; }
    interface Platform {}
  }
}
export {};
```

```js
// svelte.config.js
import adapter from '@sveltejs/adapter-node';
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';

/** @type {import('@sveltejs/kit').Config} */
export default {
  preprocess: vitePreprocess(),
  kit: { adapter: adapter() }
};
```

```ts
// vite.config.ts
import { sveltekit } from '@sveltejs/kit/vite';
import UnoCSS from 'unocss/vite';
import { defineConfig } from 'vite';

export default defineConfig({
  plugins: [UnoCSS(), sveltekit()]
});
```

```json
// tsconfig.json
{
  "extends": "./.svelte-kit/tsconfig.json",
  "compilerOptions": {
    "strict": true,
    "moduleResolution": "bundler",
    "verbatimModuleSyntax": true
  }
}
```

```json
// package.json (scripts)
{
  "scripts": {
    "dev": "bun --bun vite dev",
    "build": "bun --bun vite build",
    "preview": "bun --bun vite preview",
    "check": "svelte-kit sync && svelte-check --tsconfig ./tsconfig.json",
    "test": "bun test",
    "lint": "eslint . && prettier --check .",
    "format": "prettier --write ."
  }
}
```

`svelte-check` (not `tsc`) type-checks `.svelte` templates — a green `tsc --noEmit` alone is meaningless for Svelte files. Treat `svelte-check`, `eslint`, and `vite build` as the three gates before a change is done.

```js
// eslint.config.js — flat config
import js from '@eslint/js';
import ts from 'typescript-eslint';
import svelte from 'eslint-plugin-svelte';

export default ts.config(
  js.configs.recommended,
  ...ts.configs.recommended,
  ...svelte.configs['flat/recommended'],
  {
    files: ['**/*.svelte'],
    languageOptions: { parserOptions: { parser: ts.parser } }
  }
);
```

```json
// .prettierrc
{
  "plugins": ["prettier-plugin-svelte"],
  "overrides": [{ "files": "*.svelte", "options": { "parser": "svelte" } }]
}
```

## Anti-patterns to avoid

| Wrong (adjacent-ecosystem habit) | Right (this stack) |
|---|---|
| `export let prop;` | `let { prop } = $props();` |
| `$: doubled = count * 2;` | `let doubled = $derived(count * 2);` |
| `$: { sideEffect(); }` | `$effect(() => { sideEffect(); });` |
| `$effect` to compute a value | `$derived` / `$derived.by` |
| Mutating a `$derived` value | derive it; mutate the source `$state` |
| `import { writable } from 'svelte/store'` for app state | `$state` in a `.svelte.ts` module |
| `on:click={fn}` | `onclick={fn}` |
| `<slot />` / named slots | `{@render children()}` / snippet props |
| `createEventDispatcher()` | callback props (`onSelect`, `onClose`) |
| `use:action` for new code | `{@attach fn}` attachment |
| `new Component({ target })` | `mount(Component, { target })` |
| `import { page } from '$app/stores'`; `$page` | `import { page } from '$app/state'`; `page` |
| Reassigning to update `$state` arrays | mutate directly (`arr.push(x)`) — proxied |
| Secrets/DB in `+page.ts` | `+page.server.ts` or `$lib/server` |
| Module-level singleton for per-user state | `event.locals` + server load |
| `npm install` / `pnpm add` | `bun install` / `bun add` |
| `dotenv` | Bun auto-loads `.env`; `Bun.env` |
| `ts-node` / a TS build step | Bun runs TS directly |
| `jest` | `bun test` (`bun:test`) |
| `express` for a simple server | `Bun.serve({ routes })` |
| `fs.readFile` | `Bun.file(path).text()` |
| `tailwind.config.js` + `@tailwind` directives | `uno.config.ts` + `presetWind4` |
| `presetUno` / `presetWind` / `presetWind3` | `presetWind4` |
| Separate `@unocss/reset` with presetWind4 | built-in `preflights: { reset: true }` |
| shadcn `tailwindcss-animate` | `tw-animate-css` (or `unocss-preset-animations`) |
| Running Vite dev without `--bun` then expecting Bun runtime | `bun --bun vite dev` |
| Theme logic in `onMount` (dark-mode flash) | `<ModeWatcher />` in root layout |

## Quick reference

- Reactivity: `$state`, `$state.raw` (no proxy, replace-only), `$state.snapshot` (detached copy), `$derived`, `$derived.by`, `$effect`, `$effect.pre`, `$effect.root`, `$effect.tracking`, `$props`, `$bindable`, `$host`, `$inspect`, `$inspect.trace` (5.14+).
- Templates: `onclick` events; `{#snippet}`/`{@render}` for content (not slots); `{@attach}` (5.29+) for element lifecycle (not `use:`); `mount`/`unmount`/`hydrate` (not `new`).
- SvelteKit files: `+page.svelte`, `+page.ts` (universal load), `+page.server.ts` (server load + actions), `+layout.*`, `+server.ts`, `+error.svelte`; types from `./$types`.
- State/nav: `$app/state` (`page`, `navigating`, `updated`), `$app/navigation` (`goto`, `invalidate`, `preloadData`, `pushState`).
- Server boundary: `$lib/server`, `$env/{static,dynamic}/{private,public}`, `transport` hook, devalue serialization.
- Bun: `bun install`/`add`/`run`, `bun --bun` for Vite, `bun test`, `bunfig.toml`, `bun.lock`, `Bun.file`/`Bun.serve`/`Bun.password`/`Bun.$`/`Bun.sql`/`bun:sqlite`.
- Styling: `uno.config.ts`, `presetWind4` (built-in reset, OKLCH, modern-browser-only), `unocss/vite` global mode, `@unocss/svelte-scoped` for libraries.
- Components: shadcn-svelte on Bits UI, `cn()` = clsx + tailwind-merge, `mode-watcher` in root layout; under presetWind4 use `unocss-preset-shadcn` + `unocss-preset-animations` + empty `tailwind.config.js` for the CLI.

