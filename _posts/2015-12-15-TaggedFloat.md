---
layout: post
title: Tagged Float
---

A [tagged pointer] (https://en.wikipedia.org/wiki/Tagged_pointer) is a well know concept which every virtual machine (VM) tries to exploit. Unlike some other VM's Chakra doesn't tag a pointer. Instead, Chakra tag's the non-pointer a.k.a a float or an int. For the purpose of this blog, I will illustrate the implementation of tagged float in 64 bit. (Float means double-precision 64-bit format IEEE 754-2008 as specified by [ECMA262](http://tc39.github.io/ecma262/#sec-ecmascript-language-types-number-type)). Chakra doesn't tag floats in 32 bit but tags integers. On 64 bit Chakra tags both floats and integers. First let us see our object representation, it's size and the need for tagged floats.

###Object representation
Javascript is a Garbage Collected (GC) language. Any object or primitive (which represents javascript var) is accessed as a void pointer (`void *`) named Var in the context of Chakra runtime.

```C++
typedef void * Var;
``` 

Var typically points to a [RecyclableObject](https://github.com/Microsoft/ChakraCore/blob/master/lib/Runtime/Types/RecyclableObject.h#L191). RecyclableObject is the root of the object hierarchy which all other objects inherit. This necessitates a vTable pointer which consumes 8 bytes for each object. It holds an additional pointer to [Type](https://github.com/Microsoft/ChakraCore/blob/master/lib/Runtime/Types/Type.h#L22) which accounts for another 8 bytes. Type structure disambiguates between various kinds of RecyclableObjects such as strings, numbers, dynamic objects etc and can be shared between multiple objects. Every RecyclableObject has following two fields:

```C++
__vfptr*   // 8 bytes
type*      // 8 bytes
```

In a nutshell, 16 bytes are required to represent a simple object (again in x64).  Now let us take an example. Following `speed` variable holds a float. 

```js
var speed = 10.4;
```

To represent this Var in the engine, Chakra need to create an object named [JavascriptNumber](https://github.com/Microsoft/ChakraCore/blob/master/lib/Runtime/Library/JavascriptNumber.h) which inherits from RecyclableObject and can store a float value (10.4). Total bytes required is 24 (`sizeof(Js::JavascriptNumber) == sizeof(Js::RecyclableObject) + sizeof(double))`. Turns out our GC allocates at 16 byte boundary. 24 bytes is rounded off to 32 bytes. We need 32 bytes to represent a JavascriptNumber. In addition, to this 8 byte Var pointer is necessary for the runtime to point to this object. Can we avoid this overhead for every var pointing to a float? If we could encode a float in a pointer and mark it as a non-pointer, the runtime can recognize it and we save on bytes.

###Extra bits in a pointer
Let us look at memory address allocated by the GC carefully.
Couple of characteristics of pointers:

 1. Bottom 4 bits are always going to be zero. (Remember our GC allocates at 16 byte boundary)
 2. Top 16 bits are going to zero as the operating system only uses bottom 48 bits to represent virtual memory address (256TB is good enough).
 
We can party with these extra bits. One can use these always zero bits to encode a float. But how will runtime differentiate between a valid pointer or a float encoded in a pointer? Can we encode the entire float value which 64 bit inside a pointer which also 64 bits and don't lose precision? First it is important to understand the floating point representation. 

###IEEE 754 floating point representation
Now let us look at the 64-bit double precision IEEE 754-2008 format specified by [ECMA262](http://tc39.github.io/ecma262/#sec-ecmascript-language-types-number-type).
A floating point variable is represented as following.

|Sign|Exponent|Fraction|
|----|:------:|-------:|
|1 [63]|11 [62-52]|52 [51-00]|

See [this] (http://steve.hollasch.net/cgindex/coding/ieeefloat.html) blog for more information on floating-point format. The interesting part is the exponent. If all the 11 bits in the exponent are 1 it can represent 3 values

1. Positive infinity.
2. Negative infinity.
3. NaN

NaN's are represented by a bit pattern with an exponent of all 1's and a non-zero fraction. If the fraction is all zero it can be either +Infinity or -Infinity depending on sign bit. This lets tons of ways of expressing NaN. Ecma262 specifies that all NaN's are treated equally. So we just need one. We canonicalize all the NaN's to just the one shown below which is, in fact, a [QNaN](https://en.wikipedia.org/wiki/NaN). 

```C++
static const uint64 k_Nan    = 0xFFF8000000000000ull;
static const uint64 k_PosInf = 0x7FF0000000000000ull;
static const uint64 k_NegInf = 0xFFF0000000000000ull;
```

Now let us dig deeper into our tagging scheme.

###Tagging scheme
Remember our goal here is to tag a pointer our goal here is

1. Differentiate between a pointer and a float.
2. Not lose any data while encoding a float.

From the above IEEE 754 floating representation we know that all float values (except NaN, +Infinity, -Infinity) are guaranteed to have at least one of the exponent bits **not set**. So we xor all floats with **0xFFFC0000 00000000 00000000 00000000 or 0xFFFC<<48** and store them in the memory as pointers. This magic xor constant guarantees that all floats will have at least one bit set in the exponent part (bits 62-52). This magic constant also ensures that NaN, Infinity & -Infinity have 50th-bit set. To generalize all floats will have one of the top 16 bit set. Pointers won't have any of the top 16 bit set. 

A simple table to illustrate.

|float value or pointer|Bit pattern in hex|Bit pattern after xor|
|---:|:---:|:---:|
|0.0|0000000000000000|fffc000000000000|
|0.4|c02599999999999a|3fd999999999999a|
|Infinity|7ff0000000000000|800c000000000000|
|-Infinity|fff0000000000000|000c000000000000|
|NaN|fff8000000000000|0004000000000000|
|RecyclableObject*|00000209512b4e20|00000209512b4e20|

Note: Chakra keeps RecyclableObject pointer values as is. 

See links for [floating point conversion](http://babbage.cs.qc.edu/courses/cs341/IEEE-754.html) calculator & [xor](http://xor.pw/) calculator.

Runtime simply looks at top 16 bits (x >> 48 != 0). If any of top 16 bit is set, it is a float. Or else it is a valid pointer to a RecyclableObject. If it is a float, runtime get original double by xoring with **0xFFFC<<48** (`x == (x^0xFFFC<<48^0xFFFC<<48`). Total memory spent on a float in the VM is just 8 bytes as we directly store the float value instead of storing a pointer to any other data structure.

To close this post Chakra does bit twiddling magic to save 32 bytes of memory for each var pointing to a float. Hope this helps. Please let me know the feedback either through email or leaving a comment here. 
