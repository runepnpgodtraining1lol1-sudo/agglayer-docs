# Bridge-and-Call Guide

**Complete guide to atomic asset transfer with contract execution using AggSandbox.**

Bridge-and-call is an atomic operation that combines asset bridging with contract execution in a single transaction. It transfers tokens to a contract on the destination network and immediately executes a function call with those tokens, enabling complex cross-chain interactions like DeFi operations, NFT minting, or automated workflows.

## Overview

Bridge-and-call in Agglayer follows a **dual-bridge** model:

1. **Source Network**: Creates both asset and message bridges atomically
2. **Exit Root Update**:
    - **If L1 source**: Direct Global Exit Root update on L1
    - **If L2 source**: Local Exit Root update, then AggKit updates GER on L1
3. **Bridge Service**: AggKit syncs GER to destination network
4. **Destination Network**: Asset bridge transfers tokens, message bridge executes function
5. **Claim Process**: User claims both bridges independently (asset first, then message)

## Bridge-and-Call Process

Bridge-and-call follows the same process for all network combinations. Use these steps for any $LX (source) → $LY (destination) bridge operation.

### Step 1: Bridge-and-Call

Execute atomic bridge-and-call from source network ($LX) to destination network ($LY):

```bash
# Bridge-and-call from $LX to $LY
aggsandbox bridge bridge-and-call \
  --network-id $LX \
  --destination-network-id $LY \
  --amount $AMOUNT \
  --token-address $TOKEN_ADDRESS \
  --to-address $CONTRACT_ADDRESS \
  --call-data $CALL_DATA \
  --private-key $PRIVATE_KEY
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
# Check bridge status on source network (creates 2 bridges)
aggsandbox show bridges --network-id $LX
```

You'll see two bridges with consecutive deposit counts:

- **Asset Bridge**: `deposit_count: N`, `leaf_type: 0` (asset)
- **Message Bridge**: `deposit_count: N+1`, `leaf_type: 1` (message)

### Step 3: Prepare Call Data

```bash
# Example: Encode function call that JumpPoint will execute on your contract
TOKEN_ADDRESS="0x5FbDB2315678afecb367f032d93F642f64180aa3"
AMOUNT="1000000000000000000"
MESSAGE="Bridge-and-call from $LX!"

CALL_DATA=$(cast calldata "receiveTokensWithMessage(address,uint256,string)" \
  $TOKEN_ADDRESS $AMOUNT "$MESSAGE")
echo "Call data: $CALL_DATA"

# Alternative: Any function on your target contract
# CALL_DATA=$(cast calldata "processPayment(uint256,bytes32)" $AMOUNT 0x1234...)
```

**Important**: The `--call-data` is the actual function call that the `JumpPoint` contract will execute on your target contract. This should be a complete function call with signature and parameters.

### Step 4: Check if Claimable

First, manually check the destination network claims API:

```bash
# Wait for AggKit sync, then check destination claims
# L1<->L2: ~20-25 seconds
# L2<->L2: ~45-60 seconds
aggsandbox show claims --network-id $LY
```

**Look for both bridge transactions in the output:**
```json
[
  {
    "bridge_tx_hash": "0x8d1b60d0eaab6f609955bdd371e8004f47349cc809ff1bee81dc9d37237a031c",
    "claim_tx_hash": "",
    "origin_network": $LX,
    "destination_network": $LY,
    "amount": "1000000000000000000",
    "status": "pending",
    "type": "asset"
  },
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

- Both bridges share the same `bridge_tx_hash`
- Asset bridge: `type: "asset"`, amount > 0
- Message bridge: `type: "message"`, amount = 0
- Both have status `"pending"` (ready to claim)

For automated scripts, use this pattern:

```bash
BRIDGE_TX="0x8d1b60d0eaab6f609955bdd371e8004f47349cc809ff1bee81dc9d37237a031c"
DESTINATION_NETWORK=$LY

echo "Checking if bridge-and-call bridges are claimable on network $DESTINATION_NETWORK..."

# Check for asset bridge
ASSET_CLAIMABLE=$(aggsandbox show claims --network-id $DESTINATION_NETWORK --json | \
  jq -r --arg tx "$BRIDGE_TX" '.[] | select(.bridge_tx_hash == $tx and .type == "asset" and .status == "pending") | .bridge_tx_hash')

# Check for message bridge
MESSAGE_CLAIMABLE=$(aggsandbox show claims --network-id $DESTINATION_NETWORK --json | \
  jq -r --arg tx "$BRIDGE_TX" '.[] | select(.bridge_tx_hash == $tx and .type == "message" and .status == "pending") | .bridge_tx_hash')

if [ "$ASSET_CLAIMABLE" = "$BRIDGE_TX" ]; then
    echo "✅ Asset bridge is claimable"
else
    echo "⏳ Asset bridge not yet claimable"
fi

if [ "$MESSAGE_CLAIMABLE" = "$BRIDGE_TX" ]; then
    echo "✅ Message bridge is claimable"
else
    echo "⏳ Message bridge not yet claimable"
fi

# Wait if needed
if [ "$ASSET_CLAIMABLE" != "$BRIDGE_TX" ] || [ "$MESSAGE_CLAIMABLE" != "$BRIDGE_TX" ]; then
    echo "Waiting for bridges to become claimable..."
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

### Step 5: Claim Asset Bridge (First)

```bash
# Claim asset bridge first (MUST be done before message bridge)
ASSET_CLAIM_TX=$(aggsandbox bridge claim \
  --network-id $LY \
  --tx-hash $BRIDGE_TX \
  --source-network-id $LX | \
  grep "claim transaction submitted:" | cut -d' ' -f4)

echo "Asset claim transaction submitted: $ASSET_CLAIM_TX"
```

### Step 6: Verify Asset Claim Completion

First, manually check the destination network claims API:

```bash
# Check destination claims to find your asset claim transaction
aggsandbox show claims --network-id $LY
```

**Look for your asset claim transaction in the output:**
```json
[
  {
    "bridge_tx_hash": "0x8d1b60d0eaab6f609955bdd371e8004f47349cc809ff1bee81dc9d37237a031c",
    "claim_tx_hash": "0xa9fa5418144f7c8c1b78cd0e5560d6550411667ef937b554636a613f933b3d9f",
    "origin_network": $LX,
    "destination_network": $LY,
    "amount": "1000000000000000000",
    "status": "completed",
    "type": "asset"
  }
]
```

**What to look for:**

- Your asset claim transaction hash appears as `claim_tx_hash`
- Status changed from `"pending"` to `"completed"`
- Type is `"asset"` with amount > 0

For automated scripts, use this pattern:

```bash
ASSET_CLAIM_TX="0xa9fa5418144f7c8c1b78cd0e5560d6550411667ef937b554636a613f933b3d9f"
DESTINATION_NETWORK=$LY

echo "Monitoring asset claim status until completion..."

while true; do
    ASSET_CLAIM_DATA=$(aggsandbox show claims --network-id $DESTINATION_NETWORK --json | \
      jq --arg tx "$ASSET_CLAIM_TX" '.[] | select(.claim_tx_hash == $tx)')
    
    if [ "$ASSET_CLAIM_DATA" != "null" ] && [ "$ASSET_CLAIM_DATA" != "" ]; then
        ASSET_STATUS=$(echo "$ASSET_CLAIM_DATA" | jq -r '.status')
        echo "$(date): Asset claim status: $ASSET_STATUS"
        
        if [ "$ASSET_STATUS" = "completed" ]; then
            echo "✅ Asset claim completed! Proceeding to message claim..."
            break
        elif [ "$ASSET_STATUS" = "pending" ]; then
            echo "⏳ Asset claim pending, waiting 5 seconds..."
            sleep 5
        else
            echo "❌ Unknown asset claim status: $ASSET_STATUS"
            break
        fi
    else
        echo "❌ Asset claim not found"
        break
    fi
done
```

### Step 7: Claim Message Bridge (Second)

```bash
# Claim message bridge AFTER asset claim is completed
MESSAGE_CLAIM_TX=$(aggsandbox bridge claim \
  --network-id $LY \
  --tx-hash $BRIDGE_TX \
  --source-network-id $LX | \
  grep "claim transaction submitted:" | cut -d' ' -f4)

echo "Message claim transaction submitted: $MESSAGE_CLAIM_TX"
```

### Step 8: Verify Message Claim Completion

First, manually check the destination network claims API:

```bash
# Check destination claims to find your message claim transaction
aggsandbox show claims --network-id $LY
```

**Look for your message claim transaction in the output:**
```json
[
  {
    "bridge_tx_hash": "0x8d1b60d0eaab6f609955bdd371e8004f47349cc809ff1bee81dc9d37237a031c",
    "claim_tx_hash": "0xb8e6521234f8c7d9e1f45678901234567890abcdef1234567890abcdef123456",
    "origin_network": $LX,
    "destination_network": $LY,
    "amount": "0",
    "status": "completed",
    "type": "message"
  }
]
```

**What to look for:**

- Your message claim transaction hash appears as `claim_tx_hash`
- Status changed from `"pending"` to `"completed"`
- Type is `"message"` with amount = 0

For automated scripts, use this pattern:

```bash
MESSAGE_CLAIM_TX="0xb8e6521234f8c7d9e1f45678901234567890abcdef1234567890abcdef123456"
DESTINATION_NETWORK=$LY

echo "Monitoring message claim status until completion..."

while true; do
    MESSAGE_CLAIM_DATA=$(aggsandbox show claims --network-id $DESTINATION_NETWORK --json | \
      jq --arg tx "$MESSAGE_CLAIM_TX" '.[] | select(.claim_tx_hash == $tx)')
    
    if [ "$MESSAGE_CLAIM_DATA" != "null" ] && [ "$MESSAGE_CLAIM_DATA" != "" ]; then
        MESSAGE_STATUS=$(echo "$MESSAGE_CLAIM_DATA" | jq -r '.status')
        echo "$(date): Message claim status: $MESSAGE_STATUS"
        
        if [ "$MESSAGE_STATUS" = "completed" ]; then
            echo "✅ Message claim completed successfully!"
            break
        elif [ "$MESSAGE_STATUS" = "pending" ]; then
            echo "⏳ Message claim pending, waiting 5 seconds..."
            sleep 5
        else
            echo "❌ Unknown message claim status: $MESSAGE_STATUS"
            break
        fi
    else
        echo "❌ Message claim not found"
        break
    fi
done
```

### Step 9: Verify Contract Results

```bash
# Check if contract received tokens and executed function
echo "Verifying bridge-and-call results..."

# Check contract token balance
TOKEN_BALANCE=$(cast call $TOKEN_ADDRESS \
  "balanceOf(address)(uint256)" \
  $CONTRACT_ADDRESS \
  --rpc-url http://localhost:$PORT)

echo "Contract token balance: $TOKEN_BALANCE"

# Check if contract function was called (depends on your contract)
cast call $CONTRACT_ADDRESS \
  "lastMessage()(string)" \
  --rpc-url http://localhost:$PORT

echo "✅ Bridge-and-call operation completed successfully!"
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

## Bridge-and-Call with ETH

### Bridge with ETH Value

```bash
# Bridge tokens and send 0.1 ETH to contract function
aggsandbox bridge bridge-and-call \
  --network-id 0 \
  --destination-network-id 1 \
  --amount 1000000000000000000 \
  --token-address $TOKEN_ADDRESS \
  --to-address $CONTRACT_ADDRESS \
  --call-data $CALL_DATA \
  --msg-value 100000000000000000 \
  --private-key $PRIVATE_KEY

# When claiming the message bridge, include the ETH value
aggsandbox bridge claim \
  --network-id 1 \
  --tx-hash $BRIDGE_TX \
  --source-network-id 0 \
  --msg-value 100000000000000000
```

**Note**: Use wei values for amounts (1 ETH = 1000000000000000000 wei).

## Smart Contract Integration

### Bridge-and-Call Architecture

**Important**: Bridge-and-call uses a different architecture than regular message bridging. The `BridgeExtension` contract handles the `onMessageReceived` call and automatically deploys a `JumpPoint` contract to execute your function. Your target contract does NOT need to implement `IBridgeMessageReceiver`.

**How it works:**
1. **Bridge-and-call** sends message to `BridgeExtension` on destination network
2. **BridgeExtension** receives both asset and message bridges
3. **BridgeExtension** deploys a `JumpPoint` contract with your encoded function call
4. **JumpPoint** transfers tokens to your contract and executes the function call

### Bridge-and-Call Receiver Contract

Here's a complete contract that can receive bridge-and-call operations (no special interface required):

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.22;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract BridgeAndCallReceiver is ReentrancyGuard {
    struct Operation {
        address token;
        uint256 amount;
        string message;
        uint256 timestamp;
        address sender;
    }
    
    Operation[] public operations;
    mapping(address => uint256) public operationCounts;
    mapping(address => uint256) public tokenBalances;
    
    event TokensReceived(
        address indexed token,
        uint256 amount,
        string message,
        address indexed sender
    );
    
    // This function is called by the JumpPoint contract deployed by BridgeExtension
    function receiveTokensWithMessage(
        address token,
        uint256 amount,
        string calldata message
    ) external payable nonReentrant {
        require(token != address(0), "Invalid token address");
        require(amount > 0, "Amount must be positive");
        
        // Record the operation
        operations.push(Operation({
            token: token,
            amount: amount,
            message: message,
            timestamp: block.timestamp,
            sender: msg.sender
        }));
        
        operationCounts[msg.sender]++;
        tokenBalances[token] += amount;
        
        emit TokensReceived(token, amount, message, msg.sender);
    }
    
    function getOperationCount() external view returns (uint256) {
        return operations.length;
    }
    
    function getLastOperation() external view returns (Operation memory) {
        require(operations.length > 0, "No operations");
        return operations[operations.length - 1];
    }
    
    function getTokenBalance(address token) external view returns (uint256) {
        return tokenBalances[token];
    }
}
```

### Deploy and Test

```bash
# 1. Deploy contract on destination network
CONTRACT_ADDRESS=$(forge create src/BridgeAndCallReceiver.sol:BridgeAndCallReceiver \
  --rpc-url http://localhost:8546 \
  --private-key $PRIVATE_KEY \
  --json | jq -r '.deployedTo')

echo "Contract deployed to: $CONTRACT_ADDRESS"

# 2. Prepare call data for the deployed contract
CALL_DATA=$(cast calldata "receiveTokensWithMessage(address,uint256,string)" \
  $TOKEN_ADDRESS \
  2000000000000000000 \
  "Bridge-and-call test!")

# 3. Execute bridge-and-call
aggsandbox bridge bridge-and-call \
  --network-id 0 \
  --destination-network-id 1 \
  --amount 2000000000000000000 \
  --token-address $TOKEN_ADDRESS \
  --to-address $CONTRACT_ADDRESS \
  --call-data "$CALL_DATA" \
  --private-key $PRIVATE_KEY

# 4. Wait and claim both bridges (asset first, then message)
sleep 30
aggsandbox bridge claim --network-id 1 --tx-hash $BRIDGE_TX --source-network-id 0
aggsandbox bridge claim --network-id 1 --tx-hash $BRIDGE_TX --source-network-id 0

# 5. Verify results
cast call $CONTRACT_ADDRESS "getOperationCount()(uint256)" --rpc-url http://localhost:8546
cast call $CONTRACT_ADDRESS "getLastOperation()(address,uint256,string,uint256,address)" --rpc-url http://localhost:8546
cast call $CONTRACT_ADDRESS "getTokenBalance(address)(uint256)" $TOKEN_ADDRESS --rpc-url http://localhost:8546
```

## Troubleshooting

### **Common Issues**

#### Bridge Transaction Fails
```bash
# Check account balance and token allowance
cast call $TOKEN_ADDRESS "balanceOf(address)(uint256)" $YOUR_ADDRESS --rpc-url http://localhost:8545
cast call $TOKEN_ADDRESS "allowance(address,address)(uint256)" $YOUR_ADDRESS $BRIDGE_ADDRESS --rpc-url http://localhost:8545

# Check network status
aggsandbox status
```

#### Claim Order Error
```bash
# Verify you claimed asset bridge first
aggsandbox show claims --network-id $LY --json | jq '.[] | select(.type == "asset")'

# Check if message claim failed due to missing tokens
aggsandbox show claims --network-id $LY --json | jq '.[] | select(.type == "message")'
```

#### Contract Function Not Executed
```bash
# Check if contract received tokens
cast call $TOKEN_ADDRESS "balanceOf(address)(uint256)" $CONTRACT_ADDRESS --rpc-url http://localhost:8546

# Test function call locally
cast call $CONTRACT_ADDRESS \
  --data "$CALL_DATA" \
  --rpc-url http://localhost:8546

# Check transaction receipt
cast receipt $MESSAGE_CLAIM_TX --rpc-url http://localhost:8546
```
