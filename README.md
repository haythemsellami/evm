# :construction: This repository is a work in progress.

# Alloy-EVM

## Overview

`alloy-evm` is an abstraction layer on top of [revm](https://github.com/bluealloy/revm) providing common implementations of EVMs. Currently, alloy-evm is only used in Reth but is designed to be consumed by any project that needs to execute/trace transactions or blocks on EVM compatible chains.

`alloy-evm` is compatible with no_std and riscv targets.

## Alloy Monad EVM

`alloy-monad-evm` provides Monad chain support by wrapping [monad-revm](https://github.com/haythemsellami/revm/tree/main/crates/monad-revm) with `alloy-evm` traits (`Evm`, `EvmFactory`). It includes Monad-specific gas costs for cold storage/account access and custom precompile pricing.

#### License

<sup>
Licensed under either of <a href="LICENSE-APACHE">Apache License, Version
2.0</a> or <a href="LICENSE-MIT">MIT license</a> at your option.
</sup>

<br>

<sub>
Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in these crates by you, as defined in the Apache-2.0 license,
shall be dual licensed as above, without any additional terms or conditions.
</sub>
