---
title: "mcjs Progress Update: May 2024"
date: 2024-05-05T16:32:48+02:00
---

Good evening and welcome back, to another installment of "What's new in [mcjs](https://github.com/sebastiano-barrera/mcjs/)".  Later than promised, but
rich in good news!

(Formally, I blame the delay on an unplanned -- but not unwelcome! -- job change, but
let's not kid ourselves. This is my emotional support side project, I have no intention of
sticking to any sort of roadmap or schedule. Down with project management!)

# Scoping rules

![Screenshot from Sekiro, "Shinobi Execution" message laid over a view of the Guardian Ape, apparently defeated.](../20240505--shinobi-execution.jpg)

_(Source: https://www.reddit.com/r/Sekiro/comments/10mghn6/guys_i_just_beat_the_guardian_ape111 )_

This was called the "next dragon" in the last post, and I have very good news about it:

```
Group                                                                  % Passing
-------------------------------------------------------------------  -----------
test/language/block-scope/leave                                            100.0
test/language/block-scope/return-from                                      100.0
test/language/block-scope/shadowing                                        100.0
test/language/block-scope/syntax/for-in                                    100.0
test/language/block-scope/syntax/function-declarations                     100.0
test/language/block-scope/syntax/redeclaration                             100.0
test/language/block-scope/syntax/redeclaration-global                      100.0
```

<!-- TODO Are there more groups that are relevant? -->

All tests in the `block-scope` group pass! The dragon is slain! (Fingers crossed; things generally work as they should, but there
are a few other tests that exercise scoping rules, and I didn't pick that particular boss
out of all those in Sekiro for no reason... 😉)

The most important part of the solution to this problem is a new intermediate
representation named the "PAST".

## PAST: the Pre-bytecode AST

There are 4 types of declaration in JavaScript, each introduced by a different keyword:
`let`, `const`, `var` and `function`.

`var` is considered legacy, largely due to its behavior being unintuitive in certain respects. In
particular, it forces you to extend the scope of the variable to the whole enclosing
function, even the part of it _before_ the line where the declaration appears! `function`
behaves similarly, but functions are always expected to be somewhat "special" compared to
variables, which makes its behavior "better accepted" by the typical programmer.

```js
// this works!
console.log(f(3, 5));  // 8

// `x` exists, even though it's declared later (but it's undefined)
console.log(x);        // undefined

function f(a, b) {
  return a + b;
}

var x = 5;
console.log(x);        // 5
```

Also, `var` and `function` declarations found at the toplevel scope of a script are
automatically turned into properties of the global object (i.e. `window`, `globalThis`),
so they may "escape containment" and be accessed by any other part of the program.

Nowadays, everyone is encouraged to use `let` and `const` instead, which enjoy the more
obvious behavior of lexical scoping.  These declarations are valid "from now on", until
the end of the enclosing block, just like in most other programming languages:

```js
console.log(x);  // => Uncaught ReferenceError: x is not defined
{
  console.log(x);  // => Uncaught ReferenceError: Cannot access 'x' before initialization
  let x = 5;
  console.log(x);  // => 5
}
```

This is great, but it poses a challenge to us implementers: all 4 declaration types must
be able to coexist in the same script/module, each with their behavior. Precise rules for
their coexistence have been laid out, but they are, by their nature, somewhat convoluted and
unintuitive. If you want an introduction, the MDN articles for
[function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function#redeclarations)
and
[let](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let)
(focus on "Description") are pretty good and will save you a bunch of midnight oil,
otherwise burnt studying the ECMAScript standard.

My hair has already started thinning, so in the interest of not making a bad situation
worse, I realized I had to pick an approach that makes these 'interactions' manageable.

My choice turned out to be successful. I **"broke down" the behavior** of the 4
declaration types **into "elements"** that can then be processed separately and
independently by the bytecode compiler. In broad strokes, each declaration is now
described in terms of:

- what happens when there is a `"use strict"` directive (and when there isn't);

- what is their "visibility scope" (e.g. enclosing function, enclosing block);

- under which conditions redeclarations are allowed;

- what happens if the declaration is at the script or module's toplevel.

Once the declaration is understood in this form, the bytecode compiler to make sure that the visibility
scope is actually honored during name resolution (i.e. 'widened' whenever necessary), and
that any forbidden redeclarations are caught and prevented. I'm convinced that this
basically *requires* the program to *store* the declarations in this broken-down form. This is the
chief reason that a **new intermediate representation** needed to be introduced in the
bytecode compiler.

This new IR, named "PAST" (for _Pre-bytecode AST_, it's temporary, I know it sucks, don't
@ me) sits between the AST (a direct representation of the JS source code) and the
bytecode (directly executable by the interpreter).

I don't like to keep things too abstract for too long, so let's start by looking at an
example of a typical PAST fragment (I know, this textual representation makes me want to
gouge my eyes out, too... consider it programmer art):

```
PAST: [814-980] func () unbound[$print, sayHello, people] Sloppy block0 {
  decls:
    Decl: tmp2 = TDZ [conflicting]
  fn asmts:
    sayHello <- e0
  exprs:
    e0: CreateClosure:
      [814-871] func (whom) unbound[$print] Sloppy block1 {
        decls:
        fn asmts:
        exprs:
          e0: Read($print)
          e1: StringLiteral(Atom('Hello, ' type=inline))
          e2: Read(whom)
          e3: Binary("+", e1, e2)
          e4: Call { callee: e0, args: [e3] }
        stmts:
          [844-869] Assign(None, e4)
      }
    e1: Read(tmp2)
    e2: StringLiteral(Atom('Alice' type=inline))
    e3: StringLiteral(Atom('Bob' type=inline))
    e4: StringLiteral(Atom('Charles' type=inline))
  stmts:
    [886-913] Assign(Some(tmp2), eArrayCreate)
    [886-913] ArrayPush(tmp2, e2)
    [886-913] ArrayPush(tmp2, e3)
    [886-913] ArrayPush(tmp2, e4)
    [873-913] Assign(Some(people), e1)
    block3 {
      decls:
        Decl: i = TDZ [conflicting]
      fn asmts:
      exprs:
        e0: NumberLiteral(0.0)
      stmts:
        [914-980] Assign(Some(i), e0)
        block4 {
          decls:
          fn asmts:
          exprs:
            e0: Read(i)
            e1: Read(people)
            e2: StringLiteral(Atom('length' type=static))
            e3: ObjectGet { obj: e1, key: e2 }
            e4: Binary("<", e0, e3)
            e5: NumberLiteral(1.0)
            e6: Binary("+", e0, e5)
          stmts:
            [914-980] IfNot { test: e4 }
            [914-980] Break(block3)
            block5 {
              decls:
              fn asmts:
              exprs:
              stmts:
                block6 {
                  decls:
                  fn asmts:
                  exprs:
                    e0: Read(sayHello)
                    e1: Read(people)
                    e2: Read(i)
                    e3: ObjectGet { obj: e1, key: e2 }
                    e4: Call { callee: e0, args: [e3] }
                  stmts:
                    [958-978] Assign(None, e4)
                }
            }
            [947-950] Assign(Some(i), e6)
            [914-980] Assign(None, e6)
            [914-980] Unshare(i)
            [914-980] Jump(StmtID(0, block4))
        }
    }
}
```

It's the result of the compilation of this piece of really dumb code:

```js
function sayHello(whom) {
    $print("Hello, " + whom);
}

var people = ['Alice', 'Bob', 'Charles']
for (let i=0; i < people.length; ++i) {
    sayHello(people[i]);
}
```

It has the following characteristics:

- It distinguishes between **statements**, **expressions**, and **declarations**.

    - New variable names can only be introduced explicitly by declarations.
    
    - Expressions are "inert": they represent the _syntax_ of an expression but do not
      imply its evaluation.
    
    - Statements are the only thing that "happens" in the block. They can refer to
      expressions; when they do, executing the statement includes evaluating those
      expressions.  Each distinct reference to the same expression imples a distinct
      evaluation.

- Statements, expressions, and declarations are grouped into self-contained **blocks**,
  which also track the set of unresolved names.

    - A block is formed by 4 distinct sections, which should be conceptually understood in
    this order: variable declarations; function declarations; expressions; statements.
    (Function declarations are separate because it happens to make it easier to implement
    the semantics of `function` declarations; consider it an implementation detail.)

- **No name resolution.** Variables are always designated by their original name, exactly
  as in the source JavaScript code. Names *remain* unresolved throughout the whole PAST
  generation phase.  Anonymous variables (in a separate namespace) can be introduced to
  represent temporary values (typically to reuse the result of an expression evaluation).

- A **block can be inserted into another** block as a statement.

    - This is an important basic operation in the PAST generation algorithm. As a part of
    it, declarations from the inner block are "merged" into the outer block following an
    algorithm that honors JavaScript's rules. Afterwards, any redeclarations appearing as
    a result are checked for correctness.

While we're here, I should explain why it's important that the redeclaration check happens
at every block merge. The JavaScript standard wants us to check that certain declarations
(`let`, `const`; let's call them "exclusive declarations") do not conflict with _any_
other declaration that is visible in the same scope. A conflict is when multiple
declarations try to define the same name. There are a number of factors that make this
uncannily hard:

  - `var` and `function` are automatically **hoisted** (that's a technical term for 'moved
    upwards') to the beginning of the function scope. 
    
  - potentially conflicting exclusive and hoisted declarations may be discovered by the
    compiler in any order.
    
  - `var` and `function` are not exclusive; they admit redeclaration (almost) freely.

I tried several approaches, and I kept getting a bunch of cases wrong every time, until I
discovered the current algorithm, where `var` and `function` declarations bubble up the
chain of blocks (sit down, Satoshi) and redeclarations are checked at every step. This
way:

  - `var` and `function` get hoisted, but there are multiple chances to check for
    conflicts before they reach their "destination" at the top of the function scope.
    
  - All discovery orders are served correctly: exclusive declarations are checked
    immediately against previously discovered declarations; hoisted declarations always
    encounter their conflicting exclusive declarations at some point on their way to their
    destination.
    
  - After "surviving" all necessary rounds of redeclarations checking, conflicting `var`
    and `function` declarations are deduplicated so as to result in a single name
    definition, as per JavaScript specs.

Neat.

What did we gain by all this? Well, once all this work is done, the semantics of variable
declaration and reference in the resulting PAST are extremely simple: variables *must* be
declared with their name prior to usage; declarations are *only* valid in their block (and
children, of course).

Because variables remain referred to by name in this representation, we can delay the
allocation of virtual VM registers and the name resolution phase entirely after the PAST
is fully generated.  And because of the PAST's simple semantics, these two tasks are very
simple. Yay!

# Stackless interpreter design

For a pretty good chunk of the last few months, I thought I had done a good thing when I
changed the interpreter's design to a _stackful_ one. I have since seen the error of my
ways, and have come back to a good and right _stackless_ design.

Now, I'm going to explain what I mean by those words, which I hope aligns well with how
they are broadly understood in the field. I haven't cared much for using them precisely
until this very moment, right here at my desk while I type this blog post, so feel
free to correct me. I'll be glad to fix my understanding and this post, too.

As you know, when your JavaScript program runs and calls functions, there is a _call
stack_ that keeps track of which calls have been started and are yet to finish. It tracks
where the execution will have to return to, and the value of each local variable. The
interpreter itself, like any other Rust program, is also made out of functions that are
called and return, and as such it also has its own stack.  We'll call these the _JS stack_
and the _native stack_ respectively.

In a _stackful_ interpreter, each JS stack frame corresponds to exactly 1 native stack
frame.  In other words, in order to call (and run) a JS function, the interpreter makes a
call to an internal function of its own. For as long as mcjs was stackful, mcjs' version
of this function was called `run_frame`. When it's time for the JS function to return,
`run_frame` also returns.

In a _stackless_ interpreter, the two stacks are fully independent. In order to perform a
JS call, a typical stackless interpreter will apply some changes to some internal data
structure that represents the JS stack.  When the interpreter's main loop continues, it
will unknowingly find itself working with another JS stack frame, maybe another JS
function entirely, and another set of local variables, simply as a consequence of
`stack.top_frame()` (or whatever) returning a pointer to a different stack frame.

I had originally switched to the stackful design because I thought I could:

- Make it clearer which variables are borrowed for the full duration of a stack frame, and
  which need to be dropped and fetched again from the interpreter's state structure (the
  Rust→C translation of this is that it would make it less likely to accidentally keep a
  pointer to the wrong stack frame around);

- Make it simpler to have temporary data that only exists for the duration of a stack
  frame, without making additions to the JS stack frame itself.

- Make it simpler to implement suspend/resume of the whole interpreter (which is
  fundamental for the debugger);

- Make it simpler to implement exception handling.

  - Stack unwinding maps quite neatly to the interpreter's stack: an exception simply
    becomes a particular type of return value. Then the caller frame can check whether or
    not it has a handler for it, and decide to jump to it or return the exception again
    further. If the exception is unhandled, the exception bubbles up all the way to a
    wrapper function that transforms it into an error message and that's it.
    
These points are not necessarily that stupid, and they would even be _decent_ if it wasn't
so easy to maintain all these advantages with a stackless design, and if the stackful
design didn't make so many things harder.

Among these harder things is maintaining the state of the JS stack between a suspend and
the following resume. This is necessary for the debugger to be of any use (think about it
this way: the debugger is a GUI editor of the interpreter's state; it's useless if the
interpreter state must be reset to "zero" before the GUI can even be shown). In Rust
terms, this use case can be stated as "both the interpreter and the debugger want a `&mut`
to the interpreter's data". As a consequence, the interpreter must relinquish its `&mut`
access to the debugger, and it will later get it back to resume execution.

Now, if you're the interpreter and you're asked to resume execution, you're going to be
given a JavaScript stack of a number of frames. As we said, each of these has to correspond 1:1 with a
recursive call to `run_frame`. As far as I can tell, all you can do is what I call "stack
climbing". In short, `run_frame` is designed so that (only when resuming!) it starts by
calling itself again for the stack frame directly above until the 1:1 correspondence is
re-established.  Then, as the top frame returns, each of the lower `run_frame` instances resume execution normally. Workable but... bleh.

As I charted the way to certain features (primarily generators), I started eyeing a stackless
design again.  After thinking about it and doing some practical experiments, it turns out
that with a stackless design:

- It's still clear what is borrowed when.

    - The interpreter's main loop now has two nested labeled `loop {}` statement that make
      it easy to do the "drop & fetch again" move I mentioned above. Just `continue
      'reborrow`. (Terrible name, I know. I'm sure you're starting to see the pattern.)

- It's true that it's hard to have temporary data outside of the interpreter's stack data
  structure, but it turns out that it's not very useful, nor a particularly good
  idea. Much better to do things properly and put things where they belong.
  
- Suspending and resuming the interpreter is not harder.

    - The hard part is actually making sure that the interpreter is left in a "resumable"
      state after suspending, and that execution starts from the correct instruction upon
      resuming. Being stackful doesn't really help here.

    - Guaranteeing that execution continues properly requires doing nothing. The stack is
      still there!

- Exception handling is still OK.

    - Upon throwing an exception, the handler to be used is searched in the stack. When
      it's found, an appropriate number of stack frames are popped, the instruction
      pointer is made to point to the handler and execution resumes regularly. That's it.
      
- It opens up a cool strategy for implementing generators.  See the details in the next
  paragraph.

Overall, I'm happier with things this way.

On this topic, some _really_ good points were made in this excellent blog post, which I
can recommend wholeheartedly: [Piccolo - A Stackless Lua
Interpreter](https://kyju.org/blog/piccolo-a-stackless-lua-interpreter/).

# Generators & Iterator Protocol

This is actually a fairly recent development, which I only undertook because I wanted to
increase the fraction of [`JSON5`](https://github.com/json5/json5) that runs under mcjs.

If you want to learn what a generator is, see [this MDN
article](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator). It's
this type of thing:

```js
function* gimmeSomeNumbers() {
    let i = 0;
    yield ++i;
    yield ++i;
}
```

JavaScript's generators are, in PLT parlance, _asymmetric stackless coroutines_.
- _asymmetric_ because they can only yield to their caller;
- _stackless_ because they can't implicitly yield to their caller by calling another
  coroutine (in other words a `yield` statement can only affect the current call/stack frame);
- _coroutines_ because they are functions that can suspend themselves and be resumed.

I realized that one useful way of thinking of them is as a mechanism for _saving and
restoring a single stack frame_. Thanks to their being stackless, there is never any need
to ever look at any other part of the stack.

Here, too, I found myself choosing among multiple implementation approaches:

1. **"Compiler trick".** Basically consists in transforming the generator into a regular
   function that uses a state machine to "jump back where it left". This is the approach
   used by Facebook's [regenerator](https://facebook.github.io/regenerator/) and by
   Babel.js (in order to make generators available to pre-ES6 interpreters).

2. **The literal way.** One can add a dedicated bytecode instruction that causes the
   interpreter to literally save the stack frame (in an appropriate form) and restore it
   later.

Approach #1 is very popular, and it has a very significant advantage: once you perform
this transformation (fully contained within bytecode compilation), you're done: everything
works exactly as for regular functions. The VM does not have to know anything about
generators at all. The disadvantage is that I found the transformation to not be quite so
simple; for example, local variables have to be turned into properties of a dedicated
object (or their state won't be persisted across a yield).

I might be wrong, but I'm very satisfied with having followed approach #2.  Stack frames
have a very simple representation at runtime, and inventing an alternative form from which
it can be restored wasn't hard.

(Calling it now: I suspect I'm going to eat my hat once I start implementing a GC
seriously. That's how I conduct my life: I bet against myself, so I _always_ win
_somewhere_.)

Here's a subtle point that wasn't obvious to me prior to implementing generators: the
coroutines that power generators are invisible in JavaScript code. You can't give them a
name, and there is no syntax for them. You can only have generator _functions_ (that's
what `gimmeSomeNumbers` is). When called, they return an object whose `next` property
starts or advancesf the internal coroutine until it yields, and returns the yielded value.

That's literally what _mcjs_'s bytecode compiler does: it transforms generator functions
into functions that return an object with a `next` yadda, yadda, yadda, and this has a
cool property: the returned object implicitly adheres to the [**iterator
protocol**](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols#the_iterator_protocol),
which is going to be the basis of the [_for(... of
...)_](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...of)
syntax.  Neat! That's going to be next.

# Debugger

The debugger has seen *significant* additions. See for yourself:

![Screenshot of the mcjs debugger, mcjs_tools. Shows some instructions on the left, and
some source code on the right.](../20240505--debugger.png)

Some notable features:

- When you hover a value (e.g. instruction argument), other references to the same object
  or VM register are also highlighted. This makes it easy to understand which instruction
  last assigned the value and/or where the same register or object is going to be
  accessed.

- The UI layout is fully customizable, Visual Studio style. Quite easy to implement, all
  thanks to the fantastic [egui_tiles](https://docs.rs/egui_tiles/latest/egui_tiles/)! The
  layout can also be saved/restored in case it benefits a particular debugging "campaign".
  
- Thanks to the changes to the interpreter's API that came along with the switch to the
  stackless design, there is no longer any need for `Pin` or unsafe code, which really
  makes me feel much more at ease when developing new features.

- When stepping through instructions, you can decide to follow the interpreter into calls
  ("Into") or wait until the call returns ("Next").
  
- The interpreter can break on ony throw instruction, and/or only on those that result in
  an unhandled exception.

- Unhandled exceptions are easier to work with:
  - the full stack trace is printed, with filenames and line numbers;
  - dedicated commands allow to sat a breakpoint at the throwing instruction, and/or
    enable "break on throw".
  
# Future

<!--

TODO:

    - generators, iteration
        - for (... of ...)

    - roadmap
        - just how much of test262 do you want to implement before you switch to the JIT, goddamit?!

-->

The level of test262 coverage seems to be pretty alright right now. I'll take a look at
the currently failing tests for smaller features that are worth it to work on (some
arithmetic and logic operators are still unimplemented), but I don't expect to see any big
one.

That means it's time to put on the big boy pants and resume work on the JIT, the one
_actual_ compiler that mcjs is going to have.  I can't wait!

Also, it turns out that I *can't* keep a regular pace in writing these posts, so no more
promises now. You two are getting this post whenever I feel like it.

Okay, maybe _one_ promise: I'm going to update the "line go up" ("Are we
ECMAScript yet?") page soon. I just haven't got round to do it, unfortunately.
Bye!

