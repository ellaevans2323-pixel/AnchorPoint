# Adding a New Wallet Adapter to AnchorPoint

This guide explains how to integrate a new Stellar wallet into the AnchorPoint dashboard. AnchorPoint uses a simple adapter pattern: each wallet implements a common interface, so the SEP-10 authentication flow works identically regardless of which wallet the user has.

## Overview

The wallet adapter is responsible for two things:

1. **Signing** a SEP-10 challenge transaction XDR.
2. **Returning** the signed XDR so AnchorPoint can exchange it for a JWT.

Currently supported wallets: Freighter, Albedo, Rabet, Trezor, Ledger.

---

## 1. The Wallet Adapter Interface

Every adapter must implement this interface (TypeScript):

```typescript
// dashboard/src/wallets/types.ts

export interface WalletAdapter {
  /** Human-readable name shown in the UI */
  name: string;

  /** Unique identifier used internally */
  id: string;

  /** Returns true if the wallet extension/app is available */
  isAvailable(): boolean;

  /**
   * Request the user's Stellar public key.
   * Throws if the user denies access.
   */
  getPublicKey(): Promise<string>;

  /**
   * Sign a SEP-10 challenge transaction.
   * @param xdr  Base64-encoded transaction XDR from POST /auth
   * @param network  'testnet' | 'mainnet'
   * @returns Signed transaction XDR (base64)
   */
  signTransaction(xdr: string, network: 'testnet' | 'mainnet'): Promise<string>;
}
```

---

## 2. Create the Adapter File

Create a new file at `dashboard/src/wallets/<wallet-name>.adapter.ts`.

### Example: Freighter Adapter

```typescript
// dashboard/src/wallets/freighter.adapter.ts
import { WalletAdapter } from './types';

export const FreighterAdapter: WalletAdapter = {
  name: 'Freighter',
  id: 'freighter',

  isAvailable() {
    return typeof window !== 'undefined' && !!(window as any).freighter;
  },

  async getPublicKey() {
    const { publicKey } = await (window as any).freighter.getPublicKey();
    return publicKey;
  },

  async signTransaction(xdr, network) {
    const { signedTransaction } = await (window as any).freighter.signTransaction(xdr, {
      network: network === 'mainnet' ? 'PUBLIC' : 'TESTNET',
    });
    return signedTransaction;
  },
};
```

### Example: Hardware Wallet (Ledger)

Hardware wallets require the `@stellar/ledger-sdk` package and communicate over WebUSB/WebHID.

```typescript
// dashboard/src/wallets/ledger.adapter.ts
import { WalletAdapter } from './types';
import StellarApp from '@ledgerhq/hw-app-str';
import TransportWebUSB from '@ledgerhq/hw-transport-webusb';
import { Transaction, Networks } from '@stellar/stellar-sdk';

export const LedgerAdapter: WalletAdapter = {
  name: 'Ledger',
  id: 'ledger',

  isAvailable() {
    // WebUSB is available in Chrome/Edge; not in Firefox/Safari
    return typeof navigator !== 'undefined' && 'usb' in navigator;
  },

  async getPublicKey() {
    const transport = await TransportWebUSB.create();
    const app = new StellarApp(transport);
    const { publicKey } = await app.getPublicKey("44'/148'/0'");
    await transport.close();
    return publicKey;
  },

  async signTransaction(xdr, network) {
    const transport = await TransportWebUSB.create();
    const app = new StellarApp(transport);

    const networkPassphrase =
      network === 'mainnet' ? Networks.PUBLIC : Networks.TESTNET;

    const tx = new Transaction(xdr, networkPassphrase);
    const { signature } = await app.signTransaction(
      "44'/148'/0'",
      tx.signatureBase()
    );

    tx.addSignature(await this.getPublicKey(), signature.toString('base64'));
    await transport.close();
    return tx.toEnvelope().toXDR('base64');
  },
};
```

---

## 3. Register the Adapter

Add your adapter to the central registry:

```typescript
// dashboard/src/wallets/index.ts
import { FreighterAdapter } from './freighter.adapter';
import { AlbedoAdapter } from './albedo.adapter';
import { LedgerAdapter } from './ledger.adapter';
import { WalletAdapter } from './types';

export const WALLET_ADAPTERS: WalletAdapter[] = [
  FreighterAdapter,
  AlbedoAdapter,
  LedgerAdapter,
  // Add your new adapter here
];

export { WalletAdapter };
```

---

## 4. Use the Adapter in the SEP-10 Flow

The `App.tsx` SEP-10 flow calls the adapter like this:

```typescript
// Pseudocode — actual implementation in dashboard/src/App.tsx

async function authenticate(adapter: WalletAdapter) {
  // 1. Get the user's public key
  const publicKey = await adapter.getPublicKey();

  // 2. Request a SEP-10 challenge from the anchor
  const { transaction } = await fetch(`${apiBaseUrl}/auth`, {
    method: 'POST',
    body: JSON.stringify({ account: publicKey }),
  }).then(r => r.json());

  // 3. Sign the challenge with the wallet
  const signedXdr = await adapter.signTransaction(transaction, 'testnet');

  // 4. Exchange the signed transaction for a JWT
  const { token } = await fetch(`${apiBaseUrl}/auth/token`, {
    method: 'POST',
    body: JSON.stringify({ transaction: signedXdr }),
  }).then(r => r.json());

  return token;
}
```

---

## 5. Add a UI Entry

Add the wallet to the wallet selection UI in `App.tsx`:

```tsx
// In the wallet picker component
{WALLET_ADAPTERS.filter(w => w.isAvailable()).map(adapter => (
  <button
    key={adapter.id}
    onClick={() => authenticate(adapter)}
    className="wallet-button"
  >
    {adapter.name}
  </button>
))}
```

---

## 6. Testing Your Adapter

### Manual QA Steps

1. Start the dev server: `npm run dev`
2. Open `http://localhost:5173`
3. Click **Connect Wallet** and select your new wallet.
4. Confirm the wallet prompts for the public key.
5. Confirm the wallet prompts to sign the SEP-10 challenge.
6. Confirm a JWT is returned and the dashboard loads the authenticated state.

### Unit Test

```typescript
// dashboard/src/wallets/__tests__/my-wallet.adapter.test.ts
import { MyWalletAdapter } from '../my-wallet.adapter';

describe('MyWalletAdapter', () => {
  it('reports availability correctly', () => {
    // Mock window.myWallet
    (window as any).myWallet = {};
    expect(MyWalletAdapter.isAvailable()).toBe(true);
    delete (window as any).myWallet;
    expect(MyWalletAdapter.isAvailable()).toBe(false);
  });

  it('returns a public key', async () => {
    (window as any).myWallet = {
      getPublicKey: async () => ({ publicKey: 'GABC...' }),
    };
    const key = await MyWalletAdapter.getPublicKey();
    expect(key).toBe('GABC...');
  });
});
```

---

## 7. Checklist

- [ ] Adapter file created at `dashboard/src/wallets/<name>.adapter.ts`
- [ ] Implements all three methods: `isAvailable`, `getPublicKey`, `signTransaction`
- [ ] Registered in `dashboard/src/wallets/index.ts`
- [ ] Wallet appears in the UI when available
- [ ] Manual QA steps pass (connect → sign → JWT received)
- [ ] Unit tests added and passing (`npm test` in `dashboard/`)

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| Wallet not shown in picker | `isAvailable()` returns false | Check extension is installed / browser supports required API |
| "Invalid signature" from `/auth/token` | Wrong network passphrase | Ensure `network` param matches the anchor's configured network |
| Hardware wallet timeout | USB transport not opened | Wrap transport in try/finally and always call `transport.close()` |
| CORS error on sign | Extension blocked by CSP | Add wallet origin to `Content-Security-Policy` in `index.html` |
