# Asset Bridging Guide

**Complete guide to transferring tokens across networks using AggSandbox.**

Asset bridging is the most common cross-chain operation, enabling users to transfer ERC20 tokens and ETH between L1, L2, and L2-L2 networks. This guide covers all asset bridging scenarios with practical examples and best practices.

## Overview

Asset bridging in Agglayer follows a **lock-and-mint** model:

1. **Source Network**: Tokens are locked in the bridge contract
2. **Exit Root Update**: 
    - **If L1 source**: Direct Global Exit Root update on L1
    - **If L2 source**: Local Exit Root update, then AggKit updates GER on L1
3. **Bridge Service**: AggKit syncs GER to destination network
4. **Destination Network**: Wrapped tokens are minted for the user
5. **Claim Process**: User claims their wrapped tokens

## Asset Bridge Process

Asset bridging follows the same process for all network combinations. Use these steps for any LX (source) → LY (destination) bridge operation.

### Step 1: Bridge Assets

Bridge tokens from source network ($LX) to destination network ($LY):

```bash
# Bridge assets from $LX to $LY
aggsandbox bridge asset \
  --network-id $LX \
  --destination-network-id $LY \
  --amount $AMOUNT \
  --token-address $TOKEN_ADDRESS \
  --to-address $RECIPIENT_ADDRESS
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

### Step 3: Find Destination Token Address

```bash
# Get the wrapped/mapped token address on destination network
aggsandbox bridge utils get-mapped \
  --network-id $LY \
  --origin-network $LX \
  --origin-token $TOKEN_ADDRESS \
  --json
```

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
    "amount": "50",
    "status": "pending",
    "type": "asset"
  },
]
```

**What to look for:**

- Your bridge transaction hash appears as `bridge_tx_hash`
- `origin_network` matches your source ($LX)
- `destination_network` matches your destination ($LY)
- Status is `"pending"` (ready to claim)
- Type is `"asset"` for token bridging

#### For automated scripts, use this pattern:

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

### Step 5: Claim Tokens

```bash
# Claim tokens on destination network
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
    "amount": "50",
    "status": "completed",
    "type": "asset"
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

### Step 7: Verify Token Balance

```bash
# Check your token balance after claim completion
# Use the wrapped/mapped token address from Step 3
cast call $WRAPPED_TOKEN_ADDRESS \
  "balanceOf(address)(uint256)" \
  $RECIPIENT_ADDRESS \
  --rpc-url http://localhost:$PORT
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

## ETH Bridging

### Bridge Native ETH

```bash
# Bridge 0.1 ETH from L1 to L2
aggsandbox bridge asset \
  --network-id 0 \
  --destination-network-id 1 \
  --amount 100000000000000000 \
  --token-address 0x0000000000000000000000000000000000000000
```

**Note**: Use the zero address (`0x0000...`) for native ETH.

## Token Economics

### 💰 **Supply Conservation**

The bridge maintains **1:1 token backing**:

- **Total Supply**: `Original Tokens + Wrapped Tokens = Constant`
- **Bridge Lock**: Original tokens locked on source network
- **Wrapped Mint**: Equivalent wrapped tokens minted on destination
- **Burn on Return**: Wrapped tokens burned when bridging back

## Troubleshooting

### **Common Issues**

#### Bridge Transaction Fails
```bash
# Check account balance
cast call 0xToken "balanceOf(address)(uint256)" 0xAccount --rpc-url http://localhost:8545

# Check token allowance
cast call 0xToken "allowance(address,address)(uint256)" 0xAccount 0xBridge --rpc-url http://localhost:8545

# Check network status
aggsandbox status
```

#### Claim Fails
```bash
# Verify bridge was indexed
aggsandbox show bridges --network-id 0 | grep 0xYourTxHash

# Check if already claimed
aggsandbox bridge utils is-claimed --network 1 --index 42 --source-network 0

# Check AggKit sync status
aggsandbox logs aggkit
```

#### Balance Doesn't Update
```bash
# Check correct wrapped token address
aggsandbox bridge utils get-mapped --network 1 --origin-network 0 --origin-token 0xToken

# Verify claim completed
aggsandbox show claims --network-id 1

# Check transaction receipt
cast receipt 0xClaimTxHash --rpc-url http://localhost:8546
```