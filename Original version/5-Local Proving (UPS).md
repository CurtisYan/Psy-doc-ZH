# Local Proving

An overview of how local proving (aka. User Proving Sessions/UPS) works on Psy

# Psy Network: Local User Proving Session (UPS) Deep Dive

The User Proving Session (UPS) is a cornerstone of Psy's architecture, enabling users to process their transactions locally, maintain privacy, and contribute to the network's overall scalability. This document provides a detailed exploration of the UPS, its constituent circuits, and the cryptographic guarantees it provides. Developers building client applications, wallets, or tools that interact with Psy at a low level will find this information essential.

## Overview of the User Proving Session

A UPS is a sequence of operations performed locally by a user (or their delegate, such as client software) to:

1. **Establish a Baseline:** Anchor the session to a globally finalized state of the Psy blockchain.
2. **Execute Transactions:** Run the logic of smart contract functions (Contract Function Circuits - CFCs) for each desired transaction.
3. **Prove Correctness:** Generate Zero-Knowledge Proofs (ZKPs) for each CFC execution.
4. **Maintain State Consistency:** Prove that the user's local state (their `UCON` tree root, debt trees, etc.) is updated correctly based on the outcomes of the executed CFCs.
5. **Recursive Verification:** Chain these proofs together recursively, so each step verifies the previous one.
6. **Authorize:** Cryptographically sign the entire session's outcome using the user's ZK-powered signature circuit (SDKey).
7. **Produce a Summary:** Output a single, compact "End Cap" proof that attests to the validity of all transactions within the session.

This End Cap proof, along with the net state deltas (changes to the user's `CSTATE` leaves), is then submitted to a Realm node for inclusion in the next block.

## Key Circuits and Gadgets in a UPS

The UPS relies on a suite of specialized ZK circuits and gadgets. "Gadgets" are reusable ZK circuit components.

### 1. `UPSStartSessionCircuit` - Initiating the Session

- **Purpose:** To securely initialize the UPS, ensuring it starts from a consistent and globally validated state for the user.

- **Core Gadget:** `UPSStartStepGadget`

- Inputs/Witnesses (provided by user's client):

  - ```
    UPSStartStepInput
    ```

    :

    - ```
      header_witness
      ```

      : A target

       

      ```
      UserProvingSessionHeaderGadget
      ```

       

      that the circuit will populate and constrain. Key fields within its

       

      ```
      session_start_context
      ```

      :

      - `checkpoint_tree_root`: The hash of the `CHKP` root of the last globally finalized block the user is basing this session on.
      - `checkpoint_leaf_hash`: The hash of the `PsyCheckpointLeaf` corresponding to `checkpoint_tree_root`.
      - `checkpoint_id`: The block height/index of this checkpoint.
      - `start_session_user_leaf`: The user's `PsyUserLeaf` data as it existed at this checkpoint.

    - `checkpoint_leaf_gadget_witness`: The actual `PsyCheckpointLeafCompactWithStateRoots` data.

    - `state_roots_gadget_witness`: The `PsyGlobalStateRoots` contained within the `checkpoint_leaf_gadget_witness`.

    - `checkpoint_tree_proof_witness`: A Merkle proof showing that `checkpoint_leaf_hash` is a valid leaf in the tree rooted at `checkpoint_tree_root` at `checkpoint_id`.

    - `user_tree_proof_witness`: A Merkle proof showing that the hash of `start_session_user_leaf` is a valid leaf in the `GUSR` (Global User Tree) rooted at `state_roots_gadget_witness.user_tree_root`, at the user's `user_id`.

- What it Proves:

  1. **Checkpoint Validity:** The provided `checkpoint_leaf_hash` is indeed part of the claimed `checkpoint_tree_root` at the specified `checkpoint_id` (verified using `checkpoint_tree_proof_witness`).

  2. **Global Context Consistency:** The hash of `checkpoint_leaf_gadget_witness` matches `checkpoint_leaf_hash`. The hash of `state_roots_gadget_witness` matches the `global_chain_root` within `checkpoint_leaf_gadget_witness`.

  3. **User State Validity:** The user's `start_session_user_leaf` (hashed) is a valid leaf in the `GUSR` (rooted at `state_roots_gadget_witness.user_tree_root`) at the correct `user_id` (verified using `user_tree_proof_witness`).

  4. Header Initialization:

     - The `session_start_context` in the output `UserProvingSessionHeader` is correctly populated from the verified inputs.

     - The

        

       ```
       current_state
       ```

        

       within the header is initialized:

       - `user_leaf` is set to `start_session_user_leaf`, but with `last_checkpoint_id` updated to the current session's `checkpoint_id`.
       - `deferred_tx_debt_tree_root` and `inline_tx_debt_tree_root` are set to `EMPTY_TREE_ROOT`.
       - `tx_count` is set to 0.
       - `tx_hash_stack` (a running hash of transactions) is set to `ZERO_HASH`.
       - `ups_step_circuit_whitelist_root` is set to a known constant for subsequent UPS step circuits.

- **Output:** A ZK proof for this start step, and the initialized `UserProvingSessionHeader`.

- **Significance:** This step is crucial for preventing users from starting sessions based on invalid or outdated personal states. It anchors the entire UPS to a verifiable global consensus point.

### 2. `DapenContractFunctionCircuit` (CFC) - Executing Smart Contract Logic

- **Purpose:** To execute the logic of a specific smart contract function chosen by the user and generate a ZK proof of this local execution. This circuit is *defined by the smart contract developer* using the Dapen toolchain.

- **Core Gadget (Framework):** `PsyContractFunctionBuilderGadget` (used by Dapen compiler).

- Inputs/Witnesses (provided by user's client for each call):

  - ```
    DapenContractFunctionCircuitInput
    ```

    :

    - ```
      tx_input_ctx
      ```

       

      (Transaction Input Context):

      - ```
        transaction_call_start_ctx
        ```

        : Contains the

         

        assumed

         

        starting state for

         

        this specific contract interaction

        :

        - `start_contract_state_tree_root`: The root of the user's `CSTATE` tree for this contract *before* this function call (derived from the user's `UCON` in the previous UPS step).
        - `start_user_balance`, `start_user_event_index`.
        - `start_deferred_tx_debt_tree_root`, `start_inline_tx_debt_tree_root` (from previous UPS step).

      - `call_data_hash`, `call_data_length`: Hash and length of the function arguments.

      - `contract_id`, `method_id`.

    - `circuit_inputs`: The actual arguments for the smart contract function.

    - `session_proof_tree_root`: The root of the user's recursive proof tree built so far within this UPS. This is used for "tree-aware" proofing, ensuring this CFC proof is part of the larger session's proof structure.

- What it Proves:

  1. **Faithful Execution:** The sequence of internal operations performed by the circuit exactly matches the compiled `DPNFunctionCircuitDefinition` (the smart contract's code for that function).
  2. **State Transformation:** Given the `transaction_call_start_ctx` and `circuit_inputs`, the execution correctly produces the `end_contract_state_tree_root` (new root of the user's `CSTATE` for this contract) and `end_deferred_tx_debt_tree_root` recorded in the output `transaction_end_ctx`.
  3. **Output Consistency:** The computed `outputs_hash` and `outputs_length` (from the function's return values) match those recorded in `transaction_end_ctx`.
  4. **Assertions:** All assertions defined within the smart contract function's code hold true during this execution.
  5. **Public Input Hashing:** The circuit's main public input (a hash combining `session_proof_tree_root` and the hash of `tx_input_ctx`) is correctly computed.

- **Output:** A ZK proof for this specific CFC execution instance (CFC Proof).

- **Significance:** This is where the actual smart contract logic runs. Its ZK proof guarantees that the user's local execution was correct according to the contract code, without revealing the intermediate states or inputs if the contract is designed for privacy.

### 3. `UPSCFCStandardTransactionCircuit` - Integrating a CFC Execution

- **Purpose:** To take a verified CFC proof (from step 2) and integrate its effects into the main UPS proof chain. It verifies the CFC's validity and proves that the overall UPS state (user's `UCON` root, debt trees, transaction counters) is correctly updated. It also recursively verifies the proof from the previous UPS step.

- Core Gadgets:

  - `VerifyPreviousUPSStepProofInProofTreeGadget`: Verifies the ZK proof of the immediately preceding UPS step.
  - `UPSVerifyCFCProofExistsAndValidGadget`: Verifies the CFC proof (from step 2), its inclusion in the session's proof tree, and that the function called is globally registered in the contract's `CFT`.
  - `UPSCFCStandardStateDeltaGadget`: Calculates and proves the changes to the `UserProvingSessionHeader`.

- Inputs/Witnesses (provided by user's client):

  - ```
    UPSVerifyCFCStandardStepInput
    ```

    :

    - The ZK proof of the previous UPS step.
    - The `UserProvingSessionHeaderGadget` output by the previous UPS step.
    - The CFC Proof (from step 2) and its `transaction_end_ctx`.
    - `PsyContractFunctionInclusionProof`: A Merkle proof showing the CFC's fingerprint is in the contract's `CFT` (rooted in `GCON`, which is anchored to the session's `checkpoint_leaf_hash`).
    - `user_contract_tree_update_proof`: A `DeltaMerkleProofCore` proving the update to the user's `UCON` tree. This shows the leaf at `contract_id` changing from the `start_contract_state_tree_root` (from CFC's `start_ctx`) to `end_contract_state_tree_root` (from CFC's `end_ctx`).
    - `deferred_tx_debt_pivot_proof`, `inline_tx_debt_pivot_proof`: `MerkleProofCore`s showing the debt tree roots transition correctly from their state in the previous UPS header to their state in the CFC's `end_ctx`.

  - `current_proof_tree_root`: The root of the UPS proof tree *after* including the current CFC proof and this step's proof.

- What it Proves:

  1. **Previous Step Validity:** The previous UPS step's proof is valid, was generated by a whitelisted UPS circuit, and its public inputs match the hash of the previous `UserProvingSessionHeader`.
  2. CFC Validity & Legitimacy:
     - The CFC proof is valid and is part of the `current_proof_tree_root`.
     - The function executed by the CFC is legitimate (its fingerprint is in the contract's `CFT`, which is part of the global state referenced by `previous_step_header.session_start_context.checkpoint_leaf_hash`).
     - The `cfc_inner_public_inputs_hash` (derived from the CFC's `tx_input_ctx`) from the verified CFC proof matches the one used by `UPSCFCStandardStateDeltaGadget`. This is a **critical link** ensuring the state delta calculation is based on the *actual verified computation*.
  3. State Delta Correctness (via `UPSCFCStandardStateDeltaGadget`):
     - **`UCON` Update:** The `user_contract_tree_update_proof` correctly updates the `UCON` root from `previous_step_header.current_state.user_leaf.user_state_tree_root` to a new root reflecting the change for `contract_id` from `start_contract_state_tree_root` to `end_contract_state_tree_root`.
     - **Debt Tree Consistency:** The `deferred/inline_tx_debt_pivot_proofs` show that the debt tree roots in the CFC's `end_ctx` are valid successors to the debt tree roots in the `previous_step_header` (or `CorrectUPSHeaderHashesGadget` if overridden).
     - **Counter & Stack:** `tx_count` in the new header is incremented. `tx_hash_stack` is updated by hashing the previous stack with a `TransactionLogStackItemGadget` (containing CFC call details).
     - The `current_state.user_leaf` in the new header is updated (e.g., `user_state_tree_root` now reflects the new `UCON` root).

- **Output:** A ZK proof for this transaction step, and the new `UserProvingSessionHeaderGadget`.

- **Significance:** This is the workhorse circuit of the UPS, meticulously linking individual contract executions to the user's overall session state in a verifiable, recursive manner.

*(Steps 2 and 3 are repeated for every transaction the user wishes to include in their session.)*

### 4. `UPSStandardEndCapCircuit` - Finalizing the Session

- **Purpose:** To securely conclude the UPS, producing the single End Cap proof. It verifies the entire chain of local proofs, validates the user's ZK signature authorizing the session, and ensures the session terminates in a defined, clean state.

- Core Gadgets:

  - ```
    UPSEndCapFromProofTreeGadget
    ```

    : Orchestrates the final verifications.

    - Internally uses `VerifyPreviousUPSStepProofInProofTreeGadget` (often the `PartialFromCurrent` variant) to verify the last transaction step's proof.
    - Uses `AttestProofInTreeGadget` to verify the user's ZK Signature Proof and its inclusion in the session proof tree.

  - `UPSEndCapCoreGadget`: Enforces final session constraints.

- Inputs/Witnesses (provided by user's client):

  - ```
    UPSEndCapFromProofTreeInput
    ```

    :

    - Witnesses for verifying the last UPS transaction step's proof (attestation, previous state, whitelist proof).
    - Witnesses for verifying the ZK Signature Proof (attestation, public key parameters of the signature circuit).
    - `user_public_key_param`: The user-defined parameter part of their SDKey.
    - `nonce`: The new nonce value for the user.
    - `slots_modified`: A count of state slots modified (for stats).

  - The ZK Signature Proof itself. This proof is generated by the user's chosen signature circuit. The signature circuit's public input is a sighash derived from `PsyUserProvingSessionSignatureDataCompactGadget`.

  - `PsyUserProvingSessionSignatureDataCompactGadget` (witness): Contains `start_user_leaf_hash`, `end_user_leaf_hash` (from the final `UserProvingSessionHeader`), `checkpoint_leaf_hash` (from session start), `tx_stack_hash`, and `tx_count` (from final header).

- What it Proves (via `UPSEndCapCoreGadget` and its constituents):

  1. **Last Step Validity:** The proof of the last transaction step in the UPS is valid and used a whitelisted UPS circuit.
  2. **Signature Proof Validity:** The provided ZK Signature Proof is valid and is part of the same session proof tree as the last transaction step.
  3. Authorization:
     - The fingerprint of the signature circuit (from `sig_proof_verifier_data`) combined with `user_public_key_param` correctly reconstructs the user's registered public key (which is present in the `start_user_leaf` and `end_user_leaf` within the final `UserProvingSessionHeader`).
     - The public input of the ZK Signature Proof matches the sighash computed from the `PsyUserProvingSessionSignatureDataCompactGadget` witness, which in turn is derived from the final state of the `UserProvingSessionHeader`. This confirms the signature authorizes *this specific session's outcome*.
  4. **Nonce Progression:** The `nonce` witness is greater than `last_header_gadget.current_state.user_leaf.nonce` (from the start of the session), and the `end_user_leaf.nonce` is updated to this new `nonce`.
  5. **Checkpoint Progression:** `last_header_gadget.current_state.user_leaf.last_checkpoint_id` is updated to `last_header_gadget.session_start_context.checkpoint_id` (and this must be greater than the original `last_checkpoint_id`).
  6. **Clean State:** The `deferred_tx_debt_tree_root` and `inline_tx_debt_tree_root` in the `last_header_gadget.current_state` are `EMPTY_TREE_ROOT`.
  7. Output Generation:
     - `UPSEndCapResultCompactGadget` is correctly formed using `start_user_leaf_hash`, `end_user_leaf_hash`, `checkpoint_tree_root_hash` (from session start context), and `user_id`. Its hash is a public output.
     - `GUTAStatsGadget` is correctly formed (fees, transaction counts, slots modified). Its hash is a public output.

- **Output:** The End Cap ZK Proof. Its public inputs are the `end_cap_result_hash` and `guta_stats_hash`.

- **Significance:** This is the culmination of the user's local efforts. The End Cap proof is a compact, verifiable assertion of all the user's actions in the block, ready for efficient processing by the Psy network.

## The UPS Proof Tree

Throughout the UPS, a local Merkle tree of proofs is implicitly constructed.

- The leaves of this tree are the ZK proofs for individual CFC executions and the ZK Signature Proof.
- Intermediate nodes are the ZK proofs from `UPSCFCStandardTransactionCircuit` (or variants) and `UPSStartSessionCircuit`.
- The root of this tree is referenced by `current_proof_tree_root` in various gadgets and is implicitly verified by the chain of recursive verifications. The `UPSEndCapCircuit` can optionally include an explicit proof of this tree's aggregation if needed for certain advanced scenarios or cross-UPS interactions, though typically the recursive verification suffices.

By offloading transaction execution and initial proof generation to users, the UPS dramatically reduces the computational load on the main Psy network, enabling it to focus on aggregation and maintaining global consensus. This local-first approach is fundamental to Psy's privacy, scalability, and user-centric design.