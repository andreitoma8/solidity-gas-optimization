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

### Storage: some most expensive opcodes

The most expensive opcodes are the ones that interact with the storage of the blockchain. These are the opcodes that are used to read and write to the blockchain's state: `SSTORE` and `SLOAD`.

-   Writing a storage variable from 0 to non-zero value costs `20,000 gas units`
-   Writing a storage variable from non-zero to non-zero value costs `5,000 gas units`
-   Writing a storage variable from non-zero to 0 value refunds gas

#### Cold and Warm Storage

A storage variable that is accessed(read or write) first time in a transaction is considered `cold storage` and a variable that has been accessed before in the same transaction is considered `warm storage`. Accessing a warm storage variable costs less than accessing a cold storage variable.

- Reading a value from a warm storage slot costs `100 gas units`
- Reading a value from a cold storage slot costs `2,100 gas units`

#### Packing storage variables

If multiple storage variables declared in Solidity can be packed into a single storage slot, the compiler will do so. When reading one of the variables, the EVM will read the entire storage slot, so it's best to pack variables that are read together.

If they are not used together, it's best to declare them in separate storage slots. For example, if you declare:

```solidity
uint128 a;
uint128 b;
```

they will be packed into the same storage slot and when you'd use one of them to, the contract will actually read the whole sotrage slot, and then get the value by it's offset. This means extra computation and extra gas. In this case it's just best to declare the values as `uint256` to skip the unpacking when you use them.

### Memory usage

Memory in the EVM is cheap to use as long as the data is small, but the cost increases exponentially as the data size increases. In Solidity, memory is never cleared, so it's ideal to to reuse memory variables as much as possible.

### Transaction data gas cost

Since all the transactions are stored on the blockchain, the data size of the transaction is also stored on the blockchain. This means that we will pay gas for the data sent in the transaction. The cost of the data is `4 gas units` per 0x00 byte and `16 gas units` per non-0x00 byte.

## The 5 ways to save gas

### 1. Deployment: make the smart contract as small as possible

### 2. Execution: use as few or as cheap opcodes as possible

### 3. Execution: use as little memory as possible

### 4. Storage: use as few state changes as possible

### 5. Transaction: use as little data as possible

## Myths:

### Myth 1: Using smaller data types saves gas

This is not true. The EVM works with 256-bit words, so using smaller data types will only cost you extra gas for the conversion. When setting a value that's for example a `uint8` to a storage slot, the EVM will tax you for writing to the entire 256-bit word, not just the 8 bits that you are using. So it is good to pack variables in storage slots, but only when you are using(reading/writing) them together.

That being said, you can save gas by using smaller data types in constants, because the compiler will not store the entire 256-bit word in the bytecode, but only the value that you are using.

## The Solidity Optimizer:

The optimizer can be used to reduce the size of the bytecode generated by the Solidity compiler or it can be used to reduce the gas cost of a function call. The number of runs that is passed to the optimizer specifies roughly how often each opcode of the deployed code will be executed across the life-time of the contract. This means it is a trade-off parameter between code size (deploy cost) and code execution cost (cost after deployment). A “runs” parameter of “1” will produce short but expensive code. In contrast, a larger “runs” parameter will produce longer but more gas efficient code. The maximum value of the parameter is 2\*\*32-1.

It's best to think about how the contract will be used when chosing the number of runs. If the contract is only used once or twice, like for example a vesting contract, then it's best to use a low number of runs. If the contract is used a lot, like for example a DEX contract, then it's best to use a high number of runs.

# Tricks:

Each one of these ticks is safe to use in isolation, but they should be used with caution and analyzed in the context of your specific use case, because they could create vulnerabilities or bugs in your code.

## Make functions payable:

If your functions are non-payable, the Solidity compiler will add Opcodes to your code that will make sure that the function is not called with any Ether and will revert the transaction if it is. This is a good practice, but it will cost you extra gas.

```solidity
function f() external payable {
    // ...
}
```

## Unchecked arithmetic operations:

By default, Solidity will check for overflows and underflows in arithmetic operations. This is a good practice, but it will cost you extra gas. In the case where there is no risk of overflow/underflow, you can use the `unchecked` keyword to disable the checks.

```solidity
function f() external {
    unchecked {
        uint256 x = type(uint).max; // x will be 2**256 - 1
        x++; // x will be 0
    }
}
```

### Example:

A good example of a opportunity to use unchecked arithmetic operations is in the ERC20 standard, where you can use unchecked arithmetics for balance updates as long as you make sure that the total supply can never be overflown. That is because the total supply is always greater than or equal to the sum of all the balances.

```solidity
function transfer(address to, uint256 value) external returns (bool) {
    require(_balances[msg.sender]>=value, "ERC20: transfer amount exceeds balance");
    unchecked {
        _balances[msg.sender] -= value;
        _balances[to] += value;
    }
    emit Transfer(msg.sender, to, value);
    return true;
}
```

## Use Constant and Immutable variables:

Constant and Immutable variables are stored in the bytecode of the contract as opposed to storage variables which are stored in the state of the contract. This means that constant and immutable variables are way cheaper to access than storage variables.

### Constant variable:

Constant variables are variables that are declared in the solidity code and are known at compile time. They can not be changed after the contract is deployed.

### Immutable variable:

Immutable variables are variable that are declared and have to be initialized in the constructor. They can not be changed after the contract is deployed.

### Example:

```solidity
contract MyContract {
    uint256 public constant a = 8;
    uint256 public immutable b;

    constructor(uint256 _b) {
        b = _b;
    }
}
```

## Avoid writing to a sotrage slot with the same value it already holds

This will actually cost 100 gas, so it's best to check if the value is already set before writing to it.

### Example:

The following example only works as a gas optimization if the value was already read in the function, like for example in a require statement. Otherwise it will cost extra gas to read the value before writing to it.

```solidity
uint256 someValue = 8;

function setSomeValue(uint256 newValue) external {
    uint256 _someValue = someValue;

    require(_someValue < 100);

    if (_someValue != newValue) {
        someValue = newValue;
    }
}
```

## Push to arrays instead of rewriting it whole when possible

When writing a new array, even if it is the same as the old one, the EVM will re-write to the entire storage slot. This means that even if the same value is written at the same index, it will still cost the gas to access cold storage: `2,100 gas units` and the gas to write the same value : `100 gas units`.

### Example:

```solidity
uint256[] public someArray; // This is set to [1, 2, 3]

// If called with param [1, 2, 3, 4] his will cost gas for:
// 1. Writing to the array length storage slot as cold storage
// 2. Access cold storage and write the same value at index 0 (2100 + 100 gas units)
// 3. Access cold storage and write the same value at index 1 ...
// 4. Access cold storage and write the same value at index 2 ...
// 5. Writing to the array index 3 storage slot as cold storage
function setArray(uint256[] memory newArray) external {
    someArray = newArray;
}

// If called with 4, this will only cost gas for:
// 1. Writing to the array length storage slot as cold storage
// 2. Writing to the array index 4 storage slot as cold storage
function pushToArray(uint256 newValue) external {
    someArray.push(newValue);
}
```

## Avoid deleting long arrays

When deleting an array, the EVM will re-write to the entire storage slots used by the array to 0x00, this will refund some of the gas used to access the storage location and write to it, but up to a certain point. This is useful when deleting small arrays, but it's best to avoid it when deleting arrays with double or triple digit length.

## Avoid writing to storage slots that are set to 0x00

When writing to a storage slot that is set to 0x00 is some of the costly operations in the EVM.

### Example:

A good example of a optimization built around this is the 1inch Aggregator V5 contract that keeps a self balance of 1 tokens in the contract to avoid writing to the storage slot that is set to 0x00, which would be it's token balance if it would always send all the tokens it receives to the user. So the first ever swap of every token will basically lose 1 of the smallest unit of that token. You can actually check the balance of the contract on Etherscan looking at [Token Holdings](https://etherscan.io/address/0x1111111254eeb25477b68fb85ed929f73a960582) and see that it's always 1 token less than the total supply.

```solidity
function swap(...) external payable retuns (uint256 returnAmount) {
    // perform the swap ...

    returnAmount = dstToken.uniBalanceOf(address(this));
    require(returnAmount > 0, "1INCH: return amount is 0");
    unchecked {
        returnAmount--;
    }
}
```

## Pack variables together\*

\* when they fit inside 32 bytes and are read together.

When variables are packed together, they are stored in the same storage slot, which means that they will cost less gas to read and write both at the same time.

### Example:

```solidity
struct Stake {
    uint128 amount;
    uint128 stakeTime;
}

mapping(address => Stake) public stakes;

function stake(uint128 _amount) external {
    // stake logic ...
    stakes[msg.sender] = Stake({
        amount: _amount,,
        stakeTime: block.timestamp
    });
}
```

## Declare the array.lenght when looping through dynamic arrays

When looping through an array, it's best to declare the array length outside the loop, because otherwise the EVM will have to read the array length from storage on every iteration.

### Example:

```solidity
uint256[] public someArray;

function doSomething() external {
    uint256 _someArrayLength = someArray.length;
    for (uint256 i = 0; i < _someArrayLength; i++) {
        // do something ...
    }
}
```

## Function names

To call a Solidity function, the calldata contains the function selector, which is the first 4 bytes of the keccak256 hash of the function signature. On a low level, the EVM will compare the first 4 bytes of the calldata with the function selectors of all the functions in the contract to find the function to call. This means that the more functions a contract has, the more gas it will cost to call a function, but this can be optimized by making sure that the function selector of the function that will be most called has the lowest hex value and thus is the first function selector to be checked.(The functions are checked in increasing order of their function selector)

Also, since bytes zero are cheaper than non-zero bytes, the more bytes zero the function selector has, the cheaper it is to call the function.

### Example:

```solidity
// This function selector has the lowest hex value and the most bytes zero
// so it will be the cheapest to call: 0x0c41b033
function transferFrom(address from, address to, uint256 value) external returns (bool) {
    // transfer logic ...
}

// This function selector has the highest hex value and the least bytes zero
// so it will be the most expensive to call: 0xb483afd3
function transfer(address to, uint256 value) external returns (bool) {

    // transferFrom logic ...
}
```

## Comparison operators

EVM has no Opcodes for `>=` and `<=` operators, so they are compiled to a combination the available operators. This means that they will cost more gas than the other comparison operators. To optimize this, it's best to use the `>` and `<` operators instead.

### Example:

```solidity
if (someValue >= 1) {
    // do something ...
}

if (someValue > 0) {
    // do something ...
}
```
