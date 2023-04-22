# Solidity Gas Optimization: Theory, Examples and Tricks

## Introduction

In the EVM Networks, each transaction that writes to the blockchain(modifies it's state) costs gas. The gas is paid by the original sender of the transaction, which is a EOA(externally-owned address). The gas is used to pay for the computational resources used by the node operators to process the transaction and it depends on 4 main factors:

1. The transaction's data size
2. The amount of memory used during the transaction
3. The state changes made by the transaction
4. The [Opcodes](https://github.com/andreitoma8/learn-EVM/blob/master/README.md#evm-opcodes) used by the transaction

## Calculating the gas cost of a transaction

The gas cost of a transaction is calculated by the following formula:

`GasCost = GasPrice * GasUsed`

Where:

-   `GasPrice` is the gas price of the block in which the transaction was included in `GWEI per gas unit`
-   `GasUsed` is the amount of gas used by the transaction in `gas units`
-   `GasCost` is the total cost of the transaction in `Ether`

While the `GasPrice` is fluctuating, depending on the network's demand, the `GasUsed` is a constant in the sense that the exact same transaction will always use the same amount of gas.

### ETH transfers

The gas cost of a simple ETH transfer is `21,000 gas units`. This is because the transaction only needs to write to the blockchain and it doesn't need to use any memory or state changes in Smart Contracts.

### Opcodes Gas Cost

Opcodes are the lowest level of instructions for the EVM. The cost of executing each EVM opcode is expressed in `gas units`. A complete list can be found on the [EVM codes website](https://www.evm.codes/?fork=grayGlacier) along with descriptions, costs and stack i/o.

#### The most expensive opcodes

The most expensive opcodes are the ones that interact with the storage of the blockchain. These are the opcodes that are used to read and write to the blockchain's state: `SSTORE` and `SLOAD`.

### Transaction data gas cost

Since all the transactions are stored on the blockchain, the data size of the transaction is also stored on the blockchain. This means that we will pay gas for the data sent in the transaction. The cost of the data is `4 gas units` per 0x00 byte and `16 gas units` per non-0x00 byte.

## The 5 ways to save gas

### 1. Deployment: make the smart contract as small as possible

### 2. Execution: use as few or as cheap opcodes as possible

### 3. Execution: use as little memory as possible

### 4. Storage: use as few state changes as possible

### 5. Transaction: use as little data as possible

# Tricks:

Each one of these ticks is safe to use in isolation, but they should be used with caution and analyzed in the context of your specific use case, because they could create vulnerabilities or bugs in your code.

## Make functions payable:

If your functions are non-payable, the Solidity compiler will add Opcodes to your code that will make sure that the function is not called with any Ether and will revert the transaction if it is. This is a good practice, but it will cost you extra gas.

```solidity
function f() external payable {
    // ...
}
```
