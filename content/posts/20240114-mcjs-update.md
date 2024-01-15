---
title: "mcjs Progress Update: December, January 2024"
date: 2024-01-14T18:52:01+01:00
draft: true
tags:
 - mcjs
---

To both of my readers: I hope you had a great time during the holidays, and that you feel re-energized and ready to take on the world.

I crossed the threshold of 2024 with some much needed extra family time, some rest from work, and most importantly, by cooking up some cool new stuff for mcjs!  (I know, I promised I would put out a new post every 2 weeks, but you know how it goes. I hope you two can forgive me. I promise, the next one will be on time!)

<!-- TODO check that this link actually works -->
([mcjs](mcjs-intro.md) is the toy JavaScript implementation I'm developing as a hobby/practice/fun side project.)

<!-- TODO Publish the test page as part of the blog and link it here. -->
The theme of this new stretch of JavaScript-induced masochism has been [test262, the ECMAScript Test Suite](https://github.com/tc39/test262). In particular, setting up a useful and nice-to-use test runner, and increasing the % of passing tests. The effort is still ongoing, but I'm happy with how things are unfolding. I also found myself grappling directly with some tricky parts of the language. No surprise there, but I believe there's value in jotting this down.

Note that right now I'm focusing on the `test/language/` part of the test suite, as I'm not trying to make a full industrial-strength JavaScript VM, and I'm still *very* far even from my modest goals. I implemented try-catch a couple weeks ago. There's no GC, I'm struggling with *var* and *function*. You get it. 

Finally, speaking of *var* and *function*, it became abundantly clear that that I had underestimated how much grief they would give me. That's going to be the target for the next efforts.

# Tools!

## Test262 runner

I got an actual test262 test runner!

It's really rough around the edges, but it allows me to:

- re-run all tests with a specific version of the VM
- analyze the test run data without re-running the tests every time
- check which test cases have been fixed or broken by a specific Git commit
- check progress across mcjs versions

Most importantly, it draws a line which the 3 of us can watch and wish for it to go up with our hearts full of hope and eagerness. It is also the only bit that has any chance of making any sort of impression, so here it goes in all its glory: [The Status Page](/mcjs-status.html).

### How it works

This is more of a note to myself, as I'm not going to write real documentation for the test runner, but I will for sure somehow need it. So here goes.

All of it happens in the `mcjs_test262` crate. It requires a local checkout of the [`test262`](https://github.com/tc39/test262) repo. The test runner system is basically made up of a linear pipeline of the following stages:

1. I run `scripts/run.sh`.
    - The program `mcjs_test262` gets the location of the test262 code and a list of test cases from the `scripts/tests.json` configuration file, performs the tests, and writes out an `out/runs.json` file which lists the outcome of each test case (one JSON document per line).

    - The `mcjs_vm` crate being tested is the one located "next to" the `mcjs_test262` directory. This is what happens with a regular checkout of the `mcjs` repo, so checking out different commits is the easiest way to test different VM versions. The commit ID is used as the version ID throughout the system.

2. I run `scripts/import_bytecode.rb`. This takes the `out/runs.json` file, and adds it into the `out/tests.db` SQLite database. 
    
    - This database file persists across multiple test runs and different checkouts of the repo (in fact, the whole `out/` directory is `.gitignore`'d). It contains data about *all* the tests run that I've been doing lately.

3. Then I may:
    - run `scripts/status.sh` to get a quick overview of how many tests pass/fail by category;
    - run `scripts/gen_status_page.rb` to re-generate The Status Page;
    - run `scripts/compare.rb` to check which test cases are being fixed/broken by the changes I've made, compared to the last commit.
    - if a test case fails, I can use `scripts/debug.sh` to re-run it in the debugger, with a configuration that exactly mirrors the test runner's.

That's pretty much it. The "data analysis" is done almost completely by SQLite, expressed in SQL. The shell/Ruby scripts are mostly there as a thin CLI layer.

## Debugger in egui

mcjs sports a debugger. Not a great one; it covers only the most basic functionality, but that's good enough for its designed purpose, which is to let me "see into" the VM well enough to understand why a particular piece of JS code is behaving in a particular way, or why a test is failing (or succeding) unexpectedly.

It used to be a web application. This sounds stupid, and to a significant degree, it is: I had to figure out how to fit every workflow into HATEOAS, I had to figure out how to achieve the layouts I wanted in HTML+CSS, and I had to figure out how to connect the necessarily single-threaded JS interpreter to the potentially many concurrent request the server would receive. I managed: I got really good mileage out of [Actix Web](https://actix.rs/), [htmx](https://htmx.org/), and [Alpine.js](https://alpinejs.dev/), and Rust made it possible to solve the concurrency problem once and for all.

In the end, the stupidity caught up to me. Every single feature I wanted to add required me to "refactor" the template until they fit the shape of the web routes; I struggled with htmx's attribute inheritance (although, in general, I loved it and would definitely use it again -together with Alpine.js- for my next web application). I had to do additional work in order to fit mcjs' data structures into a Serialize struct that can be used to fill templates in. I *love* the power and flexibility (and the simplicity, if you can believe it) of HTML and CSS when it comes to building crazy layouts, transitions, supporting multiple screen sizes, not to mention the ecosystem. But the extra work just wasn't worth it for a rugged, utilitarian tool that only exists to get a simple job done quick. 

So I switched to a normal GUI application.  The least painful GUI library that I know for Rust is [egui](https://www.egui.rs/), so I picked it. I'm not 100% "😍" with it since it's got an immediate mode API, but I gotta admit, I really like how it allows me to span the gamut from quick-and-dirty "just print this button" to a quite clean sort-of, if-you-squint-really-hard, [Elm Architecture](https://guide.elm-lang.org/architecture/) kinda thing.

I managed to bang out a working thing in a few hours (spanning a few days; it was Christmas!), and I'm quite satisfied by the result. In the beginning I kept running into stupid bugs related to the immediate-mode API: updating the underlying VM's state would invalidate part of the already-drawn frame. After a while, I figured out a proper architecture, and I've been happy enough since.

It looks alright, too:

![A screenshot of the mcjs debugger. It's divided in three panes: one for commands, one listing the current function's bytecode, and another showing the original JavaScript.](../20240114-mcjs-debugger.png)

## Simple CLI shell

This is the bit that makes mcjs feel like a real thing the most, more so than the step debugger or the test statistics. You can just type `mcjs` in the shell and it will spawn a prompt, and it will run your JS code! Just like Node.js!

```
sebastiano@fedora:~/Code/mcjs$ cargo run -p mcjs_vm --bin mcjs
    Finished dev [unoptimized + debuginfo] target(s) in 0.08s
     Running `target/debug/mcjs`
Welcome to mcjs.  It's got no version number yet.
Type some JavaScript, see what you can get away with.
> (function(x) { $print(`Hello, ${x}!`) })('you')
  "Hello, you!"
> $print(12 + 34)
Number(46.0)
> 
```

(`$print` is a built-in function that acts as a stopgap until I add `console.log`.) 

# A simpler "2 stacks" design

I figured that, if most of the performance is going to come from the JIT'ed code, and I'm not going to hand roll it in Assembly, the interpreter might as well be as simple as possible. Quite notably, the added simplicity becomes essential for the JIT (more on this later).

Prior to this change, the interpreter had a single stack, with the following layout (each line is 8 bytes, the machine integer):

```
  ---- { Frame header 
         ...
                        }              
       { capture[0]
         capture[1]
         ...
         capture[NC-1]  }
       {        arg[0]
                arg[1]
                   ...
             arg[NA-1]  }
       { reg[0]
         reg[1]
         ...
         reg[NR-1]      }
  ---- { Frame header 
              ...
                        }
       { capture[0]
         capture[1]
         ...
```

The pictured frame contains NC _captures_ (pointers to the values that this closure shares with other closures), NA function call arguments, NR virtual registers values, topped with a "frame header" that contains one-off info about the current stack frame such as a pointer to the current JS function, the instruction ID, the target register ID for the return value, etc.

Captures are actually already stored in the Closure structure. They're only copied to the stack as a form of caching. It would have been entirely possible to store a pointer to the closure in the frame header, and use that instead to get to the captures.

The numbers NC, NA, and NR are stored in the frame header, and they're potentially different for each frame. This implies that it's necessary to parse the frame header in order to know the total length of the frame, which in turn is necessary to know the offset of the next frame. Moreover, since each frame is a different size, we can't express this data type as simple Rust code; rather, I had to take a raw chunk of bytes (literally a `Vec<u8>`), and implement all the reading/writing at the right offsets. Although doable, you can see how this looks complicated already.

The new design is much simpler, and comes from 3 decisions:

1. Call arguments are now regular registers. The first 8 arguments are stored in the first 8 virtual registers of the stack.
    - The plan (as of yet unimplemented), is for further arguments to be stored on a separate structure on the heap. This makes sense, as it's quite uncommon; the typical number would be 0 - 5 arguments per call.

2. Captures, up to a limited number, can be stored in the frame header.
    - Again, the rest might go to a separate chunk on the heap, but another option is to pay for the extra indirection and get them from the called Closure.
    
3. Now that we're left with just frame headers and virtual registers, just store them in two separate stacks.

The code complexity savings come from the fact that virtual registers and frame headers have a fixed size, so they amount to two arrays (two regular, bog-standard `Vec`s, actually). No parsing of headers to get to the _n_-th value, no custom implementation of `struct` on top of `[u8]`.

Although I have suspended work on the JIT for the moment, the little experience I had there makes me hope in substantial simplification on that side as well. My reasoning is: upon exiting a trace, the interpreter's state needs to be patched so that, when the interpreter resumes, it finds its state "just as if" the JIT'ed code had run normally in the interpreter. This means that the trace has to write correct frame headers and register values. But the trace is written in machine code by the JIT compiler, and the JIT compiler is written by _me_, which means that this interpreter state patching had better be dumb, dead-simple, idiot-proof, or I will for sure screw it up.

Here's to simple designs and good guesses!

# JavaScript, y r u like this

As [Gary Berhardt](https://www.destroyallsoftware.com/talks/wat) taught us to expect, this month of work didn't fail to bring about a number of head-scratchers.

## Closures in loops

Take this test case from Test262: [let-closure-inside-condition.js](https://github.com/tc39/test262/blob/main/test/language/statements/let/syntax/let-closure-inside-condition.js), in particular this excerpt:

```js
let a = []
for (let i = 0; a.push(function () { return i; }), i < 5; ++i) { }
```

What's going on here? It's a for loop that repeats 5 times, and at each repetition, a new closure is appended to the Array `a`. Each of these closures captures the counter variable `i`. Fair enough. 

Quiz for those of you who are real JavaScript wizards: what happens if, afterwards, one calls the closures stored in `a`?

One would expect this code to be equivalent to the following:

```js
let a = []
let i = 0
while (1) {
    a.push(function () { return i })
    if (!(i < 5)) break
    ++i
}
```

Meaning, that calling the stored closures yields the following values:

```
sebastiano@fedora:~$ node
Welcome to Node.js v20.10.0.
Type ".help" for more information.
> /*  ... I typed the code above ... */
> a.map(f => f())
[ 5, 5, 5, 5, 5, 5 ]
> 
```

Alright, makes sense: every closure we create captures the same variable `i`. When we call each closure, they all go and read the same variable i, returning its value every time. That's 5, which is the value that made the loop stop. (There's an extra closure, but that's expected. It's due to `push` being called before evaluating the test expression `i < 5`.)

But that's not what happens in the original test case! Here's a snapshot of what I saw, the moment I loudly groaned:

```
> let a = []
undefined
> for (let i = 0; a.push(function () { return i; }), i < 5; ++i) { }
undefined
> a.map(f => f())
[ 0, 1, 2, 3, 4, 5 ]
> 
```

I found an explanation in a [blog post from 2ality] (paragraph "let in loop heads"): "In loops, you get a fresh binding for each iteration if you let-declare a variable."

I don't admire this bit of JS's design, but I'm guessing there are many cases where people find this behavior more intuitive.

Figuring out a fix was a bit challenging at first but, satisfyingly, I found a solution that's quite simple: I added an _Unshare_ bytecode instruction that ensures that a virtual register contains its value _inline_ rather than a reference to a capture. On the next iteration of the loop, upon constructing the closure, the _ClosureAddCapture_ instruction will make sure that the value is moved to a _new_ capture, just like it happened in the first iteration of the loop. The bytecode compiler tracks the variables captured in each scope; when compiling a loop, it just adds _Unshare_ instructions for the variables that were captured in that scope. Done!

<!-- TODO explain more? include debugger screenshots? -->

## eval

At some point, I noticed that a significant number of Test262 test cases rely on `eval` to check a correctness condition or exercise a particular behavior out of the interpreter. So I accepted that I had to implement `eval` somehow, at least a restricted version of it, if I aimed at passing most test cases.

We're talking about stuff like this (taken from [binding-tests-1.js](https://github.com/tc39/test262/blob/main/test/language/expressions/arrow-function/arrow/binding-tests-1.js)):

```js
function foo(){
    return eval("()=>this");
}
```

The string passed to `eval` is literal in this case, but there are many instances where it's a concatenation of literals and values only available at runtime. <!-- TODO provide a specific example here? --> 

Most of the trouble with `eval` is that the code that it's going to execute is both (1) only discovered at runtime, when the "call" to eval happens; (2) has to be able to refer to variables from the surrounding scopes by name. Bytecode compilation allocates virtual registers to local variables and "lowers" local variable names to references to virtual registers. So how would our `eval` implementation work?

Thinking out loud: it would have to compile the newly discovered code to bytecode at the time of the call, in preparation for the interpreter to consume it later.  In the above example, the reference to `this` would have to be compiled to a reference to the same register used by the surrounding function for `this`. I could keep a mapping around... sounds fairly doable, but in order to access the same registers, I will have to "add" code to the same function, because a separate function wouldn't be able to access the same registers. It could also work for assignments.

Oh, but what if eval introduces a write to a variable captured from further away in the context? Like this:

```js
function makeBox() {
    let x = 22
    function setX(y) {
        eval('x = ' + y);
    }
    function getX() { return x }
    return { setX, getX }
}

const box = makeBox()
box(33)
box(44)
```

In the above example, the `eval` code wants to capture `x`. But during the compilation of this script, `setX` did _not_ look like it captured any variables! So the compiler did not map the identifier `x` to anything. We would have to add more stuff to track the mapping between identifiers and registers in all lexical scopes, so that such a far-reaching capture could be resolved at runtime (rather than at bytecode-compile time). Some such work has been done to allow the debugger to show variable names, but this would be more complicated.

Not tired yet? Here's another example:

```js
function Thing(idExpr) {
   eval('function f() { return ' + idExpr + '; }')
   this.id = f();
}

const t = new Thing(123)
console.log(t)
```

In non-strict mode, `eval` is allowed to introduce new variables to the scope! That means that all of the compiler's bets are off: any variable that looks undeclared or global might at some point become a well defined local variable later, if the right string is passed to eval. That makes the produced bytecode wrong.

Bleargh.  Strict mode introduces some restrictions that remove some of the pain (see [MDN: Strict mode](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode), heading _Non-leaking eval_), but it seems to me nevertheless that `eval` is already largely frowned upon in the JS community, it's a pain in the ass to implement, and the test cases from Test262 that use it can just be patched or discarded.

So, in the trash it goes. `eval` is evil, and it won't be in mcjs for the foreseeable future. 


## Non-strict mode

In a bout of magnanimity towards the history of JavaScript's semantics, I decided that it would be cool to try and implement compliant non-strict mode.

It's a decision that I'm ready to revert at any moment, because I see non-strict mode as purely legacy, a mistake of youth. I like the idea of a "strict-only", "let's learn from our past", not clean but clean-_er_ slate JavaScript implementation. But I also believe that it would be satisfying to see that I _can_, in fact, implement a weird little system like the scoping rules of young wild and free JavaScript.

For now I have only implemented ["this substitution"](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode#no_this_substitution) and a few other minor things here and there.v It's giving me just a little grief, but let's see how it goes.


# Little bits and bobs

Just for the record, I also managed to get a bunch of little features in. Here's a list and a little demo based using the new CLI shell:

- try/catch: basic exception handling kinda works now
- Coerce to string when doing string concatenation
- Object comparison by pointer

```
sebastiano@fedora:~/Code/mcjs$ rlwrap cargo run -p mcjs_vm --bin mcjs
    Finished dev [unoptimized + debuginfo] target(s) in 0.07s
     Running `target/debug/mcjs`
Welcome to mcjs.  It's got no version number yet.
Type some JavaScript, see what you can get away with.
>  try { throw 123; } catch (x) { $print("exception! " + x) }
  "exception! 123"
> $print("I got " + 99 + " problems, but eval ain't one")
  "I got 99 problems, but eval ain't one"
> var a = {x: 1, y: 2};   $print(a == a)
Bool(true)
```

There is more, but I managed to break it again in the time it took me to write this post. Oh, well.

# The current dragon: scoping rules

Scoping rules. I didn't remember them being so hard.

Did you know?

- `function` and `var` declarations with the same name can coexist in the same scope, but independently of their order, `var` always shadows the `function`.

- If a `function` declaration appears in a block _in strict mode_, then it's just block-scoped, just like `let`/`const`. In non-strict mode it's scoped to the containing function but it won't get shadowed by a `var` appearing in any outer scope.

- `let`/`const` declarations _are_ hoisted to the beginning of their block, but the variables always start as 'uninitialized'. For each variable, the time that passes between the beginning of the scope and its initialization is called its "temporal dead zone" (TDZ). Still a `ReferenceError`, but with a different error message.

- `function` declarations are hoisted to the beginning of the block/function scope _together with the assignment_, of the function/closure to the variable/virtual register. That means that at the time of that assignment it must be possible to capture any other variable that's visible from that scope, even if they're still in the TDZ. Gotta be careful to do all these types of hoisting in the right order!

- `var`'s and `function`s at the top-level scope of a _script_ (not a module) become properites of `globalThis`

I'm just having a hard time keeping all these rules straight in my head, and producing implementation code that complies with them all, while preventing it from becoming an indecipherable bowl of spaghetti.

My approach right now is to resolve all of this in the bytecode compiler, so that the bytecode only refers directly to virtual registers. With this idea, the bytecode compiler will:

- logically separate each declaration into:
    - a 'namedef' phase where the name is allocated to a vreg;
    - an 'assignment' where the virtual register is written with a value (and exits the TDZ)

- do multiple passes
    - the first pass does all the namedef's (in the necessary order to give all the guarantees above
    - the second pass "draws the rest of the owl" (compiles all statements and expression, including the 'assignment' parts as per the logical distinction drawn above).

That's where my effort will be focused on in the next few weeks.

That's it for this post.  What do you think? Interested? Could my writing improve? Do you _love_ JavaScript and want to lecture me on it? Do you think I should rewrite it in Rust? Do you think I'm just a moron and I should focus on shipping code to real users? Can't wait to hear about it! Contact me on [Twitter](https://twitter.com/seb_barrera) (will never call it X), or via any of the contacts you find in ["about"](/about). Looking forward to it!


