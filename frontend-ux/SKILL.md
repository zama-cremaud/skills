---
name: frontend-ux
description: Frontend UX patterns for confidential dApps. Encryption flows, the three decryption types, button states for async FHE operations, signature caching, XSS prevention, and how to display encrypted data. Use when building any frontend that interacts with FHEVM contracts.
---

# Frontend UX for Confidential dApps

## What You Probably Got Wrong

**You don't know the three decryption types.** There are three distinct ways to decrypt values in FHEVM. Each has different UX, different security implications, and different API calls:
1. **Public decrypt** (preferred term: "reveal public value") — anyone can read
2. **User decrypt** (preferred term: "decrypt") — only the authorized user
3. **Delegate decrypt** (preferred term: "decrypt on behalf of user") — a third party decrypts for the user

**You displayed encrypted handles as values.** Encrypted return values are opaque handles (`0x1a2b3c...`). Displaying them is meaningless. You must decrypt first — and decryption is async.

**You forgot the encrypting state.** FHE frontends have MORE loading states than regular dApps. Between "user clicks Transfer" and "transaction confirms," there's a new step: encrypting the input client-side. Your button needs to show "Encrypting..." before "Confirm in wallet..."

**You didn't cache signatures.** Every decryption requires an EIP-712 signature from the user. Without caching, users get a wallet popup on EVERY balance read. This destroys UX.

**You're vulnerable to XSS.** If an attacker injects JavaScript, they can steal cached signatures and decrypt the user's private balances. Signature/key storage is a critical security surface.

**Your frontend looks like AI slop.** Content jammed to the left with half the page empty. Flat cards with no depth. Hardcoded `bg-[#0a0a0a]` that ignores the template's theme system. Debug info (encrypted handles) visible in production. No visual hierarchy. Generic fonts. AI agents produce ugly UIs by default — this skill has a Visual Design section to prevent that. Read it before writing any JSX.

---

## Starting a New Frontend Project

**Always start with the official template:**

```bash
# Full React frontend with relayer-sdk integration
git clone https://github.com/zama-ai/fhevm-react-template
cd fhevm-react-template
npm install
```

If you already have a project and just need the SDK:
```bash
npm install @fhevm/sdk  # Check https://github.com/zama-ai/relayer-sdk for latest
```

**For extensions/plugins:** The current relayer-sdk has issues with non-browser environments. If building a browser extension or Node.js service, create a backend that proxies encryption/decryption requests.

---

## The Three Decryption Types

### 1. Public Decrypt (Reveal Public Value)

The decrypted value becomes public — readable by anyone. Used for auction results, vote tallies, game outcomes.

```typescript
// Contract side
function revealWinner() external {
    require(block.timestamp >= auctionEnd, "Not yet");
    FHE.makePubliclyDecryptable(winningBid);
    // Value will be readable by anyone after coprocessor processes it
}

// Frontend side
const publicValue = await contract.getRevealedValue();
// This is a plaintext uint64 — display directly
<span>Winning bid: {formatUnits(publicValue, 6)} USDC</span>
```

**UX:** Show "Revealing..." while waiting for coprocessor. Once revealed, display the value normally — no user signature needed.

### 2. User Decrypt (Decrypt)

Only the authorized user can decrypt their own values. Used for private balances, personal positions.

```typescript
// Read encrypted handle from contract
const encryptedBalance = await contract.balanceOf(userAddress);

// Decrypt using user's permission (requires EIP-712 signature)
const balance = await fhevm.decrypt(contractAddress, encryptedBalance);
// balance is a BigInt — display it
<span>Your balance: {formatUnits(balance, 6)} cUSDC</span>
```

**UX:** Requires wallet signature on first decrypt. Cache the signature to avoid repeated popups.

### 3. Delegate Decrypt (Decrypt on Behalf of User)

A third party (another contract or service) decrypts on behalf of the user. Used when a protocol needs to read a user's encrypted value to perform an action.

```solidity
// Contract grants delegate access
FHE.allow(balances[user], delegateAddress);
```

**UX:** The user must explicitly grant delegation. Show a clear confirmation: "Allow [Protocol] to read your balance?"

---

## Button States for Encrypted Operations

Every encrypted transaction has MORE states than a regular onchain button:

```typescript
type FHEButtonState =
  | "idle"           // Ready to click
  | "encrypting"     // Encrypting input client-side (fhevmjs)
  | "confirming"     // Waiting for wallet signature
  | "pending"        // Transaction submitted, waiting for confirmation
  | "decrypting"     // Fetching + decrypting updated values
  | "complete";      // Done

function getButtonText(state: FHEButtonState): string {
  switch (state) {
    case "encrypting": return "Encrypting...";
    case "confirming": return "Confirm in wallet...";
    case "pending": return "Transaction pending...";
    case "decrypting": return "Updating balance...";
    case "complete": return "Done!";
    default: return "Transfer";
  }
}
```

### Full Button Implementation

```tsx
const [state, setState] = useState<FHEButtonState>("idle");

<button
  disabled={state !== "idle"}
  onClick={async () => {
    try {
      // Step 1: Encrypt
      setState("encrypting");
      const input = fhevm.createEncryptedInput(contractAddress, userAddress);
      input.add64(amount);
      const encrypted = await input.encrypt();

      // Step 2: Submit transaction
      setState("confirming");
      const tx = await contract.transfer(
        recipient,
        encrypted.handles[0],
        encrypted.inputProof
      );

      // Step 3: Wait for confirmation
      setState("pending");
      await tx.wait();

      // Step 4: Refresh encrypted balance
      setState("decrypting");
      const newBalance = await contract.balanceOf(userAddress);
      const decrypted = await fhevm.decrypt(contractAddress, newBalance);
      setBalance(decrypted);

      setState("complete");
      setTimeout(() => setState("idle"), 2000);
    } catch (e) {
      console.error(e);
      setState("idle");
      // Show error to user
    }
  }}
>
  {getButtonText(state)}
</button>
```

**Critical:** Each button gets its own state. Never share loading state across buttons.

---

## Displaying Encrypted Data

### Balance Display Component

```tsx
function ConfidentialBalance({ contractAddress, userAddress }) {
  const [balance, setBalance] = useState<bigint | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    async function fetchBalance() {
      setLoading(true);
      try {
        const handle = await contract.balanceOf(userAddress);
        const decrypted = await fhevm.decrypt(contractAddress, handle);
        setBalance(decrypted);
      } catch (err) {
        // User may not have ACL permission
        setError("Unable to decrypt");
        setBalance(null);
      }
      setLoading(false);
    }
    fetchBalance();
  }, [userAddress]);

  if (loading) return <span className="animate-pulse">Decrypting...</span>;
  if (error) return <span className="text-gray-400">Hidden</span>;
  return <span>{formatUnits(balance!, 6)} cUSDC</span>;
}
```

**Rules:**
- Show "Decrypting..." while waiting for KMS response (not a number, not 0)
- Show "Hidden" if the user doesn't have ACL permission
- Never show raw handles — they're meaningless hex
- Cache decrypted values to avoid repeated KMS calls

### What Users Can and Cannot See

| Data | Visible to everyone | Visible to authorized user |
|------|--------------------|-----------------------------|
| Transaction sender/recipient | Yes | Yes |
| Function called | Yes | Yes |
| Encrypted input handles | Yes (meaningless) | Yes (can decrypt) |
| Encrypted balances | No | Yes (with signature) |
| Transfer amounts | No | Sender and recipient only |
| Total supply (if plaintext) | Yes | Yes |

---

## Signature Caching (Critical UX)

Without caching, every `fhevm.decrypt()` call triggers a wallet popup. For a page showing 3 balances, that's 3 MetaMask popups on load.

```typescript
// Cache the EIP-712 signature for the session
let cachedSignature: string | null = null;

async function getOrCacheSignature(contractAddress: string, userAddress: string) {
  if (cachedSignature) return cachedSignature;

  // This triggers the wallet popup
  const signature = await fhevm.generateSignature(contractAddress, userAddress);
  cachedSignature = signature;
  return signature;
}

// Now all decryptions in the session use the cached signature
const balance = await fhevm.decrypt(contractAddress, handle, {
  signature: await getOrCacheSignature(contractAddress, userAddress),
});
```

**Storage options:**
- **Session memory** (recommended) — lost on page reload, most secure
- **sessionStorage** — persists across page refreshes, cleared on tab close
- **Never localStorage** — survives tab close, XSS attack surface

---

## XSS Prevention (Critical Security)

Cached signatures are high-value targets. If an attacker injects JavaScript:
1. They read your cached signature from memory/storage
2. They call `fhevm.decrypt()` with your signature
3. They see all your private balances

**Mitigations:**
- **Content Security Policy** — strict CSP headers blocking inline scripts
- **No `eval()` or `innerHTML`** — standard React/Next.js already handles this
- **Sanitize all user inputs** — especially if rendering user-submitted content
- **Store signatures in memory, not DOM** — don't put them in data attributes or hidden inputs
- **Clear signatures on disconnect** — when user disconnects wallet, clear the cache

```typescript
// Clear cache on wallet disconnect
useEffect(() => {
  if (!isConnected) {
    cachedSignature = null;
  }
}, [isConnected]);
```

---

## Client-Side Encryption

### Creating Encrypted Inputs

```typescript
import { createInstance } from "fhevmjs";

// Initialize once at app startup
const fhevm = await createInstance({
  networkUrl: rpcUrl,
  gatewayUrl: "https://gateway.zama.ai",
});

// Encrypt a value for a specific contract
const input = fhevm.createEncryptedInput(contractAddress, userAddress);
input.add64(1000n);  // Encrypt 1000 as euint64
const encrypted = await input.encrypt();

// Submit to contract
await contract.transfer(
  recipient,
  encrypted.handles[0],    // externalEuint64
  encrypted.inputProof     // ZK proof
);
```

### Multiple Encrypted Values

```typescript
const input = fhevm.createEncryptedInput(contractAddress, userAddress);
input.addBool(true);     // index 0
input.add64(500n);       // index 1
input.add8(3);           // index 2
const encrypted = await input.encrypt();

await contract.multiInput(
  encrypted.handles[0],  // externalEbool
  encrypted.handles[1],  // externalEuint64
  encrypted.handles[2],  // externalEuint8
  encrypted.inputProof   // single proof for all
);
```

---

## Transaction Flow (Complete)

```
1. User enters amount (plaintext in the UI)
2. User clicks "Transfer"
3. → "Encrypting..." — fhevmjs encrypts client-side
4. → "Confirm in wallet..." — MetaMask popup
5. → "Transaction pending..." — waiting for block confirmation
6. → After confirm, re-fetch encrypted balance from contract
7. → "Decrypting..." — KMS decryption with cached signature
8. → Display updated balance
```

---

## Common Mistakes

### Displaying Raw Handles

```tsx
// ❌ WRONG — meaningless hex
<span>Balance: {encryptedBalance.toString()}</span>

// ✅ CORRECT — decrypt first
const decrypted = await fhevm.decrypt(contractAddr, encryptedBalance);
<span>Balance: {formatUnits(decrypted, 6)}</span>
```

### No Loading State for Decryption

```tsx
// ❌ WRONG — shows 0 while decrypting
<span>Balance: {balance || 0}</span>

// ✅ CORRECT — explicit loading
{isDecrypting ? <Spinner /> : <span>{formatBalance(balance)}</span>}
```

### Assuming Instant Decryption

```typescript
// ❌ WRONG — missing await
const balance = fhevm.decrypt(addr, handle);

// ✅ CORRECT
const balance = await fhevm.decrypt(addr, handle);
```

### Forgetting to Refresh After Transaction

```typescript
// ❌ WRONG — stale balance after transfer
await contract.transfer(...);
// Balance display still shows old value

// ✅ CORRECT — re-fetch and re-decrypt
await contract.transfer(...);
const newHandle = await contract.balanceOf(userAddress);
const newBalance = await fhevm.decrypt(contractAddr, newHandle);
setBalance(newBalance);
```

---

## Visual Design — Don't Ship Ugly

**AI agents produce ugly frontends by default.** Left-aligned content with half the page empty, flat cards with no depth, debug info visible, generic colors, no visual hierarchy. This section prevents that.

### Before Writing Any Frontend Code

Commit to a **design direction**. Don't just "make it work" — decide what it should feel like:

- **What is the tone?** Minimal and clean? Bold and expressive? Professional and corporate?
- **What makes it distinctive?** If your dApp looks like every other dark-mode crypto page, it's forgettable.
- **What's the layout strategy?** Centered content? Full-bleed? Dashboard grid?

### Layout Rules

```tsx
// ❌ WRONG — content hugs the left, half the page is empty
<div className="max-w-2xl px-6 py-12">

// ✅ CORRECT — centered, full-width background, contained content
<div className="min-h-screen w-full bg-base-200">
  <div className="max-w-4xl mx-auto px-6 py-12">
```

**Rules:**
- **Background covers the full viewport** — `min-h-screen w-full`. Never let the background stop halfway.
- **Content is centered** — `max-w-4xl mx-auto` or similar. Never left-aligned with dead space on the right.
- **Use the template's design system** — if using `fhevm-react-template`, use its existing Tailwind/DaisyUI classes. Don't hardcode `bg-[#0a0a0a]` when `bg-base-200` exists.
- **Responsive** — test at mobile, tablet, and desktop widths.

### Typography

```tsx
// ❌ WRONG — generic, no hierarchy
<h1 className="text-2xl font-bold">EthCC Tickets</h1>
<p className="text-gray-400">Some description</p>

// ✅ BETTER — clear size hierarchy, intentional spacing
<h1 className="text-5xl font-black tracking-tight mb-2">EthCC Tickets</h1>
<p className="text-lg text-base-content/60 max-w-lg">
  Confidential ticketing powered by FHE.
</p>
```

- **Size hierarchy matters** — h1 should be dramatically larger than body text, not just slightly bigger.
- **Don't use generic fonts** (Inter, Roboto, Arial). Pick a distinctive display font for headings.
- **Limit line width** — `max-w-lg` or `max-w-prose` on text blocks. Full-width paragraphs are unreadable.

### Cards and Depth

```tsx
// ❌ WRONG — flat, no depth, boring
<div className="bg-[#111] border border-[#222] rounded-lg p-4">

// ✅ BETTER — depth via shadow, subtle gradient, intentional rounding
<div className="bg-base-100 rounded-2xl shadow-xl p-6 ring-1 ring-base-300/50">
```

- **Use shadows for depth** — `shadow-lg`, `shadow-xl`. Flat cards look like placeholders.
- **Consistent border radius** — pick one size (e.g., `rounded-2xl`) and use it everywhere.
- **Padding matters** — `p-6` or `p-8`, not `p-4`. Give content room to breathe.

### Color

- **Use semantic colors** — `bg-base-100`, `text-base-content`, `bg-primary` — not hardcoded hex values. This respects the user's theme preference (light/dark).
- **One accent color** — pick a bold accent and use it sparingly for CTAs and important data. Not on everything.
- **Muted secondary text** — `text-base-content/60` (60% opacity) for descriptions, labels. Not full white.

### What NOT to Show

```tsx
// ❌ NEVER show in production — this is debug info
<div>Handle: 0x26e4ef4a...690200</div>

// ❌ NEVER show raw encrypted values
<span>{encryptedBalance.toString()}</span>

// ✅ Show meaningful state
<span className="text-primary font-bold">VIP</span>
// or if not yet decrypted:
<span className="animate-pulse text-base-content/40">Decrypting...</span>
```

**Production checklist:**
- [ ] No encrypted handles visible in the UI
- [ ] No "0x..." hex strings displayed to users
- [ ] No debug panels, console.logs, or dev-only state shown
- [ ] "Decrypting..." animation shown during KMS requests (not blank, not 0)
- [ ] "Encrypted" or a lock icon shown for values the user hasn't decrypted yet

### Encrypted State Visual Language

Create a consistent visual pattern for encrypted vs decrypted data:

```tsx
// Encrypted (not yet decrypted) — show a lock or blur
<div className="flex items-center gap-2 text-base-content/40">
  <LockIcon className="w-4 h-4" />
  <span>Encrypted</span>
</div>

// Decrypting — show animation
<div className="animate-pulse text-base-content/40">
  Decrypting...
</div>

// Decrypted — show the value with confidence
<div className="text-2xl font-bold text-primary">
  VIP
</div>
```

Users should always understand: **is this value encrypted, being decrypted, or readable?** Use consistent visual cues across your entire app.

### Motion

- **Page load** — stagger card reveals with `animate-fadeIn` delays
- **Decryption reveal** — animate the transition from "Encrypted" → actual value (scale up, color shift)
- **Button states** — pulse animation during "pending", checkmark on "complete"
- **Don't overdo it** — 2-3 high-impact animations, not motion on everything

### Anti-Patterns

- **Half-dark pages** — dark background that stops at the content boundary, leaving gray/white space
- **Left-aligned content** — content hugged to the left with empty right half
- **Hardcoded dark mode** — use `bg-base-200` not `bg-[#0a0a0a]` so themes work
- **Debug info in production** — encrypted handles, hex addresses, console output
- **No loading states** — blank or "0" while decrypting instead of a proper loading indicator
- **Generic AI slop** — same dark mode + purple gradient + Inter font that every AI generates

---

## Key Management for Production

For a production-ready frontend, you need to manage where and how you store decryption keys:

- **Development:** In-memory signatures, cleared on page reload
- **Staging:** sessionStorage with strict CSP
- **Production:** Consider a secure key management service, hardware-backed storage, or delegated decryption via a trusted backend

See `production-ready/SKILL.md` for the full production checklist.
