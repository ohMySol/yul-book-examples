# Stack
This section will explain 1st storage location in EVM - **Stack**. \
If you are not familiar with the stack data structure, then I recommend you take a look at [this video](https://www.youtube.com/watch?v=FNZ5o9S9prU&t=191s).
## Table of Content 
* [Quick refresh about EVM](#quick-refresh-about-evm)
* [What is stack-based architecture?](#what-is-stack-based-architecture?)
* [Why does EVM use a stack-based architecture?](#why-does-evm-use-a-stack-based-architecture?)
* [EVM Stack data storage](#evm-stack-data-storage)
* [EVM Stack basic operations](#evm-stack-basic-operations)

## Quick refresh about EVM
1.  Such blockchains as Ethereum, Polygon, Binance Chain, Optimism etc use EVM(Ethereum Virtual Machine) as the underlying environment to execute the transaction and run the smart contracts.
 
EVM Architecture: ⬇️ \
![image alt](https://github.com/ohMySol/yul-book-examples/blob/a1ae00fa8a54a9f8d84a194d0257b38f00e3d77f/EVM%20Architecture.jpg)
 
2. EVM is a stack-based machine because it operates on the stack data structure and keeps inside this data structure values and executes actions. This means that <ins>“the stack” is the primary data structure that programs work with when executing EVM opcodes</ins> .

3. The EVM has its own set of instructions known as [opcodes](https://www.evm.codes/) that it uses to execute tasks such as reading and writing to storage, calling other contracts, and performing mathematical operations.

## What is stack-based architecture?
1. It is a sort of computer architecture that stores values and performs actions using a stack data structure. The stack operates on a **LIFO** basis(last-in, first-out) which means that the most recently inserted item is stored at the top of the stack and is the first item removed.

EVM Stack example: ⬇️ \
![image alt](https://github.com/ohMySol/yul-book-examples/blob/e77fbc1827c70c079378234259da3509c8ad1e92/Stack%20data%20structure.jpg)

2. Program execution is controlled by **a stack pointer which points to the top of the stack**, and this pointer is responsible where the next value or instruction will be saved or retrieved on the stack.

3. When your smart contract is running, it adds instructions(opcodes) and values to the stack and then EVM executes these instructions.

## Why does EVM use a stack-based architecture?
As I understand EVM, as a virtual machine, should do a lot of operations in a short period of time, so we need smth quick and efficient. Stack with its LIFO data structure is a great solution for EVM, because it allows for easy and quick data and instruction handling.

## EVM Stack data storage
### EVM Stack Architecture
1. The EVM stack is **a list of 256 bit**(mainly to facilitate native hashing and elliptic curve operations) words used to store smart contract instruction inputs and outputs.The stack currently has a **maximum capacity of 1024 values**. 

EVM Stack diagram: ⬇️ \
![image alt](https://github.com/ohMySol/yul-book-examples/blob/6f0af6ba0bb527326ff6298b5798f3f39c8feb3c/EVM%20Stack.jpg)

### How EVM Stack works?
1. Storage is where the variables are permanently stored on the blockchain. If you want to manipulate the data in storage you copy it to the memory. Then, all memory code is executed on stack. \
When you define the local variable it is stored in memory and then pushed to the stack for execution.

How EVM Stack works: ⬇️ \
![image alt](https://github.com/ohMySol/yul-book-examples/blob/909faac4c7ff76dbde335e031567029a99a6712d/Ho%20EVM%20Stack%20works.png)

2. Code example(the stack adding 2 and 4):
```
PUSH1 2 // PUSH1 is an opcode that pushes the next 1-byte value (here 2) onto the stack 
PUSH1 4 // Another PUSH1 opcode pushes the next 1-byte value (here 4) onto the stack.
ADD // Add the top two values on the stack
```

```
PUSH1 2 
|__2__|

PUSH1 4
|__4__|
|__2__|

ADD
|__6__|
```
Execution flow:
1. The `ADD` opcode pops the top two values from the stack, adds them together, and pushes the result back onto the stack.
 - Pop the top value 4
 - Pop the top value 2
 - Perform the addition: 4 + 2 = 6
2. Push the result 6 back onto the stack.\
`|__6__| ` is not the top of the stack.

## EVM Stack basic operations
EVM Stack has the next base operations:
 - Pushing a value to the stack.
 - Consuming stack items by running opcodes that require arguments.
 - Swap stack items
 - Duplicate stack item.

### Pushing a value to the stack
Below is an example when a `PUSH1` opcode will be used. The EVM has PUSH opcodes ranging from `PUSH1` up to `PUSH32`.
```
contract PushExample {
    uint256 num;

    function updateNum() external {
        num += 10;
    }
}
```
Explanation:
1. In the above example contract operates with hardcoded value 10 and add it to `num`. 10 must be push onto the stack to be used in the addition operation. So any time you have a hardcoded value like this – whether a number, boolean, address, string, and so on – this will involve a push opcode.
2. In the compiled bytecode of this contract you can find `600a` value in the runtime bytecode section. `60` is an opcode for `PUSH1` instruction, and `0a` - hex representation of 10.
3. So the above instruction - `PUSH1 0x0a`, pushes `0x0a`(10) as a 32-byte value onto the stack. **Reminder** - EVM operates on a 256 bit(32 byte) value size, and that's why we pushing 1 byte value, but stack receiving `0x000000000000000000000000000000000000000000000000000000000000000a` value.

### Consuming stack items by running opcodes that require arguments
The stack’s main purpose is to pass arguments to opcodes. For example `ADD` opcode pops from the stack 2 items --> add them --> and pushes back to the stack the result.
```
contract Example {
    uint256 a = 3;
    uint256 b = 5;
    uint256 c = 6;

    function addNums() external {
        b + c;
    }
}
```
1. The above function is equivalent to this stack with opcodes:
```
PUSH1 0x03
PUSH1 0x05
PUSH1 0x06
ADD
```
Guess how will look like the stack after this function execution? \
Answer: `0x03` on the bottom of the stack and `0x0b`(11 - result of running `ADD`) on the top of the stack.

### Swap stack items
Usually opcodes are working with the topmost values in the stack, and quite often you need to work with values that are in different parts of the stack. So to operate on the right data EVM use `SWAP` opcode. \
The `SWAP` opcodes range from `SWAP1` to `SWAP16`. **This means that you cannot reach back farther than 16 stack items**.

1. Example:
 - The `SWAP1` opcode exchanges the top value on the stack with the second value. For instance:
```
Before SWAP1:
Stack: | A |
       | B |
```

```
After SWAP1:
Stack: | B |
       | A |
```

2. Opcodes example:
 - Assume that we need to add `3 + 5`, but without `SWAP1` we will add 5 + `OLD_ADD_RESULT`
```
PUSH1 0x05              // Stack: | 0x05           |
PUSH1 OLD_ADD_RESULT    // Stack: | OLD_ADD_RESULT | 5 |
PUSH1 0x03              // Stack: | 0x03 | OLD_ADD_RESULT | 5 |

ADD                     // Try to add them (pops the top two values)
 
```

 - In the below example with the help of `SWAP1` we swapped `0x03` with `OLD_ADD_RESULT` and successfully add 3 to 5 and receive back a result to stack `0x08`.
```
PUSH1 0x05            // Push 0x05 onto the stack
PUSH1 OLD_ADD_RESULT  // Push OLD_ADD_RESULT onto the stack
PUSH1 0x03            // Push 0x03 onto the stack

SWAP1                 // Swap 0x03 with OLD_ADD_RESULT

ADD                   // Add 0x03 and 0x05, push result
```

 - Final stack will look like this:
```
| 0x08           |  
| OLD_ADD_RESULT |

```

### Duplicate stack item
⚠️ When working with a stack-based machine, you don’t have access to variables So when you want to use a value more than once, and avoid redundant computation while doing so, a good option is to duplicate that value using one of the `DUP` opcodes.