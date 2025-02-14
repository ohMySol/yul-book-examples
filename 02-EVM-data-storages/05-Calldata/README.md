# Calldata
This section will explain the **Calldata** memory area in the EVM.

## Table of Content
* [Calldata Overview](#calldata-overview)
* [Calldata Layout](#calldata-layout)
* [Calldata vs Memory](#calldata-vs-memory)
* [How Calldata Works?](#how-calldata-works)
* [Calldata Types](#calldata-types)
* [Calldata for Structs](#calldata-for-structs)
* [Calldata for Fixed Arrays](#calldata-for-fixed-arrays)
* [Calldata for Dynamic Arrays](#calldata-for-dynamic-arrays)
* [Gas Costs of Calldata](#gas-costs-of-calldata)
* [Common Calldata Mistakes](#common-calldata-mistakes)
* [Copying Between Calldata and Memory](#copying-between-calldata-and-memory)

## Calldata Overview
- Calldata is a non-modifiable, read-only memory area that represents the entire data passed to a contract when calling a function. Means function selector and arguments.
- Unlike memory, and storage calldata is immutable(read-only), you can get data from calldata, but you can't modify it.
- Calldata is structured as a sequence of bytes, enabling precise and gas-efficient encoding of function arguments.
- Whenever we call a smart contract’s function, **what we’re actually doing is generating a bunch of ABI encoded data and send it to the EVM**. That generated data is called “calldata” and it includes all the information needed so that the EVM knows what function to call and with what parameter values.

## Calldata Layout
1. **Selector**: The first 4 bytes of the hash of function signature(keccak256 hash of the function name and arguments) specify the function selector.
2. **Arguments**: The remaining bytes represent encoded function arguments.

A transaction that calls a smart contract function will have an input value. That value is our calldata and here’s what it looks like:
```
|0x3fb5c1cb|000000000000000000000000000000000000000000000000000000000000000a|
|  4 bytes |                        32 bytes                                |
| Selector |          Function argument(10 in this case)                    |
```

## Calldata vs Memory
| Feature         | Memory           | Calldata       |
|------------------|------------------|----------------|
| **Mutability**   | Mutable          | Immutable      |
| **Use Case**     | Temporary storage | Function inputs |
| **Cost**         | Quadratic gas    | Lower gas cost |

## How Calldata Works?

### Common Operations
1. **Reading Arguments**: Use `calldataload` to read raw data.
2. **Copying to Memory**: Use `calldatacopy` to transfer data for manipulation.
3. **Dynamic Offsets**: Dynamic data (e.g., strings, arrays) includes offset pointers.

### Key Opcodes
1. **CALLDATALOAD**: Reads 32 bytes from calldata at a specified offset.
2. **CALLDATACOPY**: Copies a specified number of bytes from calldata to memory.
3. **CALLDATASIZE**: Returns the total size of the calldata payload.

## Calldata Encoding/Decoding

### Calldata Encoding
1. To encode types, you can pass them into the `abi.encode(parameters)` method to generate raw calldata.
2. You can also encode data to a specidic function in your interface with `abi.encodeWithSelector(selector, parameters)`. For example:
```
interface A {
  function transfer(uint256[] memory ids, address to) virtual external;
}

contract B {
  function a(uint256[] memory ids, address to) external pure returns(bytes memory) {
    return abi.encodeWithSelector(A.transfer.selector, ids, to);  // returns a calldata(selector + encoded args) for transfer function
  }
}
```
 - The method `.selector` generates the 4-bytes that represents that method on the interface. This is how UniswapV2 enables flashswaps.

### Calldata Decoding
1. We can decode the parameters with `abi.decode(...)` by passing in the parameters we want to decode the calldata into. For example:
```
(uint256 a, uint256 b) = abi.decode(data, (uint256, uint256))
```
 - **data** parameter is our calldata.

## Calldata Types
Each piece of calldata is 32 bytes long, and there are two types of calldata: **static** and **dynamic**.

###  Static Types
Static types are our value types: `uint`, `int`, `address`, `bool`, `bytes1` to `bytes32` (including function selector), and tuples (however they can have dynamic variables in them).
1. Here is an example of the contract `Example` with function `A` which takes a value arguments inside:
```
contract Example {
    function transfer(uint256 amount, address to) external view returns {}
}
```

 - When we do a call to the above function the calldata for this function will be the next:
 ```
 0xb7760c8f - function selector
 000000000000000000000000000000000000000000000000000000004d866d92 |0x00| - 1st input parameter uint256 `amount`
 00000000000000000000000068b3465833fb72a70ecdf485e0e4c7bd8665fc45 |0x20| - 2nd input parameter address `to`
 ```

###  Dynamic Types
Dynamic types are the next: `bytes`, `string`, and `dynamic arrays`.
1. They are encoded and stored in the next way:
 - **offset**. Offset - is a hexadecimal position where the data begins.
 - **dynamic data length**. Once we reach our offset the 1st what we will find is a length of our dynamic value. For arrays - this is a number of elements in the array. For bytes and strings - it represents the length of the type.
 - after the length is actual data(parameters values).
 
Example 1:
 ```
 // This is a function for which we would like to know a call data
 // Input values are: 7, [1,2,3], 9
 function threeArgs(uint256 a, uint256[] calldata b, uint256 c) external {}
 ```
Then a calldata will be the next(I already formatted it):
```
// 0xc6f922d - func selector
// 0000000000000000000000000000000000000000000000000000000000000007 |0x00| - 1st input value started after the func selector
// 0000000000000000000000000000000000000000000000000000000000000060 |0x20| - offset(pointer to where the length and the dynamic data is stored)
// 0000000000000000000000000000000000000000000000000000000000000009 |0x40| - 2nd input value 
// 0000000000000000000000000000000000000000000000000000000000000003 |0x60| - length of our dynamic array from input
// 0000000000000000000000000000000000000000000000000000000000000001 |0x80| - b[0] value
// 0000000000000000000000000000000000000000000000000000000000000002 |0xa0| - b[1] value
// 0000000000000000000000000000000000000000000000000000000000000003 |0xc0| - b[2] value
```

Example 2: \
`Hello World!` - get a calldata for this string👇
```
0000000000000000000000000000000000000000000000000000000000000020 |0x00| - offset(points to 0x20 area where the length and actual value are stored)
000000000000000000000000000000000000000000000000000000000000000c |0x20| - length of the string in hex(12)
48656c6c6f20576f726c64210000000000000000000000000000000000000000 |0x40| - string itself in hex
```