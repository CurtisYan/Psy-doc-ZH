# 全局用户树聚合：一篇指南



深入解释如何将用户的证明聚合到区块中。



# Psy 网络：全局用户树聚合 (GUTA)



一旦用户在本地处理完他们的交易并通过其**用户证明会话 (User Proving Sessions, UPS)** 生成了 "End Cap" 证明，这些证明就会被提交到 Psy 网络。**全局用户树聚合 (Global User Tree Aggregation, GUTA)** 是一个精密、多层的零知识证明 (ZKP) 流程，网络通过该流程，能够安全且可扩展地将可能数百万个独立用户的状态变更，整合为对**全局用户树 (`GUSR`)** 的单次可验证更新，并最终更新整个区块链状态。

本文档详细介绍了 GUTA 流程，该流程主要发生在 Realm 内部，然后由 Coordinator 进一步聚合，所有这些都由去中心化的**证明矿工 (Proof Miners)** 提供算力支持。



## GUTA 概述



GUTA 的核心目标是：

1. **验证用户 End Cap 证明**：确保每个提交的 End Cap 证明都是有效的，由正确的电路生成，基于有效的历史区块链状态，并得到了妥善授权。
2. **证明 `GUSR` 状态转换**：对于每个有效的 End Cap 证明（或一对证明），证明 `GUSR` 中相应叶子节点（或最近公共祖先 - Nearest Common Ancestor, NCA - 节点）的正确状态转换。`GUSR` 存储了所有用户的 `ULEAF` 数据，包括他们的 `UCON` 根、余额和 nonce。
3. **递归聚合**：使用 ZKP 按层级结构地将这些单独的 `GUSR` 转换证明进行组合。这意味着为较小的 `GUSR` 子树生成的证明会被聚合成更大子树的证明，最终汇集成一个对整个 `GUSR` 更新的证明。
4. **维护一致性**：确保所有聚合的证明都与同一个全局检查点 (`CHKP` 根) 保持一致，并遵守协议定义的电路白名单。
5. **收集统计数据**：从用户会话中聚合统计信息（如费用、交易计数等）。

GUTA 对 Psy 的水平可扩展性至关重要。它允许网络并行处理海量用户更新，然后高效地证明这些更新的集体影响。



## 阶段一：Realm 级别的 GUTA - 接收与初始聚合



Realm 负责 `GUSR` 的特定分段（分片）。它们作为用户 End Cap 证明的入口点，并执行 GUTA 的初始聚合层。

**步骤 1：End Cap 证明的接收与验证 (GUTA 入口点)**

- **动作**：一个 Realm 从其分配的用户 ID 范围内的用户那里接收到一个 End Cap 证明（及其公共输入：`end_cap_result_hash`, `guta_stats_hash`，外加底层的 `end_cap_result` 和 `guta_stats` 数据）。

- **电路** (由 Realm 编排，由证明矿工执行):

  - `GUTAVerifySingleEndCapCircuit`：当 Realm 处理单个 End Cap 证明时使用，可能是在聚合树中作为叶子节点，或者当一个子树中只有一个用户活跃时。
  - `GUTAVerifyTwoEndCapCircuit`：更常见的聚合基础场景，处理一对 End Cap 证明。

- **输入/见证数据 (Inputs/Witnesses)** (由 Realm 提供给证明矿工):

  - 一个或多个 End Cap 证明对象 (`proof_target`) 及其验证者数据。
  - 与 End Cap 证明相对应的、声称的 `end_cap_result` 和 `guta_stats` 数据。
  - `checkpoint_historical_merkle_proof`：一个 Merkle 证明，用于证实用户 `end_cap_result` 中声称的 `checkpoint_tree_root_hash` 是 Psy 链上一个有效的、历史的 `CHKP` 根。此证明锚定于**当前**区块的目标 `CHKP` 根（该根本身源自前一个区块最终化的 `CHKP` 根）。
  - (对于 `GUTAVerifyTwoEndCapCircuit`)：一个 `UpdateNearestCommonAncestorProof` 见证数据，这是一个 Merkle 证明，详细说明了两个独立用户的 `ULEAF` 转换如何在它们 `GUSR` 中的 NCA 节点处合并。
  - `known_end_cap_fingerprint_hash`：一个常量，代表有效的 `UPSStandardEndCapCircuit` 的指纹。
  - `guta_circuit_whitelist_root_hash`：一个 Merkle 树的根，该树将有效的 GUTA 聚合电路列入白名单（作为 GUTA 电路的公共输入）。

- **它证明了什么**:

  1. **End Cap 证明的有效性**：每个输入的 End Cap 证明都是一个有效的 ZK 证明，并且是由指纹为 `known_end_cap_fingerprint_hash` 的电路生成的。
  2. **公共输入匹配**：End Cap 证明的公共输入与所提供的 `end_cap_result` 和 `guta_stats` 数据正确匹配。
  3. **历史锚定**：用户 `end_cap_result` 中引用的 `checkpoint_tree_root_hash` 通过 `checkpoint_historical_merkle_proof` 进行验证。这确保了 UPS 是基于一个合法的过去状态。该历史证明的 `current_root` 成为输出的 GUTA 头部的 `checkpoint_tree_root`。
  4. **`GUSR` 状态转换** (通过 `end_cap_result`):
     - 对于 `GUTAVerifySingleEndCapCircuit`：`end_cap_result`（包含 `start_user_leaf_hash`, `end_user_leaf_hash`, `user_id`）为 `GUSR` 中的单个 `ULEAF` 定义了一个直接的转换。
     - 对于 `GUTAVerifyTwoEndCapCircuit`：`UpdateNearestCommonAncestorProofOptGadget` 验证了两个独立的 `ULEAF` 转换（源自两个 `end_cap_results`）正确地合并，从而在它们的 NCA 节点处产生了所声称的状态转换。
  5. **统计数据聚合**：(对于 `GUTAVerifyTwoEndCapCircuit`)：`GUTAStatsGadget.combine_with` 正确地将两个输入 End Cap 的 `guta_stats` 求和。

- 输出:

  此 GUTA 步骤的一个 ZK 证明，以及一个 GlobalUserTreeAggregatorHeaderGadget。此头部包含：

  - `guta_circuit_whitelist_root_hash` (向前传递)。
  - `checkpoint_tree_root` (源自 `checkpoint_historical_merkle_proof.current_root`)。
  - `state_transition`：一个 `SubTreeNodeStateTransitionGadget`，描述了已证明的对 `GUSR` 的变更（例如，在 `GUSR` 的特定 `index` 和 `level` 处，`old_node_value` -> `new_node_value`）。
  - `stats`：经过（可能）聚合的 `GUTAStatsGadget`。

- **意义**：这是用户数据进入网络聚合管道的安全网关。它在用户会话被合并之前，严格验证了每个会话的有效性。

**步骤 2：Realm 内的递归 GUTA 聚合**

- **动作**：Realm 继续编排对其 `GUSR` 分段内，由步骤 1 或之前的聚合步骤产生的 `GlobalUserTreeAggregatorHeader` 进行聚合。这形成了一个证明的二叉树。
- **电路** (由 Realm 编排，由证明矿工执行):
  - `GUTAVerifyTwoGUTACircuit`：接收两个来自较低层级的 GUTA 证明（及其头部）并进行聚合。
  - `GUTAVerifyLeftGUTARightEndCapCircuit` / `GUTAVerifyLeftEndCapRightGUTACircuit`：将一个已有的 GUTA 证明与一个新处理的 End Cap 证明（来自步骤 1）进行聚合。
  - `GUTAVerifyGUTAToCapCircuit`：如果一个子树只有一个活动分支，此电路使用 "Line Proof（线性路径证明）"（内部为 `GUTAHeaderLineProofGadget`）来高效地将一个 GUTA 证明向上传播到 Realm 的 `GUSR` 分段根，而无需进行 NCA 合并。
  - `GUTANoChangeCircuit`：如果 Realm 分段内的整个子树没有任何用户活动，此电路会生成一个“无变化”的 GUTA 头部。它验证一个检查点证明以获取当前的 `checkpoint_tree_root` 和该检查点的 `GUSR` 根，然后断言其分段的 `GUSR` 根保持不变，并输出一个状态转换为无操作（no-op）但更新了 `checkpoint_tree_root` 的 GUTA 头部。这确保了聚合树的所有部分都同步到同一个全局检查点。
- **输入/见证数据** (适用于像 `GUTAVerifyTwoGUTACircuit` 这样的聚合电路的通用情况):
  - 两个输入的 GUTA 证明（或一个 GUTA 和一个 End Cap 证明）及其各自的验证者数据。
  - 与每个输入证明相对应的 `GlobalUserTreeAggregatorHeaderGadget` 见证数据。
  - 每个输入 GUTA 证明的 `guta_whitelist_merkle_proof`，显示其生成电路在 GUTA 白名单中。
  - 如果合并两个分支，则需要一个 `UpdateNearestCommonAncestorProof` 见证数据。
  - 如果使用 `GUTAVerifyGUTAToCapCircuit`，则需要用于 Line Proof（线性路径证明） 的兄弟哈希。
- **它证明了什么** (`GUTAVerifyTwoGUTACircuit` 的通用情况):
  1. **输入证明的有效性**：两个输入证明（类型 A 和类型 B）都是有效的 ZK 证明。
  2. **白名单遵从性**:
     - 如果一个输入是 GUTA 证明，其生成电路在 `guta_circuit_whitelist_root_hash` 中（通过 `VerifyGUTAProofGadget` 验证，该 Gadget 会检查白名单证明）。
     - 如果一个输入是 End Cap 证明，其生成电路具有 `known_end_cap_fingerprint_hash`（通过 `VerifyEndCapProofGadget` 验证）。
  3. **上下文一致性**:
     - 两个输入证明的头部引用**完全相同**的 `checkpoint_tree_root`。
     - 两个输入证明的头部引用**完全相同**的 `guta_circuit_whitelist_root_hash`。
  4. **公共输入匹配**：每个输入证明的公共输入与其声称的 GUTA/EndCap 头部正确匹配。
  5. **NCA 状态转换**：`TwoNCAStateTransitionGadget` 正确地验证 `UpdateNearestCommonAncestorProof` 见证数据，将两个输入头部的 `state_transition` 合并成一个新的 `state_transition`，用于它们在 `GUSR` 中的父 NCA 节点。
  6. **统计数据求和**：`GUTAStatsGadget.combine_with` 正确地将两个输入头部的 `stats` 求和。
- **输出**：一个新的、更高层级的 GUTA ZK 证明及其 `GlobalUserTreeAggregatorHeaderGadget`。
- **意义**：这个递归过程有效地扩展了验证能力。网络无需在更高层级上单独验证所有 End Cap，只需验证聚合后的 GUTA 证明，这些证明代表了 `GUSR` 中越来越大的分段。

这个递归聚合过程（步骤 1 和 2）会一直持续，直到 Realm 为其分配的 `GUSR` 分段的**整个根**，生成一个代表当前区块净状态转换的单一 GUTA 证明和头部。



## 阶段二：Coordinator 级别的 GUTA - 聚合 Realms



Coordinator 接收所有活跃 Realm 的最终 GUTA 证明，并将它们进一步聚合，以获得整个全局 `GUSR` 树的证明。

**步骤 3：聚合 Realm GUTA 证明**

- **动作**：一个 Coordinator 节点从多个 Realm 接收最终的 GUTA 证明（及其头部）。
- **电路** (由 Coordinator 编排，由证明矿工执行):
  - 主要是 `GUTAVerifyTwoGUTACircuit` (用于合并来自两个不同 Realm 或两组已聚合 Realm 的 GUTA 证明)。
  - `GUTAVerifyGUTAToCapCircuit` (如果它是更大分支中唯一的活动，用于将单个 Realm 的 GUTA 证明或一个已聚合的 GUTA 证明向上传播到全局 `GUSR` 根)。
  - `GUTANoChangeCircuit` 可能会被使用，如果跨越多个 Realm 的 `GUSR` 的一个很大部分没有活动。
- **过程**：这在结构上与 Realm 内的递归聚合（上述步骤 2）相同，但这个聚合层级的“叶子”是每个 Realm 的根 GUTA 证明。NCA 逻辑现在应用于 `GUSR` 树中更高的节点，可能会合并来自不同 Realm 分段的转换。
- **输出**：一个单一的 GUTA ZK 证明及其 `GlobalUserTreeAggregatorHeaderGadget`，它代表了当前区块**整个全局用户树 (`GUSR`) 的根**的净状态转换。这个头部将包含区块中所有用户活动的总聚合 `GUTAStats`。



## 与其它全局状态变更的集成



GUTA 流程专门处理对 `GUSR` 的更新。其他全局树，如 `GCON` (全局合约树，用于新合约部署) 和 `URT` (用户注册树，用于新用户注册)，由 Coordinator 编排的、独立的、专用的批处理追加电路来更新：

- **用户注册**：`BatchAppendUserRegistrationTreeCircuit` 为向 `URT` 追加新用户注册生成一个证明。
- **合约部署**：`BatchDeployContractsCircuit` 为向 `GCON` 追加新合约定义生成一个证明。

这些证明，连同用于 `GUSR` 变更的最终 GUTA 证明，随后被输入到 "Part 1" 聚合电路中。

**`VerifyAggUserRegistartionDeployContractsGUTACircuit` (Part 1 聚合)**

- **动作**：Coordinator 编排这个关键的聚合步骤。
- **电路** (由证明矿工执行)：`VerifyAggUserRegistartionDeployContractsGUTACircuit`。
- **输入**:
  - 最终聚合的 GUTA 证明 (用于 `GUSR` 变更)。
  - 聚合的用户注册证明 (用于 `URT` 变更)。
  - 聚合的合约部署证明 (用于 `GCON` 变更)。
- **它证明了什么**:
  - 所有三个输入证明都是有效的，并且是由它们各自的白名单电路生成的。
  - 至关重要的是，所有三个输入证明（以及它们所代表的操作）都基于**同一个一致的 `checkpoint_tree_root` 上下文**（即，它们都从同一个前一区块的最终化状态开始）。
- **输出**：一个 "Part 1" ZK 证明和一个 `VerifyAggUserRegistartionDeployContractsGUTAHeader`。这个头部总结了 `GUSR`、`URT` 和 `GCON` 的新根哈希，并向前传递了总的 `GUTAStats`。

这个 "Part 1" 证明随后成为 `PsyCheckpointStateTransitionCircuit` 的主要输入之一，该电路生成最终的区块证明，具体细节在“端到端区块生产之旅”文档中详述。



## GUTA 的安全性与效率



- **安全性**:
  - **ZK 证明的可靠性 (Soundness)**：防止无效状态转换的数学保证。
  - **电路白名单**：只有授权的 GUTA 和 End Cap 电路才能参与聚合，通过对 `guta_circuit_whitelist_root_hash` 的 Merkle 证明和对 `known_end_cap_fingerprint_hash` 的检查来强制执行。
  - **递归验证**：每一层都以密码学方式验证其下一层。
  - **检查点锚定**：所有 GUTA 操作都与一个特定的 `checkpoint_tree_root` 绑定，确保与全局状态的一致性。
- **效率**:
  - **并行性**：End Cap 证明可以在不同的 Realm 之间，甚至在 Realm 的子树内部并行处理和聚合。证明矿工可以同时处理不同的聚合任务。
  - **对数级扩展**：树状的聚合结构意味着顺序聚合步骤的数量随用户/证明数量的对数增长，这有助于 Psy 实现 `O(log(n))` 的区块时间。
  - **证明压缩**：数百万的用户交易最终由几个聚合证明代表，并进入最终的区块证明。

GUTA 是一项复杂的 ZKP 工程壮举，它使 Psy 能够安全地将用户状态管理扩展到区块链系统中前所未有的水平，构成了其高吞吐能力的支柱。