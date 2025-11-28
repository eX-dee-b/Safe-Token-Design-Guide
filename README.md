# Designing Safe Tokens: A Deep Guide to Economic Invariants and Permission Boundaries

<p align="center">
  <img src="https://img.shields.io/badge/Language-Solidity-363636?style=for-the-badge&logo=solidity">
  <img src="https://img.shields.io/badge/Focus-Economic%20Security-red?style=for-the-badge">
  <img src="https://img.shields.io/badge/EVM-Compatible-blue?style=for-the-badge&logo=ethereum">
  <img src="https://img.shields.io/badge/Category-Token%20Design-green?style=for-the-badge">
  <img src="https://img.shields.io/badge/License-MIT-orange?style=for-the-badge">
  <img src="https://img.shields.io/badge/Status-Research--Grade-purple?style=for-the-badge">
</p>
 
## Why This Matters (More Than People Think)

Most people still design tokens like this:

> “Just fork an ERC-20, add some tax logic, maybe a blacklist, maybe a mint function, ship.”

Then they’re surprised when:

* a “trusted” admin mints infinite tokens
* fees get changed to 100% and nobody can exit
* users are selectively blacklisted
* a manual oracle changes price and liquidations explode
* upgrades introduce a backdoor

All *without* a single “classic” bug like reentrancy.

**Economic design errors** are now responsible for a huge fraction of *real* loss in DeFi.

So if you want to design tokens that are genuinely **safe**, you need two big concepts:

1. **Economic invariants**, what *must always hold* about supply, fees, liquidity, etc.
2. **Permission boundaries**, *who* is allowed to touch those invariants, under *what constraints*.

We’re going to build a mental and practical framework for both.

---

## 1. Economic Invariants: “Laws of Motion” for Tokens

Think of an economic invariant as a law like:

> “No matter what happens, **X** must never exceed **Y**.”

If you violate that, the system is no longer safe.

### 1.1 Types of Economic Invariants

Let’s classify the main invariants for ERC-20-style tokens.

1. **Supply Invariants**

   * Total supply cannot exceed a hard cap.
   * Minting can only happen under specific, transparent rules.
   * Burning must obey clear rules (no arbitrary forced burns).

2. **Fee/Tax Invariants**

   * Fees must never exceed a given ceiling (e.g., 5%).
   * Fees must be applied symmetrically and consistently.
   * Fee recipients must be known and constrained.

3. **Liquidity/Transfer Invariants**

   * Users must be able to freely transfer tokens (no silent honeypot).
   * Trading cannot be permanently disabled after launch.
   * Blacklist/whitelist powers must be extremely limited or absent.

4. **Control/Ownership Invariants**

   * No single EOA should have absolute control over monetary policy.
   * Critical changes must go through governance/timelock/multisig.
   * Ownership should be renounced when appropriate.

5. **Pricing/Oracle Invariants**

   * Prices must not be manually set by an admin.
   * Oracles must follow sound principles: aggregation, bounds, delay checks.

6. **Upgrade Invariants (for proxies)**

   * Upgrades must not change fundamental economic guarantees.
   * Admins must not gain new hidden powers.
   * Users must have predictability about future changes.

Each of these lines up with a *threat model*.

---

## 2. Permission Boundaries: The Real Attack Surface

An “invariant” is meaningless if permissions allow anyone to override it.

Permission boundaries define **who** can:

* mint
* burn
* change fees
* block transfers
* set price
* upgrade logic

### 2.1 Roles in a Typical Token

Common roles:

* `owner` / `admin`, often too powerful
* `governance`, DAO or multisig
* `minter`, allowed to create supply under strict caps
* `pauser`, allowed to temporarily pause certain functions
* `oracleUpdater`, allowed to update price, under rules

In a safe design, you want:

* fewer roles
* more constraints
* smaller blast radius when a role is compromised

---

## 3. Threat Models: “How Could This Be Abused?”

Before we talk “how to design safely”, let’s be explicit about **how tokens get abused**.

### 3.1 Common Real-World Economic Attack Patterns

1. **Slow Rug via Minting**

Admin slowly mints extra supply and dumps it over weeks/months.
No “hack”, just abuse of permission.

2. **Fee Flip Honeypot**

* Initially: low or zero fees → attracts buyers.
* Later: admin sets `sellTax = 100%` → nobody can exit.

3. **Blacklist Trap**

* Token allows buying from DEX.
* When liquidity is deep, blacklist selling addresses, or blacklist everyone except dev.

4. **Manual Oracle Massacre**

* Lending or AMM relies on `getPrice()` from Oracle contract.
* Admin calls `setPrice()` and manipulates liquidations or swap rates.

5. **Upgrade Backdoor**

* Proxy pattern with `upgradeTo()` controlled by single admin.
* Admin points implementation to malicious contract that drains funds.

None of these require a technical vulnerability; they’re **by design**.

Your goal as a safe designer: **make those impossible by construction.**

---

## 4. Designing Supply Invariants (Monetary Policy)

### 4.1 Hard Caps vs Elastic Supply

You must decide up front:

* Is your token **fixed supply** (like BTC)?
* **Capped supply** with controlled emissions?
* **Elastic supply** (rebase tokens)?

Each has different invariants.

### 4.2 Fixed or Capped Supply

**Invariant example:**

> `totalSupply <= MAX_SUPPLY` at all times.

**Safe pattern:**

```solidity
uint256 public immutable MAX_SUPPLY = 1_000_000 ether;
uint256 public totalSupply;

function _mint(address to, uint256 amount) internal {
    require(totalSupply + amount <= MAX_SUPPLY, "cap exceeded");
    totalSupply += amount;
    balanceOf[to] += amount;
}
```

Key idea: **you literally cannot violate the cap** without changing code.

**Anti-pattern:**

```solidity
function mint(address to, uint256 amount) external onlyOwner {
    totalSupply += amount;        // no cap
    balanceOf[to] += amount;
}
```

Even if the current owner is “trusted”, this design is dangerous because:

* keys get compromised
* teams change
* trust evaporates

---

### 4.3 Role-Bound Minting

Instead of a single `onlyOwner`:

```solidity
bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");

function mint(address to, uint256 amount) external onlyRole(MINTER_ROLE) {
    _mint(to, amount);
}
```

Now you can:

* have multiple minters
* revoke minters
* track role assignments

But please still enforce a cap or a time/emission schedule.

---

## 5. Designing Fee & Tax Invariants

Fees are extremely dangerous because they can silently trap value.

### 5.1 Hard Fee Caps

**Invariant example:**

> `feeBps <= 500` (5%) always.

**Safe pattern:**

```solidity
uint256 public constant MAX_FEE_BPS = 500;
uint256 public feeBps; // in basis points

function setFee(uint256 newFeeBps) external onlyOwner {
    require(newFeeBps <= MAX_FEE_BPS, "fee too high");
    feeBps = newFeeBps;
}
```

Note: now *even if the owner is malicious*, they can’t set fee to 100%.

### 5.2 No Hidden Dynamic Multipliers

Unsafe example:

```solidity
uint256 public feeBps;
uint256 public feeMultiplier; // <= this is the trap

function _transfer(address from, address to, uint256 amount) internal {
    uint256 fee = amount * feeBps * feeMultiplier / 10000;
    ...
}
```

Even if `feeBps` looks safe, `feeMultiplier` might be set to a huge number later.

**Rule of thumb:**

> If you want safe economics, keep fee logic *simple and transparent*.

---

## 6. Designing Liquidity and Transfer Invariants

This is where honeypots live.

### 6.1 Safe vs Dangerous Transfer Restrictions

**Dangerous pattern:**

```solidity
mapping(address => bool) public blacklist;

function transfer(address to, uint256 amount) external {
    require(!blacklist[msg.sender], "blacklisted");
    ...
}
```

Used for:

* freezing users
* blocking sells when convenient

A “safer” version if you *must* have blacklists:

* only active before a fixed `LAUNCH_BLOCK`
* only for known malicious contracts
* transparent list + governance-controlled

But in general, the safest invariant is:

> Anyone can transfer at any time, unless the entire contract is in an emergency pause with clear, narrow semantics.

### 6.2 Trading Enable Switches

Another classic pattern:

```solidity
bool public tradingEnabled;

function setTradingEnabled(bool b) external onlyOwner {
    tradingEnabled = b;
}

function transfer(address to, uint256 amount) external {
    require(tradingEnabled || msg.sender == owner, "trading not live");
    ...
}
```

This is fine if:

* it’s only used for a short bootstrap phase
* after launch, you **permanently** set `tradingEnabled = true` and maybe remove the setter

Otherwise it becomes a **permanent rug lever**.

**Safer pattern:**

```solidity
bool public immutable LAUNCH_MODE;
uint256 public immutable LAUNCH_END_BLOCK;

constructor() {
    LAUNCH_MODE = true;
    LAUNCH_END_BLOCK = block.number + 100; // e.g. initial 100 blocks
}

function transfer(...) external {
    if (block.number <= LAUNCH_END_BLOCK) {
        // optional special behavior
    }
    // normal behavior after
}
```

Or: no trading flags at all.

---

## 7. Designing Control / Ownership Invariants

Core rule:

> **No single externally owned account should be able to destroy the economy.**

Bad pattern:

* `owner` can mint
* `owner` can burn others
* `owner` can set fees
* `owner` can upgrade contract
* `owner` can pause transfers
* `owner` is a single EOA in MetaMask

That’s a walking rug.

### 7.1 Better Pattern: Multisig + Timelock + Renouncement

* Use a multisig (e.g. Gnosis Safe) as `owner`.
* For sensitive operations (upgrade, mint, fee change), require:

  * timelock delay
  * on-chain proposal
  * clear event emission

### 7.2 Eventually: Renounce Ownership

If your token is meant to be fully trustless (like a vanilla ERC-20), you can:

* deploy
* distribute supply
* renounce ownership

After that:

* no one can change fee
* no one can mint
* no one can rug

That’s the *ultimate* permission boundary.

---

## 8. Designing Oracle & Pricing Invariants

Oracles are dangerous because they control downstream economic logic.

### 8.1 Never Let an Owner Set Price

Bad design:

```solidity
int256 public price;

function setPrice(int256 p) external onlyOwner {
    price = p;
}
```

If this price feeds into a lending protocol, you’ve built a God-mode liquidation tool.

### 8.2 Use Real Oracles

Better:

* Chainlink Aggregator
* Uniswap TWAP oracle
* Multi-oracle median

And still:

* check that price is within sane bounds
* avoid stale data
* don’t give direct write access to a single EOA

---

## 9. Designing Upgrade Safety

Proxy patterns are useful but deadly when misused.

### 9.1 Typical Dangerous Proxy

```solidity
address public implementation;
address public admin;

function upgradeTo(address newImpl) external {
    require(msg.sender == admin);
    implementation = newImpl;
}
```

This means:

* admin can deploy malicious implementation
* admin can drain everything tomorrow

### 9.2 Safer Upgrades

* Use a timelock
* Use a DAO or multisig
* Use upgrade governance proposals
* Explicitly commit to invariant-preserving upgrades only

Some protocols even:

* finalize (lock) implementation after launch
* or split token logic from system logic such that user balances cannot be arbitrarily modified.

---

## 10. Practical Design Blueprint: From Zero to “Safe Enough”

Here’s a step-by-step process to design a **simple but safe** ERC-20.

### Step 1, Define Your Economic Intent

Answer clearly:

* Is supply fixed or capped?
* Are there any fees? If yes, why?
* Is there any need for blacklist/whitelist logic?
* Who should control what, and for how long?
* When should ownership be renounced (if ever)?

### Step 2, Write Down Your Invariants in Plain English

Example:

* “Total supply will never exceed 1M tokens.”
* “Fees will never exceed 2%.”
* “No user will ever be prevented from transferring tokens.”
* “No admin will ever be able to change the price.”
* “After launch, no entity can mint more tokens.”

### Step 3, Turn Those Into Solidity Checks

* Hard-code caps
* Add `require(fee <= MAX_FEE)`
* Avoid blacklist logic entirely
* Avoid manual oracles
* Remove mint function after initial distribution

### Step 4, Minimize Permissions

* Use roles only where necessary
* Use multisig, not EOA
* Remove critical powers once they’re no longer needed

### Step 5, Verify with an Economic Scanner

* Run your own tool (Solidity Economic Risk Scanner)
* See if it flags:

  * unbounded mint
  * manual price setters
  * complex fee + tax patterns
  * blacklist/honeypot indicators
  * excessive owner privileges

If it does → redesign until it doesn’t.

---

## 11. Applying This to Different Token Types

Safe design depends on token type.

### 11.1 Governance Token

Desiderata:

* Transparent supply schedule
* Controlled inflation via governance
* Minimal owner-only powers
* No hidden fee/blacklist logic

### 11.2 Utility Token

* Often no need for dynamic minting at all
* Keep fees simple or absent
* Avoid any blacklist or trading gate logic

### 11.3 LP / Staking Token

* Bound claims to underlying assets
* Ensure you can’t inflate supply without backing
* Avoid arbitrary redemption changes

### 11.4 Stablecoin

* Strongest invariants needed
* Collateralization and redemption rules must be transparent
* Oracles must be robust
* Governance must be heavily constrained

---

## 12. How to Read a Token Contract Like an Economic Auditor

Whenever you open a Solidity file, do this mental checklist:

1. **Search for “mint”**

   * Who can call it?
   * Is there a cap?

2. **Search for “burn”**

   * Can the contract burn arbitrary user funds?

3. **Search for “fee”, “tax”, `_transfer`**

   * Are there dynamic fees?
   * Are there adjustable parameters?

4. **Search for “blacklist”, “whitelist”, “cooldown”, “antibot”**

   * Is trading selectively gateable?

5. **Search for “setPrice” or oracles**

   * Can admin write the price?

6. **Search for “upgrade”, “implementation”, “proxy”**

   * Who controls upgrades?

7. **Search for “onlyOwner” / modifiers**

   * How many functions depend on owner?
   * Are these functions economically critical?

You can formalize this in code (as you already started doing with your scanner), but this is also a great manual review habit.

---

## 13. Conclusion: Tokens as Economic Machines

The core idea:

> A token is not just an interface, it is a **small monetary system**.

If you treat it as a trivial ERC-20 wrapper, you leave hidden powers that someone *will* eventually exploit, whether maliciously or by accident.

**Designing safe tokens means:**

* Defining economic invariants up front
* Constraining them in code
* Minimizing and decentralizing permissions
* Avoiding dangerous patterns (unbounded mint, manual price, blacklist traps)
* Using tools (like your scanner) to enforce these properties at scale

If you get this right, your tokens won’t just be “non-hackable”,
they’ll be **structurally robust economic instruments.**
