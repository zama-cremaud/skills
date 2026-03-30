---
name: ship
description: End-to-end guide for AI agents — from idea to deployed encrypted dApp. Fetch this FIRST, it routes you through all other skills.
---

# Ship an Encrypted dApp

## What You Probably Got Wrong

**You treat FHE like regular encryption.** FHE is not encrypt-store-decrypt. It's compute-on-ciphertexts. You never decrypt to check a condition — you compute the result encrypted and branch with `FHE.select()`. If you're thinking "decrypt, check, re-encrypt," you've already leaked the value.

**You encrypt everything.** Only encrypt what MUST be private. Token balances? Maybe. Transfer amounts? Yes. Token names? No. Contract owner? No. Encryption costs gas — every `FHE.add()` is a state-changing operation, not a `view` call. Encrypt the minimum set of values that gives your users privacy.

**You forget ACL permissions.** Every encrypted value in fhEVM has an Access Control List. If you store an encrypted balance and forget to call `FHE.allowThis()`, your own contract can't use it on the next call. If you forget `FHE.allow(value, user)`, the user can't decrypt their own balance. This compiles fine, deploys fine, and silently breaks at runtime.

**You try to branch on encrypted values.** `if (FHE.gt(a, b))` does not work. You cannot use encrypted booleans in `if`, `require`, or `while`. The ONLY way to branch on encrypted conditions is `FHE.select(condition, valueIfTrue, valueIfFalse)`. This changes how you design every function.

**You skip the mental model shift.** Building with FHE requires a fundamentally different way of thinking about smart contracts. Fetch `concepts/SKILL.md` before writing any code.

---

## Phase 0 — Understand the Model

Do this BEFORE writing any code.

### The Privacy Litmus Test

Encrypt it if:
- **Balances** — users shouldn't see each other's holdings
- **Transfer amounts** — transaction values should be private
- **Votes** — votes must be secret until tallying
- **Bids** — auction bids must be sealed until reveal
- **Personal data** — addresses, scores, classifications that are private by nature

Keep it plaintext if:
- Token name, symbol, decimals (public metadata)
- Total supply (unless hiding market cap is the point)
- Contract owner, admin addresses
- Timestamps, block numbers
- Any value that must be readable by other contracts without decryption

**The rule:** If revealing the value to everyone is fine, don't encrypt it. Every encrypted operation costs significantly more gas than its plaintext equivalent.

### The FHE.select() Rewrite

Every `if` statement that touches an encrypted value must be rewritten:

```solidity
// PLAINTEXT THINKING (what you're used to):
if (balance >= amount) {
    balance -= amount;
    recipientBalance += amount;
}

// FHE THINKING (what you must do):
ebool canTransfer = FHE.le(amount, balance);
euint64 transferValue = FHE.select(canTransfer, amount, FHE.asEuint64(0));
balance = FHE.sub(balance, transferValue);
recipientBalance = FHE.add(recipientBalance, transferValue);
// Don't forget ACL:
FHE.allowThis(balance);
FHE.allow(balance, sender);
FHE.allowThis(recipientBalance);
FHE.allow(recipientBalance, recipient);
```

If `canTransfer` is false, `transferValue` is 0 — so the subtraction and addition are no-ops. No branch, no information leak. **This is the core pattern of all fhEVM development.**

### MVP Contract Count

| What you're building | Contracts | Pattern |
|---------------------|-----------|---------|
| Confidential token | 1 | Confidential ERC-20 with encrypted balances |
| Encrypted voting | 1 | Vote accumulator with encrypted tallies |
| Sealed-bid auction | 1 | Encrypted bids, FHE.max for winner |
| Private credential | 1 | Encrypted attribute storage + selective reveal |
| Confidential DEX | 1-2 | Encrypted order book or AMM with hidden reserves |
| Privacy mixer | 1 | Deposit/withdraw with encrypted tracking |

**If you need more than 2 contracts for an MVP, you're over-building.**

---

## dApp Archetype Templates

### 1. Confidential Token (1 contract)

**Architecture:** One contract — ERC-20 interface with encrypted balances and transfer amounts.

**Contracts:**
- `ConfidentialERC20.sol` — encrypted balances, encrypted transfers, public total supply

**Common mistakes:**
- Forgetting ACL after balance updates (value becomes unusable)
- Using `euint256` for balances (`euint64` is standard and cheaper)
- Emitting transfer amounts in events (defeats the privacy purpose)
- Marking FHE functions as `view` (they cost gas)

**Fetch sequence:** `fhevm/SKILL.md` → `acl/SKILL.md` → `patterns/SKILL.md` → `testing/SKILL.md`

### 2. Encrypted Voting (1 contract)

**Architecture:** One contract — encrypted vote submission, homomorphic tallying, public reveal after deadline.

**Contracts:**
- `ConfidentialVoting.sol` — encrypted votes, `FHE.add()` for tallying, `makePubliclyDecryptable` for results

**Common mistakes:**
- Revealing vote counts before the deadline (use time-locked decryption)
- Allowing double-voting without encrypted duplicate detection
- Using `if` to check vote validity instead of `FHE.select()`

**Fetch sequence:** `fhevm/SKILL.md` → `acl/SKILL.md` → `security/SKILL.md` → `testing/SKILL.md`

### 3. Sealed-Bid Auction (1 contract)

**Architecture:** One contract — encrypted bids, `FHE.max()` to find winner, refunds for losers.

**Contracts:**
- `SealedBidAuction.sol` — encrypted bid submission, winner determination, reveal + settlement

**Common mistakes:**
- Comparing bids with `if` instead of `FHE.gt()` + `FHE.select()`
- Forgetting to handle ties (what if two bids are equal?)
- Revealing the winning bid amount before the auction closes

**Fetch sequence:** `fhevm/SKILL.md` → `acl/SKILL.md` → `patterns/SKILL.md` → `security/SKILL.md`

### 4. Adding Privacy to Existing Contract (0 new contracts)

**Architecture:** Modify an existing contract — replace specific `uint256` fields with `euint64`, add ACL grants.

**Common mistakes:**
- Encrypting values that other contracts read (breaks composability)
- Forgetting to update function signatures to accept `externalEuint64` + `inputProof`
- Not adding ACL grants for the contract itself (`FHE.allowThis()`)

**Fetch sequence:** `migration/SKILL.md` → `fhevm/SKILL.md` → `acl/SKILL.md` → `testing/SKILL.md`

---

## Phase 1 — Build Contracts

**Fetch:** `fhevm/SKILL.md`, `acl/SKILL.md`, `patterns/SKILL.md`, `security/SKILL.md`

Key guidance:
- Import `@fhevm/solidity/lib/FHE.sol` and inherit `ZamaEthereumConfig`
- Use `euint64` for balances and amounts — it's the standard type
- Call `FHE.allowThis()` + `FHE.allow(value, user)` after EVERY encrypted state update
- Use `FHE.fromExternal(handle, inputProof)` to validate all encrypted inputs
- Use `FHE.select()` for ALL conditional logic on encrypted values
- Emit events with metadata (addresses, timestamps) but NOT encrypted values
- Never mark functions with FHE operations as `view` — they cost gas

---

## Phase 2 — Test

**Fetch:** `testing/SKILL.md`

Key guidance:
- Use `fhevm.createEncryptedInput()` to create encrypted test inputs
- Decrypt values in tests to verify correctness — tests are the only place you should decrypt freely
- Test ACL: verify that unauthorized addresses can't use encrypted values
- Test the `FHE.select()` branches: what happens when the condition is true? False?
- Test with zero values, max values, and uninitialized encrypted values

---

## Phase 3 — Build Frontend

**Fetch:** `frontend/SKILL.md`, `tools/SKILL.md`

Key guidance:
- Users encrypt values client-side with `fhevm.createEncryptedInput()`
- Pass `encrypted.handles[i]` and `encrypted.inputProof` to contract functions
- Decryption is async — show loading states while waiting for decrypted values
- Only the user (or addresses they've granted access) can decrypt their values

---

## Phase 4 — Deploy

**Fetch:** `deployment/SKILL.md`, `addresses/SKILL.md`

Key guidance:
- Deploy to Sepolia first — FHE operations consume real coprocessor resources
- Inherit `ZamaEthereumConfig` for automatic chain configuration
- Verify contracts on the block explorer
- Test one encrypted operation end-to-end on testnet before mainnet

---

## Skill Routing Table

| Phase | What you're doing | Skills to fetch |
|-------|-------------------|-----------------|
| **Understand** | Mental models, FHE basics | `ship/` (this), `concepts/` |
| **Design** | Architecture, what to encrypt | `ship/` (this), `migration/` |
| **Contracts** | Writing encrypted Solidity | `fhevm/`, `acl/`, `patterns/`, `security/` |
| **Test** | Testing encrypted contracts | `testing/` |
| **Frontend** | Building UI for encrypted dApp | `frontend/`, `tools/` |
| **Deploy** | Ship to testnet/mainnet | `deployment/`, `addresses/` |

**Base URLs:** All skills are at `https://fheskills.com/<skill>/SKILL.md`
