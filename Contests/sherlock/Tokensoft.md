![](https://audits.sherlock.xyz/_next/image?url=https%3A%2F%2Fsherlock-files.ams3.digitaloceanspaces.com%2Fcontests%2Ftokensoft.jpg&w=96&q=75)

# [Tokensoft](https://audits.sherlock.xyz/contests/100)

| Protocol | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|
| Tokensoft | 21,000 USDC | 650 | 4  days | Jul 17 2023 | Jul 21 2023 |

## Issues found :

| Severity | Title |
|:--|:--:|
| Medium | Loss of funds during user adjusting. |

# 1. Loss of funds during user adjusting.
# Summary
Adjusting a user's total claimable value not working correctly

# Vulnerability Detail
Whenever the owner is adjusting user's total claimable value, the `records[beneficiary].total` is decreased or increased by `uint256 diff = uint256(amount > 0 ? amount : -amount);`.

However some assumptions made are not correct. Scenario:

User has bought 200 FOO tokens for example.
In `PriceTierVestingSale_2_0.sol` he calls the `initializeDistributionRecord` which sets his `records[beneficiary].total` to the purchased amount || 200. So `records[beneficiary].total = 200`
After that the owner decides to adjust his `records[beneficiary].total` to 300. So `records[beneficiary].total = 300`
User decides to claim his claimable amount which should be equal to 300. He calls the claim function in `PriceTierVestingSale_2_0.sol`.
```
function claim(
    address beneficiary // the address that will receive tokens
  ) external validSaleParticipant(beneficiary) nonReentrant {
    uint256 claimableAmount = getClaimableAmount(beneficiary);
    uint256 purchasedAmount = getPurchasedAmount(beneficiary);

    // effects
    uint256 claimedAmount = super._executeClaim(beneficiary, purchasedAmount);

    // interactions
    super._settleClaim(beneficiary, claimedAmount);
  }
```
As we can see here the `_executeClaim` is called with the `purchasedAmount` of the user which is still 200.

```
function _executeClaim(
    address beneficiary,
    uint256 _totalAmount
  ) internal virtual returns (uint256) {
    uint120 totalAmount = uint120(_totalAmount);

    // effects
    if (records[beneficiary].total != totalAmount) {
      // re-initialize if the total has been updated
      _initializeDistributionRecord(beneficiary, totalAmount);
    }
    
    uint120 claimableAmount = uint120(getClaimableAmount(beneficiary));
    require(claimableAmount > 0, 'Distributor: no more tokens claimable right now');

    records[beneficiary].claimed += claimableAmount;
    claimed += claimableAmount;

    return claimableAmount;
  }
```

Now check the if statement:

```
 if (records[beneficiary].total != totalAmount) {
      // re-initialize if the total has been updated
      _initializeDistributionRecord(beneficiary, totalAmount);
    }
```

The point of this is if the total of the user has been adjusted, to re-initialize to the corresponding amount, but since it's updated by the input value which is 200, `records[beneficiary].total = 200` , the user will lose the 100 added from the owner during the adjust

# Impact
Loss of funds for the user and the protocol

# Tool used
Manual Review

# Recommendation
I am not sure if it is enough to just set it the following way:
```
 if (records[beneficiary].total != totalAmount) {
      // re-initialize if the total has been updated
      `--` _initializeDistributionRecord(beneficiary, totalAmount);
     `++` _initializeDistributionRecord(beneficiary, records[beneficiary].total);
    }
```
Think of different scenarios if it is done that way and also keep in mind that the same holds for the decrease of `records[beneficiary].total` by adjust
