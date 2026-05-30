# WePiggy BSC — Oracle Failure Research

> **Status:** Responsible disclosure in progress — full details will be published after remediation or 90-day disclosure deadline.

## Summary

Independent security research identifying a live High severity vulnerability in the [WePiggy Protocol](https://wepiggy.com) BSC deployment. The pBNB price oracle is completely non-functional on mainnet, making all BNB-collateral liquidations impossible.

## Finding

| Property | Detail |
|---|---|
| Protocol | WePiggy — Open Source Lending Protocol |
| Chain | BNB Smart Chain (BSC / Chain ID: 56) |
| Severity | **High** |
| Status | Live on mainnet — unresolved |
| Affected Market | pBNB (`0x33A32f0ad4AA704e28C93eD8Ffa61d50d51622a7`) |
| Oracle Contract | `0x4C78015679FabE22F6e02Ce8102AFbF7d93794eA` |
| Duration | Broken for 90+ days at time of discovery |
| Discovered | May 31, 2026 |

## Vulnerability

The `WePiggyPriceProviderV1` oracle contract loops through configured price sources and reverts if none return a valid price:

```solidity
for (uint256 i = 0; i < priceOracles.length; i++) {
    if (priceOracle.available == true) {
        price = _getUnderlyingPriceInternal(_pToken, tokenConfig, priceOracle);
        if (price > 0) {
            return price;
        }
    }
}
require(price > 0, "price must bigger than zero"); // reverts if nothing valid
```

For the pBNB market, **both configured sources fail:**

| Index | Address | Type | Available | Result |
|---|---|---|---|---|
| 0 | `0x0567F2323251f0Aab15c8dFb1967E4e8A7D42aeE` | Chainlink | `false` ❌ | Skipped |
| 1 | `0xFfceAcfD39117030314A07b2C86dA36E51787948` | Customer | `true` | Returns `0` ❌ |

**Result:** Every call to `getUnderlyingPrice(pBNB)` reverts on mainnet.

## Proof of Concept

Reproducible in a browser — no tools or wallet required:

1. Go to [oracle contract on BscScan](https://bscscan.com/address/0x4C78015679FabE22F6e02Ce8102AFbF7d93794eA#readContract)
2. Call `oracles(0x33A32f0...622a7, 0)` → `available = false`
3. Call `oracles(0x33A32f0...622a7, 1)` → `available = true`, source = custom oracle
4. Go to [custom oracle](https://bscscan.com/address/0xFfceAcfD39117030314A07b2C86dA36E51787948#readProxyContract)
5. Call `getPrice(0xbb4CdB9CBd36B01bD1cBaEBF2De08d9173bc095c)` → `0`
6. Back to main oracle → call `getUnderlyingPrice(0x33A32f0...622a7)` → **REVERT**

**Confirmed revert output:**
```
execution reverted: price must bigger than zero
0x08c379a0...7072696365206d75737420626967676572207468616e207a65726f
```

## Foundry Test

Fork-based proof of concept test suite with 6 targeted tests:

```bash
# Install Foundry
curl -L https://foundry.paradigm.xyz | bash && foundryup

# Clone and run
git clone https://github.com/[your-username]/wepiggy-oracle-research
cd wepiggy-oracle-research
forge install foundry-rs/forge-std --no-commit
forge test --fork-url https://bsc-dataseed.binance.org -vvv
```

**Expected output — all 6 tests pass (passing = revert confirmed):**

```
[PASS] test_OracleSourceConfiguration()
[PASS] test_CustomOracleReturnsZeroForBNB()
[PASS] test_GetUnderlyingPriceReverts()
[PASS] test_LiquidationBrokenDueToOracleFailure()
[PASS] test_AccountLiquidityBrokenForBNBCollateral()
[PASS] test_AttackScenarioDescription()
```

## Impact

- `getUnderlyingPrice(pBNB)` reverts on every call — permanent until manually fixed
- The Comptroller cannot calculate account liquidity for any BNB-collateral position
- All BNB-collateral liquidations are impossible regardless of health factor
- Undercollateralized positions accumulate bad debt with no recovery mechanism
- ~$49,726 BSC TVL exposed at time of discovery

## Contributing Issues

| Issue | Detail |
|---|---|
| No staleness check | `latestRoundData()` called but `updatedAt` never validated |
| Single EOA reporter | Custom oracle price set by one wallet, inactive 90+ days |
| No timelock on oracle proxy | `upgradeTo()` callable instantly with no governance delay |
| No circuit breaker | No market pause triggered when oracle returns 0 |

## Disclosure Timeline

| Date | Action |
|---|---|
| May 31, 2026 | Vulnerability discovered through manual on-chain investigation |
| May 31, 2026 | On-chain revert confirmed via BscScan read contract |
| May 31, 2026 | Full report and Foundry PoC written |
| May 31, 2026 | Responsible disclosure attempted via WePiggy Discord and email |
| Pending | Awaiting WePiggy team response |
| TBD | Full public disclosure after remediation or 90-day deadline |

## Repository Structure

```
wepiggy-oracle-research/
├── README.md                          — this file
├── foundry.toml                       — Foundry project config
├── test/
│   └── WePiggyOraclePoC.t.sol        — 6-test PoC suite
└── report/
    └── WePiggy_pBNB_Oracle_Bug_Bounty_Report.docx
```

## Methodology

All research was conducted **read-only** — no transactions were sent, no funds were moved, and no mainnet state was modified. Tools used:

- BscScan Read Contract interface
- Manual Solidity source code review
- Foundry fork testing (local only)

## Researcher

**[Your Name]** — Independent security researcher  
Pakistan 🇵🇰  
Contact: [your email or Telegram]

---

> This repository is for educational and responsible disclosure purposes only.  
> Do not use this research to exploit live protocols.
