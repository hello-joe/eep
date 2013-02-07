    Author: Björn-Egil Dahlberg
    Status: Draft
    Type: Standards Track
    Erlang-Version: R15B
    Created: 30-Jan-2012
    Post-History:
****
EEP xx: System limits for Erlang
----


Abstract
========

A description of setting limits on processes for resource managing.


Rationale
=========

When deploying and running an application a scenario where aggressive memory
allocations may happen, unforeseen at development time. Improbable user input
for a process may cause it to allocate too much resources starving other
healthy processes. The most destructive case is when memory allocation
for a single process runs so far that a new heap cannot be allocated, in
turn, causing the virtual machine to halt even though the system is
healthy as a whole.

To remedy this we propose limits on processes, or as a system wide setting,
on first and foremost total heap block sizes. Total heap block sizes includes
stack and heap sizes for the primary heap block and the heap size for the
generational heap. The process message queue will implicitly be included if
measured after a collection since those messages will now have been copied
into the heap block.

If a process reaches the heap size limit it will terminate by receiving an
exit signal.

The heap size limit can be set by three different methods.

  1) At creation by `erlang:spawn_opt/2`
  2) During run time by `erlang:process_flag/2`
  3) By `erlang:system_flag/2`

The limits can also be inspected using the normal functions for these types
of settings.

  1) By `erlang:process_info/0,1`
  2) By `erlang:system_info/1`

Furthermore, the system flag setting can be set at start via command line
using an option similar to minimum heap size.

  1) erl +hls <Heap Limit Size>


Specification
=============

Limits are by default turned off. An individual process heap size limit can
either be introduced at spawn time or by the process itself with
`erlang:process_flag/2`.

Perhaps the most useful scenario is setting limits at spawn, for instance for
certain types of worker processes.

By letting the limit value option be specified by a list we make sure that
this option can be extended in future.

    spawn_opt(fun() -> start() end, [{limits, [{heap_size, N}]}])

Here the spawned process will be limited to have N words of total heap size.
If the process reaches this value it will be exited and any other process
linked to this process will also exit. Exit mimics the current behavior of
processes. In fact, it is the same behavior. Conceptually the process sends
an exit signal to itself when a limit is reached.

A running process can set it own limit by using `erlang:process_flag/2`. Below
is an example of setting a new limit to Size.

    %% currently returns OldSize only
    {heap_size, OldSize} = erlang:process_flag(limits, {heap_size, Size})

or in the future by setting multiple limits at once. In that case we propose
returning the multiple values.

    [{heap_size, OldSize},{some_limit, OldLimit}] = erlang:process_flag(limits, [
							{heap_size, Size},
							{some_limit, Limit}])

It should also be possible to set this flag on other processes than your own
by using `erlang:process_flag/3`. The syntax and return values will be the same
as above with the exception of a process identifier as an extra argument.
Currently only the flag `save_calls` is allowed to be passed to other
processes.

The third case to consider is the system wide setting via `erlang:system_flag/2`.

    %% currently returns OldSize only
    {heap_size, OldSize} = erlang:system_flag(limits, {heap_size, Size})

Any process spawned after this flag is set will have a limit Size on its
total heap sizes. If during this time a spawn option also is given via
`spawn_opt/2` the limit passed via spawn will supersede the system setting.
This behavior is similar to the one of minimum heap size which behaves in
the same manner.

A limit is turned of by setting its value to zero. Zero is the default value.

    OldSize = erlang:process_flag(limits, {heap_size, 0}) 

The system flag can also be set at start via a command line option in the
same manner as other system flags.

    erl +hls 2000

In the example above the `heap_size` limit is set to 2000 words on any process
spawned unless specified to have another limit via spawn\_opt/2 or later
changed via `process_flag/2,3`.


Heap size limits will be checked directly after a garbage collect. At this
point garbage will be excluded and any message previously in heap fragments
will be included while measuring against the heap limit. This will ensure
that we only measure against useful data with the rationale that garbage is
not useful and messages are.
 

Reference Implementation
========================

A reference implementation can be found at github.

    git fetch git://github.com/psyeugenic/otp.git egil/limits-system-gc/OTP-9856

Copyright
=========

This document has been placed in the public domain.

%% vim: syntax=markdown textwidth=77
