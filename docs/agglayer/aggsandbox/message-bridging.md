# Message Bridging Guide

**Complete guide to cross-chain contract calls using AggSandbox.**

Message bridging enables cross-chain contract execution, allowing you to trigger smart contract functions across different networks. Unlike asset bridging which transfers tokens, message bridging transfers executable data and can trigger contract interactions on the destination network.

## Overview

Message bridging in Agglayer follows a **bridge-and-execute** model:

1. **Source Network**: Message data is encoded and bridged
2. **Exit Root Update**:
    - **If L1 source**: Direct Global Exit Root update on L1
    - **If L2 source**: Local Exit Root update, then AggKit updates GER on L1
3. **Bridge Service**: AggKit syncs GER to destination network
4. **Destination Network**: Contract function is executed with the data
5. **Claim Process**: User triggers the contract execution

## Message Bridge Process

Message bridging follows the same process for all network combinations. Use these steps for any $LX (source) → $LY (destination) bridge operation.

### Step 1: Bridge Message

Bridge message from source network ($LX) to destination network ($LY):

```bash
# Bridge message from $LX to $LY
aggsandbox bridge message \
  --network-id $LX \
  --destination-network-id $LY \
  --to-address $CONTRACT_ADDRESS \
  --call-data $CALL_DATA \
```

**Network ID Reference:**

- **L1 (Ethereum)**: `--network-id 0`
- **L2-1 (zkEVM)**: `--network-id 1` 
- **L2-2 (Additional)**: `--network-id 2` (requires `--multi-l2` mode)

**Variable Reference:**

- `$LX` = Source network ID (0, 1, or 2)
- `$LY` = Destination network ID (0, 1, or 2)

### Step 2: Monitor Bridge Transaction

```bash
# Check bridge status on source network
aggsandbox show bridges --network-id $LX
```

### Step 3: Prepare Call Data

```bash
# Example: Encode message data (this becomes the 'data' parameter in onMessageReceived)
# The bridge will automatically call onMessageReceived with your encoded data
MESSAGE="Hello from $LX!"
CALL_DATA=$(cast abi-encode "string" "$MESSAGE")
echo "Call data: $CALL_DATA"

# Alternative: Encode complex data structures
# CALL_DATA=$(cast abi-encode "(string,uint256)" "Hello" 42)
```

**Important**: The `--call-data` you provide becomes the `data` parameter in `onMessageReceived`. The Unified Bridge automatically calls this function - you don't need to encode the function signature.

### Step 4: Check if Claimable

First, manually check the destination network claims API:

```bash
# Wait for AggKit sync, then check destination claims
# L1<->L2: ~20-25 seconds
# L2<->L2: ~45-60 seconds
aggsandbox show claims --network-id $LY
```

**Look for your bridge transaction in the output:**
```json
[
  {
    "bridge_tx_hash": "0x8d1b60d0eaab6f609955bdd371e8004f47349cc809ff1bee81dc9d37237a031c",
    "claim_tx_hash": "",
    "origin_network": $LX,
    "destination_network": $LY,
    "amount": "0",
    "status": "pending",
    "type": "message"
  },
]
```

**What to look for:**

- Your bridge transaction hash appears as `bridge_tx_hash`
- `origin_network` matches your source ($LX)
- `destination_network` matches your destination ($LY)
- Status is `"pending"` (ready to claim)
- Type is `"message"` for message bridging
- Amount is `"0"` for pure message bridges

For automated scripts, use this pattern:

```bash
BRIDGE_TX="0x8d1b60d0eaab6f609955bdd371e8004f47349cc809ff1bee81dc9d37237a031c"
DESTINATION_NETWORK=$LY

echo "Checking if bridge $BRIDGE_TX is claimable on network $DESTINATION_NETWORK..."

CLAIMABLE=$(aggsandbox show claims --network-id $DESTINATION_NETWORK --json | \
  jq -r --arg tx "$BRIDGE_TX" '.[] | select(.bridge_tx_hash == $tx and .status == "pending") | .bridge_tx_hash')

if [ "$CLAIMABLE" = "$BRIDGE_TX" ]; then
    echo "✅ Bridge is claimable (found in pending claims)"
else
    echo "⏳ Bridge not yet claimable, waiting..."
    # Adjust sleep time based on network combination
    if [ $DESTINATION_NETWORK -eq 2 ] || [ $SOURCE_NETWORK -eq 2 ]; then
        sleep 15  # L2-L2 takes longer
    else
        sleep 10  # L1-L2 timing
    fi
    # Check again
    aggsandbox show claims --network-id $DESTINATION_NETWORK --json | jq --arg tx "$BRIDGE_TX" '.[] | select(.bridge_tx_hash == $tx)'
fi
```

### Step 5: Claim Message

```bash
# Claim message on destination network
aggsandbox bridge claim \
  --network-id $LY \
  --tx-hash $BRIDGE_TX \
  --source-network-id $LX
```

### Step 6: Verify Claim Completion

First, manually check the destination network claims API:

```bash
# Check destination claims to find your claim transaction
aggsandbox show claims --network-id $LY
```

**Look for your claim transaction in the output:**
```json
[
  {
    "bridge_tx_hash": "0x8d1b60d0eaab6f609955bdd371e8004f47349cc809ff1bee81dc9d37237a031c",
    "claim_tx_hash": "0xa9fa5418144f7c8c1b78cd0e5560d6550411667ef937b554636a613f933b3d9f",
    "origin_network": $LX,
    "destination_network": $LY,
    "amount": "0",
    "status": "completed",
    "type": "message"
  }
]
```

**What to look for:**

- Your claim transaction hash appears as `claim_tx_hash`
- Status changed from `"pending"` to `"completed"`
- The claim is fully processed when status is `"completed"`

For automated scripts, use this pattern:

```bash
CLAIM_TX="0xa9fa5418144f7c8c1b78cd0e5560d6550411667ef937b554636a613f933b3d9f"
DESTINATION_NETWORK=$LY

echo "Monitoring claim status until completion..."

while true; do
    CLAIM_DATA=$(aggsandbox show claims --network-id $DESTINATION_NETWORK --json | \
      jq --arg tx "$CLAIM_TX" '.[] | select(.claim_tx_hash == $tx)')
    
    if [ "$CLAIM_DATA" != "null" ] && [ "$CLAIM_DATA" != "" ]; then
        CLAIM_STATUS=$(echo "$CLAIM_DATA" | jq -r '.status')
        echo "$(date): Claim status: $CLAIM_STATUS"
        
        if [ "$CLAIM_STATUS" = "completed" ]; then
            echo "✅ Claim completed successfully!"
            break
        elif [ "$CLAIM_STATUS" = "pending" ]; then
            echo "⏳ Claim still pending, waiting 5 seconds..."
            sleep 5
        else
            echo "❌ Unknown claim status: $CLAIM_STATUS"
            break
        fi
    else
        echo "❌ Claim not found"
        break
    fi
done
```

### Step 7: Verify Contract Execution

```bash
# Check if contract function was executed (depends on your contract)
cast call $CONTRACT_ADDRESS \
  "lastMessage()(string)" \
  --rpc-url http://localhost:$PORT

# Check contract state changes
echo "Contract verification complete"
```

**RPC Port Reference:**

- **L1**: `http://localhost:8545`
- **L2-1**: `http://localhost:8546`
- **L2-2**: `http://localhost:8547`

## Network Combinations

### L1 <-> L2 Bridging
```bash
# L1 → L2-1: LX=0, LY=1
# L2-1 → L1: LX=1, LY=0
# Sync time: ~20-25 seconds
```

### L2 <-> L2 Bridging
```bash
# Start multi-L2 mode first
aggsandbox stop
aggsandbox start --multi-l2 --detach

# L2-1 → L2-2: LX=1, LY=2
# L2-2 → L2-1: LX=2, LY=1
# Sync time: ~45-60 seconds (longer for L2-L2)
```

## Message Bridging with ETH

### Bridge Message with ETH Value

```bash
# Bridge message with 0.1 ETH
aggsandbox bridge message \
  --network-id 0 \
  --destination-network-id 1 \
  --to-address $CONTRACT_ADDRESS \
  --call-data $CALL_DATA \
  --msg-value 100000000000000000 \
  --private-key $PRIVATE_KEY

# When claiming, include the ETH value
aggsandbox bridge claim \
  --network-id 1 \
  --tx-hash $BRIDGE_TX \
  --source-network-id 0 \
  --msg-value 100000000000000000
```

**Note**: Use wei values for `--msg-value` (1 ETH = 1000000000000000000 wei).

## Smart Contract Integration

### Required Interface

**CRITICAL**: All contracts receiving bridged messages MUST implement the `IBridgeMessageReceiver` interface:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity ^0.8.22;

/**
 * @dev Define interface for PolygonZkEVM Bridge message receiver
 */
interface IBridgeMessageReceiver {
    function onMessageReceived(address originAddress, uint32 originNetwork, bytes memory data) external payable;
}
```

When you claim a message, the Unified Bridge automatically calls `onMessageReceived` on your contract with:

- `originAddress`: The address that initiated the bridge on the source network
- `originNetwork`: The source network ID (0, 1, or 2)
- `data`: The call data you specified in the bridge message command

### Receiving Contract Example

Here's a complete contract that implements the required interface:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.22;

interface IBridgeMessageReceiver {
    function onMessageReceived(address originAddress, uint32 originNetwork, bytes memory data) external payable;
}

contract MessageReceiver is IBridgeMessageReceiver {
    string public lastMessage;
    uint256 public messageCount;
    address public lastSender;
    uint32 public lastOriginNetwork;
    mapping(address => uint256) public senderCounts;
    
    event MessageReceived(address indexed originAddress, uint32 originNetwork, string message);
    
    // This function is called automatically by the Unified Bridge
    function onMessageReceived(
        address originAddress, 
        uint32 originNetwork, 
        bytes memory data
    ) external payable override {
        // Decode the message from the bridge data
        string memory message = abi.decode(data, (string));
        
        // Store the message details
        lastMessage = message;
        lastSender = originAddress;
        lastOriginNetwork = originNetwork;
        messageCount++;
        senderCounts[originAddress]++;
        
        emit MessageReceived(originAddress, originNetwork, message);
    }
    
    // Helper function to get message details
    function getLastMessageDetails() external view returns (
        string memory message,
        address sender,
        uint32 originNetwork,
        uint256 count
    ) {
        return (lastMessage, lastSender, lastOriginNetwork, messageCount);
    }
}
```

### Deploy and Test

```bash
# 1. Deploy contract on destination network
CONTRACT_ADDRESS=$(forge create src/MessageReceiver.sol:MessageReceiver \
  --rpc-url http://localhost:8546 \
  --private-key $PRIVATE_KEY \
  --json | jq -r '.deployedTo')

echo "Contract deployed to: $CONTRACT_ADDRESS"

# 2. Prepare message data (will be passed to onMessageReceived)
MESSAGE="Hello from bridge!"
CALL_DATA=$(cast abi-encode "string" "$MESSAGE")

# 3. Bridge the message
BRIDGE_TX=$(aggsandbox bridge message \
  --network-id 0 \
  --destination-network-id 1 \
  --to-address $CONTRACT_ADDRESS \
  --call-data "$CALL_DATA" \
  --private-key $PRIVATE_KEY | \
  grep "bridge transaction submitted:" | cut -d' ' -f4)

echo "Bridge transaction: $BRIDGE_TX"

# 4. Wait and claim (this will call onMessageReceived automatically)
sleep 25
aggsandbox bridge claim \
  --network-id 1 \
  --tx-hash $BRIDGE_TX \
  --source-network-id 0

# 5. Verify the message was received through onMessageReceived
cast call $CONTRACT_ADDRESS "lastMessage()(string)" --rpc-url http://localhost:8546
cast call $CONTRACT_ADDRESS "getLastMessageDetails()(string,address,uint32,uint256)" --rpc-url http://localhost:8546
```

## Troubleshooting

### **Common Issues**

#### Bridge Transaction Fails
```bash
# Check account balance for gas
cast balance $YOUR_ADDRESS --rpc-url http://localhost:8545

# Check contract exists on destination
cast code $CONTRACT_ADDRESS --rpc-url http://localhost:8546

# Check network status
aggsandbox status
```

#### Claim Fails
```bash
# Verify bridge was indexed
aggsandbox show bridges --network-id 0 | grep $BRIDGE_TX

# Check if already claimed
aggsandbox bridge utils is-claimed --network 1 --index 42 --source-network 0

# Check AggKit sync status
aggsandbox logs aggkit
```

#### Function Not Executed
```bash
# Test function call locally
cast call $CONTRACT_ADDRESS \
  --data "$CALL_DATA" \
  --rpc-url http://localhost:8546

# Check contract interface
cast interface $CONTRACT_ADDRESS --rpc-url http://localhost:8546

# Check transaction receipt
cast receipt $CLAIM_TX --rpc-url http://localhost:8546
```
