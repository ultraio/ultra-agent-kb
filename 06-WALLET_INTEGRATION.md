# 06 — Wallet Integration (`@ultraos/wallet-sdk`)

**Last Updated:** 2026-09-03
**Read this to:** connect and transact through both the Ultra Wallet browser extension and
the hosted Web Wallet without treating their different capability models as interchangeable.
The public npm package plus this document are sufficient; a dapp never needs wallet source,
private keys, or raw `window.ultra` calls.

## Contents

1. Non-negotiable model
2. Install and choose a provider
3. Canonical dual-provider wrapper
4. Connect and derive state
5. Sign and broadcast
6. Extension lifecycle
7. Web Wallet lifecycle
8. Chain reads and local development
9. Testing and acceptance gate
10. Failure handling and traps

## 1. Non-negotiable model

- Use `@ultraos/wallet-sdk`, never raw `window.ultra`, for application calls.
- **Extension and Web Wallet share connect/sign, not lifecycle APIs.** Record which provider
  you selected and branch on it.
- The extension is injected as `window.ultra`; it owns its selected account/network and emits
  events. It supports custom/localhost networks.
- Web Wallet is a popup transport bound to one hosted environment when its SDK instance is
  constructed. It has no events, no live account/network query, and no network switching.
- Absence of `window.ultra` means “extension unavailable,” **not** “wallet unavailable.” The
  dapp must still offer Web Wallet on a deployed hosted environment.
- Reads bypass the wallet. Use `@wharfkit/antelope` with the endpoint associated with the
  resolved extension network or selected Web Wallet environment.

Published SDK version: **0.3.2**. New dapps must pin `^0.3.2` (0.3.1 and earlier break plain
Node/Vitest/SSR ESM imports). Verify `npm view @ultraos/wallet-sdk version` before adopting APIs
not documented here; the source monorepo can be ahead of npm.

### Capability matrix for published SDK 0.3.2 + current wallets

| Capability | Extension | Web Wallet |
| --- | --- | --- |
| `connect`, `disconnect` | Yes | Yes, popup |
| `signTransaction`, `signMessage` | Yes | Yes, popup |
| `getChainId` | Yes | Yes |
| Account identity | Connect result + live queries | **Connect result only** |
| `getAccounts`, `getSelectedAccount`, `getAvailableAuthorizations` | Yes | Do not call; current Web Wallet server does not expose them |
| `getNetwork`, `getNetworks` | Yes | Throws “Not supported in web provider” |
| `switchNetwork`, `addNetwork` | `switchNetwork` yes; `addNetwork` route is unavailable | Throws “Not supported in web provider” |
| `accountChanged`, `networkChanged`, `disconnect` events | Yes | No-op / unsupported |
| Localhost/custom networks | Yes | No |
| Silent `onlyIfTrusted` restore on page load | Yes | Do not use; it would open a popup |
| `purchaseItem` | Do not depend on it without a current product-specific validation | Do not use; current UI route is incomplete |

Current production availability (verified 2026-09-03): `https://web-wallet.ultra.io` serves
Mainnet. SDK 0.3.2 contains `https://web-wallet.staging.ultra.io` for `testnet`, but that hostname
is not currently deployed in public DNS. **Do not show Web Wallet for Testnet until that endpoint
is deployed and a connect smoke test passes.** This limitation does not affect Testnet through
the extension.

## 2. Install and choose a provider

```bash
npm install @ultraos/wallet-sdk@^0.3.2 @wharfkit/antelope
```

Two product designs are valid:

1. **Automatic fallback (Ultra Bridge pattern):** use extension when injected; otherwise use
   Web Wallet for the selected deployed environment.
2. **Explicit choice (Ultra Tool Kit pattern):** show separate “Ultra Wallet (Extension)” and
   “Ultra Wallet (Web)” buttons. Disable only the extension button when injection is absent;
   disable only Web Wallet on unsupported/undeployed environments.

Do not rely on SDK auto-detection without also retaining the selected provider kind. Your state
layer needs that kind to avoid calling extension-only methods on Web Wallet.

```ts
export type WalletKind = 'extension' | 'web';
export type WalletEnvironment = 'mainnet' | 'testnet';

export function extensionAvailable(): boolean {
  return typeof window !== 'undefined' && !!(window as any).ultra;
}

export function chooseWallet(): WalletKind {
  return extensionAvailable() ? 'extension' : 'web';
}
```

## 3. Canonical dual-provider wrapper

This is the minimum safe architecture. It follows the shipped Bridge/Tool Kit split: one
extension singleton, one Web Wallet instance per environment, and an explicit active kind.

```ts
import { UltraWalletSDK } from '@ultraos/wallet-sdk';
import type {
  BlockchainTransaction, ConnectParams, ConnectResult,
  SignTransactionResult, UltraResponse, WalletEventType,
} from '@ultraos/wallet-sdk';

type WalletKind = 'extension' | 'web';
type WalletEnvironment = 'mainnet' | 'testnet';
const deployedWebWalletEnvironments = new Set<WalletEnvironment>(['mainnet']);

let extensionSdk: UltraWalletSDK | null = null;
const webSdks = new Map<WalletEnvironment, UltraWalletSDK>();
let active: { kind: WalletKind; sdk: UltraWalletSDK; environment: WalletEnvironment } | null = null;

function extensionAvailable() {
  return typeof window !== 'undefined' && !!(window as any).ultra;
}

function sdkFor(kind: WalletKind, environment: WalletEnvironment): UltraWalletSDK {
  if (kind === 'extension') {
    if (!extensionAvailable()) throw new Error('Ultra Wallet extension is not installed');
    return extensionSdk ??= new UltraWalletSDK({ provider: 'extension' });
  }
  if (!deployedWebWalletEnvironments.has(environment)) {
    throw new Error(`Ultra Web Wallet is not deployed for ${environment}`);
  }
  let sdk = webSdks.get(environment);
  if (!sdk) {
    sdk = new UltraWalletSDK({ provider: 'web', environment });
    webSdks.set(environment, sdk);
  }
  return sdk;
}

export async function connect(
  environment: WalletEnvironment,
  params: ConnectParams = {},
  kind: WalletKind = extensionAvailable() ? 'extension' : 'web',
): Promise<UltraResponse<ConnectResult>> {
  const sdk = sdkFor(kind, environment);
  active = { kind, sdk, environment };
  return sdk.connect(params);
}

function current() {
  if (!active) throw new Error('connect a wallet first');
  return active;
}

function currentExtension() {
  const wallet = current();
  if (wallet.kind !== 'extension') throw new Error('method requires the Extension provider');
  return wallet.sdk;
}

export const providerKind = () => active?.kind ?? null;
export const getChainId = () => current().sdk.getChainId();
export const signTransaction = (actions: BlockchainTransaction[]): Promise<UltraResponse<SignTransactionResult>> =>
  current().sdk.signTransaction(actions);

// These fail locally before the SDK when the active provider is Web Wallet.
export const getSelectedAccount = () => currentExtension().getSelectedAccount();
export const getNetwork = () => currentExtension().getNetwork();
export const switchNetwork = (chainId: string) => currentExtension().switchNetwork(chainId);
export const on = (event: WalletEventType, cb: (data: any) => void) => {
  currentExtension().on(event, cb);
};
export const off = (event: WalletEventType, cb: (data: any) => void) => {
  currentExtension().off(event, cb);
};

export async function disconnect() {
  if (!active) return;
  try { await active.sdk.disconnect(); } finally { active = null; }
}

export function dispose() {
  extensionSdk?.dispose();
  for (const sdk of webSdks.values()) sdk.dispose();
  extensionSdk = null;
  webSdks.clear();
  active = null;
}
```

If the user changes Web Wallet environment, disconnect and reconnect using the instance for the
new environment. Never call `switchNetwork` on Web Wallet.

## 4. Connect and derive state

SDK calls have **two failure channels**. They can resolve with a non-success `UltraResponse`, or
reject/throw (popup blocked/closed, handshake timeout, concurrent request, environment mismatch,
transport failure). Always handle both:

```ts
try {
  const response = await wallet.connect(environment, {}, kind);
  if (response.status !== 'success' || !response.data) {
    throw new Error(response.message || 'wallet connection rejected');
  }
  const data = response.data;
  // derive identity/network below
} catch (error) {
  // show a user-safe popup/provider/network error; do not expose raw sensitive data
}
```

Derive identity from the connect response for **both** providers. Prefer the modern extension
shape, then fall back to the legacy fields returned by Web Wallet/current older wallets:

```ts
function identityFromConnect(data: ConnectResult) {
  if (data.selectedAccount?.accountName) {
    const permissions = data.selectedAccount.permissions ?? [];
    const permission = permissions.find((p) => p.name === 'active')?.name
      ?? permissions[0]?.name ?? 'active';
    return { account: data.selectedAccount.accountName, permission };
  }
  return { account: String(data.blockchainid || '').split('@')[0], permission: 'active' };
}
```

Then branch:

- **Extension:** refresh with `getSelectedAccount()` and `getNetwork()`; retain connect/last-known
  state if a transient query fails. The wallet-selected account/network is authoritative.
- **Web Wallet:** do not call those methods. Keep identity from `connect()`, obtain chain ID with
  `getChainId()`, and choose the read RPC from the environment used to construct the SDK.

Only attempt `connect({onlyIfTrusted:true})` automatically when the extension is present. Web
Wallet is user-gesture popup UI; never open it during page load or silent restore.

## 5. Sign and broadcast

The SDK action shape is `{contract, action, data, authorization}`, not raw Antelope RPC’s
`{account, name, authorization, data}`:

```ts
const actions = [{
  contract: 'eosio.token',
  action: 'transfer',
  authorization: [{ actor: account, permission }],
  data: {
    from: account,
    to: 'yourcontract',
    quantity: '1.00000000 UOS',
    memo: 'your routing memo',
  },
}];

try {
  const response = await wallet.signTransaction(actions);
  if (response.status !== 'success') throw new Error(response.message || 'transaction rejected');
  if (response.data.unsignedAuth?.length) throw new Error('transaction was not fully signed');
  const transactionId = response.data.transactionHash;
} catch (error) {
  // includes declined request, popup closed/blocked, timeout, and transport failures
}
```

`signTransaction` signs and broadcasts by default. Pass `{signOnly:true}` only when another
system will broadcast. Multi-action transactions are one array. Use exact asset precision.
`signMessage` accepts messages beginning with `0x`, `UOSx`, or `message:`.

Never put private keys, wallet passwords, bearer tokens, seed phrases, or funded test fixtures in
the dapp, KB, source control, screenshots, logs, or examples. Real-wallet test credentials must
come from an approved secret store and tests must skip cleanly when they are absent.

## 6. Extension lifecycle

Subscribe after a successful connect and unsubscribe/dispose on teardown:

- `accountChanged`: do not trust the payload as final state; re-query `getSelectedAccount()`.
  `selected:null` can mean no account on the new chain, not a user logout.
- `networkChanged`: re-query `getNetwork()`, rebuild the read client, and guard against a
  `switchNetwork`/event feedback loop.
- `disconnect`: clear local connection state; do not call `disconnect()` back from the handler.

There is no `chainChanged` event. SDK 0.3.2 manages extension listener registration, service-worker
recovery, and its heartbeat. Call `dispose()` to release those resources.

The production extension injects only on HTTPS pages. A downloaded extension will not inject on
plain `http://localhost`; use an HTTPS local server for real manual QA. A self-built unpacked
development artifact may retain loopback matches, but do not design production behavior around it.

## 7. Web Wallet lifecycle

- Construct with `{provider:'web', environment:'mainnet'}` (or a verified deployed environment).
- Keep one instance per environment; its popup origin is part of message validation.
- Treat every connect/sign/disconnect operation as popup-based and user initiated.
- Do not subscribe to events and do not call account/network/switch APIs from the capability matrix.
- On environment change, clear Web Wallet-derived state, select the corresponding read RPC, and
  require an explicit reconnect.
- Explain blocked-popup recovery in the UI. Do not retry automatically; browsers require a fresh
  user gesture.

The Web Wallet popup uses one in-flight request. Serialize wallet operations in UI state
(`busy=true` until completion) rather than firing concurrent requests.

## 8. Chain reads and local development

Reads use `APIClient` directly:

```ts
import { APIClient } from '@wharfkit/antelope';
let client = new APIClient({ url: activeNodeUrl });
```

- Extension: rebuild using the authoritative `getNetwork().data.nodeUrl`.
- Web Wallet: map the constructor environment to the corresponding public RPC.
- Local/custom: extension only. Resolve a local chain ID with `get_info`; never hardcode it.

For real extension QA, build/load the extension and serve the dapp over HTTPS. Follow `04` for a
local chain and approved disposable fixtures. This document intentionally contains no key or
credential material.

## 9. Testing and acceptance gate

Tests must prove provider branching, not merely transaction business logic:

1. **Unit:** extension present → extension SDK; absent → Web SDK bound to selected environment.
2. **Extension integration mock:** injected `window.ultra`; connect, live account/network queries,
   events, signing, disconnect, non-success envelopes and thrown errors.
3. **Web integration mock:** no `window.ultra`; popup `ready`/JSON-RPC exchange, legacy connect
   identity, `getChainId`, sign success/decline, popup blocked/closed, timeout, and serialization.
   Assert no extension-only method is called.
4. **Real Extension smoke:** persistent headed Chromium with the built MV3 extension; use disposable
   approved fixtures supplied outside source control.
5. **Real Web Wallet smoke:** deployed origin, popup connect and a non-destructive/manual signing
   check in the intended environment. Never automate production-value movement.

A dapp may claim **dual-wallet support** only if all of these are true:

- no extension still leaves a usable Web Wallet connect path;
- extension present selects it automatically or offers both explicit buttons;
- provider kind is retained in state;
- Web Wallet identity comes from `connect()` and network comes from its environment/`getChainId()`;
- extension-only APIs/events are guarded;
- Web Wallet environment changes force reconnect;
- both resolved failures and thrown/rejected errors are handled;
- every action has structured `authorization` and partial signatures are rejected;
- tests cover both provider branches, with no credentials in code or output.

The Tip Jar (`09`) implements this shape. Its local-chain E2E remains an extension-provider mock;
its provider-selection unit tests separately prove the Web Wallet branch. A real Web Wallet smoke
is still required before deploying a product that claims live Web Wallet support.

## 10. Failure handling and traps

1. `window.ultra` missing disables only the extension path, not Web Wallet.
2. Web Wallet on localhost/custom/testnet-without-a-deployment is unsupported; fail before popup.
3. Check response `status` **and** use `try/catch` around every SDK call.
4. User rejection is commonly code `4001`; popup handshake timeout `4300`; popup unavailable/
   blocked `4301`. Treat codes as UX signals, not secrets or diagnostics to dump wholesale.
5. `unsignedAuth` non-empty means partial signing; do not report success.
6. Do not call Web Wallet account/network/event methods just because they exist on the SDK class.
7. Do not silently open Web Wallet during app startup.
8. Serialize Web Wallet popup calls.
9. Rebuild the read client when the extension network changes or Web environment changes.
10. Call `dispose()` on teardown.
