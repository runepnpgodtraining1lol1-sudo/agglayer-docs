## **Overview**  
Vault Bridge is designed to work with **any bridge**.

* **Today**: Any chain connected to the Agglayer can use Vault Bridge. Chains connected to the Agglayer using CDK OP Stack can also use the optional features like the Native Converter and Bridged USDC Standard.   
* **Future:** Support for other bridging routes across both EVM and non-EVM chains is coming soon. Future support for these routes will also include features like Native Converter for local token minting and deconverting and Bridged USDC Standard compatibility.

[Contact our team to learn more about onboarding today or future integration plans.](https://info.polygon.technology/vaultbridge-intake-form)

## **1\. Base Assets \- Vault Bridge Tokens**  
**What it is**

* Vault Bridge Tokens (vbTokens) form the foundation of Vault Bridge. Every supported asset (USDC, ETH, WBTC, USDS, USDT) has a corresponding vbToken on Ethereum.  

**Why it matters**

* vbTokens are 1:1 backed and redeemable on Ethereum.  
* Yield accrues on the underlying in curated vaults, distributed periodically as new vbTokens.  
* vbTokens behave like standard ERC-20s on L2s.

**Contracts involved**

* Vault Bridge Token (per asset) — ERC-4626 vault that mints vbTokens 1:1 and routes the underlying asset into external yield vaults.

## **2\. Native Converter (optional feature)** 
**What it is**

* A lightweight module deployed on L2 that lets users convert local assets ↔ vbTokens directly on the chain without returning to Ethereum for every swap.

**Why it matters**

* Improves UX: users don’t need to bridge just to move into or out of vbTokens.  
* Makes existing liquidity productive: e.g., convert bridged USDC → vbUSDC locally, with periodic rebalancing to Ethereum.

**Contracts involved**

* **Native Converter (L2)**: Handles local minting/deconverting of vbTokens on L2 as well as migration to L1  
* **Custom Token** **(L2):** Allows vbTokens to match local conventions (e.g., vbETH behaving like WETH). Includes permissions for mint/burn.  
* **Migration Manager (L1)**:  Receives backing from L2 → L1 and completes migration flow so converted assets earn yield at the source.

## **3\. Bridged USDC Standard (optional feature)**

**What it is**

* A version of vbUSDC that is compatible with [Circle’s Bridged USDC Standard](https://www.circle.com/blog/bridged-usdc-standard), ensuring compatibility with a future upgrade to Native Circle USDC on L2.

**Why it matters**

* Ensures compatibility with a future upgrade to Circle-native USDC without having to manually migrate balances. 
