# Memento

A memory debugging library for C (or C++) programs.

## Features

Memento builds into your C (or C++) project. In both release
and debug builds it builds away to nothing. In a build with
a magic predefine (imaginatively enough "MEMENTO"), it
intercepts malloc/free/realloc/etc and handles them itself.

It watches for common errors, such as leaks, overruns, underruns,
write after free, and double frees, without needing any special
facilities beyond that provided by the standard C runtime.

Allocations are tracked by number, so that deterministic programs
can be repeatedly rerun multiple times to meaningfully compare
results.

When run under a debugger, Memento can be driven interactively
so as to stop on particular allocation numbers, to examine
pointers, and to watch for allocation events on particular
addresses. Repeated runs on deterministic input therefore
effectively allow 'rewinding' to before problems occur, and
stepping forwards.

On specific systems (Windows, or Linux with backtrace support)
full callstacks can be gathered for each allocation event,
allowing the full history of a block to be watched.

Source level annotations can be added to the application linked
with Memento to label blocks to simplify debugging. Calls can be
made to check pointers and ints heuristically for validity.

With appropriate source level annotations Memento can track
reference counting for blocks, and will include this data in the block
history.

Memento can simulate a range of different memory exhaustion
conditions; either by total amount used, or by total number
of allocations. In addition it can perform automated 'squeezing'
runs where an executable's response to memory exhaustion can
be tested at every possible point.

Memento can be used with Valgrind's Memcheck tool (henceforth just
Valgrind), to further enhance its abilities.

## Contents of the distribution.

 * [memento.h](memento.h)

   The memento header file. #include "memento.h" in all your source
   files, ideally as the first thing you do - certainly before anything
   that makes reference to an allocation function.

   Most significant projects already have a top-level include file, so
   this can be inserted there.

   This header file will compile away to nothing unless MEMENTO is defined.

 * [memento.c](memento.c)

   The memento source file. Compile this with your other C files, using
   the same flags.

   This will compile away to nothing unless MEMENTO is defined.

 * README.md

   This readme.

 * [COPYING.txt](COPYING.txt)

   The ISC license under which this software is released.

 * [scripts](scripts/README.md)

   Various helpful scripts.

 * [docs](docs/README.md)

   Documentation.
