# LendingMarket

Single-collateral money market. Suppliers deposit the base asset and earn interest, borrowers post collateral and draw the base asset against it, and positions that fall below their borrowing power can be liquidated.

## Running it

The project is self-contained. `forge-std` is vendored under `lib/`, so no network access or `forge install` is needed.

```bash
forge build
forge test
forge test -vvv --match-test <name>     # a single test, with traces
```

Requires Foundry. If you do not have it: `curl -L https://foundry.paradigm.xyz | bash && foundryup`

## Layout

```
src/
  LendingMarket.sol                 the implementation, and the only file you need to change
  TransparentUpgradeableProxy.sol   the proxy the market is deployed behind
  interfaces/                       IERC20, IPriceOracle (Chainlink-style aggregator)
  mocks/                            test doubles for the base asset, collateral and oracle
script/
  Deploy.s.sol                      staging deployment, mirrors the production topology
test/
  LendingMarket.t.sol               the suite the market ships with today
```

## How it works

**Accounting.** Supplier and borrower balances are stored as *principal* and scaled by two global indices, `baseSupplyIndex` and `baseBorrowIndex`. `accrueInterest()` advances both to the current timestamp. Present value is `principal * index / 1e18`, so a balance grows without anyone touching per-account storage.

**Scaling.** Everything is 1e18 fixed point unless stated otherwise. The oracle quotes the collateral in base-asset terms, also 1e18. So with WETH at 2,000 base, `getPrice()` returns `2000e18`.

**Borrowing power.** `collateralBalance * price * collateralFactor`, all 1e18 scaled. A position is healthy while its borrowing power covers its debt. `isHealthy()` is the single source of truth for this, and it is checked on any path that increases risk.

**Liquidation.** Once a position is unhealthy, anyone may repay part of its debt and receive collateral in exchange, plus the liquidation incentive. The incentive is 1e18 scaled and always at or above par, so `1.10e18` means the liquidator receives 110% of what they repaid, valued in collateral.

**Deployment.** The market is an implementation behind a `TransparentUpgradeableProxy`. All market state lives in the proxy's storage, which is why any change you make has to be layout-compatible with what is already deployed. The admin owns upgrades and risk parameters. A separate pause guardian exists so the market can be stopped quickly without reaching for the admin key.

## Roles

| Role | Held by | Remit |
|---|---|---|
| `admin` | governance timelock | upgrades, risk parameters, wiring |
| `pauseGuardian` | operations multisig | stopping the market in an incident |

The split matters. The guardian key is warmer and held by more people than the admin key, so anything the guardian can reach should be limited to stopping the market, not changing how it prices or values anything.

## Listed markets

See `script/Deploy.s.sol` for the current staging configuration.

## Review findings

Five defects are fixed in `src/LendingMarket.sol`. Each has a regression test in `test/LendingMarket.t.sol` that fails on the original code and passes on the fix.

**1. Liquidation seizes collateral without applying the price. Critical.**
`liquidate` computed the seize amount as `repayAmount * liquidationIncentive / 1e18`. That is a base amount, but it was taken out of the collateral balance one to one. With WETH at 1,800, a liquidator who repaid 1,000 base received all 10 WETH of the borrower, worth 18,000, and left 14,000 of debt with nothing behind it. The seize amount is now `repayAmount * liquidationIncentive / price`. Test: `test_liquidationSeizeIsPricedInCollateral`.

**2. The pause guardian can replace the oracle. High.**
`setOracle` was `onlyGuardian`. The guardian key is the warmer one, and whoever holds it could point the market at an oracle they control, then borrow the pool against dust or make every position liquidatable. `setOracle` is now `onlyAdmin`, which matches the role split described above. Test: `test_guardianCannotSetOracle`.

**3. Stale oracle prices are accepted. High.**
`getPrice` ignored `updatedAt`. If the feed stalled, the market kept lending and liquidating at the last price it saw. `getPrice` now reverts when the answer is older than `MAX_PRICE_AGE`, set to one hour. While the feed is stale this blocks `borrow`, `liquidate`, and `withdrawCollateral` for accounts with debt. `supply`, `withdraw` and `repay` keep working. Test: `test_stalePriceBlocksBorrow`.

**4. `withdrawCollateral` checks health against old debt. Medium.**
It did not call `accrueInterest()` before `isHealthy`, so the check used the borrow index from the last interaction with the market. A borrower close to the limit could withdraw collateral that the accrued interest should have locked. It now accrues first. Test: `test_withdrawCollateralAccruesInterestFirst`.

**5. Principal conversions round in favour of the user. Low.**
`borrow` and `withdraw` converted the amount to principal rounding down. Once an index is above 1.0, `borrow(1)` recorded zero debt and `withdraw(1)` burned zero principal, so anyone could take one wei per call with no collateral and no deposit. With the 18 decimal base asset in the staging deployment this is not profitable. It becomes profitable when one base unit is worth more than the gas of a looped call, which is the case for a base asset with few decimals and a high unit price on a chain with cheap gas. I have seen this class of rounding issue cost real money in production, so I do not leave the direction wrong. Both paths now round the principal up. Tests: `test_borrowRoundsPrincipalUp`, `test_withdrawRoundsPrincipalUp`.

### Shipping this to the live proxy

All five changes are logic only. No storage variable was added, removed, reordered or retyped, and `MAX_PRICE_AGE` is a constant, so it lives in the bytecode. The output of `forge inspect LendingMarket storage-layout` is identical before and after. External function signatures are unchanged. The new implementation can go behind the existing proxy with a plain `upgradeTo` and needs no migration call.

An upgrade fixes the code, not the state. If the guardian already swapped the oracle before the upgrade, the admin has to set it back afterwards.

### Found but not fixed

- `getPrice` casts the signed oracle answer to `uint256` without checking that it is positive. A zero or negative answer would misprice every position. The fix is `require(answer > 0)` and a test.
- When the seize amount is capped at the collateral the borrower has, `repayAmount` is not scaled down, so the liquidator overpays.

## Scope note

- I went first for anything that moves value at a wrong price or lets the wrong role set the price: liquidation math, oracle control, oracle freshness, and the health check on stale debt. I left out the two items listed above.
- I am unsure about the one hour staleness window. In production I would derive the feed heartbeat on chain from the gaps between recent rounds, as I built for WiseLending, and store it in a variable appended to the layout.
- With more time I would add a test that deploys the original implementation behind the proxy, builds state, upgrades, and checks that every balance survives.
- I used Claude Code. I found the `setOracle` and `withdrawCollateral` issues reading the contract myself. It pointed me at the liquidation math and the oracle handling, and I checked both against the code before fixing them. It suggested leaving the rounding direction out as dust. I rejected that and fixed it.
