![](https://audits.sherlock.xyz/_next/image?url=https%3A%2F%2Fsherlock-files.ams3.digitaloceanspaces.com%2Fgitcoin.jpg&w=256&q=75)

# [AlloV2](https://audits.sherlock.xyz/contests/109)

| Protocol | Contest Pot | nSLOC | Length | Start | End |
|:--:|:--:|:--:|:--:|:--:|:--:|
| AlloV2 | 58,400 USDC | 1,648 | 10 days | Sept 11 2023 | Sept 21 2023 |

## Issues found :

| Severity | Title |
|:--|:--:|
| High | Approved allocator can send as many votes as he wants to an accepted recipient |
| Medium | Registering a recipient for a RFPSimpleStrategy while useRegistryAnchor is true will always revert |

# 1. Approved allocator can send as many votes as he wants to an accepted recipient.

# Summary

In QVSimpleStrategy.sol an approved allocator can allocate votes to an accepted recipient.


# Vulnerability Detail

The problem here is that an allocator can allocate as many votes as he wants since after he gave a recipient votes his voiceCredits doesn't increase.

Allocator's struct:

```
   /// @notice The details of the allocator
    struct Allocator {
        uint256 voiceCredits;
        mapping(address => uint256) voiceCreditsCastToRecipient;
        mapping(address => uint256) votesCastToRecipient;
    }
```

When an allocator wants to allocate votes to a recipient he calls allocate on the Allo.sol contract, which then will call _allocate in QVSimpleStrategy:

```
function _allocate(bytes memory _data, address _sender) internal virtual override {

        (address recipientId, uint256 voiceCreditsToAllocate) = abi.decode(_data, (address, uint256));

        // spin up the structs in storage for updating
        Recipient storage recipient = recipients[recipientId];
        Allocator storage allocator = allocators[_sender];

        // check that the sender can allocate votes
        if (!_isValidAllocator(_sender)) revert UNAUTHORIZED();

        // check that the recipient is accepted
        if (!_isAcceptedRecipient(recipientId)) revert RECIPIENT_ERROR(recipientId);

        // check that the recipient has voice credits left to allocate
        if (!_hasVoiceCreditsLeft(voiceCreditsToAllocate, allocator.voiceCredits)) revert INVALID();

        _qv_allocate(allocator, recipient, recipientId, voiceCreditsToAllocate, _sender);
    }
```
_hasVoiceCreditsLeft is used to check if an allocator has voiceCredits left. In _allocate above _hasVoiceCreditsLeft is used the following way:
first param is the arbitrary amount an allocator wants to allocate.
second param is the allocator.voiceCredits

So in order for an allocator to have voiceCredits, the sum between voiceCreditsToAllocate and allocator.voiceCredits shouldn't be bigger than the maxVoiceCreditsPerAllocator

```
function _hasVoiceCreditsLeft(uint256 _voiceCreditsToAllocate, uint256 _allocatedVoiceCredits)
        internal
        view
        override
        returns (bool)
    {
        return _voiceCreditsToAllocate + _allocatedVoiceCredits <= maxVoiceCreditsPerAllocator;
    }
```

Then _qv_allocate is called in QVBaseStrategy.sol:

```
  function _qv_allocate(
        Allocator storage _allocator,
        Recipient storage _recipient,
        address _recipientId,
        uint256 _voiceCreditsToAllocate,
        address _sender
    ) internal onlyActiveAllocation {
        // check the `_voiceCreditsToAllocate` is > 0
        if (_voiceCreditsToAllocate == 0) revert INVALID();

        // get the previous values
        uint256 creditsCastToRecipient = _allocator.voiceCreditsCastToRecipient[_recipientId];
        uint256 votesCastToRecipient = _allocator.votesCastToRecipient[_recipientId];

        // get the total credits and calculate the vote result
        uint256 totalCredits = _voiceCreditsToAllocate + creditsCastToRecipient;
        uint256 voteResult = _sqrt(totalCredits * 1e18);

        // update the values
        voteResult -= votesCastToRecipient;
        totalRecipientVotes += voteResult;
        _recipient.totalVotesReceived += voteResult;

        _allocator.voiceCreditsCastToRecipient[_recipientId] += totalCredits;
        _allocator.votesCastToRecipient[_recipientId] += voteResult;

        // emit the event with the vote results
        emit Allocated(_recipientId, voteResult, _sender);
    }
```

_voiceCreditsToAllocate is an arbitrary amount an allocator wishes to allocate to a recipient, as you can see the votes a recipient will receive are calculated like that:

```
uint256 totalCredits = _voiceCreditsToAllocate + creditsCastToRecipient;

uint256 voteResult = _sqrt(totalCredits * 1e18);
```

Meaning _voiceCreditsToAllocate is the arbitrary amount and creditsCastToRecipient is the already allocated votes to a recipient by the allocator. After that the voteResult is calculated. The problem here is that the allocator's voiceCredits are not increased by the _voiceCreditsToAllocate, which means that the _hasVoiceCreditsLeft check will always pass and allocator will have infinite amount of voiceCredits to allocate.

# Impact

Allocator can give an infinite amount of votes to a recipient

# Tool used
Manual Review

# Recommendation
Increase the voiceCredits of the allocator with the amount he wishes to allocate a recipient



# 2. Registering a recipient for a RFPSimpleStrategy while useRegistryAnchor is true will always revert.

# Summary
Whenever a recipient is being registered for a strategy, there is a variable deciding if the registry's anchor should be used or a recipient address of choice. For the RFPSimpleStrategy, the variable useRegistryAnchor is used to decide if a registry's anchor should be used or a recipient address.


# Vulnerability Detail
However when registering recipients to RFPSimpleStrategy and useRegistryAnchor is set to true, their registration will always revert.

```
 function _registerRecipient(bytes memory _data, address _sender)
        internal
        override
        onlyActivePool
        returns (address recipientId)
    {
        bool isUsingRegistryAnchor;
        address recipientAddress;
        address registryAnchor;
        uint256 proposalBid;
        Metadata memory metadata;

        // Decode '_data' depending on the 'useRegistryAnchor' flag
        if (useRegistryAnchor) {

            //@audit when useRegistryAnchor is true, no recipientAddress is set

            /// @custom:data when 'true' -> (address recipientId, uint256 proposalBid, Metadata metadata)
            (recipientId, proposalBid, metadata) = abi.decode(_data, (address, uint256, Metadata));

            // If the sender is not a profile member this will revert
            if (!_isProfileMember(recipientId, _sender)) revert UNAUTHORIZED();

        } else {
            //  @custom:data when 'false' -> (address recipientAddress, address registryAnchor, uint256 proposalBid, Metadata metadata)
            (recipientAddress, registryAnchor, proposalBid, metadata) =
                abi.decode(_data, (address, address, uint256, Metadata));

            // Check if the registry anchor is valid so we know whether to use it or not
            isUsingRegistryAnchor = registryAnchor != address(0);

            // Ternerary to set the recipient id based on whether or not we are using the 'registryAnchor' or '_sender'
            recipientId = isUsingRegistryAnchor ? registryAnchor : _sender;
            
            // Checks if the '_sender' is a member of the profile 'anchor' being used and reverts if not
            if (isUsingRegistryAnchor && !_isProfileMember(recipientId, _sender)) revert UNAUTHORIZED();
        }

        // Check if the metadata is required and if it is, check if it is valid, otherwise revert
        if (metadataRequired && (bytes(metadata.pointer).length == 0 || metadata.protocol == 0)) {
            revert INVALID_METADATA();
        }

        if (proposalBid > maxBid) {
            // If the proposal bid is greater than the max bid this will revert
            revert EXCEEDING_MAX_BID();
        } else if (proposalBid == 0) {
            // If the proposal bid is 0, set it to the max bid
            proposalBid = maxBid;
        }
        
        //@audit when useRegistryAnchor is set to true, no recipientAddress will be set so this will always revert
        // If the recipient address is the zero address this will revert
        if (recipientAddress == address(0)) revert RECIPIENT_ERROR(recipientId);

        // Get the recipient
        Recipient storage recipient = _recipients[recipientId];

        if (recipient.recipientStatus == Status.None) {
            // If the recipient status is 'None' add the recipient to the '_recipientIds' array
            _recipientIds.push(recipientId);

            emit Registered(recipientId, _data, _sender);
        } else {
            emit UpdatedRegistration(recipientId, _data, _sender);
        }

        // update the recipients data
        recipient.recipientAddress = recipientAddress;
        recipient.useRegistryAnchor = isUsingRegistryAnchor ? true : recipient.useRegistryAnchor;
        recipient.proposalBid = proposalBid;
        recipient.recipientStatus = Status.Pending;
    }
```

As you can see by the @Audit tags, when useRegistryAnchor is set to true, no recipientAddress address is set, then the tx will be reverted because of this check:

`if (recipientAddress == address(0)) revert RECIPIENT_ERROR(recipientId);`


# Impact
Registering a recipient when useRegistryAnchor is set to true, will always revert.


# Tool used
Manual Review

# Recommendation
The problem is only present at this particular strategy, look at the other strategies for a proper set up.
