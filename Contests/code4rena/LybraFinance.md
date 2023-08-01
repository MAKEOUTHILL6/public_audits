![](https://code4rena.com/_next/image?url=https%3A%2F%2Fstorage.googleapis.com%2Fcdn-c4-uploads-v0%2Fuploads%2FAhr3uHj65Gw.0&w=96&q=75)

# [Lybra Finance](https://code4rena.com/contests/2023-06-lybra-finance#top)

| Protocol | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|
| Lybra Finance | 60,500 USDC | 1680 | 10  days | 23 Jun 2023 | 3 Jul 2023 |

## Issues found :

| Severity | Title |
|:--|:--:|
| Unconfirmed | First time staker will lose LBR tokens when unstaking. |
| Unconfirmed | Wrong keeper ratio limit. |
| Unconfirmed | Owner will be address(0) because it is not initialized. |
| Unconfirmed | Wrong interface function leading to wrong asset price. |
| Unconfirmed | Wrong hardcoded asset address. |


# 1. First time staker will lose LBR tokens when unstaking

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/ProtocolRewardsPool.sol#L87-L98
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/ProtocolRewardsPool.sol#L100-L107
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/ProtocolRewardsPool.sol#L155-L159


# Vulnerability details

## Impact
User will lose his LBR tokens

## Proof of Concept
Normal engagement with the `ProtocolRewardsPool.sol` would be:

1.) User calls stake and burns his LBR in order to receive esLBR tokens.
2.) `(not important step, just more financial losses)` He decides that he wants more LBR tokens so he locks his esLBR tokens.
3.) Time passes and he finally wants to `unstake()`
###### Please look at the comments in the code below
```
function unstake(uint256 amount) external {
      //@note this check passes, lock in period has ended
      require(block.timestamp >= esLBRBoost.getUnlockTime(msg.sender), "Your lock-in period has not ended. You can't convert your esLBR now.");
      // burns his desired amount of esLBR tokens
      esLBR.burn(msg.sender, amount);
      // initiates a withdraw, look below for `withdraw()` code
      withdraw(msg.sender);
      uint256 total = amount;
      if (time2fullRedemption[msg.sender] > block.timestamp) {
          total += unstakeRatio[msg.sender] * (time2fullRedemption[msg.sender] - block.timestamp);
      }
      unstakeRatio[msg.sender] = total / exitCycle;
      time2fullRedemption[msg.sender] = block.timestamp + exitCycle;
      emit UnstakeLBR(msg.sender, amount, block.timestamp);
  }
```
```
function withdraw(address user) public {
      uint256 amount = getClaimAbleLBR(user);
      if (amount > 0) {
          LBR.mint(user, amount);
      }
      lastWithdrawTime[user] = block.timestamp;
      emit WithdrawLBR(user, amount, block.timestamp);
  }
```
In order to receive the eligible amount of LBR tokens, `getClaimAbleLBR()` is called:

```
function getClaimAbleLBR(address user) public view returns (uint256 amount) {
      // 0 > 0
      if (time2fullRedemption[user] > lastWithdrawTime[user]) {
          amount = block.timestamp > time2fullRedemption[user] ? unstakeRatio[user] * (time2fullRedemption[user] - lastWithdrawTime[user]) : unstakeRatio[user] * (block.timestamp - lastWithdrawTime[user]);
      }
  }`
```
In order for a user to get his share of LBR token this check must pass, but since it's the first time for a user to `unstake()` and `withdraw()` both variables `time2fullRedemption[user]` and `lastWithdrawTime[user]` will be 0, hence `0 > 0` and the `if` won't pass making the `amount` variable in `withdraw()` to be equal to 0 and no LBR is minted.

User loses LBR tokens

## Tools Used
Manual Audit

## Recommended Mitigation Steps

Easiest fix I can think of is this:

```
function getClaimAbleLBR(address user) public view returns (uint256 amount) {
      // 0 > 0
     -- if (time2fullRedemption[user] > lastWithdrawTime[user])
     ++ if (time2fullRedemption[user] >= lastWithdrawTime[user]) {
          amount = block.timestamp > time2fullRedemption[user] ? unstakeRatio[user] * (time2fullRedemption[user] - lastWithdrawTime[user]) : unstakeRatio[user] * (block.timestamp - lastWithdrawTime[user]);
      }
  }`
```

## Assessed type

Invalid Validation

# 2. Wrong keeper ratio limit

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/configuration/LybraConfigurator.sol#L224-L228

# Vulnerability details

## Impact
Contract manager won't be able to set the `vaultKeeperRatio[pool]` for more than `0.05%`

## Proof of Concept
``` /**
  * @notice Set the reward ratio for the liquidator after liquidation.
  * @param pool The address of the pool to set the reward ratio for.
  * @param newRatio The new reward ratio to set, limited to a maximum of 5%.
  */

 function setKeeperRatio(address pool,uint256 newRatio) external checkRole(TIMELOCK) {
     //@audit wrong value, should be 500
     require(newRatio <= 5, "Max Keeper reward is 5%");
     vaultKeeperRatio[pool] = newRatio;
     emit KeeperRatioChanged(pool, newRatio);
 }
```
As we can see from the function comments, the maximum value is limited to 5% => 500, but in the `require` statement the maximum newRatio can't be bigger than 0,05% => 5

## Tools Used
Manual Audit

## Recommended Mitigation Steps
```
function setKeeperRatio(address pool,uint256 newRatio) external checkRole(TIMELOCK) {
     //@audit wrong value, should be 500
     -- require(newRatio <= 5, "Max Keeper reward is 5%");
     ++ require(newRatio <= 500, "Max Keeper reward is 5%");
     vaultKeeperRatio[pool] = newRatio;
     emit KeeperRatioChanged(pool, newRatio);
 }
```

## Assessed type

Invalid Validation

# 3. Owner will be address(0) because it is not initialized

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/esLBRBoost.sol#L5
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/EUSDMiningIncentives.sol#L12
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/ProtocolRewardsPool.sol#L12
https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/miner/stakerewardV2pool.sol#L4


# Vulnerability details

## Impact
Owner will be address(0) making the functions which use the onlyOwner modifier insolvable

## Proof of Concept
There are contracts in the protocol which use the `Ownable` from OZ:
`esLBRBoost.sol, EUSDMiningIncentives.sol, ProtocolRewardsPool.sol, StakingRewardsV2.sol`, but they are not initializing it. Now `Ownable` requires the contract to specifically give an address to be initialized as the owner

```
constructor(address initialOwner) {
     _transferOwnership(initialOwner);
 }

```

Hence if the owner is not initialized as required in the new `OZ's Ownable` "update" it will be set automatically to address(0). For more information about this issue read this: https://github.com/OpenZeppelin/openzeppelin-contracts/issues/4368

## Tools Used
Manual Audit

## Recommended Mitigation Steps
Initialize the owner in the constructor


## Assessed type

Access Control

# 4. Wrong interface function leading to wrong asset price

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/LybraRETHVault.sol#L9-L11


# Vulnerability details

## Impact
Asset price won't be correct since the function from the interface is incorrect and also doesn't even exist for the rETH token

## Proof of Concept
Whenever price of asset is calculated it uses `getExchangeRatio`:

```
function getAssetPrice() public override returns (uint256) {
    return (_etherPrice() * IRETH(address(collateralAsset)).getExchangeRatio()) / 1e18;
}
```

HOWEVER when you go in the contract for the rETH token - 0xae78736Cd615f374D3085123A210448E74Fc6393. You can see that there is no such function as `getExchangeRatio`:

```
interface IRETH {
//@audit wrong interface function, getExchangeRate
function getExchangeRatio() external view returns (uint256);
}
```

As sponsor confirmed, the right function is `getExchangeRate()`

## Tools Used
Manual Audit

## Recommended Mitigation Steps
To receive the right asset price you should:

```
interface IRETH {
-- function getExchangeRatio() external view returns (uint256);
++ function getExchangeRate() external view returns (uint256);
}
```

```
function getAssetPrice() public override returns (uint256) {
    -- return (_etherPrice() * IRETH(address(collateralAsset)).getExchangeRatio()) / 1e18;
    ++ return (_etherPrice() * IRETH(address(collateralAsset)).getExchangeRate()) / 1e18;
}
```

## Assessed type

Other

# 5. Wrong hardcoded asset address.

# Lines of code

https://github.com/code-423n4/2023-06-lybra/blob/7b73ef2fbb542b569e182d9abf79be643ca883ee/contracts/lybra/pools/LybraWbETHVault.sol#L16


# Vulnerability details

## Impact
Wrong asset address will be used for the wBETH vault

## Proof of Concept
The current wBETH hardcoded in the contract is:
```
//@audit rETH
//WBETH = 0xae78736Cd615f374D3085123A210448E74Fc6393
```
But that is actually the rETH address

## Tools Used
Manual Audit

## Recommended Mitigation Steps
Change to correct wBETH address

## Assessed type

Other
