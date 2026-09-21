# Mock Bank System

A **simulated** two-bank web app that demonstrates how account-to-account
transfers work — both within one bank and across two different banks. It's
built to mirror the real Nigerian interbank flow (NIBSS/NIP) in structure,
without touching any real money, real bank, or real NIBSS connection.

**This is not connected to real money or a real bank.** It's a learning/demo
tool. See "Why this isn't a real transfer system" at the bottom.

## What's inside

- **No `npm install` needed.** Uses only Node's built-ins: `http` for the
  server and `node:sqlite` for the database (Node 22.5+).
- One codebase (`server.js`) that becomes "Bank A" or "Bank B" depending on
  the `BANK_CODE` env var you launch it with — run it twice to get two
  independent banks with their own database, own accounts, own ledger.
- A small web UI per bank: sign in as a demo customer, see balance and
  transaction history, send a transfer to any account at either bank.

## Run it

Open two terminals (or run both in background):

```bash
# Terminal 1 — Bank A
BANK_CODE=011 PORT=3001 node server.js

# Terminal 2 — Bank B
BANK_CODE=023 PORT=3002 node server.js
```

Then open:
- http://localhost:3001 — Alpha Microfinance Bank (Mock)
- http://localhost:3002 — Beta Microfinance Bank (Mock)

Demo accounts (seeded automatically, PIN `1234` for all):

| Bank | Account number | Name |
|---|---|---|
| 011 (Alpha) | 0111234567 | Chidinma Okafor |
| 011 (Alpha) | 0111234568 | Emeka Nwosu |
| 023 (Beta)  | 0231234567 | Aisha Bello |
| 023 (Beta)  | 0231234568 | Tunde Fashola |

Try: log into Alpha as Chidinma, transfer ₦5,000 to Aisha's account (023 /
0231234567), then open the Beta tab and refresh — the money has arrived.

## How a transfer actually moves (and how this mirrors it)

**Same bank** — `server.js` debits and credits both accounts in one atomic
SQLite transaction. No network call needed, same as real life.

**Different bank** — this is the part people usually don't see:

1. Your bank debits your account and marks the transaction `pending`.
2. Your bank looks up which bank owns the destination account number
   (`registry.js` — in real life, this lookup and the routing is NIBSS's
   job).
3. Your bank calls the destination bank's `/api/inward-transfer` endpoint
   directly (in real life, this hop goes through NIBSS's NIP switch).
4. The destination bank validates the account and credits it, using the
   `sessionId` as an idempotency key so a network retry can never credit
   twice.
5. If the destination bank rejects it or is unreachable, the source bank
   **reverses its own debit** — this is exactly what "failed transfer,
   please try again" means in a real app under the hood.

## Project structure

```
server.js        HTTP server + all transfer logic (local, outward, inward)
db.js            SQLite schema, atomic debit/credit, transaction log
registry.js      Bank code → base URL (stands in for NIBSS's routing table)
seed-config.js   Demo accounts seeded on first run
public/          Web UI (vanilla HTML/CSS/JS, no build step)
```

## Why this isn't a real transfer system

Real interbank transfers require:
- A CBN banking license (or partnership with a licensed institution)
- Formal NIBSS/NIP membership and certified integration
- A production core banking system, not a demo SQLite file
- KYC/AML compliance, PCI-DSS if cards are involved, and security audits

If your group's project is for a **licensed** bank, the real integration
work happens against that bank's core banking system and NIBSS credentials
— this project is the pattern to build the *app layer* against, once those
credentials exist. It's also a solid final-year-project artifact on its own:
it demonstrates atomic transactions, idempotency, ledger design, and
service-to-service API design, all real concepts examiners will recognize.

## Known simplifications (worth mentioning if you present this)

- Login uses a PIN over a query string for simplicity — a real app never
  does this; it'd use a hashed password/PIN and a signed session token.
- `/api/accounts` lists every account with no auth, purely so the login
  dropdown can populate itself. Remove this in anything beyond a demo.
- No HTTPS/TLS locally — deploy behind TLS (Render/any host does this for
  you) before this is reachable outside your machine.
