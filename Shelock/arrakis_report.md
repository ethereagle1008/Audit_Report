# Title1: Operator can hijack rebalance control flow inside swap.router.call to manipulate pool prices and sandwich attack subsequent liquidity minting

## Summary
Malicious operator can supply crafted swap payload to rebalance which allows him to hijack control flow inside swap.router.call.
As a result, the operator can manipulate pool prices, effectively bypassing the deviation check that is performed earlier within SimpleManager. Exploiting this vulnerability, the operator can sandwich attack subsequent liquidity minting to steal the protocol.

## Vulnerability Detail
The malicious operator can do the following steps to launch the attack.

### Hijack control flow in uniswap router call.
The malicious operator first need to deploy a malicious ERC20 token and make a uniswap liquidity pool. Uniswap v3 router is Multicall, which means swap payload can both have legitimate swaps and malicious swaps that swap this malicious ERC20 token.

Passing a crafted swap payload to rebalance will make the rebalance call into the malicious token contract during swap.

Frontrun liquidity minting in the malicious contract
The operator now has the ability to manipulate pool prices inside his malicious contract.

The deviation check inside SimpleManager is bypassed at this stage. He need to bypass subsequent liquidity minting slippage checks inside ArrakisV2.

Inside ArrakisV2::rebalance, the slippage checks are applied to the aggregated results. A sandwich attacker can bypass these checks by pushing up the price of one pool and reducing the price of another. This will counteract the two slippage effects and correct the final aggregated results.

```solidity
    (uint256 amt0, uint256 amt1) = IUniswapV3Pool(pool).mint(
        address(this),
        rebalanceParams_.mints[i].range.lowerTick,
        rebalanceParams_.mints[i].range.upperTick,
        rebalanceParams_.mints[i].liquidity,
        ""
    );
    aggregator0 += amt0;
    aggregator1 += amt1;
}
require(aggregator0 >= rebalanceParams_.minDeposit0, "D0");
require(aggregator1 >= rebalanceParams_.minDeposit1, "D1");
```

### Backrun to finalize the sandwich attack
The slippage protection is bypassed and the rebalance transaction will success.

Now the operator can backrun the rebalance transaction to finalize the sandwich attack to profit.

## Impact
The malicious operator can sandwich liquidity minting to steal the protocol.

Severity set to High because sandwich liquidity adding will definitely make a profit(a case study) and the bypass of slippage protection will enlarge the profit.

## Code Snippet
https://github.com/sherlock-audit/2023-06-arrakis/blob/main/v2-core/contracts/ArrakisV2.sol#L334-L337

https://github.com/sherlock-audit/2023-06-arrakis/blob/main/v2-core/contracts/ArrakisV2.sol#L398-L409

## Tool used
Manual Review

## Recommendation
I have three directions to fix this vulnerability but non of them are straightforward fixes.

1. Check the swap payload to make sure that it only swaps on the target pools.
2. Reinforcing slippage checks on liquidity minting. That is enforcing deviation check inside ArrakisV2 or checking slippage of each liqudity minting instead of checking the aggregated results.
3. Seperate the three operations(burn, swap and mint) in ArrakisV2::rebalance into three function calls and make checks around these three calls inside SimpleManager.


# Title 2: Operator rebalance should be rate limited

## Summary
`SimpleManager::rebalance` includes slippage protection during swaps to prevent sandwich attacks. Operator still can gain minimal profits from this slippage with sandwich attack. Since `rebalance` is not rate limited, operator can repeat the rebalacne and sandwich attack to steal the protocol.

## Vulnerability Detail
The slippage of swap in `SimpleManager::rebalance` determines the maximum profit a sandwich bot can obtain through such swaps.

While the profits obtained through a single `rebalance` operation in a sandwich attack are minimal due to this slippage protection, the operator can repeatedly perform the attack to accumulate profit.

## Impact
Operator can steal the protocol with repeated sandwich attacks.

## Code Snippet
Slippage checking inside `rebalance`:

https://github.com/sherlock-audit/2023-06-arrakis/blob/main/v2-manager-templates/contracts/SimpleManager.sol#L205

## Tool used
Manual Review

## Recommendation
Implement a rate limit on the `rebalance` function.
