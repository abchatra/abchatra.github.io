---
layout: post
title: Tagged Float
---

#Tagged Floats

###Introduction
[Tagged pointer] (https://en.wikipedia.org/wiki/Tagged_pointer) is a well know concept which every VM tries to expolit. Unlink other VM we don't tag a pointer. Instead we tag the non-pointer a.k.a a float or an int. For the purpose of this blog I will illustrate the implementation of tagged float in 64 bit. Chakra doesn't tag float in 32 bit, but tags the integer. On 64 bit Chakra tags both float and and integer. First let us see why we need tagged floats.

###Object representation
Javascript is a Garbage Collected language. Any object is represented as a Var which ia always a pointer.
```C++
typedef void * Var;
``` 
Var typically points to a [RecyclableObject](https://github.com/Microsoft/ChakraCore/blob/master/lib/Runtime/Types/RecyclableObject.h#L191). In short RecyclableObject is an object structure which holds an information about the object. It is at the root of object hierarcy which all other ojects inherit. This necessiates a vitual pointer which consumes 8 bytes for each object. It holds an additional pointer called type which is another 8 bytes. Type pointer disambiguites between various kinds of RecyclableObjects such as Strings, numbers, dynamic objects etc. 
```c++
+	__vfptr	
+   type  
```

So in nutshell 16 bytes are required to represent a simple object (again in x64).  Now lets take an example: 
```js
var metric = 10.4;
```
To represent a variable `metric` which holds an integer we need to create an object called [JavascriptNumber](https://github.com/Microsoft/ChakraCore/blob/master/lib/Runtime/Library/JavascriptNumber.h) which inherits from RecyclableObject and can store a double 10.4. Total bytes required is around 24 (`sizeof(Js::JavascriptNumber) == sizeof(Js::RecyclableObject) + sizeof(double)`. Turns out our recycle allocates at 16 byte boundary. 24 bytes is rounded off to 32 bytes. We need 32 bytes to represent JavascriptNumber. In addition to this 8 bytes Var ponter is necessary in the runtime to point this object. Here comes the saviour. 
