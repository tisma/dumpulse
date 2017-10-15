Dumpulse: a dumb heartbeat daemon in 256 bytes of RAM and ≈245 bytes of code
============================================================================

The problem this solves is the following: you have a distributed
system of microcontrollers that talk to each other over a
network — for example, the CAN network of an automobile — and you
want to monitor the health of all of its subsystems over a
much-lower-bandwidth link, such as a nominally 8kbps Mobitex link.

It is desirable in such a situation to have some kind of
heartbeat-aggregation agent instead of communicating individually
with each subsystem, for the following reasons:

1. The heartbeat-aggregation agent can send a single packet reporting
   the health of the whole system, rather than one packet per
   microcontroller.  If there are, say, 64 microcontrollers, and the
   per-packet overhead is 64 bytes’ worth (roughly correct for Mobitex
   MPAK and link-layer overhead), the entire health report will
   require at least 4096 bytes, which is probably about 8 seconds of
   continuous Mobitex transmission.  By contrast, a single packet
   containing, say, 4 bytes of status data per service will be only
   320 bytes including the overhead, which works out to 640
   milliseconds of airtime, more than an order of magnitude
   improvement.

2. A particular microcontroller may be failed at the time that the
   remote monitoring system attempts to conduct the health check.
   But, to diagnose the problem, it may be desirable to determine with
   more precision when the failure happened.  A heartbeat-aggregation
   agent can store the last-known-good heartbeat information from the
   failed component and report it on request.

3. Any network can lose or corrupt data being transmitted, but
   high-bandwidth, low-latency controller-area networks are many
   orders of magnitude more reliable than low-bandwidth, high-latency
   radio networks, and the higher rate of retransmissions that is
   feasible over the faster networks works to reduce the impact of
   isolated packet loss or corruption.  A remote monitoring system
   that successfully gets a response from a heartbeat-aggregation
   agent knows that any missing data is missing because of a fault
   local to the controller-area network, not due to a radio-network
   failure.  By contrast, if the remote monitoring system expects to
   receive a status report from each controller over the radio
   network, a missing status report may be due to a subsystem failure
   or merely a lost over-the-air packet.

However, the heartbeat-aggregation agent is itself a potential single
point of failure in the monitoring system, so it’s desirable to keep
it as simple and robust as possible and to run it on a reliable
platform.  For example, if you run it from Flash ROM on a bare
microcontroller, only hardware failures or failures in the agent
software itself can cause it to fail; if, instead, you run it on
Linux, then kernel memory corruption, system overload, filesystem
corruption, bootloader failure, and the like all come into play.

It’s also important for the agent to have little or no configuration
of its own, because any configuration data is a potential point of
failure, and if it’s baked into the firmware for a processor that also
does other things, updating the configuration data poses the risk of
breaking whatever else you’re running on that processor.

So this program, Dumpulse, implements a simple protocol suitable for
aggregating heartbeat data sent over a physically secured network like
I²C or CAN, running on bare metal or under a minimal real-time
operating system like eCos, FreeRTOS, or RTEMS, although it also works
on Linux.  It provides no authentication and only minimal data
integrity checking.  All the packets except the 256-byte health report
are 8 bytes or less, so they can be sent as single CAN bus frames,
although the example Linux implementation uses UDP.  It takes no
configuration data.

Calling interface
-----------------

This module exposes one data type, `dumpulse`, which can be allocated
where you like and needs no finalization but should be initialized to
all zero bytes, and one function:

    uint8_t dumpulse_process_packet(dumpulse *p, char *data, void *context);

It returns nonzero if it successfully processed the packet and zero if
it did not, but generally speaking you don’t need to care.

It requires you to provide two functions:

    uint16_t dumpulse_get_timestamp(void);
    void dumpulse_send_packet(void *context, char *data, size_t len);

If it is necessary to send a reply packet, the `context` provided to
`dumpulse_process_packet` will be provided to your
`dumpulse_send_packet` function before `dumpulse_process_packet`
returns.  You can use it, for example, to store the address to which
to send the reply packet.  Because it’s only used before
`dumpulse_process_packet` returns, it is okay for it to be a
stack-allocated data structure.  `dumpulse_send_packet` should make a
copy of the data it is provided, or transmit it over the network,
before returning; just saving a pointer to it is not okay, because
that data will be mutated when the next heartbeat packet arrives.

Because `dumpulse_send_packet` has no return value, Dumpulse cannot
tell if the response packet was successfully sent or not, and so it
always considers a health report request message to have been
successfully handled.

Dumpulse does not allocate memory and can only fail in the sense of
not being able to process a packet because it is ill-formed.

Protocol
--------

The heartbeat message consists of 8 bytes: a four-byte little-endian
Adler-32 checksum of the other four bytes, a single byte with the
constant value 241, a single-byte variable ID identifying the variable
to update (from 0 to 63), a byte indicating the ostensible identity of
the packet sender, and a final byte containing the value of the
variable, in that order with no padding.  That is, with four bytes per
row:


    +----+----+----+----+
    |     Adler-32      |
    +----+----+----+----+
    |0xf1| var|from| val|
    +----+----+----+----+

The health report request message is the fixed 8 ASCII bytes
“AreyouOK”, and will be responded to with a 260-byte response
consisting of a four-byte little-endian Adler-32 of the rest of the
message, followed by 64 four-byte triples (16-bit timestamp, sender
ID, value).  That is, again with four bytes per row:

    +----+----+----+----+
    |     Adler-32      |
    +----+----+----+----+
    |timestamp|from| val| heartbeat variable 0
    +----+----+----+----+
    |timestamp|from| val| heartbeat variable 1
    .    .    .    .    .
    :    :    :    :    :
    |timestamp|from| val| heartbeat variable 63
    +----+----+----+----+

The 16-bit timestamp has a resolution determined by the application;
there’s a tradeoff between wraparound time and precision.  Taking the
low 16 bits of a `time_t` is adequate for many applications, giving a
wraparound time of 18.2 hours and a precision of 1 second.

Performance and code weight
---------------------------

Compiled for Linux with amd64 with GCC 5.4.0 with `-Os`, dumpulse is
245 bytes of amd64 code and 55 instructions.  It should be similar for
most other processors; for example, compiled for the 386 with `-m32`,
it’s 244 bytes and 61 instructions; compiled for the AVR ATTiny88 with
GCC 4.9.2 with `-Os -mmcu=attiny88`, it’s 201 bytes and 96
instructions; compiled for the ATMega328, it’s 205 bytes and 96
instructions.  However, all of these versions will pull in `memcmp`
from libc, and the AVR versions also `__do_copy_data`, if you aren’t
using them already in your program.

XXX why is there no undefined reference to `dumpulse_get_timestamp`?

These numbers of course do not count the size of the
`dumpulse_get_timestamp()` and `dumpulse_send_packet()` functions you
must provide.

Reliability and security
------------------------

This protocol is not designed to be operated in an environment, such
as a LAN or the public internet, where malicious actors can inject
packets.  Anyone who can send packets to Dumpulse can overwrite the
whole heartbeat table with arbitrary data, and they can cause Dumpulse
to send many times more data than they are themselves sending, which
could, for instance, provide them with a lever for a network
denial-of-service attack.  Furthermore, they may be able to cause it
to send almost arbitrary data in the health report, limited only by
timestamp control and the minor restrictions of Adler-32, which could
be used to bounce data to nodes they cannot send messages to but can
forge messages from, if any such exist.  However, they cannot crash
Dumpulse or cause it to use a large amount of CPU time.

However, it should be sufficiently tolerant of random or corrupted
data that you can run it over a noisy broadcast network without a
separate addressing layer underneath (something such as UDP which
ensures it only receives data intended for it).

Naïvely calculating, the Adler-32 and magic number in the heartbeat
message provide a probability of a random or corrupted 8-byte packet
being incorrectly accepted of 2⁻³², since for either of the two 32-bit
halves of the message you can calculate an other half (I think unique)
that would make it a valid heartbeat message.  In practice the value
may be somewhat higher or lower; the Adler-32 output is biased toward
low-valued bytes such as 1, 2, 3, 4, 5, 6, 7, and 8, though the bias
toward 0 bytes is very slight, and low-valued bytes are also likely to
occur disproportionately in random packets found in the wild.  In
particular, the byte 2 (^B) occurs about ⅔ of the time, and the byte 6
(^F) occurs about ⅓ of the time.

This suggests that if we are running our heartbeat data raw over a CAN
bus carrying constant traffic of 4000 frames per second, we should
expect to see a non-heartbeat frame randomly misrecognized as a
heartbeat frame about once every million seconds, about once every 12
days.  However, in ¾ of those cases, the variable won’t be in the
valid range of 0–63; in the remaining cases, if it’s being written to
a variable that’s being used by an active heartbeat, it will almost
certainly be overwritten by a valid heartbeat before the remote
monitoring proces requests a health report.

While the health report request message could occur randomly, the
consequence of sending an unnecessary health report is very mild.
Because its fifth byte is “o” and not 0xf1, it will never be confused
with a heartbeat message.