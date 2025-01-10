---
layout: post
title:  "Exceptions or Lies"
date:   2025-01-07
categories: exceptions nullable truth
---

I've been working on my programming language, `del`, and trying to figure out how to implement arrays. Like most simple language features, there are a bunch of little things to consider that you probably wouldn't think about until you sit down and try to implement it. Right now I'm trying to figure out how to implement arrays. I want creating an array to be just like creating any other object, where the size of the array is a parameter in the array constructor. So declaring an array of 100 `int`s would look like this: `let arr = new Array<int>(100)`. But this raises a question: what should I do if I pass a number less than 1 to the array constructor?

There are a bunch of things that I could do here: 
- I could have it return an array with a default size, or maybe something like size equal to the absolute value of the number passed in. I don't like this, since it's unintuitive and could lead to further errors.
- Return null, a nullable array, or an error type. I don't like this either - then every time you create an array you need to add null checking and potentially return null.
- I could add unsigned integers to my language and force the array size to be unsigned. This is what rust does. This is not the worst option, but I find unsigned numbers to not be that useful. Most users would probably just cast to unsigned without thinking about it anyway. I also don't want to allow arrays of size 0, and unsigned integers can still be zero.
- Throw an exception. I don't like exceptions - but this seems like the correct thing to do. Passing a negative number to an array constructor should be pretty unusual, and I would rather have the flexibility of allowing arrays to be initialized to a dynmaic value. This is what Java does - though strangely only for negative numbers - for some reason it allows zero. 

I think I do want to allow some exceptions in my language, but minimize them to the extent that I can. Exceptions are nice for things that happen rarely, but would incur some massive performance hit if you got rid of them, like array indexing or math (division by zero... zero values make life so painful). Also for things like assertions - if someone is using a function / constructor the wrong way, I think an exception will quickly let them know that they're using it incorrectly, rather than making them propagate the error or by doing something unexpected that doesn't crash.

I was also thinking about how to implement indexing and wondering about what other languages do, so I took a look to see what Haskell does. Surprisingly most of the functions in `Data.List.Index` will just return your original list if you give them an invalid index. This is documented, but if I were looking at source code that used these functions, I might not expect it. I think this also might be a case where exceptions make more sense (rust considers this to be the case, anyhow).
