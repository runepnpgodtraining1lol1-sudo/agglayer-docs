# CLI Commands Reference

**Complete reference for all AggSandbox CLI commands with examples and use cases.**

## Global Options

Available for all commands:

```bash
--verbose, -v      # Enable verbose output for debugging
--quiet, -q        # Quiet mode (only errors and warnings)
--help, -h         # Show comprehensive help
--version, -V      # Show version information
--log-format       # Set output format (pretty, compact, json)
```

## Core Commands

### `aggsandbox start`

Start the sandbox environment with all required services.

```bash
aggsandbox start [OPTIONS]
```

**Options:**

- `--detach, -d` - Start in background (detached mode)
- `--build, -b` - Rebuild Docker images before starting
- `--fork, -f` - Use real blockchain data from FORK_URL
- `--multi-l2, -m` - Enable multi-L2 mode (adds L2-2 network)
- `--claim-all, -c` - Auto-sponsor all claims

**Examples:**
```bash
# Basic start
aggsandbox start --detach

# Start with real data
aggsandbox start --fork --detach

# Start multi-L2 mode for L2-L2 testing
aggsandbox start --multi-l2 --detach

# Start with automatic claiming
aggsandbox start --claim-all --detach
```

### `aggsandbox stop`

Stop the sandbox environment.

```bash
aggsandbox stop [OPTIONS]
```

**Options:**

- `--volumes, -v` - Remove Docker volumes (destructive)

**Examples:**
```bash
# Graceful stop
aggsandbox stop

# Clean stop (removes all data)
aggsandbox stop --volumes
```

### `aggsandbox status`

Check the status of all sandbox services.

```bash
aggsandbox status
```

**Example Output:**
```
✅ All services are running
📊 Bridge service: Ready
🌐 L1 Ethereum: Ready (http://localhost:8545)
🌐 L2 zkEVM: Ready (http://localhost:8546)
```

### `aggsandbox info`

Display comprehensive sandbox configuration.

```bash
aggsandbox info [OPTIONS]
```

**Options:**

- `--verbose, -v` - Show detailed configuration

**Example Output:**
```
📋 Agglayer Sandbox Information

Available Accounts
-----------------------
(0): 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
(1): 0x70997970C51812dc3A010C7d01b50e0d17dc79C8

Private Keys
-----------------------
(0): 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
(1): 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d

Polygon Sandbox Config:
L1 (Ethereum): Chain ID: 1, RPC: http://localhost:8545
L2 (zkEVM): Chain ID: 1101, RPC: http://localhost:8546
```

### `aggsandbox logs`

View service logs for debugging.

```bash
aggsandbox logs [OPTIONS] [SERVICE]
```

**Options:**

- `--follow, -f` - Follow log output in real-time

**Examples:**
```bash
# View all logs
aggsandbox logs

# Follow all logs
aggsandbox logs --follow

# View specific service
aggsandbox logs anvil-l1
aggsandbox logs aggkit

# Follow specific service
aggsandbox logs --follow anvil-l1

# Show last N lines (use shell pipe)
aggsandbox logs anvil-l1 | tail -10
```

### `aggsandbox restart`

Restart all sandbox services.

```bash
aggsandbox restart
```

## Bridge Commands

### `aggsandbox bridge asset`

Bridge ERC20 tokens or ETH between networks.

```bash
aggsandbox bridge asset [OPTIONS]
```

**Required Options:**

- `--network-id, -n <ID>` - Source network ID (0=L1, 1=L2-1, 2=L2-2)  
- `--destination-network-id, -d <ID>` - Destination network ID
- `--amount, -a <AMOUNT>` - Amount to bridge (in token units)
- `--token-address, -t <ADDRESS>` - Token contract address

**Optional Options:**

- `--to-address <ADDRESS>` - Recipient address (defaults to sender)
- `--gas-limit <LIMIT>` - Gas limit override
- `--gas-price <PRICE>` - Gas price in wei
- `--private-key <KEY>` - Private key to use

**Examples:**
```bash
# Bridge ERC20 tokens L1 → L2
aggsandbox bridge asset \
  --network-id 0 \
  --destination-network-id 1 \
  --amount 100 \
  --token-address 0x5FbDB2315678afecb367f032d93F642f64180aa3

# Bridge ETH L1 → L2
aggsandbox bridge asset \
  --network-id 0 \
  --destination-network-id 1 \
  --amount 1000000000000000000 \
  --token-address 0x0000000000000000000000000000000000000000

# Bridge L2 → L2 (Multi-L2 mode - fully working!)
aggsandbox bridge asset \
  --network-id 1 \
  --destination-network-id 2 \
  --amount 50 \
  --token-address 0x5FbDB2315678afecb367f032d93F642f64180aa3
```

### `aggsandbox bridge claim`

Claim previously bridged assets.

```bash
aggsandbox bridge claim [OPTIONS]
```

**Required Options:**

- `--network-id, -n <ID>` - Network to claim on
- `--tx-hash, -t <HASH>` - Original bridge transaction hash
- `--source-network-id, -s <ID>` - Source network of original bridge

**Optional Options:**

- `--deposit-count, -c <COUNT>` - Specific deposit to claim
- `--token-address <ADDRESS>` - Token address (auto-detected)
- `--data <HEX>` - Custom data for message claims
- `--msg-value <VALUE>` - ETH value for message claims
- `--private-key <KEY>` - Private key to use

**Examples:**
```bash
# Basic claim (wait ~20-45 seconds after bridge for sync)
aggsandbox bridge claim \
  --network-id 1 \
  --tx-hash 0xabc123... \
  --source-network-id 0

# Claim specific deposit (for bridge-and-call)
aggsandbox bridge claim \
  --network-id 1 \
  --tx-hash 0xabc123... \
  --source-network-id 0 \
  --deposit-count 0

# Claim with ETH value (for message bridges)
aggsandbox bridge claim \
  --network-id 1 \
  --tx-hash 0xabc123... \
  --source-network-id 0 \
  --msg-value 1000000000000000000
```

### `aggsandbox bridge message`

Bridge with contract calls (currently has CLI bug - calls bridge-and-call).

```bash
aggsandbox bridge message [OPTIONS]
```

**Required Options:**

- `--network, -n <ID>` - Source network ID
- `--destination-network, -d <ID>` - Destination network ID
- `--target, -t <ADDRESS>` - Target contract address
- `--data <HEX>` - Contract call data (hex encoded)

**Optional Options:**

- `--amount, -a <AMOUNT>` - Amount of ETH to send
- `--fallback-address <ADDRESS>` - Fallback if call fails
- `--private-key <KEY>` - Private key to use

**Examples:**
```bash
# Bridge message with contract call
aggsandbox bridge message \
  --network 0 \
  --destination-network 1 \
  --target 0xContractAddress \
  --data 0xEncodedCallData

# Bridge ETH with message
aggsandbox bridge message \
  --network 0 \
  --destination-network 1 \
  --target 0xContract \
  --data 0xCallData \
  --amount 1000000000000000000
```

### `aggsandbox bridge bridge-and-call`

Execute atomic asset bridge + contract call.

```bash
aggsandbox bridge bridge-and-call [OPTIONS]
```

**Required Options:**

- `--network, -n <ID>` - Source network ID
- `--destination-network, -d <ID>` - Destination network ID
- `--token, -t <ADDRESS>` - Token contract address
- `--amount, -a <AMOUNT>` - Amount to bridge
- `--target <ADDRESS>` - Target contract address
- `--data <HEX>` - Contract call data
- `--fallback <ADDRESS>` - Fallback address if call fails

**Optional Options:**

- `--msg-value <VALUE>` - ETH value for contract call
- `--private-key <KEY>` - Private key to use

**Examples:**
```bash
# Bridge tokens and call contract
aggsandbox bridge bridge-and-call \
  --network 0 \
  --destination-network 1 \
  --token 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  --amount 100 \
  --target 0xContractAddress \
  --data 0xCallData \
  --fallback 0xFallbackAddress
```

## Information Commands

### `aggsandbox show bridges`

Show bridge information for a network.

```bash
aggsandbox show bridges --network-id <ID> [OPTIONS]
```

**Options:**

- `--network-id <ID>` - Network ID to query (required)
- `--json` - Output raw JSON

**Examples:**
```bash
# Show L1 bridges
aggsandbox show bridges --network-id 0

# Get JSON output for scripting
aggsandbox show bridges --network-id 0 --json
```

**JSON Response:**
```json
{
  "bridges": [
    {
      "tx_hash": "0xabc123...",
      "deposit_count": 42,
      "amount": "100",
      "destination_network": 1,
      "destination_address": "0x70997970...",
      "block_num": 15,
      "block_timestamp": 1234567890
    }
  ]
}
```

### `aggsandbox show claims`

Show claims for a network.

```bash
aggsandbox show claims --network-id <ID> [OPTIONS]
```

**Options:**

- `--network-id <ID>` - Network ID to query (required)
- `--json` - Output raw JSON

**Examples:**
```bash
# Show L2 claims
aggsandbox show claims --network-id 1

# Get JSON for automation
aggsandbox show claims --network-id 1 --json
```

**JSON Response:**
```json
{
  "claims": [
    {
      "tx_hash": "0xdef456...",
      "amount": "100",
      "status": "completed",
      "global_index": "12345",
      "origin_network": 0,
      "destination_network": 1,
      "block_num": 20
    }
  ]
}
```

### `aggsandbox show claim-proof`

Generate cryptographic proof for claiming.

```bash
aggsandbox show claim-proof [OPTIONS]
```

**Options:**

- `--network-id <ID>` - Target network for claiming
- `--leaf-index <INDEX>` - Leaf index in global exit tree
- `--deposit-count <COUNT>` - Deposit count when exit created
- `--json` - Output raw JSON

**Example:**
```bash
aggsandbox show claim-proof \
  --network-id 0 \
  --leaf-index 0 \
  --deposit-count 42 \
  --json
```

## Bridge Utility Commands

### `aggsandbox bridge utils get-mapped`

Get wrapped token address for an origin token.

```bash
aggsandbox bridge utils get-mapped [OPTIONS]
```

**Required Options:**

- `--network <ID>` - Target network
- `--origin-network <ID>` - Origin network
- `--origin-token <ADDRESS>` - Origin token address

**Example:**
```bash
aggsandbox bridge utils get-mapped \
  --network 1 \
  --origin-network 0 \
  --origin-token 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  --json
```

### `aggsandbox bridge utils precalculate`

Pre-calculate wrapped token address before deployment.

```bash
aggsandbox bridge utils precalculate [OPTIONS]
```

**Required Options:**

- `--network <ID>` - Target network
- `--origin-network <ID>` - Origin network  
- `--origin-token <ADDRESS>` - Origin token address

**Example:**
```bash
aggsandbox bridge utils precalculate \
  --network 0 \
  --origin-network 1 \
  --origin-token 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  --json
```

### `aggsandbox bridge utils get-origin`

Get origin token information from wrapped token.

```bash
aggsandbox bridge utils get-origin [OPTIONS]
```

**Required Options:**

- `--network <ID>` - Network with wrapped token
- `--wrapped-token <ADDRESS>` - Wrapped token address

**Example:**
```bash
aggsandbox bridge utils get-origin \
  --network 1 \
  --wrapped-token 0xc2bbbe7ec542aeab737751add2e6fe44f39aae72 \
  --json
```

### `aggsandbox bridge utils is-claimed`

Check if a bridge has been claimed.

```bash
aggsandbox bridge utils is-claimed [OPTIONS]
```

**Required Options:**

- `--network <ID>` - Network to check
- `--index <INDEX>` - Bridge index
- `--source-network <ID>` - Source network

**Example:**
```bash
aggsandbox bridge utils is-claimed \
  --network 1 \
  --index 42 \
  --source-network 0 \
  --json
```

### `aggsandbox bridge utils build-payload`

Build complete claim payload from bridge transaction.

```bash
aggsandbox bridge utils build-payload [OPTIONS]
```

**Required Options:**

- `--tx-hash <HASH>` - Bridge transaction hash
- `--source-network <ID>` - Source network

**Example:**
```bash
aggsandbox bridge utils build-payload \
  --tx-hash 0xabc123... \
  --source-network 0 \
  --json
```

### `aggsandbox bridge utils compute-index`

Calculate global bridge index from local index.

```bash
aggsandbox bridge utils compute-index [OPTIONS]
```

**Required Options:**

- `--local-index <INDEX>` - Local bridge index
- `--source-network <ID>` - Source network

**Example:**
```bash
aggsandbox bridge utils compute-index \
  --local-index 42 \
  --source-network 0 \
  --json
```

### `aggsandbox bridge utils network-id`

Get bridge contract network ID.

```bash
aggsandbox bridge utils network-id [OPTIONS]
```

**Required Options:**

- `--network <ID>` - Network to query

**Example:**
```bash
aggsandbox bridge utils network-id \
  --network 1 \
  --json
```

## Event Monitoring

### `aggsandbox events`

Monitor and decode blockchain events.

```bash
aggsandbox events [OPTIONS]
```

**Required Options:**

- `--network-id <ID>` - Network to monitor

**Optional Options:**

- `--blocks <COUNT>` - Number of recent blocks to scan (default: 10)
- `--address <ADDRESS>` - Filter events from specific contract

**Examples:**
```bash
# Monitor L1 events
aggsandbox events --network-id 0

# Monitor L2 events with filter
aggsandbox events \
  --network-id 1 \
  --blocks 20 \
  --address 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512
```
