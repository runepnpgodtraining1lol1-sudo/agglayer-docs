# Installation Guide

**Complete installation and setup guide for AggSandbox.**

## System Requirements

### Minimum Requirements
- **OS**: Linux, macOS, or Windows with WSL2
- **RAM**: 4GB minimum, 8GB recommended
- **Storage**: 2GB free space
- **CPU**: 2 cores minimum, 4 cores recommended

### Required Software

#### **Docker & Docker Compose**
```bash
# Install Docker (Ubuntu/Debian)
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Verify installation
docker --version
docker compose version
```

#### **Rust Toolchain**
```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source ~/.cargo/env

# Verify installation
rustc --version
cargo --version
```

#### **Make & Git**
```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y make git

# macOS (with Homebrew)
brew install make git

# Verify installation
make --version
git --version
```

## Installation Methods

### Installation

```bash
# Clone repository
git clone https://github.com/agglayer/aggsandbox
cd aggsandbox

# Install CLI tool
make install

# Verify installation
aggsandbox --help
```

## Post-Installation Setup

### Verify Installation

```bash
# Check CLI is available
aggsandbox --help

# Check Docker integration
docker --version
docker compose version

# Check Rust toolchain
rustc --version
```

### Test Basic Functionality

```bash
# Start sandbox
aggsandbox start --detach

# Check status
aggsandbox status

# View configuration
aggsandbox info

# Stop sandbox
aggsandbox stop
```

## Configuration Options

### Environment Variables

Create optional `.env` file for customization:

```bash
# Copy example configuration
cp .env.example .env

# Edit configuration
nano .env
```

**Available Variables:**
```bash
# Network Configuration
FORK_URL_L1=https://eth-mainnet.alchemyapi.io/v2/your-key
FORK_URL_L2=https://polygon-mainnet.alchemyapi.io/v2/your-key

# Service Ports
L1_RPC_PORT=8545
L2_RPC_PORT=8546
BRIDGE_API_PORT=5577

# Docker Configuration
DOCKER_COMPOSE_FILE=docker-compose.yml
DOCKER_NETWORK=aggsandbox_default

# Logging
LOG_LEVEL=info
DEBUG_MODE=false
```

### Docker Customization

Customize `docker-compose.yml` for specific needs:

```yaml
# Custom resource limits
services:
  anvil-l1:
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '0.5'
        reservations:
          memory: 512M
          cpus: '0.25'
```

## Network Modes

### Standard Mode

Basic L1 <-> L2 bridging:

```bash
aggsandbox start --detach
```

**Provides:**

- L1 Ethereum (Network 0, Port 8545)
- L2 zkEVM (Network 1, Port 8546)
- Bridge service with AggKit

### Multi-L2 Mode

Enables L2 <-> L2 bridging:

```bash
aggsandbox start --multi-l2 --detach
```

**Provides:**

- L1 Ethereum (Network 0, Port 8545)
- L2-1 zkEVM (Network 1, Port 8546)
- L2-2 Additional Chain (Network 2, Port 8547)
- Enhanced bridge service

### Fork Mode

Use real blockchain data:

```bash
# Set fork URLs in .env
FORK_URL_L1=https://eth-mainnet.alchemyapi.io/v2/your-key
FORK_URL_L2=https://polygon-mainnet.alchemyapi.io/v2/your-key

# Start in fork mode
aggsandbox start --fork --detach
```

**Benefits:**

- Real contract addresses
- Real token balances
- Real network state
- Production-like testing

## Troubleshooting Installation

### Common Issues

#### **Command Not Found**
```bash
# Check if installed
ls -la ~/.local/bin/aggsandbox

# Check PATH
echo $PATH | grep -q "$HOME/.local/bin"

# Fix PATH
export PATH="$HOME/.local/bin:$PATH"

# Reinstall if needed
make install
```

#### **Docker Permission Denied**
```bash
# Add user to docker group (Linux)
sudo usermod -aG docker $USER

# Restart shell or logout/login
newgrp docker

# Test Docker access
docker run hello-world
```

#### **Rust Not Found**
```bash
# Source Rust environment
source ~/.cargo/env

# Or reinstall Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

#### **Port Conflicts**
```bash
# Check port usage
lsof -i :8545
lsof -i :8546
lsof -i :5577

# Kill conflicting processes
sudo kill -9 $(lsof -t -i:8545)

# Or change ports in docker-compose.yml
```

#### **Build Failures**
```bash
# Clean build
make clean
make install

# Check build logs
make build 2>&1 | tee build.log

# Update dependencies
rustup update
```

### System-Specific Issues

#### **macOS**
```bash
# Install Xcode command line tools
xcode-select --install

# Install Homebrew dependencies
brew install make git

# Fix potential M1 issues
arch -arm64 make install
```

#### **Ubuntu/Debian**
```bash
# Update package manager
sudo apt-get update

# Install build essentials
sudo apt-get install -y build-essential curl

# Fix potential SSL issues
sudo apt-get install -y ca-certificates
```

#### **Windows (WSL2)**
```bash
# Ensure WSL2 is running
wsl --set-default-version 2

# Install in WSL2 environment
sudo apt-get update
sudo apt-get install -y make git curl build-essential

# Follow Linux installation steps
```

## Uninstallation

### Clean Uninstall

```bash
# Stop sandbox
aggsandbox stop --volumes

# Uninstall CLI
cd aggsandbox
make uninstall

# Remove repository
cd ..
rm -rf aggsandbox
```
