---
layout: post
title: Tagged Float
---

[Tagged pointer] (https://en.wikipedia.org/wiki/Tagged_pointer) is a well know concept which every virtual machine (VM) tries to exploit. Unlike some other VM's Chakra doesn't tag a pointer. Instead Chakra tag's the non-pointer a.k.a a float or an int. For the purpose of this blog I will illustrate the implementation of tagged float in 64 bit. Chakra doesn't tag float in 32 bit, but tags the integer. On 64 bit Chakra tags both float and an integer. First let us see why we need tagged floats.

###Object representation
Javascript is a Garbage Collected (GC) language. Any object is accessed as a Var which is always a pointer.

```C++
typedef void * Var;
``` 

Var typically points to a [RecyclableObject](https://github.com/Microsoft/ChakraCore/blob/master/lib/Runtime/Types/RecyclableObject.h#L191). In short RecyclableObject is an object structure which holds an information about the object. It is at the root of object hierarchy which all other objects inherit. This necessitates a vTable pointer which consumes 8 bytes for each object. It holds an additional pointer to _type_ which accounts for another 8 bytes. [Type](https://github.com/Microsoft/ChakraCore/blob/master/lib/Runtime/Types/Type.h#L22) structure disambiguate between various kinds of RecyclableObjects such as strings, numbers, dynamic objects etc and can be shared between multiple objects.  

```c++
+    __vfptr*    
+   type*  
```

In a nutshell 16 bytes are required to represent a simple object (again in x64).  Now lets take an example: 

```js
var velocity = 10.4;
```

To represent a variable `velocity` which holds an integer we need to create an object called [JavascriptNumber](https://github.com/Microsoft/ChakraCore/blob/master/lib/Runtime/Library/JavascriptNumber.h) which inherits from RecyclableObject and can store a double value (10.4). Total bytes required is 24 (`sizeof(Js::JavascriptNumber) == sizeof(Js::RecyclableObject) + sizeof(double)`. Turns out our GC allocates at 16 byte boundary. 24 bytes is rounded off to 32 bytes. We need 32 bytes to represent JavascriptNumber. In addition, to this 8 byte Var ponter is necessary for the runtime to point this object. Can we do better?

###Extra bits in a pointer
Lets look at memory address allocated by the GC carefully.
Couple of characteristics of pointers:

 1. Bottom 4 bits are always going to be zero. (Remember our GC allocates at 16 byte boundary)
 2. Top 16 bits are going to zero as operating system only uses bottom 48 bits to represent virtual memory (256TB is good enough).
 
We can party with these extra bits. How exactly GC ignores this is for another post. The assumption is if you tag any of these bits and use it for any other purpose GC doesn't care. It ignores entire value of that pointer. For tagged float Chakra uses top 14 bits out of 16.

###IEEE 754 floating point representation
Now lets look at the 64-bit double IEEE 754-2008 format specified by [ECMA262](http://tc39.github.io/ecma262/#sec-ecmascript-language-types-number-type).
A floating point variable is represented as following.

|Sign|Exponent|Fraction|
|----|:------:|-------:|
|1 [63]|11 [62-52]|52 [51-00]|

See [this] (http://steve.hollasch.net/cgindex/coding/ieeefloat.html) blog for more information on floating-point format.
NaN's are represented by a bit pattern with an exponent of all 1's and a non-zero fraction. If the fraction is all zero it can be either +Infinity or -Infinity depending on sign bit. This lets tons of ways of expressing NaN. Ecma262 specifications say all the NaN's are treated equally. So we just need one. We canonicalize all the NaN's to just the one shown below which is in fact a QNaN. 

```C++
static const uint64 k_Nan    = 0xFFF8000000000000ull;
static const uint64 k_PosInf = 0x7FF0000000000000ull;
static const uint64 k_NegInf = 0xFFF0000000000000ull;
```


