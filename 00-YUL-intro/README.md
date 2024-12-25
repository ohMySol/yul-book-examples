# YUL introduction.
This section will introduce you in YUL - a low level language for smart contracts.
## Table of Content 
* [Intro to Yul](#intro-to-yul)
* [Why do we use Yul?](#why-do-we-use-yul-?)
* [Yul != assembly](#yul-!=-assembly)
* [Basic code example](#basic-code-example)
## Intro to Yul
1. Yul - is a a low level programming language which allows you keep more granular control over your smart contract execution. You can write an entire smart contract in Yul, or you can use a "inline assembly" mode and write a Yul code parts inside your smart contract.
## Why do we use Yul?
1. Mainly for performance, the compiler is stil not great at optimising code.  [Check this blog](https://makemake.site/post/solidity-considered-harmful).
![image alt](https://github.com/ohMySol/yul-book-examples/blob/757cfa2b1761d13362df86cb63f36a8e160bb176/SolidityVsYul.jpg)