---
layout: post
title: Inline Cache
---

Inline cache [wiki](https://en.wikipedia.org/wiki/Inline_caching) page and [type](http://abchatra.github.com/Type) are necessary read before we begin our discussion on implementation of inline cache in Chakra. Quote from wiki:
*The concept of inline caching is based on the empirical observation that the objects that occur at a particular call site are often of the same type. In those cases, performance can be increased greatly by storing the result of a method lookup "inline", i.e. directly at the call site.*

Note, in this context of blog call site means any property access location (line of code), not just a location where a function is called. First let us understand the cost of property lookup in Chakra.

<!--more-->  

###Cost of property lookup


Let us take an example:

```js
function GetX(o)
{
   if (typeof o == 'object')
   {
    return o.x;
   }
   return null;
}
```

