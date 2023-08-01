![](https://audits.sherlock.xyz/_next/image?url=https%3A%2F%2Fsherlock-files.ams3.digitaloceanspaces.com%2Fcontests%2Ftokensoft.jpg&w=96&q=75)

# [Tokensoft](https://audits.sherlock.xyz/contests/100)

| Protocol | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|
| Tokensoft | 14,000 USDC | 650 | 4  days | Jul 17 2023 | Jul 21 2023 |

## Issues found :

| Severity | Title |
|:--|:--:|
| Unconfirmed | Loss of funds during user adjusting. |

# 1. Loss of funds during user adjusting.


Summary
Adjusting a user's total claimable value not working correctly

Vulnerability Detail
Whenever the owner is adjusting user's total claimable value, the records[beneficiary].total is decreased or increased by uint256 diff = uint256(amount > 0 ? amount : -amount);.

However some assumptions made are not correct. Scenario:

User has bought 200 FOO tokens for example.
In PriceTierVestingSale_2_0.sol he calls the initializeDistributionRecord which sets his records[beneficiary].total to the purchased amount || 200. So records[beneficiary].total = 200
After that the owner decides to adjust his records[beneficiary].total to 300. So records[beneficiary].total = 300
User decides to claim his claimable amount which should be equal to 300. He calls the claim function in PriceTierVestingSale_2_0.sol.
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
As we can see here the _executeClaim is called with the purchasedAmount of the user which is still 200.

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
Now check the if statement:

 if (records[beneficiary].total != totalAmount) {
      // re-initialize if the total has been updated
      _initializeDistributionRecord(beneficiary, totalAmount);
    }
The point of this is if the total of the user has been adjusted, to re-initialize to the corresponding amount, but since it's updated by the input value which is 200, records[beneficiary].total = 200 , the user will lose the 100 added from the owner during the adjust

Impact
Loss of funds for the user and the protocol

Code Snippet
2023-06-tokensoft-MAKEOUTHILL6/contracts/contracts/claim/PriceTierVestingSale_2_0.sol

Lines 75 to 109 in 644d289

   function getPurchasedAmount(address buyer) public view returns (uint256) { 
     /** 
     Get the quantity purchased from the sale and convert it to native tokens 
    
     Example: if a user buys $1.11 of a FOO token worth $0.50 each, the purchased amount will be 2.22 FOO 
     - buyer total: 111000000 ($1.11 with 8 decimals) 
     - decimals: 6 (the token being purchased has 6 decimals) 
     - price: 50000000 ($0.50 with 8 decimals) 
  
     Calculation: 111000000 * 1000000 / 50000000 
  
     Returns purchased amount: 2220000 (2.22 with 6 decimals) 
     */ 
     return (sale.buyerTotal(buyer) * (10 ** soldTokenDecimals)) / price; 
   } 
  
   function initializeDistributionRecord( 
     address beneficiary // the address that will receive tokens 
   ) external validSaleParticipant(beneficiary) { 
     _initializeDistributionRecord(beneficiary, getPurchasedAmount(beneficiary)); 
   } 
  
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
  
2023-06-tokensoft-MAKEOUTHILL6/contracts/contracts/claim/abstract/AdvancedDistributor.sol

Lines 105 to 131 in 644d289

 function adjust(address beneficiary, int256 amount) external onlyOwner { 
   DistributionRecord memory distributionRecord = records[beneficiary]; 
   require(distributionRecord.initialized, 'must initialize before adjusting'); 
  
   uint256 diff = uint256(amount > 0 ? amount : -amount); 
   require(diff < type(uint120).max, 'adjustment > max uint120'); 
  
   if (amount < 0) { 
     // decreasing claimable tokens 
     require(total >= diff, 'decrease greater than distributor total'); 
     require(distributionRecord.total >= diff, 'decrease greater than distributionRecord total'); 
     total -= diff; 
     records[beneficiary].total -= uint120(diff); 
     token.safeTransfer(owner(), diff); 
     // reduce voting power 
     _burn(beneficiary, tokensToVotes(diff)); 
   } else { 
     // increasing claimable tokens 
     total += diff; 
     records[beneficiary].total += uint120(diff); 
     // increase voting pwoer 
     _mint(beneficiary, tokensToVotes(diff)); 
   } 
  
   emit Adjust(beneficiary, amount); 
 } 
  
2023-06-tokensoft-MAKEOUTHILL6/contracts/contracts/claim/abstract/Distributor.sol

Lines 66 to 84 in 644d289

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
Tool used
Manual Review

Recommendation
I am not sure if it is enough to just set it the following way:

 if (records[beneficiary].total != totalAmount) {
      // re-initialize if the total has been updated
      `--` _initializeDistributionRecord(beneficiary, totalAmount);
     `++` _initializeDistributionRecord(beneficiary, records[beneficiary].total);
    }
Think of different scenarios if it is done that way and also keep in mind that the same holds for the decrease of records[beneficiary].total by adjust
