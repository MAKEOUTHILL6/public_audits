![](https://audits.sherlock.xyz/_next/image?url=https%3A%2F%2Fsherlock-files.ams3.digitaloceanspaces.com%2Fcontests%2Flooksrare.jpg&w=256&q=75)

# [LooksRare Yolo](https://audits.sherlock.xyz/contests/163)

| Protocol | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|
| LooksRare Yolo | 27,000 USDC | 1,171 | 6 days | 18 Jan 2024 | 24 Jan 2024 |

## Issues found :

| Severity | Title |
|:--|:--:|
| High | User can get free entries to rounds |
| Medium | A ready to be withdrawn round can be forcefully extended by a single user |


# 1. User can get free entries to rounds

# Summary
User can get as many free entries as he want for as many rounds as he want

# Vulnerability Detail

User can get as many entries as he wants for as many rounds as he want for the cost of only 1 deposit for 1 round

This can be done by abusing depositETHIntoMultipleRounds:

```
function depositETHIntoMultipleRounds(uint256[] calldata amounts) external payable nonReentrant whenNotPaused {

        uint256 numberOfRounds = amounts.length;

        if (msg.value == 0 || numberOfRounds == 0) {
            revert ZeroDeposits();
        }

        uint256 startingRoundId = roundsCount;

        Round storage startingRound = rounds[startingRoundId];

        _validateRoundIsOpen(startingRound);

        _setCutoffTimeIfNotSet(startingRound);

        uint256 expectedValue;

        uint256[] memory entriesCounts = new uint256[](numberOfRounds);
 
        for (uint256 i; i < numberOfRounds; ++i) {

            uint256 roundId = _unsafeAdd(startingRoundId, i);

            Round storage round = rounds[roundId];

            uint256 roundValuePerEntry = round.valuePerEntry;

            if (roundValuePerEntry == 0) {
                (, , roundValuePerEntry) = _writeDataToRound({roundId: roundId, roundValue: 0});
            }

            _incrementUserDepositCount(roundId, round);

            uint256 depositAmount = amounts[i];
            
            if (depositAmount % roundValuePerEntry != 0) {
                revert InvalidValue();
            }

            uint256 entriesCount = _depositETH(round, roundId, roundValuePerEntry, depositAmount);

            expectedValue += depositAmount;

            entriesCounts[i] = entriesCount;
        }

        if (expectedValue != msg.value) {
            revert InvalidValue();
        }

        emit MultipleRoundsDeposited(msg.sender, startingRoundId, amounts, entriesCounts);

        if (
            _shouldDrawWinner(
                startingRound.numberOfParticipants,
                startingRound.maximumNumberOfParticipants,
                startingRound.deposits.length
            )
        ) {
            _drawWinner(startingRound, startingRoundId);
        }
        
    }
```

The user can construct the input array like that for example (keep in mind numbers are just for easier understanding):
**uint256[] calldata amounts** = **[1eth, 0, 0]** - basically 1eth for the current(first) round, the 0 for the second and 0 for the third. And this will be valid since:

```
uint256 numberOfRounds = amounts.length;

        if (msg.value == 0 || numberOfRounds == 0) {
            revert ZeroDeposits();
        }

```

msg.value > 0 and also there are numberOfRounds - 3.

Then:

```
        uint256 startingRoundId = roundsCount;

        Round storage startingRound = rounds[startingRoundId];

        _validateRoundIsOpen(startingRound);

        _setCutoffTimeIfNotSet(startingRound);

        uint256 expectedValue;

        uint256[] memory entriesCounts = new uint256[](numberOfRounds);

```

just some checks let's assume everything is alright here. Let's go into the for loop of the function:

```
for (uint256 i; i < numberOfRounds; ++i) {

            uint256 roundId = _unsafeAdd(startingRoundId, i);

            Round storage round = rounds[roundId];

            uint256 roundValuePerEntry = round.valuePerEntry;

            if (roundValuePerEntry == 0) {
                (, , roundValuePerEntry) = _writeDataToRound({roundId: roundId, roundValue: 0});
            }

            _incrementUserDepositCount(roundId, round);

            uint256 depositAmount = amounts[i];
            
            if (depositAmount % roundValuePerEntry != 0) {
                revert InvalidValue();
            }

            uint256 entriesCount = _depositETH(round, roundId, roundValuePerEntry, depositAmount);

            expectedValue += depositAmount;

            entriesCounts[i] = entriesCount;
        }
```

Now for the first round we have prepared 1eth, we are added as participants for that round by _incrementUserDepositCount, then our deposit amount is stored in uint256 depositAmount = amounts[i];, then our deposit is pushed to the deposits for the current round by _depositETH and important that expectedValue += depositAmount, expectedValue will be later used as a check to ensure we have provided enough amount for our deposits. Now let's get to the second round but now our amount is 0, the same scenario as above is performed and since there are no checks to prevent 0 amount deposits, we are once again added as participants for the round and our 0 amount deposit is added to the deposits of the round. Lets just end the for cycle here, you get the idea now.

`if (expectedValue != msg.value) { revert InvalidValue(); }`

this check at the end ensures that we have provided enough msg.value for every deposit we made, but as you can see we made 3 total deposits but we paid only for 1, so this check will pass successfully.

# Impact
A user can use this to fill up a round with 0 amount deposits to ddos a round, or he can use this in his advantage to get as many deposits as he can to have better chance to be selected for the winner of the round by fulfillRandomWords.

# Tool used
Manual Review

# Recommendation
Make sure 0 amount deposits are impossible.



# 2. A ready to be withdrawn round can be forcefully extended by a single user

# Summary
A malicious depositor can forcefully extend a ready-to-be withdrawn round.

# Vulnerability Detail

STATE
roundCount = 1 - OUR CURRENT ROUND STATE

Depositors can deposit for future(multiple) rounds by depositETHIntoMultipleRounds:

```
    function depositETHIntoMultipleRounds(uint256[] calldata amounts) external payable nonReentrant whenNotPaused {

        uint256 numberOfRounds = amounts.length;

        if (msg.value == 0 || numberOfRounds == 0) {
            revert ZeroDeposits();
        }

        uint256 startingRoundId = roundsCount;

        Round storage startingRound = rounds[startingRoundId];

        _validateRoundIsOpen(startingRound);

        _setCutoffTimeIfNotSet(startingRound);

        uint256 expectedValue;

        uint256[] memory entriesCounts = new uint256[](numberOfRounds);
 
        for (uint256 i; i < numberOfRounds; ++i) {

            uint256 roundId = _unsafeAdd(startingRoundId, i);

            Round storage round = rounds[roundId];

            uint256 roundValuePerEntry = round.valuePerEntry;

            if (roundValuePerEntry == 0) {
                (, , roundValuePerEntry) = _writeDataToRound({roundId: roundId, roundValue: 0});
            }

            _incrementUserDepositCount(roundId, round);

            uint256 depositAmount = amounts[i];

            if (depositAmount % roundValuePerEntry != 0) {
                revert InvalidValue();
            }

            uint256 entriesCount = _depositETH(round, roundId, roundValuePerEntry, depositAmount);

            expectedValue += depositAmount;

            entriesCounts[i] = entriesCount;
        }

        if (expectedValue != msg.value) {
            revert InvalidValue();
        }

        emit MultipleRoundsDeposited(msg.sender, startingRoundId, amounts, entriesCounts);

        if (
            _shouldDrawWinner(
                startingRound.numberOfParticipants,
                startingRound.maximumNumberOfParticipants,
                startingRound.deposits.length
            )
        ) {
            _drawWinner(startingRound, startingRoundId);
        }
        
    }

```

A group of depositors want take part in the next round(2) by using the above function, they can do that and fill the whole round with the needed deposits count in order to be considered valid to be withdrawn(finalized):

```
function _shouldDrawWinner(
        uint256 numberOfParticipants,
        uint256 maximumNumberOfParticipants,
        uint256 roundDepositCount
    ) private pure returns (bool shouldDraw) {
        shouldDraw =
            numberOfParticipants == maximumNumberOfParticipants ||
            (numberOfParticipants > 1 && roundDepositCount == MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND);
    }
```

Now they must wait for the current round to finish in order for them to be able to finalize the next round(2), but in the meantime, a malicious user can extended the withdraw period for the next round. Here is how:

In depositETHIntoMultipleRounds it is possible to get free entries to a round, BUT there is a catch, for example if I want to get free entries for the second(next) round, I would have to construct the input array the following way [1eth, 0], the catch is that there always must be some value > 0, I will explain in detail now:

Back to depositETHIntoMultipleRounds, just keep in mind that we are still in the first round:

```
function depositETHIntoMultipleRounds(uint256[] calldata amounts) external payable nonReentrant whenNotPaused {

        uint256 numberOfRounds = amounts.length;

        if (msg.value == 0 || numberOfRounds == 0) {
            revert ZeroDeposits();
        }

        uint256 startingRoundId = roundsCount;

        Round storage startingRound = rounds[startingRoundId];

        _validateRoundIsOpen(startingRound);

        _setCutoffTimeIfNotSet(startingRound);

        uint256 expectedValue;

        uint256[] memory entriesCounts = new uint256[](numberOfRounds);
 
        for (uint256 i; i < numberOfRounds; ++i) {

            uint256 roundId = _unsafeAdd(startingRoundId, i);

            Round storage round = rounds[roundId];

            uint256 roundValuePerEntry = round.valuePerEntry;

            if (roundValuePerEntry == 0) {
                (, , roundValuePerEntry) = _writeDataToRound({roundId: roundId, roundValue: 0});
            }

            _incrementUserDepositCount(roundId, round);

            uint256 depositAmount = amounts[i];
            
            if (depositAmount % roundValuePerEntry != 0) {
                revert InvalidValue();
            }

            uint256 entriesCount = _depositETH(round, roundId, roundValuePerEntry, depositAmount);

            expectedValue += depositAmount;

            entriesCounts[i] = entriesCount;
        }

        if (expectedValue != msg.value) {
            revert InvalidValue();
        }

        emit MultipleRoundsDeposited(msg.sender, startingRoundId, amounts, entriesCounts);

        if (
            _shouldDrawWinner(
                startingRound.numberOfParticipants,
                startingRound.maximumNumberOfParticipants,
                startingRound.deposits.length
            )
        ) {
            _drawWinner(startingRound, startingRoundId);
        }
        
    }
```

STATE

roundCount = 1 - still active, not ready to be drawn(finished).

NEXT ROUND
Second round - not active, waiting for the current round to finish in order for it to become the current, but it's ready to be withdrawn because roundDepositCount == MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND.

Now when we go back at _shouldDrawWinner we see that in order for a round to be considered valid to be withdrawn it should:

```
shouldDraw =
            numberOfParticipants == maximumNumberOfParticipants ||
            (numberOfParticipants > 1 && roundDepositCount == MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND);
```

Either max participants or max deposits, have currently done the second thing(deposits), keep in mind that they should be roundDepositCount == MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND - !! == !! . So if we somehow can make + 1 deposit this statement won't be true and we would have to somehow get other depositors to deposit in our round in order to reach numberOfParticipants == maximumNumberOfParticipants since like I said if we somehow manage to get more than MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND this will never return true. Well spoiler alert, it's possible.

Back to the depositETHIntoMultipleRounds:

We can actually construct the input uint256[] calldata amounts such way that we can get a free entry for the round.
uint256[] calldata amounts = [1eth, 0] - 1 eth for the first round, 0 for the second - this is completely valid input for this function.

```
uint256 numberOfRounds = amounts.length;

if (msg.value == 0 || numberOfRounds == 0) {
revert ZeroDeposits();
}
```

This if check will pass since we have some eth for the first round and numOfRounds is 2.

Now let's get into the for loop in the function:

```
for (uint256 i; i < numberOfRounds; ++i) {

            uint256 roundId = _unsafeAdd(startingRoundId, i);

            Round storage round = rounds[roundId];

            uint256 roundValuePerEntry = round.valuePerEntry;

            if (roundValuePerEntry == 0) {
                (, , roundValuePerEntry) = _writeDataToRound({roundId: roundId, roundValue: 0});
            }

            _incrementUserDepositCount(roundId, round);

            uint256 depositAmount = amounts[i];
            
            if (depositAmount % roundValuePerEntry != 0) {
                revert InvalidValue();
            }

            uint256 entriesCount = _depositETH(round, roundId, roundValuePerEntry, depositAmount);

            expectedValue += depositAmount;

            entriesCounts[i] = entriesCount;
        }
```

Keep in mind that 1eth is just a value to make things easier, let's assume that it will not fill up the first round and we go to the second already filled up round but with 0 as an amount, we can see that _incrementUserDepositCount will just add us as participants for the round, but then uint256 entriesCount = _depositETH(round, roundId, roundValuePerEntry, depositAmount); will actually push our deposit to the second round with no checks if the maximum value for deposits has been reached!

```
function _depositETH(
        Round storage round,
        uint256 roundId,
        uint256 roundValuePerEntry,
        uint256 depositAmount
    ) private returns (uint256 entriesCount) {

        entriesCount = depositAmount / roundValuePerEntry;

        uint256 roundDepositCount = round.deposits.length;

        _validateOnePlayerCannotFillUpTheWholeRound(_unsafeAdd(roundDepositCount, 1), round.numberOfParticipants);

        uint40 currentEntryIndex = _getCurrentEntryIndexWithoutAccrual(round, roundDepositCount, entriesCount);
        
        // This is equivalent to
        // round.deposits.push(
        //     Deposit({
        //         tokenType: YoloV2__TokenType.ETH,
        //         tokenAddress: address(0),
        //         tokenId: 0,
        //         tokenAmount: msg.value,
        //         depositor: msg.sender,
        //         withdrawn: false,
        //         currentEntryIndex: currentEntryIndex
        //     })
        // );
        // unchecked {
        //     roundDepositCount += 1;
        // }
        uint256 roundDepositsLengthSlot = _getRoundSlot(roundId) + ROUND__DEPOSITS_LENGTH_SLOT_OFFSET;

        uint256 depositDataSlotWithCountOffset = _getDepositDataSlotWithCountOffset(
            roundDepositsLengthSlot,
            roundDepositCount
        );

        // We don't have to write tokenType, tokenAddress, tokenId, and withdrawn because they are 0.
        _writeDepositorAndCurrentEntryIndexToDeposit(depositDataSlotWithCountOffset, currentEntryIndex);
        _writeDepositAmountToDeposit(depositDataSlotWithCountOffset, depositAmount);

        assembly {
            sstore(roundDepositsLengthSlot, add(roundDepositCount, 1))
        }

    }
```

In this function there is no check to prevent more deposits than intended so now our deposits for this round are actually > MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND and in order for this round to be considered as ready to be withdrawn it must fill up with participants. However that's not the problem. So now since our +1 deposit is made this will always return false until the round is filled up with participants:

```
function _shouldDrawWinner(
        uint256 numberOfParticipants,
        uint256 maximumNumberOfParticipants,
        uint256 roundDepositCount
    ) private pure returns (bool shouldDraw) {
        shouldDraw =
            numberOfParticipants == maximumNumberOfParticipants ||
            (numberOfParticipants > 1 && roundDepositCount == MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND);
    }
```

Now assume that the first round has been finalized and _startRound has been called to start the next second round:

```
function _startRound(uint256 _roundsCount) private returns (uint256 roundId) {

        unchecked {
            roundId = _roundsCount + 1;
        }

        roundsCount = uint40(roundId);

        Round storage round = rounds[roundId];
        
        //if there is existing information for this current round, go in
        if (round.valuePerEntry == 0) {
            
            //create the additional info in order to get this round going
            // On top of the 4 values covered by _writeDataToRound, this also writes the round's status to Open (1).
            _writeDataToRound({roundId: roundId, roundValue: 1});

            emit RoundStatusUpdated(roundId, RoundStatus.Open);

        } else {

            uint256 numberOfParticipants = round.numberOfParticipants;

            if (
                !paused() &&
                _shouldDrawWinner(numberOfParticipants, round.maximumNumberOfParticipants, round.deposits.length)
            ) {

                _drawWinner(round, roundId);

            }
            
            
             else {

                uint40 _roundDuration = roundDuration;

                // This is equivalent to
                // round.status = RoundStatus.Open;
                // if (round.numberOfParticipants > 0) {
                //   round.cutoffTime = uint40(block.timestamp) + _roundDuration;
                // }
                uint256 roundSlot = _getRoundSlot(roundId);

                assembly {

                    // RoundStatus.Open is equal to 1.
                    let roundValue := or(sload(roundSlot), 1)

                    if gt(numberOfParticipants, 0) {
                        roundValue := or(roundValue, shl(ROUND__CUTOFF_TIME_OFFSET, add(timestamp(), _roundDuration)))
                    }

                    sstore(roundSlot, roundValue)
                }

                emit RoundStatusUpdated(roundId, RoundStatus.Open);
            }
        }
    }
```

Now the problem here is that round.valuePerEntry has already been set for that round, then the second if will be skipped too because _shouldDrawWinner will always return false now until maxParticipants is met, and we go straight into the else which will:

```
           else {

                uint40 _roundDuration = roundDuration;

                // This is equivalent to
                // round.status = RoundStatus.Open;
                // if (round.numberOfParticipants > 0) {
                //   round.cutoffTime = uint40(block.timestamp) + _roundDuration;
                // }
                uint256 roundSlot = _getRoundSlot(roundId);

                assembly {

                    // RoundStatus.Open is equal to 1.
                    let roundValue := or(sload(roundSlot), 1)

                    if gt(numberOfParticipants, 0) {
                        roundValue := or(roundValue, shl(ROUND__CUTOFF_TIME_OFFSET, add(timestamp(), _roundDuration)))
                    }

                    sstore(roundSlot, roundValue)
                }

                emit RoundStatusUpdated(roundId, RoundStatus.Open);
            }
        }
    }

```

And now we have an over-fullfilled round with deposits, which has an round.cutoffTime = uint40(block.timestamp) + _roundDuration; and is OPEN.

This will temporary brick the round since there is 2 ways in order to be finalized - either more recipients should join in order to make this statement return true numberOfParticipants == maximumNumberOfParticipants(which might not be incentivizing for the people to join) or the whole roundDuration should pass in order to invoke drawWinner.



# Impact
User can forcefully extend the finalizing of a round.


# Tool used
Manual Review

# Recommendation
Make sure in _depositETH to check if the deposit limit is reached
