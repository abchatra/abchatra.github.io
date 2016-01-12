---
layout: post
title: Inline Cache
draft: true
---
In this post, we are going to understand the inline cache with respect to Chakra. We will briefly look at the need for inline cache as well. If you haven't heard of inline cache at all, this [wiki](https://en.wikipedia.org/wiki/Inline_caching) page and the [type](http://abchatra.github.com/Type) blog are necessary to read before we deep dive. To quote from wiki:
*The concept of inline caching is based on the empirical observation that the objects that occur at a particular call site are often of the same type. In those cases, performance can be increased greatly by storing the result of a method lookup "inline", i.e. directly at the call site.*

Note, in this context of blog call site means any property access location (line of code), not just a location where a function is called. First let us understand the cost of property lookup in Chakra.

<!--more-->  

**Blog in progress...**

###Cost of property lookup

Example is the best way to begin:
```js
function Car(make, model) {
  this.make = make;
  this.model = model;
}

var mycar = new Car("Honda", "Accord");

function GetModel(machine)
{
  return machine.model;
}
print(GetModel(mycar));
```

Function *GetModel* returns *model* property from *machine* object. Typically *machine* is instance of *Car*. Though *GetModel* can be invoked from a different type of object such as *SewingMachine* and still the contract of code is valid. Sewing machines also have model property. 

```js
function SewingMachine(brand, model) {
  this.brand = brand;
  this.model = model;
}
var mymachine = new SewingMachine("Brother", "CS6000i");
print(GetModel(mymachine));
```

When the *model* property is accessed in script following steps are taken. (Read about [type](http://abchatra.github.io/Type) here)

1.  Fetch the type (machine->type) of the object. 
2.  Invoke dictionary lookup in property map ((machine->type->typeHandler) to get the slot number corresponding to property *model*. Lets say property model is at slot 1;machine->slots[1]

Step 2 is super expensive for runtimes. 


###Dumping bytecode

Let us dump the bytecode using debug version of ch.exe to see inline caches. Ch.exe is a lightweight console host for hosting ChakraCore. See *using ChakraCore* section [here](https://github.com/microsoft/chakracore) for how to build ch.exe . 

```
ch.exe test.js -dump:bytecode

Function GetModel ( (#1.2), #3) (In0, In1) (size: 11 [10])
      5 locals (1 temps from R4), 2 inline cache
  Line  12: return myCar.model;
  Col    6: ^
    0012   ProfiledLdFld        R0 = R3.model #0 <0>
    0016   Br                   x:0021 (   8)

```

