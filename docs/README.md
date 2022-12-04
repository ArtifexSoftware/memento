# Memento Documentation

## Installation

There is almost no installation required for Memento.
Simply copy memento.h and memento.c into the source
code for your project.

If you are going to distribute your code, remember to
include the required attribution as per the license.

## Building Memento into a project

The best way to include Memento into a project is to
create a new configuration for it.

In the same way that you probably already have `release`
and `debug` builds for your project, I'd recommend
creating a `memento` build.

For IDEs such as Visual Studio, this can be as simple as
copying the existing 'Debug' configuration to a new
'Memento' one.

For Makefile based builds, this will normally involve
creating a new target or targets. For instance, in
a project foo that had targets for `foo` and `debugfoo`,
you'd probably want to create a `memfoo` target.

Next, ensure that memento.c is added to the list of
files to be compiled for your project (including in
release and debug builds!) - it will compile away to
nothing in such builds, so there is no 'cost' in this.

Next, ensure that every C file (or at least every C file
that calls allocation related functions) includes
'memento.h' as early as possible.

For small projects this can be as simple as manually
including a `#include "memento.h"` at the top of
each C file. For larger projects, there is frequently
a project-wide header file already, and just adding
`#include "memento.h"` to this is enough.

Finally, edit the configurations to predefine the
'MEMENTO' symbol for the memento build targets.
For IDEs such as Visual Studio, this is a simple
matter of adding MEMENTO into the build properties
for that configuration.

Now, when you build that configuration of your program
you should have Memento built in.

## Running a memento build

Run your program from the command line exactly as before.
Other than being a bit slower, no differences should be
visible (unless problems are found!) until the program
exits, whereupon details will be displayed of the 
memory usage.

If memory corruptions are found, then details of these
will be displayed at the time at which they are found.

If memory leaks are found, then a full list of such
blocks, plus (depending on the platform in use) the 
allocation history for those blocks will also be
displayed.

This information can be large, so often it can be a good
idea to run with stderr redirected to file.

The behaviour of Memento can be adjusted by various
environment variables. These are discussed below.

## Running a memento build under the debugger

While Memento can be useful by itself, it is doubly
useful when used in conjunction with a debugger.

While the exact mechanics of the operation of different
debuggers varies, the central concepts remain the same.

Firstly, whenever Memento encounters a problem (or,
more generally, when it thinks it might be a good time
to communicate to the user) it will call a function
`Memento_breakpoint()`. By default, this function does
nothing at all; it exists so that a debugger can put
a breakpoint on it, and hence control can return to the
user.

Accordingly, in an IDE based debugger such as Visual
Studio, you'd add a breakpoint for `Memento_breakpoint`,
and it would be remembered on every run of the program.

In debuggers such as gdb you will most probably need to
add a breakpoint manually. For example: `break Memento_breakpoint`

Start your program as normal under the debugger, with
all your usual arguments. Debuggers such as Visual Studio
will typically start execution immediately, whereas
debuggers like gdb will load the application and stand
ready to start. In the case of gdb this is a good time
to `break Memento_breakpoint` and then `run`
to actually start execution.

As soon as the program makes its first allocation
Memento will be called, and it will call
`Memento_breakpoint`. This will cause the debugger
to stop.

At this point, we can issue various commands to
control the behaviour of Memento. There are a range
of possible commands all of which are accessesd by
getting the debugger to call some simple C functions.

Again, exactly how this done depends on the debugger
in use.

Within Visual Studio, this is achieved by bringing
up the 'Quick Watch' window (typically reached by
Alt-Ctrl-Q). Type the required command into the box
and hit return (e.g. `Memento_setParanoia(1)`).

With gdb, this is achieved by using the `call` command.
For instance `call Memento_setParanoia(1)`.

Any output from the command should be seen on stderr.

The possible commands are detailed below.

## What Memento does (and does not do) as it runs

Memento hooks (via `#define`s) allocation functions
such as `malloc`, `free`, `realloc`, `calloc`,
`strdup` etc.

Whenever a block is allocated, Memento 'overallocates'
the block, requesting more bytes that were asked for.

It uses some of these extra bytes at the start to store
a header with some information in, and some guard
(sentinel) bytes. Some more guard bytes are used at
the end.

The central portion of the allocated block (between the
two guard sections) is then returned to the caller
(your application).

Other functions such as `free` or `realloc` know about
these padding bytes and allow for them.

So, your C program calls `malloc`, `free`, `realloc`
etc as normal and (all being well) should never know
that Memento is getting involved.

All of Memento's magic happens during such interceptions.
Memento only ever gets called during a `malloc`, `free`
etc. It doesn't rely on any compiler (or operating
system) specific trickery.

This is both a strength, and a weakness.

It's a strength because this means Memento can run on
any platform with a conformant C runtime - no toolchain
support issues, or complex ports to do.

It's a weakness, because it means that Memento can't
spot problems as they happen, but only after they've
happened. Fortunately, this turns out not to be such
a limitation after all, as we shall see.

## Allocation events and block numbering

Every time Memento is called for a memory operation,
it increments a counter, known as the 'sequence' number.

Whenever a block is allocated, the current sequence
number is stored in it. On platforms where backtraces
are supported (such as Windows and Linux with libbacktrace)
the callstack is stored in the history for the block
too.

Whenever a block is realloced or freed, the current
sequence number is stored in the history for that block,
with (again, where supported) the backtrace for the
block.

By keeping such data, when a block is leaked, we can
say things like "The block allocated on event 10 leaks".

If your application is deterministic, then we can run
it several times on the same input, and the same blocks
should leak, at the same points each time.

## The block lists

Memento keeps a record of all the blocks that are
currently allocated. When blocks are freed they move
from the 'allocated' list to a 'free' list.

Once the total size of the blocks in the free list
reaches a certain limit, they get freed back to the
system.

This means that blocks hang around in the system for a
while after they would usually have died.

## Magic numbers

Memento fills blocks with varoius magic numbers.

The guard bytes before the returned block are filled
with 0xA6. The guard bytes after a returned block are
filled with 0xA7. Blocks themselves are filled with 0xA8
when initially malloced, and filled with 0xA9 once freed.

These exact numbers are largely unimportant, but serve
to help Memento spot overwrites/underwrites and writes
after the block has been freed.

Memento periodically checks the pre/post guard sections
of all the blocks it knows about, and the main body of
blocks that are on the free list to ensure they still
have the expected fill values.

Everytime it checks a block and finds it to be consistent
it will store the current sequence number in it (call
this *n*).

Whenever it finds an inconsistent block, it can then
inform the user that it became corrupted between event
*n* and the current sequence number.

## Controlling the frequency of Mementos consistency checks

In an ideal world, Memento would check all the blocks it
knows about on every single allocation event.
Unfortunately, in the real world, for all but the
simplest programs, this is far too slow.

Accordingly, the frequency with which it runs a check
is controlled by a setting, known as the 'paranoia' level.

With a paranoia level of 1, Memento runs a check on
every event. With a paranoia level of 2, on every second
event. etc.

Negative paranoia levels allow checks to back off with
checks occuring at longer and longer intervals.

There are 4 ways to control the paranoia level:

 1 The user can set the MEMENTO_PARANOIA environment
   variable on startup.

 2 The user can call `Memento_setParanoia(x)` from
   the debugger.

 3 The user can set MEMENTO_PARANOIDAT environment
   variable. Once this number of events have passed
   the paranoia level is set to 1.

 4 The user can call `Memento_paranoidAt(n)` from
   the debugger.

## Triggering a consistency check

The user can trigger a consistency check at any point
by calling the `Memento_checkAllMemory()` function.

This can be done either from within the debugger, or
by embedding such calls in the code and recompiling.

## Finding a block number

Sometimes you will have a pointer, and will wish to
know which block this pointer is in. This can be
achieved by calling `Memento_find(ptr)`.

## "Rewinding"

Suppose you have run Memento, and been told that, say,
block *x* has become corrupted, it was last
checked to be consistent at event *y*, and
it's now event *z*.

A typical approach to solving that might be to rerun
the program from scratch under the debugger, remembering
to set a breakpoint on `Memento_breakpoint`.

When execution stops there initially, call
`Memento_breakAt(x)` and continue.

Execution will then continue, and stop in the middle
of the block's allocation. The environment and backtrace
can be examined, and control can be stepped onwards
to investigate the circumstances of the allocation.

When the user is satisfied, he can then call
`Memento_paranoidAt(y)` and continue the execution
again.

This time the code will run through to event *y*
and begin to check consistency at every event after that.

Eventually, assuming the program/input is deterministic,
the error should occur again. This time telling us
that block *x* has been corrupted, between event *w*
and *w+1*.

Now we can rerun the code, this time calling
`Memento_breakAt(w)`.

The code will stop at on allocation event *w*.

At this point, the heap should be intact - we can
vertify this by calling `Memento_checkAllMemory()`.

We can step through further and further in the debugger
and hopefully catch the problem as it occurs.

## Which event am I on?

Calling `Memento_sequence()` will return the current
sequence number. This can be useful when stepping in
a debugger to know where you are within execution.

Alternatively, it can be compiled into code so that
debugging code can be triggered during a particular
range of events.

## Increasing the number of events

Sometimes the amount of work done between allocation
events can be larger than we'd like. To narrow down
these gaps, we can insert calls to `Memento_tick()` in
our code and recompile.

Each call to `Memento_tick()` effectively inserts a
new 'no-op' allocation event.

Accordingly, the numbering of allocation events within
a run will change, but you may now have an event that
you can restart at closer to your failure point.

## Labelling blocks

After using Memento for a while, you may come to decide
that it'd be nice if blocks were labelled by type in the
Memento debugging.

For this purpose, we have the `Memento_label()` call.

Simply wrapping an allocation with Memento_label allows
a static string to be associated with the block. For
example:

`foo_t *foo = malloc(sizeof(foo_t));`

might become:

`foo_t *foo = Memento_label(malloc(sizeof(foo_t)), "foo");`

A sprinkling of these throughout your code can drastically
simplify the job of reading Memento leak reports.

## Simulating memory exhaustion

One of the most common causes of problems in C is what
happens when memory runs out. This is traditionally
hard to simulate. Memento offers a couple of ways to
approach this issue.

Firstly, we can place a hard limit on the amount of
memory that Memento will allow to be allocated.

If the user calls `Memento_setMax(bytes)` (or
alternatively sets the MEMENTO_MAXMEMORY environment
variable) then Memento will fail any allocation that
would take the total memory usage above this amount.

Alternatively, we can arrange for Memento to fail all
allocations after a certain number.

This is achieved by calling `Memento_failAt(n)` (or
by setting the MEMENTO_FAILAT environment variable).

## Memory squeezing

Even with the above facilities, it can be tricky to be
sure that the error trapping is correct for every single
possible place where an allocation might fail.

Accordingly, one technique is to 'squeeze' an executable
by running it repeatedly. The first time, you fail every
allocation after the *n*th, allowing the program to exit
and checking that it copes gracefully. Then you run it
again, this time failing every allocation after the
*(n+1)*th.

By repeating this process, you can exhaustively check
all the possible error handling exit routes within your
code.

The big problem with this, of course, is that it will
take longer and longer to run each additional iteration.
If it takes *t* seconds to run the program through to
fail at allocation *n*, the next run (to fail at *n+1*)
will take longer than *t* seconds.

Memento offers a solution to this, whereby, on systems
that support `fork()` (i.e. Linux and similar),
we can be far more efficient.

By setting the MEMENTO_SQUEEZEAT environment variable
before running your program, it will run through to the
requested event, *n* say. At that point it will
`fork()`, producing 2 identical copies of the running
program - we'll call them parent and child.

The parent sits patiently and waits for the child to
complete.

The child fails the allocation, and runs to completion.

Any crashes or leaks are reported to the user.

Then the parent continues. It allows allocation *n* to
complete, and continues to allocation *n+1*. At this
point it calls `fork()` again, and the process repeats.

In this way, memory squeezing can proceed with each
iteration only having to test the closedown routines
rather than rerunning the bulk of the test each time.



