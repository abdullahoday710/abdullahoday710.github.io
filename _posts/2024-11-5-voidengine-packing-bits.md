---
title: "Packing bits for multiplayer games"
layout: post
date: 2025-2-2 10:48 PM
# image: /assets/images/markdown.jpg
headerImage: false
tag:
- gamedev
category: blog
author: abdullah
description: My humble approach to optimizing packets for my multiplayer game engine
---

## Introduction
In this article, We are going to explore my humble approach to optimize network bandwidth for my game engine and how the engine is structured overall..

- <a href="#engine-structure" style="color: white;">Engine structure</a>
- <a href="#appraoches-used-to-optimize-networking" style="color: white;">Appraoches used to optimize networking</a>
- <a href="#why-bit-packing-" style="color: white;">Why bit packing ?</a>
- <a href="#the-basic-idea-of-bit-packing" style="color: white;">The basic idea of bit packing</a>
- <a href="#writing-a-bit-packer" style="color: white;">Writing a bit packer</a>
- <a href="#using-it-in-action" style="color: white;">Using it in action</a>

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
As you can see, To represent the number "69" in a 32 bit integer, We are wasting 25 bits, And if we are representing it in 16 bits, We are wasting 9 bits.

Another example is booleans, A boolean can either have ***True*** or ***False***, So you might assume that in binary it's represented as 0 or 1, And you are correct.
However, In modern computers, The smallest amount of data you can work with is a ***Byte*** so to represent a boolean, You will have to waste 7 bits.

Here is the example in c++ again...

```c++
bool my_boolean = true;

// so my_boolean will look like this in memory :
my_boolean = 00000001
```

Now you can probably tell why bit packing is important for network applications, Each wasted bit counts so we need to do something about it


## The basic idea of bit packing
Now enough talking about theory and let's get to actual coding.

As I mentioned above, The smallest amount of data you can work with in modern computers is a ***Byte***, So to write a bit packer, We will need a ***Byte buffer*** natually, And that ***Byte buffer*** will have our packed bits.

Let's take a simple example, I mentioned above that to represent 1 boolean we need a single byte of memory, But technically, We can have 8 booleans in one byte, Right ?

We can do it by using bitwise operators.

```c++
// Let's assume we have these booleans
bool b1 = false;
bool b2 = true;
bool b3 = true;
bool b4 = true;
bool b5 = false;
bool b6 = true;
bool b7 = false;
bool b8 = false;
```
to pack these booleans in a single ***Byte*** the byte should look like this at the end :
  ```c++
    0 | 1 | 1 | 1 | 0 | 1 | 0 | 0
    b1  b2  b3  b4  b5  b6  b7  b8
  ``` 

Now to pack them, We will use bitwise operators like this :

``` c++
    unsigned char packed = (b1 << 0) | (b2 << 1) | (b3 << 2) | (b4 << 3) |
                           (b5 << 4) | (b6 << 5) | (b7 << 6) | (b8 << 7);
```

So it's pretty straight forward, b1 takes the first bit, b2 takes the second bit.. etc..

In this simple example, We managed to save 7 bytes of data, It doesn't seem like a lot, But in the world of fast paced multiplayer games where each millisecond counts. It is worth something.


## Writing a bit packer
Now we understand how bit packing works, Let's get into my actual implementation in VoidEngine.

I have 2 separate classes for the bit packer, The "Writer" class And the "Reader" class.

While packing bits, we are working with 32 bits at a time, We call them ***Words***, So we start in the buffer at the first word, Which has 32 bits available for us to work with. If we used the whole word, We start writing bits into the next word and so on.

Here is how the writer class roughly look like :

```c++
class BitBuffer_Write
{
public:

	BitBuffer_Write(uint32_t sizeBytes);

  void WriteBits(uint32_t data, uint32_t numBits);
  void FlushBits();

private:

	uint32_t* m_Buffer = nullptr;
	// the current bit we are working with (where we are in the current word)
	uint32_t m_CurrentBit = 0;

  // the current word we are working with (where we are in the buffer.)
	uint32_t m_CurrentWord = 0;

	// the max number of bits the buffer can handle
	uint32_t m_NumBits = 0;

	// the max number of words the buffer can handle
	uint32_t m_NumWords = 0;

	// how many bits we have written so far
	uint32_t m_BitsWritten = 0;

	// the buffer size in bytes
	uint32_t m_BufferSize = 0;

  // temporary 64-bit storage used to accumulate bits before flushing them to the actual buffer.
	uint64_t m_Scratch = 0;
};
```


### The constructor
In VoidEngine, I know (at compile time) the required size for each entity in the game, So the bit buffer has a static size, Given to it in the constructor

```c++
	BitBuffer_Write::BitBuffer_Write(uint32_t sizeBytes)
	{
    // the size in bytes MUST be a multiple of 4 !
		if ((sizeBytes % 4) != 0)
		{
			sizeBytes = TableUtils::RoundUpToNearestMultiple(sizeBytes, 4);
		}
		m_Buffer = (uint32_t*)malloc(sizeBytes);
		m_CurrentBit = 0;
		m_NumBits = sizeBytes * 8;
		m_BufferSize = sizeBytes;
		m_NumWords = sizeBytes / 4;
	}
```

### Write bits !!!!!
Now it's time to actually write some bits, And the function to do that is ***WriteBits()*** (duh)

```c++
	void BitBuffer_Write::WriteBits(uint32_t data, uint32_t numBits)
	{
		V_ASSERT(numBits > 0, "amount of bits to write must be more than 0");
		V_ASSERT(numBits <= 32, "You can't write more than 32 bits at a time.");
		V_ASSERT(numBits + m_CurrentBit <= m_NumBits, "Buffer overflow");

		data &= (uint64_t(1) << numBits) - 1;

		m_Scratch |= uint64_t(data) << (64 - m_CurrentBit - numBits);

		m_CurrentBit += numBits;

		if (m_CurrentBit >= 32)
		{
			V_ASSERT(m_CurrentWord < m_NumWords, "");
			m_Buffer.get()[m_CurrentWord] = ConvertBigEndianToLittleEndian(uint32_t(m_Scratch >> 32));
			m_Scratch <<= 32;
			m_CurrentBit -= 32;
			m_CurrentWord++;
		}

		m_BitsWritten += numBits;
	}
```

Well that's a handful, Let's break it down to understand what is going on.

First, We have our standard assertions, Checking if we are not parsing some bogus data or we are not trying to write more than 1 word at a time.

### Masking the input data
Now we have validated our input, It's time to mask the input data.
Let's understand what this line does ***data &= (uint64_t(1) << numBits) - 1;***

It's better if I explain it in an example.

Above I gave the example about the number 69 that needed 7 bits only to be represented.
***00000000 00000000 00000000 01000101***
We need a bit mask to ensure only the lowest numBits bits of data are kept, preventing any excess bits from interfering with the buffer.

or in other words, Mask the lowest 7 bits as 1
***00000000 00000000 00000000 01111111***

this mask is helpful if we have excess bits that we don't need in the input data.


### Writing Data to m_Scratch
```c++
m_Scratch |= uint64_t(data) << (64 - m_CurrentBit - numBits);
```

The new data bits are shifted left to align them properly in m_Scratch.

So after that, We incremant the current bit counter and check if scratch buffer needs flushing :

```c++
		m_CurrentBit += numBits;

		if (m_CurrentBit >= 32)
		{
			V_ASSERT(m_CurrentWord < m_NumWords, "");
			m_Buffer[m_CurrentWord] = ConvertBigEndianToLittleEndian(uint32_t(m_Scratch >> 32));
			m_Scratch <<= 32;
			m_CurrentBit -= 32;
			m_CurrentWord++;
		}

		m_BitsWritten += numBits;
```

### Flushing remaining bits

After we finish writing, And we have remaining bits in the last word that didn't completely take the 32 bits of the word, Flush it.


```c++
	void BitBuffer_Write::FlushBits()
	{
		if (m_CurrentBit != 0)
		{
			V_ASSERT(m_CurrentWord < m_NumWords, "");
			if (m_CurrentWord >= m_NumWords)
			{
				return;
			}
			m_Buffer[m_CurrentWord++] = ConvertBigEndianToLittleEndian(uint32_t(m_Scratch >> 32));

			m_BitsWritten += m_CurrentBit;
		}
	}
```


## Using it in action

Now we have the bit packer ready, Let's use it to encode some data.

Let's say I have an entity I want to send over the network, The entity has these member data :

- Health (Integer between 0 and 100) (4 bytes without packing)
- Ammo (Integer between 0 and 500) (4 bytes without packing)
- IsAlive (boolean) (1 byte without packing)
- IsHostile (boolean) (1 byte without packing)

So wtihout using bit packing, This entity will take up 10 bytes of network traffic, With our bit packer above, We can do better.

## Calculating required bits for an integer
To calculate the required bits for an integer, We need to use some binary math.

Let's suppose we want to calculate the bits required to store any number between **25** and **100**.

First, We subtract the max number from the min number :

***100 - 25 = 75***

And the base-2 logarithm of the resulting number, written as log2(n), answers the question ***What is the highest power of 2 that fits inside n?***

So, log2(n) gives the highest power of 2 that fits inside n, which corresponds to the zero-based index of the most significant bit (MSB).
To count the total number of bits, we add 1.

This will result in the final function to calculate the bits required to store a number between **25** and **100**:

```c++
	int bits_required(uint32_t min, uint32_t max)
	{
		return log2(max - min) + 1;
	}
```

Back to our example above, Let's calculate the bit required for each member field in our entity :

- Health (min : 0, max : 100) ==> bits_required(0,100) ==> 7 bits
- Ammo (min : 0, max : 500) ==> bits_required(0, 500) ==> 9 bits
- IsAlive (boolean) ==> 1 bit
- IsHostile (boolean) ==> 1 bit

so the total packed size of the entity is going to be :

```c++
7 + 9 + 1 + 1 => 18 bits only !
```

***18 bits !*** to convert that to bytes we should divide it by 8. which will result in 2.25 bytes only.

so original unpacked entity size is ***10*** bytes and we managed to reduce it to ***2.25*** bytes, That's a huge improvement.


### Using it in code

After calculating the required byte size for our entity, We can use our bit packer to pack it :

```c++
int current_health = 50;
int current_ammo = 140;
bool is_alive = true;
bool is_hostile = false;

// entity packed size is 2.25 bytes, rounding that up to 3.
// the constructor itself will check if it's a multiple of 4 and will round up accordingly.
BitBuffer_Write writer(3);

writer.WriteBits(current_health, bits_required(0,100));
writer.WriteBits(current_ammo, bits_required(0,500));
writer.WriteBits(is_alive, 1);
writer.WriteBits(is_hostile, 1);

writer.Flush();
```

That's pretty much all for my proccess to pack bits in my engine, I will leave unpacking up to you to figure out :)
