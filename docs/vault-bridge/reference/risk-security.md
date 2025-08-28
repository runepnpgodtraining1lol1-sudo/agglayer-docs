## **1\. Risk Management**

Assets deposited into Vault Bridge are placed in Morpho vaults, which are managed by experienced curators, such as Steakhouse Financial and Gauntlet. Each Morpho Vault has a Credora Network credit rating (e.g., [Vault Bridge USDC Morpho Vault](https://app.morpho.org/ethereum/vault/0xBEefb9f61CC44895d8AEc381373555a64191A9c4/vault-bridge-usdc?subTab=risk)).

## **2\. Withdrawals**

The protocol leverages Morpho’s buffer of assets (e.g., USDC) to enable fast, gas-efficient withdrawals.

## **3\. Audits**

Vault Bridge specific contracts have undergone formal third-party audits.

View audit reports [here.](https://github.com/agglayer/vault-bridge/tree/main/audits)

## **4\. Solvency**

Vault Bridge enforces a 1:1 ratio between vbTokens and the underlying token.

* Small fluctuations in the liquid amount of the underlying token are offset by yield generation and are typically not noticeable.  
* Larger fluctuations, while rare, are managed using a configurable slippage percentage (currently 1% for all vbTokens). When redeeming with slippage, users still receive the underlying token at a 1:1 ratio, as any losses within the slippage threshold are covered by the vbToken contract operator.