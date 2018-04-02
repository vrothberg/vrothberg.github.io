---
layout: post
author: Valentin Rothberg
title:  "Pointers in Golang considered harmful"
date:   2018-04-02
tags: golang pointers harmful pattern bug
---

*In the following article, I describe a painful to debug bug pattern in Golang when using pointers in loops.  When I first hit this pattern, I was working on a rather big code base ([Moby](https://github.com/moby/moby)), which further complicated localizing and understanding the cause.  I was convinced of having found a bug in the Golang compiler, and had to disassemble some binaries before finding out that Golang is fine, but that its syntax and my programming habits and conventions of Golang tricked my brain into the believe of having written safe code.  As I've seen other people falling into the very same trap, I decided to share my findings here and to raise awareness.*

### Pointers and Golang
Pointers are a powerful concept of many programming languages by giving us more control over whats happening in memory rather than just leaving everything to the compiler and the run-time.  Also, pointers make our lifes easier and more miserable at the same time.  As we have more control over how our programs deal with memory, we can come up with elegant yet efficient solutions, making execution potentially faster and less resource demanding.  Sometimes it's just a sad fact of life that we have to take care of the memory layout, for instance, when talking to hardware.  On the contrary, the concept of pointers and the consequence of writing very explicit code is the source of many bugs, errors and also of serious security flaws.  Using pointers on large code bases requires a lot of concentration and thorough testing, but one ringing telephone may just be enough distraction to make even simple mistakes.  That's why certain programming languages abstract pointers away and exchange some performance for some usability and user-friendliness.

As a systems language, Golang necessarily implements pointers as we need means to reference memory; and we need to be explicit about it.  However, Golang does not strictly enforce the usage of pointers as the compiler can allocate memory implicitly for some types, it has a built-in garbage collector so that we don't have to worry much about freeing memory, and it avoids common pitfalls and mistakes, for instance with the `range` statement to safely iterate over loops.  While we have to type quite some boilerplate code in C, for instance, when iterating over an array, Golang allows us to simply use `index, value := range array` and we do not have to worry about sizes, memory allocation or off-by-ones.  In many ways, Golang helps my brain to focus on what's really important when I program: solving a specific problem in a comparatively efficient way.

### A harmful usage pattern of pointers
Although Golang makes coding very fast and friendly, the abstraction of memory and pointers are the core of all evil of this article:  **Golang conditions me to forget about pointers**.

With all the relevant pieces together, let us get concrete and have a look at the following code:

```golang
package main

import (
	"fmt"
)

type foo struct {
	i int
}

func main() {
	foos := []foo{foo{i: 0}, foo{i: 1}, foo{i: 2}, foo{i: 3}}
	ptrs := []*int{}
	for i, x := range foos {
		fmt.Printf("foo %d: %+v\n", i, x)
		ptrs = append(ptrs, &x.i)
	}
	for i, p := range ptrs {
		fmt.Printf("Pointer %d points to %d\n", i, *p)
	}
	fmt.Printf("Pointers: %+v\n", ptrs)
}
```

The problem in the upper code example becomes clear when having a look at the output of the printf debugging statements:
```
foo 0: {i:0}
foo 1: {i:1}
foo 2: {i:2}
foo 3: {i:3}
Pointer 0 points to 3
Pointer 1 points to 3
Pointer 2 points to 3
Pointer 3 points to 3
Pointers: [0x10414020 0x10414020 0x10414020 0x10414020]
```

We can see that all elements in the `ptrs []*int` array point to the very same memory location, which is the last element of `foos []foo`.  That happens because the loop variable **x** in `for i, x := range foos` is allocated once and its memory location (i.e., **&x**) is used in each iteration.  In other words, for each iteration in the loop, the value of the current element is copied to the memory of the loop variable.  In yet another sequence of words: the semantics of a loop variable in Golang is copy-by-value, not copy-by-reference.

Spotting the mistake in the upper minimal example is rather easy, however it becomes really tricky to debug in more complex scenarios and when we are not directly addressing a loop variable's memory.  Just image, we call something like `x.DoSomething()` or `DoSomething(&x)` instead of using `&x.i` or `&(&x).i` directly.  As mentioned above, tracking down this issue took me quite a few hours and I have seen others falling into the very same trap, which made me reflect on why this erroneous pattern occurs and how we can take counter-measures to avoid it.

### Why does the pointer pattern happen?
This is strictly my personal view, but I think that **Golang may abstract pointer usage a little bit too much**.  Let's consider an exemplary method `func (f *foo) bar()` and a variable `f := foo{}`.  Both, calling it via `f.bar()` or via `(&f).bar()` are valid in Golang as the compiler will figure out the context on its own.  Although I generally enjoy not having to think much if a given receiver has a call-by-value or call-by-reference semantics, it might just condition me and others to not care as much as we might should.  Using pointers as a method's receiver is just so tempting.  We can operate directly on a given object and do not need to copy data around.  More background information can be found in Golang's [spec on selectors](https://golang.org/ref/spec#Selectors).

### How can we avoid the pointer pattern?
Please notice, that there is no single solution as the problem very much depends on the context, so I just want to mention a few possibilities.

1. If you need pointer semantics, do not use the loop variable, but reference elements in the array directly via an index (e.g., `&myArray[index]`).

2. If you need pointer semantics, allocate dedicated memory on the heap by declaring a variable in the loop (e.g., `bar := loopVariable`).

3. If you invoke a method on the loop variable or pass it to some functions, be sure of the calling semantics.  If you are or cannot be sure, use one of the workarounds mentioned above.

4. When writing an API, use pointers only when you're utterly sure that using such cannot cause the problem mentioned above.

### A case for static analysis?
All counter-measures mentioned above require human awareness (a) for the problem itself and (b) also when writing and reviewing code.  That's usually a **perfect use-case for static analysis**.  My absolut favorit tool for such cases when using C is [Coccinelle](http://coccinelle.lip6.fr/).  Coccinelle is a program matching and transformation engine.  Coccinelle is comparatively easy to use as it can be used via a dedicated scripting language, SmPL (**S**e**m**antic **P**atch **L**anguage), which has bindings for Python and OCaml.  Coccinelle had and has a [huge impact on the Linux kernel](https://lwn.net/Articles/698724/) and there are many research papers available on how, where and why to use Coccinelle, so please refer to the [website](http://coccinelle.lip6.fr) if you want to know more.

The only tool I am aware of which does something similar to go what Coccinelle does to C programs is gofmt.  [gofmt](https://golang.org/cmd/gofmt/) has some kind of code transformation engine, but in a very simplified form as it only supports to match and subsequently rewrite [expressions](https://golang.org/ref/spec#Expressions).  Formulating rules based on expression only makes it hard to come up with something comparable to SmPL scripts, which allow us to match (nearly) all kinds of syntactical elements and tokens in nearly all kinds of context.  This leaves us alone with either writing very specific rules -- which isn't very practical in reality -- or trying to extend the scope of gofmt...or to just be aware of the problem.
