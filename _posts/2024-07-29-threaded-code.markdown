---
layout: post
title:  "Threaded Interpreter"
date:   2024-07-26 
categories: interpreter compiler bytecode thread vm
---

## Background

I've spent the last several months, almost a year at this point, building an intepreter for my own programming langauge called [del](https://github.com/danieltuveson/del). I've tried building interpreters a few times in the past, but I would always get hung up on building the parser, and so I didn't spend much time learning about the 'backend' of interpreters. Going into building `del`, I knew this, so I hacked together a parser fairly quickly and started building out the bytecode compiler and a virtual machine. Recently I've been wondering how to improve the performance, and stumbled into a fairly common optimization that most interpreters use called [threaded code](https://en.wikipedia.org/wiki/Threaded_code).

## A Quick Introduction to Interpreters

Despite most people's assumptions, interpreters actually often have compilation steps, even if they don't compile to machine code. Most parse code into some intermediate tree-like representation called an abstract syntax tree (an AST), but then further transform that into what is called bytecode. Bytecode is like a fake, simplified assembly language that is then run by a virtual machine (VM). A VM is basically just a big loop over the bytecode. When it encounters an instruction, it executes a bit of code, moves on to the next instruction, and so on until the program terminates.

To give a brief illustration, the code `4 + 1 - 2` might be represented as an AST like `(SUBTRACT (ADD 4 1) 2)`, and that could get converted to bytecode like `PUSH 4; PUSH 1; ADD; PUSH 2; SUBTRACT;`.

You might be wondering: why go to the trouble of doing an extra compilation step when you could just walk through the instructions in the AST? Well, many interpreters are initially written that way, since it's the most intuitive and easy way to write an interpreter. The downside to this is that it tends to be much slower for long running programs. Ruby, for instance, was originally written in this way, but was replaced with a VM implementation for performance reasons.

The hand-wavy explanation I can give for why the VM implementation is faster is basically that an AST is a bunch of different chunks of memory linked together by pointers, whereas bytecode is just a big array of memory. Looping through a big contiguous array tends to be faster than dereferencing a bunch of pointers and jumping to different places in memory that may or may not be near each other. Someone more knowlegable about how processors work could probably give a better explanation, but that's the gist of it.

Interpreting bytecode is an optimization, but it's still much slower than native code. How can we get more performance out of a bytecode interpreter? Well... you could do just-in-time compilation (JIT compilation), where you take that bytecode and compile it into actual machine code at runtime. That's what the main implementations of Java, C#, and JavaScript (V8/NodeJS) do, and many interpreted languages have alternative implementations that do this (PyPy is a JIT compiled version of Python, for example). However this opens up a whole can of worms: different CPUs have different architectures, and require writing different assembly. And each operating system has it's own set of unique system calls. Having to do a bunch of machine and os-specific compilation steps defeats one of the main benefits of an interpreter, which is that your code gets to be cross platform with little extra effort (or at least as cross-platform as the language that it is running on top of).

So if we don't have the time, the patience, or the knowlege of assembly to do JIT compiling, can we still get more performance out of our interpreter? Yes! Well yes, if you're writing in C or C++. But not on Windows. But otherwise, yes!

## A Simple Example VM

Before we get into writing the threaded VM, let's look at a simple example of a VM using a loop and switch. The following describes a simple VM that operates on a stack datastructure. The VM has 6 operations:
- PUSH: Push a value onto the stack. Takes as an argument the value to push.
- ADD: Pop two values off of the stack and add them together.
- SUBTRACT: Pop two values off of the stack, and subtract the second value from the first.
- DUPLICATE: Make a copy of the value on the top of the stack and push it onto the stack.
- JUMP IF: Pop a value off of the stack. If it is greater than 0, jump to the given location. Takes as an argument the location in the bytecode where we want to jump to. Our bytecode is an array of integers, so the location is just the index of the element in the array that we want to jump to.
- RET: Terminate the progarm and return the value at the top of the stack.

We'll define that using an enum as follows:

{% highlight c %}  
enum Bytecode {
    PUSH = 1,
    ADD = 2,
    SUBTRACT = 3,
    DUPLICATE = 4,
    JUMP_IF = 5,
    RET = 6
};
{% endhighlight %}

As I described above, most VMs will look like a loop over a big switch statment. Something like the following:

{% highlight c %}  
// Utility macros for manipulating the stack
#define pop() stack[--sp]
#define push(value) stack[sp++] = (value)
#define next() ip++

int looped(const int *bytecode)
{
    int stack[10] = { 0 };
    int sp = 1;
    int ip = 0;
    int temp1, temp2;
    while (true) {
        switch(bytecode[ip]) {
            case PUSH:
                next();
                push(bytecode[next()]);
                break;
            case ADD:
                next();
                temp1 = pop();
                temp2 = pop();
                push(temp1 + temp2);
                break;
            case SUBTRACT:
                next();
                temp1 = pop();
                temp2 = pop();
                push(temp1 - temp2);
                break;
            case DUPLICATE:
                next();
                temp1 = pop();
                push(temp1);
                push(temp1);
                break;
            case JUMP_IF:
                next();
                if (pop() > 0) {
                    ip = bytecode[ip];
                } else {
                    next();
                }
                break;
            case RET:
                goto end;
        }
    }
    end:
    return stack[sp - 1];
}
{% endhighlight %}

This works pretty well, and is decently fast. But we're wasting a bit of time. Instead of getting to the bottom of the loop, jumping to the top, and then jumping to the next instruction, why not just jump straight to the next instruction? Someone in the crowd right now is shouting "because [goto considered harmful](https://homepages.cwi.nl/~storm/teaching/reader/Dijkstra68.pdf)!" In many cases unstructured jumping can lead to hard-to-understand code, but in this scenario, it's exactly what we need. Most people would prefer their intrepreter be a little bit faster even if that means the source code of the interpreter is a little harder to understand. 

By default, standards-compliant C does not actually provide a way of using goto-labels as values, so we need a gcc-specific extension in order for this code to work (though clang also supports this extension). The relevant documentation for this feature can be found here: [labels as values](https://gcc.gnu.org/onlinedocs/gcc/Labels-as-Values.html). As of writing this, the documentation for this feature explicitly mentions that this is useful for writing fast interpreters!

{% highlight c %}  
#define cont() goto *(targets[bytecode[ip]])

int threaded(const int *bytecode)
{
    // Order of the items in this array must match the values of bytecode
    void *targets[] = { NULL, &&PUSH, &&ADD, &&SUBTRACT, &&DUPLICATE, &&JUMP_IF, &&RET };
    int stack[10] = { 0 };
    int sp = 1;
    int ip = 0;
    int temp1, temp2;
    cont();
    PUSH:
        next();
        push(bytecode[next()]);
        cont();
    ADD:
        next();
        temp1 = pop();
        temp2 = pop();
        push(temp1 + temp2);
        cont();
    SUBTRACT:
        next();
        temp1 = pop();
        temp2 = pop();
        push(temp1 - temp2);
        cont();
    DUPLICATE:
        next();
        temp1 = pop();
        push(temp1);
        push(temp1);
        cont();
    JUMP_IF:
        next();
        if (pop() > 0) {
            ip = bytecode[ip];
        } else {
            next();
        }
        cont();
    RET:
        goto end;
    end:
    return stack[sp - 1];
}
{% endhighlight %}


With this little helper macros that we've added, this isn't much more difficult to read than the loop version, and it's one less jump. That doesn't seem like a lot, but it adds up, especially if we're iterating through these instructions billions of times. 

If you want to run this and time it yourself, the code can be found [here](https://github.com/danieltuveson/bytecode). It includes some example bytecode that iterates from `1` to `INT_MAX`. If you want to hack on it, or just get a better sense of what it's doing, I'd suggest enabling logging and changing `INT_MAX` to a small integer.

Check out the Lua interpreter if you want a real-world example of a threaded interpreter. It actually doesn't look too different than what we have above. The main difference is that their VM uses a couple of helper macros to allow them to seemlessly integrate this feature while also allowing the compiler to toggle this optimization on and off, that way their code can enable this feature only for C compilers that support it. Checkout `luaV_execute` in [lvm.c](https://github.com/lua/lua/blob/master/lvm.c) for their VM code, which uses the [ljumptab.h](https://github.com/lua/lua/blob/master/ljumptab.h) macros to enable threaded code.

