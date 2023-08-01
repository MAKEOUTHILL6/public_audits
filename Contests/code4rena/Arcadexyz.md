![](https://code4rena.com/_next/image?url=https%3A%2F%2Fstorage.googleapis.com%2Fcdn-c4-uploads-v0%2Fuploads%2FLh9u6MWaWVh.0&w=96&q=75)

# [Arcadexyz](https://code4rena.com/contests/2023-07-arcadexyz#top)

| Protocol | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|
| Arcadexyz | 90,500 USDC | 2000  | 7  days | 21 Jul 2023 | 28 Jul 2023 |

## Issues found :

| Severity | Title |
|:--|:--:|
| Unconfirmed | User can be set with an NFT with multiplier = 0 and break delegatee votes. |
| Unconfirmed | User excessive funds won't be refunded when minting badges. |

# 1. User can be set with an NFT with multiplier = 0 and break delegatee votes.

 # Lines of code

https://github.com/code-423n4/2023-07-arcade/blob/f8ac4e7c4fdea559b73d9dd5606f618d4e6c73cd/contracts/NFTBoostVault.sol#L305-L330
https://github.com/code-423n4/2023-07-arcade/blob/f8ac4e7c4fdea559b73d9dd5606f618d4e6c73cd/contracts/NFTBoostVault.sol#L469-L479
https://github.com/code-423n4/2023-07-arcade/blob/f8ac4e7c4fdea559b73d9dd5606f618d4e6c73cd/contracts/NFTBoostVault.sol#L579-L599
https://github.com/code-423n4/2023-07-arcade/blob/f8ac4e7c4fdea559b73d9dd5606f618d4e6c73cd/contracts/NFTBoostVault.sol#L627-L637


# Vulnerability details

## Impact
The function `_registerAndDelegate` in `NFTBoostVault`, there is a check which explicitly forbids a user to be set an NFT that has a 0 multiplier, however that check can be bypassed when NFT is set through the `updateNFT` function. NFT with an 0 multiplier can break the vote logic for a delegatee
## Proof of Concept
```
function _registerAndDelegate(
    address user,
    uint128 _amount,
    uint128 _tokenId,
    address _tokenAddress,
    address _delegatee
) internal {
    uint128 multiplier = 1e3;

    if (_tokenAddress != address(0) && _tokenId != 0) {
        if (IERC1155(_tokenAddress).balanceOf(user, _tokenId) == 0) revert NBV_DoesNotOwn();

        multiplier = getMultiplier(_tokenAddress, _tokenId);

        if (multiplier == 0) revert NBV_NoMultiplierSet();
    }
```

Explicitly checked that an NFT can't have a 0 multiplier.

```
function updateNft(uint128 newTokenId, address newTokenAddress) external override nonReentrant {
    if (newTokenAddress == address(0) || newTokenId == 0) revert NBV_InvalidNft(newTokenAddress, newTokenId);

    if (IERC1155(newTokenAddress).balanceOf(msg.sender, newTokenId) == 0) revert NBV_DoesNotOwn();

    NFTBoostVaultStorage.Registration storage registration = _getRegistrations()[msg.sender];

    if (registration.delegatee == address(0)) revert NBV_NoRegistration();

    if (registration.tokenAddress != address(0) && registration.tokenId != 0) {
        //@audit this won't change shit
        _withdrawNft();
    }

    registration.tokenAddress = newTokenAddress;
    registration.tokenId = newTokenId;

    _lockNft(msg.sender, newTokenAddress, newTokenId, 1);

    _syncVotingPower(msg.sender, registration);
}
```
However here we can see that the check prohibiting this action is missing and user can set an NFT with 0 multiplier either maliciously or accidentally.

Scenario:
User has made a registration and set the following:
-----------------------
Registration:
uint128 newVotingPower = (_amount * multiplier) / MULTIPLIER_DENOMINATOR;
For simplicity we assume no multiplier is set so multiplier = 1e3
uint128 newVotingPower = 50 * 1e3 = 50
registration.amount = 50
registration.latestVotingPower = 50
And the delegate is the creator of the registration with no prior vote power
delegatee = 0 + 50 = 50
----------------------
Say that he now wants to change his NFT to earn some boost, he uses the `updateNFT()` and sets the corresponding tokenID and tokenAddress in the input, but since there is no check to confirm that they actually have any multiplier set, if thats the case the user will be setting an NFT with 0 multiplier which leads to the following when votes are synced:
----------------------
updateNft
```
function updateNft(uint128 newTokenId, address newTokenAddress) external override nonReentrant {
    if (newTokenAddress == address(0) || newTokenId == 0) revert NBV_InvalidNft(newTokenAddress, newTokenId);

    if (IERC1155(newTokenAddress).balanceOf(msg.sender, newTokenId) == 0) revert NBV_DoesNotOwn();

    NFTBoostVaultStorage.Registration storage registration = _getRegistrations()[msg.sender];

    if (registration.delegatee == address(0)) revert NBV_NoRegistration();

    // since we don't have nft set yet that part is skipped
    if (registration.tokenAddress != address(0) && registration.tokenId != 0) {
        _withdrawNft();
    }

    registration.tokenAddress = newTokenAddress;
    registration.tokenId = newTokenId;

    _lockNft(msg.sender, newTokenAddress, newTokenId, 1);

    _syncVotingPower(msg.sender, registration);
}
```
After `updateNFT` confirms that a user has control over the NFT and also that no prior NFT exists, it straight
sets the `registration.tokenAddress = newTokenAddress;` and `registration.tokenId = newTokenId;` locks the nft
and goes straight into the `_syncVotingPower` with updated NFT values.

```
function _syncVotingPower(address who, NFTBoostVaultStorage.Registration storage registration) internal {

    NFT with 0 multiplier:

    History.HistoricalBalances memory votingPower = _votingPower();
    // delegateeVotes will be 50 based on registration values
    uint256 delegateeVotes = votingPower.loadTop(registration.delegatee);
------------------
    // //newVotingPower will be 0 because of this calculation in `_currentVotingPower`:
function _currentVotingPower(
    NFTBoostVaultStorage.Registration memory registration
) internal view virtual returns (uint256) {
     // 50
    uint128 locked = registration.amount - registration.withdrawn;
      // there is NFT set so locked * multiplier = 50 * 0 = 0
    if (registration.tokenAddress != address(0) && registration.tokenId != 0) {
        return (locked * getMultiplier(registration.tokenAddress, registration.tokenId)) / MULTIPLIER_DENOMINATOR;
    }

    return locked;
}
---------------
    // 0
    uint256 newVotingPower = _currentVotingPower(registration);
    // 0 - 50 = -50
    int256 change = int256(newVotingPower) - int256(uint256(registration.latestVotingPower));

    if (change == 0) return;
    if (change > 0) {
        votingPower.push(registration.delegatee, delegateeVotes + uint256(change));
    } else {
        delegatee = 50 - 50 = 0
        votingPower.push(registration.delegatee, delegateeVotes - uint256(change * -1));
    }
    //0
    registration.latestVotingPower = uint128(newVotingPower);

    emit VoteChange(who, registration.delegatee, change);
}
```

Thus delegatee will lose voting power which is not correct. The multiplier should be defaulted to 1e3 if there is no multiplier set for this NFT.

## Tools Used
Manual Audit

## Recommended Mitigation Steps
Add the same checks as in `_registerAndDelegate`.


## Assessed type

Invalid Validation

# 2. User excessive funds won't be refunded when minting badges.

# Lines of code

https://github.com/code-423n4/2023-07-arcade/blob/f8ac4e7c4fdea559b73d9dd5606f618d4e6c73cd/contracts/nft/ReputationBadge.sol#L109


# Vulnerability details

## Impact
Whenever a user is minting badges in `ReputationBadge.sol` the function `mint` is called. Which calculates
the mint price based on the badge price and the amount that is going to be minted. But there is a check `if (msg.value < mintPrice) revert RB_InvalidMintFee(mintPrice, msg.value);` which requires the user to send more or equal to the `mintPrice`.


## Proof of Concept
`mint` function:
```
function mint(
    address recipient,
    uint256 tokenId,
    uint256 amount,
    uint256 totalClaimable,
    bytes32[] calldata merkleProof
) external payable {
    uint256 mintPrice = mintPrices[tokenId] * amount;
    uint48 claimExpiration = claimExpirations[tokenId];
    if (block.timestamp > claimExpiration) revert RB_ClaimingExpired(claimExpiration, uint48(block.timestamp));
    //@audit if a user sends more, there is no return of his funds
    if (msg.value < mintPrice) revert RB_InvalidMintFee(mintPrice, msg.value);
    if (!_verifyClaim(recipient, tokenId, totalClaimable, merkleProof)) revert RB_InvalidMerkleProof();

    if (amountClaimed[recipient][tokenId] + amount > totalClaimable) {
        revert RB_InvalidClaimAmount(amount, totalClaimable);
    }

    // increment amount claime
    amountClaimed[recipient][tokenId] += amount;

    // mint to recipient
    _mint(recipient, tokenId, amount, "");
}
```
Since it requires the msg.value to not be < `mintPrice`, means that a user can actually send more than what he is supposed and he won't be refunded.

## Tools Used
Manual Audit

## Recommended Mitigation Steps
`--` if (msg.value < mintPrice) revert RB_InvalidMintFee(mintPrice, msg.value);
`++` if (msg.value != mintPrice) revert RB_InvalidMintFee(mintPrice, msg.value);




## Assessed type

ETH-Transfer
