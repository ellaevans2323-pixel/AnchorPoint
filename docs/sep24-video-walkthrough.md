# SEP-24 Flow — Video Walkthrough Script

This document is the production script and shot list for the AnchorPoint SEP-24 interactive deposit/withdrawal walkthrough video. It covers the full end-to-end flow from wallet connection through transaction confirmation.

**Target length:** ~8 minutes  
**Audience:** Developers integrating AnchorPoint; anchor operators evaluating the dashboard.

---

## Pre-Recording Setup

Before recording, ensure the following are running:

```bash
# Terminal 1 — backend + demo server
npm run dev

# Terminal 2 — verify health
curl http://localhost:3002/health
# Expected: {"status":"ok","timestamp":"..."}
```

Browser: Chrome with Freighter extension installed and funded testnet account.  
Screen resolution: 1920×1080. Zoom browser to 125% for readability.

---

## Scene 1 — Introduction (0:00–0:45)

**[Screen: AnchorPoint dashboard landing page]**

> "Welcome to AnchorPoint — the open-source Stellar anchor dashboard. In this walkthrough, I'll show you the complete SEP-24 interactive flow: how a user connects their wallet, authenticates with SEP-10, initiates a deposit, completes KYC in the interactive webview, and monitors the transaction through to completion."

**[Highlight the sidebar navigation]**

> "The dashboard has four main sections: Overview, Deposit, Withdraw, and Transaction History. Let's start by connecting a wallet."

---

## Scene 2 — Wallet Connection & SEP-10 Auth (0:45–2:00)

**[Click the "Connect Wallet" button in the top-right]**

> "Clicking Connect Wallet opens the wallet picker. AnchorPoint supports Freighter, Albedo, Rabet, and hardware wallets like Ledger and Trezor."

**[Select Freighter]**

> "I'll select Freighter. The dashboard calls `POST /auth` on the backend, which generates a SEP-10 challenge transaction."

**[Freighter popup appears — show the signing prompt]**

> "Freighter asks me to sign the challenge. This is a standard Stellar transaction with a `manage_data` operation — it proves I control this account without sending any funds."

**[Click Approve in Freighter]**

> "Once signed, the dashboard sends the signed XDR to `POST /auth/token`. The backend verifies the signature and returns a JWT. From this point, all API calls are authenticated."

**[Dashboard shows connected state with public key truncated]**

> "The dashboard now shows my account address. The JWT is stored in memory — never in localStorage — for security."

---

## Scene 3 — Initiating a Deposit (2:00–3:30)

**[Click "Deposit" in the sidebar]**

> "Let's initiate a deposit. I'll select USDC as the asset and enter an amount."

**[Fill in the deposit form: asset_code = USDC, amount = 100]**

> "The dashboard calls `POST /sep24/transactions/deposit/interactive` with the asset code, my account address, and the amount."

**[Show the network request in DevTools briefly]**

```json
POST /sep24/transactions/deposit/interactive
{
  "asset_code": "USDC",
  "account": "GABC...XYZ",
  "amount": "100.00"
}
```

> "The anchor responds with a transaction ID and a URL."

```json
{
  "type": "interactive_customer_info_needed",
  "url": "http://localhost:3000/deposit?transaction_id=abc123&asset_code=USDC",
  "id": "abc123-..."
}
```

---

## Scene 4 — The Interactive Webview (3:30–5:00)

**[The webview opens inside the dashboard]**

> "The anchor's interactive URL opens in an embedded webview. This is where the anchor collects any additional information — KYC fields, bank details, or document uploads — using their own UI."

**[Fill in the KYC form: first name, last name, country]**

> "For this demo, the anchor asks for basic KYC: name and country. In production, this could include document uploads, liveness checks, or bank account details for fiat off-ramps."

**[Click Submit in the webview]**

> "After submitting, the anchor processes the information. The webview signals completion back to the parent dashboard using the `postMessage` API — a standard SEP-24 mechanism."

**[Webview closes, dashboard shows "Pending" status]**

> "The webview closes and the transaction moves to Pending status. The anchor is now processing the deposit on their end."

---

## Scene 5 — Transaction Monitoring (5:00–6:30)

**[Navigate to Transaction History]**

> "Let's check the Transaction History tab. The dashboard polls `GET /api/transactions` to show real-time status updates."

**[Show the transaction row with status "pending_external"]**

> "The transaction shows `pending_external` — the anchor is waiting for the fiat funds to arrive. Once confirmed, it will move to `pending_stellar`."

**[Simulate status change to "completed" — refresh or wait]**

> "When the anchor confirms receipt, the status updates to `completed` and the USDC appears in the user's Stellar account."

**[Show the transaction detail view]**

```
Transaction ID:  abc123-...
Asset:           USDC
Amount:          100.00
Status:          completed
Stellar TX:      https://stellar.expert/explorer/testnet/tx/...
Created:         2024-01-15 14:32:00 UTC
```

---

## Scene 6 — Withdrawal Flow (6:30–7:30)

**[Click "Withdraw" in the sidebar]**

> "The withdrawal flow is symmetric. I select USDC, enter an amount, and the dashboard calls `POST /sep24/transactions/withdraw/interactive`."

**[Fill in withdrawal form: asset_code = USDC, amount = 50]**

> "The anchor returns a URL for the withdrawal webview, where the user provides their bank account or destination details."

**[Open webview, fill in bank account field, submit]**

> "After the user submits their bank details, the anchor generates a Stellar payment address. The user sends USDC to that address to complete the withdrawal."

**[Show the pending withdrawal in Transaction History]**

> "The withdrawal appears in Transaction History with status `pending_user_transfer_start` — waiting for the user's Stellar payment."

---

## Scene 7 — Summary & Developer Notes (7:30–8:00)

**[Return to Overview screen]**

> "That's the complete SEP-24 flow. To summarize:"

**[Show bullet points on screen]**

1. **SEP-10** — Wallet signs a challenge → JWT issued
2. **POST /sep24/transactions/deposit/interactive** → returns `url` + `id`
3. **Interactive webview** — anchor collects KYC/payment details
4. **Transaction History** — poll `GET /api/transactions` for status updates
5. **Withdrawal** — same flow, user sends Stellar payment to complete

> "For developers: all endpoints are documented in `docs/postman-collection.json`. To add a new wallet, see `docs/wallet-adapter-guide.md`. The full source is on GitHub."

---

## API Reference (Quick Summary)

| Step | Method | Endpoint | Auth |
|------|--------|----------|------|
| Get SEP-10 challenge | POST | `/auth` | None |
| Exchange for JWT | POST | `/auth/token` | None |
| Start deposit | POST | `/sep24/transactions/deposit/interactive` | JWT |
| Start withdrawal | POST | `/sep24/transactions/withdraw/interactive` | JWT |
| Poll status | GET | `/api/transactions` | JWT |

---

## Recording Notes

- Pause 1 second after each click before narrating the result.
- Keep DevTools open on the Network tab during Scenes 2–4 to show real requests.
- Use `⌘+Shift+5` (macOS) or OBS to record. Export at 1080p/30fps.
- Add captions for accessibility before publishing.
- Recommended background music: none (technical tutorial).
