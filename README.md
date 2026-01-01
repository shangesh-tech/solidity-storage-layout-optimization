# solidity-storage-layout-optimization

## The Core Concept: 32-Byte Storage Slots

**The EVM stores data in 32-byte (256-bit) slots**. Think of storage like a giant warehouse with shelves - each shelf holds exactly 32 bytes.

```
┌─────────────────────────────────────┐
│     STORAGE SLOT 0 (32 bytes)       │  ← One shelf
├─────────────────────────────────────┤
│     STORAGE SLOT 1 (32 bytes)       │  ← Another shelf
├─────────────────────────────────────┤
│     STORAGE SLOT 2 (32 bytes)       │
└─────────────────────────────────────┘
```

**Critical rule:** Reading or writing to a storage slot costs **~2,100 gas (cold) or ~100 gas (warm)**. So **fewer slots = lower gas**.[4][5]

***

## The Packing Rules (MEMORIZE THIS)

**Rule 1:** Solidity packs variables **sequentially from right to left** (least significant byte first).[2][3][1]

**Rule 2:** If a variable **doesn't fit in remaining space**, it moves to the **next slot**.[3][1][4]

**Rule 3:** Dynamic types (`string`, `array`, `mapping`) **ALWAYS start a new slot**.[1][2]

**Rule 4:** Variables after dynamic types **start a new slot**.[2][1]

***

## Type Sizes (You Need to Know These)

| Type | Size (bytes) | Can Pack With |
|------|--------------|---------------|
| `bool` | 1 byte | address, uint248, etc. |
| `uint8` | 1 byte | address, uint248, etc. |
| `uint16` | 2 bytes | address, uint240, etc. |
| `uint32` | 4 bytes | address, uint224, etc. |
| `uint64` | 8 bytes | address, uint192, etc. |
| `uint128` | 16 bytes | address, uint128, etc. |
| `uint256` | 32 bytes | **Takes full slot** |
| `address` | 20 bytes | uint96, bool, uint8, etc. |
| `string` | Variable | **Starts new slot** |
| `mapping` | Variable | **Starts new slot** |
| `array[]` | Variable | **Starts new slot** |

***

## Example 1: BAD Packing (Wastes 2 Slots)

```solidity
struct UserBad {
    address owner;     // 20 bytes → Slot 0 (12 bytes wasted)
    uint256 balance;   // 32 bytes → Slot 1 (too big, new slot)
    bool isActive;     // 1 byte   → Slot 2 (31 bytes wasted!)
}
// TOTAL: 3 slots = ~42,000 gas
```

**Visual:**
```
Slot 0: [owner (20 bytes)][WASTED 12 bytes]
Slot 1: [balance (32 bytes)]
Slot 2: [isActive (1 byte)][WASTED 31 bytes]
```

***

## Example 2: GOOD Packing (Saves 1 Slot)

```solidity
struct UserGood {
    address owner;     // 20 bytes → Slot 0
    bool isActive;     // 1 byte   → Slot 0 (PACKED!)
    uint256 balance;   // 32 bytes → Slot 1
}
// TOTAL: 2 slots = ~22,000 gas (saves ~20,000 gas!)
```

**Visual:**
```
Slot 0: [owner (20 bytes)][isActive (1 byte)][WASTED 11 bytes]
Slot 1: [balance (32 bytes)]
```

**Why this works:** `address` (20 bytes) + `bool` (1 byte) = 21 bytes, fits in one slot.[4][3]

***

## Example 3: Your Election Struct (BEFORE vs AFTER)

### BEFORE (Inefficient):
```solidity
struct ElectionStruct {
    address creatorAddress;        // 20 bytes → Slot 0
    string name;                   // Variable → Slot 1+ (string starts new slot)
    string description;            // Variable → Slot 2+
    string image;                  // Variable → Slot 3+
    CandidateStruct[] candidates;  // Variable → Slot 4 (array starts new slot)
    uint256 deadline;              // 32 bytes → Slot 5
    mapping(address => bool) hasVoted; // Variable → Slot 6 (mapping starts new slot)
    uint256 totalVotes;            // 32 bytes → Slot 7
    bool cancelled;                // 1 byte   → Slot 8 (WASTES 31 BYTES!)
}
// TOTAL: 9 slots
```

### AFTER (Optimized):
```solidity
struct ElectionStruct {
    address creatorAddress;        // 20 bytes → Slot 0
    bool cancelled;                // 1 byte   → Slot 0 (PACKED!)
    string name;                   // Variable → Slot 1+
    string description;            // Variable → Slot 2+
    string image;                  // Variable → Slot 3+
    CandidateStruct[] candidates;  // Variable → Slot 4
    uint256 deadline;              // 32 bytes → Slot 5
    mapping(address => bool) hasVoted; // Variable → Slot 6
    uint256 totalVotes;            // 32 bytes → Slot 7
}
// TOTAL: 8 slots (saves ~5,000 gas per election!)
```

**Key insight:** Moving `bool cancelled` right after `address` packs them together.[3][2][4]

***

## The Algorithm to Optimize ANY Struct

**Step 1: Identify variable sizes**
- Count bytes for each type (see table above)

**Step 2: Group by size**
- Small types (<32 bytes)
- Full-slot types (uint256, etc.)
- Dynamic types (string, array, mapping)

**Step 3: Order by this pattern:**
```solidity
struct Optimized {
    // 1. PACK small types together (address + bool, uint128 + uint128, etc.)
    address owner;
    bool flag;
    
    // 2. Then full-slot types
    uint256 bigNumber;
    
    // 3. LAST: Dynamic types (they can't pack anyway)
    string name;
    mapping(address => uint256) balances;
    uint256[] ids;
}
```

**Why this order:** Dynamic types always start new slots, so put them last to avoid wasting space before them.[1][2][4]

***

## Example 4: Complex Real-World Optimization

### BEFORE (6 slots):
```solidity
struct TokenData {
    uint256 totalSupply;    // Slot 0
    address owner;          // Slot 1 (20 bytes, wastes 12)
    string name;            // Slot 2+
    uint8 decimals;         // Slot 3 (1 byte, wastes 31!)
    bool paused;            // Slot 4 (1 byte, wastes 31!)
    mapping(address => uint256) balances; // Slot 5
}
```

### AFTER (4 slots):
```solidity
struct TokenData {
    address owner;          // Slot 0 (20 bytes)
    uint8 decimals;         // Slot 0 (1 byte)
    bool paused;            // Slot 0 (1 byte) → 22 bytes total, PACKED!
    uint256 totalSupply;    // Slot 1
    string name;            // Slot 2+
    mapping(address => uint256) balances; // Slot 3
}
// Saves 2 slots = ~10,000 gas!
```

***

## When NOT to Pack (Important!)

**Don't pack if you're accessing variables separately frequently**. Packing adds computational overhead to extract/update individual values.[4][3]

```solidity
// BAD: If you only read `balance` frequently
struct User {
    address owner;    // You'll pay gas to unpack this
    bool active;      // And this
    uint256 balance;  // Just to read this
}

// BETTER: If balance is accessed alone often
struct User {
    uint256 balance;  // Slot 0 (fast access)
    address owner;    // Slot 1
    bool active;      // Slot 1 (packed)
}
```

**Rule:** Pack variables that are **read/written together in the same transaction**.[5][4]

***

## Tools to Check Your Packing

**1. Foundry's `forge inspect` command:**
```bash
forge inspect YourContract storage-layout --pretty
```

**2. Hardhat's storage layout plugin:**
```bash
npx hardhat check-storage-layout
```

These show you exactly which variables are in which slots.[6][3]

***

## Your Takeaway Checklist

For **every struct** you write:

1. ✅ List all variable types and their sizes
2. ✅ Group: small types → full-slot types → dynamic types
3. ✅ Pack small types together (address + bool, uint128 + uint128)
4. ✅ Put dynamic types (string, array, mapping) **last**
5. ✅ Run `forge inspect storage-layout` to verify
6. ✅ Test gas costs before/after to measure savings

