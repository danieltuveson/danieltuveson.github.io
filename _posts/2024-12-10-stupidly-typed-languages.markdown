---
layout: post
title:  "Stupidly Typed Programming Languages"
date:   2024-12-10
categories: programming languages stupid satire
---

Earlier this week, I was browsing a programming forum and saw two fellows arguing about the definitions of “strong” and “weak” typing. One of the fellows argued that Python has a “strong” type system, because every object has a type, and that C has a “weak” type system because the programmer can circumvent the type system by casting. The other argued that this definition deviated from the original definition of “strong” typing in that a program with “strong” types will not run unless the program satisfies a set of constraints determined by a typechecker, and that by this definition, C would be “strongly typed” and Python would be “weakly typed”.

“Alas,” a third fellow chimed in, “‘strongly typed’ and ‘weakly typed’ are not rigorously defined academic terms,” and then linked to this [wikipedia page](https://en.wikipedia.org/wiki/Strong_and_weak_typing).

“Shut up nerd,” the original fellow responded. The second fellow concurred, “I agree with you, but I agree with OP that you are a nerd and should shut up.”

Definitions are hard. The task of defining a term, such that it is consistently used in an unambiguous manner is hard. Difficult. A struggle. Tough stuff. You get it.

As such, I propose a piece of new unambiguous terminology that can be used to denigrate programming languages that you dislike. The set of all programming languages can be split into two disjoint sets: “nerdly typed” and “stupidly typed” languages. Given the following example:

```
function main() {
    x = 1
    print("Hello from a stupid programming language!")
    x.im_stupid()
}
```

A language implementation must be considered “stupidly typed” if its closest equivalent to the above pseudocode would print “Hello from a stupid programming language!” before printing out any other text. If a language satisfies this property, it is a stupid language, and if you use it you are probably an idiot. If it does not, then you are programming in a language for nerds, and you are probably a loser that doesn’t have any friends.

Now I know what you’re thinking. What if a language has a builtin `im_stupid()` method on integers? I’m sorry, but by the canonical definition, it is a stupidly typed programming language. And even worse, you’re a nerd for being so pedantic.

