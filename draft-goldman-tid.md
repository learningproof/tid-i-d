---
title: "The Time-Ordered ID and b32tid Binary Encoding"
abbrev: "TID-I-D"
docname: draft-goldman-tid-latest
date:
category: info
ipr: trust200902
# area: ART
wg: Multiformats (proposed)
submissiontype: IRTF
v: 3
keyword:
 - base-encoding
 - base32
 - timestamp
 - distributed systems
pi: [toc, tocindent, sort refs, symrefs, strict, compact, inline]
venue:
#  group: Multiformats (proposed)
  github: "learningproof/tid-i-d"
#  mail: multiformats@ietfa.amsl.com
author:
 - name: A. Goldman
   org: 3box Labs
   uri: 3boxlabs.com
   email: aarongoldman at gmail dot com
 - name: J. Caballero
   org: learningProof UG
   uri: learningproof.xyz
   email: bumblefudge at learningproof dot xyz
normative:
informative:
  RFC4648:
  RFC9562:
  RFC5102:
---

--- abstract

Developed for use in a high-bandwidth distributed social networking context, the
TID is essentially a highly-compact UUIDv6 variant that optimizes for a few
specific properties (most notably being sortable both bytewise and lexically)
and fits into the space-efficient `int64` type of languages that support it. One
way it achieves these properties is by using a bespoke base-32 encoding alphabet
rather than the similar base32hex encoding.

--- middle

# Introduction

The TID is essentially a UUIDv6 variant that optimizes for a few specific
properties:

1. sortable both bytewise and lexically when encoded with a base32 variant (also
specified in this document);
1. collision-resistant for up to 024 independent parallel timestamping services,
with this set of 1024 broken up in 3 contiguous namespace to support various
use-cases described below;
1. based on microseconds since unix epoch to simplify translation to other
timestamp formats;
1. fits in an `int64` for efficient storage, sorting and compute; and,
1. works well across the type systems or most major compiled languages in use
today for application-level development.

Many minor choices, such as the choice of code points in the alternate base32
encoding and the signed nature of the timestamp bytespace are primarily informed
by cross-language ergonomics.

## Base32tid encoding

The 32 code points chosen to encode from binary, in order, are:

~~~~ bash
value               1111111111222222222233
          01234567890123456789012345678901
encoding  234567abcdefghijklmnopqrstuvwxyz
~~~~

This differs from the canonical `base32` encoding defined in {{?RFC4648}} in two
ways: firstly, the letters are lowercase, and secondly, the letters and numbers
are inverted in the mapping sequence, such that bytewise ordering of binary TIDs
and lexical sorting of base-encoded TIDs will always achieve the same ordering.

~~~~ bash
value                1111111111222222222233
           01234567890123456789012345678901
base32tid  234567abcdefghijklmnopqrstuvwxyz
base32     ABCDEFGHIJKLMNOPQRSTUVWXYZ234567
~~~~

# Composing TIDs

The canonical form of a TID is a 64-byte string which takes this form:

~~~~ bash
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                         signed_microseconds_since_1970
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
                                                |   clock_id        |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~

## Timestamp Component

The form of timestamp used to generate a TID is the number of microseconds since
the Unix epoch (1970-01-01T00:00:00+00:00), i.e. with three more digits than an
{{?RFC5102}} `dateTimeMilliseconds`. In cases where a timestamping service may
be returning timestamps faster than 1000 times every millisecond, uniqueness
should be favored over microsecond accuracy; i.e., the ID generator should
return the current microsecond since epoch OR the last microsecond returned plus
1, whichever is greater.

The effective range of TIDs is limited by the compaction into `int64` form and
the 10 bytes used to encode the nodeId segment; for reasons that will be
explained below, it is further limited by one byte to avoid some translation
problems with the targeted encodings and type systems, leaving 53 bytes of
signed space for a subset of unix microsecond timestamps. Effectively, this
means that the range of microseconds before or after 1970, expressed as a signed
integer, is not (-2^63+1) to (2^63-1), but (-2^53+1) to (2^53-1). The following
tables shows the min, zero, and max values of the integer range of microseconds,
expressed in the 11-codepoint `base32tid` encoding. The additional 2 codepoints
for the nodeId segment, explained below, are omitted for clarity.

|tid          |microseconds     |valid?            |ISO timestamp      |
|---          |---              |---               |---                |
|s222-222-2222|-9007199254740991|yes (min value)   |1684-07-28T00:12:25|
|2222-222-2222|                0|yes               |1970-01-01T00:00:00|
|bzzz-zzz-zzzz| 9007199254740991|yes (max value)   |2255-06-05T23:47:34|
|zzzz-zzz-zzzz|18014398509481982|no (binary unsafe)|2540-11-07T23:35:09|

Note that half the possible range of values encodable in 11 codepoints are
considered invalid TIDs, as their binary form would not fit safely in an `int64`
bytestring. As the canonical form of TIDs is an `int64` bytestring, the invalid
half of the string-encodable range should not be mistaken for valid TIDs and
software handling these TID should validate strings accordingly.

## Node Identifier Component

The `node` identifier, by analogy to the equivalent element in an {{?RFC9562}}
UUIDv6, is a spatially-unique identifier, but occupying a much smaller space (10
bits, as opposed to UUIDv6's 48 bits). It is divided into three contiguous
ranges. The first 32 values (0-31, i.e. "20" - "2z" base-encoded) are reserved
for "best effort" collision-resistance TIDs. The bulk of the range, (32-991,
i.e. "30" - "yz" base-encoded) is reserved for context-dependent use. The
remaining 32 entries (992-1023, i.e. "z0" - "zz" base-encoded) are reserved for
globally unique TIDs.

"Best effort" node identifiers can be generated without coordination or deferral
to external authorities, but are considered likely to collide when merged with
data from external sources.

Context-dependent node identifiers should be use in the context of a specific
application where they can be derived stably from application context. The
application developer should take steps to ensure the that in any given time
range, no node identifiers are in simultaneous use by two different actors. No
process is specified for coordinating leases of node identifiers to actors.

Globally-unique node identifiers should only be used after being registered
globally. At time of writing, there is only one public TimeID service
operating.

| ClockID | URL               | public key | from          | to      |
|---------|-------------------|------------|---------------|---------|
| z0      | http://ccn.bz/tid | todo       | 2222-222-2222 | ongoing |

## Base-Encoded String Expression

The TimeID concatenates the timestamp and the node identifier. The string format
is 11 code points of timestamp and 2 code points of node identifier, displayed
for readability with `-` segment dividers after the 4th, 7th, and 11th code
points:

~~~~ bash
STTT-TTT-TTTT-CC
~~~~

Where:

* S represents a character with limited range, representing 3 bytes of timestamp
* each T represents 5-bytes of the timestamp, and
* each C represent a 5-byte character from the node identifier

The timestamp is expressed in a string-form TID as the first 11 codepoints, i.e.
55 bytes, but in the binary form as the first 54 bytes of an `int64`. Note that
the range of times that fit into those 54 bytes is actually a little smaller
than 0 +/- (2^55-1); namely, it is 0 +/- (2^53-1). This is to accomodate various
type-system quirks in the targeted languages and encodings: firstly, unsigned
integers are problematic for JSON encoding (particularly JSON tooling),
requiring the "assumed byte" (the 55th byte hard-coded into the decoding
process) to designate a positive or negative number. Similarly, the 54th byte
has to "sign extend" that positive or negative sign to keep the total integer
expressible in a sign-extended 63 bytes to accomodate a quirk of the Java type
system that converts integers bigger than 63 bytes to `float64`s. Sacrificing
these two bytes of range safeguards round-trip translation of these `int64`s to
JSON or Java and back.

We can take the timestamp `2024-07-19T09:40:46.480310` as an example
to show the process. This is `1721382046481000` microseconds since Unix
epoch. On a node with identifer `01`, this base-encodes to:

~~~~ bash
3kxn-lhr-3gxq-23
 | |   |    |  |
 | |   |    |  NodeID 0-1023
 | |   |    microsecond
 | |   second
 | 10 hours (9.54 hours)
 year (1.115 years)
~~~~

When using TimeIDs in string form, "prefixes" can be used to mask ranges, i.e.

| start                       | end                          | tid            |
|-----------------------------|------------------------------|----------------|
|`2024-07-19T09:40:46.480310` | `2024-07-19T09:40:46.480310` | `3kxn-lhr-3gxq`|
|`2024-07-19T09:40:46.434304` | `2024-07-19T09:40:47.482879` | `3kxn-lhr`     |
|`2024-07-19T04:28:52.498432` | `2024-07-19T14:01:32.236799` | `3kxn`         |
|`2023-07-08T13:57:40.263936` | `2024-08-18T19:23:52.352767` | `3k`           |

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Security Considerations

This minimal format makes very few guarantees and is built specifically for
distributed systems without assuming coordinated time-keeping. The security
implications of application-specific coordination of node identifiers are left
as an exercise for the reader.

# IANA Considerations

Currently, the registry for global TID services is maintained in a git
repository maintained by the authors at github dot com. In the unlikely event
that this this specification should become normative, a more formal registry
governed according to IANA best current practice would be justified.

--- back

# Test Vectors

For clarity, we've cross-computed a UUIDv6 for the test vector used above, as
well as computed a TID for the test vector given for UUIDv6 in {{?RFC9562}},
i.e. the 60-bit timestamp: `0x1EC9414C232AB00` (138648505420000000), i.e.
Tuesday, February 22, 2022 2:22:22.000000 PM GMT-05:00.

| Timestamp                  | milliseconds  |
|----------------------------|---------------|
| 2024-07-19T09:40:46.480310 | 1721382046481 |
| 2022-02-02T07:22:22.000000 | 1645557742000 |

| microseconds     | tid           | UUIDv6                               |
|------------------|---------------|--------------------------------------|
| 1721382046481000 | 3kxn-lhr-3gxq | todo                                 |
| 1645557742000000 | 3iso-34e-qpw2 | 1EC9414C-232A-6B00-B3C8-9F6BDECED846 |

# Acknowledgments

TODO acknowledge.
