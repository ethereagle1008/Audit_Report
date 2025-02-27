# Title: `sendMessage` function of `AvailBridge` contract doesn't refund left ether to users.

## Summary
`sendMessage` function of `AvailBridge` contract calculates the fee value for message data and revert if paid eth is less than it. But when the paid ether is greater than fee value, it doesn't refund ether.

## Vulnerability Detail
Fee values for sending message is calculated as the length of message data multiplied by `feePerByte`.
```javascript
    function getFee(uint256 length) public view returns (uint256) {
        return length * feePerByte;
    }
```
`feePerByte` is updated by admin during using protocol.
```javascript
    function updateFeePerByte(uint256 newFeePerByte) external onlyRole(DEFAULT_ADMIN_ROLE) {
        feePerByte = newFeePerByte;
    }
```
Therefore, if updating `feePerByte` frontruns for `sendMessage`, calculated fee may be less than paid ether.
But `sendMessage` doesn't refund left ether and it makes users lose their fund to should be received back.

## Impact
`sendMessage` doesn't refund left ether after paying fee and users may lose their fund to should be received back.

## Code Snippet

https://github.com/sherlock-audit/2023-12-avail-biginfo2012/blob/098a33fab3a96c4d1d37d81c5fbbd1be613f1a45/contracts/src/AvailBridge.sol#L300-L321

## Tool Used

Manual Review

## Recommendation

Add logic that refunds the left ether after paying fee to `sendMessage` function.
```diff
    function sendMessage(bytes32 recipient, bytes calldata data) external payable whenNotPaused {
        uint256 length = data.length;
        if (length >= MAX_DATA_LENGTH) {
            revert ExceedsMaxDataLength();
        }
        // ensure that fee is above minimum amount
        if (msg.value < getFee(length)) {
            revert FeeTooLow();
        }

+       if (msg.value > getFee(length)) {
+       	(bool success, ) = payable(msg.sender).call{value: msg.value - getFee(length)}("");
+       	if (!success) {
+       		revert ;
+       	}
+       }
        uint256 id;
        unchecked {
            id = messageId++;
        }
        fees += msg.value;
        Message memory message = Message(
            MESSAGE_TX_PREFIX, bytes32(bytes20(msg.sender)), recipient, ETH_DOMAIN, AVAIL_DOMAIN, data, uint64(id)
        );
        // store message hash to be retrieved later by our light client
        isSent[id] = keccak256(abi.encode(message));

        emit MessageSent(msg.sender, recipient, id);
    }
```