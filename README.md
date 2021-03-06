[![Included in the Arctic Code Vault](https://img.shields.io/badge/Arctic%20Code%20Vault-blue)](https://archiveprogram.github.com/)

# Substrate Node + EVM Pallet Template

**This project has been archived.** Please refer to more up-to-date projects, such as
[Parity's Frontier project](https://github.com/paritytech/frontier) and the fantastic work being
done by the [Moonbeam Network](https://docs.moonbeam.network/).

A [FRAME](https://substrate.dev/docs/en/next/conceptual/runtime/frame)-based
[Substrate](https://substrate.dev/en/) node with the
[EVM pallet](https://substrate.dev/docs/en/next/conceptual/runtime/frame#evm), ready for hacking
:rocket:

## Upstream

This project was forked from the
[Substrate Node Template](https://github.com/substrate-developer-hub/substrate-node-template).

## Build & Run

To build the chain, execute the following commands from the project root:

```
$ ./scripts/init.sh && cargo build --release
```

To execute the chain, run:

```
$ ./target/release/substrate-evm --dev
```

A [`makefile`](/makefile) is provided in order to document and encapsulate commands such as these.
In this case, the above commands are associated with the `build-chain` and `start-chain` `make`
targets.

## Genesis Configuration

The development [chain spec](/src/chain_spec.rs) included with this project defines a genesis block
that has been pre-configured with an EVM account for
[Alice](https://substrate.dev/docs/en/next/development/tools/subkey#well-known-keys). When
[a development chain is started](https://github.com/substrate-developer-hub/substrate-node-template#run),
Alice's EVM account will be funded with a large amount of Ether (`U256::MAX`). The
[Polkadot UI](https://substrate.dev/docs/en/next/development/front-end/polkadot-js#polkadot-js-apps)
can be used to see the details of Alice's EVM account. In order to view an EVM account, use the
`Developer` tab of the Polkadot UI `Settings` app to define the EVM `Account` type:

```json
"Account": {
    "nonce": "U256",
    "balance": "U256"
  }
```

Use the `Chain State` app's `Storage` tab to query `evm > accounts` with Alice's EVM account ID
(`0x57d213d0927ccc7596044c6ba013dd05522aacba`); the value that is returned should be:
`{"nonce":0,"balance":"0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff"}`.

> Further reading:
> [EVM accounts](https://github.com/danforbes/danforbes/blob/master/writings/eth-dev.md#Accounts)

Alice's EVM account ID was calculated using
[a provided utility](/utils/README.md#--evm-address-address).

## Contract Deployment

The [`truffle`](/truffle) directory contains a [Truffle](https://www.trufflesuite.com/truffle)
project that defines [an ERC-20 token](/truffle/contracts/MyToken.sol). For convenience, this
repository also contains
[the compiled bytecode of this token contract](/truffle/build/contracts/MyToken.json#L259), which
can be used to deploy it to the Substrate blockchain.

> Further reading:
> [the ERC-20 token standard](https://github.com/danforbes/danforbes/blob/master/writings/eth-dev.md#EIP-20-ERC-20-Token-Standard)

Use the Polkadot UI `Extrinsics` app to deploy the contract from Alice's account (submit the
extrinsic as a signed transaction) using `evm > create` with the following parameters:

```
init: <contract bytecode>
value: 0
gas_limit: 4294967295
gas_price: 1
```

The values for `gas_limit` and `gas_price` were chosen for convenience and have little inherent or
special meaning.

While the extrinsic is processing, open the browser console and take note of the output. Once the
extrinsic has finalized, the EVM pallet will fire a `Created` event with an `address` field that
provides the address of the newly-created contract. In this case, however, it is trivial to
[calculate this value](https://ethereum.stackexchange.com/a/46960):
`0x11650d764feb44f78810ef08700c2284f7e81dcb`. That is because EVM contract account IDs are
determined solely by the ID and nonce of the contract creator's account and, in this case, both of
those values are well-known (`0x57d213d0927ccc7596044c6ba013dd05522aacba` and `0x0`, respectively).

Use the `Chain State` app to view the EVM accounts for Alice and the newly-created contract; notice
that Alice's `nonce` has been incremented to `1` and her `balance` has decreased. Next, query
`evm > accountCodes` for both Alice's and the contract's account IDs; notice that Alice's account
code is empty and the contract's is equal to the bytecode of the Solidity contract.

## Contract Storage

The ERC-20 contract that was deployed inherits from
[the OpenZeppelin ERC-20 implementation](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol)
and extends its capabilities by adding
[a constructor that mints a maximum amount of tokens to the contract creator](/truffle/contracts/MyToken.sol#L8).
Use the `Chain State` app to query `evm > accountStorage` and view the value associated with Alice's
account in the `_balances` map of the ERC-20 contract; use the ERC-20 contract address
(`0x11650d764feb44f78810ef08700c2284f7e81dcb`) as the first parameter and the storage slot to read
as the second parameter (`0xa7473b24b6fd8e15602cfb2f15c6a2e2770a692290d0c5097b77dd334132b7ce`). The
value that is returned should be
`0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff`.

The storage slot was calculated using
[a provided utility](/utils/README.md#--erc20-slot-slot-address).

> Further reading:
> [EVM layout of state variables in storage](https://solidity.readthedocs.io/en/v0.6.2/miscellaneous.html#layout-of-state-variables-in-storage)

## Contract Usage

Use the `Extrinsics` app to invoke the `transfer(address, uint256)` function on the ERC-20 contract
with `evm > call` and transfer some of the ERC-20 tokens from Alice to Bob.

```
target: 0x11650d764feb44f78810ef08700c2284f7e81dcb
input: 0xa9059cbb0000000000000000000000008bc395119f39603402d0c85bc9eda5dfc5ae216000000000000000000000000000000000000000000000000000000000000000dd
value: 0
gas_limit: 4294967295
gas_price: 1
```

The value of the `input` parameter is an EVM ABI-encoded function call that was calculated using
[the Remix web IDE](http://remix.ethereum.org); it consists of a function selector (`0xa9059cbb`)
and the arguments to be used for the function invocation. In this case, the arguments correspond to
Bob's EVM account ID (`0x8bc395119f39603402d0c85bc9eda5dfc5ae2160`) and the number of tokens to be
transferred (`0xdd`, or 221 in hex).

> Further reading:
> [the EVM ABI specification](https://solidity.readthedocs.io/en/v0.6.2/abi-spec.html)

After the extrinsic has finalized, use the `Chain State` app to query `evm > accountStorage` to see
the ERC-20 balances for both Alice and Bob.

## Polkadot UI

This project contains [a Polkadot UI](/ui) with some custom components for the EVM pallet.

## Raspberry Pi

See [the `rpi` directory](/rpi) for information about cross-compiling a Substrate node for execution
on a Raspberry Pi.
