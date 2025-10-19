# Psy Jargon

Definitions/acronyms we use often

This document provides a glossary of common terms, acronyms, and jargon used throughout the Psy blockchain documentation. Understanding these definitions is crucial for developers to fully grasp the architecture and mechanics of the Psy network.

We hope this list can shrink overtime.

## A

- **Agent:** In Psy's agent-centric architecture, this often refers to a user account, especially one that might operate autonomously based on its programmable signature circuit logic, potentially without a traditional private key.
- **ABI (Application Binary Interface):** Defines how to call functions in a smart contract, including function signatures and data structures. Psy contracts will have an ABI definition, likely in a format compatible with its Dapen toolchain.
- **Aggregation Proof:** A Zero-Knowledge Proof that verifies one or more other "child" ZK proofs and, typically, also proves a combined state transition or computation based on the child proofs. Central to Psy's scalability.

## B

- **Block Proof:** The final, single Zero-Knowledge Proof for an entire block (checkpoint) in Psy. It recursively verifies all transactions and state changes within that block and also verifies the previous block's proof, thus attesting to the validity of the entire chain from genesis. This is the output of the `PsyCheckpointStateTransitionCircuit`.
- **Block Time:** The target interval at which new blocks (checkpoints) are finalized on the Psy network. Psy aims for `O(log(n))` block times, meaning the time to produce a block scales logarithmically with the number of active users, not linearly with transactions.

## C

- **CFC (Contract Function Circuit):** A Zero-Knowledge circuit representing the executable logic of a single smart contract function on Psy. Developed using high-level languages (e.g., via Dapen) and compiled into a ZK-provable format. CFCs are executed locally by users.
- **CFT (Contract Function Tree):** A per-contract Merkle tree whose leaves are the ZK circuit fingerprints (verifier data hashes) of all whitelisted, executable functions for that contract. Stored as part of the `CLEAF` in `GCON`. Ensures only verified code can be invoked.
- **CHKP (Checkpoint Tree / Root):** The ultimate root of trust for a given block in Psy. It's an append-only Merkle tree where the Nth leaf is the hash of the Nth block's overall state (the `PsyCheckpointLeaf`). The `CHKP` Root immutably represents the entire state snapshot of the blockchain at that block.
- **`PsyCheckpointLeaf`:** The data structure whose hash becomes a leaf in the `CHKP` tree. It contains the roots of all major global trees (`GUSR`, `GCON`, `URT`, etc.) and aggregated block statistics for a specific block.
- **`PsyCheckpointStateTransitionCircuit`:** The ZK circuit that generates the final block proof. It verifies all aggregated state transition proofs from the Coordinator layer and the previous block's proof, proving the correct update to the `CHKP` tree.
- **CLEAF (Contract Leaf):** A leaf node in the `GCON` tree. Contains global metadata for a deployed contract, including its deployer's identifier hash, the root of its `CFT`, and the required height (size) for its associated `CSTATE` trees.
- **Coordinator:** A network role responsible for orchestrating the final stages of block proof aggregation, managing global state components (like `GCON`, `URT`, `CHKP` root), processing global operations (registrations, deployments), and interfacing with Proof Miners for these tasks.
- **CSTATE (Contract State Tree):** The most granular state tree in PARTH. It is specific to a *single user AND a single contract*. It holds the actual state variables (storage slots) pertinent to that user within that contract. Smart contract logic (CFCs) primarily operates on and modifies a user's `CSTATE` trees.

## D

- **DA Miner (Data Availability Miner):** A network role responsible for storing blockchain state data (especially user `CSTATE` deltas and trees) and proving its availability.
- **Dapen (DPN):** The toolchain and framework used in Psy for developing smart contracts in high-level languages (like TypeScript/JavaScript inspired syntax) and compiling them into verifiable Contract Function Circuits (CFCs).
- **Delta Merkle Proof:** A type of Merkle proof that not only proves the existence of a leaf but also proves the transition of that leaf from an old value to a new value, and consequently, the transition of the Merkle tree root from an old root to a new root. Used extensively in Psy for proving state changes.
- **Deferred Transaction / Debt Tree:** A mechanism within UPS allowing a transaction to create an obligation (debt) for a subsequent, related transaction (e.g., an external call's response) to be processed later within the same UPS or a future one. Managed via Merkle trees.

## E

- **End Cap Proof:** The final, single ZK proof generated at the conclusion of a User Proving Session (UPS). It summarizes and validates the entire sequence of a user's local transactions for a block and is authorized by the user's ZK-powered signature. This is what users submit to Realms.
- **`EDATA` (Event Data Tree):** A conceptual global tree for storing event data emitted by smart contracts.

## F

- **Fingerprint (Circuit Fingerprint):** The hash of a ZK circuit's verifier data. Used to uniquely identify a circuit and ensure that only whitelisted circuits are executed or aggregated.

## G

- **GCON (Global Contract Tree):** A global Merkle tree that stores information about all smart contracts deployed on the Psy network. Each leaf (`CLEAF`) contains metadata for a specific contract, including its `CFT` root.
- **`GDT` (Global Deposit Tree):** A global tree likely used to manage and prove deposits into the Psy ecosystem from external chains or sources.
- **GUSR (Global User Tree):** A global Merkle tree that aggregates all registered users on Psy. Each leaf (`ULEAF`) corresponds to a user ID and contains their public key commitment, balance, nonce, last synchronized checkpoint ID, and the root of their personal `UCON` tree.
- **GUTA (Global User Tree Aggregation / Aggregator):** The process and set of ZK circuits responsible for recursively aggregating user End Cap proofs (and their implied `GUSR` state transitions) across Realms and up to the Coordinator level.
- **`GlobalUserTreeAggregatorHeader`:** A standardized data structure output by GUTA circuits, representing an aggregated state transition for a segment of the `GUSR` tree, along with relevant metadata like the `checkpoint_tree_root` it's based on and the `guta_circuit_whitelist_root`.
- **`GWT` (Global Withdrawal Tree):** A global tree likely used to manage and prove withdrawals from the Psy ecosystem.

## H

- **Horizontal Scalability:** The ability of a system to increase its throughput by adding more processing units (e.g., Proof Miners in Psy's case) in parallel, without a proportional increase in coordination overhead or bottlenecks. Psy achieves this through PARTH and parallel ZKP aggregation.

## K

- **KVQ (Key-Value Queue):** A custom Rust library in Psy providing an abstraction layer for structured, type-safe interaction with a backend like Redis, used for storing proofs, state, and managing job queues.

## L

- **Local Proving:** The process in Psy where users execute their transactions and generate ZK proofs for them on their own devices (or via a trusted delegate), rather than submitting raw transactions to the network for execution and validation.
- **Line Proof:** A Merkle proof optimization used in aggregation trees (like GUTA) to efficiently propagate a verified state transition from a lower node up a direct, unbranching path in the tree to a higher-level node.

## M

- **Merkle Tree:** A tree data structure in which every leaf node is labelled with the cryptographic hash of a data block, and every non-leaf node is labelled with the cryptographic hash of the labels of its child nodes. Used extensively in Psy for state commitment and verification.
- **MEV (Maximal Extractable Value):** Value extracted from users by network participants (miners/validators) by reordering, inserting, or censoring transactions. Psy's design aims to minimize or eliminate harmful MEV.

## N

- **NCA (Nearest Common Ancestor):** In a tree structure, the NCA of two nodes is the lowest (deepest) node that has both nodes as descendants. NCA proofs are used in GUTA to combine state transitions from two separate branches of the `GUSR` tree.
- **Nonce:** A number, typically associated with an account, that is incremented with each transaction or session to prevent replay attacks. Psy users have a nonce managed within their `ULEAF`.

## P

- **PARTH (Parallel Ascending Recursive Tree Hierarchy):** Psy's novel state architecture that enables massive parallelism by partitioning state (especially user-contract state) and defining strict access rules (localized writes, historical global reads).
- **PoUW (Proof of Useful Work):** Psy's consensus mechanism where network participants (Proof Miners, DA Miners) are rewarded for performing work that is directly useful for the network's operation, security, and scalability (e.g., generating ZK aggregation proofs, storing and proving data availability).
- **Proof Miner:** A network role in Psy that performs the computationally intensive ZK proof generation for aggregating user End Cap proofs and other network-level proofs.
- **Public Key (Psy):** In Psy, a user's public key is not a traditional ECC key. It's the hash of the verifier data of their chosen ZK-based signature circuit, combined with a user-defined parameter. This enables "Software Defined Keys" (SDKeys).

## Q

- **`QHashOut`:** A type used in Psy's circuit definitions, likely representing the output of a specific hash function (e.g., Poseidon) used within the ZK proofs, typically consisting of multiple field elements.
- **`QProvingJobDataID`:** An identifier for a specific ZK proving job within Psy's distributed proving system.

## R

- **Realm:** A network role in Psy that acts as a distributed ingestion point for user End Cap proofs and state deltas. Realms manage specific segments (shards) of the Global User Tree (`GUSR`) and initiate the first level of GUTA proof aggregation for users within their segment.
- **Recursive Proof:** A ZK proof that verifies one or more other ZK proofs. This technique is fundamental to Psy's ability to scale verification, allowing many proofs to be compressed into a single, final proof.

## S

- **SDKey (Software Defined Key):** Psy's programmable public key system, where the key's behavior and validation logic are defined by a ZK circuit.
- **Signature (Psy):** In Psy, a "signature" authorizing a User Proving Session is itself a ZK proof generated by the user's chosen signature circuit. This proof attests that the conditions defined in the signature circuit were met for the given session data.
- **Signature Circuit:** The ZK circuit whose verifier data hash (plus params) constitutes a user's public key. It defines the logic for generating a valid signature proof for that user.
- **Spiderman Append Proof:** A specific type of ZK proof (likely optimized for Merkle tree appends) used in Psy for operations like batch appending to the `URT` or `GCON`.
- **`StateReaderGadget`:** A component within Dapen CFCs that handles interactions with the contract's state (`CSTATE`), simulating reads and writes in a provable manner.

## U

- **UCON (User Contract Tree):** A per-user Merkle tree that maps Contract IDs to the roots of that user's corresponding `CSTATE` trees. It represents the user's state footprint across all contracts they've interacted with. Stored as part of the `ULEAF`.
- **ULEAF (User Leaf):** A leaf node in the `GUSR` tree, corresponding to a unique User ID. It contains the user's public key commitment, balance, nonce, last synchronized checkpoint ID, and the root of their `UCON` tree.
- **UPS (User Proving Session):** The process by which a user locally executes a sequence of their transactions, generates ZK proofs for each, and recursively combines these into a single "End Cap" proof for submission to the network.
- **`UPSStartSessionCircuit`:** The ZK circuit that initiates a UPS, anchoring it to a globally finalized checkpoint and the user's valid starting state.
- **`UPSCFCStandardTransactionCircuit`:** A ZK circuit within UPS that verifies a CFC proof and proves the correct update to the UPS state (e.g., `UCON` root, debt trees) based on that CFC's execution.
- **`UPSStandardEndCapCircuit`:** The ZK circuit that finalizes a UPS, verifying the entire chain of local proofs, the user's signature, and ensuring all session conditions (like cleared debts) are met.
- **URT (User Registration Tree):** A global Merkle tree that commits to user public keys during the registration process, ensuring uniqueness and linking registrations to cryptographic identities.

## V

- **Validator:** A term common in other blockchains (especially PoS) for nodes that validate transactions and propose blocks. Psy **does not use validators** in this traditional sense; its security relies on "Proof of Math" (ZKPs) and PoUW.
- **Verifier Data:** Data specific to a ZK circuit that is required by a verifier to check a proof generated by that circuit. The hash of the verifier data often serves as the circuit's unique fingerprint.

## Z

- **ZKP (Zero-Knowledge Proof):** A cryptographic protocol by which one party (the prover) can prove to another party (theverifier) that a statement is true, without revealing any information beyond the validity of the statement itself.