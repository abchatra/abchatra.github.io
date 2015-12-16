---
layout: post
title: Tagged Float
---

#Tagged Floats

[Tagged pointer] (https://en.wikipedia.org/wiki/Tagged_pointer) is a well know concept which every VM tries to expolit. Unlink other VM we don't tag a pointer. Instead we tag the non-pointer a.k.a a float or an int. For the purpose of this blog I will illustrate the implementation of tagged float in 64 bit. Chakra doesn't tag float in 32 bit, but tags the integer. On 64 bit Chakra tags both float and and integer.

Javascript is a Garbage Collected language. Any object is represented as a Var which ia always a pointer.
```C++
typedef void * Var;
``` 
Var typically points to a [RecyclableObject](https://github.com/Microsoft/ChakraCore/blob/master/lib/Runtime/Types/RecyclableObject.h#L191). In short RecyclableObject is an object structure which holds an information about the object. It is at the root of object hierarcy which all other ojects inherit. This necessiates a vitual pointer which consumes 8 bytes for each object. It holds an additional pointer type 
