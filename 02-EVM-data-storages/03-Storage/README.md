# Storage
This section will explain the 2nd storage location in EVM - **Storage**.

## Table of Content
* [Storage Layout](#storage-layout)
* [Storage gas cost](#Storage-gas-cost)
* [How storage works?](#how-storage-works?)
* [Storage Packed Values](#storage-packed-values)
* [Storage Struct](#storage-struct)
* [Storage Fixed Array](#storage-fixed-array)
* [Storage Dynamic Data](#storage-dynamic-data)
* [Sorage Dynamic Array](#storage-dynamic-array)
* [Storage Mapping](#storage-mapping)
* [Storage Nested Mapping](#storage-nested-mapping)
* [Storage Mapping To Dynamic Array](#storage-mapping-to-dynamic-array)
* [How write to arrays and mappings?](#how-to-write-to-arrays-and-mappings?)
* [Common Storage Mistakes](#common-storage-mistakes)

## Storage Layout 
 - EVM storage(contracts storage) is a memory location where data is stored permanently. Once a function finish it's execution storage data won't be cleared out. 
 - As u may understand, storage is the most expensive memory area in EVM, because data exists here permanently.
 - Storage consists of multiple slots of 32 bytes. Storage size is 2^256 slots, and obviously each slot can store values up to 32 bytes. You can think of this like a mapping uint256 slot => uint256 value. \
Storage illustration: ⬇️ \
![image alt](https://github.com/ohMySol/yul-book-examples/blob/e53fc1ed08d0f6e9fe08949580e220eba83249a6/Images/Storage%20illustration.jpg)

## Storage gas cost
### Terms
1. <ins>Touch</ins> - when you store(SSTORE) smth or read(SLOAD) smth from the storage slot, then this slot is considered “touched” for the rest of the transaction.
2. <ins>Cold vs. Warm Slots</ins> - cold slots are accessed for the first time in a transaction (**cost 2100 gas**), while warm slots have already been accessed (**cost 100 gas**).
3. <ins>Clean vs. Dirty Slots</ins> - a clean slot retains its original value from the transaction start, while a dirty slot's value differs.
4. <ins>Static/Dynamic Costs</ins> - all opcodes have fixed static costs; some include dynamic costs based on context (e.g., memory expansion for MSTORE or slot status for SSTORE).
5. <ins>Gas Refunds</ins> - SSTORE can provide refunds when reducing work, such as setting a value to zero.

### Common Gas Cost Scenarios
1. Highest Cost: Writing a non-zero value to an empty slot (**22,100 gas**).
2. High Cost: Writing a non-zero value to a non-empty slot (**2900 gas** if warm, **5000** if cold).
3. Medium Cost: Reading from a cold slot (**2100 gas**) or setting a value to zero with refunds (cost as low as **200 gas**).
4. Low Cost: Reading from or writing the same value to a warm slot (**100 gas**), or reverting a slot's value to its original state (**almost full refund**).

### Optimisation practices
1. Writing to warm slots or reusing already-touched storage slots minimizes gas costs.
2. Avoid unnecessary reads/writes to reduce dynamic costs.
3. Take advantage of gas refunds for zeroing out or restoring original values.

**For more information I highly recommend checking [this article](https://learnevm.com/chapters/evm/storage).**

## How storage works?
### How are the slots assigned?
1. Remember slots are assigned in the order the variables are declared. Exceptions are dynamic arrays and mappings.

Example how the slots are assigned:
```
contract Example {
    uint256 a;   // slot 0
    uint256 b;   // slot 1
    uint256 c;   // slot 2
    bool isTrue; // slot 3
}
```
### SLOAD, SSTORE
1. [SSLOAD](https://www.evm.codes/?fork=cancun#54) and [SSTORE](https://www.evm.codes/?fork=cancun#55) are the only 2 methods that allow us to read from storage and write to it.

 - SSTORE - evm instruction for writing 32 bytes to storage. Usage: **sstore(s, v)**, where **s** - slot to write, **v** - value to write.  
```
contract WriteToStorage {
    // slot 0
    uint256 a;
    
    function example() external returns(uint256) {
        assembly {
            sstore(a.slot, 10) // a.slot receives an allocated slot for 'a' variable. In our case it's slot 0.
        }

        return a; // in tx logs you will see a returned 10 value
    }
}
```

 - SLOAD - evm instruction for loading 32 bytes from storage. Usage: **sload(s)**, where **s** - slot to read from.
```
contract ReadFromStorage {
    // slot 0
    uint256 a = 123;
    
    function example() external view returns(uint256 res) {
        assembly {
            res := sload(a.slot) // return 32 bytes from slot 0
        }
    }
}
```

## Storage Packed Values
### What are packed variables? 
1. Variables with the size less than 32 bytes are packed into a single slot(they are packed from right to left).

 - Imagine we have 2 variables `uin128 a` and `uint128` - they both are 16 bytes size, so they will be packed into a single slot.

Example:
```
contract PackedSlot {
    // slot 0
    uint128 a = 123; 
    uint128 b = 456;
    
    function example() external view returns(bytes32 res) {
        assembly {
            res := sload(0) // load avalues from slot 0
        }
    }
}
```
 - Result is `0x000000000000000000000000000001c80000000000000000000000000000007b`. As I mentioned, the packed result is stored from right to left in a 32 byte slot. So `7b` is 123 and `1c8` is 456.

### Reading from packed slot(returning 256 bits value)
Below is an example of how we can read from the packed slot in Yul.
1. Check how the slot 0 is packed. All the values are packed from right to left.
```
contract PackedStorage {
    uint128 public C = 4; // slot 0
    uint96 public D = 6;  // slot 0
    uint16 public E = 8;  // slot 0
    uint8 public F = 1;   // slot 0
    
    // function returns 32 bytes from `slot`. Our target slot is 0.
    function readBySlot(uint256 slot) external view returns (bytes32 value) {
        assembly {
            value := sload(slot) // result is 0x0001000800000000000000000000000600000000000000000000000000000004
        }
    }
}
```

2. Let's get an offset of variable `E` from the above contract. Offset is a place in bytes array of the packed slot where our variable starts(sounds difficult but take a look on the below example)
```
// returns the location of variable `E` in 32 bytes packed slot 
function getOffsetE() external pure returns (uint256 offset) {
        assembly {
            offset := E.offset // returns 28. This means that in bytes array of the packed slot value for `E` variable starts 28 bytes left(shift right by 28 bytes).
            // |00|01|00|08|00|00|00|00|00|00|00|00|00|00|00|06|00|00|00|00|00|00|00|00|00|00|00|00|00|00|00|04| - our bytes array. E value starts 28 bytes left.
            // | 1| 2| 3| 4| 5| 6| 7| 8| 9|10|11|12|13|14|15|16|17|18|19|20|21|22|23|24|25|26|27|28|29|30|31|32| - bytes position numbers. Count 28 bytes left.
            // Our E value is stored in 2 bytes starting from the 4th byte position(32 - 28 = 4): 0008 - our E value
        }
    }
```

3. Now we can put everything together and read the `E` value from the packed storage slot. We will jump to the needed bytes with the help of shift right opcode - **shr(nb, v)**, **nb** - number of bits to shift, **v** - value which we need to shift. We will basically move our bytes array to the right until we reach our 4th byte.
 - To read a needed value we also need to do a **bit masking**. After shifting, the slot may still contain residual data from other variables, so bit masking <ins>is required to isolate the specific value you are interested in from other data stored within the same slot</ins>. 
 - The bitmask is constructed to match the size of the variable. For example, in our case **0xffff** is a mask for a uint16 (16 bits or 2 bytes), <ins>which ensures that only the last 16 bits of the shifted value are preserved, while the rest are zeroed out</ins>.

```
function readEWithShr() external view returns(uint256 e) {
        assembly {
            let value := sload(E.slot) 
            // value atm = 0x0001000800000000000000000000000600000000000000000000000000000004
            let shifted := shr(mul(E.offset, 8), value) // 'E.offset' is in bytes, but shr(takes num of bits to shift, value). By mul on 8 we receive bits for shifting.
            // after shifting = 0x0000000000000000000000000000000000000000000000000000000000010008
            // equivalent to
            // 0x000000000000000000000000000000000000000000000000000000000000ffff
            e := and(0xffff, shifted) // we masking the last 2 bytes(16 bits) | ff == 11111111(1 byte) | with masking we will receive 8 value back for 'e'
            // after masking: 0x000000000000000000000000000000000000000000000000000000000000ffff
        }
}
```
Sum up the flow: load a packed slot value --> shift right to the variable offset --> do bit masking of the needed bytes.

### Reading from packed slot(returning value less than 256 bits)
Here we do everything like in previous example, but if we returning a size which is less than 256 bit, then we can omit a bit masking, because when assigning a value from a 256-bit storage word to a smaller type (like uint96), the EVM automatically truncates the higher-order bits that don't fit into the smaller type.
```
function readDWithShr() external view returns(uint96 d) {
        assembly {
            let value := sload(D.slot)
            d := shr(mul(D.offset, 8), value) // the EVM automatically truncates the result to 96 bits.
        }
}
```
Sum up the flow: load a packed slot value --> shift right to the variable offset.
### Writing to packed slot
 1. Here again we need to use a bit masking. Below I listed a main rules for bit masking(will be more understandable on practice):
  - Value and 00 = 00 | if you take a value and 'and' it with 00 it will be 00 in result.
  - Value and FF = Value | if you take a value and 'and' it with ff it will be value in result.
  - Value or 00 = Value | if you take a value and 'or' it with 00 it will be Value.
2. Bit masks can be hard coded because variable storage slot and offsets are fixed.
```
// we'll set 10(0x0a in hex) as new E
function writeToE(uint16 newE) external {
    assembly {
        let e := sload(E.slot)
        // e = 0x0001000800000000000000000000000600000000000000000000000000000004
        
        let clearedE := and(e, 0xffff0000ffffffffffffffffffffffffffffffffffffffffffffffffffffffff)
        // mask     = 0xffff0000ffffffffffffffffffffffffffffffffffffffffffffffffffffffff |  with 0000 we masking the number of bytes we want to change and set it to 0, because V and 00 = 00.
        // e        = 0x0001000800000000000000000000000600000000000000000000000000000004 | here start reminding masking rules.
        // clearedE = 0x0001000000000000000000000000000600000000000000000000000000000004 | all values except 8 are preserved.
        
        let shiftedNewE := shl(mul(E.offset, 8), newE) // put new E value to the correct location
        // shiftedNewE = 0x0000000a00000000000000000000000000000000000000000000000000000000
    
        let newVal := or(shiftedNewE, clearedE) // 'or' will preserve the original values and put a new E value.
        // shiftedNewE = 0x0000000a00000000000000000000000000000000000000000000000000000000 | set up new value with shifting left
        // clearedE    = 0x0001000000000000000000000000000600000000000000000000000000000004 | what we cleared before shifting
        // newVal      = 0x0001000a00000000000000000000000600000000000000000000000000000004 | what we receive after or operation
        
        sstore(E.slot, newVal) // store new E value in the packed slot
    }
}
```
Sum up the flow: load a packed slot value --> clear out a variable you would like to change --> shift new value to a correct position --> do 'or' operation to preserve other values and set a new one --> sstore new packed value back to storage slot. 

## Storage Struct
In storage structs we can have a **usual struct** and a **packed struct**. 

### Read Usual Struct
1. Usual struct is stored as a usual 32 byte variable - in different slots. Struct values are stored one by one.
```
contract StructStorage {
    struct UsualStruct {
        // slot 1
        uint256 num1;
        // slot 2
        uint256 num2;
        // slot 3
        address owner;
    }
    
    // create a random struct
    UsualStruct public usualStruct = UsualStruct({num1: 33, num2: 44, owner: 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4});

    function readUsualStruct() public view returns(uint256 x, uint256 y, address owner) {
        assembly {
            x := sload(0)     // load `num1` value from slot 0
            y := sload(1)     // load `num2` value from slot 1
            owner := sload(2) // load `owner` value from slot 2
        }
    } 
}
```

### Read Packed Struct
1. Packed struct is stored in a packed slot as a packed variable - in a single slot. We just shift right for the needed amount of bits, and read the next value from the struct.
```
contract StructStorage {
    // | owner | num2 | num1 |  how value located in bytes array
    // |  128  |  64  |  64  |  bytes
    struct PackedStruct {
        // all in slot 0
        uint32 num1;
        uint32 num2;
        address owner;
    }
    
    // create a random struct
    PackedStruct public packedStruct = PackedStruct({num1: 1, num2: 2, owner: 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4});


    function readPackedStruct() public view returns(uint32 x, uint32 y, address owner) {
        assembly {
            let s := sload(packedStruct.slot)  // loads 32 bytes from slot 0.
            x := s                             // "num1" occupies the first 32 bits (4 bytes) of 's', and these bytes already represent the desired value for "a". No need for shr().
            y := shr(32, s)                    // shifting right 32 bits(4 bytes) to skip the 'num1' value and read the next 32 bits representing 'num2'.
            owner := shr(64, s)                // shifting right 64 bits(8 bytes) to skip the 'num1' and 'num2' values and read the next 160 bits(20 bytes) representing 'owner'.
        }
    } 
}
```

## Storage Fixed Array
The length of fixed-size arrays is not stored in storage. Solidity's compiler doesn't store metadata about fixed-size arrays because their size is predetermined.

### Read from fixed array(32 bytes value)
1. To get the slot where array element is stored - <ins>add slot where array is declared</ins> to <ins>index of the needed element</ins>.
```
contract ReadFixedArr {
    uint256[3] fixedArray;
    
    constructor() {
        fixedArray = [99, 999, 9999];
    }
    
    function getFixedArrayElem(uint256 index) external view returns (uint256 e) {
        assembly {
            e := sload(add(fixedArray.slot, index)) // add slot of the fixed array to element index
        }
    }
}

```

### Read from fixed array(less than 32 bytes value)
1. If each element of a fixed size array is less than 32 bytes, then they will be packed to a single slot.
2. So in this `uin128[5] fixedArray = [7, 8, 9, 10, 11];` - 2 values fit in 1 slot.
```
contract ReadFixedArr {
    uint128[5] fixedArray; // array stored in slot 0
    
    constructor() {
        fixedArray = [7, 8, 9, 10, 11];
    }
    
    function getFixedArrayElem(uint256 index) external view returns (uint128 e) {
        assembly {
            let slot := sload(add(0, div(index, 2)))  //we dividing i/2, because once index increased by 2, we should go to the next slot.
            // Ex, we pass 0, 1 indexes - means slot 1 is passed. When we come to index 2 we already watching at slot 1, becasue 2/2=1
            
            // If the index is odd, extract the right 128 bits by shifting the storage slot 128 bits to the right
            // If the index is even, the value is already in the left 128 bits, so no shifting is needed.
            switch mod(index, 2)
            case 1 { e := shr(128, slot) }
            default { e := slot }
        }
    }
}
```

## Storage Dynamic Data
1. When dealing with non-value types(dynamic arrays, byte arrays, strings) <ins>it is important to understand how EVM handles these types</ins>. They are stored and retrieved in a different way, and here is how:

 - Those values are usually stored in 2 parts: **1st - their length** and **2nd the actual value**.
 - The **length** of the dynamic data is stored at the slot where the variable is defined.
 - The **data** starts from a computed slot derived from the hash of the starting slot.

## Storage Dynamic Array
### Get length of the dynamic array
1. Dynamic array length is stored in the slot where the array is declared.
```
contract ReadDynamicArr {
    uint256[] dynaimicArray;
    
    constructor() {
        dynamicArray = [10, 20, 30];
    }
    
    function getArrayLength() external view returns (uint256 length) {
        assembly {
            length := sload(dynamicArray.slot) // return 3
        }
    }
}

```

### Read from dynamic array(32 bytes value)
Here things become slightly more interesting. It is the same mechanism as for fixed array, but with keccak256().
1. To get the slot where the array element is stored - <ins>add keccak256 hash of the slot where array is declared</ins> to <ins>index of the needed element</ins>. Below will be 2 examples of how to do this.

 - This is a more easier and readable way, but it is not the best one, because we calculate hash in pure Solidity not in Yul.
```
contract ReadDynamicArr {
    uint256[] dynaimicArray;
    
    constructor() {
        dynamicArray = [10, 20, 30];
    }
    
    function getDynamicArrayElem(uint256 index) external view returns (uint256 res) {
        uint256 slot;
        assembly {
            slot := dynaimicArray.slot                      // get `dynaimicArray` slot 
        }
        bytes32 location = keccak256(abi.encode(slot));     // calculate the hash of the slot where array is declared

        assembly {
            res := sload(add(location, index))              // add hash to the index of the element we need
        }
    }
}
```

 - This is a more advanced approach with Memory area usage. In the next section I'll explain how memory works.
 - Briefly, here we basically receiving a free memory pointer, save to free memory our slot value(**this is basically = abi.encode(slot)**), then hash the slot value which we read from the memory in **keccak256()** instruction and in the end just simply add a hashed slot to the index from the function argument.
```
contract ReadDynamicArr {
    uint256[] dynaimicArray;
    
    constructor() {
        dynamicArray = [10, 20, 30];
    }
    
    function getDynamicArrayElem(uint256 index) external view returns (uint256 res) {
        assembly {
            let slot := dynaimicArray.slot               // get `dynaimicArray` slot
            let ptr := mload(0x40)                       // get the free memory pointer
            mstore(ptr, slot)                            // store in free memory our slot value 
            
            let location := keccak256(ptr, 0x20)         // calculate hash of the slot
            res := sload(add(location, index))           // add hash of the slot to element index and receive the element
        }
    }
}
```

### Read from dynamic array(less than 32 bytes value)
Here I am reading a packed value from a dynamic array fully in Yul. In the beginning everything is pretty much the same as with a fixed array(free mem pointer, slot, hash), but after this we need to do a bit of more work to receive a value + remind a bit of masking.
1. Each storage slot is 32 bytes (256 bits), and each uint8 takes 1 byte
2. `let slotOffset := div(index, 32)` - a storage slot containing the index-th element.
3. `let byteOffset := mod(index, 32)` - a  byte position of the index-th element within the slot.
4. `packedData` - reads the storage slot containing the packed uint8 values.
5. `val` - shifts the storage slot to the right, moving to the desired byte position + masks the result to extract only the desired byte (8 bits).
```
contract ReadDynamicArr {
    uint8[] dynaimicArray;
    
    constructor() {
        dynamicArray = [1, 2, 3];
    }
    
    function readArrayWithPackedValue(uint256 index) external view returns (uint8 val) {
            assembly {
                let slot := smallArray.slot
                let ptr := mload(0x40)
                mstore(ptr, slot)
                
                let location := keccak256(ptr, 0x20)
                
                let slotOffset := div(index, 32)                        // The storage slot containing the packed value
                let byteOffset := mod(index, 32)                        // The byte within the slot
    
                let packedData := sload(add(location, slotOffset))      // Load the Packed Slot
                val := and(shr(mul(byteOffset, 8), packedData), 0xff)   // Extract the value      
            }
        }
}
```

## Storage Mapping
1. To get the slot where the mapping value is stored you need to calculate the keccak256 hash of the <ins>mapping key</ins> + <ins>slot where the mapping is declared</ins>.  

### Read from mapping
1. Easy way:
```
contract ReadMapping {
    mapping(uint256 => uint256) public usualMapping;
    
    constructor() {
        usualMapping[10] = 5;
    }
    
    function getMapping(uint256 key) external view returns (uint256 val) {
        uint256 slot;
        assembly {
            slot := usualMapping.slot                                   // get mapping slot
        }

        bytes32 location = keccak256(abi.encode(key, uint256(slot)));   // calculate the hass of the value location(keccak256(key, mapping slot))

        assembly {
            val := sload(location)                                      // read value
        }
    }
}
```

2. Advanced way:
```
contract ReadMapping {
    mapping(uint256 => uint256) public usualMapping;
    
    constructor() {
        usualMapping[10] = 5;
    }
    
    function getValueFromMapping(uint256 key) external view returns(uint256 val) {
        assembly {
            let ptr := mload(0x40)                      // get free memory pointer
            mstore(ptr, key)                            // store key in memory
            mstore(add(ptr, 0x20), usualMapping.slot)   // store mapping slot in memory
            let slot := keccak256(ptr, 0x40)            // calculate hash 
            val := sload(slot)                          // read value
        }   
    }
}
```

## Storage Nested Mapping
1. To get value from nested mapping we follow the next formula: keccak256(2nd key, keccak256(1st key, mapping slot)). Looks a bit complicated, but in simple words:
 - 1st step - we do a usual hash from <ins>mapping 1st key</ins> + <ins>slot where the mapping is declared</ins>.
 - 2nd step we do a hash from <ins>mapping 2nd key</ins> + <ins>hash we calculated in the 1st step</ins>.
 - Done

### Read from nested mapping
1. Easy way:
```
contract ReadNestedMapping {
    mapping(uint256 => mapping(uint256 => uint256)) public nestedMapping;
    
    constructor() {
        nestedMapping[10][77] = 123;
    }
    
    function getValueFromNestedMapping() external view returns (uint256 val) {
        uint256 slot;
        assembly {
            slot := nestedMapping.slot                              // get nested mapping slot
        }

        bytes32 location = keccak256(                               // outer hash keccak256(key 2, inner hash)
            abi.encode(
                uint256(77),                                        
                keccak256(abi.encode(uint256(10), uint256(slot)))   // inner hash keccak256(key 1, mapping slot)
            )
        );
        assembly {
            val := sload(location)                                  // read value
        }
    }
}
```

2. Advanced way:
```
contract ReadNestedMapping {
    mapping(uint256 => mapping(uint256 => uint256)) public nestedMapping;
    
    constructor() {
        nestedMapping[10][77] = 123;
    }
    
    function getValueFromNestedMapping(uint256 key_1, uint256 key_2) external view returns(uint256 val) {
        assembly {
            let ptr := mload(0x40)                           // set free mem pointer
            mstore(ptr, key_1)                               // store in memory 1st key
            mstore(add(ptr, 0x20), nestedMapping.slot).      // store in memory slot of the nested mapping
            
            let hash_1 := keccak256(ptr, 0x40)               // this is a calculation of the 1st hash which is inside the main hash | calculating hash from key_1 + nested mapping slot
             
            mstore(add(ptr, 0x40), key_2)                    // store in memory 2nd key
            mstore(add(ptr, 0x60), hash_1)                   // store in memory 1st hash
            
            let location := keccak256(add(ptr, 0x40), 0x40)  // calculate final location of the slot from which we will return a value | calculating hash from key_2 + 1st hash
            
            val := sload(location)                           // read value
        }   
    }
}
```

## Storage Mapping To Dynamic Array
1. If we will just hash the key + mapping slot, then we'll receive just a length of the array/list.
2. So the steps we need to do to receive a slot of the value:
 - We need to hash the key and mapping slot.
 - Hash the previously produced hash. This will give us a location of the element in the array.
 - Add a hash to index you of the element you need.
 - Done
 - 
### Read from mapping to list
1. Easy way:
```
contract ReadMappingToList {
    mapping(address => uint256[]) public addressToList;
    
    constructor() {
        addressToList[address(this)] = [42, 333, 444, 555];
    }
    
    function getValueFromMappingAddrToList(uint256 index)
        external
        view
        returns (uint256 val)
    {
        uint256 slot;
        assembly {
            slot := addressToList.slot          // get slot of the mapping
        }

        bytes32 location = keccak256(           // outer keccak256(inner hash)
            abi.encode(
                keccak256(                      // inner keccak256(key, mapping slot)
                    abi.encode(
                        address(this),
                        uint256(slot)
                    )
                )
            )
        );
        assembly {
            val := sload(add(location, index))  // read value by adding calculated location to index of the array element we need
        }
    }
}

```
2. Advanced way:
```
contract ReadMappingToList {
    mapping(address => uint256[]) public addressToList;
    
    constructor() {
        addressToList[address(this)] = [42, 333, 444, 555];
    }
    
    function getValueFromMappingAddrToList(address key, uint256 index) external view returns(uint256 val) {
        assembly {
            let ptr := mload(0x40)                             // get free mem pointer
            mstore(ptr, key)                                   // store in memory key
            mstore(add(ptr, 0x20), addressToList.slot)         // store in memory slot of the mapping
            
            let hash_1 := keccak256(ptr, 0x40)                 // if we will have only this hash(this hash-location will return the dynamic array length and that's it)
            
            mstore(add(ptr, 0x40), hash_1)                     // store in memory 1st hash
            
            let location := keccak256(add(ptr, 0x40), 0x20)    // calculate final location of the element we need to extract from the array which is value in mapping

            val := sload(add(location, index))                 // read value like previously in array
        }
    }
}
```

## How write to arrays and mappings?
You already know how to do this - just calculate the slot where you need to store your value, and instead of `sload()` opcode use `sstore()`.
1. What to do the next: write value to mapping and then check that value was written successfully(check read function in previous mapping sections).

Mapping write example:
```
contract ReadMapping {
    mapping(uint256 => uint256) public usualMapping;
    
    constructor() {
        usualMapping[10] = 5;                                               // atm mapping has only 5 at key 10
    }
    
    function storeValueToMapping(uint256 key, uint256 newVal) external {
        assembly {
            let ptr := mload(0x40)                                          // get free memory pointer
            mstore(ptr, key)                                                // store new key in memory 
            mstore(add(ptr, 0x20), usualMapping.slot)                       // store mapping slot in memory
            let slot := keccak256(ptr, 0x40)                                // calculate the hash of the slot where to store new value
            sstore(slot, newVal)                                            // write new value to previously calculated slot
        }
    }
}
```

## Common Storage Mistakes

### Overwriting data in the slot
1. Solidity automatically assigns storage slots to state variables when you declare a new variable, but when writing to any slot in Yul, you should always keep in mind a layout of your contract storage when writing to an arbitrary slot. It is quite easy to overwrite the original data in the slot. \
Example when you declared first a function in Yul, and later decided to add some storage variable and forgot about contract storage layout:
 - In the code below we have a contract which calculates the sum of 2 numbers and stores the result in the `result` variable. All good atm.
```
contract UsualContract {
    uint256 public a = 22;  // slot 0 
    uint256 public b = 30;  // slot 1
    uint256 public result;  // slot 2

    function sum() external {
        assembly {
            let A := sload(a.slot)  
            let B := sload(b.slot)
            sstore(2, add(A, B))    // store sum result in `result` variable
        }
    }
}
```
 - For some reason you decided to add a new variable `totalOperationsDone` to calculate the count of calls of the `sum()` function. You add the `totalOperationsDone` before the `results` variable. Plus you forgot to change the slot number where you will write a result of your operation. As a result you will write a result of your operation in `totalOperationsDone` and this will be a critical bug.
 ```
 contract BrokenContract {
    uint256 public a = 22;               // slot 0
    uint256 public b = 30;               // slot 1
    uint256 public totalOperationsDone;  // slot 2 | Add variable to slot 2 and move result to slot 3
    uint256 public result;               // slot 3

    function sum() external {
        assembly {
            let A := sload(a.slot)  
            let B := sload(b.slot)
            sstore(2, add(A, B))         // due to mistake in storage layout, the sum will be stored in `totalOperationsDone`, not in the `result`
        }
        totalOperationsDone += 1;        // expects that `totalOperationsDone` will increase after every `sum()` call, but on practice it will be overwritten each time to 53
    }
}
 ```
2. Conclusion: 
 - Always keep in mind your storage layout.
 - I would recommend using the `.slot` method when possible, to avoid writing to incorrect slots.

### Writing incorrect data type in the slot
1. In Solidity you at least have a data type checks and you won't be able to write to `uint` variable a `string`, but in Yul you can. In Yul you don't have data type checks and incorrect assignment can lead to critical bugs.\
Example, when accidentally assign a string instead of uint:
```
contract BrokenStorage2 {
    uint256 public a = 22;          // after `sum()` func is executed, `a` is storing a broken value(22252023922882658565525207896905291916629407203809125848297821787814667223040)
    uint256 public b = 30;

    function sum() external returns(uint256 res) {
        assembly {
            let A := sload(a.slot)
            let B := sload(b.slot)
            
            sstore(a.slot, "123")   // assigning a `string` instead of `uint`
            
            res := add(A, B)        // first call will be okay, but all other calls to this function will return a broken result
        }
    }
}
```
2. Conclusion: 
 - Always keep in mind that in Yul you are responsible for everything(including data types), because the number of helpers are minimal. 
 - Always think about what you assign to a storage variable.