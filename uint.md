## Complete Solidity Unsigned Integer (uint) Reference Table

| Type | Size (bytes) | Size (bits) | Min Value | Max Value | Common Use Cases |
|------|-------------|-------------|-----------|-----------|------------------|
| `uint8` | 1 byte | 8 bits | 0 | 255 | Age, percentage (0-100), status codes, boolean flags, small counters [2][6] |
| `uint16` | 2 bytes | 16 bits | 0 | 65,535 | Year (2025), port numbers, basis points (0-10000), token IDs for small collections [6][7] |
| `uint32` | 4 bytes | 32 bits | 0 | 4,294,967,295 | Timestamps (until year 2106), large counters, Unix time [2][3] |
| `uint64` | 8 bytes | 64 bits | 0 | 18,446,744,073,709,551,615 | Timestamps (far future), millisecond timestamps, very large counters [2][3] |
| `uint128` | 16 bytes | 128 bits | 0 | 340,282,366,920,938,463,463,374,607,431,768,211,455 | Token amounts for low-decimal tokens, intermediate calculations [2][3] |
| `uint256` | 32 bytes | 256 bits | 0 | 2^256 - 1 (≈1.16 × 10^77) | Token balances (18 decimals), ETH amounts (wei), large financial values, default integer [2][3][4] |

***

## Signed Integer (int) Reference Table

| Type | Size (bytes) | Size (bits) | Min Value | Max Value | Common Use Cases |
|------|-------------|-------------|-----------|-----------|------------------|
| `int8` | 1 byte | 8 bits | -128 | 127 | Temperature (Celsius), small +/- deltas [2][6] |
| `int16` | 2 bytes | 16 bits | -32,768 | 32,767 | Price changes, elevation (meters), coordinate offsets [2][6] |
| `int32` | 4 bytes | 32 bits | -2,147,483,648 | 2,147,483,647 | Latitude/longitude (scaled), large +/- values [2][3] |
| `int64` | 8 bytes | 64 bits | -9,223,372,036,854,775,808 | 9,223,372,036,854,775,807 | Financial gains/losses, time deltas [2][3] |
| `int128` | 16 bytes | 128 bits | -2^127 | 2^127 - 1 | Signed token amounts, debt tracking [2][3] |
| `int256` | 32 bytes | 256 bits | -2^255 | 2^255 - 1 | Large signed financial calculations, profit/loss tracking [2][3][4] |

***

## Practical Code Examples

### Example 1: Using Correct Sizes Saves Gas
```solidity
struct UserBad {
    uint256 age;        // WASTEFUL - uses 32 bytes for value max 120
    uint256 level;      // WASTEFUL - uses 32 bytes for value max 100
}

struct UserGood {
    uint8 age;          // 1 byte - sufficient for 0-255
    uint8 level;        // 1 byte - sufficient for 0-255
    // These pack together in 1 slot instead of 2!
}
```

### Example 2: Election Voting Example
```solidity
struct Election {
    address creator;           // 20 bytes
    uint32 startTime;          // 4 bytes - Unix timestamp (valid until 2106)
    uint32 endTime;            // 4 bytes - packs with startTime and creator!
    uint16 candidateCount;     // 2 bytes - max 65,535 candidates
    uint8 status;              // 1 byte - 0=pending, 1=active, 2=ended
    // All fit in 1 storage slot (31 bytes total)!
}
```

### Example 3: Token Amounts
```solidity
// ALWAYS use uint256 for financial amounts with 18 decimals
uint256 ethAmount = 1.5 ether;  // = 1,500,000,000,000,000,000 wei

// USDC (6 decimals) could use uint128, but uint256 is safer
uint256 usdcAmount = 1000 * 10**6;  // 1000 USDC
```

### Example 4: Basis Points (Finance)
```solidity
// Basis points: 1 bp = 0.01%
// Range: 0-10,000 (0% to 100%)
uint16 feeInBasisPoints = 250;  // 2.5%
uint16 maxFee = 10000;          // 100%

// uint16 is perfect (max 65,535) [web:59]
```

***

## Critical Rules to Remember

**1. `uint` = `uint256` (they're aliases)**[3][5]
- Always prefer `uint256` for clarity[5][3]

**2. Default to `uint256` for arithmetic**[4]
- EVM is optimized for 256-bit operations[4]
- Smaller types add conversion gas costs during calculations[4]

**3. Use smaller types ONLY for storage packing**[2][6]
- `uint8 age` saves gas only when packed with other variables[2]
- Standalone `uint8` doesn't save gas vs `uint256`[4]

**4. Never use small types for loop counters**[4]
```solidity
// BAD - wastes gas on type conversion
for (uint8 i = 0; i < 100; i++) { }

// GOOD - EVM-optimized
for (uint256 i = 0; i < 100; i++) { }
```

***

## Decimals and Floats: THEY DON'T EXIST

**Solidity has NO floating-point types**. Here's how to handle decimals:[1][10]

### Method 1: Fixed-Point Arithmetic (Most Common)
```solidity
// Represent 1.5 ETH as 1.5 * 10^18 wei
uint256 onePointFiveEth = 1.5 ether;  // = 1,500,000,000,000,000,000

// Represent $19.99 with 2 decimals
uint256 price = 1999;  // = $19.99 (divide by 100 for display)

// Represent 50% as basis points
uint256 percentage = 5000;  // 5000 bp = 50% (divide by 10000)
```

### Method 2: Using Constants for Precision
```solidity
uint256 constant PRECISION = 1e18;  // 18 decimals

function calculatePercentage(uint256 amount, uint256 percent) 
    public pure returns (uint256) 
{
    // percent = 5000 means 50% (5000/10000)
    return (amount * percent) / 10000;
}

// Example: 100 ETH * 50% = 50 ETH
// calculatePercentage(100 ether, 5000) = 50 ether
```

### Method 3: Libraries for Precise Math
```solidity
// Using PRBMath library for advanced fixed-point math
import {PRBMathUD60x18} from "prb-math/PRBMathUD60x18.sol";

uint256 result = PRBMathUD60x18.mul(1.5e18, 2.3e18);
// Multiplies 1.5 * 2.3 = 3.45 (in 18-decimal format)
```

***

## When to Use Each Integer Size

| If your maximum value is... | Use | Example |
|------------------------------|-----|---------|
| < 256 | `uint8` | Age, small flags [2][6] |
| < 65,536 | `uint16` | Year, basis points [7] |
| < 4.3 billion | `uint32` | Timestamps, IDs [2][3] |
| < 18 quintillion | `uint64` | Millisecond timestamps [2] |
| Anything larger or financial | `uint256` | ETH, tokens, balances [3][4] |

***

## Your Cheat Sheet for Next Project

**Storage struct optimization:**
```solidity
struct Optimized {
    // Pack small types first
    address owner;      // 20 bytes
    uint32 timestamp;   // 4 bytes
    uint16 count;       // 2 bytes
    uint8 status;       // 1 byte
    bool active;        // 1 byte
    // ↑ All fit in 1 slot (28 bytes)!
    
    // Then uint256
    uint256 balance;
    
    // Dynamic types last
    string name;
    mapping(address => uint256) data;
}
```

**Loop counters: Always `uint256`**[4]

**Financial amounts: Always `uint256`**[3][4]

**Decimals: Multiply by 10^18 and use integers**[10]
