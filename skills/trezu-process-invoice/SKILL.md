---
name: trezu-process-invoice
description: Turn incoming invoices into USDT payment proposals on a Trezu (Sputnik DAO) treasury on NEAR mainnet via near-cli-rs. One-time guided setup creates a proposals-only implicit account, funds it, and registers it as a Requestor on trezu.app; afterwards every new invoice is extracted, validated, and submitted as a Transfer proposal.
---

# Trezu Invoice → USDT Payment Proposals

You help the user run an invoice-payment workflow on a Trezu treasury (a Sputnik DAO
on NEAR **mainnet** — mainnet only, always pass `network-config mainnet`). The workflow
has two parts:

1. **One-time setup** — create a dedicated implicit account, fund it, and register it
   as a **Requestor** (proposals-only) member of the treasury on https://trezu.app.
2. **Recurring processing** — whenever a new invoice arrives, extract the payment
   details and submit a USDT `Transfer` proposal to the treasury via `near-cli-rs`.

## Standing rule

**Whenever a new invoice arrives — as an attachment or pasted text, on any channel —
process it into a payment proposal (Part 2) without being asked.** This rule is also
written to memory during setup so it survives across sessions.

## On every activation

Check memory for `projects/treasury/.trezu-setup-complete`:

- **Missing** → run **Part 1: Setup** below.
- **Present** → read the treasury config from `projects/treasury/AGENTS.md` and run
  **Part 2: Invoice processing**.

## Tooling rules

- **The only CLI in this workflow is `near-cli-rs`** (the `near` command). There is
  no Trezu CLI — do not search for, install, or attempt to invoke one. Trezu is a web
  app: the user interacts with it exclusively through the `https://trezu.app/...`
  pages referenced below; everything on-chain goes through `near-cli-rs`.
- **There are no credentials for `{TREASURY_ID}`, and none are ever needed.** The
  treasury is a smart contract (a Sputnik DAO) governed collectively by its members —
  nobody holds its keys. Never try to log in as it, import keys for it, or sign
  anything with it. The only account this workflow ever signs with is the implicit
  account `{IMPLICIT_HEX}` (plus the user's own `{SIGNER}` account during the two
  setup funding steps). The treasury account appears in commands only as a call
  target (`add_proposal`, `get_policy`) or a token receiver.
- `near-cli-rs` commands must be run as **single plain invocations** — no command
  chaining (`;`, `&&`), no subshells (`$(...)`), no pipes. Run one command, read its
  output, and substitute literal values into the next command yourself.

## Steps are strict

The steps in this skill are **exact procedures, not suggestions**. Follow them in
order, using the exact commands and URLs given — only substituting the `{PLACEHOLDER}`
values. Do not skip, reorder, merge, or improvise steps, and do not substitute
alternative tools or flows. If a step fails, report the error and resolve it with the
user before continuing; do not invent a workaround.

---

# Part 1: Setup (one-time, guided)

Walk the user through these steps in order. Confirm each step succeeded before moving on.

## Step 1 — Create the treasury on trezu.app

Ask the user to:

1. Sign up at https://trezu.app and create a treasury.
2. Provide the treasury's account ID — call it `{TREASURY_ID}`
   (e.g. `test-123.sputnik-dao.near`).

## Step 2 — Find the credentials directory

```
near config show-connections
```

Read `credentials_home_dir` from the output — call it `{CREDENTIALS_DIR}`
(e.g. `/Users/den/.near-credentials`).

**Never scan, list, or read the contents of `{CREDENTIALS_DIR}`** — it holds private
keys for the user's other accounts. Its value is used only to construct the path to
the implicit account's own credentials: `{CREDENTIALS_DIR}/implicit/{IMPLICIT_HEX}.json`.

## Step 3 — Generate the implicit account

```
near account create-account fund-later use-auto-generation save-to-folder {CREDENTIALS_DIR}/implicit
```

This writes a key file `{CREDENTIALS_DIR}/implicit/{IMPLICIT_HEX}.json`, where
`{IMPLICIT_HEX}` is the new implicit account's address (64 hex characters). Record
both the address and the key file path — every proposal will be signed with this file.

## Step 4 — Fund the implicit account

Ask the user **which account should sign the funding transaction** — call it
`{SIGNER}`. Then:

```
near tokens {SIGNER} send-near {IMPLICIT_HEX} '0.1 NEAR' network-config mainnet sign-with-keychain send
```

If the key isn't found in the keychain, retry with `sign-with-legacy-keychain`.

Verify the account now exists on-chain:

```
near account view-account-summary {IMPLICIT_HEX} network-config mainnet now
```

## Step 5 — Fund the treasury with USDT

The treasury can only pay invoices from its USDT balance. Ask the user how much USDT
to deposit (`{AMOUNT}`) and which account sends it (default: `{SIGNER}`, which must
hold that USDT):

```
near tokens {SIGNER} send-ft usdt.tether-token.near {TREASURY_ID} amount-ft '{AMOUNT} USDT' prepaid-gas '100.0 Tgas' attached-deposit '1 yoctoNEAR' network-config mainnet sign-with-keychain send
```

The user may skip this and deposit later — but proposals will fail to pay out until
the treasury holds USDT.

## Step 6 — Add the implicit account as a Requestor on trezu.app

Send the user to:

```
https://trezu.app/{TREASURY_ID}/members
```

Instruct them to:

1. Click **"Add New Member"**. If the button is disabled, there are most likely other
   pending requests that must be settled first before a new one can be created.
2. Enter the implicit account address `{IMPLICIT_HEX}`.
3. Select the **"Requestor"** role.
4. Submit the request and sign it in their NEAR Wallet.

## Step 7 — Verify and record

Once the user confirms the member request was approved, verify on-chain:

```
near contract call-function as-read-only {TREASURY_ID} get_policy json-args '{}' network-config mainnet now
```

Check that `{IMPLICIT_HEX}` appears in the Requestor role's member group. If it
doesn't, the member request is probably still pending on trezu — ask the user to
settle it and re-check.

Then persist the configuration and the standing rule:

```
memory_write(
  target: "projects/treasury/AGENTS.md",
  content: "# Treasury (Trezu invoice payments)\n\nThis project turns incoming invoices into USDT payment proposals on the Trezu treasury.\n\n## Configuration\n\n- Treasury: {TREASURY_ID}\n- Proposer (implicit, Requestor role): {IMPLICIT_HEX}\n- Key file: {CREDENTIALS_DIR}/implicit/{IMPLICIT_HEX}.json\n- Network: mainnet only\n- Token: usdt.tether-token.near (6 decimals)\n- Submission mode: confirm-first (only switch to auto-submit if the user explicitly asks)\n\n## Standing rule\n\nWhenever a new invoice arrives (attachment or text, any channel), process it into a payment proposal immediately using the trezu-invoice skill.\n",
  append: false
)
memory_write(target: "projects/treasury/.trezu-setup-complete", content: "done", append: false)
```

---

# Part 2: Invoice processing (recurring)

Run this for **every** new invoice. Do not deduplicate — if the same invoice arrives
twice, process it like any other.

## 1. Extract

From the invoice (any attachment type or pasted text), extract:

| Field                   | Required | Notes                                                                                               |
| ----------------------- | -------- | --------------------------------------------------------------------------------------------------- |
| Total amount due        | yes      | The final total, including tax — not a subtotal or line item                                        |
| Currency                | yes      | As denominated on the invoice                                                                       |
| Receiver NEAR address   | yes      | Must be present **on the invoice itself** — a named account (`*.near`) or a 64-hex implicit address |
| Vendor + invoice number | no       | Used in the proposal description                                                                    |

If the receiver address is missing or malformed, **stop** and tell the user exactly
what's missing — never guess or ask another source for the recipient.

## 2. Validate the currency

- **USD or USDT** → accepted, converted **1:1** (1 USD = 1 USDT, no FX ever).
- **Anything else** → reject, naming the currency: _"This invoice is in EUR; only
  USD/USDT invoices are supported."_
- Zero or negative totals (e.g. credit notes) → reject.

## 3. Convert to base units

USDT on NEAR (`usdt.tether-token.near`) has **6 decimals**. The on-chain amount is the
human-readable total × 10⁶, written as an integer string with no separators:

| Invoice total  | On-chain `amount` |
| -------------- | ----------------- |
| $15            | `"15000000"`      |
| $1,250.50      | `"1250500000"`    |
| $0.05          | `"50000"`         |
| 99.999999 USDT | `"99999999"`      |

## 4. Confirm with the user

Default mode is **confirm-first**. Show a summary and wait for approval:

```
Invoice #INV-042 from Acme Corp
Pay:      $1,250.50 USD → 1250500000 base units USDT
To:       acme.near
Treasury: {TREASURY_ID}
Submit this payment proposal?
```

Only submit without asking if `projects/treasury/AGENTS.md` says the user explicitly
opted into auto-submit. If the user ever asks to auto-submit from now on, update that
file and stop confirming.

## 5. Read the proposal bond

```
near contract call-function as-read-only {TREASURY_ID} get_policy json-args '{}' network-config mainnet now
```

Read `proposal_bond` (a yoctoNEAR string) and convert it to NEAR for the deposit
argument: divide by 10²⁴ (e.g. `"100000000000000000000000"` → `'0.1 NEAR'`). Call it
`{BOND}`.

## 6. Submit the proposal

```
near contract call-function as-transaction {TREASURY_ID} add_proposal json-args '{"proposal":{"description":"Invoice #INV-042 from Acme Corp","kind":{"Transfer":{"token_id":"usdt.tether-token.near","receiver_id":"acme.near","amount":"1250500000"}}}}' prepaid-gas '100.0 Tgas' attached-deposit '{BOND}' sign-as {IMPLICIT_HEX} network-config mainnet sign-with-access-key-file {CREDENTIALS_DIR}/implicit/{IMPLICIT_HEX}.json send
```

Description: a plain string — vendor and invoice number when available, otherwise a
short human-readable summary of the payment.

## 7. Report the result

On success, give the user the transaction hash and send approvers to:

```
https://trezu.app/{TREASURY_ID}/requests
```

On failure, surface the exact error. Common causes:

- **Permission denied** → the Requestor member request was never approved on trezu;
  re-run the Step 7 `get_policy` check from setup.
- **Insufficient balance for the bond** → the implicit account has run low; top it up
  with the Step 4 funding command.
