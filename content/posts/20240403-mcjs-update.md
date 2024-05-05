---
title: "mcjs Progress Update: May 2024"
date: 2024-04-03T16:32:48+02:00
draft: true
---

Welcome back, to another installment of "What's new in mcjs".  Later than promised, but
rich in good news!

(Formally, I blame the delay on an unplanned -- but not unwelcome! -- job change, but
let's not kid ourselves. This is my emotional support side project, I have no intention of
sticking to any sort of roadmap or schedule. That's the neat part!)

# Scoping rules

![Screenshot from Sekiro, "Shinobi Execution" message laid over a view of the Guardian Ape, apparently defeated.](../20240505--shinobi-execution.jpg)

_(Credit: https://www.reddit.com/r/Sekiro/comments/10mghn6/guys_i_just_beat_the_guardian_ape111 )_

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

All tests in the `block-scope` group pass! The dragon is slain! (Fingers crossed; I didn't
pick that particular boss out of all those in Sekiro for no reason... 😉)

The most important part of the solution to this problem is a new intermediate
representation named the "PAST".

## PAST: the Pre-bytecode AST

There are 4 types of declaration in JavaScript, each introduced by a different keyword:
`let`, `const`, `var` and `function`.

`var` is considered legacy due to some of its behavior being quite unintuitive. In
particular, it forces you to extend the scope of a variable to the whole enclosing
function, even the part of it _before_ the line where the declaration appears! `function`
behaves similarly, but functions are always expected to be somewhat "more special" than
variables, which makes its behavior "better accepted" by the typical programmer. Nowadays,
everyone is encouraged to use `let` and `const` instead, which enjoy the more "obvious"
behavior of lexical scoping.  These declarations are valid "from now on", until the end of
the enclosing block, just like in most programming languages.

This is great, but it poses a challenge to us implementors: all 4 declaration types must
be able to coexist in the same script/module, each with their behavior. The rules for the
coexistance of are somewhat convoluted and unintuitive. Technically, one would have to
read the relevant chapters of the ECMAScript standard to understand them fully, but I've
found the MDN articles for
[function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function#redeclarations)
and
[let](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let)
(especially under the "Description" heading) to serve the purpose better.

My hair has already started thinning, so in the interest of not making a bad situation
worse, I realized I had to pick an approach that makes these 'interactions' manageable.

My choice turned out to be successful. I **"broke down" the behavior** of the 4
declaration types **into "elements"** that can then be processed separately and
independently by the bytecode compiler. In broad strokes, each declaration is now
described in terms of:

- what happens when there is a `"use strict"` directive (or when there isn't);

- what is their "visibility scope";

    - `var` is visible throughout the whole enclosing function;
    - `let`/`const` only in the smallest enclosing block;
    - for `function`, it depends on whether we're in `"use strict"`, and there are even
      some differences between implementations!

- under which conditions redeclarations are allowed;

- what happens if the declaration is at the script or module's toplevel.

Once the declaration is understood in this form, I had to make sure that the visibility
scope was actually honored during name resolution (i.e. 'widen' it whenever necessary),
and that any forbidden redeclarations were caught and prevented. I'm convinced that this
basically *requires* the program to *store* the declarations in this new form. The
consequence is that a **new intermediate representation** needed to be introduced in the
bytecode compiler.

This new IR, named "PAST" (for _Pre-bytecode AST_, it's temporary, I know it sucks, don't
@ me) sits between the AST (a direct representation of JavaScript source code) and the
executable bytecode. It has the following characteristics:

- It distinguishes between **statements**, **expressions**, and **declarations**.

   - New valid variable names can only be introduced by a declaration.

- **No name resolution.** It keeps referring to variables by their original name, exactly
  as in the source JavaScript code. Names *remain* unresolved throughout the whole PAST
  generation phase.  Anonymous variables (in a separate namespace) are also introduced to
  represent temporary values.

- It groups statements, expressions, and declarations into self-contained **blocks**,
  which also track the set of unresolved names.
  
- A **block can be 'nested'** as a statement within another block.

    - In this case, declarations from the inner block are "merged" into the outer block
    following an algorithm that honors JavaScript's rules. Among other things, we do so
    by checking redeclarations at each merge.

Note that, once fully formed, the semantics of variable declaration and reference are
extremely simple: variables *must* be declared with their name prior to usage;
declarations are *only* valid in their block (and children, of course).

Because variables remain referred to by name in this representation, we can delay the
allocation of virtual VM registers and the name resolution phase entirely after the PAST
is fully generated.  And because of the PAST's simple semantics, these two tasks are very
simple. Yay!

<!---
 TODO Add example of PAST
--->

# Stackless interpreter design

For a pretty good chunk of the last few months, I thought I had done a good thing when I
changed the interpreter's design to a _stackful_ one. I have since seen the error of my
ways, and have come back to a good and right _stackless_ design.

Now, I'm going to explain what I mean by those words, which I hope aligns well with how
they are broadly understood in the field. I haven't cared much for using them precisely
until right now, when I made myself write this post, so feel free to correct me. I'll be
glad to fix my understanding and this post, too.

As you know, when your JavaScript program runs and calls functions, there is a call stack
that keeps track of where to return to when each function runs through, which also
includes local variables.  But the interpreter itself is also made out of functions, which
are called and return, and as such it also has its own stack.  We'll call these the _JS
stack_ and the _native stack_ respectively.

In a _stackful_ interpreter, each JS stack frame corresponds to exactly 1 native stack
frame.  In other words, in order to call (and run) a JS function, the interpreter makes a
call to an internal function of its own (maybe it's called `run_js_function`). And when
`run_js_function` returns, the JS function can also be said to be returning.

In a _stackless_ interpreter, the two stacks are fully independent. Looking again into the
case of a JS call, a typical stackless interpreter will apply some changes to some
internal data structures (tracking the JS stack).  When the interpreter's main loop
continues, it will unknowingly find itself working with another JS stack frame, maybe
another JS function entirely, and another set of local variables, simply as a consequence
of the JS stack's "top frame" pointer pointing to the new stack frame.

I had originally switched to the stackful design because I thought I could:

- Make it clearer which variables are borrowed for the full duration of a stack frame, and
  which need to be dropped and fetched again from the interpreter's state structure
  (`mcjs_vm::interpreter::stack::InterpreterData`, in case you're following along at
  home);

- Make it simpler to have temporary data that only exists for the duration of a stack
  frame, without making additions to the JS stack frame itself.

- Make it simpler to implement suspend/resume of the whole interpreter (which is
  fundamental for the debugger);

- Make it simpler to implement exception handling.

  - Stack unwinding maps quite neatly to the interpreter's stack: an exception simply
    becomes a particular type of return value; then the caller frame can check whether or
    not it has a handler for it, or return it further. The final caller is a wrapper
    function that transforms it into an error message and that's it.
    
These points aren't so bad, and they would even be good if it wasn't so easy to still
maintain these advantages with a stackless design, and the stackful design didn't make
certain things so much harder.

One requirement that is somewhat harder to satisfy with a stackful design is to maintain
the state of stack data structure between a suspend and the following resume. This is
necessary in order for the debugger to be of any use. In Rust terms, both the interpreter
and the debugger want a `&mut` to the interpreter's data. The interpreter must therefore
relinquish its `&mut` access to the debugger, and then it has to get it back and resume
execution later.

In _mcjs_, the interpreter's main function was named `run_frame`.  If you have _N_
JavaScript stack frames at the time of suspension, and each corresponds 1:1 with a
recursive call to `run_frame`, then you can only do what I call "stack climbing": you
start by calling `run_frame` for the first stack frame, and `run_frame` is designed so
that (only when resuming!)  it calls itself again for the stack frame directly above until
the 1:1 correspondence is re-established.  Bleh.

So I restored and improved the previous stackless design. Now:

- It's still clear what is borrowed when.

    - The interpreter's main loop now has two nested labeled `loop {}` statement that make
      it easy to do the "drop & fetch again" move I mentioned above.

- It's hard to have temporary data outside of the interpreter's stack data structure, but
  it turns out that it's not very useful. Much better to do things properly and put things
  where they belong.
  
- Suspending and resuming the interpreter is not harder.

    - The hard part is actually making sure that the interpreter is left in a "resumable"
      state after suspending, and that execution starts from the correct instruction upon
      resuming. Being stackful doesn't really help here.

    - Guaranteeing that execution continues properly requires doing nothing. The stack is
      still there!

- Exception handling is still OK

    - Upon throwing an exception, the handler to be used is searched in the stack. When
      it's found, an appropriate number of stack frames is popped, the instruction pointer
      is made to point to the handler and execution resumes. That's it.
      
- It opens up a cool strategy for implementing generators.  See the details in the next
  paragraph.

Overall, I'm happier with things this way.

On this topic, some _really_ good points were made in the following blog post: [Piccolo -
A Stackless Lua Interpreter](https://kyju.org/blog/piccolo-a-stackless-lua-interpreter/).

# Generators & Iterator Protocol

This is actually a fairly recent development, which I only undertook because I wanted to
increase the fraction of [`JSON5`](https://github.com/json5/json5) that runs under mcjs.

If you want to learn what a generator is, see [this MDN
article](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Generator).

JavaScript's generators are, in PLT parlance, _asymmetric stackless coroutines_.
- _asymmetric_ because they can only yield to their caller;
- _stackless_ because they can't implicitly yield to their caller by calling another coroutine;
- _coroutines_ because they are functions that can suspend themselves and be resumed.

One useful way of thinking of them is as a _single_ stack frame that can be _saved and
restored_. They're stackless, so the state is fully contained within a single stack frame,
and there is never any need to ever act on multiple stack frames.

Again, I found myself choosing between multiple approaches to implement this stack frame
"save/resume" function:

1. **"Compiler trick".** Basically consists in transforming the generator into a regular
   function that uses a state machine to "know where to resume". This is the approach used
   by Facebook's [regenerator](https://facebook.github.io/regenerator/) and by Babel.js
   (in order to make generators available to pre-ES6 interpreters).

2. **The literal way.** One can add a dedicated bytecode instruction that causes the
   interpreter to literally save the stack frame (in an appropriate form) and restore it
   later.

Approach #1 is very popular, and it has a very significant advantage: once you perform
this transformation (fully contained within bytecode compilation), you're done: everything
works exactly as for regular functions. The VM does not have to know anything about
generators at all. The disadvantage is that I found the transformation is be not quite so
simple; for example, local variables have to be turned into properties of a dedicated
object (or their state won't be persisted across a yield).

I might be wrong, but I'm very satisfied with having followed approach #2.  Stack frames
have a very simple representation at runtime, and inventing an alternative form from which
it can be restored wasn't hard.

(Although I suspect I'm going to have to eat my hat once I start implementing a GC
seriously. You'll get news from me, then.)

<!-- TODO Spend a couple words on the Iterator protocol -->
<!-- TODO Example? Does it make sense? -->

# Debugger

The debugger has seen *significant* additions. See for yourself:

![Screenshot of the mcjs debugger, mcjs_tools. Shows some instructions on the left, and
some source code on the right.](../20240505--debugger.png)

Some notable features:

- When you hover a value (instruction argument), the related values are also
  highlighted. This makes it easy to understand which instruction last assigned the value
  and/or where the same register or object is going to be accessed.

- The UI layout is fully customizable, Visual Studio-style. Quite easy to implement, all
  thanks to the fantastic [egui_tiles](https://docs.rs/egui_tiles/latest/egui_tiles/)! It
  can also be saved/restored in case a particular debugging "move" benefits from a
  specific combination of views.
  
- Thanks to the changes to the interpreter's API that came along with the switch to the
  stackless design, there is no longer any need for `Pin` or unsafe code, which really
  makes me feel much more courageous when I have to make changes.

- When stepping through instructions, you can decide to follow calls ("Into") or not
  ("Next").
  
- The interpreter can break on ony throw instruction, and/or only on those that result in
  an unhandled exception.

- Unhandled exceptions are easier to work with:
  - the full stack trace is printed, with filenames and line numbers;
  - a dedicated button allows setting a breakpoint at the throwing instruction.
  
# Future

<!--

TODO:

    - generators, iteration
        - for (... of ...)

    - roadmap
        - just how much of test262 do you want to implement before you switch to the JIT, goddamit?!

-->

The level of test262 coverage seems to be alright. I'll take a look at the currently
failing tests for features that are worth it to work on, but I don't expect to see any big
one.

That means it's time to put on the big boy pants and resume work on the JIT, the one
_actual_ compiler that mcjs is going to have.  I can't wait!

Turns out, I *can't* keep a regular pace in writing these posts, so no more promises
now. You two are getting this post whenever I feel like it. See you, space cowboy!

