---
title: "Packing bits for multiplayer games"
layout: post
date: 2024-11-5 10:48 PM
image: /assets/images/markdown.jpg
headerImage: false
tag:
- gamedev
category: blog
author: abdullah
description: My humble approach to optimizing packets for my multiplayer game engine
---

## Introduction
In this article, We are going to explore my humble approach to optimize network bandwidth for my game engine and how the engine is structured overall..

- [Engine structure](#engine-structure)
- [Why bit packing ?](#why-bit-packing-)
- [Appraoches used to optimize networking](#appraoches-used-to-optimize-networking)
- [Writing a bit packer](#writing-a-bit-packer)

## Engine structure
<span class="structure">
The engine is structured around the concept of client/server, Where the server and client are separate programs.
The server has the absolute control on the world state, While the clients only receive the state updates from the server, each state update sent to clients is called "Snapshots" and they are being sent at the chosen tick rate of the server, default is 60 ticks per second</span>

## Appraoches used to optimize networking
I mainly use 2 approaches to optimize the networking for my engine, ***Bit packing*** and ***Delta compression***

In this article, We will be diving into bit packing, And I will explain delta compression in the next article

## Why bit packing ?
Let's not think about networking for a second, And go back to the basics of how computers work.

Let's assume we want to store the number "69" in our computer RAM.
Computers doesn't really understand numbers like we do, Instead, Computers understands it in binary format.
So the number "69" could look like this in your computer memory

```c++ 
1000101
```

But wait... it's not as simple as that !

the number "69" as our example above, is represented as
 ```c++
  1000101
  ``` 
is 7 bits, count the 0's and 1's and you will find that they are actually 7.

modern computers doesn't store numbers with packed bits like this. instead, We pick out a "size" for storing our numbers, Let's consider this example in c++ :

```c++ 
int32_t my_32_bit_number = 69 // will store the number "69" in a 32 bit signed integer
int16_t my_16_bit_number = 69 // will store the number "69" in a 16 bit signed integer
```

so both numbers are ***technically*** 69, But they have different memory representation like the following:

```c++
my_32_bit_number = 00000000 00000000 00000000 01000101
my_16_bit_number = 00000000 01000101
```

as you can see, to represent the number "69" in a 32 bit integer, we are wasting 25 bits, and if we are representing it in 16 bits, we are wasting 9 bits.

Now you can probably tell why bit packing is important for network applications, Each wasted bit counts so we need to do something about it


## Writing a bit packer
TODO

