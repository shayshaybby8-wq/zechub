# Zinder: A Service-Oriented Indexer for Zcash

Blockchains store massive amounts of raw, unorganized data. Digging through millions of blocks every time an application or wallet needs specific information such as *"Which transactions belong to this wallet?"* is slow and resource-heavy.

An **indexer** solves this by continuously processing raw blockchain data and organizing it into a fast, searchable format so applications can query relevant information instantly.

**Zinder** is a self-hosted chain-data service designed specifically for Zcash. It indexes the blockchain once from a Zebra full node and exposes the organized data through query protocols, making it seamless for wallets and third-party applications to retrieve what they need.

Zcash already has indexers, the problem is these indexers frequently constrained product development because they were built around specific consumer protocols, most notably `lightwalletd` or tailored directly to legacy wallet SDKs like `zcashd`. Developers had the tools and capability to build new Zcash features, but instead of serving as a flexible foundation, the existing indexing infrastructure couldn't support their needs cleanly.

## Tight Coupling

The major issue is tight coupling. Coupling means an indexer's internal architecture is heavily dependent on the specific protocol or requirements of the application consuming its data.

When an indexer is built strictly around `lightwalletd`'s requirements, its internal assumptions become hardcoded to match. If a new application requires different data structures or query behaviors, the indexer's internal code must be altered or rewritten to accommodate it.

A consumer protocol should act as a simple **interface** to the indexer, not the blueprint that dictates how the indexer is built internally.

## How Zinder Addresses the Problem

Zinder reverses this relationship by separating the core indexing logic from consumer interfaces:

* It makes its own native query protocol called `WalletQuery`, which is the primary architectural interface of the system.
* It treats legacy protocols, like `lightwalletd` compatibility, as separate, pluggable modules rather than core dependencies.
* It decouples **chain ingestion**, **wallet projections**, and **query serving** so that one workload does not constrain or overload another.
* It indexes Zcash data once from a Zebra full node to maintain a consistent canonical view, serving multiple applications and wallets through independently deployable interfaces.


# WalletQuery Protocol

`WalletQuery` is not another blockchain or another indexer. It is the interface through which a wallet or application asks Zinder for the chain data it needs.

It gives clients a clear and predictable contract with the indexer through:

* Typed errors
* Capability discovery
* Resumable chain events
* Epoch-pinned reads

## 1. Epoch-Pinned Reads

Blockchain data is constantly changing, so different requests could otherwise see different versions of the chain.

`WalletQuery` solves this by tying reads to a specific **`ChainEpoch`**:

```text
Wallet → WalletQuery → Zinder
                     ↓
                ChainEpoch X
```

This ensures that related requests use the same consistent view of the chain, preventing the wallet from accidentally combining data from different chain states or competing tips.

## 2. Explicit Capability Discovery

`WalletQuery` allows a server to advertise exactly which features it supports through `ServerInfo`.

```text
Wallet → "What do you support?"
Zinder → "These are my capabilities."
Wallet → "I'll use those features."
```

This prevents clients from making assumptions based on the Zinder version and makes the client-server relationship more predictable.

## 3. Resumable Chain Events

`WalletQuery` can provide chain events that clients can resume from a known position.

Instead of rebuilding the chain state from scratch, a wallet can continue following changes from where it previously stopped.

# Zinder's Service-Oriented Architecture

Zinder is designed as a service-oriented indexer with its own protocol at the center. The chain is indexed once from a Zebra node, stored in a canonical representation, and then different services read from that shared indexed state.

The important architectural idea is that the **indexer is the center, not the wallet protocol**.

Zinder has separate components for ingestion, wallet projection, querying, and compatibility.

The canonical store is written by `zinder-ingest`, while `zinder-projector` builds the wallet-specific projection. `zinder-query` then serves the native `WalletQuery` protocol.

Then, if an existing wallet expects `lightwalletd`, Zinder doesn't redesign itself around `lightwalletd`. Instead, it adds a separate compatibility layer.

The `zinder-compat-lightwalletd` service translates the `lightwalletd` `CompactTxStreamer` protocol onto Zinder's native serving layer.

That is the architectural distinction the forum post is emphasizing: **lightwalletd support is a module of Zinder, rather than the foundation Zinder is built around.**

# Zinder vs. Traditional lightwalletd Architecture

This approach that Zinder takes is different from that of the Traditional `lightwalletd` whose architecture is primarily designed around serving light wallets, with the `CompactTxStreamer` protocol acting as the main interface between the blockchain backend and wallets.

This means the backend is largely shaped around what the `lightwalletd` protocol requires.

Rather than making the entire indexer depend on `lightwalletd`, Zinder treats `lightwalletd` support as a separate compatibility layer that translates between the two protocols.

In this way, traditional lightwalletd architecture is more consumer-protocol-driven, while Zinder is indexer-driven and protocol-independent, allowing different applications to consume the same underlying indexed data without forcing the indexer's internal architecture to conform to one protocol.

# How Zinder Improves Developer Experience

## 1. A Clear Native Protocol

* Developers can use `WalletQuery` instead of having to build directly around the internal behavior of an indexer.
* This gives applications a stable interface for retrieving Zcash data.

## 2. Consistent Chain State

* With epoch-pinned reads, developers can request data from a specific `ChainEpoch`.
* This reduces the risk of receiving inconsistent data when the blockchain is changing.

## 3. Explicit Capabilities

* Through `ServerInfo`, developers can ask a Zinder server what it actually supports.
* They don't have to guess based on the server version or attempt unsupported operations.

## 4. Better Error Handling

* `WalletQuery` provides typed errors and stable reason codes.
* Instead of interpreting vague errors, applications can programmatically determine what went wrong and respond appropriately.

## 5. Resumable Synchronization

* Developers can build wallets and applications that resume from a known point in the chain rather than repeatedly reconstructing their state.
* This makes synchronization more reliable, especially after interruptions.

## 6. Protocol Independence

* Developers aren't forced to design their application around `lightwalletd`'s assumptions.
* Zinder's architecture separates the underlying indexed data from the protocol used to consume it.

# How Zinder Can Improve Wallet/User Experience

## 1. More Reliable Synchronization

Epoch-pinned reads help the wallet work from a consistent view of the blockchain. This can reduce situations where different parts of a wallet's synchronization process see conflicting chain states.

## 2. Faster and Smoother Recovery

Resumable chain events allow a wallet to continue synchronization from where it stopped instead of rebuilding its state from scratch after an interruption.

## 3. Fewer Unexpected Failures

Typed errors and stable reason codes allow wallet developers to handle different failure conditions properly.

This can translate into clearer error messages and more graceful recovery for users.

## 4. Better Compatibility Across Servers

Capability discovery lets a wallet determine what a particular Zinder server supports before attempting to use a feature.

This can prevent unnecessary failures caused by unsupported functionality.

## 5. More Responsive Wallet Applications

Because Zinder separates the canonical chain data from wallet-specific projections, wallet queries can be served from data structures designed specifically for wallet needs rather than repeatedly processing raw blockchain data.

