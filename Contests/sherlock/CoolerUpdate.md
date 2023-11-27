![](https://audits.sherlock.xyz/_next/image?url=https%3A%2F%2Fsherlock-files.ams3.digitaloceanspaces.com%2Fcontests%2Fcooler.jpg&w=96&q=75)

# [Cooler Update](https://audits.sherlock.xyz/contests/107)

| Protocol | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|
| Cooler Update | 11,500 USDC | 512 | 3 days | Aug 25 2023 | Aug 28 2023 |

## Issues found :

| Severity | Title |
|:--|:--:|
| High | Lender can lose funds after lending debt token. |

# 1. Lender can lose funds after lending debt token.
# Summary
Borrower can request a loan for particular amount of time and debt tokens he wants to borrow also collateral is added in advance. Therefore the borrower has time to repay the borrowed debt before the loan expires, or a lender defaults the loan and gets borrower's collateral

# Vulnerability Detail
One of the new features implemented is the manual claiming of accrued debt for a lender.

```
 /// @notice Claim debt tokens if repayDirect was false.
    /// @param  loanID_ index of loan in loans[].
    function claimRepaid(uint256 loanID_) external {
        Loan memory loan = loans[loanID_];

        // Update storage.
        uint256 claim = loan.unclaimed;
        delete loans[loanID_].unclaimed;

        // Transfer repaid debt back to the lender.
        debt().safeTransfer(loan.lender, claim);
    }
```

Thus when repayDirect is set to false then whenever a borrower repays a loan, debt will be added to loan.unclaimed

```
 // Check whether repayment needs to be manually claimed or not.
        if (loan.repayDirect) {
            repayTo = loan.lender;
        } else {
            repayTo = address(this);
            loan.unclaimed += repaid_;
        }
```

Malicious borrower can cause loss of funds for the lender. Scenario:

Lender has set repayDirect to false so he can manually pick up his accrued debt.
He lends some debt tokens to a borrower.
Malicious borrower waits for the block.timestamp to reach near the loan.expiry in order to repay loan last second before it expires.
Now since the loan is repaid the debt tokens which the borrower had to return to the lender are added into the loan.unclaimed for this particular loanID
Thus the lender has to claim his debt by calling claimRepaid()

```
/// @notice Claim debt tokens if repayDirect was false.
   /// @param  loanID_ index of loan in loans[].
   function claimRepaid(uint256 loanID_) external {
       Loan memory loan = loans[loanID_];

       // Update storage.
       uint256 claim = loan.unclaimed;
       delete loans[loanID_].unclaimed;

       // Transfer repaid debt back to the lender.
       debt().safeTransfer(loan.lender, claim);
   }
```

Now since the loan is expired and everything is paid back, malicious borrower can call claimDefaulted() BEFORE the lender claims his debt with claimRepaid() and delete this particular loanID without the lender being able to claim his debt.


```
function claimDefaulted(uint256 loanID_) external returns (uint256, uint256, uint256) {
        Loan memory loan = loans[loanID_];
        
        delete loans[loanID_];

        if (block.timestamp <= loan.expiry) revert NoDefault();

        // Transfer defaulted collateral to the lender. 
        //@audit nothing will be transferred since the borrower has paid back the loan                                                                                    
        collateral().safeTransfer(loan.lender, loan.collateral);

        // Log the event.
        factory().newEvent(loanID_, CoolerFactory.Events.DefaultLoan, 0);

        // If necessary, trigger lender callback.
        if (loan.callback) CoolerCallback(loan.lender).onDefault(loanID_, loan.amount, loan.collateral);
        return (loan.amount, loan.collateral, block.timestamp - loan.expiry);
    }
```

# Impact
Malicious borrower can cause loss of debt tokens for a lender

# Tool used
Manual Review

# Recommendation
Since you have added an option for a lender to be able to claim his debt manually, make sure before defaulting and deleting a loan to check if there is any amount left in the loan.unclaimed.
