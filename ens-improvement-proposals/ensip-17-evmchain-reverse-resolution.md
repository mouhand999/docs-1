---
description: >-
  Specifies reverse resolution in a cross-chain context
---

# ENSIP-XX: EVM-chain Reverse Resolution

| **Author**    | Jeff Lau \<jeff@ens.domains> |
| ------------- | ---------------------------- |
| **Status**    | Draft                        |
| **Submitted** | 2023-03-14                   |

## Abstract

This ENSIP specifies a way for reverse resolution to be used on other EVM chains. This allows setting primary names on L2s, as well as resolving any other records set on this specific reverse record, such as the avatar.

## Motivation

Reverse resolution has been in use since ENS's inception, however at the time Ethereum had no concrete scaling plans. In the past 5 years, we've seen layer 2s and sidechains become more prevalent and we first allowed support for these with ENSIP-9 (formerly EIP-2304) to allow addresses from other chains to be stored on ENS. Reverse resolution can be expanded to allow the forward resolution to first check the records for the chain that the user is on.

With account abstraction becoming more popular, it is becoming increasingly important to be able to setup different addresses for each L2, but using the same ENS name. It is no longer good practise to assume that the address of a user in mainnet is also controlled by the same user on an EVM-compatible L2. This can be solved with ENS names resolving to different addresses on each chain, but also additionally the primary ENS name being set to the same ENS name on each chain preserving the identity of the user across chains.

## Specification

### Overview

* Reverse registrars will be setup on each L2, with a corresponding registry
* Reverse registrar will only allow setting the name, without resolver customisability. This is to allow a consistent storage slot that can be checked on L1.
* User can now claim their reverse on this L2 and set it to their ENS name
* Their ENS name will need to set their record for the same EVM-cointype as the network, which is specified in [ENSIP-11](https://docs.ens.domains/ensip/11).
<<<<<<< HEAD
* A dapp can then detect the chainID that a user is on, find the corresponding cointype and resolve their primary ens name by resolving the name record at [userAddress].[coinType].reverse. This will be resolved via CCIP-read and look up the reverse record on the corresponding EVM-chain.
=======
* A dapp can then detect the chainID that a user is on, find the corresponding cointype and resolve their primary ens name by resolving the name record at [userAddress].[evmChainCointype].reverse. This will be resolved via CCIP-read and look up the reverse record on the corresponding EVM-chain.
>>>>>>> 00d9d74b6149bd3113b191291d05aa31f3b2782a
* Dapp will then resolve this name via ENS on L1 to check if the forward resolution matches. This forward resolution can be on L1, or the user can set up CCIP-read records for each network and put those addresses wherever they want.
* Once matched, the dapp can then also resolve any text records on the primary ENS name, such as avatar.

### Resolving Primary ENS Names by a dapp

1) If a dapp has a connected user it SHOULD first check the chainId of the user.
2) If the chainId is not 1, it SHOULD then construct the ENS name [address].[coinType].reverse to obtain the primary ENS name of the user. 
3) If none is found, it SHOULD check check mainnet [address].default.reverse.
4) If the dapp finds an ENS name, it MUST first check the forward resolution. The forward resolution MUST match the same coin type as the chain id of the user.

Note: The dapp MUST NOT use the reverse record set for mainnet even if the Primary ENS name has not been set on the target chain, and must instead show the address.

### Resolving an avatar by a dapp on L2

ENSIP-12 was concieved before the ENS L2 reverse resolution specification and therefore should be updated to reflect the current state of ENS primary name resolution. This means that all avatar records are able to be updated on a per-chain basis by updating the avatar record on their reverse node.

#### Example for an EVM Address

To determine the avatar URI for a specific EVM chain address, the client MUST reverse-resolve the address by querying the ENS registry on Ethereum for the resolver of `<address>.<coinType>.reverse`, where `<address>` is the lowercase hex-encoded Ethereum address, without leading '0x'. Then, the client calls `.text(namehash('<address>.<coinType>.reverse'), 'avatar')` to retrieve the avatar URI for the address.

If a resolver is returned for the reverse record, but calling `text` causes a revert or returns an empty string, the client MUST call `.name(namehash('<address>.<coinType>.reverse'))`. If this method returns a valid ENS name, the client MUST:

1. Validate that the reverse record is valid, by resolving the returned name and calling `addr` on the resolver, checking it matches the original evmChainId (converted to cointype) address.
2. Perform [ENSIP-12 Avatar text record resolution](https://docs.ens.domains/ensip/12) on the primary name.

A failure at any step of this process MUST be treated by the client identically as a failure to find a valid avatar URI.

### Examples of valid L2 reverse resolution

* Arbitrum: 0F32b753aFc8ABad9Ca6fE589F707755f4df2353.2147525809.reverse
* Optimism:
0F32b753aFc8ABad9Ca6fE589F707755f4df2353.2147483658.reverse
* Base: 0F32b753aFc8ABad9Ca6fE589F707755f4df2353.2147492101.reverse
* Polygon ZKEVM: 0F32b753aFc8ABad9Ca6fE589F707755f4df2353.2147484749.reverse
* ZKSync Era: 0F32b753aFc8ABad9Ca6fE589F707755f4df2353.2147483972.reverse

### Deprecating use of mainnet primary ENS name on other chains

ENS has not been explicit about how to use the mainnet `addr()` record and it is often used as a backup to a user not having an address record set. The mainnet reverse record has also historically been used on other EVM chains due to no alternative on that specific chain. There are a few reasons why it would undesirable to encourage use of mainnet primary name and/or `addr(node, 60)` record as a backup for it not being set on another EVM chain.

An example of why this could be confusing:

Dapp is on Arbitrum and uses mainnet primary ENS name. It resolves the ENS name's mainnet address and uses that to verify the reverse record is correct. It also uses the mainnet address to allow in-app transfers.

Mainnet primary ENS name that has an `addr(node, 60)` that is a smart contract wallet. The smart contract wallet is only on Ethereum and the user in unable to use `CREATE2` to deploy the same smart contract wallet on arbitrum. User 2 sees this Primary ENS name and wants to send funds to User 1. User 2 resolves the `addr()` of the ENS name and sends the funds to an address that doesn't exist on arbitrum and User 1 has no way to access the counterfactual address on that chain.

If we mandated that the address cannot use `addr(node, 60)`, but only the address of the chain in question, it would be possible to use mainnet as a backup. However the fact remains that you would still need to claim and set your Primary ENS name on mainnet, and the possibility for confusion seem to outweigh the benefits of using mainnet (high gas) as a catch-all back up for other L2 EVM chains (low gas). Additionally this would only be useful for EVM-compatible chains and would not benefit non-EVM L2s that have a different address format. 

## Being explicit about default Primary ENS Name

To make things explicit we will allow the signing of a message to confirm that the address in question would like to use mainnet or another network as fallback. This would either resolve directly on mainnet. Defaults would only be applicable to EoAs that can sign a message. This is because smart contract accounts would not be able to reliably set a default on all chains.

### Setting default

1) Sign a message to set a default record
2) call `setName()` on the default registrar on L1

Possibility: Allowing L1 Primary ENS names to also use `default.reverse`. This could be incorporated into the public resolver's `name()` function.

### Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).