
# Boost AA Wallet contest details

- Join [Sherlock Discord](https://discord.gg/MABEWyASkp)
- Submit findings using the issue page in your private contest repo (label issues as med or high)
- [Read for more details](https://docs.sherlock.xyz/audits/watsons)

# Q&A

### Q: On what chains are the smart contracts going to be deployed?
EVM-compatible networks (excluding zkSync Era and other where CREATE2 operates abnormally). Specifically issues related to address derivation on ZKSync are considered out of scope
https://docs.zksync.io/build/developer-reference/ethereum-differences/evm-instructions#address-derivation
___

### Q: If you are integrating tokens, are you allowing only whitelisted tokens to work with the codebase or any complying with the standard? Are they assumed to have certain properties, e.g. be non-reentrant? Are there any types of [weird tokens](https://github.com/d-xo/weird-erc20) you want to integrate?
Protocol should work with all native and ERC20 tokens that adhear to standard including weird tokens.
No assumptions are made in regards to the properties of integrating tokens and weird trait tokens should be handled normally. Specifically we'd expect to interact with USDT and other stable coins/decimal 6 coins and erc20z from Zora as well as other fractional NFT contracts as long as they adhere to ERC20 standard. We aren't whitelisting tokens so any erc20 should work (including weird ones).
___

### Q: Are there any limitations on values set by admins (or other roles) in the codebase, including restrictions on array lengths?
BoostCore.sol::protocolFee = Between 0 - FEE_DENOMINATOR
BoostCore.sol::referralFee = Between 0 - FEE_DENOMINATOR
___

### Q: Are there any limitations on values set by admins (or other roles) in protocols you integrate with, including restrictions on array lengths?
No
___

### Q: For permissioned functions, please list all checks and requirements that will be made before calling the function.
For any permissioned function that takes a bytes payload and handles `abi.decode` at the start of the function the payload and approval amounts will be validated before being pushed on-chain. Particularly relevant for clawback and disburse on the budgets.

for `setCreateBoostAuth` the address will be confirmed to implement the interface functions before being set on-chain.

For `setProtocolFeeReceiver` the address will be known to be safe for receiving funds by way of  ` safeTransferETH` before being set on-chain

___

### Q: Is the codebase expected to comply with any EIPs? Can there be/are there any deviations from the specification?
Expectation is strict compliance with EIP-712 [Side effect -> 155, 191] by way of SignerValidator.
Expectation is strict compliance with EIP-20 by way of ERC20Incentive and ERC20VariableIncentive.
Expectation is strict compliance with EIP1155 by way of ERC1155ncentive
Expectaion is strict compliance with EIP-165 by way of every Boost Component (actions, allowlists, budgets, incentives and validators)
Signatures should be optionally with EIP-1271
Full system should be optionally compliant with EIP-4337

___

### Q: Are there any off-chain mechanisms or off-chain procedures for the protocol (keeper bots, arbitrage bots, etc.)?
We handle off-chain signature generation for the SignerValidator.

Here is the relevant code responsible for off-chain signature generation. We've confirmed protocol tests work with on-chain signature generation and off-chain using this helper:
https://github.com/rabbitholegg/boost-protocol/blob/main/packages/sdk/src/utils.ts#L856-L917

Corresponding SDK tests:
https://github.com/rabbitholegg/boost-protocol/blob/9a4d71b63a1ccb169ebb7f3617b45cb69fa58df8/packages/sdk/src/Validators/SignerValidator.test.ts#L53-L109

It should be considered an issue only if a permissioned signer can pull funds in excess of the Boosts spend limit from another non-permissioned boost or from a budget.
___

### Q: Are there any hardcoded values that you intend to change before (some) deployments?
No
___

### Q: If the codebase is to be deployed on an L2, what should be the behavior of the protocol in case of sequencer issues (if applicable)? Should Sherlock assume that the Sequencer won't misbehave, including going offline?
Assuming the protocol is deployed on multiple chains (which it will be) If there is a sequencer issue on a given chain, it should not have any unexpected side effects on unaffected chains, or cause issues where loss of funds is possible on unaffected chains.
___

### Q: Should potential issues, like broken assumptions about function behavior, be reported if they could pose risks in future integrations, even if they might not be an issue in the context of the scope? If yes, can you elaborate on properties/invariants that should hold?
Yes. 
Regardless of future integrations a ManagedBudget should only be able to have funds moved out of itself through the existing defined functions in the contract (clawback, disburse).

Assuming an ERC20Incentive and SignerValidator regardless of future modules or integrations given these two modules for a boost the only path to disburse funds should be through boostCore:claimIncentive or directly through clawback.
___

### Q: Please discuss any design choices you made.
We understand there may be griefing vectors related to registering junk namespaces on the registry, and spamming `createBoost` but we'll deploy auth reactively to these issues when/if they manifest.

We understand that it may be possible to configure a junk-boost (i.e. incompatible modules) and configuration of a valid boost is an end-user concern up until the point where it can lead to loss of funds beyond the end users permissioned spends.

Boost creators are trusted to not prevent incentives claims.
___

### Q: Please list any known issues and explicitly state the acceptable risks for each known issue.
If a boost creator configures a junk-boost it's acceptable for them to lose all of the funds allocated to that boost but it shouldn't be possible for more loss of funds than the budget for that boost or for unrelated budgets or boosts. 

The assumption is for a given boost any user who is permissioned on that boost (i.e. a signer, etc) has the ability to drain that boost, but no more than that boost.

User input validation issues should only be relevant if the user configuration leads to loss of funds from a non-permissioned source (a budget they don't have permissions for, a boost they didn't deploy etc)
___

### Q: We will report issues where the core protocol functionality is inaccessible for at least 7 days. Would you like to override this value?
no
___

### Q: Please provide links to previous audits (if any).
N/A
___

### Q: Please list any relevant protocol resources.

I would recommend starting at the readme and then going through the e2e tests focused on the SignerValidator flow first.

https://github.com/rabbitholegg/boost-protocol?tab=readme-ov-file#boost-protocol
https://github.com/rabbitholegg/boost-protocol/blob/main/packages/evm/test/e2e/EndToEndSignerValidator.t.sol
sequenceDiagram
    participant User
    participant ExternalContract
    participant BackendValidator
    participant Core
    participant SignerValidator
    
    User ->> ExternalContract: Executes action
    BackendValidator->> ExternalContract: Validates action off-chain
    BackendValidator->> User: Generates signature
    User ->> Core: Submits signature to Core
    Core ->> SignerValidator: Calls Validator with signature
    SignerValidator->> SignerValidator: Uses EIP-712 signature validation
    Core ->> User: Claims Incentive
___

### Q: Additional audit information.
The most important point of focus should be on BoostCore - any potential hangups around BoostCore are the most important. For budgets the only budget we'll be using at launch will be ManagedBudget so that's the most important to focus on. Similarly the most important Action to focus on is the EventAction.

An issue leading to loss of funds would only be considered high severity if it can cause cross-boost losses or cross-budget loses. Misconfigurations in an individual boost or budget leading to loss of funds from that boost or budget are not of high importance to us.
___



# Audit scope


[boost-protocol @ 5711a9160c51abec01056c5a70501fd13f6ac489](https://github.com/rabbitholegg/boost-protocol/tree/5711a9160c51abec01056c5a70501fd13f6ac489)
- [boost-protocol/packages/evm/contracts/BoostCore.sol](boost-protocol/packages/evm/contracts/BoostCore.sol)
- [boost-protocol/packages/evm/contracts/BoostRegistry.sol](boost-protocol/packages/evm/contracts/BoostRegistry.sol)
- [boost-protocol/packages/evm/contracts/actions/AAction.sol](boost-protocol/packages/evm/contracts/actions/AAction.sol)
- [boost-protocol/packages/evm/contracts/actions/AContractAction.sol](boost-protocol/packages/evm/contracts/actions/AContractAction.sol)
- [boost-protocol/packages/evm/contracts/actions/AEventAction.sol](boost-protocol/packages/evm/contracts/actions/AEventAction.sol)
- [boost-protocol/packages/evm/contracts/actions/EventAction.sol](boost-protocol/packages/evm/contracts/actions/EventAction.sol)
- [boost-protocol/packages/evm/contracts/allowlists/AAllowList.sol](boost-protocol/packages/evm/contracts/allowlists/AAllowList.sol)
- [boost-protocol/packages/evm/contracts/allowlists/ASimpleAllowList.sol](boost-protocol/packages/evm/contracts/allowlists/ASimpleAllowList.sol)
- [boost-protocol/packages/evm/contracts/allowlists/ASimpleDenyList.sol](boost-protocol/packages/evm/contracts/allowlists/ASimpleDenyList.sol)
- [boost-protocol/packages/evm/contracts/allowlists/SimpleAllowList.sol](boost-protocol/packages/evm/contracts/allowlists/SimpleAllowList.sol)
- [boost-protocol/packages/evm/contracts/allowlists/SimpleDenyList.sol](boost-protocol/packages/evm/contracts/allowlists/SimpleDenyList.sol)
- [boost-protocol/packages/evm/contracts/auth/PassthroughAuth.sol](boost-protocol/packages/evm/contracts/auth/PassthroughAuth.sol)
- [boost-protocol/packages/evm/contracts/budgets/ABudget.sol](boost-protocol/packages/evm/contracts/budgets/ABudget.sol)
- [boost-protocol/packages/evm/contracts/budgets/AManagedBudget.sol](boost-protocol/packages/evm/contracts/budgets/AManagedBudget.sol)
- [boost-protocol/packages/evm/contracts/budgets/ManagedBudget.sol](boost-protocol/packages/evm/contracts/budgets/ManagedBudget.sol)
- [boost-protocol/packages/evm/contracts/incentives/AAllowListIncentive.sol](boost-protocol/packages/evm/contracts/incentives/AAllowListIncentive.sol)
- [boost-protocol/packages/evm/contracts/incentives/ACGDAIncentive.sol](boost-protocol/packages/evm/contracts/incentives/ACGDAIncentive.sol)
- [boost-protocol/packages/evm/contracts/incentives/AERC1155Incentive.sol](boost-protocol/packages/evm/contracts/incentives/AERC1155Incentive.sol)
- [boost-protocol/packages/evm/contracts/incentives/AERC20Incentive.sol](boost-protocol/packages/evm/contracts/incentives/AERC20Incentive.sol)
- [boost-protocol/packages/evm/contracts/incentives/AERC20VariableIncentive.sol](boost-protocol/packages/evm/contracts/incentives/AERC20VariableIncentive.sol)
- [boost-protocol/packages/evm/contracts/incentives/AIncentive.sol](boost-protocol/packages/evm/contracts/incentives/AIncentive.sol)
- [boost-protocol/packages/evm/contracts/incentives/APointsIncentive.sol](boost-protocol/packages/evm/contracts/incentives/APointsIncentive.sol)
- [boost-protocol/packages/evm/contracts/incentives/AllowListIncentive.sol](boost-protocol/packages/evm/contracts/incentives/AllowListIncentive.sol)
- [boost-protocol/packages/evm/contracts/incentives/CGDAIncentive.sol](boost-protocol/packages/evm/contracts/incentives/CGDAIncentive.sol)
- [boost-protocol/packages/evm/contracts/incentives/ERC1155Incentive.sol](boost-protocol/packages/evm/contracts/incentives/ERC1155Incentive.sol)
- [boost-protocol/packages/evm/contracts/incentives/ERC20Incentive.sol](boost-protocol/packages/evm/contracts/incentives/ERC20Incentive.sol)
- [boost-protocol/packages/evm/contracts/incentives/ERC20VariableIncentive.sol](boost-protocol/packages/evm/contracts/incentives/ERC20VariableIncentive.sol)
- [boost-protocol/packages/evm/contracts/incentives/PointsIncentive.sol](boost-protocol/packages/evm/contracts/incentives/PointsIncentive.sol)
- [boost-protocol/packages/evm/contracts/shared/ACloneable.sol](boost-protocol/packages/evm/contracts/shared/ACloneable.sol)
- [boost-protocol/packages/evm/contracts/shared/BoostError.sol](boost-protocol/packages/evm/contracts/shared/BoostError.sol)
- [boost-protocol/packages/evm/contracts/shared/BoostLib.sol](boost-protocol/packages/evm/contracts/shared/BoostLib.sol)
- [boost-protocol/packages/evm/contracts/shared/IBoostClaim.sol](boost-protocol/packages/evm/contracts/shared/IBoostClaim.sol)
- [boost-protocol/packages/evm/contracts/tokens/Points.sol](boost-protocol/packages/evm/contracts/tokens/Points.sol)
- [boost-protocol/packages/evm/contracts/validators/ASignerValidator.sol](boost-protocol/packages/evm/contracts/validators/ASignerValidator.sol)
- [boost-protocol/packages/evm/contracts/validators/AValidator.sol](boost-protocol/packages/evm/contracts/validators/AValidator.sol)
- [boost-protocol/packages/evm/contracts/validators/SignerValidator.sol](boost-protocol/packages/evm/contracts/validators/SignerValidator.sol)

