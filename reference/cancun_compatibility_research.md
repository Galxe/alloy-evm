# Reth Ethereum Hardfork 兼容性代码调研文档

本文档详细列出了 Reth 项目中与 Cancun 及之前所有 hardfork 相关的兼容性代码。每个 hardfork 章节包括代码位置、代码片段以及对应的 EIP 引用说明。

## Homestead Hardfork

### Transaction Signature Recovery 方式

[crates/rpc/rpc/src/debug.rs:163-182](file:///home/neko/reth/crates/rpc/rpc/src/debug.rs#L163-L182)

```rust
// Depending on EIP-2 we need to recover the transactions differently
let senders =
    if self.provider().chain_spec().is_homestead_active_at_block(block.header().number()) {
        block
            .body()
            .transactions()
            .iter()
            .map(|tx| tx.recover_signer().map_err(Eth::Error::from_eth_err))
            .collect::<Result<Vec<_>, _>>()?
            .into_iter()
            .collect()
    } else {
        block
            .body()
            .transactions()
            .iter()
            .map(|tx| tx.recover_signer_unchecked().map_err(Eth::Error::from_eth_err))
            .collect::<Result<Vec<_>, _>>()?
            .into_iter()
            .collect()
    };
```

**兼容性描述**：根据 Homestead hardfork 是否激活，选择不同的 transaction signer recovery 方式。在 Homestead 之前，使用 `recover_signer_unchecked` 不进行严格的 signature 验证；在 Homestead 及之后，使用 `recover_signer` 进行完整的 signature 验证。

**EIP 引用**：EIP-2（Homestead Hard-fork Changes）- 修改了 transaction signature 验证规则，要求 `s` 值必须在特定范围内以防止 transaction malleability 攻击。

## Byzantium Hardfork

### Receipt Root 验证

[crates/ethereum/consensus/src/validation.rs:36-53](file:///home/neko/reth/crates/ethereum/consensus/src/validation.rs#L36-L53)

```rust
// Before Byzantium, receipts contained state root that would mean that expensive
// operation as hashing that is required for state root got calculated in every
// transaction This was replaced with is_success flag.
// See more about EIP here: https://eips.ethereum.org/EIPS/eip-658
if chain_spec.is_byzantium_active_at_block(block.header().number()) &&
    let Err(error) = verify_receipts(
        block.header().receipts_root(),
        block.header().logs_bloom(),
        receipts,
    )
{
    let receipts = receipts
        .iter()
        .map(|r| Bytes::from(r.with_bloom_ref().encoded_2718()))
        .collect::<Vec<_>>();
    tracing::debug!(%error, ?receipts, "receipts verification failed");
    return Err(error)
}
```

**兼容性描述**：在 Byzantium 激活后才验证 receipt root。在 Byzantium 之前，receipt 中包含 state root，这意味着每笔 transaction 都需要进行昂贵的 state root hash 计算。Byzantium 引入了 EIP-658，用简单的 success/failure boolean flag 替代了 state root。

**EIP 引用**：EIP-658（Embedding Transaction Status Code in Receipts）- 在 receipt 中嵌入 transaction status code，用 boolean success flag 替代昂贵的 state root 计算。

## London Hardfork

### Base Fee 验证

[crates/consensus/common/src/validation.rs:40-51](file:///home/neko/reth/crates/consensus/common/src/validation.rs#L40-L51)

```rust
/// Ensure the EIP-1559 base fee is set if the London hardfork is active.
#[inline]
pub fn validate_header_base_fee<H: BlockHeader, ChainSpec: EthereumHardforks>(
    header: &H,
    chain_spec: &ChainSpec,
) -> Result<(), ConsensusError> {
    if chain_spec.is_london_active_at_block(header.number()) && header.base_fee_per_gas().is_none()
    {
        return Err(ConsensusError::BaseFeeMissing)
    }
    Ok(())
}
```

**兼容性描述**：在 London hardfork 激活后，验证 block header 必须包含 `base_fee_per_gas` 字段。如果 London 激活但该字段缺失，返回 `BaseFeeMissing` 错误。

**EIP 引用**：EIP-1559（Fee Market Change for ETH 1.0 Chain）- 引入了新的 transaction pricing 机制，包括 base fee 和 priority fee。

### Base Fee 计算验证

[crates/consensus/common/src/validation.rs:302-331](file:///home/neko/reth/crates/consensus/common/src/validation.rs#L302-L331)

```rust
/// Validates the base fee against the parent and EIP-1559 rules.
#[inline]
pub fn validate_against_parent_eip1559_base_fee<ChainSpec: EthChainSpec + EthereumHardforks>(
    header: &ChainSpec::Header,
    parent: &ChainSpec::Header,
    chain_spec: &ChainSpec,
) -> Result<(), ConsensusError> {
    if chain_spec.is_london_active_at_block(header.number()) {
        let base_fee = header.base_fee_per_gas().ok_or(ConsensusError::BaseFeeMissing)?;

        let expected_base_fee = if chain_spec
            .ethereum_fork_activation(EthereumHardfork::London)
            .transitions_at_block(header.number())
        {
            alloy_eips::eip1559::INITIAL_BASE_FEE
        } else {
            chain_spec
                .next_block_base_fee(parent, header.timestamp())
                .ok_or(ConsensusError::BaseFeeMissing)?
        };
        if expected_base_fee != base_fee {
            return Err(ConsensusError::BaseFeeDiff(GotExpected {
                expected: expected_base_fee,
                got: base_fee,
            }))
        }
    }

    Ok(())
}
```

**兼容性描述**：验证 block 的 base fee 是否正确计算：
- 如果是 London fork 激活的第一个 block，base fee 应为 initial value（1 gwei）
- 否则根据 parent block 的 gas usage 计算预期 base fee

**EIP 引用**：EIP-1559 - 定义了 base fee 的计算公式：基于 parent block gas usage 与 target 的偏差调整。

### Gas Limit Elasticity Multiplier

[crates/consensus/common/src/validation.rs:348-394](file:///home/neko/reth/crates/consensus/common/src/validation.rs#L348-L394)

```rust
/// Validates gas limit against parent gas limit.
///
/// The maximum allowable difference between self and parent gas limits is determined by the
/// parent's gas limit divided by the [`GAS_LIMIT_BOUND_DIVISOR`].
#[inline]
pub fn validate_against_parent_gas_limit<
    H: BlockHeader,
    ChainSpec: EthChainSpec + EthereumHardforks,
>(
    header: &SealedHeader<H>,
    parent: &SealedHeader<H>,
    chain_spec: &ChainSpec,
) -> Result<(), ConsensusError> {
    // Determine the parent gas limit, considering elasticity multiplier on the London fork.
    let parent_gas_limit = if !chain_spec.is_london_active_at_block(parent.number()) &&
        chain_spec.is_london_active_at_block(header.number())
    {
        parent.gas_limit() *
            chain_spec.base_fee_params_at_timestamp(header.timestamp()).elasticity_multiplier
                as u64
    } else {
        parent.gas_limit()
    };
    // ...
}
```

**兼容性描述**：处理 London fork boundary 的 gas limit 验证。当 parent block 是 pre-London 而当前 block 是 London 时，parent block 的 gas limit 需要乘以 elasticity multiplier（通常为 2），因为 London 引入的 EIP-1559 将 block capacity 翻倍。

**EIP 引用**：EIP-1559 - 将 block gas limit 翻倍，引入 elasticity multiplier 概念。

### London Fork Boundary 处理

[crates/ethereum/evm/src/lib.rs:236-250](file:///home/neko/reth/crates/ethereum/evm/src/lib.rs#L236-L250)

```rust
// If we are on the London fork boundary, we need to multiply the parent's gas limit by the
// elasticity multiplier to get the new gas limit.
if self.chain_spec().fork(EthereumHardfork::London).transitions_at_block(parent.number + 1)
{
    let elasticity_multiplier = self
        .chain_spec()
        .base_fee_params_at_timestamp(attributes.timestamp)
        .elasticity_multiplier;

    // multiply the gas limit by the elasticity multiplier
    gas_limit *= elasticity_multiplier as u64;

    // set the base fee to the initial base fee from the EIP-1559 spec
    basefee = Some(INITIAL_BASE_FEE)
}
```

**兼容性描述**：在 EVM 环境配置中处理 London fork boundary。当构建 London 激活后的第一个 block 时：
1. 将 gas limit 乘以 elasticity multiplier
2. 将 base fee 设置为 EIP-1559 规定的 initial value

**EIP 引用**：EIP-1559 - 定义了 initial base fee（1 gwei）和 gas elasticity 机制。

## Paris Hardfork（The Merge）

### Post-Merge Block Header 验证

[crates/ethereum/consensus/src/lib.rs:94-127](file:///home/neko/reth/crates/ethereum/consensus/src/lib.rs#L94-L127)

```rust
fn validate_header(&self, header: &SealedHeader<H>) -> Result<(), ConsensusError> {
    let header = header.header();
    let is_post_merge = self.chain_spec.is_paris_active_at_block(header.number());

    if is_post_merge {
        if !header.difficulty().is_zero() {
            return Err(ConsensusError::TheMergeDifficultyIsNotZero);
        }

        if !header.nonce().is_some_and(|nonce| nonce.is_zero()) {
            return Err(ConsensusError::TheMergeNonceIsNotZero);
        }

        if header.ommers_hash() != EMPTY_OMMER_ROOT_HASH {
            return Err(ConsensusError::TheMergeOmmerRootIsNotEmpty);
        }
    } else {
        #[cfg(feature = "std")]
        {
            let present_timestamp = std::time::SystemTime::now()
                .duration_since(std::time::SystemTime::UNIX_EPOCH)
                .unwrap()
                .as_secs();

            if header.timestamp() >
                present_timestamp + alloy_eips::merge::ALLOWED_FUTURE_BLOCK_TIME_SECONDS
            {
                return Err(ConsensusError::TimestampIsInFuture {
                    timestamp: header.timestamp(),
                    present_timestamp,
                });
            }
        }
    }
    // ...
}
```

**兼容性描述**：根据 Paris（The Merge）hardfork 状态进行不同的验证：
- **Post-merge**：
  - [difficulty](file:///home/neko/reth/crates/chainspec/src/api.rs#135-138) 必须为 0
  - [nonce](file:///home/neko/reth/crates/transaction-pool/src/validate/eth.rs#598-615) 必须为 0
  - `ommers_hash` 必须为 `EMPTY_OMMER_ROOT_HASH`
- **Pre-merge**：
  - 验证 timestamp 不能太超前于当前时间

**EIP 引用**：
- EIP-3675（Upgrade Consensus to Proof-of-Stake）- 定义了 PoS 转换后 block header 字段的变化
- EIP-4399（Supplant DIFFICULTY opcode with PREVRANDAO）- 用 `prevrandao` 替代 [difficulty](file:///home/neko/reth/crates/chainspec/src/api.rs#135-138)

### Ommers 读取逻辑

[crates/storage/storage-api/src/chain.rs:178-188](file:///home/neko/reth/crates/storage/storage-api/src/chain.rs#L178-L188)

```rust
let ommers = if chain_spec.is_paris_active_at_block(header.number()) {
    Vec::new()
} else {
    // Pre-merge: fetch ommers from database using direct database access
    provider
        .tx_ref()
        .cursor_read::<tables::BlockOmmers<H>>()?
        .seek_exact(header.number())?
        .map(|(_, stored_ommers)| stored_ommers.ommers)
        .unwrap_or_default()
};
```

**兼容性描述**：读取 block body 时根据 Paris 状态决定是否读取 ommers：
- Post-merge：ommers 始终为空 vector
- Pre-merge：从 database 读取 ommers 数据

**EIP 引用**：EIP-3675 - Merge 后不再有 uncle blocks（ommers）。

## Shanghai Hardfork

### Withdrawals 验证

[crates/consensus/common/src/validation.rs:53-72](file:///home/neko/reth/crates/consensus/common/src/validation.rs#L53-L72)

```rust
/// Validate that withdrawals are present in Shanghai
///
/// See [EIP-4895]: Beacon chain push withdrawals as operations
///
/// [EIP-4895]: https://eips.ethereum.org/EIPS/eip-4895
#[inline]
pub fn validate_shanghai_withdrawals<B: Block>(
    block: &SealedBlock<B>,
) -> Result<(), ConsensusError> {
    let withdrawals = block.body().withdrawals().ok_or(ConsensusError::BodyWithdrawalsMissing)?;
    let withdrawals_root = alloy_consensus::proofs::calculate_withdrawals_root(withdrawals);
    let header_withdrawals_root =
        block.withdrawals_root().ok_or(ConsensusError::WithdrawalsRootMissing)?;
    if withdrawals_root != *header_withdrawals_root {
        return Err(ConsensusError::BodyWithdrawalsRootDiff(
            GotExpected { got: withdrawals_root, expected: header_withdrawals_root }.into(),
        ));
    }
    Ok(())
}
```

**兼容性描述**：验证 Shanghai 升级后 block 中的 withdrawals list：
1. 检查 block body 必须包含 [withdrawals](file:///home/neko/reth/crates/consensus/common/src/validation.rs#53-73) 字段
2. 计算 withdrawals 的 Merkle root
3. 验证计算的 root 与 block header 中的 `withdrawals_root` 匹配

**EIP 引用**：EIP-4895（Beacon Chain Push Withdrawals as Operations）- 允许 validators 从 beacon chain 提取 staked ETH。

### Withdrawals 条件验证

[crates/consensus/common/src/validation.rs:204-207](file:///home/neko/reth/crates/consensus/common/src/validation.rs#L204-L207)

```rust
// EIP-4895: Beacon chain push withdrawals as operations
if chain_spec.is_shanghai_active_at_timestamp(block.timestamp()) {
    validate_shanghai_withdrawals(block)?;
}
```

**兼容性描述**：只有在 Shanghai hardfork 激活后才验证 withdrawals 字段。

**EIP 引用**：EIP-4895。

### Withdrawals Root Block Header 验证

[crates/ethereum/consensus/src/lib.rs:132-141](file:///home/neko/reth/crates/ethereum/consensus/src/lib.rs#L132-L141)

```rust
// EIP-4895: Beacon chain push withdrawals as operations
if self.chain_spec.is_shanghai_active_at_timestamp(header.timestamp()) &&
    header.withdrawals_root().is_none()
{
    return Err(ConsensusError::WithdrawalsRootMissing)
} else if !self.chain_spec.is_shanghai_active_at_timestamp(header.timestamp()) &&
    header.withdrawals_root().is_some()
{
    return Err(ConsensusError::WithdrawalsRootUnexpected)
}
```

**兼容性描述**：验证 `withdrawals_root` 字段的存在性：
- Shanghai 激活后：必须存在
- Shanghai 之前：必须不存在

**EIP 引用**：EIP-4895。

### Withdrawals 读取逻辑

[crates/storage/storage-api/src/chain.rs:166-177](file:///home/neko/reth/crates/storage/storage-api/src/chain.rs#L166-L177)

```rust
// If we are past shanghai, then all blocks should have a withdrawal list,
// even if empty
let withdrawals = if chain_spec.is_shanghai_active_at_timestamp(header.timestamp()) {
    withdrawals_cursor
        .seek_exact(header.number())?
        .map(|(_, w)| w.withdrawals)
        .unwrap_or_default()
        .into()
} else {
    None
};
```

**兼容性描述**：从 storage 读取 block body 时，根据 Shanghai 状态决定 withdrawals 字段：
- Shanghai 激活后：返回 withdrawals list（可能为空）
- Shanghai 之前：返回 None

**EIP 引用**：EIP-4895。

### Block 组装时的 Withdrawals 字段

[crates/ethereum/evm/src/build.rs:63-69](file:///home/neko/reth/crates/ethereum/evm/src/build.rs#L63-L69)

```rust
let withdrawals = self
    .chain_spec
    .is_shanghai_active_at_timestamp(timestamp)
    .then(|| ctx.withdrawals.map(|w| w.into_owned()).unwrap_or_default());

let withdrawals_root =
    withdrawals.as_deref().map(|w| proofs::calculate_withdrawals_root(w));
```

**兼容性描述**：在组装新 block 时，仅当 Shanghai 激活时才包含 withdrawals 字段及其 root hash。

**EIP 引用**：EIP-4895。

### Payload 验证中的 Shanghai 字段

[crates/ethereum/payload/src/validator.rs:89-92](file:///home/neko/reth/crates/ethereum/payload/src/validator.rs#L89-L92)

```rust
shanghai::ensure_well_formed_fields(
    sealed_block.body(),
    chain_spec.is_shanghai_active_at_timestamp(sealed_block.timestamp),
)?;
```

**兼容性描述**：在验证 execution payload 时检查 Shanghai 相关字段的格式。

**EIP 引用**：EIP-4895。

## Cancun Hardfork

### Blob Gas 验证

[crates/consensus/common/src/validation.rs:74-92](file:///home/neko/reth/crates/consensus/common/src/validation.rs#L74-L92)

```rust
/// Validate that blob gas is present in the block if Cancun is active.
///
/// See [EIP-4844]: Shard Blob Transactions
///
/// [EIP-4844]: https://eips.ethereum.org/EIPS/eip-4844
#[inline]
pub fn validate_cancun_gas<B: Block>(block: &SealedBlock<B>) -> Result<(), ConsensusError> {
    // Check that the blob gas used in the header matches the sum of the blob gas used by each
    // blob tx
    let header_blob_gas_used = block.blob_gas_used().ok_or(ConsensusError::BlobGasUsedMissing)?;
    let total_blob_gas = block.body().blob_gas_used();
    if total_blob_gas != header_blob_gas_used {
        return Err(ConsensusError::BlobGasUsedDiff(GotExpected {
            got: header_blob_gas_used,
            expected: total_blob_gas,
        }));
    }
    Ok(())
}
```

**兼容性描述**：验证 block header 中的 [blob_gas_used](file:///home/neko/reth/crates/consensus/common/src/validation.rs#470-506) 与 block body 中所有 blob transactions 使用的 blob gas 总和是否匹配。

**EIP 引用**：EIP-4844（Shard Blob Transactions）- 定义了 blob transaction 和 blob gas 机制。

### Cancun Gas 条件验证

[crates/consensus/common/src/validation.rs:209-211](file:///home/neko/reth/crates/consensus/common/src/validation.rs#L209-L211)

```rust
if chain_spec.is_cancun_active_at_timestamp(block.timestamp()) {
    validate_cancun_gas(block)?;
}
```

**兼容性描述**：只有在 Cancun hardfork 激活后才验证 blob gas 字段。

**EIP 引用**：EIP-4844。

### EIP-4844 Block Header 字段验证

[crates/ethereum/consensus/src/lib.rs:143-157](file:///home/neko/reth/crates/ethereum/consensus/src/lib.rs#L143-L157)

```rust
// Ensures that EIP-4844 fields are valid once cancun is active.
if self.chain_spec.is_cancun_active_at_timestamp(header.timestamp()) {
    validate_4844_header_standalone(
        header,
        self.chain_spec
            .blob_params_at_timestamp(header.timestamp())
            .unwrap_or_else(BlobParams::cancun),
    )?;
} else if header.blob_gas_used().is_some() {
    return Err(ConsensusError::BlobGasUsedUnexpected)
} else if header.excess_blob_gas().is_some() {
    return Err(ConsensusError::ExcessBlobGasUnexpected)
} else if header.parent_beacon_block_root().is_some() {
    return Err(ConsensusError::ParentBeaconBlockRootUnexpected)
}
```

**兼容性描述**：验证 block header 的 Cancun 特定字段：
- Cancun 激活后：验证 [blob_gas_used](file:///home/neko/reth/crates/consensus/common/src/validation.rs#470-506)、`excess_blob_gas`、[parent_beacon_block_root](file:///home/neko/reth/crates/payload/primitives/src/lib.rs#200-308) 的有效性
- Cancun 之前：这些字段必须不存在

**EIP 引用**：
- EIP-4844 - [blob_gas_used](file:///home/neko/reth/crates/consensus/common/src/validation.rs#470-506) 和 `excess_blob_gas` 字段
- EIP-4788（Beacon Block Root in the EVM）- [parent_beacon_block_root](file:///home/neko/reth/crates/payload/primitives/src/lib.rs#200-308) 字段

### EIP-4844 Standalone Block Header 验证

[crates/consensus/common/src/validation.rs:225-260](file:///home/neko/reth/crates/consensus/common/src/validation.rs#L225-L260)

```rust
/// Validates that the EIP-4844 header fields exist and conform to the spec. This ensures that:
///
///  * `blob_gas_used` exists as a header field
///  * `excess_blob_gas` exists as a header field
///  * `parent_beacon_block_root` exists as a header field
///  * `blob_gas_used` is a multiple of `DATA_GAS_PER_BLOB`
///  * `excess_blob_gas` is a multiple of `DATA_GAS_PER_BLOB`
///  * `blob_gas_used` doesn't exceed the max allowed blob gas based on the given params
///
/// Note: This does not enforce any restrictions on `blob_gas_used`
pub fn validate_4844_header_standalone<H: BlockHeader>(
    header: &H,
    blob_params: BlobParams,
) -> Result<(), ConsensusError> {
    let blob_gas_used = header.blob_gas_used().ok_or(ConsensusError::BlobGasUsedMissing)?;

    if header.parent_beacon_block_root().is_none() {
        return Err(ConsensusError::ParentBeaconBlockRootMissing)
    }

    if !blob_gas_used.is_multiple_of(DATA_GAS_PER_BLOB) {
        return Err(ConsensusError::BlobGasUsedNotMultipleOfBlobGasPerBlob {
            blob_gas_used,
            blob_gas_per_blob: DATA_GAS_PER_BLOB,
        })
    }

    if blob_gas_used > blob_params.max_blob_gas_per_block() {
        return Err(ConsensusError::BlobGasUsedExceedsMaxBlobGasPerBlock {
            blob_gas_used,
            max_blob_gas_per_block: blob_params.max_blob_gas_per_block(),
        })
    }

    Ok(())
}
```

**兼容性描述**：验证 EIP-4844 block header 字段的有效性：
1. [blob_gas_used](file:///home/neko/reth/crates/consensus/common/src/validation.rs#470-506) 必须存在且是 `DATA_GAS_PER_BLOB`（131072）的倍数
2. [parent_beacon_block_root](file:///home/neko/reth/crates/payload/primitives/src/lib.rs#200-308) 必须存在
3. [blob_gas_used](file:///home/neko/reth/crates/consensus/common/src/validation.rs#470-506) 不能超过每 block 最大 blob gas 限制

**EIP 引用**：
- EIP-4844 - Blob gas 相关约束
- EIP-4788 - Parent beacon block root 要求

### Blob Gas 字段组装

[crates/ethereum/evm/src/build.rs:78-94](file:///home/neko/reth/crates/ethereum/evm/src/build.rs#L78-L94)

```rust
let mut excess_blob_gas = None;
let mut blob_gas_used = None;

// only determine cancun fields when active
if self.chain_spec.is_cancun_active_at_timestamp(timestamp) {
    blob_gas_used =
        Some(transactions.iter().map(|tx| tx.blob_gas_used().unwrap_or_default()).sum());
    excess_blob_gas = if self.chain_spec.is_cancun_active_at_timestamp(parent.timestamp) {
        parent.maybe_next_block_excess_blob_gas(
            self.chain_spec.blob_params_at_timestamp(timestamp),
        )
    } else {
        // for the first post-fork block, both parent.blob_gas_used and
        // parent.excess_blob_gas are evaluated as 0
        Some(
            alloy_eips::eip7840::BlobParams::cancun()
                .next_block_excess_blob_gas_osaka(0, 0, 0),
        )
    };
}
```

**兼容性描述**：组装新 block 时，仅当 Cancun 激活时才计算 blob gas 字段：
- [blob_gas_used](file:///home/neko/reth/crates/consensus/common/src/validation.rs#470-506)：所有 transactions 的 blob gas 总和
- `excess_blob_gas`：基于 parent block 计算，如果 parent block 是 pre-Cancun 则使用 0 作为初始值

**EIP 引用**：EIP-4844 - 定义了 `excess_blob_gas` 的计算规则。

### EVM 环境配置中的 Cancun Boundary 处理

[crates/ethereum/evm/src/lib.rs:221-230](file:///home/neko/reth/crates/ethereum/evm/src/lib.rs#L221-L230)

```rust
// if the parent block did not have excess blob gas (i.e. it was pre-cancun), but it is
// cancun now, we need to set the excess blob gas to the default value(0)
let blob_excess_gas_and_price = parent
    .maybe_next_block_excess_blob_gas(blob_params)
    .or_else(|| (spec_id == SpecId::CANCUN).then_some(0))
    .map(|excess_blob_gas| {
        let blob_gasprice =
            blob_params.unwrap_or_else(BlobParams::cancun).calc_blob_fee(excess_blob_gas);
        BlobExcessGasAndPrice { excess_blob_gas, blob_gasprice }
    });
```

**兼容性描述**：配置下一个 block EVM 环境时处理 Cancun fork boundary。如果 parent block 没有 `excess_blob_gas`（pre-Cancun），但当前 block 是 Cancun，将 `excess_blob_gas` 设置为 0。

**EIP 引用**：EIP-4844 - 规定 fork 激活时 `excess_blob_gas` 的初始值为 0。

### Cancun Payload 字段验证

[crates/ethereum/payload/src/validator.rs:94-98](file:///home/neko/reth/crates/ethereum/payload/src/validator.rs#L94-L98)

```rust
cancun::ensure_well_formed_fields(
    &sealed_block,
    sidecar.cancun(),
    chain_spec.is_cancun_active_at_timestamp(sealed_block.timestamp),
)?;
```

**兼容性描述**：验证 execution payload 的 Cancun 相关字段格式。

**EIP 引用**：EIP-4844。

### Cancun Block Header 和 Sidecar 字段格式验证

[crates/payload/validator/src/cancun.rs:34-68](file:///home/neko/reth/crates/payload/validator/src/cancun.rs#L34-L68)

```rust
/// Checks that Cancun fields on block header and sidecar are present if Cancun is active and vv.
#[inline]
pub fn ensure_well_formed_header_and_sidecar_fields<T: Block>(
    block: &SealedBlock<T>,
    cancun_sidecar_fields: Option<&CancunPayloadFields>,
    is_cancun_active: bool,
) -> Result<(), PayloadError> {
    if is_cancun_active {
        if block.blob_gas_used().is_none() {
            // cancun active but blob gas used not present
            return Err(PayloadError::PostCancunBlockWithoutBlobGasUsed)
        }
        if block.excess_blob_gas().is_none() {
            // cancun active but excess blob gas not present
            return Err(PayloadError::PostCancunBlockWithoutExcessBlobGas)
        }
        if cancun_sidecar_fields.is_none() {
            // cancun active but cancun fields not present
            return Err(PayloadError::PostCancunWithoutCancunFields)
        }
    } else {
        if block.blob_gas_used().is_some() {
            // cancun not active but blob gas used present
            return Err(PayloadError::PreCancunBlockWithBlobGasUsed)
        }
        if block.excess_blob_gas().is_some() {
            // cancun not active but excess blob gas present
            return Err(PayloadError::PreCancunBlockWithExcessBlobGas)
        }
        if cancun_sidecar_fields.is_some() {
            // cancun not active but cancun fields present
            return Err(PayloadError::PreCancunWithCancunFields)
        }
    }
    Ok(())
}
```

**兼容性描述**：验证 Cancun 字段的存在性：
- Cancun 激活后：[blob_gas_used](file:///home/neko/reth/crates/consensus/common/src/validation.rs#470-506)、`excess_blob_gas` 和 sidecar 字段必须存在
- Cancun 之前：这些字段必须不存在

**EIP 引用**：EIP-4844。

### Blob Transaction Pre-Cancun 验证

[crates/payload/validator/src/cancun.rs:76-88](file:///home/neko/reth/crates/payload/validator/src/cancun.rs#L76-L88)

```rust
/// Checks transactions field and sidecar w.r.t new Cancun fields and new transaction type EIP-4844.
///
/// Checks that:
/// - doesn't contain EIP-4844 transactions unless Cancun is active
/// - checks blob versioned hashes in block and sidecar match
#[inline]
pub fn ensure_well_formed_transactions_field_with_sidecar<T: Transaction + Typed2718, H>(
    block_body: &BlockBody<T, H>,
    cancun_sidecar_fields: Option<&CancunPayloadFields>,
    is_cancun_active: bool,
) -> Result<(), PayloadError> {
    if is_cancun_active {
        ensure_matching_blob_versioned_hashes(block_body, cancun_sidecar_fields)?
    } else if block_body.has_eip4844_transactions() {
        return Err(PayloadError::PreCancunBlockWithBlobTransactions)
    }
    Ok(())
}
```

**兼容性描述**：验证 transactions 与 Cancun 状态的兼容性：
- Cancun 激活后：验证 blob versioned hashes 匹配
- Cancun 之前：不允许包含 EIP-4844 blob transactions

**EIP 引用**：EIP-4844 - Blob transaction type（type 0x03）仅在 Cancun 后支持。

### RPC Config 中的 System Contracts

[crates/rpc/rpc-eth-api/src/helpers/config.rs:54-65](file:///home/neko/reth/crates/rpc/rpc-eth-api/src/helpers/config.rs#L54-L65)

```rust
fn build_fork_config_at(
    &self,
    timestamp: u64,
    precompiles: BTreeMap<String, Address>,
) -> Option<EthForkConfig> {
    let chain_spec = self.provider.chain_spec();

    let mut system_contracts = BTreeMap::<SystemContract, Address>::default();

    if chain_spec.is_cancun_active_at_timestamp(timestamp) {
        system_contracts.extend(SystemContract::cancun());
    }
    // ...
}
```

**兼容性描述**：构建 EIP-7910 fork config 时，如果 Cancun 激活则添加 Cancun 引入的 system contracts（如 Beacon Roots Contract：0x000F3df6D732807Ef1319fB7B8bB8522d0Beac02）。

**EIP 引用**：
- EIP-7910 - eth_config RPC endpoint
- EIP-4788 - Beacon Roots Contract

## 总结

本文档涵盖了从 Homestead 到 Cancun 的所有主要 hardfork 兼容性代码：

| Hardfork | 主要 EIP | 关键兼容性代码 |
|----------|----------|----------------|
| Homestead | EIP-2 | Transaction signature recovery 方式 |
| Byzantium | EIP-658 | Receipt root 验证 |
| London | EIP-1559 | Base fee 验证、Gas limit elasticity multiplier |
| Paris | EIP-3675, EIP-4399 | Difficulty/Nonce/Ommers 验证 |
| Shanghai | EIP-4895 | Withdrawals 验证 |
| Cancun | EIP-4844, EIP-4788 | Blob gas、Parent beacon block root 验证 |

所有兼容性代码遵循以下原则：
1. **向前兼容**：在 fork 未激活时拒绝包含新字段的 block
2. **向后兼容**：在 fork 激活后要求包含必需的新字段
3. **Fork boundary 处理**：正确处理 fork 激活时刻的第一个 block
