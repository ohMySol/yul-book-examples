# Memory
This section will explain the 3rd memory location in EVM - **Memory**.

## Table of Content
* [Memory Layout](#memory-layout)
* [Memory gas cost](#memory-gas-cost)
* [How memory works?](#how-memory-works?)
* [Memory for Variables](#memory-for-variables)
* [Memory for Structs](#memory-for-structs)
* [Memory for Fixed Arrays](#memory-for-fixed-arrays)
* [Memory for Dynamic Arrays](#memory-for-dynamic-arrays)
* [Memory for Return, Revert, Keccak256](#memory-for-return-revert-keccak256)
* [Logs](#logs)
* [Common memory mistakes](#common-memory-mistakes)
* [Copying Between Memory and Storage](#copying-between-memory-and-storage)


## Memory Layout
- EVM memory is a volatile memory area used during contract execution. Once a function execution is completed, the memory is cleared.
- Memory is like an array of bytes, where each element can hold 1 byte which can be represented as 2 hexadecimal characters(from 0 to 9 and from a to f). 
- Size of EVM Memory is 2^256 elements, ,<ins>but in practice we are using a small amount from this size</ins>. This is due to the cost of memory allocation - it increases quadratically. <ins>So the more **memory** you use the more you will pay</ins>.
Memory was created in this way in order to prevent users from abusing memory in Ethereum nodes.

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
2. `0x40` - stores a <ins>free memory pointer</ins>. Free memory pointer - this is basically a pointer to memory area where we can safely store new data without overriding already existing data in memory. \
Initially the free memory pointer points to 32 bytes starting from the `0x80` location.
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
1. Remember that data in memory is stored in 32 bytes chunks, so we just write to different chunks of new data one by one.
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
1. The same goes here, when we load smth from memory we read it in 32 bytes chunks.
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

## Memory for Dynamic Arrays
1. Dynamic arrays are different from fixed arrays, because here we follow the rule that **| length | actual value |**. 
2. When using dynamic data in memory - <ins>the length is stored in the first 32 bytes</ins>, and <ins>after the first 32 bytes the actual value starts</ins>.
```
uint256[] memory arr = [77,55, 33];

// the above dynamic memory array will looks like:
|  32 bytes  |  32 bytes  |  32 bytes  |  32 bytes  | - memory
-----------------------------------------------------
|     3      |    77      |     55     |   33       | - how dynamic data type(dynamic array) is stored in memory
-----------------------------------------------------
| arr length |   arr[0]   |   arr[1]   |  arr[2]    | - layout of the stored dynamic array
```

### Write to memory dynamic array
1. Store array position in memory(not necessary to do this to store data in dynamic array).
2. Store array length in memory(not necessary to do this to store data in dynamic array).
3. Store elements in memory.
4. Update free memory pointer. Remember when you are doing operations with memory <ins>in Solidity, your free memory pointer is updated automatically</ins>, but in YUL it is not working. <ins>After writing to memory in Yul, you should manually update a free memory pointer in order to omit data overriding</ins>.
```
contract WriteToDynamicArray {
    function writeToDynamicArr() external pure returns(bytes32 p, bytes32 fmptr, uint256[] memory) {
        uint256[] memory arr = new uint256[](0);  // memory dynamic array is initialized in free memory -> 0x80 
        
        assembly {
            p := arr                              // location where dynamic array starts in memory(0x80)
            mstore(arr, 3)                        // set the length of the array(store 3 in the first 32 bytes)
            mstore(add(arr, 0x20), 123)           // move pointer to the next free 32 bytes(add(arr, 0x20)) and store 123 val
            mstore(add(arr, 0x40), 456)           // move pointer to the next free 32 bytes(add(arr, 0x40)) and store 456 val | jumping over 0x40, because we need to skip the length(32 bytes) + first value(32 bytes) = 64 bytes or 0x40
            mstore(add(arr, 0x60), 789)           // move pointer to the next free 32 bytes(add(arr, 0x60)) and store 789 val | jumping over 0x60, because we need to skip the length(32 bytes) + 1st value(32 bytes) + 2nd value(32 bytes) = 96 bytes or 0x60
            
            mstore(0x40, add(arr, 0x80))          // update free memory pointer and point it to a memory area which is free | our array starts from 0x80 location and takes 128 bytes or 0x80, so we store in 0x40 a new free memory location     
            
            fmptr := mload(0x40)                  // get new free memory pointer
        }
        
        return (p, fmptr, arr);
    }
}
```

### Read from memory dynamic array

```
contract ReadFromDynamicArray {
    function readFromDynamicArr() external pure returns(bytes32 p, uint256 len, uint256 a, uint256 b, uint256 c) {
        uint256[] memory arr = new uint256[](3);  // memory dynamic array is initialized in free memory -> 0x80 
        
        // fill arr with values
        arr[0] = uint256(11);
        arr[1] = uint256(22);
        arr[2] = uint256(33);

        assembly {
            p := arr                        // location where dynamic array starts in memory(0x80)
            len := mload(arr)               // get the length of the array(load 4 from the first 32 bytes)
            a := mload(add(arr, 0x20))      // move pointer to the next 32 bytes(add(arr, 0x20)) to read val under arr[0]
            b := mload(add(arr, 0x40))      // move pointer to the next free 32 bytes(add(arr, 0x40)) to read val under arr[1]
            c := mload(add(arr, 0x60))      // move pointer to the next free 32 bytes(add(arr, 0x60)) to read val under arr[2]
        }
        
        return (p, len, a, b, c);
    }
}
```

## Memory for Return, Revert, Keccak256
1. Remember that if you want to return smth, hash smth with keccak256 or revert with custom error you need to store this in memory first. That's because <ins>EVM only returns values stored in memory, and because `keccak256() only looks in memory</ins>.

### Return
1. Store your values in memory.
2. Use a `return(sml, nob)` opcode to return your values from memory. `return()` takes 2 arguments: `sml` - start memory location from where to start reading values, `nob` - number of bytes to read. 
```
contract Return {
    function returnSmth() external pure returns(uint256, uint256) {
        mstore(0x00, 2)     // store 2 in memory scratch space
        mstore(0x20, 4)     // store 4 in memory scratch space
        
        return(0x00, 0x40)  // reading 0x40(64 bytes) starting from 0x00 memory location and return values.
    }
}
```

### Revert with message
1. If you want to revert with a custom message, then you should store the message value in memory first.
2. Use `revert(sml, nob)` opcode to revert in case of error. `revert()` - takes 2 arguments: - `sml` start memory location from where to start reading values for revert, `nob` - number of bytes to read.
```
contract Revert {
    function revertCall() external pure {
        assembly {
            let error := "Caller is not authorized!"        // Define the error message
            mstore(0x00, error)                             // Store the message in memory at 0x00
            revert(0x00, 0x20)                              // Revert with the first 32 bytes of data starting from 0x00
        }
    }
}

// you will receive this output: 0x43616c6c6572206973206e6f7420617574686f72697a65642100000000000000 - decode it and you will see an error msg.
```

### Revert without message
1. If you want to save gas and you don't need a custom message, then you can write like this `revert(0,0)`. This means that error data location starts at 0x0 and size of the data is 0, so no error data is returned, it just stops execution and reverts the transaction.
```
contract Revert {
    function revertCall() external pure {
        assembly {
            if iszero(eq(caller(), 0x1d22514C2c914151166C126067Ba683A988AD5ca)) { // check if caller equals a specific address
                revert(0,0) // revert with no error data
            }
        }
    }
}
```

### Keccak256
1. Store your values in memory.
2. Use a `keccak256(sml, nob)` opcode to hash your values stored in memory. `keccak256()` takes 2 arguments: `sml` - start memory location from where to start reading values for hashing, `nob` - number of bytes to read.
 - To encode data before hashing we need just to store it in memory close to each other like this:
```
assembly {
    // the below 2 lines are equivalent to abi.encode(123, msg.sender)
    mstore(0x40, 123)                   // store `123` in 0x80 location
    mstore(add(0x40, 0x20), caller())   // store `mg.sender` in 0xa0 location
}
```

```
contract HashDataInMemory {
    function hashData() external pure returns(bytes32) {
        assembly {
            let ptr := mload(0x40)
        
            mstore(ptr, 1)                      // store 1 in memory location 0x80
            mstore(add(ptr, 0x20), 2)           // store 2 in memory location 0xa0
            mstore(add(ptr, 0x40), 3)           // store 4 in memory location 0xc0

            mstore(0x40, add(ptr, 0x60))        // update free memory pointer
        
            mstore(0x00, keccak256(ptr, 0x60))  // store hash in memory scratch space 0x00 | starting from the `ptr`, we are looking for the next 96 bytes(0x60) and hash them.
        
            return(0x00, 0x20)                  // return the 32 bytes from 0x00 where our hash is stored
        }                
    }
}
```

## Logs
In this section I decided to combine both logs with `indexed` parameters and `non-indexed`. In Solidity, <ins>indexed arguments are handled implicitly by the compiler, which organizes them into topics</ins>. In Yul, <ins>you must explicitly put indexed arguments, calculate the event signature hash and set memory area for non indexed arguments<ins>.
1. We have 2 types of log arguments: `indexed` and `non-indexed`. Also in EVM we have 2 areas where event arguments can be stored: `log topics` and `log data`. Indexed arguments are stored directly in the log topics, and non-indexed arguments are stored in the log data area.
2. **Log Topics** - is an array of up to 4 elements. This array consists of:
 - the first topic is always the event signature hash(keccak-256 of the event's name and parameter types, ex: `keccak256("Log1(uint256, uint256)")`).
 - next topics are the values of **indexed arguments**.
3. **Log Data** - is a chunk of memory which stores **all non-indexed arguments** of event.
4. To emit an event in Yul, we need to use `log` or `logN` opcode where `N` - is a number of event topics(up to 4). \
For ex: `log3(sml, nob, t1, t2, t3)`:
 - `sml` - start memory location from where to start reading non-indexed arguments.
 - `nob` - number of bytes to read for non-indexed arguments.
 - `t1` - first topic, always `keccak256()` hash of the event signature.
 - `t2` - first indexed argument.
 - `t3` - second indexed argument.

### Emit event with indexed arguments
1. Get the hash of the event signature.
2. Call `logN` opcode to emit an event.
```
contract LogEvent {
    event Log1(uint256 indexed a, uint256 indexed b);             // event with indexed parameters.

    function emitIndexedEvent() external {
        bytes32 signature = keccak256("Log1(uint256, uint256)");  // keccak256(event signature) is captured
        
        assembly {
            // The first and second args. denote the area of memory to return as the non-indexed arguments
            // Since we are not indexing any arguments we can set them to 0
        
            // The 1st topic, t1, is the hash of the event signature
            // The 2nd topic, t2, is the indexed data 'a'
            // The 3rd topic, t3, is the indexed data 'b'
            log3(0, 0, signature, 5, 6)
        }
    }
}
```

 - In the transaction logs you will see this:
 ```
 "data": "0x", - nothing in the log data, because we don't have non-indexed args.
    "topics": [
      "0x380e8667b0508e702edf34909096333031642d7b360c40d86cf692570d58b2fa", - topic 1 = event signature hash
      "0x0000000000000000000000000000000000000000000000000000000000000005", - topic 2 = indexed argument 5
      "0x0000000000000000000000000000000000000000000000000000000000000006"  - topic 3 = indexed argument 6
    ]
 ```

### Emit event with non-indexed arguments
<ins>Non-indexed arguments must first be stored in memory</ins>. Non-indexed arguments will be stored in the `log data` area.
1. Get the hash of the event signature.
2. Call `logN` opcode to emit an event. 
```
contract LogEvent {
    event Log2(uint256 indexed a, uint256 b);                     // event with indexed and non-indexed parameters

    function emitNonIndexedEvent() external {
        bytes32 signature = keccak256("Log2(uint256, uint256)");  // keccak256(event signature) is captured
        
        assembly {
            mstore(0x00, 0x22)  // non-indexed args. are first stored in memory | 0x22 = decimal 34

            // 0x00, 0x20 - memory area from where we will read a non-indexed parameter   
            // The 1st topic, t1, is the hash of the event signature
            // The 2nd topic, t2, is the indexed data 'a'
            log2(0x00, 0x20, signature, 123) // we use log2, because we have only 2 topics(event signature hash, and 1 indexed argument)
        }
    }
}
```
 
- In the transaction logs you will see this:
```
"data": "0x0000000000000000000000000000000000000000000000000000000000000022", - non-indexed value 0x22(34) is stored in the log data
    "topics": [
      "0x9b80f0b7d5e715f6a406db3bdf3ec3222f9f2db123596c1b7d2ff4ff3ea58713",   - topic 1 = event signature hash
      "0x000000000000000000000000000000000000000000000000000000000000007b"    - topic 2 = indexed argument 123
    ]
```

## Common memory mistakes

### Overwriting existing data
1. In memory if you don’t reallocate or update the free memory pointer in Yul after using memory, you risk overwriting existing data or corrupting memory when the next operation requiring memory allocation is performed. This can lead to unexpected behavior.
2. In Solidity this happens automatically, but in Yul you need to do this manually. Below code examples shows how quickly we can do a bug if we'll use memory not in the correct way:
```
contract MemoryOverwriting1 {
    function owerwriteMemory1(uint256 val1, uint256 val2) external pure returns (uint256 val) {
        assembly {
            let ptr := mload(0x40)       // get a free memory pointer(points to 0x80)
            mstore(ptr, val1)            // store in 0x80 our `val1`
        
            let newPtr := mload(0x40)    // get new free memory pointer without reallocation(also points to 0x80)
            mstore(newPtr, val2)         // store in 0x80 our `val2` and overwrite initial `val1` stored at 0x80 
        
            val := mload(ptr)            // return `val2` instead of `val1`
        }
    }
}
```

and 

```
contract MemoryOverwriting2 {
    function owerwriteMemory2(uint256[1] memory foo) external pure returns (uint256) { // `foo` is already stored in `0x80` memory location.
        assembly {
            mstore(0x40, 0x80)                  // this pointer should be moved forward to `0xa0`, because `foo` is already stored in `0x80` memory location.
        }
       
        uint256[1] memory bar = [uint256(6)];   // here `foo` is going to be overwritten with `bar`, because in assembly block we incorrectly set a free memory pointer again to 0x80
        return foo[0];                          // `bar` value will be returned back instead of `foo` value
    }
}
```

### Inefficient Memory Allocation With Arrays
1. Copying a storage array to memory, especially when it has small elements like uint8, can be wasteful because:
 
- each element occupies an entire 32-byte memory slot, even if in storage it was uint8. EVM unpacks the storage array and allocates 32 bytes <ins>for each element in memory, regardless of the element size</ins>(uint8 is only 1 byte but occupies 32 bytes in memory).

 - this can significantly increase the tx cost and lead to out-of-gas errors for large arrays.

 - remember about the rule: <ins>the more memory we use the more expensive it will be to us, because memory cost grows quadratically</ins>.
```
contract MemoryArrayUnpack {
    uint8[] foo = [1,2,3,4,5,6];            // all these values will be stored into 1 32 bytes slot in storage, because values are uint8 size
    
    function unpacked() external view {
        uint8[] memory bar = foo;           // however in memory `foo` array will be unpacked, and each array elem. will be stored in a separate 32 bytes memory location.
    }
}
```

### Assembly Return holds execution
1. Be careful with **assembly return**, because it will hold(stop) an execution of your program and everything after `return()` opcode will be ignored. Pretty obvious because usual Solidity `return()` does the same.  
I am mentioning this, because sometimes you want to continue your code in Solidity and forget about the `return()` opcode in your Yul block. So this can also lead to bugs and unexpected behaviour.
2. The return opcode signifies the end of the current execution context:

 - the specified memory data is sent to the caller as the function's result.

 - the program counter stops execution for the current function or contract.

 - no further instructions are executed in the same call context.
```
contract MemoryReturnBug {
    function returnValues() public pure returns (uint256, uint256) {
        assembly {
            mstore(0x80, 11)    // store 11 at 0x80 location
            mstore(0xa0, 12)    // store 12 at 0xa0 location
            return(0x80, 0x40)  // return values, and this will hold(stop) execution of your code
        }
    }

    function returnBug() public pure returns (uint256, uint256) {
        (uint256 x, uint256 y) = returnValues(); // execution stops in this func on the line: 'return(0x80, 0x40)' 
        return(123, 456);                        // this line won't be reached by evm, because it will stop on return from assembly block, and you will receive 11, 12 instead of 123, 456
    }
```