# Alloy-Monad-EVM Technical Specification

Implementation guide for `alloy-monad-evm` - the alloy wrapper crate that enables Foundry integration for Monad.

---

## Overview

Foundry uses `alloy-evm` crate (not reth-evm). The pattern is:

```
┌─────────────────────────────────────────────────────────────┐
│  Foundry (anvil, forge)                                     │
│    └── EitherEvm                                            │
│          ├── Eth(alloy_evm::EthEvm)                        │
│          ├── Op(alloy_op_evm::OpEvm)                       │
│          └── Monad(alloy_monad_evm::MonadEvm)  ← NEW       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  alloy-monad-evm (THIS CRATE) ✅ COMPLETE                   │
│    - MonadEvm (implements alloy_evm::Evm trait)            │
│    - MonadEvmFactory (implements alloy_evm::EvmFactory)    │
│    - Re-exports: MonadContext, MonadHandler                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  monad-revm (COMPLETE)                                      │
│    - MonadSpecId, MonadHandler, MonadPrecompiles           │
│    - MonadInstructions, monad_gas_params()                 │
│    - MonadEvm (raw REVM wrapper)                           │
│    - MonadContext, MonadBuilder, DefaultMonad              │
└─────────────────────────────────────────────────────────────┘
```

---

## Architecture

### Key Design Principles

1. **Follow op-revm/alloy-op-evm pattern** - The implementation mirrors how Optimism integrates with alloy-evm
2. **Re-use types from monad-revm** - Import `MonadContext`, `MonadBuilder` etc. instead of redefining
3. **Builder pattern** - Use `Context::monad().with_db(db).build_monad_with_inspector(inspector)` pattern
4. **Trait delegation** - Call methods directly on inner: `self.inner.transact(tx)`, `self.inner.inspect_tx(tx)`

### Type Hierarchy

```
alloy_monad_evm::MonadEvm<DB, I, P>
    └── inner: monad_revm::MonadEvm<MonadContext<DB>, I, MonadInstructions<...>, P>
                    └── .0: revm::Evm<CTX, INSP, INST, PRECOMPILES, FRAME>

MonadContext<DB> = Context<BlockEnv, TxEnv, CfgEnv<MonadSpecId>, DB, Journal<DB>, ()>
```

---

## Crate Structure

```
alloy-monad-evm/
├── Cargo.toml
└── src/
    └── lib.rs          # All implementation in single file (like alloy-op-evm)
```

---

## Implementation

### `Cargo.toml`

```toml
[package]
name = "alloy-monad-evm"
description = "Monad EVM implementation"

version.workspace = true
edition.workspace = true
rust-version.workspace = true
authors.workspace = true
license.workspace = true
homepage.workspace = true
repository.workspace = true

[lints]
workspace = true

[dependencies]
alloy-evm.workspace = true
alloy-primitives.workspace = true
revm.workspace = true
monad-revm = { git = "https://github.com/haythemsellami/revm", branch = "monad-revm", package = "monad-revm" }

[dev-dependencies]
test-case.workspace = true

[features]
default = ["std"]
std = [
    "alloy-primitives/std",
    "revm/std",
    "alloy-evm/std",
]
asm-keccak = ["alloy-evm/asm-keccak", "alloy-primitives/asm-keccak", "revm/asm-keccak"]
```

### `src/lib.rs`

```rust
#![cfg_attr(not(test), warn(unused_crate_dependencies))]

//! Alloy EVM implementation for Monad blockchain.
//!
//! This crate provides:
//! - [`MonadEvm`]: Wrapper implementing [`alloy_evm::Evm`] trait
//! - [`MonadEvmFactory`]: Factory implementing [`alloy_evm::EvmFactory`] trait
//! - [`MonadContext`]: Type alias for Monad EVM context (re-exported from monad-revm)

use alloy_evm::{precompiles::PrecompilesMap, Database, Evm, EvmEnv, EvmFactory};
use alloy_primitives::{Address, Bytes};
use monad_revm::{
    instructions::MonadInstructions, precompiles::MonadPrecompiles, DefaultMonad, MonadBuilder,
    MonadEvm as InnerMonadEvm, MonadSpecId,
};
use revm::{
    context::{BlockEnv, TxEnv},
    context_interface::result::{EVMError, HaltReason, ResultAndState},
    handler::PrecompileProvider,
    inspector::NoOpInspector,
    interpreter::InterpreterResult,
    Context, ExecuteEvm, InspectEvm, Inspector, SystemCallEvm,
};
use std::ops::{Deref, DerefMut};

// Re-export monad-revm types for external users
pub use monad_revm::{handler::MonadHandler, MonadContext};

/// Monad EVM implementation.
///
/// This is a wrapper type around the `monad_revm::MonadEvm` with optional [`Inspector`] (tracing)
/// support. [`Inspector`] support is configurable at runtime because it's part of the underlying
/// [`InnerMonadEvm`](monad_revm::MonadEvm) type.
#[allow(missing_debug_implementations)] // MonadEvm doesn't impl Debug
pub struct MonadEvm<DB: Database, I, P = MonadPrecompiles> {
    inner: InnerMonadEvm<MonadContext<DB>, I, MonadInstructions<MonadContext<DB>>, P>,
    inspect: bool,
}

impl<DB: Database, I, P> MonadEvm<DB, I, P> {
    /// Provides a reference to the EVM context.
    pub const fn ctx(&self) -> &MonadContext<DB> {
        &self.inner.0.ctx
    }

    /// Provides a mutable reference to the EVM context.
    pub const fn ctx_mut(&mut self) -> &mut MonadContext<DB> {
        &mut self.inner.0.ctx
    }
}

impl<DB: Database, I, P> MonadEvm<DB, I, P> {
    /// Creates a new Monad EVM instance.
    ///
    /// The `inspect` argument determines whether the configured [`Inspector`] of the given
    /// [`InnerMonadEvm`](monad_revm::MonadEvm) should be invoked on [`Evm::transact`].
    pub const fn new(
        evm: InnerMonadEvm<MonadContext<DB>, I, MonadInstructions<MonadContext<DB>>, P>,
        inspect: bool,
    ) -> Self {
        Self { inner: evm, inspect }
    }
}

impl<DB: Database, I, P> Deref for MonadEvm<DB, I, P> {
    type Target = MonadContext<DB>;

    #[inline]
    fn deref(&self) -> &Self::Target {
        self.ctx()
    }
}

impl<DB: Database, I, P> DerefMut for MonadEvm<DB, I, P> {
    #[inline]
    fn deref_mut(&mut self) -> &mut Self::Target {
        self.ctx_mut()
    }
}

impl<DB, I, P> Evm for MonadEvm<DB, I, P>
where
    DB: Database,
    I: Inspector<MonadContext<DB>>,
    P: PrecompileProvider<MonadContext<DB>, Output = InterpreterResult>,
{
    type DB = DB;
    type Tx = TxEnv;
    type Error = EVMError<DB::Error>;
    type HaltReason = HaltReason;
    type Spec = MonadSpecId;
    type BlockEnv = BlockEnv;
    type Precompiles = P;
    type Inspector = I;

    fn block(&self) -> &BlockEnv {
        &self.block
    }

    fn chain_id(&self) -> u64 {
        self.cfg.chain_id
    }

    fn transact_raw(
        &mut self,
        tx: Self::Tx,
    ) -> Result<ResultAndState<Self::HaltReason>, Self::Error> {
        if self.inspect {
            self.inner.inspect_tx(tx)
        } else {
            self.inner.transact(tx)
        }
    }

    fn transact_system_call(
        &mut self,
        caller: Address,
        contract: Address,
        data: Bytes,
    ) -> Result<ResultAndState<Self::HaltReason>, Self::Error> {
        self.inner.system_call_with_caller(caller, contract, data)
    }

    fn finish(self) -> (Self::DB, EvmEnv<Self::Spec>) {
        let Context { block: block_env, cfg: cfg_env, journaled_state, .. } = self.inner.0.ctx;

        (journaled_state.database, EvmEnv { block_env, cfg_env })
    }

    fn set_inspector_enabled(&mut self, enabled: bool) {
        self.inspect = enabled;
    }

    fn components(&self) -> (&Self::DB, &Self::Inspector, &Self::Precompiles) {
        (
            &self.inner.0.ctx.journaled_state.database,
            &self.inner.0.inspector,
            &self.inner.0.precompiles,
        )
    }

    fn components_mut(&mut self) -> (&mut Self::DB, &mut Self::Inspector, &mut Self::Precompiles) {
        (
            &mut self.inner.0.ctx.journaled_state.database,
            &mut self.inner.0.inspector,
            &mut self.inner.0.precompiles,
        )
    }
}

/// Factory for creating [`MonadEvm`] instances.
///
/// Implements [`alloy_evm::EvmFactory`] for integration with Foundry.
#[derive(Debug, Default, Clone, Copy)]
#[non_exhaustive]
pub struct MonadEvmFactory;

impl EvmFactory for MonadEvmFactory {
    type Evm<DB: Database, I: Inspector<MonadContext<DB>>> = MonadEvm<DB, I, Self::Precompiles>;
    type Context<DB: Database> = MonadContext<DB>;
    type Tx = TxEnv;
    type Error<DBError: core::error::Error + Send + Sync + 'static> = EVMError<DBError>;
    type HaltReason = HaltReason;
    type Spec = MonadSpecId;
    type BlockEnv = BlockEnv;
    type Precompiles = PrecompilesMap;

    fn create_evm<DB: Database>(
        &self,
        db: DB,
        input: EvmEnv<MonadSpecId>,
    ) -> Self::Evm<DB, NoOpInspector> {
        let spec_id = input.cfg_env.spec;
        MonadEvm {
            inner: Context::monad()
                .with_db(db)
                .with_block(input.block_env)
                .with_cfg(input.cfg_env)
                .build_monad_with_inspector(NoOpInspector {})
                .with_precompiles(PrecompilesMap::from_static(
                    MonadPrecompiles::new_with_spec(spec_id).precompiles(),
                )),
            inspect: false,
        }
    }

    fn create_evm_with_inspector<DB: Database, I: Inspector<Self::Context<DB>>>(
        &self,
        db: DB,
        input: EvmEnv<MonadSpecId>,
        inspector: I,
    ) -> Self::Evm<DB, I> {
        let spec_id = input.cfg_env.spec;
        MonadEvm {
            inner: Context::monad()
                .with_db(db)
                .with_block(input.block_env)
                .with_cfg(input.cfg_env)
                .build_monad_with_inspector(inspector)
                .with_precompiles(PrecompilesMap::from_static(
                    MonadPrecompiles::new_with_spec(spec_id).precompiles(),
                )),
            inspect: true,
        }
    }
}
```

---

## Key Concepts for Maintainers

### monad-revm Exports Used

The following types are imported from `monad-revm`:

| Type | Purpose |
|------|---------|
| `MonadContext<DB>` | Context type alias: `Context<BlockEnv, TxEnv, CfgEnv<MonadSpecId>, DB, Journal<DB>, ()>` |
| `MonadBuilder` | Trait providing `.build_monad()` and `.build_monad_with_inspector()` on `Context` |
| `DefaultMonad` | Trait providing `Context::monad()` constructor |
| `MonadEvm` | Inner newtype wrapper: `MonadEvm(pub Evm<...>)` |
| `MonadInstructions<CTX>` | Type alias for `EthInstructions<EthInterpreter, CTX>` with custom gas params |
| `MonadPrecompiles` | Precompile set for Monad |
| `MonadSpecId` | Hardfork specification enum |
| `MonadHandler` | Handler with gas-limit charging (no refunds) |

### How Transaction Execution Works

1. `MonadEvmFactory::create_evm()` builds EVM using builder pattern:
   ```rust
   Context::monad()           // Creates default MonadContext
       .with_db(db)           // Sets database
       .with_block(block)     // Sets block environment
       .with_cfg(cfg)         // Sets config with MonadSpecId
       .build_monad_with_inspector(inspector)  // Wraps in monad_revm::MonadEvm
       .with_precompiles(precompiles)          // Adds precompile map
   ```

2. `MonadEvm::transact_raw()` delegates to inner:
   ```rust
   if self.inspect {
       self.inner.inspect_tx(tx)  // Uses InspectEvm trait
   } else {
       self.inner.transact(tx)    // Uses ExecuteEvm trait
   }
   ```

3. These traits (`ExecuteEvm`, `InspectEvm`, `SystemCallEvm`) are implemented in `monad_revm::api::exec` and use `MonadHandler` internally for Monad-specific gas handling.

### Monad-Specific Behavior

| Aspect | Ethereum | Monad |
|--------|----------|-------|
| Gas charging | Based on `gas_used` | Based on `gas_limit` |
| Gas refunds | Yes | No (disabled in `MonadHandler`) |
| Reimburse caller | Yes | No (disabled in `MonadHandler`) |
| Cold SLOAD cost | 2100 | 8100 |
| Cold account access | 2600 | 10100 |

---

## Foundry Integration

After creating `alloy-monad-evm`, update Foundry:

### 1. Add Dependency

```toml
# foundry/Cargo.toml
[workspace.dependencies]
alloy-monad-evm = { git = "https://github.com/...", branch = "..." }
```

### 2. Update `EitherEvm`

```rust
// foundry/crates/evm/core/src/either_evm.rs
use alloy_monad_evm::{MonadEvm, MonadContext};
use monad_revm::MonadSpecId;

pub enum EitherEvm<DB, I, P>
where
    DB: Database,
{
    Eth(EthEvm<DB, I, P>),
    Op(OpEvm<DB, I, P>),
    Monad(MonadEvm<DB, I, P>),  // ADD
}

// Add Monad case to ALL trait method implementations
impl<DB, I, P> Evm for EitherEvm<DB, I, P> { ... }
```

### 3. Update `NetworkConfigs`

```rust
// foundry/crates/evm/networks/src/lib.rs

#[derive(Clone, Debug, Default, Parser, Copy, Serialize, Deserialize, PartialEq)]
pub struct NetworkConfigs {
    #[arg(help_heading = "Networks", long, conflicts_with_all = ["optimism", "celo"])]
    #[serde(default)]
    monad: bool,  // ADD
    // ...
}

impl NetworkConfigs {
    pub fn with_monad() -> Self {
        Self { monad: true, ..Default::default() }
    }

    pub fn is_monad(&self) -> bool {
        self.monad
    }
}
```

### 4. Update Executor

```rust
// foundry/crates/anvil/src/eth/backend/executor.rs
use alloy_monad_evm::MonadEvmFactory;

pub fn new_evm_with_inspector<DB, I>(...) -> EitherEvm<DB, I, PrecompilesMap> {
    if env.networks.is_monad() {
        let evm_env = EvmEnv::new(
            env.evm_env.cfg_env.clone().with_spec(MonadSpecId::Monad),
            env.evm_env.block_env.clone(),
        );
        EitherEvm::Monad(MonadEvmFactory::default().create_evm_with_inspector(db, evm_env, inspector))
    } else if env.networks.is_optimism() {
        // ...existing OP code
    } else {
        // ...existing ETH code
    }
}
```

---

## Testing

### Unit Test Example

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use alloy_primitives::address;
    use revm::database_interface::EmptyDB;

    #[test]
    fn test_factory_create_evm() {
        let factory = MonadEvmFactory::default();
        let env = EvmEnv {
            block_env: BlockEnv::default(),
            cfg_env: CfgEnv::new_with_spec(MonadSpecId::Monad),
        };

        let evm = factory.create_evm(EmptyDB::default(), env);
        assert_eq!(evm.chain_id(), 1); // Default chain ID
    }

    #[test]
    fn test_precompiles_available() {
        let factory = MonadEvmFactory::default();
        let env = EvmEnv {
            block_env: BlockEnv::default(),
            cfg_env: CfgEnv::new_with_spec(MonadSpecId::Monad),
        };

        let evm = factory.create_evm(EmptyDB::default(), env);

        // ECRECOVER should be available
        let ecrecover = address!("0x0000000000000000000000000000000000000001");
        assert!(evm.precompiles().get(&ecrecover).is_some());
    }
}
```

### Integration Test with Anvil

```bash
# Start anvil with Monad
anvil --monad

# Deploy and test
cast send --create 0x6080... --rpc-url http://localhost:8545
cast run <tx-hash> --rpc-url http://localhost:8545
```

### Forge Test

```bash
forge test --monad -vvvv
```

---

## Summary

| Component | Crate | Status |
|-----------|-------|--------|
| `MonadSpecId` | monad-revm | ✅ Complete |
| `MonadHandler` | monad-revm | ✅ Complete |
| `MonadInstructions` | monad-revm | ✅ Complete |
| `MonadPrecompiles` | monad-revm | ✅ Complete |
| `monad_revm::MonadEvm` | monad-revm | ✅ Complete |
| `MonadContext` | monad-revm | ✅ Complete |
| `MonadBuilder` | monad-revm | ✅ Complete |
| `alloy_monad_evm::MonadEvm` | alloy-monad-evm | ✅ Complete |
| `MonadEvmFactory` | alloy-monad-evm | ✅ Complete |
| `EitherEvm::Monad` | Foundry | ❌ To implement |
| `NetworkConfigs.monad` | Foundry | ❌ To implement |

### Key Differences from alloy-op-evm

| Aspect | alloy-op-evm | alloy-monad-evm |
|--------|--------------|-----------------|
| Spec type | `OpSpecId` | `MonadSpecId` |
| Transaction type | `OpTransaction<TxEnv>` | `TxEnv` (standard) |
| Context chain data | `L1BlockInfo` | `()` (none) |
| Handler | `OpHandler` | `MonadHandler` |
| HaltReason | `OpHaltReason` | `HaltReason` (standard) |
| Error type | `EVMError<_, OpTransactionError>` | `EVMError<_>` (standard) |
| Instructions | Standard (EthInstructions) | MonadInstructions (custom gas) |

### Next Steps

1. ~~Create `alloy-monad-evm` crate~~ ✅
2. ~~Compile and fix any API mismatches~~ ✅
3. Add to Foundry's dependencies
4. Update `EitherEvm` with Monad variant
5. Add `--monad` flag to NetworkConfigs
6. Update executor
7. Test gas cost changes
