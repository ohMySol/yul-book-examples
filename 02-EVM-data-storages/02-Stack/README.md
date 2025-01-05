# Stack
This section will explain 1st storage location in EVM - **Stack**. \
If you are not familiar with the stack data structure, then I recommend you take a look at [this video](https://www.youtube.com/watch?v=FNZ5o9S9prU&t=191s).
## Table of Content 
* [Quick refresh about EVM](#quick-refresh-about-evm)
* [What is stack-based architecture?](#what-is-stack-based-architecture?)
* [Why does EVM use a stack-based architecture?](#why-does-evm-use-a-stack-based-architecture?)
* [EVM Stack data storage](#evm-stack-data-storage)

## Quick refresh about EVM
1.  Such blockchains as Ethereum, Polygon, Binance Chain, Optimism etc use EVM(Ethereum Virtual Machine) as the underlying environment to execute the transaction and run the smart contracts.
 
EVM Architecture: ⬇️ \
![image alt](https://github.com/ohMySol/yul-book-examples/blob/a1ae00fa8a54a9f8d84a194d0257b38f00e3d77f/EVM%20Architecture.jpg)
 
2. EVM is a stack-based machine because it operates on the stack data structure and keeps inside this data structure values and executes actions.

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
