# Contributing to sdl-sdk

Welcome! `sdl-sdk` is part of the **[Stellar Wave Program](https://www.drips.network/wave/stellar)** on Drips. Contributors who resolve labeled issues during a Wave earn **XLM rewards** funded by the Stellar Development Foundation.

`sdl-sdk` has the highest ecosystem multiplier of any repo in the project — every method you add is immediately usable by every developer building on Stellar.

This guide is written against the real project structure. Read it before opening a PR.

---

## Table of contents

- [Project structure](#project-structure)
- [How the SDK is organized](#how-the-sdk-is-organized)
- [Local setup](#local-setup)
- [LensClient architecture](#lensclient-architecture)
- [How to add a new protocol namespace](#how-to-add-a-new-protocol-namespace)
- [How to add a method to an existing namespace](#how-to-add-a-method-to-an-existing-namespace)
- [Working with types](#working-with-types)
- [Working with the WebSocket module](#working-with-the-websocket-module)
- [React hooks (optional export)](#react-hooks-optional-export)
- [Examples](#examples)
- [Build and publish](#build-and-publish)
- [Testing](#testing)
- [Pull request process](#pull-request-process)
- [Code style](#code-style)

---

## Project structure

```
sdl-sdk/
├── .github/                     # CI workflows and issue templates
├── .sixth/                      # Internal tooling config (do not edit)
├── examples/                    # Runnable usage examples — one file per use case
│   └── node_modules/
├── src/
│   ├── assets/                  # Asset namespace — lens.assets.*
│   ├── client/
│   │   └── LensClient.ts        # Main client class — the public API entry point
│   ├── protocols/               # Protocol namespaces — lens.protocols.*, lens.blend.*, etc.
│   ├── types/                   # All shared TypeScript types and interfaces
│   ├── utils/                   # Internal utility functions (fetcher, retry, URL builder)
│   ├── websocket/               # WebSocket connection manager and streaming types
│   └── index.ts                 # Public entry point — all exports from this file
├── .gitignore
├── package.json
├── pnpm-lock.yaml
├── pnpm-workspace.yaml          # Workspace config (src/ + examples/)
├── README.md
├── tsconfig.json
└── tsup.config.ts               # Build config — outputs ESM + CJS + .d.ts declarations
```

---

## How the SDK is organized

`LensClient` is the single entry point. All protocol-specific methods are organized into **namespaces** — properties of the client that group related methods together.

```typescript
const lens = new LensClient();

// Namespaces
lens.protocols    // All-protocol queries (getAll, getTotalTVL, getTVLHistory)
lens.blend        // Blend-specific (getPools, getRateHistory, getLiquidations)
lens.aquarius     // Aquarius-specific (getPairs, getPairVolume)
lens.soroswap     // Soroswap-specific (getPairs, getRouteDistribution)
lens.assets       // Asset data (get, getHolders, getTransfers)
lens.activity     // Recent events (getRecent, stream)
```

Each namespace is a class instance attached to `LensClient` in the constructor. Namespaces receive the shared `fetcher` and `socketManager` instances from the client — they don't create their own HTTP or WebSocket connections.

---

## Local setup

### Requirements

- Node.js 20+
- pnpm

### Steps

```bash
# 1. Fork and clone
git clone https://github.com/<your-username>/sdl-sdk.git
cd sdl-sdk

# 2. Install dependencies (installs both src/ and examples/)
pnpm install

# 3. Build in watch mode
pnpm dev        # Rebuilds on every file change

# 4. Run tests
pnpm test
```

### Linking locally to test in another project

```bash
# In sdl-sdk
pnpm build
pnpm link --global

# In your test project
pnpm link --global @stellar-defi-lens/sdk

# Then import normally
import { LensClient } from '@stellar-defi-lens/sdk';
```

### Available scripts

```bash
pnpm dev           # Watch mode build (tsup --watch)
pnpm build         # Full build: ESM + CJS + .d.ts declarations
pnpm test          # Run all tests (Vitest)
pnpm test:watch    # Watch mode
pnpm test:cov      # Coverage report
pnpm lint          # ESLint
pnpm typecheck     # tsc --noEmit
```

---

## LensClient architecture

```typescript
// src/client/LensClient.ts
import { Fetcher } from '../utils/fetcher';
import { SocketManager } from '../websocket/socket';
import { ProtocolsNamespace } from '../protocols/protocols.namespace';
import { BlendNamespace } from '../protocols/blend.namespace';
import { AssetsNamespace } from '../assets/assets.namespace';
import { ActivityNamespace } from '../protocols/activity.namespace';
import type { LensClientOptions } from '../types';

export class LensClient {
  readonly protocols: ProtocolsNamespace;
  readonly blend: BlendNamespace;
  readonly aquarius: AquariusNamespace;
  readonly soroswap: SoroswapNamespace;
  readonly assets: AssetsNamespace;
  readonly activity: ActivityNamespace;

  constructor(options: LensClientOptions = {}) {
    const baseUrl  = options.apiUrl ?? 'https://api.stellar-defi-lens.xyz';
    const wsUrl    = options.wsUrl  ?? 'wss://api.stellar-defi-lens.xyz';
    const timeout  = options.timeout ?? 10_000;

    const fetcher       = new Fetcher({ baseUrl, timeout });
    const socketManager = new SocketManager({ wsUrl });

    this.protocols = new ProtocolsNamespace(fetcher);
    this.blend     = new BlendNamespace(fetcher);
    this.aquarius  = new AquariusNamespace(fetcher);
    this.soroswap  = new SoroswapNamespace(fetcher);
    this.assets    = new AssetsNamespace(fetcher);
    this.activity  = new ActivityNamespace(fetcher, socketManager);
  }
}
```

### The Fetcher

`src/utils/fetcher.ts` is the internal HTTP client. It handles:
- Base URL prepending
- Request timeout
- Error handling (non-2xx → typed `LensApiError`)
- JSON serialization

All namespace methods use the shared `fetcher` instance — they never call `fetch()` directly.

```typescript
// Inside a namespace method
async getPools(): Promise<BlendPool[]> {
  return this.fetcher.get<BlendPool[]>('/protocols/blend/pools');
}
```

---

## How to add a new protocol namespace

### Step 1 — Create the namespace file

```bash
touch src/protocols/<protocol>.namespace.ts
touch src/protocols/<protocol>.namespace.test.ts
```

### Step 2 — Implement the namespace class

```typescript
// src/protocols/phoenix.namespace.ts
import type { Fetcher } from '../utils/fetcher';
import type { PhoenixPair, PairsQueryParams } from '../types';

export class PhoenixNamespace {
  constructor(private readonly fetcher: Fetcher) {}

  /**
   * Returns all Phoenix AMM pairs sorted by TVL.
   */
  async getPairs(params?: PairsQueryParams): Promise<PhoenixPair[]> {
    const query = params ? new URLSearchParams(params as Record<string, string>) : '';
    return this.fetcher.get<PhoenixPair[]>(`/protocols/phoenix/pairs?${query}`);
  }

  /**
   * Returns volume timeseries for a specific pair.
   */
  async getPairVolume(pairId: string, params?: TimeRangeParams): Promise<VolumeDataPoint[]> {
    const query = params ? new URLSearchParams(params as Record<string, string>) : '';
    return this.fetcher.get<VolumeDataPoint[]>(`/protocols/phoenix/pairs/${pairId}/volume?${query}`);
  }
}
```

### Step 3 — Add the types

Add `PhoenixPair` and any other new interfaces to `src/types/models.ts`.

### Step 4 — Register the namespace in LensClient

Import the new class in `src/client/LensClient.ts`, add a `readonly` property, and instantiate it in the constructor.

### Step 5 — Export from the public entry point

Add the export to `src/index.ts`:

```typescript
export { PhoenixNamespace } from './protocols/phoenix.namespace';
export type { PhoenixPair } from './types';
```

### Step 6 — Write tests and an example

Add unit tests in `<protocol>.namespace.test.ts` and a runnable example in `examples/`.

---

## How to add a method to an existing namespace

1. Identify the namespace file in `src/protocols/` or `src/assets/`
2. Add the new async method — use `this.fetcher.get<T>()` for data, query param serialization handled by `URLSearchParams`
3. Add the TypeScript return type to `src/types/models.ts` if it doesn't exist
4. Add a JSDoc comment with a short description and parameter docs
5. Export the new type from `src/index.ts` if it's new
6. Add unit tests
7. Update the relevant example in `examples/` to demonstrate the new method

---

## Working with types

All shared TypeScript types live in `src/types/`. The structure mirrors the namespace organization:

```
src/types/
├── models.ts          # All data model interfaces (ProtocolSummary, BlendPool, etc.)
├── options.ts         # LensClientOptions and query parameter types
├── errors.ts          # LensApiError and error-related types
└── index.ts           # Re-exports everything — import from '@stellar-defi-lens/sdk' not from types/
```

### Conventions for types

- **Interfaces over types** for object shapes — interfaces are extensible
- **Discriminated unions** for events and multi-variant responses
- **`bigint` for on-chain amounts** — amounts from Soroban are bigints; convert to `number` only for display
- **ISO 8601 strings for timestamps** in API responses, `number` (Unix ms) in data points for chart compatibility

```typescript
// Models use interfaces
export interface BlendPool {
  poolAddress: string;
  assets: BlendPoolAsset[];
}

// Query params use type (they're simple objects)
export type TVLQueryParams = {
  from?: string;    // ISO 8601
  to?: string;      // ISO 8601
  interval?: '1h' | '1d' | '1w';
};

// Events use discriminated unions
export type ActivityEvent =
  | { protocol: string; type: 'supply';      asset: string; amountUsd: number; user: string; ledger: number; timestamp: number; }
  | { protocol: string; type: 'borrow';      asset: string; amountUsd: number; user: string; ledger: number; timestamp: number; }
  | { protocol: string; type: 'swap';        assetIn: string; assetOut: string; amountUsd: number; user: string; ledger: number; timestamp: number; }
  | { protocol: string; type: 'liquidation'; asset: string; amountUsd: number; user: string; ledger: number; timestamp: number; };
```

---

## Working with the WebSocket module

`src/websocket/` manages the Socket.io connection. The `SocketManager` class handles:
- Lazy connection (only connects when `stream()` is first called)
- Auto-reconnect with exponential backoff
- Typed event emission and subscription

The `ActivityNamespace.stream()` method is the only namespace method that uses the WebSocket:

```typescript
// src/protocols/activity.namespace.ts
stream(
  options: { protocols?: string[] },
  callback: (event: ActivityEvent) => void
): () => void {
  const socket = this.socketManager.connect();
  socket.emit('subscribe', options);
  socket.on('event', callback);

  // Returns an unsubscribe function
  return () => {
    socket.off('event', callback);
    this.socketManager.maybeDisconnect();
  };
}
```

The `unsubscribe` function pattern ensures consumers can clean up properly — critical for React `useEffect` cleanup.

---

## React hooks (optional export)

React hooks live in `src/protocols/` alongside the namespace they wrap. They are exported from the `@stellar-defi-lens/sdk/react` sub-path (configured in `package.json` exports map).

```typescript
// src/protocols/blend.hooks.ts
import { useQuery } from '@tanstack/react-query';
import { useLensClient } from './context';
import type { BlendPool } from '../types';

export function useBlendPools() {
  const lens = useLensClient();
  return useQuery<BlendPool[]>({
    queryKey: ['blend', 'pools'],
    queryFn: () => lens.blend.getPools(),
    staleTime: 30_000,
  });
}
```

**Rules for hooks:**
- Always use `useLensClient()` to get the client instance — never instantiate `LensClient` inside a hook
- `queryKey` arrays must be unique and descriptive: `['blend', 'pools']`, `['blend', 'rates', 'USDC', '1d']`
- Export from `src/react/index.ts`, which is the entry for the `/react` sub-path

---

## Examples

`examples/` contains runnable Node.js scripts demonstrating real use cases. Each new namespace method should have a corresponding example.

```bash
# Run an example
cd examples
pnpm install
node --loader ts-node/esm blend-pools.ts
```

Example structure:

```typescript
// examples/blend-pools.ts
import { LensClient } from '../src/index';

async function main() {
  const lens = new LensClient();

  const pools = await lens.blend.getPools();
  pools.forEach(pool => {
    pool.assets.forEach(asset => {
      console.log(`${asset.symbol}: ${asset.borrowApy}% borrow / ${asset.supplyApy}% supply`);
    });
  });
}

main().catch(console.error);
```

---

## Build and publish

The SDK builds to ESM and CJS with full `.d.ts` type declarations using `tsup`.

```bash
pnpm build
```

Output:
```
dist/
├── index.js          # CJS entry
├── index.mjs         # ESM entry
├── index.d.ts        # Type declarations
├── react/
│   ├── index.js
│   ├── index.mjs
│   └── index.d.ts
└── ...
```

The `package.json` exports map ensures the correct entry is used based on the consumer's environment (`require` vs `import`).

Publishing to npm is handled by the maintainers via CI on merge to `main`.

---

## Testing

```bash
pnpm test          # Run all tests (Vitest)
pnpm test:watch    # Watch mode
pnpm test:cov      # Coverage report
```

### Unit tests — mock the fetcher

Test namespace methods by mocking the `Fetcher` instance. Never make real HTTP calls in tests.

```typescript
// src/protocols/blend.namespace.test.ts
import { describe, it, expect, vi } from 'vitest';
import { BlendNamespace } from './blend.namespace';
import type { Fetcher } from '../utils/fetcher';

const mockFetcher = {
  get: vi.fn(),
} as unknown as Fetcher;

const namespace = new BlendNamespace(mockFetcher);

describe('BlendNamespace', () => {
  it('getPools returns typed pool array', async () => {
    const mockPools = [{ poolAddress: 'CBLEND...', assets: [] }];
    mockFetcher.get = vi.fn().mockResolvedValue(mockPools);

    const result = await namespace.getPools();

    expect(mockFetcher.get).toHaveBeenCalledWith('/protocols/blend/pools');
    expect(result).toEqual(mockPools);
  });

  it('getPools passes sort params to fetcher', async () => {
    mockFetcher.get = vi.fn().mockResolvedValue([]);
    await namespace.getPools({ sortBy: 'tvl', order: 'desc' });
    expect(mockFetcher.get).toHaveBeenCalledWith(
      expect.stringContaining('sortBy=tvl')
    );
  });
});
```

### Type tests

For critical type exports, add type-level tests using `expectTypeOf` from Vitest:

```typescript
import { expectTypeOf } from 'vitest';
import type { BlendPool, ActivityEvent } from '../types';

it('ActivityEvent is a discriminated union', () => {
  expectTypeOf<ActivityEvent>().toHaveProperty('type');
});
```

---

## Pull request process

1. **Branch from `main`**: `git checkout -b feat/phoenix-namespace`
2. **One PR per issue** — tight scope
3. **Checklist before opening:**
   - `pnpm test` passes
   - `pnpm typecheck` passes (no TypeScript errors, no `any`)
   - `pnpm build` produces valid output in `dist/`
   - All new public methods have JSDoc comments
   - All new types exported from `src/index.ts`
   - Unit tests cover the happy path and at least one error case
   - An example in `examples/` demonstrates the new functionality
4. **PR description must include:**
   - Link to the issue
   - The new method signatures and return types
   - A usage example showing the developer experience

---

## Code style

- **TypeScript strict mode** — no `any`, no type assertions without a comment
- **Namespace methods are pure async functions** — they call `this.fetcher.get()` and return typed data; no side effects
- **JSDoc on all public methods** — short description, `@param` for non-obvious params, `@returns` if not clear from type
- **No direct `fetch()` calls** — always go through `this.fetcher`
- **`bigint` for on-chain amounts** — never cast to `number` in the SDK itself; let consumers decide
- **Exports are deliberate** — only export what developers need; internal utilities stay internal
