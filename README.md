# sdl-sdk

> Typed TypeScript SDK for the Stellar DeFi Lens analytics platform.

<p>
  <a href="https://www.drips.network/wave/stellar"><img src="https://img.shields.io/badge/Stellar-Wave%20Program-blueviolet?style=flat-square" /></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-22c55e?style=flat-square" /></a>
  <img src="https://img.shields.io/badge/TypeScript-5.x-3178C6?style=flat-square" />
  <img src="https://img.shields.io/badge/npm-%40stellar--defi--lens%2Fsdk-CB3837?style=flat-square" />
</p>

Part of [Stellar DeFi Lens](https://github.com/stellar-defi-lens) — the open-source DeFi analytics layer for Stellar.

---

## Why this exists

Stellar DeFi data is trapped behind raw Soroban RPC calls, custom XDR decoding, and protocol-specific event formats. Any dApp developer who wants to show a user their Blend borrow rate or Aquarius pool TVL has to write significant infrastructure code just to get started.

`sdl-sdk` changes that. It provides a single, typed client that abstracts all of that complexity — so you can pull live Stellar DeFi analytics with one import and a single function call.

```typescript
import { LensClient } from "@stellar-defi-lens/sdk";

const lens = new LensClient();

// Get total Stellar DeFi TVL
const tvl = await lens.protocols.getTotalTVL();
console.log(tvl); // 147_920_000 (USD)

// Get Blend lending pool data
const pools = await lens.blend.getPools();
pools.forEach(pool => {
  console.log(`${pool.asset}: ${pool.borrowApy}% borrow APY`);
});

// Stream live events
lens.activity.stream({ protocols: ['blend', 'aquarius'] }, (event) => {
  console.log(`${event.protocol}: ${event.type} — $${event.amountUsd}`);
});
```

---

## Installation

```bash
npm install @stellar-defi-lens/sdk
# or
pnpm add @stellar-defi-lens/sdk
# or
yarn add @stellar-defi-lens/sdk
```

---

## Quick start

```typescript
import { LensClient } from "@stellar-defi-lens/sdk";

// Uses the public Stellar DeFi Lens API by default
const lens = new LensClient();

// Or point to a self-hosted sdl-api instance
const lens = new LensClient({
  apiUrl: "https://your-sdl-api.example.com",
  wsUrl: "wss://your-sdl-api.example.com",
});
```

---

## API reference

### `LensClient`

The main entry point. All protocol namespaces are properties of `LensClient`.

```typescript
const lens = new LensClient(options?: LensClientOptions);
```

**Options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `apiUrl` | `string` | Public SDL API | Base URL for `sdl-api` REST |
| `wsUrl` | `string` | Public SDL API | WebSocket URL for live events |
| `timeout` | `number` | `10000` | Request timeout in ms |

---

### `lens.protocols`

```typescript
// All protocols — TVL, volume, 24h change
const all = await lens.protocols.getAll(): Promise<ProtocolSummary[]>

// Single protocol
const blend = await lens.protocols.get("blend"): Promise<ProtocolDetail>

// Total DeFi TVL across all protocols
const tvl = await lens.protocols.getTotalTVL(): Promise<number>

// TVL timeseries
const history = await lens.protocols.getTVLHistory("blend", {
  from: new Date("2026-01-01"),
  to: new Date(),
  interval: "1d",          // "1h" | "1d" | "1w"
}): Promise<TVLDataPoint[]>

// Volume timeseries
const volume = await lens.protocols.getVolumeHistory("aquarius", {
  interval: "1d",
}): Promise<VolumeDataPoint[]>
```

**Types:**

```typescript
interface ProtocolSummary {
  id: string;
  name: string;
  type: "lending" | "dex" | "oracle" | "rwa";
  tvl: number;           // USD
  volume24h: number;     // USD
  change24h: number;     // percentage (e.g. 2.4 = +2.4%)
  change7d: number;
}

interface TVLDataPoint {
  timestamp: number;     // Unix ms
  value: number;         // USD
}
```

---

### `lens.blend`

```typescript
// All Blend lending pools with rates + utilization
const pools = await lens.blend.getPools(): Promise<BlendPool[]>

// Single pool by address
const pool = await lens.blend.getPool(poolAddress): Promise<BlendPool>

// Recent liquidation events
const liquidations = await lens.blend.getLiquidations({
  limit: 20,
}): Promise<LiquidationEvent[]>

// Borrow rate history for an asset
const rates = await lens.blend.getRateHistory("USDC", {
  interval: "1d",
}): Promise<RateDataPoint[]>
```

**Types:**

```typescript
interface BlendPool {
  poolAddress: string;
  assets: BlendPoolAsset[];
}

interface BlendPoolAsset {
  symbol: string;
  contractId: string;
  totalSupply: number;    // USD
  totalBorrow: number;    // USD
  supplyApy: number;      // percentage
  borrowApy: number;      // percentage
  utilization: number;    // 0–1
}

interface LiquidationEvent {
  user: string;
  asset: string;
  amountUsd: number;
  ledger: number;
  timestamp: number;
}
```

---

### `lens.aquarius`

```typescript
// All liquidity pairs sorted by TVL
const pairs = await lens.aquarius.getPairs({
  sortBy: "tvl",    // "tvl" | "volume24h" | "apy"
  limit: 50,
}): Promise<AquariusPair[]>

// Single pair
const pair = await lens.aquarius.getPair(pairId): Promise<AquariusPair>

// Volume history for a pair
const volume = await lens.aquarius.getPairVolume(pairId, {
  interval: "1d",
}): Promise<VolumeDataPoint[]>
```

**Types:**

```typescript
interface AquariusPair {
  pairId: string;
  token0: TokenInfo;
  token1: TokenInfo;
  tvl: number;
  volume24h: number;
  fees24h: number;
  apy: number;
}

interface TokenInfo {
  symbol: string;
  contractId: string;   // "native" for XLM
  priceUsd: number;
}
```

---

### `lens.soroswap`

```typescript
// All Soroswap pairs
const pairs = await lens.soroswap.getPairs({ limit: 50 }): Promise<SoroswapPair[]>

// Route distribution — what % of volume goes through each AMM
const routes = await lens.soroswap.getRouteDistribution(): Promise<RouteDistribution>
```

---

### `lens.assets`

```typescript
// Asset detail (works for any tracked token — RWA or standard)
const asset = await lens.assets.get(contractId): Promise<AssetDetail>

// Top holders
const holders = await lens.assets.getHolders(contractId, {
  limit: 20,
}): Promise<HolderRecord[]>

// Transfer history
const transfers = await lens.assets.getTransfers(contractId, {
  from: new Date("2026-01-01"),
  limit: 50,
}): Promise<TransferRecord[]>
```

**Types:**

```typescript
interface AssetDetail {
  contractId: string;
  name: string;
  symbol: string;
  decimals: number;
  totalSupply: number;
  holderCount: number;
  priceUsd: number;
  marketCap: number;
  volume24h: number;
  yieldApy?: number;     // Present for yield-bearing RWAs
}
```

---

### `lens.activity`

```typescript
// Recent significant events (REST — snapshot)
const events = await lens.activity.getRecent({
  protocol: "blend",      // optional filter
  type: "liquidation",    // optional filter
  limit: 50,
}): Promise<ActivityEvent[]>

// Live stream (WebSocket)
const unsubscribe = lens.activity.stream(
  { protocols: ["blend", "aquarius"] },
  (event: ActivityEvent) => {
    console.log(event);
  }
);

// Stop streaming
unsubscribe();
```

**Types:**

```typescript
interface ActivityEvent {
  protocol: string;
  type: "supply" | "withdraw" | "borrow" | "repay" | "swap" | "liquidation" | "bad_debt";
  asset: string;
  assetContractId: string;
  amountUsd: number;
  user: string;
  ledger: number;
  timestamp: number;
}
```

---

## React hooks

`sdl-sdk` ships optional React hooks for projects using React + TanStack Query.

```typescript
import {
  useTotalTVL,
  useProtocols,
  useProtocol,
  useBlendPools,
  useAquariusPairs,
  useAsset,
  useActivityFeed,
} from "@stellar-defi-lens/sdk/react";
```

All hooks accept an optional `LensClient` instance. If omitted, they use a default client pointed at the public API.

```tsx
import { useBlendPools } from "@stellar-defi-lens/sdk/react";

function BlendPoolsWidget() {
  const { data: pools, isLoading, error } = useBlendPools();

  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;

  return (
    <ul>
      {pools.map(pool =>
        pool.assets.map(asset => (
          <li key={asset.contractId}>
            {asset.symbol}: {asset.borrowApy.toFixed(2)}% borrow APY
          </li>
        ))
      )}
    </ul>
  );
}
```

**Live activity hook (WebSocket):**

```tsx
import { useActivityFeed } from "@stellar-defi-lens/sdk/react";

function LiveFeed() {
  const events = useActivityFeed({ protocols: ["blend", "aquarius"] });

  return (
    <ul>
      {events.map((event, i) => (
        <li key={i}>
          [{event.protocol}] {event.type} — ${event.amountUsd.toLocaleString()}
        </li>
      ))}
    </ul>
  );
}
```

---

## Project structure

```
sdl-sdk/
├── src/
│   ├── client/
│   │   └── LensClient.ts            # Main client class
│   ├── namespaces/
│   │   ├── protocols.namespace.ts   # lens.protocols.*
│   │   ├── blend.namespace.ts       # lens.blend.*
│   │   ├── aquarius.namespace.ts    # lens.aquarius.*
│   │   ├── soroswap.namespace.ts    # lens.soroswap.*
│   │   ├── assets.namespace.ts      # lens.assets.*
│   │   └── activity.namespace.ts    # lens.activity.*
│   ├── react/
│   │   ├── hooks/                   # useProtocols, useBlendPools, etc.
│   │   └── index.ts                 # React-specific exports
│   ├── types/
│   │   ├── models.ts                # All shared data model types
│   │   ├── options.ts               # LensClientOptions + query param types
│   │   └── index.ts                 # Re-exports
│   ├── http/
│   │   └── fetcher.ts               # Internal fetch wrapper (timeout, error handling)
│   ├── ws/
│   │   └── socket.ts                # WebSocket connection manager
│   └── index.ts                     # Public entry point
├── .env.example
├── package.json
├── tsconfig.json
├── tsup.config.ts                   # Build config (ESM + CJS + .d.ts)
└── CONTRIBUTING.md
```

---

## Getting started (development)

```bash
git clone https://github.com/stellar-defi-lens/sdl-sdk.git
cd sdl-sdk
pnpm install
pnpm dev        # Watch mode — rebuilds on change
pnpm build      # Full build: ESM + CJS + type declarations
pnpm test       # Vitest unit tests
pnpm test:watch # Watch mode
```

### Linking locally for development

To test the SDK in another local project before publishing:

```bash
# In sdl-sdk
pnpm build
pnpm link --global

# In your test project
pnpm link --global @stellar-defi-lens/sdk
```

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). SDK contributions have the highest ecosystem multiplier — every method you add is immediately usable by every dApp developer building on Stellar.

Issues labeled `wave:trivial`, `wave:medium`, `wave:high` are part of the Stellar Wave Program on Drips.

---

## License

MIT © Stellar DeFi Lens contributors
