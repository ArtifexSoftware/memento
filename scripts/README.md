# Scripts for use with Memento

## vdb.pl

A script for launching an executable under valgrind and gdb
in a convenient form.

### Usage:

`vdb.pl <program and args>`

Once launched, use standard gdb instructions to drive it.

For example:

 * `break Memento_breakpoint`
 * `run`
 * `call Memento_breakAt(1024)`
 * `call Memento_setParanoia(1)`
 * `c`

In addition, a couple of new commands are implemented:

 * `xb expression`
   or
   `xb expression addr expression length`

   This displays details for a block of memory, both the bytes themselves, and the 'validity' bits.

   This does what you'd hope:

   `mon xb <expression_addr> <expression_length>`

   but it copes with expressions for the address and length
   rather than insisting on raw numbers.

 * `xs <expression>`

   This displays details for a zero terminated string, including validity bits.

   This does what you'd hope:

      mon xb <expression> <strlen(expression)>

   would do.

 * xv: `xv expression`

   This displays details for a value, including validity bits.

   This does what you'd hope:

    `mon xb &<expression> sizeof(<expression>)`

   would do.

### Requirements:

 * Valgrind must be installed, with the vdb executable on the PATH.
 * gdb must be installed and on the PATH.
 * [Optional, but highly recommended] rlwrap (for command line history and editing).

## squeeze2html.pl

A script for converting the output from squeeze runs into readable
HTML.

Syntax:

`squeeze2html.pl < in > out.html`

or (more usefully):

`squeeze2html.pl -q < in > out.html`

(to hide the ones that passed)

This can usefully be used as part of a filter with a running Memento process:

`MEMENTO_SQUEEZEAT=1 membin/gs -sDEVICE=png16m -o /dev/null 2>&1 | squeeze2html.pl -q > squeezelog.html`

## squeeze2text.py

A script for converting the output from squeeze runs into more readable
text.

This script throws away lots of information from the
output log to just keep the headlines (which runs leak,
which runs crash etc). This can save an awful lot of
discspace!

The failing iterations can then be rerun later to be examined.

For example:

`MEMENTO_SQUEEZEAT=1 ./membin/gpdl -sDEVICE=bit -o /dev/null examples/tiger.eps 2>&1 | toolbin/squeeze2text.py -o squeeze.txt`

