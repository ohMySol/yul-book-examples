# YUL syntax overview.
This section will show a basic synax, that you'll you quite often. Here you will see variables declaration, control structures, loops, functions in Yul. **Please do not be afraid if you don't understand some opcodes or logic, I'll explain everything in the next sections :)** \
All information I mostly took from official [Solidity documentation](https://docs.soliditylang.org/en/latest/yul.html). Check it for more examples and information.
## Table of Content 
* [Quick syntax overview](#quick-syntax-overview)
* [Blocks](#blocks)
* [Variable Declarations](#variable-declarations)
* [Accessing Variables](#accessing-variables)
* [Control Structures](#control-structures)
* [Loops](#loops)
* [Functions](#functions)
* [Functions Visibility](#functions-visibility)
## Quick syntax overview
 Yul can contain: 
 - literals, i.e. 0x123, 42 or "abc" (strings up to 32 characters)
 - calls to builtin functions, e.g. add(1, mload(0))
 - variable declarations, e.g. let x := 7, let x := add(y, 3) or let x (initial value of 0 is assigned)
 - identifiers (variables), e.g. add(3, x)
 - assignments, e.g. x := add(y, 3)
 - blocks where local variables are scoped inside, e.g.
 ```
 { let x := 3 { let y := add(x, 1) } } 
 ```
 - if statements, e.g.
 ```
 if lt(a, b) { sstore(0, 1) }
 ```
 - switch statements, e.g.
 ```
 switch mload(0) case 0 { revert() } default { mstore(0, 1) }

 ```
 - for loops, e.g.
 ```
 for { let i := 1 } lt(i, 10) { add(i, 1) } { mstore(0, 10) }
 ```
 - function definitions, e.g.
 ```
 function f(a, b) -> c { c := add(a, b) }
 ```
## Blocks
Blocks of code are delimited by **{}** and the variable scope is defined by the block.

Block example:
```
function addition(uint256 x, uint256 y) public pure returns (uint256) {
     assembly {
        let result := add(x, y)   // x + y
        mstore(0x80, result)      // store result in memory
        return(0x80, 0x20)        // return 32 bytes from memory 
      }
 }
```
## Variable Declarations
We already know how to assign a value to a variable: `let y := 4`. But what happens under the hood? \
Under the hood a `let` keyword do the next:
 - Creates a new stack slot
 - The new slot is reserved for the variable.
 - The slot is then automatically removed again when the end of the block is reached.
 
Since **variables are stored on the stack, they do not directly influence memory or storage**, but they can be used as pointers to memory or storage locations in the built-in functions mstore, mload, sstore and sload

## Accessing Variables
We can access variables that are defined outside the block, if they are local to a Solidity function.

Accessing example:
```
function assembly_local_var_access() public pure {
    uint b = 5;               // local var-l
    assembly {                // use var-s defined inside an assembly block
        let x := add(2, 3)  
        let y := 10  
        z := add(x, y)
    }
    assembly {               // use var-l defined outside an assembly block
        let x := add(2, 3)
        let y := mul(x, b)   // var-l 'b' is a local variable from this function
    }
}
```
## Control Structures
### If Statements 
Please note, there is no **else** in Yul. If you need multiple choices, then you can use the **switch** statement which I describe further below

If example:
```
assembly {
    if lt(a, b) { sstore(0, 1) } // also note that single line statements still requires braces
}
```

### Switch Statements
Note:
 - Control does not flow from one case to the next
 - If all possible values of the expression type are covered, a default case is not allowed.

Switch example:
```
assembly {
    let x := 0
    switch calldataload(4)  // function selector
    case 0 {
        x := calldataload(0x24) // load 32 bytes from 0x24
    }
    default {
        x := calldataload(0x44)
    }
    sstore(0, div(x, 2))
}
```
## Loops
### For Loop
In Yul we have a for loop which consists from 4 parts:
 - Initialisation part
 - Condition part 
 - Post-iteration part
 - Body

**Note**, if the initializing part declares any variables at the top level, the scope of these variables extends to all other parts of the loop. \
Also you can use `break` and `continue` statements in the body to exit the loop or skip to the post-part.

For loop example:
```
function for_loop_assembly(uint256 n, uint256 value) public pure returns (uint256) {
    assembly {
       for { let i := 0 } lt(i, n) { i := add(i, 1) } { 
           result := mul(2, value) // we multiply 2 by `value`
       }
           
       mstore(0x0, result) // store final result in the memory scratch space 
       return(0x0, 0x20).  // return 32 bytes from memory scratch space
    }
         
}
```
### While Loop
Yul doesn't have a while loop, but we can modify a for loop to behave as a while loop 😁

While loop example:
```
assembly {
    let x := 0
    let i := 0
    for { } lt(i, 0x100) { } {     // while(i < 0x100)
        x := add(x, mload(i))
        i := add(i, 0x20)
    }
}
```
## Functions
An Yul function definition has the `function` keyword, a name, two parentheses () and a set of curly braces { ... }. It can also declare parameters. Their type does not need to be specified as we would in Solidity functions.
 - If you call a function that returns multiple values, you have to assign them to multiple variables using `a, b := f(x)` or `let a, b := f(x)`.
 - Return values can be defined by using `->`

Function example:
```
assembly {
    function my_assembly_function(param1, param2) -> my_result { //my_result - is a return value
        
        // param2 - (4 * param1)
        my_result := sub(param2, mul(4, param1))
    
    }
    let some_value = my_assembly_function(4, 9) // calling assembly function inside assembly block
}
```

## Functions Visibility
We do not have the public / internal / private idea that we have in Solidity. Assembly functions are no part of the external interface of a contract. **Functions are only visible in the block that they are defined in**.