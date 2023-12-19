---
title: "A Modest Compiler for JavaScript"
date: 2023-12-19T16:58:03+01:00
draft: true
toc: false
images:
tags:
  - mcjs
---

<!--- intro [like the top of an email] -->

In the last year or so, I've been working on **mcjs**, a Modest Compiler for JavaScript: a *toy* (hence the modesty) implementation of a JavaScript VM including both an interpreter and a tracing JIT compiler, with an internal design inspired (loosely) by LuaJIT.  It's still *far* from any sort of completion but it's a very enjoyable learning activity nonetheless. 

This blog exists because this little side project has now accrued enough "history" that it seemed like a good idea to write down and collect some notes, if only for myself to remember why I made a certain choice or another. And if some reader somewhere finds it useful or interesting, then it'll be all the more worth it :)

## Brief history of its predecessor: mclua

(You can skip this one, no hard feelings.)

<!--- Really keep this? Should it be shorter? -->

<!-- TODO  make it clearer: it was and is for *fun* -->
I started mcjs as my *second* attempt at writing a compiler. In 2021, after temporarily changing my living arrangement in response to the COVID situation in Italy, I started devoting a part of the proverbial "nights and weekends" to learning how compilers really work, a topic for which I had long felt a strong but unsatisfied curiosity and interest. It would also be way to practice my software development skills while the ["curfew"](https://it.wikipedia.org/wiki/Gestione_della_pandemia_di_COVID-19_in_Italia#Allentamento_delle_misure_di_contenimento_(26_aprile_-_5_agosto_2021)) (sorry, couldn't find a page in English) prevented us all from spending our nights in pubs as we would have otherwise.

There is plenty of good reading material online, ranging from [academic papers](https://arxiv.org/abs/2011.05608) to [blog posts](https://sillycross.github.io/2022/11/22/2022-11-22/) to [tweets](https://twitter.com/DrawsMiguel/status/1727355995507376630). With this sort of DIY curriculum, I managed to cover a bunch of topics, including various intermediate representations (SSA, A-normal, CPS), basic optimizations such as constant folding and dead code elimination, register allocation.  

This was all put into practice by working on *mclua*, a Modest Compiler for the Lua programming language.  The intention, if you can believe the naïvety of it, was to *purposely* go against the grain and write an Ahead-of-Time compiler instead of the classic interpreter, pretending that I didn't know any better (I do, I promise), and then try to learn as much as possible from this "artificially misguided" endeavor. To top it off, the compiler itself was also written in Lua.

I can say now that the experience has been a lot of fun while it lasted. Lua is an unusual choice for compiler, but if performance is of no concern, one can take advantage of its "dynamicness" to build some pretty comfy abstractions. There is also a certain satisfaction that comes from working with the most bare-bones of tools: no debuggers, no IDE/IntelliSense, just the 'lua' standalone interpreter and a ton of unit tests. I can join the choir of those saying how delightful Lua is in its minimalism, "obviousness", and flexibility.

In the end, though, the misguided-ness of this attempt caught up to me and I decided to stop working on mclua.  By this time, I had already learned a ton about compilers, I had already felt the magic of seeing your own compiler crunch an apparently complicated function into a handful of sharp and specific IR instructions, while the relative dearth of real-world standalone Lua code that I could use for testing my compiler on was starting to feel more and more limiting.

So, after some time, I decided to pivot to JavaScript and JIT compilers, which brings me to mcjs.

## So what is mcjs, really?

I wanted to have *more* fun with compilers and programming languages implementations than I was already having, so I started *mcjs* as yet another side project.

This time, I wanted to do things more like "the grownups": mcjs is a bytecode interpreter with a JIT tier. Although the source language is no longer Lua, I tried to take some lessons from LuaJIT, of which I admire the good trade-off between runtime performance and implementation simplicity.

Here is what I want mcjs to be and *not* to be:

- *Compatibility*. It should implement enough JavaScript and Node.js APIs to run a non-trivial library or application. It *won't* implement all of Node.js or run an actual system. That's fine!
    - To avoid scope creep, I'm just going to ignore some of the more advanced features of the language, or heavily simplify them.  For example, generators and async/await will probably not make it; I will only implement a simplified variant of ES modules, while skipping CommonJS entirely (or maybe I'll change my mind and do it the other way around!).

- *Performance*. It should be within 10x of interpreter-only V8/Node.js. In other words, *any* performance that is not obviously the result of a bug will be fine.
    - Still, the JIT should be measureably faster than the interpreter.
    - I don't think that the interpreter will ever be as fast as LuaJIT's, which is hand-coded in Assembly with hand-picked registers by a master of the craft. I'm fine with a dumb interpreter written in plain portable Rust. I might refine the bytecode design and/or add an intermediate compilation tier based on "quick and dirty" [copy-and-patch](https://arxiv.org/abs/2011.13127) at some point.

- *Learning*. This the real goal: knowing how the runtime interpreted sausage is made. Of particular interest to me are optimization techniques, JIT compilation techniques and efficient bytecode design. I'm going into this without a detailed understanding of what I'll need to study. I have a clear plan laid out in broad strokes in my head; I'll just see what's needed and study it when time comes.

Another non-goal is creating a real open source project and community. Still, if you have any comment or critique, or want to discuss any of the topics presented here, I'd *love* to hear it!

### Did it  *have*  to be JavaScript?

I didn't pick JavaScript after thinking very long and hard about it. It seemed cool, and that was it. But there were some things on my mind that tipped the scale:

- There is a [world-economy-moving amount](https://www.npmjs.com/) of JavaScript code readily available. <!--- It's useful to have many "real" packages that are designed to run in a standalone process (in Node.js), accomplish some generic task, and are simple enough to test my compiler on without implementing too many "system" APIs. In my experience, most of the simple-enough open source Lua code you can find is made to extend a larger application and it relies on the the application's extension APIs to be run and tested. The prospect of one day running simple but "real" programs or libraries is good for motivation! -->
- There are multiple production-quality parsers ready to use, multiple implementations you can compare yourself to (and they even write fantastic [blog](https://v8.dev/blog) [posts](https://webkit.org/blog/10308/speculation-in-javascriptcore/)), and a lot of high quality documentation.
- I kinda like using it, especially with all the features from ES6 and later.
- JavaScript has an official spec. I still think it's not as good as using an existing implementation as a test oracle, but I appreciate being able to look up a detail and read a clear, black-on-white definition of how things ought to work.

There are also plenty of drawbacks: JavaScript is a *much* more complicated language, notoriously plagued by [weird, unintuitive semantics](https://www.destroyallsoftware.com/talks/wat) of even the most basic parts of it. Still, the point here is having fun and learning, and I'm going to do plenty of both with JS.

## Alright, show me what you got

The code for mcjs is currently [hosted on GitHub](https://github.com/sebastiano-barrera/mcjs/). It's not particularly refined or polished for public fruition. What you see is what you get, really. It's written in Rust, so building and running it on a basic level should be as easy as running `cargo build` and `cargo test`.

<!---
As a general overview the main modules are:

* The *bytecode compiler*.  Takes care of transforming the source JavaScript code into bytecode (split into functions, grouped into modules).

* The *loader*.  The data structure that stores and caches compiled code. It also implements the module search logic.

* The *interpreter*.  Just what it says on the tin: runs bytecode and holds the VM's runtime state.

* The *debugger*.  A bare-bones debugger that I'm using to help myself understand failures of the test suite.

* The *JIT compiler*.  Ties into the interpreter to record and compile traces, so that they can be called from and return to the interpreter.
    * It's currently broken and disabled. More on this later.
    * It's a *tracing* JIT, like LuaJIT, but maybe this will change.

More notes on each part will come in subsequent posts. 
-->

Some work is already done:

* A good chunk of the syntax is supported already. All the basic expressions, statements and declarations are in, alongside an extremely simplified version of ES modules (`import`/`export`).

* The interpreter is capable of running the [JSON5](https://www.npmjs.com/package/json5) library to a non-trivial degree (although there is still *a lot* to be fixed).  A copy of it is included in the source tree as a test asset.

* The debugger has most of the basics (setting/clearing a breakpoint, next instruction, continue, restart, view details of the values and objects on the stack/heap).

* I started some work the JIT compiler, but I decided to "shelve" it in favor of going further with the interpreter first. This is because the JIT implementation and design is by its nature very tightly coupled with the interpreter's internals.  I realized that the interpreter is still too far from complete and susceptible to change in the near future, so I would risk reworking the JIT completely many times before having the whole VM work fine. So, it'll just wait.

But many more are needed in order to reach the goal. The most important are:

 + Complete the implementation of the basic parts of the language's semantics, at least the basic parts of the language.  Objects have to work *just* right. 
 + Garbage collector. Right now it just leaks everything!
 + Increase the coverage of the standard library.
 + Fix, fix, fix bugs
 + Finally, resume work on the JIT!

Considering the above, my current strategy is to:

1. work on the interpreter until it is able to run more and more of the 'language' section of the official ECMAscript's [test262](https://github.com/tc39/test262) suite;

1. keep running it until the interpreter passes most tests (modulo exclusions that I will justify on a case-by-case basis).

The above was the primary motivation for finishing up the "ESM-lite" deal and the debugger. This being a "nights and weekends" sort of deal, I haven't been able to reach a satisfying level of productivity, but I'd say the situation is not too dire yet.  For the same reason, there is a non-zero chance that I'll just rewrite the entire roadmap or change my goals more or less radically. My feeling is that I'll stick with it, though.

So, this is it. I will write more posts, relating on the "interesting bits" of the code, and on the challenges that I face as I go along with it. I'm aiming at a frequency of 1 post per 1-2 weeks.  Looking forward to it!

