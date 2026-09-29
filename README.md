# Noir Protocol, Bootstrapping the Zcash multi-Token Economy
## Mission
Noir is a programmable privacy complement to Zcash. It gives ZEC holders private access to Zcash-anchored tokens and DeFi, entirely within NoirWallet, as a practical step toward the private multi-asset economy that ZSAs originally proposed to bring onchain. Importantly, it also preserves ZEC as the only asset on Zcash.
## Overview
Noir Protocol separates responsibility cleanly: Zcash handles private assets, transaction signing, and final settlement; NEAR handles programmable execution; Noir Protocol itself handles identity isolation, private execution, network privacy, and asset aggregation between the two. The result is wallet, network, and gas abstraction,  no extra wallet to create, no other private key to hold, and gas abstracted away regardless of destination chain. All of this stays invisible to the user inside NoirWallet, with other Zcash wallets able to integrate the same layer over an SDK.
From a single Zcash seed, NoirWallet derives three fully isolated identities, each on its own HD derivation path, so no single identity ever exposes a user's full fund position, DeFi activity, and network origin together:

| Identity | Holds funds? | Used for |
|---|---|---|
| Shielded Address | Yes, the real funds account | ZEC and Zcash-anchored assets, deposits, final settlement |
| Public Auth Address | No | Signing ordinary, public DeFi operations |
| Confidential Auth Address | No | Signing Confidential Intents; never registered alongside the Public Auth Address, with no on-chain link between them |

## Core design
Noir Protocol separates funds, authorization, public execution, and final holdings into four layers, with a network-privacy layer (the Zero-Indexer, detailed below) sitting underneath both execution paths:

![Core Design](https://img.zknoir.com/doc/CoreDesign.png)

| Layer | Component | Role |
|---|---|---|
| 1 | Shielded Address | Funds and settlement |
| 2 | Public / Confidential Auth Addresses | Authorization only, isolated from funds and from each other |
| 3 | Zero-Indexer → User Global Contract or Private Relay | Network privacy, then execution routing |
| 4 | Public DeFi or Confidential Account | Where the trade actually happens, and where private holdings consolidate |

For DeFi not yet reachable directly inside the Confidential Account, the trade is completed by withdrawing from the Confidential Account to a batch of freshly created, one-time addresses rather than one fixed account, each recreated per trade, belonging only to that trade, and automatically deposited back into the Confidential Account once the trade completes. The public chain sees ordinary accounts trading; it can't determine which accounts share a user, how much any user bought in total, or who ultimately holds the resulting tokens.
## Zero-Indexer: the network privacy layer
On-chain privacy doesn't by itself solve network-layer privacy: an RPC, relayer, or indexer that sees a user's IP, request time, signature, and target transaction together can re-link that user to their on-chain activity through metadata alone. Noir Protocol addresses this with a network-privacy layer built on the architectural ideas behind Zcash's zero-indexer:

![Zero-Indexer](https://img.zknoir.com/doc/ZeroIndexerClear.png)

- NoirWallet sends a signed request to a TEE Shim / Privacy Entry,  downstream services never see the real IP.
- The request is forwarded, still inside the TEE, to a TEE Hub / Privacy Relay.
- The Hub aggregates requests from multiple users, randomizes forwarding order, delays some, and batches submissions.
- The Operator submits at meaningfully reduced correlation between IP, authorization, and on-chain transaction.

The goal isn't to hide what's already public on-chain, only to weaken how strongly those three signals can be tied together. The design leaves room to combine with network-layer tools like Nym or Tor in the future.
## Public execution path
Every Zcash user is assigned an independent NEAR Global Contract account (e.g. xxxxx.noir.near), running code Noir Protocol deploys once and references for every user — no per-user redeployment, no local NEAR private key for the user. It exists because the Confidential Account isn't yet fully permissionless: many new projects and tokens can't enter the confidential environment directly, so users need a way to reach any NEAR smart contract or DeFi application in the meantime.

![Zero-Indexer](https://img.zknoir.com/doc/PublicExecutionPathClear.png)

- The Public Auth Address signs the action.
- The Zero-Indexer privacy-relays it, decorrelating IP.
- The Operator submits the authorized transaction and pays gas, it cannot alter a signed action; any tampering fails verification.
- The User Global Contract verifies the Zcash signature, nonce, and deadline, then makes the cross-contract call.

What gets signed is a compact intent, not a raw transaction: protocol ID, NEAR network, the user's own Global Contract account, target contract and method, an arguments hash (commits to the call without exposing it), amount, nonce, and deadline.

This reach extends beyond NEAR: the User Global Contract can also drive NEAR Chain Signatures (threshold-MPC signing for Bitcoin, Solana, Cosmos, XRP, Aptos, Sui, and EVM chains) and coordinate through NEAR Intents. Chain Signatures is general-purpose: because MPC can sign any transaction payload, a Zcash-authorized action can call any contract on any supported chain directly, with no per-protocol integration required. NEAR Intents is narrower — it fulfils an intent only when a solver takes the other side, and solver activity today is concentrated in swaps and transfers. DeFi reachable via Chain Signatures is available directly through the Public Path; DeFi reachable only through NEAR Intents depends on solver liquidity.
## Confidential execution path and arbitrary DeFi
The long-term design avoids routing Confidential authorization through public NEAR MPC / chain signatures, since an MPC sign request leaves behind a payload hash, derivation path, signer, signature, and timestamp, all usable for correlation. The goal is for NEAR Intents (intents.far) to natively verify Zcash transparent-address signatures instead, so Confidential authorization never touches the public NEAR chain:

![Zero-Indexer](https://img.zknoir.com/doc/ConfidentialExecutionPath.png)

- The Confidential Auth Address signs locally.
- The Zero-Indexer privacy-relays it, decorrelating IP.
- intents.far verifies the Zcash signature.
- The Confidential Account executes the private swap, loan, or transfer.
This depends on native support from NEAR Intents. NEAR's existing Confidential Intents (GA since March 2026), TEE-secured private shards with selective disclosure for auditors, is the infrastructure this would build on.

For DeFi not yet reachable directly inside Confidential, for example, purchasing $20,000 of a token, the trade is completed by withdrawing from the Confidential Account to several freshly created, one-time addresses:

![Zero-Indexer](https://img.zknoir.com/doc/ConfidentialExecutionPath2.png)

| Account | Example amount |
|---|---:|
| A | $2,741 |
| B | $4,182 |
| C | $3,091 |

Each address is recreated per trade, belongs only to that trade, and is discarded once the trade completes and the tokens are automatically deposited back into the Confidential Account. The public chain sees ordinary accounts trading; it can't tell which accounts share a user, how much any one user bought, or who ultimately holds the result. Wallets that want this mode need to integrate the Noir SDK, since it depends on generating one-time identities, splitting orders, and routing through the Zero-Indexer automatically.
## State commitments and the Zcash checkpoint
Noir Protocol commits to its own history rather than relying on a single operator's assertion. Every accepted authorization becomes a leaf in an append-only Merkle Mountain Range (an unbounded alternative to a fixed-height Merkle tree), which feeds one of two roots, combined into a canonical NoirRoot alongside the current NEAR finalized block.

![Zero-Indexer](https://img.zknoir.com/doc/StateCommitment.png)

Public path: PublicIntent → PublicAuthLeaf → Public Auth MMR → NoirPublicRoot → NoirRoot → Zcash anchor transaction. Since the underlying data is public, any third party can independently rebuild a leaf, verify a signature, and recompute the root.

Confidential path: ConfidentialIntent → TEE verify and execute → ConfidentialAuthLeaf → Confidential Auth MMR → NoirConfidentialRoot → NoirRoot. Intent contents remain private and are processed and batched entirely inside the TEE, with each batch formed when either a maximum count or time interval is reached, limiting how much operation timing and frequency can be inferred from the public root. NoirConfidentialRoot is a TEE-attested commitment: third parties can verify through Remote Attestation that the leaves and root were generated by an approved version of the Noir Protocol code running inside a genuine TEE, without revealing or independently re-executing individual intents. The approved TEE code measurement is publicly registered and anchored to Zcash, allowing wallets and third parties to verify that the execution code has not been replaced or modified. Neither root records user balances.

![Zero-Indexer](https://img.zknoir.com/doc/StateCommitment2.png)

NoirRoot is periodically anchored onto Zcash itself via a 45-byte OP_RETURN payload (a 4-byte magic word, 1-byte protocol version, 8-byte state sequence, 32-byte NoirRoot), so Zcash — not a NEAR indexer, not Noir's own servers — becomes the canonical history. At launch, Noir Protocol publicly declares a Genesis Anchor UTXO (TXID, vout, anchor address, genesis NoirRoot, protocol version); each later checkpoint spends the prior Anchor UTXO and creates exactly one successor of equal value, paying its own fee from a separate UTXO, so the Anchor UTXO itself is never drained. Because a UTXO can't be validly double-spent, this gives Noir Protocol an append-only checkpoint history for free. The signing key is isolated from every user asset and runs under a fixed TEE/HSM policy — even a compromised key grants no control over user funds. A public Noir Root Indexer (distinct from the Zero-Indexer) follows this chain from genesis and serves historical root and inclusion-proof queries.
## Roles, principles, and supported products

| Component | Responsible for |
|---|---|
| NoirWallet | All three keys, local signing, one-time account generation, intent construction, Zero-Indexer connectivity, UI |
| Zero-Indexer | Running in a verifiable TEE, isolating IP from downstream services, aggregating/delaying/randomizing requests, never holds a key or decides transaction content |
| Operator | Submitting authorized public requests to NEAR and paying gas, inside the Zero-Indexer TEE, cannot alter a signed action |
| User Global Contract | Zcash signature verification, nonce/replay protection, NEAR execution, cross-contract calls |
| NEAR Intents / Confidential | Confidential Zcash signature verification, private balances and ownership, private DeFi execution |
| Zcash | Shielded assets, fund privacy, transaction signing, ZEC settlement, the NoirRoot checkpoint anchor |

**Core privacy principles**: funds identity is not execution identity; public identity is not private identity; on-chain privacy is not network privacy; Confidential operations avoid public MPC's correlatable signature trail; public execution identities are as one-time as possible for high-privacy trades; Noir Protocol never learns a user's Confidential balance; the TEE isolates metadata but never controls assets, so its failure can't hand over funds.

**Products**: private swap, private token trading and holdings, private lending and borrowing, private perps, private prediction markets, private asset transfer, and ZEC-backed DeFi, ZEC as private collateral, borrow against it, trade privately, convert back to ZEC, settle to the shielded address, without ever selling the underlying ZEC or exposing a full DeFi identity.

![Zero-Indexer](https://img.zknoir.com/doc/Roles.png)

## Path to ZSA
Native ZSA (ZIP-226/227) will let issuers mint assets directly inside Zcash's shielded Orchard pool, private by default from the moment of issuance, with only total supply visible for auditability. It's still in Draft, in development since 2022, with no mainnet activation date yet.

Noir Protocol follows the same principle ZSA embodies: a public, verifiable commitment on Zcash with the substance kept private, the function the NoirRoot checkpoint serves for protocol state. It is asset-agnostic by construction: any token reaching the Confidential Account, whether NEAR-native, bridged, or (once available) issued as native ZSA, gets the same identity isolation, one-time execution accounts, and private holdings. NoirWallet and the Confidential Account don't need to change when native ZSA ships; issuers and users simply gain a fully shielded-native option alongside what's already supported.

## Why NEAR, not a wrapped-asset bridge / not a new L2
NEAR was chosen specifically for programmable execution: general-purpose smart contracts, Global Contract accounts that let one deployment serve every user, and Chain Signatures and NEAR Intents that extend a Zcash-authorized action to other chains without bespoke per-chain integration. This is what makes Noir Protocol's multichain DeFi vision concrete rather than aspirational, NEAR functions as coordination and execution infrastructure, not the final destination.

NEAR is also not a new chain built for this proposal: Noir Protocol reuses an existing, live network for pre-consensus, ordering, batching, and finalizing state, before that state is checkpointed onto Zcash, rather than standing up a new L1 or L2 of its own. That network carries its own security: over 200 validators, with roughly 46% of NEAR's supply staked and subject to slashing for misbehavior, running in production since April 2020,  security Noir Protocol inherits rather than has to bootstrap.

That infrastructure is already battle-tested. NEAR Intents, the system both the Public and Confidential paths rely on, has processed billions in cumulative $ZEC swap volume, and NEAR Confidential Intents (GA since March 2026) has already surpassed $70 million in TVL. Within the Zcash ecosystem itself, wallets including Zashi, Zodl, and Vizor already run on NEAR Intents, with NEAR Intents expanding to single-flow swaps from 100+ tokens directly into ZEC.

Noir Protocol's own contribution sits on top of this: identity isolation, the Zero-Indexer, order splitting and one-time execution accounts, and the NoirRoot commitment anchored back to Zcash,  solving privacy and execution together, not just custody of a wrapped asset on a second chain.

## Product vision
Our goal is for Noir Protocol to become the reference path from Zcash-as-a-single-privacy-asset to Zcash-as-a-private-multi-asset-economy, proving out, in production, the token and DeFi experience the community is anticipating from native ZSA, and handing that experience off cleanly once ZSA activates. Users should never need to leave NoirWallet, manage a NEAR wallet, expose a long-term DeFi address, reveal their token holdings, or withdraw final funds to a transparent address, with four categories of information hidden as much as possible throughout: who you are, where your funds are, what you are doing, and where the request came from.

NoirWallet will serve as the first reference wallet for Noir Protocol, with an SDK planned to let other Zcash wallets and applications connect to this privacy execution layer over time. 
