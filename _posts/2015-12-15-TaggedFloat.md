---
layout: post
title: Tagged Float
---

###Introduction
[Tagged pointer] (https://en.wikipedia.org/wiki/Tagged_pointer) is a well know concept which every VM tries to expolit. Unlink other VM we don't tag a pointer. Instead we tag the non-pointer a.k.a a float or an int. For the purpose of this blog I will illustrate the implementation of tagged float in 64 bit. Chakra doesn't tag float in 32 bit, but tags the integer. On 64 bit Chakra tags both float and and integer. First let us see why we need tagged floats.

###Object representation
Javascript is a Garbage Collected language. Any object is represented as a Var which ia always a pointer.

```C++
typedef void * Var;
``` 

Var typically points to a [RecyclableObject](https://github.com/Microsoft/ChakraCore/blob/master/lib/Runtime/Types/RecyclableObject.h#L191). In short RecyclableObject is an object structure which holds an information about the object. It is at the root of object hierarcy which all other ojects inherit. This necessiates a vitual pointer which consumes 8 bytes for each object. It holds an additional pointer called type which is another 8 bytes. Type pointer disambiguites between various kinds of RecyclableObjects such as Strings, numbers, dynamic objects etc. 

```c++
+	__vfptr*	
+   type*  
```

So in nutshell 16 bytes are required to represent a simple object (again in x64).  Now lets take an example: 

```js
var velocity = 10.4;
```

To represent a variable `velocity` which holds an integer we need to create an object called [JavascriptNumber](https://github.com/Microsoft/ChakraCore/blob/master/lib/Runtime/Library/JavascriptNumber.h) which inherits from RecyclableObject and can store a double 10.4. Total bytes required is 24 (`sizeof(Js::JavascriptNumber) == sizeof(Js::RecyclableObject) + sizeof(double)`. Turns out our recycle allocates at 16 byte boundary. 24 bytes is rounded off to 32 bytes. We need 32 bytes to represent JavascriptNumber. In addition to this 8 bytes Var ponter is necessary in the runtime to point this object. Here comes the saviour. 

###Tagging scheme
Lets look at memory address allocated by the GC carefully.
Couple of characteristics of pointers:

 1. Bottom 4 bits are always going to be zero. (Remember our recycler allocates at 16 byte boundary)
 2. Top 16 bits are going to zero as Operating system only uses bottom 48 bits to represent virtual memory (256TB is good enough).
 
We can party with these extra bits. How exactly GC ignores this is for another post. Assumption is if you tag any of these bits and use it for any other purpose GC doesn't care. It ignores entire value of that pointer.  

Now lets look at the 64-bit double IEEE 754-2008 format specified by [ECMA262](http://tc39.github.io/ecma262/#sec-ecmascript-language-types-number-type)

A floating point variable is represented as following.

|Sign|	Exponent|	Fraction|
|------------- |:-------------:| -----:|
|	1 [63]|	11 [62–52]|	52 [51–00]|

See [this] (http://steve.hollasch.net/cgindex/coding/ieeefloat.html) blog for more infomration on floating point format

```C++
static const UINT64 k_Nan = 0xFFF8000000000000ull;
```
