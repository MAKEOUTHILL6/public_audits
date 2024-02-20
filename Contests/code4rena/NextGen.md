![](https://code4rena.com/_next/image?url=https%3A%2F%2Fstorage.googleapis.com%2Fcdn-c4-uploads-v0%2Fuploads%2FneYKTfpiuGS.0&w=256&q=75)

# [NextGen](https://code4rena.com/reports/2023-10-nextgen)

| Protocol | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|
| NextGen | 60,500 USDC | 1265 | 14 days | Oct 30 2023 | Nov 13 2023 |

## Issues found :

| Severity | Title |
|:--|:--:|
| High | All participants in an auction can steal money from the protocol. |
| Medium | Possible loss of funds for bidders trying to participate in an auction. |

# 1. All participants in an auction can steal money from the protocol.

# Impact

An auction winner can get the won NFT for free.

A bidder can get his refund twice, stealing funds from users.


# Proof of Concept

Let's take a look at the `claimAuction` function.

```
function claimAuction(uint256 _tokenid) public WinnerOrAdminRequired(_tokenid,this.claimAuction.selector){
        require(block.timestamp >= minter.getAuctionEndTime(_tokenid) && auctionClaim[_tokenid] == false && minter.getAuctionStatus(_tokenid) == true);
        auctionClaim[_tokenid] = true;
        uint256 highestBid = returnHighestBid(_tokenid);
        address ownerOfToken = IERC721(gencore).ownerOf(_tokenid);
        address highestBidder = returnHighestBidder(_tokenid);
        for (uint256 i=0; i< auctionInfoData[_tokenid].length; i ++) {
            if (auctionInfoData[_tokenid][i].bidder == highestBidder && auctionInfoData[_tokenid][i].bid == highestBid && auctionInfoData[_tokenid][i].status == true) {
                IERC721(gencore).safeTransferFrom(ownerOfToken, highestBidder, _tokenid);
                (bool success, ) = payable(owner()).call{value: highestBid}("");
                emit ClaimAuction(owner(), _tokenid, success, highestBid);
            } else if (auctionInfoData[_tokenid][i].status == true) {
                (bool success, ) = payable(auctionInfoData[_tokenid][i].bidder).call{value: auctionInfoData[_tokenid][i].bid}("");
                emit Refund(auctionInfoData[_tokenid][i].bidder, _tokenid, success, highestBid);
            } else {}
        }
    }
```

The whole idea is that, a winner can claim his won NFT when block.timestamp >= minter.getAuctionEndTime(_tokenid) - mind that it's >=, and he has the the highest bid, so the function has a for loop going through all bids that were present for this tokenID.

Now let's look at the cancelBid function:

```
function cancelBid(uint256 _tokenid, uint256 index) public {
        require(block.timestamp <= minter.getAuctionEndTime(_tokenid), "Auction ended");
        require(auctionInfoData[_tokenid][index].bidder == msg.sender && auctionInfoData[_tokenid][index].status == true);
        auctionInfoData[_tokenid][index].status = false;
        (bool success, ) = payable(auctionInfoData[_tokenid][index].bidder).call{value: auctionInfoData[_tokenid][index].bid}("");
        emit CancelBid(msg.sender, _tokenid, index, success, auctionInfoData[_tokenid][index].bid);
    }
```

It checks if block.timestamp <= minter.getAuctionEndTime(_tokenid) - mind that it's <=. Then finds the msg.sender's bid and refunds it, also setting his status to false so he can't call it again and get another refund, but that actually does not hold.

As you can see cancelling a bid and claiming a bid can happen in the same block.timestamp, because of the = sign present in both <= & >=.

Scenario on how a bidder can get refunded twice:

It's all about timing, a bidder can time when an auction winner has called claimAuction, and bundle his cancelBid in the same block, but after claimAuction. Getting the refund twice is possible because in claimAuction whenever the winner calls it, it loops through all participants, and if they are not the highest bidder(the winner), they get their bid refunded, but the status is not set to false as you can see in claimAuction's code snippet above, so the bidder can then call cancelBid, and get his refund one more time because this check will pass require(auctionInfoData[_tokenid][index].bidder == msg.sender && auctionInfoData[_tokenid][index].status == true);.
That way a bidder can steal non-refunded amount of ETH in the contract. Also this attack can be made even easier if an auction winner participates in the bid with a second account so he knows exactly when the claimAuction will be called and again take advantage of other user's refunds.

If there is enough amount of ETH in the contract to cover the auction winner's bid, he can even get the NFT for free.



# Tool used
Manual Review

# Recommendation
In `claimAuction` make the following changes:

`--require(block.timestamp >= minter.getAuctionEndTime(_tokenid) && auctionClaim[_tokenid] == false && minter.getAuctionStatus(_tokenid) == true);`

`++require(block.timestamp > minter.getAuctionEndTime(_tokenid) && auctionClaim[_tokenid] == false && minter.getAuctionStatus(_tokenid) == true);`


# 2. Possible loss of funds for bidders trying to participate in an auction.

# Impact

A bidder can bid for an auction the same time an auction is being finalized(claimed) and depending on which is executed first, it's possible for a bidder to lose his funds.

# Proof of Concept

```
function participateToAuction(uint256 _tokenid) public payable {
        require(msg.value > returnHighestBid(_tokenid) && block.timestamp <= minter.getAuctionEndTime(_tokenid) && minter.getAuctionStatus(_tokenid) == true);
        auctionInfoStru memory newBid = auctionInfoStru(msg.sender, msg.value, true);
        auctionInfoData[_tokenid].push(newBid);
    }
```

Participating in an auction is available when msg.value > returnHighestBid(_tokenid) && block.timestamp <= minter.getAuctionEndTime(_tokenid) && minter.getAuctionStatus(_tokenid) == true.

The important detail here is that: block.timestamp <= minter.getAuctionEndTime(_tokenid) - keep in mind it's <=.

```
function claimAuction(uint256 _tokenid) public WinnerOrAdminRequired(_tokenid,this.claimAuction.selector){
        require(block.timestamp >= minter.getAuctionEndTime(_tokenid) && auctionClaim[_tokenid] == false && minter.getAuctionStatus(_tokenid) == true);
        auctionClaim[_tokenid] = true;
        uint256 highestBid = returnHighestBid(_tokenid);
        address ownerOfToken = IERC721(gencore).ownerOf(_tokenid);
        address highestBidder = returnHighestBidder(_tokenid);
        for (uint256 i=0; i< auctionInfoData[_tokenid].length; i ++) {
            if (auctionInfoData[_tokenid][i].bidder == highestBidder && auctionInfoData[_tokenid][i].bid == highestBid && auctionInfoData[_tokenid][i].status == true) {
                IERC721(gencore).safeTransferFrom(ownerOfToken, highestBidder, _tokenid);
                (bool success, ) = payable(owner()).call{value: highestBid}("");
                emit ClaimAuction(owner(), _tokenid, success, highestBid);
            } else if (auctionInfoData[_tokenid][i].status == true) {
                (bool success, ) = payable(auctionInfoData[_tokenid][i].bidder).call{value: auctionInfoData[_tokenid][i].bid}("");
                emit Refund(auctionInfoData[_tokenid][i].bidder, _tokenid, success, highestBid);
            } else {}
        }
    }
```

Claiming an auction can be done when block.timestamp >= minter.getAuctionEndTime(_tokenid) - again keep in mind it's <=, which means participating and claiming an auction can be done in the same block.timestamp which can be a problem depending on which transaction is executed first, because if the claimAuction is executed before participateToAuction and in the same block.timestamp, bidder's bid will be lost because there are no checks to see in participateToAuction if an auction has been claimed, it only checks if they are in the same block.timestamp. So participateToAuction WILL NOT revert and user will lose their bid, since there is no way to refund it.

# Tool used
Manual Review

# Recommendation
In `claimAuction` make the following changes:

`-- require(block.timestamp >= minter.getAuctionEndTime(_tokenid) && auctionClaim[_tokenid] == false && minter.getAuctionStatus(_tokenid) == true);`

`++ require(block.timestamp > minter.getAuctionEndTime(_tokenid) && auctionClaim[_tokenid] == false && minter.getAuctionStatus(_tokenid) == true);`



