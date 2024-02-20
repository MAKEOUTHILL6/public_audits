![](https://code4rena.com/_next/image?url=https%3A%2F%2Fstorage.googleapis.com%2Fcdn-c4-uploads-v0%2Fuploads%2Fp44LEz6BxLQ.0&w=256&q=75)

# [Curves](https://code4rena.com/audits/2024-01-curves#top)

| Protocol | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|
| Curves | 36,500 USDC | 660 | 8 days | Jan 8 2024 | Jan 16 2024 |

## Issues found :

| Severity | Title |
|:--|:--:|
| High | User can gain unfair advantage on fees - draining FeeSplitter. |
| High | Malicious `curvesTokenSubject` can trap users inside token. |
| High | No access control implemented in FeeSplitter. |
| Low | LOW_REPORT |


# 1. User can gain unfair advantage on fees - draining FeeSplitter.

# Impact

First let's dive into how fee distribution works in order to understand the vulnerability better.

Users have to interact with Curves.sol's buyCurvesToken first.

```
function buyCurvesToken(address curvesTokenSubject, uint256 amount) public payable {

        uint256 startTime = presalesMeta[curvesTokenSubject].startTime;

        if (startTime != 0 && startTime >= block.timestamp) revert SaleNotOpen();

        _buyCurvesToken(curvesTokenSubject, amount);
    }
```

Let's assume there is no presale for the token so we go straight into _buyCurvesToken.

```
function _buyCurvesToken(address curvesTokenSubject, uint256 amount) internal {

        uint256 supply = curvesTokenSupply[curvesTokenSubject];

        if (!(supply > 0 || curvesTokenSubject == msg.sender)) revert UnauthorizedCurvesTokenSubject();

        uint256 price = getPrice(supply, amount);
        (, , , , uint256 totalFee) = getFees(price);

        if (msg.value < price + totalFee) revert InsufficientPayment();

        curvesTokenBalance[curvesTokenSubject][msg.sender] += amount;
        curvesTokenSupply[curvesTokenSubject] = supply + amount;
        _transferFees(curvesTokenSubject, true, price, amount, supply);

        // If is the first token bought, add to the list of owned tokens
        if (curvesTokenBalance[curvesTokenSubject][msg.sender] - amount == 0) {
            _addOwnedCurvesTokenSubject(msg.sender, curvesTokenSubject);
        }
    }
```

Assume there is already supply for that token. So we go ahead and buy the desired amount of tokens. Then when the amount is added to our balance, we go into _transferFees.

```
function _transferFees(
        address curvesTokenSubject,
        bool isBuy,
        uint256 price,
        uint256 amount,
        uint256 supply
    ) internal {
        (uint256 protocolFee, uint256 subjectFee, uint256 referralFee, uint256 holderFee, ) = getFees(price);
        {
            bool referralDefined = referralFeeDestination[curvesTokenSubject] != address(0);
            {
                address firstDestination = isBuy ? feesEconomics.protocolFeeDestination : msg.sender;
                uint256 buyValue = referralDefined ? protocolFee : protocolFee + referralFee;
                uint256 sellValue = price - protocolFee - subjectFee - referralFee - holderFee;
                (bool success1, ) = firstDestination.call{value: isBuy ? buyValue : sellValue}("");
                if (!success1) revert CannotSendFunds();
            }
            {
                (bool success2, ) = curvesTokenSubject.call{value: subjectFee}("");
                if (!success2) revert CannotSendFunds();
            }
            {
                (bool success3, ) = referralDefined
                    ? referralFeeDestination[curvesTokenSubject].call{value: referralFee}("")
                    : (true, bytes(""));
                if (!success3) revert CannotSendFunds();
            }

            if (feesEconomics.holdersFeePercent > 0 && address(feeRedistributor) != address(0)) {
                feeRedistributor.onBalanceChange(curvesTokenSubject, msg.sender);
                feeRedistributor.addFees{value: holderFee}(curvesTokenSubject);
            }
        }
        emit Trade(
            msg.sender,
            curvesTokenSubject,
            isBuy,
            amount,
            price,
            protocolFee,
            subjectFee,
            isBuy ? supply + amount : supply - amount
        );
    }
```

Our focus should be on the last lines of the function:

```
if (feesEconomics.holdersFeePercent > 0 && address(feeRedistributor) != address(0)) {
                feeRedistributor.onBalanceChange(curvesTokenSubject, msg.sender);
                feeRedistributor.addFees{value: holderFee}(curvesTokenSubject);
            }
```

When we go into the if statement, onBalanceChange is called, which is used to set the user's data.userFeeOffset to the current data.cumulativeFeePerToken.

```
function onBalanceChange(address token, address account) public onlyManager {
        TokenData storage data = tokensData[token];
        data.userFeeOffset[account] = data.cumulativeFeePerToken;
        if (balanceOf(token, account) > 0) userTokens[account].push(token);
    }
```

**Before going into the onBalanceChange example:**

data.userFeeOffset = 0;
data.cumulativeFeePerToken = 100

**After going into the onBalanceChange example:**

data.userFeeOffset = 100;
data.cumulativeFeePerToken = 100

That way the user starts earning rewards from the point he deposited tokens for that specific curvesTokenSubject - that's the important part.

Next feeRedistributor.addFees is called:

```
function addFees(address token) public payable onlyManager {

        uint256 totalSupply_ = totalSupply(token);

        if (totalSupply_ == 0) revert NoTokenHolders();

        TokenData storage data = tokensData[token];

        data.cumulativeFeePerToken += (msg.value * PRECISION) / totalSupply_;
    }
```

Let's say that after addFees now data.cumulativeFeePerToken = 120 for example.

So final state:
data.userFeeOffset = 100;
data.cumulativeFeePerToken = 120

Now when we call getClaimableFees in FeeSplitter:

```
function getClaimableFees(address token, address account) public view returns (uint256) {

        TokenData storage data = tokensData[token];

        uint256 balance = balanceOf(token, account);

        uint256 owed = (data.cumulativeFeePerToken - data.userFeeOffset[account]) * balance;

        return (owed / PRECISION) + data.unclaimedFees[account];
    }
```

We can see that we will get the owed amount only for that period 100 to 120 = 20. That's the right way to be implemented

However what if we can somehow obtain balance for a token without invoking onBalanceChange and whenever we want to claim we get instantaneous access to the all time fees earned for that token? Meaning when we call getClaimableFees our data.userFeeOffset is 0. Well here is how and it's quite simple.



# Proof of Concept

So in order to get any fees we must have balance for that specific token because this calculation in getClaimableFees will always return 0 otherwise:

`uint256 owed = (data.cumulativeFeePerToken - data.userFeeOffset[account]) * balance;`

There is a way for a user to get balance for a token without invoking the onBalanceChange and that's transferCurvesToken:

```
function transferCurvesToken(address curvesTokenSubject, address to, uint256 amount) external {
        if (to == address(this)) revert ContractCannotReceiveTransfer();
        _transfer(curvesTokenSubject, msg.sender, to, amount);
    }
```

```
function _transfer(address curvesTokenSubject, address from, address to, uint256 amount) internal {

        if (amount > curvesTokenBalance[curvesTokenSubject][from]) revert InsufficientBalance();

        // If transferring from oneself, skip adding to the list
        if (from != to) {
            _addOwnedCurvesTokenSubject(to, curvesTokenSubject);
        }

        curvesTokenBalance[curvesTokenSubject][from] = curvesTokenBalance[curvesTokenSubject][from] - amount;
        curvesTokenBalance[curvesTokenSubject][to] = curvesTokenBalance[curvesTokenSubject][to] + amount;

        emit Transfer(curvesTokenSubject, from, to, amount);
    }
```

As you can see here any user can transfer any amount to an arbitrary to address for any token.

Scenario:

Alice has 100 "ATokens", and wants to take advantage of the fees vulnerability, she directly transfers 70 tokens with transferCurvesToken to her second account for example, then when that second account calls claimFees in FeeSplitter.

```
function claimFees(address token) external {

        updateFeeCredit(token, msg.sender);

        uint256 claimable = getClaimableFees(token, msg.sender);

        if (claimable == 0) revert NoFeesToClaim();

        tokensData[token].unclaimedFees[msg.sender] = 0;

        payable(msg.sender).transfer(claimable);

        emit FeesClaimed(token, msg.sender, claimable);
    }
```

Let's go into updateFeeCredit:

```
function updateFeeCredit(address token, address account) internal {

        TokenData storage data = tokensData[token];

        uint256 balance = balanceOf(token, account);

        if (balance > 0) {

            uint256 owed = (data.cumulativeFeePerToken - data.userFeeOffset[account]) * balance;

            data.unclaimedFees[account] += owed / PRECISION;
            data.userFeeOffset[account] = data.cumulativeFeePerToken;

        }
        
    }
```

uint256 balance for that user will be 70 as the example above.
Then the interesting thing here is that the owed amount will be:

`uint256 owed = (data.cumulativeFeePerToken - data.userFeeOffset[account]) * balance; == uint256 owed = (120 - 0) * balance;`

**NOTE that our user's data.userFeeOffset is 0, it's not set to the current data.cumulativeFeePerToken**

Then the owed amount is added to Alice's second account's data.unclaimedFees and data.userFeeOffset[account] is set to the current data.cumulativeFeePerToken

As you saw the user gets instantaneous access to all fees earned from the beginning, now this second account of Alice can transfer those 70 tokens of "ATokens" to another account and repeat this for the specific token to drain the FeeSplitter.


# Tool used
Manual Review

# Recommendation

Implement the onBalanceChange when a user directly transfers tokens to another user in order to mitigate the issue.


# 2. Malicious `curvesTokenSubject` can trap users inside token.

# Impact

curvesTokenSubject owner can at any time call setReferralFeeDestination

```
function setReferralFeeDestination(
        address curvesTokenSubject,
        address referralFeeDestination_
    ) public onlyTokenSubject(curvesTokenSubject) {
        referralFeeDestination[curvesTokenSubject] = referralFeeDestination_;
    }
```

The referral of a curvesTokenSubject is used in _transferFees which is invoked by both sellCurvesToken and buyCurvesToken.

A malicious curvesTokenSubject can trap user's token inside the curvesTokenSubject by changing the referral to a malicious contract that would revert on transfers - reverting sellCurvesToken.


# Proof of Concept

User decides to sell his curvesTokenSubject tokens so he calls sellCurvesToken:

```
function sellCurvesToken(address curvesTokenSubject, uint256 amount) public {

        uint256 supply = curvesTokenSupply[curvesTokenSubject];

        if (supply <= amount) revert LastTokenCannotBeSold();

        if (curvesTokenBalance[curvesTokenSubject][msg.sender] < amount) revert InsufficientBalance();

        uint256 price = getPrice(supply - amount, amount);

        curvesTokenBalance[curvesTokenSubject][msg.sender] -= amount;

        curvesTokenSupply[curvesTokenSubject] = supply - amount;

        _transferFees(curvesTokenSubject, false, price, amount, supply);
    }
```

Nothing fancy here, let's get straight into _transferFees:

```
function _transferFees(
        address curvesTokenSubject,
        bool isBuy,
        uint256 price,
        uint256 amount,
        uint256 supply
    ) internal {

        (uint256 protocolFee, uint256 subjectFee, uint256 referralFee, uint256 holderFee, ) = getFees(price);

        {
            bool referralDefined = referralFeeDestination[curvesTokenSubject] != address(0);

            {
                address firstDestination = isBuy ? feesEconomics.protocolFeeDestination : msg.sender;

                uint256 buyValue = referralDefined ? protocolFee : protocolFee + referralFee;

                uint256 sellValue = price - protocolFee - subjectFee - referralFee - holderFee;

                (bool success1, ) = firstDestination.call{value: isBuy ? buyValue : sellValue}("");
                if (!success1) revert CannotSendFunds();
            }

            {
                (bool success2, ) = curvesTokenSubject.call{value: subjectFee}("");
                if (!success2) revert CannotSendFunds();
            }

            {
                (bool success3, ) = referralDefined
                    ? referralFeeDestination[curvesTokenSubject].call{value: referralFee}("")
                    : (true, bytes(""));
                if (!success3) revert CannotSendFunds();
            }

            if (feesEconomics.holdersFeePercent > 0 && address(feeRedistributor) != address(0)) {

                feeRedistributor.onBalanceChange(curvesTokenSubject, msg.sender);
                feeRedistributor.addFees{value: holderFee}(curvesTokenSubject);
            }

        }

        emit Trade(
            msg.sender,
            curvesTokenSubject,
            isBuy,
            amount,
            price,
            protocolFee,
            subjectFee,
            isBuy ? supply + amount : supply - amount
        );
    }
```

First there is a check if a referral is set for the particular curvesTokenSubject - bool referralDefined = referralFeeDestination[curvesTokenSubject] != address(0); .

Then in the function, if there is an existing referral we send the referralFee there, which is in eth and this opens an attacker vector for a malicious revert if the referral is made so that it doesn't accept eth and success returns false in order to revert the whole transaction and make the user unable to sell his curvesTokenSubject tokens.

Now user's funds become trapped in curvesTokenSubject.
The malicious curvesTokenSubject can always switch back to the "non-malicious" referral in order to accept buy orders, then monitor the mempool for sell orders and switch to the malicious one in order to revert them.



# Tool used
Manual Review

# Recommendation

Make the referral to be an EOA or make it that there is whitelist of trusted referrals which can be used in order to avoid those malicious reverts. Also implement a timelock on this function.


# 3. No access control implemented in FeeSplitter.

# Impact

In FeeSplitter.sol function setCurves lacks access control allowing anyone to set curves to any arbitrary address which in return can provide fake outputs on functions like balanceOf and totalSupply to steal fees from the contract.


# Proof of Concept

```
function setCurves(Curves curves_) public {
        curves = curves_;
    }
```

No access control set, by changing curves to some arbitrary contract address, an attacker can manipulate the outputs of:

```
function balanceOf(address token, address account) public view returns (uint256) {
        return curves.curvesTokenBalance(token, account) * PRECISION;
    }
```

and

```
function totalSupply(address token) public view returns (uint256) {
        //@dev: this is the amount of tokens that are not locked in the contract. The locked tokens are in the ERC20 contract
        return (curves.curvesTokenSupply(token) - curves.curvesTokenBalance(token, address(curves))) * PRECISION;
    }
```

Which will allow the user to steal fees from the feeSplitter in current scenarios.



# Tool used
Manual Review

# Recommendation

Implement an access control on setCurves.


# 4. LOWS_REPORT.

 **1 LOW** - In FeeSplitter user will lose fees if he buyCurvesToken or sellCurvesToken if he hasn't claimed it beforehand.

For example In FeeSplitter:

data.userFeeOffset[account] = 100; data.cumulativeFeePerToken = 200;

The moment user calls buyCurvesToken or sellCurvesToken it will invoke onBalanceChange. Which:

```
function onBalanceChange(address token, address account) public onlyManager {
        TokenData storage data = tokensData[token];
        data.userFeeOffset[account] = data.cumulativeFeePerToken;
        if (balanceOf(token, account) > 0) userTokens[account].push(token);
    }
```
Will set the data.userFeeOffset to the current data.cumulativeFeePerToken. So if a user doesn't call claimFees before buying more tokens he will lose fees.

**2 LOW** - It's possible that no buying of tokens can occur in Curves.sol for one block. That occurs because of the if (startTime != 0 && startTime >= block.timestamp) revert SaleNotOpen(); in buyCurvesToken and

```
if (
           presalesMeta[curvesTokenSubject].startTime == 0 ||
           presalesMeta[curvesTokenSubject].startTime <= block.timestamp
       ) revert PresaleUnavailable();
```

in buyCurvesTokenWhitelisted.


As you can see both require startTime to not be in a state where it is == block.timestamp.


