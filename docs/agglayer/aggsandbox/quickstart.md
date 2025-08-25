# AggSandbox Quickstart

**Get your first cross-chain bridge operation running in under 10 minutes.**

This guide will walk you through installing AggSandbox, starting the sandbox environment, and executing your first bridge operation. By the end, you'll have successfully bridged tokens from L1 to L2 and claimed them on the destination network.

## Prerequisites

Before you begin, ensure you have the following installed:

### System Requirements

- **Docker** >= 20.0 and Docker Compose >= 1.27
- **Rust** >= 1.70.0 - [Install Rust](https://rustup.rs/)
- **Make** (usually pre-installed on Unix systems)
- **Git** for cloning the repository

### Verify Prerequisites
```bash
# Check all required tools
docker --version && echo "✅ Docker installed"
docker compose version && echo "✅ Docker Compose installed"
rustc --version && echo "✅ Rust installed"
make --version && echo "✅ Make installed"
git --version && echo "✅ Git installed"
```

## Installation

### Step 1: Clone and Install
```bash
# Clone the repository
git clone https://github.com/agglayer/aggsandbox
cd aggsandbox

# Install the CLI tool
make install
```

This will:

- Build and install the CLI to `~/.local/bin`
- Set up bridge service dependencies
- Configure bridge functionality

### Step 2: Verify Installation
```bash
aggsandbox --help
```

You should see comprehensive help with examples and rich formatting.

## Your First Bridge Operation

### Step 1: Setup Environment
```bash
# Copy environment configuration
cp .env.example .env
```

### Step 2: Start the Sandbox
```bash
# Start the bridge environment without automatic claim
aggsandbox start --detach

# Start the complete bridge environment with automatic claim sponsoring
aggsandbox start --detach --claim-all
```

> **💡 Pro Tip**: The `--claim-all` flag enables **automatic claim sponsoring**, which means the Claimsponsor service will automatically sponsor all bridge claims without requiring manual `aggsandbox bridge claim` commands. This is perfect for testing and development workflows!

This starts:

- **L1 Ethereum node** (Anvil on port 8545)
- **L2 Polygon zkEVM node** (Anvil on port 8546)
- **Bridge service** with AggKit
- **Pre-deployed contracts** and test tokens

### Step 2: Check Status
```bash
# Verify everything is running
aggsandbox status
```

Expected output:
```
✅ All services are running
📊 Bridge service: Ready
🌐 L1 Ethereum: Ready (http://localhost:8545)
🌐 L2 zkEVM: Ready (http://localhost:8546)
```

### Step 3: View Configuration
```bash
# See available accounts and contracts
aggsandbox info
```

This shows:

- **Available accounts** with addresses and private keys
- **Network configuration** (RPCs, Chain IDs)
- **Contract addresses** (Bridge contracts, test tokens)

### Step 4: Your First Bridge
```bash
# Bridge 10 ERC20 tokens from L1 to L2
aggsandbox bridge asset \
  --network 0 \
  --destination-network 1 \
  --amount 10 \
  --token-address 0x5FbDB2315678afecb367f032d93F642f64180aa3 \
  --to-address 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
```

Expected output:
```
✅ Bridge transaction submitted: 0xabc123...
💡 Tokens will be available on L2 once claimed
```

### Step 5: Monitor the Bridge
```bash
# Check bridge status on L1
aggsandbox show bridges --network-id 0 --json
```

Look for your transaction in the bridges list:
```json
{
  "bridges": [
    {
      "tx_hash": "0xabc123...",
      "amount": "10",
      "deposit_count": 42,
      "destination_network": 1
    }
  ]
}
```

### Step 6: Claim Your Tokens
```bash
# Wait ~30 seconds for AggKit sync, then claim on L2
aggsandbox bridge claim \
  --network 1 \
  --tx-hash 0xabc123... \
  --source-network 0
```

Expected output:
```
✅ Claim transaction submitted: 0xdef456...
🎉 Assets should be available once the transaction is mined!
```

### Step 7: Verify Success
```bash
# Check claims on L2
aggsandbox show claims --network-id 1 --json
```

Your claim should appear with status "completed":
```json
{
  "claims": [
    {
      "amount": "10",
      "status": "completed",
      "tx_hash": "0xdef456..."
    }
  ]
}
```

## 🎉 Congratulations!

You've successfully completed your first cross-chain bridge operation! You've:

- ✅ **Bridged tokens** from L1 to L2
- ✅ **Monitored** the bridge transaction
- ✅ **Claimed tokens** on the destination network
- ✅ **Verified** the operation completed successfully

## What You Just Did

### The Bridge Flow
1. **Bridge Request**: Initiated on L1 (source network)
2. **GER Update**: L1 directly updates Global Exit Root (no AggKit for L1 operations)
3. **AggKit Sync**: AggKit-L2 indexes L1 GER updates and syncs to L2
4. **Cross-Chain Sync**: State synchronized between networks
5. **Claim Execution**: Tokens released on L2 (destination network)

### Key Concepts
- **Networks**: L1 (ID: 0), L2 (ID: 1)
- **Tokens**: ERC20 tokens become wrapped tokens on destination
- **Timing**: ~30 seconds for AggKit sync between networks
- **Verification**: Multiple ways to confirm success

## Next Steps

### **Explore More Bridge Operations**
- **[Asset Bridging Guide](asset-bridging.md)** - Complete asset bridge operations
- **[Message Bridging](message-bridging.md)** - Cross-chain contract calls
- **[Bridge-and-Call](bridge-and-call.md)** - Advanced atomic operations

### **Build Tests**
- **[Testing Guide](testing-guide.md)** - Create automated bridge tests
- **[Python Framework](api-integration.md)** - Programmatic bridge testing
- **[Test Examples](test-examples.md)** - Ready-to-use test templates

### **Advanced Usage**
- **[CLI Reference](cli-commands.md)** - Complete command documentation
- **[Configuration](configuration.md)** - Customize your environment
- **[Multi-L2 Mode](networks.md)** - Test L2-L2 bridging

### **Production Integration**
- **[Integration Patterns](integration-patterns.md)** - DApp integration examples
- **[Monitoring](monitoring.md)** - Production monitoring setup
- **[Security](advanced-concepts.md)** - Security considerations

## Common Next Actions

### Test Different Bridge Types
```bash
# Bridge ETH instead of ERC20
aggsandbox bridge asset \
  --network 0 --destination-network 1 \
  --amount 1000000000000000000 \
  --token-address 0x0000000000000000000000000000000000000000

# Try L2 to L1 bridging
aggsandbox bridge asset \
  --network 1 --destination-network 0 \
  --amount 5 \
  --token-address 0x5FbDB2315678afecb367f032d93F642f64180aa3
```

### Enable Multi-L2 Mode
```bash
# Stop current sandbox
aggsandbox stop

# Start with L2-L2 support
aggsandbox start --multi-l2 --detach

# Test L2-L2 bridging
aggsandbox bridge asset \
  --network 1 --destination-network 2 \
  --amount 10 \
  --token-address 0x5FbDB2315678afecb367f032d93F642f64180aa3
```

### Use the Python Testing Framework
```bash
# Run automated bridge tests
python3 test/L1-L2/test_bridge_asset_and_claim.py 25
python3 test/L2-L1/test_bridge_asset_and_claim.py 15
python3 test/L1-L2/test_bridge_and_call_and_claim.py 5
```

## Troubleshooting Quick Fixes

### Sandbox Won't Start
```bash
# Check Docker is running
docker --version

# Clean restart
aggsandbox stop --volumes
aggsandbox start --detach
```

### Bridge Claims Fail
```bash
# Check if bridge is indexed
aggsandbox show bridges --network-id 0

# Wait longer for AggKit sync (especially for L2-L2)
# L1<->L2: ~30 seconds
# L2<->L2: ~60-90 seconds
```

### Command Not Found
```bash
# Ensure ~/.local/bin is in PATH
export PATH="$HOME/.local/bin:$PATH"

# Or reinstall
make install
```
