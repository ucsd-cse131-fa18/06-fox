# Fox

![A Fox Snake, Pantherophis gloydi](https://upload.wikimedia.org/wikipedia/commons/thumb/f/ff/Pantherophis_gloydi.jpg/440px-Pantherophis_gloydi.jpg)

The `fox` system is the `egg` extended with a *garbage collector*.
That is, you have a core language with functions, numbers, booleans,
and _tuples_, but now, you will extend the C **run-time** to reclaim
and recycle (heap) memory that is no longer being used.

## Download 

1. Use the _fork link_ from the class website to create your private clone of the starter code.

2. Do `git clone https://github.com/ucsd-cse131-fa18/06-fox-XXX` where `XXX` is your private repo.

3. Link your clone to the "upstream" to get any updates

```
$ make upstream
```

after this you can get "updates" (in case we modify the starter code), with 

```
$ make update 
```

4. Save (and submit) your work with: 

```
$ git commit -a -m MESSAGE
$ git push
```

## Language
=======

Fox has the _same_ language features and semantics as Egg-eater, except
that *you* will implement the automated memory management.

A heads up – there aren't a ton of lines of code needed to 
complete the assignment; I wrote around 350 lines in `gc.c`.  

## Runtime and Memory Model

Tuples expressions should evaluate their sub-expressions in order, and store
the resulting values on the heap.  The layout for a tuple on the heap is:

```
  (4 bytes)   (4 bytes)  (4 bytes)   (4 bytes)
-------------------------------------------------------------------
| # elements | GC-Word  | element_0 | element_1 | ... | element_n |
-------------------------------------------------------------------
```

That is,

* One word stores the _count_ of the number of elements in the tuple,
* One word is the **GC-Word** that will store meta-data needed for collection,
* The subsequent words are used to store the values themselves.

A _tuple value_ is stored in variables and registers as the address
of the first word in the tuple's memory, but with an additional `1`
added to the value to act as a tag.

For example, if the start address of the above memory
were `0x0adadad0`, the tuple value would be `0x0adadad1`.  
With this change, we extend the set of tag bits to the
following:

- Numbers: `0` in least significant bit
- Booleans: `111` in least three significant bits
- Tuples: `001` in least three significant bits

Visualized differently, the value layout is:

```
0xWWWWWWW[www0] - Number
0xFFFFFFF[1111] - True
0x7FFFFFF[1111] - False
0xWWWWWWW[w001] - Tuple
```

### Initializing and Accessing tuple Contents

This is the same as in `egg-eater`, except, you should:

* Initialize the GC word with `0x0`
* Recall that the `i-th` element is stored at an offset of `i+2` from the tuple base pointer.


### Checking for Memory Usage, and the GC Interface

Before allocating a closure or a pair, `Fox`
*checks* that enough space is available on the heap.  
The instructions for this are implemented in
`tupleReserve` in `Compile.hs`.  

If there is not enough room, the generated code
calls the `try_gc` function in `main.c` with the
information needed to start automatically
reclaiming memory.

When the program detects that there isn't enough
memory for the value it's trying to create, it:

1. Calls `try_gc` with several values:

    - The current top of the stack (i.e. current `esp`)
    - The current base pointer  (i.e. current `ebp`)
    - The amount of memory it's trying to allocate
    - The current value of `ESI`

   These correspond to the arguments of `try_gc`.

2. Then expects that `try_gc` either:
   - Makes enough space for the value (via the GC algorithm below),
     and returns a new address to use for `ESI`.
   - Terminates the program in an error if enough space cannot be
     made for the value.

There are a few other pieces of information that the
algorithm needs, which the runtime and `main.c`
collaborate on setting up.

To run the mark/compact algorithm, we require:

  - `HEAP`, a global variable stores the heap's starting
    location upon startup.

  - `HEAP_END` and `HEAP_SIZE`, two global variables that
    respectively store the heap's ending location and size.
    The generated instructions rely on `HEAP_END` to check
    for begin out of space.

  - Information about the shape of the stack: This is
    described below – we know a lot given our choice of
    stack layout with `EBP` and return pointers.

  - `STACK_BOTTOM` stores the *beginning of the stack* and is set by the
    the instructions in the prelude of `our_code_starts_here`,
    using the initial value of `EBP`.  This a useful value, because when
    traversing the stack we will consider the values between base pointers.

  - `ESP` which will hold the value of the *end of the stack*, at the point
    when garbage collection is triggered.

All of this has been set up for you -- in the `prelude`
described in `Asm.hs`, but you do need to understand it.  
So study `try_gc`, the new variables in `main.c`, and
the code in `Compile.hs` that relates to allocating and
storing values (especially the `tupleReserve` function
and the instructions it generates).

## Managing the Stack

As discussed below, your `gc` code will first traverse the heap
starting with *roots* corresponding to "live" local variables on
the stack. Thus, your generated assembly must ensure that at all
points the stack slots either refer to live tuples **or, if unused,
are reset to 0**. We do this in two places:

* The code in `funEntry` that handles the adjusting of `EBP` and `ESP`
  and the creation of a new stack frame inside the callee, now, in addition
  to shifting `ESP` by `4*n` also "clears" the `n` slots via the instructions:

```haskell
-- instructions for setting up stack for `n` local vars
funEntry :: Int -> [Instruction]
funEntry n = [ IPush (Reg EBP)                       -- save caller's ebp
             , IMov  (Reg EBP) (Reg ESP)             -- set callee's ebp
             , ISub  (Reg ESP) (Const (4 * n))       -- allocate n local-vars
             ]
          ++ [ clearStackVar i | i <- [1..n] ]       -- zero out stack-vars

clearStackVar :: Int -> Instruction
clearStackVar i = IMov (Sized DWordPtr (stackVar i)) (Const 0)
```

* Your code for compiling `Let` expressions should follow the above
  idea (e.g. use `clearStackVar`) to "clear" the slot for the binder
  `x` *after* executing the instructions for `e2`.

```haskell
-- "clear" the stack position for 'x' after executing these instructions for e2
compileEnv env (Let x e1 e2 _)   = error "TBD:compileEnv:Let"
```


## Managing Memory

The bulk of your work in this assignment is in writing `C` code
to implement a mark-compact algorithm that reclaims unused memory
by rearranging the contents of the heap.

### Mark/Compact

The algorithm works in *four* phases:

1. **Mark** – Starting from all the references on the stack, all of the
   reachable data on the heap is _marked_ as live.  Marking is done by
   setting the *least-significant* bit of the GC word to 1.

2. **Forward** – For each live value on the heap, a new address is
   calculated and stored.  These addresses are calculated to compact
  the data into the *front of the heap* with no gaps.  

3. **Redirect** - The forwarded addresses, which are stored in the
   remainder of the GC word, are then used to update all the *internal*
   values on the stack and the heap to update them to point to the new,
   *forwarded* locations.  Note that this step does not yet
   move any data, just set up forwarding pointers.

4. **Compact** – Each live value on the heap is copied to its
   forwarding location, and has its GC word zeroed out for future
   garbage collections.

The end result is a heap that stores:

* only the data reachable from the heap,
* in as little space as possible (given our heap layout).  

Allocation can proceed from the end of the compacted space
by resetting `ESI` to the final address.

We discuss the three phases in more detail next.

#### Mark

In the first phase, we take the initial heap and stack,
and set all the GC words of live data to 1.  
The live data is all the data reachable from references
on the stack, excluding return pointers and base pointers
(which don't represent data on the heap).  

We can do this by looping over the words on the stack,
and doing a depth-first traversal of the heap from any
reference values (pairs or closures) that we find.

The `stack_top` and `first_frame` arguments to `mark`
point to the top of the stack, which contains a previous
base pointer value, followed by a return pointer.

If `f` had local variables, then they would cause
`stack_top` to point higher. `stack_bottom` points
at the highest (in terms of number) word on the stack.  
Thus, we want to loop from `stack_top` to `stack_bottom`,
traversing the heap from each reference we find.  

We also want to *skip* the words corresponding to
`first_frame` and the word after it, and each pair
of base pointer and return pointer from there down
(if there are multiple function calls active).

Along the way, we also keep track of `max_address`,
the highest start address of a live value, to use
later.

#### Forward

To set up the forwarding of values, we traverse
the heap starting from the beginning (`heap_start`).  
We keep track of two pointers, one to the next space
to use for the eventual location of compacted data,
and one to the currently-inspected value.

For each value, we check if it is live:

* If it is live:
  - Set its forwarding address to the current compacted data pointer,
  - Increase the compacted pointer by the size of the value.  

* If it is not live:
  - Simply continue onto the next value

Use the tuple-elements metadata to compute the size
of each block to determine which address to inspect
next.  

The traversal stops when we reach the `max_address`
computed during the *mark* phase (so we don't
accidentally treat the undefined data in those
spaces as real data).

#### Redirect

Then we traverse all of the stack and heap values
*again* to update any *internal* pointers to use the
new addresses.

#### Compact

Finally, we traverse the heap, starting from the
beginning, and copy the values into their
forwarding positions.  

Since all the internal pointers and stack
pointers have been updated already, once
the values are copied, the heap becomes
consistent again.  

We track the last compacted address so
that we can return the first free address,
which will be returned and used as the new
address from which to start allocation.  

While copying the blocks, we also *zero out*
all of the GC words, so that the next time
we mark the heap we have a fresh start.

I also highly recommend that you walk the
rest of the heap and set the words to some
special value, e.g. the value `0x0cab005e` – 
the "caboose" of the heap.  
This will make it much easier when debugging to
tell where the heap ends, and also stop a runaway
algorithm from interpreting leftover heap data as
live data accidentally.

#### Debugging

You will get SEG-faults.

We recommend using `lldb` or `gdb` to debug them.
For example even just this:

```
$ make tests/output/gc-0.run
$ lldb tests/output/gc-0.run
...

lldb> run
```

will show you the last point before the generated
code is about to SEG-FAULT.

Also, there are two helper functions
`print_stack` and `print_heap` defined
for you in `gc.c` that you can use to
print out snapshots to do old-school
printf-debugging.

The former prints the whole stack.
The latter takes an array and a number
of elements to print, and prints them
one per line like so:

```
  0/0x100df0: 0x5 (5)
  1/0x100df4: 0x0 (0)
  ...
  23/0x100e4c: 0x4 (4)
  24/0x100e50: 0xcab005e (212533342)
```

The first number is the 0-based index from
the start of the array, and the second is
the memory address.  After the colon is the
value, in hex form and in decimal form (in
parentheses).  This is a useful layout of
information to have at a glance for interpreting
the structure of the heap.

While automated testing and a debugger are
both invaluable, sometimes there's just no
substitute for pretty-printing the heap
after each phase in a tricky test.


## Recommended TODO List

1. Fill in the code for `Tuple`-creation and
   `GetItem` - access and printing by copying
   over the corresponding parts from `egg-eater`
   and adjusting them to make space for the GC word;

2. Update the case for `Let` in `compileEnv` to reset
   stack slots after they go out of "scope";

3. Make sure all your egg-eater tests pass!

4. Fill in the four functions in `gc.c`; feel free to use
   or disregard the helper code provided that provides
   C types for our `Value` and conversions between those
   and plain `int` (which is how everything is actually
   stored on the heap.)

5. Fill in the `print_tuple()` function in `types.c`. 

6. Test your code, and ensure that the `fox` tests pass!

A note on support code – a lot is provided, but you can
feel free to overwrite it with your own implementation,
if you prefer.


## Tests (5-for-5)

* You **must** add 5 new tests in `yourTests` in order
  to get **base credit**. Feel free to add more, we'll
  just take the first 5.

* The creators of the **5 hardest tests** get **5%**
  of total points as **extra credit**

* A test's "score" is the total number of compilers
  submitted by the entire class on which the test
  produces the **wrong** output.

* That is, the **harder** a test, the **higher** its
  score.

* To create a GC test, you need to specify, in addition
  to the program and its desired output, the HEAPSIZE
  in which that output can be produced; e.g. the following
  test in  `tests/fox.json` requires that `gc-copy-1`
  successfully executes in a heap of 14 bytes:

```json
, { "name"  : "gc-copy-1" ,  "code"   : "file" , "heap" : 14, "result" : { "value" : "(60001, 60002, 60003)"} }
```  

  You can craft interesting tests by shrinking the heap size
  parameter to find the smallest size at which a test should
  pass.

## Interactive REPL

```bash
$ stack ghci
```

and then you are inside `ghci` where you can use the function `run` to
**rapidly** test your code.

```
λ> :t run
run :: FilePath -> Program -> IO Result
```

For example to directly compile and run a string do:

```
λ> run "" (Code "1 + 2")
```

To run a program whose code is in a file `tests/input/file.fox` do:

```
λ> run "file" File
```

To run a program **in a heap of 30 bytes** (useful to test your GC...) do:

```
λ> runHeap 30 "file" File
```

To try a heap of size `n` use `n` instead of `30`.

We will test your GC by checking whether you can run certain programs
in a sufficiently small heap.

When you edit your code, instead of having to
rebuild, you can more rapidly type `:reload` in
`ghci` and then re-run the above commands.

## Valgrind

If you wish, use `valgrind` (installed in the lab)
to help debug your code, for example thus:

```
$ make tests/output/file.vresult
```



## Handing In

Please fill the following form to register your group members and your ieng account username.

https://goo.gl/forms/ETbo4ExwVRXhAbaW2

**IMPORTANT: You have to submit the assignments individually (i.e. both students in a group should submit using their own ieng account). Both members can turn in the same code.**

## Submission Instructions

First, make sure that your code works with the provided test cases by running
the command

``` sh
make test
```

When you're ready, run the following command to submit your homework:

```sh
make turnin
```

Before submitting your code, you have to fill
[this form](https://goo.gl/forms/3KBuBTilmvMKFD223)
to register your groups. **You need to fill it in for each assignment**.

We will use this form to match your  github account username to
your student id, so you **must** fill it even if you have worked
on the assignment individually.

To submit your code, just do:

```bash
$ make turnin 
```

This will simply do a `git commit` followed by a `git push` to send us your code.
**We will use the _most recent commit_ of your code (on `master` branch) as your submission.**

[See Piazza Question #84](https://piazza.com/class/jmkt5fpo5h4387?cid=84)
