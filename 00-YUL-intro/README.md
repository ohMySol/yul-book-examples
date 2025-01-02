# YUL introduction
This section will introduce you in [YUL](https://docs.soliditylang.org/en/latest/yul.html) - a low level language for smart contracts which acts as a bridge between high-level languages (like Solidity) and EVM bytecode.
## Table of Content 
* [What is Yul?](#what-is-yul?)
* [Why do we use Yul?](#why-do-we-use-yul?)
* [Difference between Yul and Assembly](#difference-between-yul-and-assembly)
* [Yul Syntax](#yul-syntax)
* [Basic Yul code example](#basic-yul-code-example)
* [Yul Data Types](#yul-data-types)
* [Conclusion](#conclusion)
* [Useful Tools](#useful-tools)
## What is Yul?
 - Yul - is a a low level programming language which allows you keep more granular control over your smart contract execution. You can write an entire smart contract in Yul, or you can use "inline assembly" mode and write a Yul code parts inside your smart contract.
 - **Remember in Yul we have only 1 data type: bytes32**
## Why do we use Yul?
 - Mainly for performance, the compiler is still not great at optimising code.  [Check this blog](https://makemake.site/post/solidity-considered-harmful).
![image alt](https://github.com/ohMySol/yul-book-examples/blob/757cfa2b1761d13362df86cb63f36a8e160bb176/SolidityVsYul.jpg)
## Difference between Yul and Assembly
 - **Assembly** is a low-level language, really close to what your computer can understand. It’s a sequence of instructions for your computer to execute (here: the EVM).
 - **Yul** is just the name of the (almost) assembly for the EVM. Almost, because it’s a bit easier to write than pure assembly and it has the concept of variables, functions, for-loops, if statements, etc whereas pure assembly doesn’t.
## Yul Syntax
```
contract Example {
    function foo() public {
        assembly {
            // Yul code starts here
        }
    }
}
```
 - To declare a variable we use a `let` keyword
```
contract Example {
    function foo() public {
        assembly {
            let bar := 1
        }
    }
}
```
## Basic Yul code example
 - Below is a simple **readNumStruct** function which is using an inline assembly(Yul). You can see that main part of the code is written in usual Solidity except the readNumStruct body.
```
contract StructStorage {
   struct Num {
        // slot 0
        uint256 num1;
        // slot 1
        uint256 num2;
        // slot 2
        address owner;
    }
    
    Num public num = Num({num1: 33, num2: 44, owner: 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4});


    // function return values from the above struct
    function readNumStruct() public view returns(uint256 x, uint256 y, addresses  owner) {
        assembly {
            x := sload(0)
            y := sload(1)
            owner := sload(2)
        }
    }
}
```
## Yul Data Types
 - **As I am already said, Yul has only 1 type - bytes 32**, and compiler will automatically do a conversions as needed. This means that in the Yul language, every value is ultimately treated as a 32-byte value.
 - You can use also the next literals data types: 
 1. Integers written in either **decimal** or **hexadecimal** notation.
 ```
 let x := 42    // Decimal literal for the integer 42
 let y := 0x2A  // Hexadecimal literal for the same number (42 in decimal)
 ```
2. String literals represented in **ASCII** or **Unicode** text values. Be carefull, **strings are limited to a maximum of 32 bytes in Yul because all data is represented as bytes32**. \
 Plain text characters: `"abc"` \
 Hexadecimal escapes: `\xNN` (e.g., `"\x41\x42\x43"` represents "ABC"). \
 Unicode escapes: `\uNNNN` where N are hexadecimal digits. (e.g., `"\u0041\u0042\u0043"`  represents "ABC").
3. Hexadecimal string literals represent as a fixed binary data in hexadecimal format. This data is prefixed with `hex"..."`.
```
let hexString := hex"616263"  // Represents "abc" in ASCII
```
 - The compiler automatically converts all literals into the bytes32 representation.
 ```
 42 (integer literal) is automatically stored as
 bytes32(0x000000000000000000000000000000000000000000000000000000000000002A).
 ```
or
```
"abc" (string literal) is stored as 
bytes32(0x6162630000000000000000000000000000000000000000000000000000000000)
```
## Conclusion
My advice is to be ready to count in hexadecimal 😄