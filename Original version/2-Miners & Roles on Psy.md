# Network Roles

A high-level introduction to miners, users and other roles on Psy

# Network Roles in Psy

The Psy blockchain operates through a decentralized network of participants, each fulfilling specific roles crucial for the network's functionality, security, scalability, and resilience. These roles are incentivized through Psy's Proof of Useful Work (PoUW) mechanism, ensuring that network contributions are directly beneficial to its users and overall health. Understanding these roles is key for developers to grasp how the Psy ecosystem functions and how their applications interact with the underlying infrastructure.

There are five primary network roles in Psy:

1. **Users (Agents)**
2. **Proof Miners**
3. **Coordinators**
4. **Realms**
5. **Data Availability (DA) Miners**
6. **(Implicit) Network Vanguards**

## 1. Users (Agents)

Users, also conceptualized as "Agents" in Psy's agent-centric architecture, are the primary actors initiating state changes on the blockchain. They are the consumers and initiators of transactions and smart contract interactions.

**Key Functions & Characteristics:**

- Local Transaction Execution & Proving:
  - Unlike traditional blockchains where users submit transactions to a mempool for network execution, Psy users (or their client software/delegates) execute their smart contract calls (Contract Function Circuits - CFCs) **locally**.
  - For each transaction, a Zero-Knowledge Proof (ZKP) is generated locally, attesting to the correct execution of the contract logic.
  - Multiple local transaction proofs are then recursively combined during a **User Proving Session (UPS)** into a single, compact "End Cap" proof. This End Cap proof summarizes all of a user's activity for a given block.
- State Interaction:
  - Psy utilizes the PARTH state model. Each user has their own isolated state trees for each contract they interact with (`CSTATE` trees, aggregated under their `UCON` tree).
  - A user's transactions can **only write to their own state trees**.
  - Users can **read any global state** (e.g., other users' balances or contract states) but only as it was finalized in the *previous* block.
- Programmable Public Keys (SDKeys):
  - A user's identity is not tied to a fixed cryptographic scheme like secp256k1. Instead, a user's "public key" on Psy is the hash of the verifier data of an arbitrary ZK circuit (the "signature circuit") plus a user-defined parameter.
  - This signature circuit defines the conditions under which a valid signature (which is itself a ZK proof) can be generated for the user's End Cap proof.
  - This enables:
    - Support for existing wallet signatures (e.g., Ethereum, Bitcoin).
    - Custom multi-factor authentication, N/M multisigs.
    - Spending limits, time-locks, or other logic embedded directly into the key.
    - **Autonomous Agents:** Accounts that can operate without a private key, where signature generation is purely a function of the transaction content and blockchain state. This is revolutionary for creating on-chain AMMs, oracles, or AI-driven entities.
- Submission of Proofs & Deltas:
  - After completing a UPS, the user submits their End Cap proof and the corresponding state deltas (the actual changes to their `CSTATE` leaves) to a Realm node.

**Incentives:** Users are not directly incentivized in the PoUW sense but are the primary beneficiaries of the network's services (secure, scalable, private computation and state). They pay transaction fees, which contribute to incentivizing other network roles.

**Why this is useful for other Users/Developers:**

- **Privacy:** Sensitive data used in local transaction execution never leaves the user's device unencrypted.
- **Control & Flexibility:** SDKeys give users unprecedented control over their account security and functionality.
- **Scalability:** Local proving offloads significant computational burden from the core network.
- **Agentic Possibilities:** Developers can design applications with truly autonomous on-chain components.

## 2. Proof Miners

Proof Miners are the computational backbone of Psy's ZK-rollup-like architecture. They perform the intensive Zero-Knowledge proof generation work required to aggregate user proofs and finalize block state transitions.

**Key Functions:**

- Aggregating User Proofs (GUTA):
  - Proof Miners take End Cap proofs (submitted by users to Realms) and recursively aggregate them using specialized ZK circuits (GUTA - Global User Tree Aggregation circuits).
  - This aggregation process typically involves verifying pairs of proofs (either two End Cap proofs or two lower-level GUTA proofs) and generating a new, single proof that attests to the validity of both children and proves the correct state transition at their Nearest Common Ancestor (NCA) in the Global User Tree (`GUSR`).
  - This process continues hierarchically until a Realm generates a single GUTA proof for all user activity within its segment of the `GUSR`.
- Aggregating Realm & Global Proofs:
  - Coordinators direct Proof Miners to further aggregate the GUTA proofs from different Realms.
  - They also generate proofs for global operations like new contract deployments (`BatchDeployContractsCircuit`) and new user registrations (`BatchAppendUserRegistrationTreeCircuit`).
- Generating the Final Block Proof:
  - Ultimately, Proof Miners generate the `PsyCheckpointStateTransitionCircuit` proof. This is the single ZK proof for the entire block, which verifies all aggregated GUTA, contract deployment, and user registration proofs. Crucially, it also verifies the previous block's proof, creating an unbroken chain of cryptographic validity back to genesis.

**Incentives:**

- Percentage of Non-State Transaction Fees:

   

  Proof Miners receive a share of the non-state transaction fees from the End Cap proofs they help aggregate. The distribution is proportional to the number of proofs they successfully aggregate within a block, ensuring that miners who contribute more earn more.

  - `rewards_for_miner_k = min(total_non_state_tx_fees_paid_in_block_n, block_reward_floor) * (number_of_proofs_aggregated_in_block_n_by_miner_k / total_proofs_aggregated_in_block_n) * C` (where C is a constant close to 1).

- **Fixed Per-Block Floor Reward:** A baseline reward per block ensures network stability and incentivizes participation even during periods of lower transaction volume.

**Why their work is useful for Users/Developers:**

- **Enables Trustless Verification:** Their primary utility is transforming potentially millions of individual user transaction proofs into a single, small, and quickly verifiable block proof. This allows anyone (even on a light client or smartwatch) to verify the entire state of the Psy blockchain from genesis in milliseconds.
- **Scalability Engine:** The network's ability to process more transactions is directly tied to the available computational power of Proof Miners. More Proof Miners mean more parallel proof aggregation, leading to higher throughput.
- **Security:** By generating these complex ZKPs, they are integral to the "Proof of Math" security model of Psy.
- **Aligned Incentives:** Proof Miners are incentivized to process as many user proofs as quickly and efficiently as possible, as their rewards scale with the volume of useful work (aggregation) they perform. This directly benefits users by ensuring their transactions are included and finalized.

## 3. Coordinators

Coordinators act as high-level orchestrators within the Psy network. They manage the overall block construction process, interfacing between Realms, Proof Miners, and global state components. While they play a critical role, they do not validate transactions themselves but rather manage the workflow of proof aggregation.

**Key Functions:**

- **Receiving Realm Proofs:** Collect aggregated GUTA proofs from all active Realms for the current block.
- Orchestrating Final Aggregation:
  - Submit these Realm GUTA proofs, along with witnesses for global operations (user registrations, contract deployments), to the decentralized Proof Miner network for further aggregation into the "Part 1" proof (e.g., `VerifyAggUserRegistartionDeployContractsGUTACircuit`).
- Managing Global State Components:
  - Store and provide access to deployed smart contract code (bytecode) and their `CFT` (Contract Function Tree) roots, which whitelist valid function fingerprints.
  - Store a copy of every user's public key (the hash of their signature circuit's verifier data + params).
  - Maintain the `CHKP` (Checkpoint Tree), an append-only Merkle tree where the Nth leaf is the Nth block's `CHKP` root.
  - Store the top levels of the `GUSR` (Global User Tree), where each leaf points to a Realm's `GUSR` sub-tree root.
- Processing Global Operations:
  - Handle API requests for new user registrations and smart contract deployments.
  - Generate the witnesses for the ZK proofs of these operations (`BatchAppendUserRegistrationTreeCircuit`, `BatchDeployContractsCircuit`).
- Final Block Proof Orchestration:
  - Generate the witness for the final block proof (`PsyCheckpointStateTransitionCircuit`).
  - Relay this final proving job to the Proof Miner network.
  - Store and disseminate the latest final block proof.
- **Block Timing & Synchronization:** Responsible for setting the "tic-toc" block time and syncing key data (like the latest `CHKP` root) to Realms after each block.

**Incentives:**

- **Percentage of Non-State Transaction Fees:** Coordinators receive a share of the non-state transaction fees from all transactions included in the blocks they successfully coordinate.

**Why their work is useful for Users/Developers:**

- **Ensuring Block Finalization:** They are essential for bringing together all the parallel work done by users and Realms into a single, coherent, and provable block.
- **Global State Consistency:** They maintain critical global data structures (like `GCON` and `URT`) that are necessary for the functioning of smart contracts and user identity.
- **Facilitating Network Growth:** By processing user registrations and contract deployments, they enable the network to expand.
- **Aligned Incentives:** Coordinators are incentivized to maximize the number of user transactions included in each block. This means they strive for efficient communication with Realms, rapid relay of proving jobs to Proof Miners, and high availability of essential data like contract code, all of which benefit users and dApp developers.

## 4. Realms

Realms are distributed ingestion and initial aggregation points for user transactions and state. They manage specific segments (shards) of the Global User Tree (`GUSR`).

**Key Functions:**

- Receiving User Submissions:
  - Users submit their End Cap proofs and corresponding state deltas to an API endpoint provided by a Realm responsible for their User ID range.
- Initial Proof Aggregation (GUTA):
  - Realms verify incoming End Cap proofs.
  - They generate witnesses for these End Cap proofs and submit them to the decentralized Proof Miner network to be aggregated using GUTA circuits (e.g., `GUTAVerifySingleEndCapCircuit`, `GUTAVerifyTwoEndCapCircuit`).
  - This aggregation continues recursively within the Realm until a single GUTA proof is generated, representing the net state transition for the Realm's entire segment of the `GUSR` for that block.
- State Delta Management:
  - Store the state deltas received from users.
  - Relay these state deltas to Data Availability (DA) Miners that peer with the Realm, ensuring data persistence and accessibility.
- Data Serving:
  - Store and serve user-specific data for users within their assigned `GUSR` segment. This includes the Realm's portion of the `GUSR`, the `ULEAF` data for each user, and all `UCON` and `CSTATE` trees for those users. This data is essential for users to construct proofs for new transactions.
- **Submitting Realm Proofs:** Send their final aggregated Realm GUTA proof to the Coordinators for inclusion in the overall block proof.

**Incentives:**

- **Percentage of Non-State Transaction Fees:** From users whose transactions are processed within their Realm.
- **Percentage of Stateful Transaction Fees:** From users within their Realm, reflecting the storage and data serving aspects of their role.

**Why their work is useful for Users/Developers:**

- **Scalable Ingestion Point:** By sharding user submissions, Realms prevent a single point of congestion for incoming transactions.
- **Data Accessibility for Local Proving:** They (along with DA Miners) provide users with the necessary historical state data to execute and prove their transactions locally.
- **Efficient User Experience:** Realms are incentivized to provide a fast, reliable service for their users, including quick acceptance of End Cap proofs and efficient data retrieval, because their income depends on user activity within their segment.
- **Decentralized Aggregation:** They initiate the distributed proof aggregation process, contributing to the overall scalability and security of the network.
- **No Anti-User MEV:** Like Coordinators, Realms only make money when their users successfully transact. There's no incentive structure for them to engage in front-running or other exploitative behaviors common on other platforms.

## 5. Data Availability (DA) Miners

DA Miners are responsible for the robust and decentralized storage of the Psy blockchain's state, particularly the granular user contract state (`CSTATE`) data.

**Key Functions:**

- Storing Chain Data:
  - Receive state deltas from Realms.
  - Store and maintain copies of user `CSTATE` trees and other relevant blockchain state data.
- Proving Data Availability:
  - Periodically, DA Miners must generate ZK proofs attesting that they are correctly storing specific, randomly challenged pieces of data (e.g., a Merkle proof for a random leaf in a user's `CSTATE` tree or a global state tree).
- Serving Data to Users:
  - Make the stored state data available to users who need it to construct proofs for their transactions (often via Realms).

**Incentives:**

- **Portion of Stateful Transaction Fees:** Rewarded based on successfully proving storage of randomly challenged data. This ensures they are incentivized to store the data correctly and comprehensively.
- **Fees from Affiliated Realms:** DA Miners may peer with specific Realms and receive a portion of the fees those Realms earn, incentivizing them to serve data efficiently to the users of those Realms.

**Why their work is useful for Users/Developers:**

- **Data Persistence & Resilience:** DA Miners ensure that all blockchain state is redundantly stored and resistant to loss or censorship. This is critical for the long-term integrity of the chain.
- **Accessibility for Proving:** They guarantee that users can always retrieve the necessary historical state to prove new transactions, which is fundamental to Psy's local proving model.
- **Censorship Resistance:** Decentralized data storage makes it extremely difficult for any single entity to censor or deny access to blockchain data.
- **Open Verifiable Ledger:** Their work underpins the ability for anyone to independently verify balances and smart contract information.

## 6. Network Vanguards (Protocol Enforcement)

Network Vanguards are not continuously operating nodes in the same way as the others but serve crucial roles in maintaining network health, fairness, and efficiency through monitoring and intervention capabilities.

**Key Functions:**

- Type A (Realm Peering/Sync Monitoring):
  - Monitor and test the real-time peering and data synchronization efficiency between the multiple decentralized operators running a specific Realm. (Realms are operated by a rotating set of entities).
- Type B (Coordinator Peering/Sync Monitoring):
  - Monitor and test the real-time peering and data synchronization efficiency between the multiple decentralized operators fulfilling the Coordinator role (which is also rotational).
- Type C (Censorship Backstop):
  - If a Realm appears to be censoring a user's transactions (i.e., not including their valid End Cap proof), a Type C Vanguard can provide a proof on-chain of this valid, unincluded transaction.
  - This forces the censoring Realm operators to either include the transaction in a subsequent block or risk losing their operational slot (and potentially a staked deposit) for that Realm.

**Incentives:**

- **Fixed Share of Non-Stateful Fees:** A small, fixed portion of block rewards is allocated to Vanguards for their ongoing monitoring services.
- **Slashing Rewards (for Type C):** If a Type C Vanguard successfully forces the inclusion of a censored transaction due to Realm operator non-compliance, they may receive a portion of the slashed deposit from the non-compliant Realm operator(s).

**Why their work is useful for Users/Developers:**

- **Network Health & Efficiency:** Types A and B help ensure the underlying communication infrastructure between decentralized operators is robust and performant, which is vital for low latency and high throughput.
- **Censorship Resistance:** Type C Vanguards provide a powerful mechanism to deter and penalize censorship at the Realm level, ensuring fair access for all users. This is a critical guarantee for a truly open and permissionless platform.
- **Maintaining Protocol Integrity:** They act as watchdogs, ensuring that the decentralized operators of critical infrastructure (Realms, Coordinators) adhere to protocol rules.

By distributing these responsibilities and aligning incentives through PoUW, Psy creates a robust, scalable, and user-centric ecosystem where each participant contributes meaningfully to the network's success.