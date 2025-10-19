# How a Block is Made

A journey from transaction to block proof

# Psy Network: End-to-End Block Production Journey

Understanding the lifecycle of transactions and proofs within the Psy network, from a user's local actions to the finalization of a globally verifiable block, is crucial for developers. This document traces this intricate journey, highlighting the roles of different network participants and the flow of Zero-Knowledge Proofs (ZKPs) that underpin Psy's security and scalability.

The process can be broadly divided into four main phases:

1. **Phase 1: User Proving Session (UPS) - Local Execution & Proof Generation**
2. **Phase 2: Realm-Level Ingestion & Initial Aggregation (GUTA Entry)**
3. **Phase 3: Coordinator-Level Aggregation & Global Operations**
4. **Phase 4: Final Block Proof Generation & Finalization**

## Phase 1: User Proving Session (UPS) - Local Execution & Proof Generation

This phase occurs on the user's device or a delegate chosen by the user. The goal is to process a batch of the user's desired transactions for the upcoming block and generate a single, compact "End Cap" proof that attests to their validity and is authorized by the user.

**Step 1.1: Session Initialization**

- **Action:** The user (or their client software) initiates a User Proving Session.
- **Circuit:** `UPSStartSessionCircuit` is executed locally.
- **Purpose:** To establish a cryptographically secure starting point for the session.
- Process:
  1. The user's client fetches the latest finalized global `CHKP` (Checkpoint) root from a Realm or Coordinator.
  2. It also fetches the user's current `ULEAF` (User Leaf) data from their `GUSR` (Global User Tree) segment, as it existed at that `CHKP` root. This `ULEAF` contains the root of their `UCON` (User Contract Tree) and other metadata like nonce and balance.
  3. The `UPSStartSessionCircuit` takes these (e.g., `CHKP` root, Merkle proof of `ULEAF` in `GUSR` anchored to that `CHKP` root) as inputs/witnesses.
  4. **Proves:** That the session starts from a valid, globally recognized state for that user (i.e., their `ULEAF` was part of the claimed `CHKP` root) and initializes the session's internal state (e.g., transaction count to 0, debt trees to empty).
  5. **Output:** A "start session" proof and an initial `UserProvingSessionHeader`. This header contains the `session_start_context` (linking to the global `CHKP` root) and the `current_state` of the UPS (which initially mirrors the start state but with an updated `last_checkpoint_id`).

**Step 1.2: Local Contract Function Execution & CFC Proof Generation (Repeated for each transaction)**

- **Action:** For each transaction the user wants to include (e.g., calling a function on a smart contract).
- **Circuit:** A specific `DapenContractFunctionCircuit` (CFC) corresponding to the smart contract function being called is executed locally.
- **Purpose:** To execute the smart contract logic and generate a ZK proof of its correct execution for *this specific instance* (inputs, current user `CSTATE`).
- Process:
  1. The user's client provides the necessary inputs to the CFC:
     - The `tx_ctx_header` (transaction context header): This includes the *assumed* starting state for this specific contract interaction (e.g., the root of the user's `CSTATE` for this contract, derived from their `UCON` from the previous UPS step or start session).
     - Call arguments for the smart contract function.
     - The `session_proof_tree_root` (root of the user's local recursive proof tree built so far in this UPS).
  2. The Dapen runtime executes the CFC logic locally, simulating state changes to the user's `CSTATE` for that contract.
  3. **Proves:** That the CFC's internal operations faithfully followed its defined code, transforming the assumed `start_contract_state_tree_root` to the `end_contract_state_tree_root` and producing the claimed outputs, given the inputs.
  4. **Output:** A CFC proof and the `transaction_end_ctx` (containing end state roots and output hashes).

**Step 1.3: Integrating CFC into UPS & Recursive Verification (Repeated for each transaction)**

- **Action:** After a CFC proof is generated, it's integrated into the main UPS proof chain.
- **Circuit:** `UPSCFCStandardTransactionCircuit` (or variants for deferred transactions like `UPSVerifyPopDeferredTxStepGadget`).
- **Purpose:** To verify the locally generated CFC proof, ensure it's a legitimate and registered function, and prove that the UPS state (user's `UCON` root, debt trees, transaction counters) is correctly updated based on the CFC's outcome. It also recursively verifies the proof from the previous UPS step.
- Process:
  1. Inputs/Witnesses:
     - The proof from the previous UPS step (or the "start session" proof if this is the first transaction).
     - The `UserProvingSessionHeader` from the previous UPS step.
     - The CFC proof generated in Step 1.2.
     - Witnesses for state updates:
       - A delta Merkle proof for the update to the user's `UCON` tree (showing the leaf for the specific `contract_id` changing from the old `CSTATE` root to the new `CSTATE` root).
       - Pivot Merkle proofs for debt trees (if applicable), showing they transition correctly.
       - Witnesses for the contract function's inclusion in the global `GCON`/`CFT` (fetched from a Realm/Coordinator, anchored to the session's `CHKP` root).
  2. Proves:
     - The previous UPS step's proof was valid and used a whitelisted UPS circuit.
     - The CFC proof (from Step 1.2) is valid, its function is globally registered in the contract's `CFT` (verified against the session's `CHKP` context), and the CFC proof is part of the same UPS proof tree.
     - The `UPSCFCStandardStateDeltaInput` (derived from the CFC's `tx_ctx_header`) correctly links to the CFC's verified computation.
     - The user's `UCON` root is correctly updated based on the `user_contract_tree_update_proof` (i.e., the specific contract's `CSTATE` root within `UCON` is updated).
     - Debt tree roots transition correctly.
     - Transaction count is incremented, and the transaction hash stack is updated.
  3. **Output:** A new UPS step proof and an updated `UserProvingSessionHeader` reflecting the new state after this transaction.

**Step 1.4: Session Finalization & Signature**

- **Action:** Once all desired transactions have been processed locally.
- **Circuit:** `UPSStandardEndCapCircuit` is executed locally.
- **Purpose:** To securely conclude the UPS, producing the final "End Cap" proof. This proof attests to the validity of the entire sequence of local transaction proofs, is authorized by the user's ZK-based signature, and ensures the session ends in a clean state (e.g., no outstanding debts).
- Process:
  1. Inputs/Witnesses:
     - The proof from the last transaction step in the UPS.
     - The final `UserProvingSessionHeader`.
     - A ZK Signature Proof: This is a proof generated by the user's chosen "signature circuit." The signature circuit takes as input a sighash (derived from `PsyUserProvingSessionSignatureDataCompactGadget`, which includes `start_user_leaf_hash`, `end_user_leaf_hash`, `checkpoint_leaf_hash`, `tx_stack_hash`, `tx_count`, nonce, etc.) and proves that the conditions for signing are met (e.g., if it's a traditional key, it verifies an ECC signature; if it's programmatic, it checks defined logic).
  2. Proves:
     - The last UPS step proof was valid and used a whitelisted circuit.
     - The ZK Signature proof is valid and was generated by a circuit whose verifier data hash (plus user params) matches the user's registered public key.
     - The sighash used in the signature proof correctly corresponds to the final state and transaction summary of the UPS.
     - The user's nonce is correctly incremented in their final `ULEAF`.
     - The `last_checkpoint_id` in the final `ULEAF` is correctly updated to the session's `checkpoint_id`.
     - All internal UPS debt trees are empty.
  3. **Output:** The End Cap proof, its public inputs (including `end_cap_result_hash` and `guta_stats_hash`), and the state deltas (the actual changed leaf values in the user's `CSTATE` trees).

**Step 1.5: Submission to Realm**

- **Action:** The user (or client) submits the generated End Cap proof, its public inputs, and the associated state deltas to a **Realm** node responsible for their User ID range. This is typically done via a Realm Edge API.

## Phase 2: Realm-Level Ingestion & Initial Aggregation (GUTA Entry)

Realms are the first point of contact for user-submitted proofs on the network. They verify these proofs and begin the aggregation process.

**Step 2.1: End Cap Proof Verification & GUTA Header Creation**

- **Action:** A Realm node receives an End Cap proof from a user.

- Circuit(s) (Executed by Proof Miners, orchestrated by Realm):

  - `GUTAVerifySingleEndCapCircuit` (if processing one End Cap in isolation)
  - `GUTAVerifyTwoEndCapCircuit` (if processing a pair of End Caps, the common case for starting aggregation)

- **Purpose:** To securely ingest the user's End Cap proof into the GUTA process, verify its validity against protocol rules and historical state, and transform it into a standard `GlobalUserTreeAggregatorHeader` format.

- Process:

  1. The Realm provides the End Cap proof and its claimed results/stats as witness to the Proof Miner network. It also provides a Merkle proof (`checkpoint_historical_merkle_proof`) showing that the `checkpoint_tree_root` the user based their UPS on was indeed a valid, historical checkpoint.

  2. Proves (within the GUTA circuit):

     - The End Cap proof itself is a valid ZK proof.
     - It was generated by the official, known `UPSStandardEndCapCircuit` (fingerprint check).
     - Its public inputs match the claimed `end_cap_result` and `guta_stats`.
     - The `checkpoint_tree_root` claimed in the `end_cap_result` is verified against the `checkpoint_historical_merkle_proof`.
     - (For `GUTAVerifyTwoEndCapCircuit`): The state transitions from two user End Caps (e.g., `start_leaf_A` -> `end_leaf_A` and `start_leaf_B` -> `end_leaf_B` in `GUSR`) are correctly combined into a single state transition at their Nearest Common Ancestor (NCA) in the `GUSR` tree, using an NCA proof witness. Statistics are summed.

  3. Output:

      

     A

      

     ```
     GlobalUserTreeAggregatorHeader
     ```

     . This header contains:

     - The `guta_circuit_whitelist_root` (ensuring subsequent GUTA steps use valid circuits).
     - The `checkpoint_tree_root` that this aggregation step is anchored to (this will be the *current* block's target `CHKP` root, derived from the historical proof).
     - A `SubTreeNodeStateTransition` representing the change to the `GUSR` (either a single user leaf update or an NCA parent update).
     - Aggregated `GUTAStats`.

  4. The Realm also stores the user's state deltas and relays them to peering Data Availability (DA) Miners.

**Step 2.2: Recursive GUTA Aggregation within the Realm**

- **Action:** The Realm continues to orchestrate the aggregation of `GlobalUserTreeAggregatorHeader`s.
- Circuit(s) (Executed by Proof Miners):
  - `GUTAVerifyTwoGUTACircuit` (aggregates two GUTA sub-proofs)
  - `GUTAVerifyLeftGUTARightEndCapCircuit` / `GUTAVerifyLeftEndCapRightGUTACircuit` (aggregates a GUTA sub-proof with a new End Cap proof)
  - `GUTAVerifyGUTAToCapCircuit` (uses a line proof to propagate a GUTA proof up the tree if no merging is needed)
  - `GUTANoChangeCircuit` (handles cases where no user activity occurred in a subtree, ensuring checkpoint consistency).
- **Purpose:** To recursively combine verified state transitions within the Realm's segment of the `GUSR` tree, building a hierarchical proof tree.
- Process:
  1. Pairs of GUTA headers (or a GUTA header and a newly processed End Cap header) are taken as input.
  2. Proves:
     - Both input proofs (and their headers) are valid and used whitelisted GUTA/EndCap circuits.
     - Both input proofs reference the *same* `checkpoint_tree_root` and `guta_circuit_whitelist_root`.
     - NCA logic correctly combines their respective `GUSR` state transitions into a new transition at their parent node.
     - Statistics are correctly summed.
  3. This process repeats, moving up the Realm's portion of the `GUSR` tree, until a single `GlobalUserTreeAggregatorHeader` (and its corresponding GUTA proof) is produced for the root of the Realm's `GUSR` segment.
- **Output:** A single aggregated GUTA proof from the Realm, representing all user activity within its shard for that block.

**Step 2.3: Submission to Coordinator**

- **Action:** The Realm submits its final aggregated GUTA proof (and the associated header) to a **Coordinator** node.

## Phase 3: Coordinator-Level Aggregation & Global Operations

Coordinators aggregate proofs from all Realms and also handle proofs for global state changes.

**Step 3.1: Aggregating Realm GUTA Proofs**

- **Action:** The Coordinator receives GUTA proofs from multiple Realms.
- **Circuit(s) (Executed by Proof Miners, orchestrated by Coordinator):** Primarily `GUTAVerifyTwoGUTACircuit` and `GUTAVerifyGUTAToCapCircuit`.
- **Purpose:** To combine the state transitions proven by each Realm into a proof for the entire `GUSR` tree.
- **Process:** Similar to Step 2.2, but now aggregating proofs that represent entire Realm subtrees. This continues until a single GUTA proof for the root of the global `GUSR` is obtained.

**Step 3.2: Processing Global Operations**

- **Action:** The Coordinator also processes operations that affect global trees other than `GUSR`.
- Circuits (Executed by Proof Miners, orchestrated by Coordinator):
  - **User Registrations:** `BatchAppendUserRegistrationTreeCircuit`. Proves the correct batch append of new user public key commitments to the `URT` (User Registration Tree).
  - **Contract Deployments:** `BatchDeployContractsCircuit`. Proves the correct batch append of new contract definitions (`CLEAF` data, including `CFT` roots) to the `GCON` (Global Contract Tree).
- **Purpose:** To generate ZK proofs for these global state changes.
- **Output:** Proofs for `URT` updates and `GCON` updates.

**Step 3.3: "Part 1" Aggregation - Combining All Major State Changes**

- **Action:** The Coordinator combines the aggregated GUTA proof (for `GUSR` changes), the User Registration proof (for `URT` changes), and the Contract Deployment proof (for `GCON` changes).
- **Circuit (Executed by Proof Miners, orchestrated by Coordinator):** `VerifyAggUserRegistartionDeployContractsGUTACircuit`.
- **Purpose:** To create a single "Part 1" proof that attests to all major state tree modifications for the block, ensuring they are all consistent with the same underlying `CHKP` root (from the previous block, used as the basis for user sessions and global ops).
- Proves:
  - The input GUTA proof, User Registration proof, and Contract Deployment proof are all valid and used their respective whitelisted circuits.
  - All these proofs are based on the same historical `CHKP` root context.
- **Output:** A "Part 1" proof and a `VerifyAggUserRegistartionDeployContractsGUTAHeader` which summarizes the net changes to `GUSR` root, `URT` root, and `GCON` root, along with aggregated GUTA stats.

## Phase 4: Final Block Proof Generation & Finalization

This is the ultimate step where the entire block's validity is encapsulated in one proof.

**Step 4.1: Final Block Transition Proof**

- **Action:** The Coordinator orchestrates the generation of the final block proof.

- **Circuit (Executed by Proof Miners, orchestrated by Coordinator):** `PsyCheckpointStateTransitionCircuit`.

- **Purpose:** To generate the definitive ZK proof for the current block. This proof verifies all aggregated work from Phase 3, computes the new global state roots, forms the new `PsyCheckpointLeaf`, proves its correct append to the `CHKP` tree, and critically, verifies the *previous block's* `PsyCheckpointStateTransitionCircuit` proof.

- Process:

  1. Inputs/Witnesses:

     - The "Part 1" proof and its header (from Step 3.3).
     - The previous block's `PsyCheckpointStateTransitionCircuit` proof (unless this is the genesis block).
     - The `previous_block_chkp_root` (which is a public input to this circuit, taken from the actual previous block's finalized `CHKP` root).
     - Witnesses for new block metadata like `block_time` and `final_random_seed_contribution`.
     - Merkle proofs for appending to the `CHKP` tree and proving the previous `CHKP` leaf.

  2. Proves:

     - The "Part 1" proof is valid and used the correct circuit.

     - The previous block's `PsyCheckpointStateTransitionCircuit` proof is valid (if not genesis), and its `new_checkpoint_tree_root` (output) matches the `previous_block_chkp_root` (input to current circuit) and the `old_root` of the `CHKP` append proof for the current block. This is the **recursive chain link**.

     - The new

        

       ```
       PsyCheckpointLeaf
       ```

        

       is computed correctly:

       - New global roots for `URT`, `GCON`, `GUSR` are taken from the verified "Part 1" proof's header.
       - Other global tree roots (e.g., `GDT`, `GWT`) are either updated by similar aggregation proofs (not detailed here for brevity but would follow a similar pattern) or carried over if unchanged.
       - New block statistics (combining GUTA stats with block time, randomness, etc.) are computed.

     - The `CHKP` tree append operation is correct, transitioning from the `previous_block_chkp_root` to the `new_checkpoint_tree_root` by appending the newly computed `PsyCheckpointLeaf` hash.

  3. **Public Inputs of this Final Proof:** The `previous_block_chkp_root` and the `new_checkpoint_tree_root`.

- **Output:** The final Block Proof for the current block.

**Step 4.2: Block Finalization & Dissemination**

- **Action:** The Coordinator receives the final Block Proof from the Proof Miner network.
- Process:
  1. The Coordinator verifies this final proof.
  2. If valid, this block is considered finalized. The `new_checkpoint_tree_root` becomes the canonical state root for this block height.
  3. The Coordinator disseminates this final Block Proof and the new `CHKP` root to the network (other Coordinators, Realms, users).
  4. Realms update their local view of the latest finalized `CHKP` root, which users will then use as the basis for their next User Proving Session.

This intricate, end-to-end ZK-proofed journey ensures that every state transition in Psy is cryptographically secured and validated, from individual user actions to the global block state, all while enabling massive parallelism and scalability. The recursive verification of block proofs means that verifying the latest block proof is sufficient to trust the integrity of the entire chain.