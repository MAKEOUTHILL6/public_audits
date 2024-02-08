![](https://audits.sherlock.xyz/_next/image?url=https%3A%2F%2Fsherlock-files.ams3.digitaloceanspaces.com%2Fcontests%2Freal-wagmi.jpg&w=256&q=75)

# [RealWagmi #2](https://audits.sherlock.xyz/contests/118)

| Protocol | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|
| RealWagmi #2 | 42,500 USDC | 1,283 | 7 days | 16 Oct 2023 | 23 Oct 2023 |

## Issues found :

| Severity | Title |
|:--|:--:|
| High | Whenever a user wants to takeOverDebt will never work |

# 1. Whenever a user wants to takeOverDebt will never work.

# Summary
In LiquidityBorrowingManager.sol a user can takeOverDebt for a specific borrower by providing the borrower's borrowingKey:

```
function takeOverDebt(bytes32 borrowingKey, uint256 collateralAmt)
```

# Vulnerability Detail

In order for a user to successfully take over a borrower's debt, he has to provide :

```
(collateralAmt <= minPayment).revertError(
                ErrLib.ErrorCode.COLLATERAL_AMOUNT_IS_NOT_ENOUGH
            );
```

collateralAmt is the amount of collateral to be provided by the new borrower.
minPayment is the minimum payment required based on the collateral balance for the old borrower.

Then loans and keys are removed from the old borrower

```
_removeKeysAndClearStorage(oldBorrowing.borrower, borrowingKey, oldLoans);
```

After that the

```
(uint256 feesDebt, bytes32 newBorrowingKey, BorrowingInfo storage newBorrowing) = _initOrUpdateBorrowing(
               oldBorrowing.saleToken,
               oldBorrowing.holdToken,
               accLoanRatePerSeconds
           );
```

is called which returns the msg.sender's (new borrower) bytes32 newBorrowingKey then:

```
// Add the new borrowing key and old loans to the newBorrowing
_addKeysAndLoansInfo(newBorrowing.borrowedAmount > 0, borrowingKey, oldLoans);
```

The problem is that the old loans and the initialization of the borrower are added again to the OLD borrower because borrowingKey is used in _addKeysAndLoansInfo rather than the new borrower's newBorrowingKey.

# Impact
User can't take over another borrower's debt.

# Tool used
Manual Review

# Recommendation
- `_addKeysAndLoansInfo(newBorrowing.borrowedAmount > 0, borrowingKey, oldLoans);`
++`_addKeysAndLoansInfo(newBorrowing.borrowedAmount > 0, newBorrowingKey, oldLoans);`
