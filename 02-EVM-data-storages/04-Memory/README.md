# Memory
This section will explain the 3rd memory location in EVM - **Memory**.

## Table of Content
* [Memory Layout](#memory-layout)
* [Memory gas cost](#memory-gas-cost)
* [How memory works?](#how-memory-works?)
* [Memory for Arrays](#memory-for-arrays)
* [Memory for Structs](#memory-for-structs)
* [Dynamic Data in Memory](#dynamic-data-in-memory)
* [Copying Between Memory and Storage](#copying-between-memory-and-storage)
* [Optimising Memory Usage](#optimising-memory-usage)

## Memory Layout
- EVM memory is a volatile memory area used during contract execution. Once a function execution is completed, the memory is cleared.
- Memory is like an array of bytes, where each element can hold 1 byte which can be represented as 2 hexadecimal characters(from 0 to 9 and from a to f).
- Size of EVM Memory is 2^256 elements, ,<ins>but on practice we are using a small amount from this size</ins>. This is due to the cost of memory allocation - it increases quardatically. <ins>So the more **memory** you use the more you will pay</ins>.
Memory was created in this way in order to prevent users to abuse memory in Ethereum nodes.

Memory illustration: ⬇️  
![image alt](https://github.com/ohMySol/yul-book-examples/blob/1567d0f679e02878b452e40e10ea22576b73f5c0/Images/EVM%20Memory.jpg)

## Memory gas cost
### Terms
1. **Memory Expansion**: As memory grows, additional gas costs are incurred to allocate the new memory space.
2. **Initial Allocation**: The first 32 bytes of memory cost **3 gas**.
3. **Expansion Costs**: Memory costs grow quadratically with the total memory size in 32-byte chunks. The gas for the `MSTORE` or `MLOAD` opcodes includes the cost of memory expansion.

### Common Gas Costs
1. **Reading Memory**: `MLOAD` opcode costs **3 gas** per word.
2. **Writing to Memory**: `MSTORE` opcode costs **3 gas**, plus expansion costs if applicable.
3. **Copying to Memory**: Operations like `CALLDATACOPY` or `CODECOPY` include additional costs for memory expansion.

### Optimisation Practices
1. Minimise memory allocation to reduce expansion costs.
2. Avoid unnecessary memory copying operations.
3. Use memory-efficient algorithms to handle large data sets.

**For more information, see [this article](https://learnevm.com/chapters/evm/memory).**

## How memory works?
### Memory Slots
- Memory operates in a sequential manner, with slots indexed by word (32 bytes). This means that Read/Write operation in memory happens in chunks of 32 bytes. EVM uses 32 bytes memory slots, so values are always encoded in 32 bytes.
- Values stored in memory are еemporary and persist only during transaction execution.

Basic example of writing to memory:
```
contract MemoryExample {
    function example() external pure returns (uint256) {
        assembly {
            let ptr := mload(0x40) // Get the free memory pointer
            mstore(ptr, 123)       // Store 123 at the pointer location
            return(ptr, 0x20)      // Return the 32-byte value
        }
    }
}
```

### Memory Special Areas
1. `0x00` - `0x20` - first 64 bytes is a scratch space. Can be used to store anything here. For ex. you don't want to get a free memory pointer, so you can quickly store smth in the scratch space with less computations.
2. `0x40` - stores a <ins>free memory pointer</ins>. Free memory pointer - this is basically a pointer to memory area where we can store safely new data without overriding already existing data in memory. \
Initially free memory pointer points to 32 bytes starting from the `0x80` location.
3. `0x60` - zero slot.
4. `0x80` - initial free memory starts here.