---
name: migration
description: Converting existing Solidity contracts to use FHE for selective encryption. Step-by-step process for adding privacy to specific values without rewriting everything.
---

# Migration — Adding Privacy to Existing Contracts

## What You Probably Got Wrong

**You tried to encrypt everything.** Don't. Identify which values need privacy, encrypt only those. Every encrypted operation costs more gas, and encrypted values can't be used in standard DeFi composability.

**You kept the same function signatures.** Functions that accept encrypted inputs need `externalEuint64` + `bytes calldata inputProof` parameters. The interface changes.

**You forgot to rewrite the branching logic.** Every `if`, `require`, and `while` that touches an encrypted value must be converted to `FHE.select()`. This is the hardest part of migration.

---

## The Migration Process

### Step 1: Identify What to Encrypt

Go through every state variable and ask: **"Does this value need to be hidden?"**

```solidity
// BEFORE: Standard ERC-20
contract MyToken {
    mapping(address => uint256) public balances;        // ← ENCRYPT (private)
    mapping(address => mapping(address => uint256)) public allowances; // ← ENCRYPT
    uint256 public totalSupply;                         // ← KEEP PLAINTEXT
    string public name;                                 // ← KEEP PLAINTEXT
    string public symbol;                               // ← KEEP PLAINTEXT
    uint8 public decimals;                              // ← KEEP PLAINTEXT
}
```

**Rules of thumb:**
- Encrypt: balances, transfer amounts, votes, bids, personal data
- Keep plaintext: names, symbols, decimals, total supply, timestamps, addresses (usually), public config

### Step 2: Change Types

```solidity
// AFTER: Replace uint256 with euint64 for private values
import "@fhevm/solidity/lib/FHE.sol";
import {ZamaEthereumConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

contract MyToken is ZamaEthereumConfig {
    mapping(address => euint64) internal balances;       // euint64, internal
    mapping(address => mapping(address => euint64)) internal allowances;
    uint64 public totalSupply;                           // Keep plaintext
    string public name;
    string public symbol;
    uint8 public constant decimals = 6;
}
```

**Note:** `euint64` max is ~18.4 quintillion — sufficient for 6-decimal tokens. Use `euint64` by default, not `euint256`.

### Step 3: Update Function Signatures

```solidity
// BEFORE
function transfer(address to, uint256 amount) public returns (bool) {

// AFTER — encrypted amount from user input
function transfer(
    address to,
    externalEuint64 encryptedAmount,
    bytes calldata inputProof
) public returns (bool) {
    euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);
    // ...
}
```

### Step 4: Rewrite Branching Logic

This is the hardest part. Every `if` that touches an encrypted value becomes `FHE.select()`.

```solidity
// BEFORE
function transfer(address to, uint256 amount) public returns (bool) {
    require(balances[msg.sender] >= amount, "Insufficient balance");
    balances[msg.sender] -= amount;
    balances[to] += amount;
    emit Transfer(msg.sender, to, amount);
    return true;
}

// AFTER
function transfer(
    address to,
    externalEuint64 encryptedAmount,
    bytes calldata inputProof
) public returns (bool) {
    euint64 amount = FHE.fromExternal(encryptedAmount, inputProof);

    // Can't use require — use FHE.select instead
    ebool canTransfer = FHE.le(amount, balances[msg.sender]);
    euint64 transferValue = FHE.select(canTransfer, amount, FHE.asEuint64(0));

    // Update balances (both always execute)
    euint64 newFrom = FHE.sub(balances[msg.sender], transferValue);
    balances[msg.sender] = newFrom;
    FHE.allowThis(newFrom);
    FHE.allow(newFrom, msg.sender);

    euint64 newTo = FHE.add(balances[to], transferValue);
    balances[to] = newTo;
    FHE.allowThis(newTo);
    FHE.allow(newTo, to);

    // Event: no amount (privacy)
    emit Transfer(msg.sender, to);
    return true;
}
```

### Step 5: Add ACL to Every State Update

After every line that writes an encrypted value to storage:

```solidity
// Pattern: store → allowThis → allow(user)
balances[user] = newBalance;
FHE.allowThis(newBalance);
FHE.allow(newBalance, user);
```

### Step 6: Update Events

```solidity
// BEFORE — amount in event
event Transfer(address indexed from, address indexed to, uint256 amount);

// AFTER — no amount
event Transfer(address indexed from, address indexed to);
```

### Step 7: Update `balanceOf` Return Type

```solidity
// BEFORE
function balanceOf(address account) public view returns (uint256) {
    return balances[account];
}

// AFTER — returns encrypted handle, only authorized can decrypt
function balanceOf(address account) public view returns (euint64) {
    return balances[account];
}
```

---

## Migration Checklist

- [ ] Identified which values to encrypt (minimal set)
- [ ] Changed types: `uint256` → `euint64`, `bool` → `ebool` for private values
- [ ] Added `ZamaEthereumConfig` inheritance
- [ ] Updated function signatures: added `externalEuint64` + `inputProof` params
- [ ] Added `FHE.fromExternal()` for all encrypted inputs
- [ ] Rewrote all `if/require` on encrypted values to `FHE.select()`
- [ ] Added `FHE.allowThis()` + `FHE.allow()` after every encrypted state update
- [ ] Removed encrypted values from events
- [ ] Removed `view` modifier from functions with FHE operations
- [ ] Updated return types for functions that return encrypted values
- [ ] Added `FHE.isInitialized()` checks where needed
- [ ] Updated tests to use `fhevm.createEncryptedInput()` and `fhevm.decrypt*()`
- [ ] Updated frontend to use fhevmjs for encryption/decryption
