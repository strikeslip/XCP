## Bitcoin OnChain Art Deep Dive: Stamps, Counterwallet & XCP Extension

## Research Summary

This document breaks down the architecture of Bitcoin Stamps — the unprunable UTXO art protocol — from its origins in Counterwallet through to deploying Stamps via the new XCP/Extension browser wallet. It covers the technical mechanism of BASE64-to-multisig encoding, the evolution from Classic Stamps to OLGA, and a step-by-step guide for using the XCP Extension to create and deploy Stamps on Bitcoin.

---

## 1. The Origin: Counterwallet & Counterparty (XCP)

### What Was Counterwallet?

Counterwallet was the original web-based wallet for the Counterparty (XCP) protocol — a meta-protocol layer built on top of Bitcoin since 2014. It allowed users to create, send, trade, and manage custom digital assets (tokens) directly on the Bitcoin blockchain. The repository at `CounterpartyXCP/counterwallet` was **archived on July 19, 2024**, with 1,394 commits and 150 stars — now read-only.

Counterparty is historically significant: it powered some of the earliest NFT collections on any blockchain, including Spells of Genesis, Rare Pepes, Bassmint, and Phockheads — all predating Ethereum-based NFTs.

### How Counterparty Works (Relevant to Stamps)

Counterparty embeds meta-protocol data into Bitcoin transactions. Every Counterparty wallet is also a Bitcoin wallet with extra functionality. The key features relevant to Stamps:

- **Asset Issuance**: Users can create tokens with a `description` field embedded in the transaction. Named assets (4-12 uppercase characters) cost 0.5 XCP to issue. Numeric assets (prefixed with `A`) are free — only requiring a Bitcoin tx fee.
- **Description Field**: This is the critical field — it carries arbitrary text data that gets encoded into the Bitcoin transaction.
- **Bare Multisig Encoding**: When the description data exceeds 80 bytes (the OP_RETURN limit), Counterparty automatically encodes it across multiple **bare 2-of-3 multisig outputs** in the transaction. This is the key mechanism that makes Stamps possible.
- **DEX, Dispensers, Transfers**: Assets can be traded on Counterparty's built-in decentralized exchange, sold via dispensers, or transferred peer-to-peer.

---

## 2. Bitcoin Stamps: The Technical Mechanism

### What Makes a STAMP?

Bitcoin Stamps were created by **Mike In Space** (@mikeinspace) beginning March 7, 2023. The protocol definition lives at `github.com/mikeinspace/stamps`.

A Bitcoin Stamp is a Counterparty transaction containing a valid `STAMP:base64` string in the **description key** of an asset issuance. The stamp data — image art — lives permanently in Bitcoin's UTXO set, making it **impossible to prune** from a full node.

### The BASE64 → Multisig Pipeline (Classic Stamps)

Here is exactly how a Classic Bitcoin Stamp gets embedded into a UTXO transaction:

**Step 1 — Image Preparation**
- Create pixel art, ideally 24×24 pixels, 8-color-depth PNG or GIF
- File size constraint: roughly 7-8KB max (limited by Bitcoin transaction size)
- The fewer bytes the better — stamping is expensive

**Step 2 — BASE64 Encoding**
- The image binary is converted to a BASE64 string
- The string is prepended with `STAMP:` prefix
- Example: `STAMP:iVBORw0KGgoAAAANSUhE...` (the full base64 payload)

**Step 3 — Counterparty Transaction Construction**
- A Counterparty `issuance` transaction is created
- The `STAMP:base64` string is placed in the **description** field
- The asset **must be a numerical asset** (e.g., `A1997663462583877600`)
- The issuer sets quantity, divisibility, etc.

**Step 4 — Bare Multisig Encoding (The Key Innovation)**
- Because the `STAMP:base64` string far exceeds 80 bytes, Counterparty cannot use `OP_RETURN`
- Instead, Counterparty defaults to **bare 2-of-3 multisig** encoding
- The data is chunked across multiple multisig outputs in the transaction
- Each multisig output contains ~7,800 sats (historically; newer methods reduce this)
- The spending keys for these "fake multisig" outputs are assigned to **burn addresses** via "Key Burn" technology — making the outputs effectively unspendable

**Step 5 — Broadcast & Confirmation**
- The transaction is broadcast to the Bitcoin network
- It goes through normal mining/confirmation
- Sigop transactions (multisig) confirm slower than typical transactions in the mempool
- Expect hours or days for confirmation at moderate fee rates

**Step 6 — Indexing**
- Counterparty nodes decode the transaction
- The Stampchain indexer (`stampchain-io/btc_stamps`) extracts `STAMP:` prefixed descriptions
- The base64 is decoded back to the original image
- The stamp is assigned a sequential number based on transaction timestamp
- Images are served via `stampchain.io` API for web consumption

### Why This Matters: Unprunable UTXO Art

The critical distinction from Ordinals:

| Property | Bitcoin Stamps | Ordinals |
|----------|---------------|----------|
| Data Location | UTXO set (transaction outputs) | Witness data (SegWit) |
| Prunability | **Cannot be pruned** from full nodes | Can be pruned (witness data) |
| Cost | ~4x more expensive (no witness discount) | Cheaper (witness fee discount) |
| Permanence | Absolute — nodes must store this data | Dependent on node configuration |
| Encoding | BASE64 via Counterparty multisig | Raw data in witness |

Bitcoin Stamps are permanent because every full node **must** download and retain UTXO data to validate the blockchain. Witness data, by contrast, can be discarded.

---

## 3. Evolution: OLGA and Beyond

### OLGA (Octet Linked Graphics & Artifacts)

Introduced at **Block 833,000**, OLGA is a newer encoding format within the Stamps protocol:

- **Eliminates BASE64 encoding** — stores raw octets directly
- Uses **P2WSH** (Pay-to-Witness-Script-Hash) encoding instead of bare multisig
- Reduces transaction size by ~50%
- Reduces minting cost by 60-70%
- Supports Stamps up to **64KB**
- Maintains all original immutability properties
- Data still lives in the UTXO set — still unprunable

### POSH Stamps

POSH Stamps integrate with the Counterparty **named asset** system:
- Require XCP to issue (0.5 XCP fee for named assets)
- Allow vanity names (e.g., `MYARTWORK` instead of `A1997663462583877600`)
- Otherwise follow the same encoding pipeline

### SRC-20 Tokens

A fungible token standard built on Bitcoin Stamps, analogous to BRC-20:
- Originally used Counterparty encoding
- From block 796,000, SRC-20 encodes directly onto Bitcoin (no XCP dependency)
- Supports deploy, mint, and transfer operations

### SRC-721 & SRC-721r

Recursive NFT standards allowing layered composition of up to 10 stamp images via JSON references.

---

## 4. The Stampchain Ecosystem (GitHub: stampchain-io)

The `stampchain-io` GitHub organization hosts the open-source infrastructure:

| Repository | Purpose |
|-----------|---------|
| `btc_stamps` | Bitcoin Stamps Indexer (Python) — the reference indexer |
| `BTCStampsExplorer` | API / Explorer (TypeScript) — powers stampchain.io |
| `stamps_sdk` | TypeScript SDK for programmatic stamp interaction |
| `stamps` | Protocol definition (forked from mikeinspace/stamps) |
| `stampchain-mcp` | MCP (Model Context Protocol) server for Stamps |
| `counterparty-arm64` | Counterparty deployment for ARM64/AWS Graviton |

Notable API consumers include KuCoin, SuperEx, OpenStamp, and Leather wallet.

---

## 5. XCP Extension: The New Counterparty Browser Wallet

### Overview

The **XCP Extension** (`github.com/XCP/extension`) is a modern Counterparty Web3 browser extension wallet — the spiritual successor to Counterwallet for the browser era. It is built with TypeScript (98.9%), React, and Tailwind, using the WXT framework for extension development.

### Features

- **Multiple wallets and address types**: SegWit, Taproot, Legacy
- **Send/receive BTC and Counterparty assets**
- **Create dispensers and DEX orders**
- **Issue and manage assets** — this is the key feature for Stamps
- **Connect to dApps via provider API**
- **BIP-322 message signing**
- **Hardware wallet support** (Trezor)

### Security Architecture

- AES-256-GCM encryption with PBKDF2 (600,000 iterations)
- Local transaction verification (detects malicious API responses)
- Audited crypto libraries (noble/scure family — Cure53 audited)
- Minimal permissions, MV3 strict CSP, no remote code execution
- Only **12 runtime dependencies** (most wallets ship dozens)
- Not yet independently audited (self-reported checklist at `AUDIT.md`)

### Key Dependencies

| Package | Purpose |
|---------|---------|
| @noble/curves, @noble/hashes | Audited cryptography |
| @scure/* | BIP32/39 derivation |
| bignumber.js | Arbitrary precision arithmetic |
| React + react-router-dom | UI framework |
| @headlessui/react | Accessible components |
| webext-bridge | Extension messaging |

---

## 6. Step-by-Step: Deploying STAMPS via XCP Extension

### Prerequisites

Before you can deploy a Stamp using the XCP Extension, you need:

**1. Install the XCP Extension**
- The extension is **not yet on Chrome Web Store** (listed as "Coming Soon")
- For now, install from source:
  ```
  git clone https://github.com/XCP/extension.git
  cd extension
  npm install
  npm run build
  ```
- Go to `chrome://extensions/` in Chrome
- Enable **Developer Mode** (toggle in upper right)
- Click **"Load unpacked"**
- Select the build output directory from the extension folder

**2. Create or Import a Wallet**
- Open the XCP Extension popup
- Create a new wallet (generates a seed phrase — **back this up securely**)
- Or import an existing 12-word seed from Counterwallet, Freewallet, etc.
- Select an address type: **Legacy** is most compatible with Counterparty historically, but SegWit addresses are now supported

**3. Fund Your Wallet**
- Send **BTC** to your wallet address — you'll need enough for:
  - Transaction fees (variable based on mempool congestion)
  - Multisig output dust (each output requires ~546-7,800 sats depending on method)
  - A Classic Stamp can cost anywhere from 0.001-0.01+ BTC depending on image size and fee environment
- For **POSH Stamps** (named assets), you also need **0.5 XCP** for the naming fee
  - XCP can be acquired on Counterparty's DEX or dispensers

**4. Prepare Your Image**
- Create pixel art: recommended **24×24 pixels, 8-color-depth**
- Format: PNG, GIF, or WebP
- Optimize aggressively — every byte costs real money
- Tools: Use image optimizers to minimize file size
- Convert image to BASE64 string
  - Command line: `base64 -i image.png` (macOS) or `base64 image.png` (Linux)
  - Or use any online BASE64 encoder
- Prepend `STAMP:` to the BASE64 string
- The full description becomes: `STAMP:<your_base64_string>`

### The Stamping Process via XCP Extension

**Step 1 — Open Asset Issuance**
- Navigate to the **"Issue Asset"** or **"Manage Assets"** section in the XCP Extension
- Select **Numeric Asset** for a free stamp (no XCP required)
- Or select **Named Asset** if you want a vanity name (requires 0.5 XCP)

**Step 2 — Configure the Asset**
- **Asset Name**: For numeric assets, the system generates one (e.g., `A11223344556677`)
- **Quantity**: Set to 1 for a 1-of-1, or any number for editions
- **Divisible**: Typically `No` for art stamps
- **Description**: This is where the magic happens — paste your `STAMP:<base64_string>` here

**Step 3 — Set Transaction Fee**
- The XCP Extension will estimate the fee
- Be aware: Stamps use bare multisig, which means **sigop-heavy transactions**
- These confirm slower than standard transactions at the same fee rate
- Consider setting a higher fee rate if you want faster confirmation
- The total cost = mining fee + dust in multisig outputs

**Step 4 — Review & Sign**
- The extension will display the transaction details
- Review: source address, asset details, description (your STAMP data), fees
- The extension performs **local transaction verification** — it checks the constructed transaction against what the API returned
- Sign the transaction with your private key (or hardware wallet via Trezor)

**Step 5 — Broadcast**
- The signed transaction is broadcast to the Bitcoin network
- Monitor confirmation status in the extension or via mempool.space
- Expect slower-than-normal confirmation for multisig-heavy transactions

**Step 6 — Verify Your Stamp**
- After confirmation, check your stamp at:
  - `stampchain.io` — the official indexer/explorer
  - `xchain.io` — Counterparty block explorer
  - `xcp.io` — modern Counterparty explorer
- Verify the description field shows your `STAMP:base64` string
- Decode the base64 to confirm the image is intact

**Step 7 — Lock the Asset (Recommended)**
- After stamping, **lock the asset** to prevent supply changes
- This ensures collectors know the supply is fixed
- In XCP Extension: find the asset → select Lock option
- **Do NOT** change the description after stamping
- **Do NOT** transfer ownership of the asset (this creates an expensive duplicate multisig transaction)

### Using OLGA Encoding (Advanced)

For OLGA stamps (cheaper, post-block 833,000):
- OLGA eliminates the need for BASE64 encoding
- Uses P2WSH instead of bare multisig
- The process requires the **Stamps SDK** (`stampchain-io/stamps_sdk`) or platforms like **OpenStamp** that support OLGA natively
- Currently, direct OLGA stamping through the XCP Extension may require the extension to integrate the Stamps SDK or for you to construct the transaction externally
- Platforms supporting OLGA: OpenStamp, Stamped.ninja, and tools that use the stamps_sdk

---

## 7. Alternative Stamping Methods

For context, the XCP Extension is one of several paths to creating stamps:

| Method | Type | Notes |
|--------|------|-------|
| **XCP Extension** | Browser extension | Asset issuance with STAMP: description |
| **OpenStamp** | Web platform | Full minting, trading, OLGA support |
| **Stamped.ninja** | Web platform | Minting + marketplace |
| **CounterWallet V2** | Web wallet | Connects to Unisat, OKX, Leather, FreeWallet |
| **Stamp Wallet (Chrome)** | Chrome extension | Dedicated stamps wallet (caution: source code availability unclear) |
| **stamps_sdk** | TypeScript SDK | Programmatic stamping |
| **Freewallet** | Desktop wallet | Legacy Counterparty wallet, still functional |
| **Direct construction** | Manual | Build the raw transaction using Counterparty API |

---

## 8. Key Considerations & Gotchas

### Cost Structure
- Classic Stamps: No witness discount. ~4x more expensive than Ordinals
- OLGA Stamps: 60-70% cheaper than Classic, but still significant
- Named assets: Additional 0.5 XCP fee
- Multisig outputs: Each contains dust sats that are effectively burned

### Common Mistakes
- **Do NOT** increase token issuance after stamping (triggers expensive duplicate multisig tx)
- **Do NOT** change the description after creation
- **Do NOT** transfer asset ownership after issuance
- **Do NOT** delete trailing `=` signs from BASE64 (invalidates the stamp even if the image still renders)
- **Validate your BASE64** before broadcasting — decode it back to image first

### Mempool Behavior
- Sigop-heavy multisig transactions are deprioritized in the mempool
- They confirm much slower than typical transactions at the same fee rate
- Sites like mempool.space may show optimistic confirmation times that don't apply to stamp transactions

### XCP Extension Status
- **Chrome Web Store**: Not yet available (Coming Soon)
- **Current install method**: Build from source + Load Unpacked
- **Audit status**: Self-reported checklist, no independent audit yet
- **Open issues**: 0 (as of research date)
- **Commits**: 276 on main branch

---

## 9. Architecture Diagram: Stamp Data Flow

```
┌─────────────┐    BASE64 Encode    ┌───────────────────────┐
│  Pixel Art  │ ──────────────────► │ STAMP:<base64_string> │
│  24×24 PNG  │                     │  (Description Field)  │
└─────────────┘                     └──────────┬────────────┘
                                               │
                                    XCP Extension / Wallet
                                               │
                                    ┌──────────▼────────────┐
                                    │  Counterparty Asset   │
                                    │  Issuance Transaction │
                                    │  (Numeric or Named)   │
                                    └──────────┬────────────┘
                                               │
                           Data > 80 bytes ─── │
                                               │
                                    ┌──────────▼────────────┐
                                    │  Bare 2-of-3 Multisig │
                                    │  Output Encoding      │
                                    │  (Chunked across      │
                                    │   multiple outputs)   │
                                    └──────────┬────────────┘
                                               │
                                    ┌──────────▼───────────┐
                                    │  Bitcoin Transaction │
                                    │  Broadcast & Mining  │
                                    └──────────┬───────────┘
                                               │
                                    ┌──────────▼─────────────┐
                                    │  UTXO Set              │
                                    │  (Permanent, Unprunable│
                                    │   on ALL full nodes)   │
                                    └──────────┬─────────────┘
                                               │
                                    ┌──────────▼───────────┐
                                    │  Stampchain Indexer  │
                                    │  Decodes base64 →    │
                                    │  Original Image      │
                                    │  stampchain.io/docs  │
                                    └──────────────────────┘
```

---

## 10. Source Repositories Reference

| Repository | URL | Status |
|-----------|-----|--------|
| Counterwallet (Legacy) | github.com/CounterpartyXCP/counterwallet | Archived Jul 2024 |
| Bitcoin Stamps Protocol | github.com/mikeinspace/stamps | Active |
| Stampchain Indexer | github.com/stampchain-io/btc_stamps | Active |
| Stampchain Explorer/API | github.com/stampchain-io/BTCStampsExplorer | Active |
| Stamps SDK | github.com/stampchain-io/stamps_sdk | Active |
| Stampchain MCP | github.com/stampchain-io/stampchain-mcp | Active |
| XCP Extension | github.com/XCP/extension | Active, Pre-release |
| Counterparty Core | github.com/CounterpartyXCP/counterparty-core | Active (v10.4.0) |

---

*Research compiled February 2026. Protocol specifications and tooling are actively evolving. Always verify current documentation before stamping.*
