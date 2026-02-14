## How To Issue a Token with STAMP: Description via XCP Chrome Extension

## Complete Step-by-Step Operational Guide

---

## PHASE 0: Understanding What You're Building

When you "issue an asset" with a `STAMP:` description through the XCP Extension, here's exactly what happens under the hood:

```
Your Image (24×24 PNG)
    ↓
base64 encode → "iVBORw0KGgo..."
    ↓
prepend prefix → "STAMP:iVBORw0KGgo..."
    ↓
XCP Extension calls Counterparty API:
  compose/issuances {
    source: "your_btc_address",
    asset: "A11223344556677",     ← numeric (free) or "MYART" (0.5 XCP)
    quantity: 1,
    divisible: false,
    description: "STAMP:iVBORw0KGgo..."
  }
    ↓
API returns unsigned raw transaction (hex)
    ↓
XCP Extension signs locally with your private key
    ↓
Broadcast to Bitcoin network
    ↓
Counterparty encoding logic:
  - description ≤ 80 bytes? → OP_RETURN (never for stamps, too small)
  - description > 80 bytes? → BARE 2-of-3 MULTISIG outputs
  - Modern Counterparty Core: → P2TR (Taproot) encoding preferred
    ↓
Transaction mined → UTXO set → UNPRUNABLE FOREVER
    ↓
Stampchain indexer finds "STAMP:" prefix → decodes base64 → indexes image
```

**Key insight**: The XCP Extension doesn't "know" about Stamps per se. It simply issues a Counterparty asset. The magic is in the **description field** — by putting `STAMP:<base64>` there, the Stampchain indexer recognizes it as a Stamp.

---

## PHASE 1: Install XCP Extension

### Option A: Chrome Web Store (when available)

The extension is listed as "Coming Soon" to Chrome Web Store. Check `xcp.io` for updates.

### Option B: Build from Source (current method)

**Requirements**: Node.js, npm, Git, Chrome/Brave/Edge browser

```bash
# 1. Clone the repository
git clone https://github.com/XCP/extension.git

# 2. Enter directory
cd extension

# 3. Install dependencies
npm install

# 4. Build production version
npm run build

# OR for development with hot reload:
npm run dev
```

**Load into Chrome:**

1. Open `chrome://extensions/` in your browser
2. Toggle **Developer Mode** ON (top-right switch)
3. Click **"Load unpacked"**
4. Navigate to and select the build output directory (typically `.output/chrome-mv3/`)
5. The XCP Wallet icon appears in your extensions toolbar
6. Pin it for easy access

**Verify**: Click the extension icon — you should see the XCP Wallet interface.

---

## PHASE 2: Create / Import Wallet

### Create New Wallet

1. Click the XCP Extension icon
2. Select **"Create New Wallet"**
3. Set a strong password (this encrypts your seed locally with AES-256-GCM, PBKDF2 600k iterations)
4. **WRITE DOWN YOUR 12-WORD SEED PHRASE** — offline, on paper, never digital
5. Confirm the seed phrase when prompted
6. Your wallet generates addresses automatically

### Import Existing Wallet

If you have a seed from Counterwallet, Freewallet, or another Counterparty wallet:

1. Select **"Import Wallet"**
2. Enter your 12-word mnemonic
3. Set a password
4. Your existing addresses and balances will load

### Choose Address Type

The XCP Extension supports multiple address types:

| Type | Prefix | Use Case |
|------|--------|----------|
| **Legacy** | `1...` | Maximum compatibility with older XCP tools |
| **SegWit** | `bc1q...` | Lower base fees, modern standard |
| **Taproot** | `bc1p...` | Newest, enables P2TR encoding (cheaper for large data) |

**For Stamps**: Legacy addresses have the longest track record of compatibility. SegWit works with current Counterparty Core. Taproot addresses enable P2TR data encoding which is now preferred by Counterparty Core for data >80 bytes (more efficient than bare multisig).

---

## PHASE 3: Fund Your Wallet

### BTC Required

You need BTC to cover:

| Cost Component | Estimate | Notes |
|----------------|----------|-------|
| **Mining fee** | Variable | Depends on mempool, tx size |
| **Multisig dust outputs** | ~546-7,800 sats each | Multiple outputs per stamp |
| **Total per stamp** | ~0.001 - 0.01+ BTC | Varies with image size and fee environment |

**Rule of thumb**: Fund with at least **0.01 BTC** for your first stamp to have comfortable margin.

### XCP Required (Named Assets Only)

| Asset Type | XCP Cost | When Needed |
|------------|----------|-------------|
| **Numeric** (`A` prefix) | **0 XCP** | Free — only BTC tx fee |
| **Named** (4-12 chars) | **0.5 XCP** | Burned permanently |
| **Subasset** | **0.25 XCP** | E.g., `MYART.PIECE1` |

**Where to get XCP:**
- Counterparty DEX (decentralized exchange, built into the protocol)
- Dispensers on xchain.io or tokenscan.io
- OTC trades in the Counterparty Telegram community

### Send BTC to Your XCP Extension Address

1. Click your address in the extension to copy it
2. Send BTC from any exchange or wallet
3. Wait for at least 1 confirmation
4. Balance appears in the extension

---

## PHASE 4: Prepare Your Stamp Image

### Image Specifications

| Parameter | Recommended | Maximum |
|-----------|-------------|---------|
| **Dimensions** | 24×24 pixels | Any (but cost scales with size) |
| **Color depth** | 8 colors | Any |
| **Format** | PNG, GIF | Also: WebP, SVG, JPG |
| **File size** | < 500 bytes | ~7-8 KB (Bitcoin tx limit) |
| **Style** | Pixel art | Anything that fits |

**The smaller the file, the cheaper the stamp.** Every byte costs real money.

### Optimize Your Image

1. Create your pixel art (tools: Aseprite, Piskel, GIMP, Photoshop)
2. Export as PNG or GIF with minimal color palette
3. Run through an optimizer:
   ```bash
   # PNG optimization
   pngquant --quality=0-10 --speed 1 image.png
   optipng -o7 image.png

   # Or use online tools like TinyPNG
   ```
4. Check file size:
   ```bash
   ls -la image.png
   # Target: under 500 bytes for cheapest stamps
   # Median of first 400 stamps was 322 bytes
   ```

### Convert to BASE64

**Command Line (macOS):**
```bash
base64 -i image.png
```

**Command Line (Linux):**
```bash
base64 -w 0 image.png
```

**Command Line (Windows PowerShell):**
```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("image.png"))
```

**Python one-liner:**
```python
import base64; print("STAMP:" + base64.b64encode(open("image.png","rb").read()).decode())
```

**JavaScript/Node:**
```javascript
const fs = require('fs');
console.log("STAMP:" + fs.readFileSync("image.png").toString("base64"));
```

### Construct the Description String

Take your base64 output and prepend `STAMP:`:

```
STAMP:iVBORw0KGgoAAAANSUhEUgAAABgAAAAYCAYAAADgdz34AAAA...
```

**CRITICAL VALIDATION — Do this before spending any BTC:**

1. Copy ONLY the base64 portion (everything after `STAMP:`)
2. Paste into a base64-to-image decoder (e.g., `base64.guru/converter/decode/image`)
3. Confirm your image renders correctly
4. Check that trailing `=` or `==` padding characters are intact
5. **Do NOT delete the `=` signs** — this invalidates the stamp even if image still previews

---

## PHASE 5: Issue the Asset (Create the Stamp)

This is the core operation. In the XCP Extension:

### Step 1 — Navigate to Asset Issuance

- Open XCP Extension
- Find the **"Issue Asset"** or **"Create Asset"** function
- This maps to the Counterparty `create_issuance` / `compose/issuances` API call

### Step 2 — Select Asset Type

**Numeric Asset (FREE — recommended for first stamp):**
- No XCP required
- System generates a name like `A1997663462583877600`
- Only costs BTC transaction fee
- This is what most stamps use

**Named Asset (0.5 XCP):**
- You choose a name: 4-12 uppercase letters, NOT starting with 'A'
- Examples: `MYART`, `PIXELGODS`, `SHOOKSTAMP`
- This creates a "POSH Stamp" — vanity-named on-chain
- The 0.5 XCP is burned (destroyed) permanently

### Step 3 — Configure Asset Parameters

| Field | Value for Stamps | Why |
|-------|------------------|-----|
| **Quantity** | `1` | 1-of-1 unique art (or set editions: 10, 100, etc.) |
| **Divisible** | `No` / `false` | Art tokens should be whole units |
| **Description** | `STAMP:iVBORw0K...` | **THIS IS THE STAMP DATA** |
| **Lock** | Not yet — do this after | Lock supply after confirming everything works |

### Step 4 — Paste Your STAMP Description

In the **Description** field, paste your complete `STAMP:<base64_string>`.

**The description field is the entire mechanism.** This string:
- Gets encoded by Counterparty into the Bitcoin transaction
- If > 80 bytes → encoded via bare multisig outputs (Classic) or P2TR (modern Counterparty Core)
- Becomes permanently embedded in Bitcoin's UTXO set
- Gets detected by the Stampchain indexer via the `STAMP:` prefix
- Gets decoded from base64 back to your original image

### Step 5 — Set Transaction Fee

The XCP Extension will calculate and display the fee. Key considerations:

- **Stamp transactions are sigop-heavy** (multisig outputs count as multiple signature operations)
- Bitcoin Core deprioritizes high-sigop transactions in the mempool
- A stamp transaction confirms **significantly slower** than a normal BTC send at the same fee rate
- **Recommendation**: Set fee rate 20-50% higher than you'd normally use
- Check `mempool.space` for current conditions, but know their time estimates don't apply well to stamp txs

**Fee breakdown:**
```
Total Cost = Mining Fee + (Number of Multisig Outputs × Dust per Output)

Where:
- Mining Fee = tx_size_vbytes × fee_rate_sat/vB
- Dust per Output = 546 sats minimum (modern), up to 7,800 sats (legacy Counterwallet)
- Number of Outputs = ceil(base64_length / bytes_per_multisig_key)
```

### Step 6 — Review Transaction

Before signing, the XCP Extension displays:
- **Source address**: Your funded address
- **Asset name**: Numeric ID or your chosen name
- **Quantity**: Number of tokens
- **Description**: Your `STAMP:` string (verify it's complete)
- **Fee**: Total BTC cost
- **Outputs**: The multisig outputs that encode your data

The extension performs **local transaction verification** — it independently reconstructs the expected transaction and compares it against what the Counterparty API returned. This detects any malicious API tampering.

### Step 7 — Sign and Broadcast

1. Click **Sign** / **Confirm**
2. The extension signs the raw transaction with your private key locally
3. Transaction is broadcast to the Bitcoin network
4. You receive a **transaction ID (txid)**
5. **Save this txid** — you'll need it to verify your stamp

---

## PHASE 6: Wait for Confirmation

### Monitor Your Transaction

- **Mempool.space**: `mempool.space/tx/<your_txid>` — shows mempool position
- **XCP Extension**: Should show pending status
- **Note**: Sigop transactions sit longer in mempool than normal txs

### Typical Wait Times

| Fee Environment | Expected Wait |
|-----------------|---------------|
| Low fees (<10 sat/vB) | Could be hours to days for stamps |
| Medium (10-30 sat/vB) | 1-6 hours typically |
| High (>50 sat/vB) | Usually within 1-3 blocks |

**Do not panic if it takes a while.** This is normal for multisig-encoded transactions.

---

## PHASE 7: Verify Your Stamp

Once confirmed (1+ confirmations):

### Check on Counterparty Explorers

**xchain.io:**
- Search your txid or asset name
- The description field should show `STAMP:` followed by your base64 string
- The asset should appear under your address's holdings

**xcp.io:**
- Modern Counterparty block explorer
- Search by asset or address

### Check on Stampchain

**stampchain.io:**
- Your stamp should appear in the directory
- Assigned a sequential stamp number based on transaction timestamp
- Image decoded and rendered from the on-chain base64 data
- Verify via API: `stampchain.io/docs`

### Manual Verification

1. Copy the description field base64 string from xchain.io
2. Paste into a base64-to-image decoder
3. Confirm it matches your original image exactly

---

## PHASE 8: Post-Issuance Actions

### Lock the Asset (CRITICAL)

After verifying your stamp renders correctly:

1. In XCP Extension, find your asset
2. Select **"Lock Issuance"** or equivalent
3. This sets `description: "LOCK"` on the asset
4. **This is permanent** — no more tokens can ever be created
5. Collectors need this guarantee that supply is fixed

**API equivalent:**
```json
{
  "method": "create_issuance",
  "params": {
    "source": "your_address",
    "asset": "A11223344556677",
    "quantity": 0,
    "description": "LOCK"
  }
}
```

### Things You MUST NOT Do After Stamping

| Action | Why Not |
|--------|---------|
| **Change the description** | Overwrites your STAMP data — destroys the stamp |
| **Issue additional tokens** | Triggers another expensive multisig tx encoding the current description |
| **Transfer asset ownership** | Also triggers an expensive multisig tx duplicating description data |
| **Delete trailing `=` from base64** | Invalidates the stamp in the indexer even if images still render |

### Set Up a Dispenser (Optional — to sell your stamp)

If you want to sell your stamp for BTC:

1. In XCP Extension → **Create Dispenser**
2. Set the asset, price in BTC per unit, and escrow amount
3. Anyone can send BTC to the dispenser address to receive the stamp token automatically
4. Fully decentralized — no marketplace needed

### Trade on DEX (Optional)

The Counterparty DEX allows peer-to-peer trading:
- Create an order to sell your stamp for XCP, BTC, or any other Counterparty asset
- No counterparty risk — protocol acts as escrow

---

## Quick Reference: The Minimum Viable Stamp

For the fastest path from zero to stamp:

```bash
# 1. Create a 24×24 pixel art PNG, save as stamp.png

# 2. Generate the description string
python3 -c "
import base64
data = base64.b64encode(open('stamp.png','rb').read()).decode()
print(f'STAMP:{data}')
print(f'Size: {len(data)} chars')
" > stamp_description.txt

# 3. Copy that string

# 4. In XCP Extension:
#    - Issue Numeric Asset
#    - Quantity: 1
#    - Divisible: No
#    - Description: [paste STAMP:base64 string]
#    - Sign & Broadcast

# 5. Wait for confirmation

# 6. Lock the asset

# 7. Verify at stampchain.io
```

---

## Encoding Methods Reference

The XCP Extension / Counterparty Core determines encoding automatically based on data size:

| Data Size | Encoding Method | Cost Efficiency | Stamp Type |
|-----------|----------------|-----------------|------------|
| ≤ 80 bytes | OP_RETURN | Most efficient | Never for stamps (too small) |
| > 80 bytes (legacy) | Bare 2-of-3 Multisig | Expensive, linear scaling | Classic Stamp |
| > 64 bytes (modern) | **P2TR (Taproot)** | ~4x cheaper than multisig | Modern via Counterparty Core v10+ |
| Any (OLGA) | P2WSH | 50-70% cheaper than classic | OLGA Stamp (via stamps_sdk) |

**Current Counterparty Core** (v10.4+) prefers P2TR for data encoding when available. This means stamps issued today through the XCP Extension may automatically use Taproot encoding rather than bare multisig — resulting in lower fees while maintaining UTXO-set permanence.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Transaction stuck in mempool | Wait, or use RBF (Replace-By-Fee) if supported. Stamp txs are slow by nature. |
| Stamp not appearing on stampchain.io | Allow time for indexer to process. Check xchain.io first to confirm the XCP issuance succeeded. |
| "Invalid base64" error | Check for line breaks, missing padding (`=`), or corrupted characters in your base64 string. |
| Extension won't load | Ensure Developer Mode is on. Rebuild with `npm run build`. Check Chrome version compatibility. |
| Insufficient funds | Remember: stamp cost = mining fee + dust outputs. Budget more than a simple BTC send. |
| Asset already exists (named) | Named assets are globally unique. Choose a different name. Numeric assets are always unique. |
| Description too long | Reduce image size. Optimize PNG. Use fewer colors. Consider OLGA encoding for larger files. |

---

## Architecture: What the XCP Extension Does Internally

```
┌─────────────────────────────────────────────────────┐
│                 XCP EXTENSION                       │
│                                                     │
│  1. User fills Issue Asset form                     │
│     - asset type, quantity, description             │
│                                                     │
│  2. Extension calls Counterparty API v2:            │
│     POST /v2/addresses/<addr>/compose/issuances     │
│     {                                               │
│       "asset": "A11223344556677",                   │
│       "quantity": 1,                                │
│       "divisible": false,                           │
│       "description": "STAMP:iVBORw0K..."            │
│     }                                               │
│                                                     │
│  3. API returns unsigned raw transaction (hex)      │
│     - Counterparty Core has encoded the description │
│       into multisig/P2TR outputs automatically      │
│                                                     │
│  4. Extension VERIFIES the transaction locally      │
│     - Decodes the raw tx                            │
│     - Checks outputs match expected encoding        │
│     - Detects if API returned malicious tx          │
│                                                     │
│  5. Extension signs with local private key          │
│     - noble/curves library (Cure53 audited)         │
│     - Key never leaves the extension                │
│     - OR signs via Trezor hardware wallet           │
│                                                     │
│  6. Broadcasts signed tx to Bitcoin network         │
│                                                     │
└─────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│              BITCOIN NETWORK                        │
│                                                     │
│  Transaction contains:                              │
│  - Input: your UTXO (BTC to spend)                  │
│  - Output 1: OP_RETURN with XCP prefix identifier   │
│  - Outputs 2-N: Multisig/P2TR with encoded data     │
│    (each chunk of your STAMP:base64 description)    │
│  - Output N+1: Change back to your address          │
│                                                     │
│  Multisig outputs use Key Burn:                     │
│  - Spending keys → burn address                     │
│  - Outputs are effectively UNSPENDABLE              │
│  - Data permanently in UTXO set                     │
│                                                     │
└─────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────┐
│           STAMPCHAIN INDEXER                        │
│           (stampchain-io/btc_stamps)                │
│                                                     │
│  1. Monitors all Counterparty issuance transactions │
│  2. Checks description field for "STAMP:" prefix    │
│  3. Extracts base64 payload                         │
│  4. Validates base64 decodes to valid image         │
│  5. Assigns sequential stamp number by timestamp    │
│  6. Serves via API at stampchain.io/docs            │
│                                                     │
│  Your stamp is now:                                 │
│  - Permanently on Bitcoin (UTXO set)                │
│  - Indexed and viewable                             │
│  - Tradeable as a Counterparty asset                │
│  - A Bitcoin Stamp forever                          │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

*Guide current as of February 2026. The XCP Extension is pre-release software — verify current status at github.com/XCP/extension and xcp.io before proceeding.*
