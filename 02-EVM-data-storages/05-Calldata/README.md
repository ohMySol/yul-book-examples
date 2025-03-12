# Calldata
This section will explain the **Calldata** memory area in the EVM.

## Table of Content
* [Calldata Overview](#calldata-overview)
* [Calldata Layout](#calldata-layout)
* [Calldata vs Memory](#calldata-vs-memory)
* [How Calldata Works?](#how-calldata-works)
* [Calldata Types](#calldata-types)
* [External Calls](#external-calls)
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

## Extenal Calls
Below is several examples of extenal calls done with the help of inline assembly.
This is the contract which I will try to call:
```
contract OtherContract {
    // "0c55699c": "x()"
    uint256 public x;

    // "71e5ee5f": "arr(uint256)"
    uint256[] public arr;

    // "9a884bde": ""get21()"
    function get21() external pure returns (uint256) {
        return 21;
    }

    // "73712595": "revertWith999()"
    function revertWith999() external pure returns (uint256) {
        assembly {
            mstore(0x00, 999)
            revert(0x00, 0x20)
        }
    }

    // "196e6d84": "multiply(uint128,uint16)",
    function multiply(uint128 _x, uint16 _y) external pure returns (uint256) {
        return _x * _y;
    }

    // "4018d9aa": "setX(uint256)"
    function setX(uint256 _x) external {
        x = _x;
    }

    // "7c70b4db": "variableReturnLength(uint256)",
    function variableReturnLength(uint256 len)
        external
        pure
        returns (bytes memory)
    {
        bytes memory ret = new bytes(len);
        for (uint256 i = 0; i < ret.length; i++) {
            ret[i] = 0xab;
        }
        return ret;
    }
}

```

1. External call to a view function:
```
function externalViewCallNoArgs(address _a) external view returns (uint256) {
    assembly {
        mstore(0x00, 0x9a884bde)    // store selector of the function we want to call
        // 000000000000000000000000000000000000000000000000000000009a884bde
        //                                                         |       |
        //                                                         28      32
        let res := staticcall(
            gas(),                  // How much gas we want to pass to function we want to call. In our case we pass all the remaining gas, that we have after execution of the `externalViewCallNoArgs` func. But we also can hardcode value to our own one.
            _a,                     // Address of the contract we are calling.
            28,                     // Argument offset in bytes, or in simple words, where the data starts including selector.
            32,                     // Argument size in bytes.
            0x00,                   // Memory area where to start write a return value from external call
            0x20                    // Size or the returned data.
        )
        
        if iszero(res) {            // Check if the received data is not 0
                revert (0, 0)       
        }
        
        return(0x00, 0x20)          // Return back the results from external call
    }
}
```

2. Get value through external revert.
Yes it is possible to return a value through a function which reverts.
```
function getValueThroughRevert(address _a) external view returns (uint256) {
        assembly {
            mstore(0x00, 0x73712595)    // Store in memory function selector
            pop(                        // Do a staticcall and remove a returned result from the stack(means return 0 from the stack)
                staticcall(
                    gas(), 
                    _a, 
                    28, 
                    32, 
                    0x00, 
                    0x20
                )
            ) 
            return(0x00, 0x20)          // Return data received from revert
        }
    }
```

3. Call external multiply function.
```
    function externalCallMultiply(address _a) external view returns (uint256 result) {
    assembly {
        let ptr := mload(0x40)
        let oldPtr := ptr
        mstore(ptr, 0x196e6d84)
        mstore(add(ptr, 0x20), 3)
        mstore(add(ptr, 0x40), 11)
        mstore(0x40, add(oldPtr, 0x60))
        //  00000000000000000000000000000000000000000000000000000000196e6d84
        //                                                          |
        //                                                          add(oldPtr, 28)
        //  0000000000000000000000000000000000000000000000000000000000000003
        //  000000000000000000000000000000000000000000000000000000000000000b
        //                                                                 |
        //                                                                 0x40 pointer
        
        let res := staticcall(
            gas(),
            _a,
            add(oldPtr, 28),    // The first 4 bytes of the calldata (function selector) need to be extracted starting from the 28th byte within the memory block.
            mload(0x40),        // The updated free memory pointer tells EVM the total length of calldata, load memory location boundary to let EVM know where to search for args, so we starting from 28th byte of the 1st 32 bytes slot, and finish where 0x40 will point to
            0x00,
            0x20
        )
        
        if iszero(res) {            // Check if the received data is not 0
                revert (0, 0)       
        }
        
        return(0x00, 0x20)  
    }
}
```

4. Call external state changing function.
```
function externalStateChangingCall(address _a) external {
    assembly {
        mstore(0x00, 0x4018d9aa)
        mstore(0x20, 228)
        // memory now looks like this
        // 0x000000000000000000000000000000000000000000000000000000004018d9aa...
        // 00000000000000000000000000000000000000000000000000000000000000e4
        
        let res := call(
            gas(),
            _a,
            0,      // callvalue() - amount of Ether we forwarding to the recipient in this transaction. If func is not payable it should be 0.
            28,
            0x40,
            0,
            0
        )
        
        if iszero(res) {
            return(0,0)
        }
    }
}
```

5. Call external function with unknown return data size.
```
    function unknownReturnSize(address _a, uint256 _length) external view returns (bytes memory) {
        assembly {
            mstore(0x00, 0x7c70b4db)
            mstore(0x20, _length)
            
            let success := staticcall(
                gas(),
                _a,
                28,
                0x40,
                0, // |
                0  // |- returnOffset and returnSize are 0, because we don't know the return data size 
            )
            
            if iszero(success) {
                return(0, 0)
            }
            
            returndatacopy(
                0,                      // copy to memory, starting at memory slot 0
                0,                      // copy a return data from zero all the way up until the total data size
                returndatasize()        // return data size(return the length of the returned data from the external call)
            )
            
            return(0, returndatasize()) // return data starting from memory position 0, and data size we get from `returndatasize()`
        }
    }
```