# Memory
This section will explain the 3rd memory location in EVM - **Memory**.

## Table of Content
* [Memory Layout](#memory-layout)
* [Memory gas cost](#memory-gas-cost)
* [How memory works?](#how-memory-works?)
* [Memory for Variables](#memory-for-variables)
* [Memory for Structs](#memory-for-structs)
* [Memory for Fixed Arrays](#memory-for-fixed-arrays)
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

### Memory Special Areas
1. `0x00` - `0x20` - first 64 bytes is a scratch space. Can be used to store anything here. For ex. you don't want to get a free memory pointer, so you can quickly store smth in the scratch space with less computations.
2. `0x40` - stores a <ins>free memory pointer</ins>. Free memory pointer - this is basically a pointer to memory area where we can store safely new data without overriding already existing data in memory. \
Initially free memory pointer points to 32 bytes starting from the `0x80` location.
3. `0x60` - zero slot.
4. `0x80` - initial free memory starts here.

### MLOAD, MSTORE, MSTORE8, MSIZE, MCOPY
1. [MLOAD](https://www.evm.codes/?fork=cancun#51) - reads 32 bytes(256 bits) from memory starting from a specified location. `mload(0x40)` - reads 32 bytes starting from 0x40 memory location, and returnes a value stored in these 32 bytes(free memory pointer is returned).
2. [MSTORE](https://www.evm.codes/?fork=cancun#52) - writes 32 bytes(256 bits) to memory at starting from a specified location. `mstore(0x80, 22)` - write value 22, starting from 0x80 memory location.
3. [MSTORE8](https://www.evm.codes/?fork=cancun#53) - writes 1 byte(8 bits) to memory starting from a specified location.
4. [MSIZE](https://www.evm.codes/?fork=cancun#59) - returns the size of the allocated memory (in bytes).
```
// Initially, MSIZE returns 0 because no memory beyond 0x80 is allocated.
// After storing data at start, memory size increases to at least 0xA0 (since 0x12345678 occupies one 32-byte word starting at 0x80).
{
    let start := mload(0x40)       // Get the free memory pointer (initially 0x80)
    mstore(start, 0x12345678)      // Store some data at `start`

    let memSize := msize()         // Get the current memory size

    log1(0, 0, memSize)            // Log memory size (rounded to nearest 32 bytes)
}
```
5. [MCOPY](https://www.evm.codes/?fork=cancun#5e) - copies a specified number of bytes from one memory region to another.
```
assembly {
    let src := 0x40             // Source memory location
    let dest := 0xA0            // Destination memory location
    let length := 0x20          // Number of bytes to copy (32 bytes)

                                // Write some data to the source memory
    mstore(src, 0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef)

    mcopy(dest, src, length)    // Copy data from `src` to `dest`

}
```

### How to count in memory ?
1. Memory operates in a sequential manner, with slots indexed by word (32 bytes). This means that Read/Write operation in memory happens in chunks of 32 bytes. EVM uses 32 bytes memory slots, so values are always encoded in 32 bytes.
2. `0x20` = 32 bytes. So when you see for ex `mload(add(0x80, 0x20))` - this means that we firstly add 32 bytes(0x20) to `0x80` to jump to the next memory location `0xA0`, and secondly we read 32 bytes from `0xA0`.
3. When you add memory locations like 0x80 + 0x20, the calculation is performed as hexadecimal addition:
```
   0x80
+  0x20
-------
   0xA0
```

## Memory for Variables

### Write to memory variable
1. As u already know we do this with `mstore(p, v)`, where `p` - position in memory start writing from, and `v` - is our value to write.
```
contract WriteToMemory {
    function writeToMem() external pure {
       assembly {
            mstore(0x00, 123)   // store `123` value in memory scratch space
       }
    }
}
```

### Read from memory variable
1. To read variable we do `mload(p)`, where `p` - position in memory start reading from.
```
contract ReadFromMemory {
    function readFromMem() external pure returns (uint256 val) {
        assembly {
            mstore(0x00, 123)   // store `123` value in memory scratch space
            val := mload(0x00)  // read 32 bytes starting from 0x00 mem location
        }
    }
}
```

## Memory for Structs

### Write to memory struct
1. Remember that data in memory stored in 32 bytes chunks, so we just write to different chunks new data one by one.
```
contract WriteToStruct {
    struct Point {
      uint256 a;
      uint32 b;
      uint32 c;
    }

    function writeToStruct() external pure returns(uint256 a, uint32 b, uint32 c) {
        Point memory point;     // struct is declared in free memory -> 0x80(because we just start using memory) 
        
        assembly {
            mstore(0x80, 12)   // store `12` in 0x80 location.
            mstore(0xa0, 15)   // store `15` in the next 32 bytes | 0xa0 = add(0x80, 0x20)
            mstore(0xc0, 20)   // store `20` in the next 32 bytes | 0xc0 = add(0xa0, 0x20)
        }

        // check that values are successfully stored
        a = point.a;
        b = point.b;
        c = point.c;
    }
}
```

### Read from memory struct
1. The same goes here, when we will load smth from memory we read it in 32 bytes chunks.
```
contract ReadFromStruct {
    struct Point {
      uint256 a;
      uint32 b;
      uint32 c;
    }

    function readFromStruct() public pure returns (uint256 a, uint256 b, uint256 c) {
        Point memory point = Point({a: 1, b: 2, c: 3}); // memory struct is declared in free memory -> 0x80
        
        assembly {
            a := mload(0x80)    // load the first 32 bytes starting from 0x80
            b := mload(0xa0)    // load the next 32 bytes starting from 0xa0 = add(0x80, 0x20)
            c := mload(0xc0)    // load the next 32 bytes starting from 0xc0 = add(0xa0, 0x20)
        }
    }
}
```

## Memory for Fixed Arrays

### Write to memory fixed array
1. Here the same rules as for memory structs. In storage fixed array we won't have array length stored somewhere, because obviously we know array length in advance.
```
contract WriteToFixedArray {
    function writeToFixedArr() external pure returns(uint256 a, uint256 b, uint256 c) {
        uint256[3] memory arr; // memory fixed array is declared in free memory -> 0x80 
        
        assembly {
            mstore(0x80, 123)  // store `123` in 0x80 location.
            mstore(0xa0, 456)  // store `456` in the next 32 bytes | 0xa0 = add(0x80, 0x20)
            mstore(0xc0, 789)  // store `789` in the next 32 bytes | 0xc0 = add(0x80, 0x20)
        }
        
        // check that values are successfully stored
        a = arr[0];
        b = arr[1];
        c = arr[2];
    }
}
```

### Read from memory fixed array
1. Here the same rules as for memory structs.
```
contract ReadFromFixedArray {
    function readFromFixedArr() external pure returns(uint256 a, uint256 b, uint256 c) {
        uint96[3] memory arr = [uint96(33), uint96(22), uint96(77)]; // memory fixed array is declared in free memory -> 0x80 
        
        assembly {
            a := mload(0x80)  // load the first 32 bytes starting from 0x80
            b := mload(0xa0)  // load the next 32 bytes starting from 0xa0 = add(0x80, 0x20)
            c := mload(0xc0)  // load the next 32 bytes starting from 0xc0 = add(0xa0, 0x20)
        }
    }
}
```