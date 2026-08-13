# Domain Name System: Consolidated Specification

Merges the message-format and behavioural provisions of RFC 1034 and RFC 1035 with
the amendments in RFC 2181, RFC 2308, RFC 2782, RFC 3596, RFC 3597, RFC 4035,
RFC 6891 and RFC 8767. Master file syntax, inverse queries, zone transfer mechanics
and DNSSEC validation are out of scope.

Every provision is annotated with its source. A citation of the form
[RFC1035, Section 4.1.1] means that provision is drawn from Section 4.1.1 of
RFC 1035; a citation naming two documents means the provision is assembled from
both. Where documents conflict, the later one governs and the conflict is stated
explicitly rather than silently resolved.

**Verification status.** Sections sourced to RFC 1034, RFC 1035, RFC 3597 and
RFC 6891 were written against the full text of those documents. The provisions
attributed to RFC 2181, Section 8 and RFC 8767, Section 4 were verified against those sections
directly. RFC 3596 (AAAA) and RFC 2782 (SRV) were not read in full; the field
layouts given for those two types are from general knowledge and should be checked
against the RFCs before relying on them. Provisions marked **[convention]** are
implementation practice with no RFC citation - they are flagged rather than dressed
up as normative requirements.

---

## 1. Introduction

### 1.1 Requirements Language

MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are to be interpreted as described in
RFC 2119. RFC 1034 and RFC 1035 predate that convention; requirement levels
attributed to them here reflect their prose, which uses "must", "should" and
"required" informally.

### 1.2 Conventions [RFC1035, Section 2.3.2]

Transmission order is resolved to the octet level. Where a diagram shows a group of
octets, they are transmitted in reading order. Within an octet representing a
numeric quantity, the leftmost bit in the diagram is the most significant, numbered
bit 0. For a multi-octet quantity the leftmost bit of the whole field is the most
significant, and the most significant octet is transmitted first.

An _offset_ is a count of octets from the first octet of the message - that is, from
the first octet of the ID field. Offset zero is the first octet of ID [RFC1035, Section 4.1.4].

### 1.3 Terminology [RFC1034, Section 2.4, Section 3.1, Section 3.6; RFC9499]

- **Zone** - a contiguous region of the name space administered as a unit.
- **Label** - an octet string forming one component of a domain name.
- **Name** - an ordered sequence of labels terminated by the root label.
- **Root label** - the zero-length label terminating every name.
- **Owner** - the name at which a resource record is found.
- **RDATA** - the type- and class-dependent payload of a resource record.
- **RRset** - the records at one owner sharing type and class. All records in an
  RRset MUST carry the same TTL [RFC2181, Section 5.2].
- **Authoritative** - holding zone data from local configuration rather than cache.
- **Zone cut** - the boundary between a parent zone and a delegated child zone.
- **Glue** - address records for a name server whose name lies below the cut it
  serves; carried in the parent zone, not part of its authoritative data, and used
  only in referrals [RFC1034, Section 4.2.1].
- **Requestor / Responder** - the sides that send a request and answer it [RFC6891, Section 2].

---

## 2. Message Format [RFC1035, Section 4.1]

```
+---------------------+
|       Header        |  12 octets, always present
+---------------------+
|      Question       |  QDCOUNT entries
+---------------------+
|       Answer        |  ANCOUNT resource records
+---------------------+
|      Authority      |  NSCOUNT resource records
+---------------------+
|      Additional     |  ARCOUNT resource records
+---------------------+
```

Sections are contiguous, without padding or delimiters. All four are of variable
length; a parser MUST process them in order, since the position of each section is
determined only by the lengths of those preceding it. The last three sections share
an identical format: a possibly empty list of concatenated resource records.

The header specifies which of the remaining sections are present. The content, but
not the format, of the sections varies with OPCODE [RFC1034, Section 3.7].

The answer section holds records answering the question; the authority section holds
records pointing toward an authoritative server; the additional section holds
records related to the query but not strictly answers.

### 2.1 Header [RFC1035, Section 4.1.1]

```
                                 1  1  1  1  1  1
   0  1  2  3  4  5  6  7  8  9  0  1  2  3  4  5
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                       ID                      |
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |QR|   OPCODE  |AA|TC|RD|RA| Z|AD|CD|   RCODE   |
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                    QDCOUNT                    |
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                    ANCOUNT                    |
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                    NSCOUNT                    |
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                    ARCOUNT                    |
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

> **Amendment.** RFC 1035, Section 4.1.1 shows bits 9-11 as a single three-bit reserved
> field named Z, with the diagram reading `|QR| Opcode |AA|TC|RD|RA| Z | RCODE |`.
> RFC 4035 subsequently allocated bits 10 and 11 as AD and CD. The diagram above
> reflects the current allocation; an implementation reading RFC 1035 alone will see
> the older one.

**ID** (16 bits) - assigned by the program generating the query and copied into the
corresponding reply, allowing the requestor to match replies to outstanding queries.

**QR** (1 bit) - query (0) or response (1).

**OPCODE** (4 bits) - the kind of query. Set by the originator and copied into the
response.

**AA** (1 bit) - Authoritative Answer. Valid in responses; indicates the responder is
an authority for the name in the question section. Note that the answer section may
carry several owner names because of aliases: AA corresponds to the name matching
the query name, or to the first owner name in the answer section. It therefore
guarantees nothing about other names in the response [RFC1035, Section 4.1.1; RFC1034, Section 6.2.7].
Responses to QCLASS=\* queries can never be authoritative unless the responder can
guarantee coverage of all classes, which in practice it cannot [RFC1034, Section 3.7.1; RFC1035, Section 6.2].

**TC** (1 bit) - TrunCation. The message was shortened because its length exceeded
what the transmission channel permits.

**RD** (1 bit) - Recursion Desired. May be set in a query and is copied into the
response. When set it directs the server to pursue the query recursively; support is
optional. A server should not perform recursive service unless asked via RD, since
doing so obstructs diagnosis of servers and their databases [RFC1034, Section 4.3.1].

**RA** (1 bit) - Recursion Available. Set or cleared in every response according to
whether the server is willing to provide recursive service, irrespective of whether
it was requested: RA signals availability, not use. A requestor confirms recursion
occurred by observing both RD and RA set in the reply [RFC1034, Section 4.3.1].

**Z** (1 bit) - Reserved. MUST be zero in all queries and responses.

**AD** (1 bit) - Authentic Data [RFC4035]. Zero where DNSSEC validation is not done.

**CD** (1 bit) - Checking Disabled [RFC4035]. Zero where DNSSEC validation is not done.

**RCODE** (4 bits) - set as part of responses. Extended to 12 bits by the OPT
mechanism (Section 8).

**QDCOUNT / ANCOUNT / NSCOUNT / ARCOUNT** (16 bits each, unsigned) - the number of
entries in the question section and of records in the answer, authority and
additional sections respectively. Each MUST equal the number actually present.

#### 2.1.1 OPCODE

| Value | Mnemonic | Source                                                        |
| ----- | -------- | ------------------------------------------------------------- |
| 0     | QUERY    | Standard query [RFC1035, Section 4.1.1]                       |
| 1     | IQUERY   | Inverse query [RFC1035, Section 4.1.1]; obsoleted by RFC 3425 |
| 2     | STATUS   | Server status request [RFC1035, Section 4.1.1]                |
| 4     | NOTIFY   | RFC 1996                                                      |
| 5     | UPDATE   | RFC 2136                                                      |

RFC 1035 marks 3-15 as reserved; 4 and 5 were allocated subsequently.

A server receiving an OPCODE it does not support returns a Not Implemented error;
support for returning that error is itself mandatory even where the operation is
optional [RFC1034, Section 3.7.2; RFC1035, Section 6.4].

#### 2.1.2 RCODE [RFC1035, Section 4.1.1]

| Value | Mnemonic | Meaning                                                                                                                        |
| ----- | -------- | ------------------------------------------------------------------------------------------------------------------------------ |
| 0     | NOERROR  | No error condition                                                                                                             |
| 1     | FORMERR  | The server was unable to interpret the query                                                                                   |
| 2     | SERVFAIL | The server could not process the query owing to its own problem                                                                |
| 3     | NXDOMAIN | The name does not exist; meaningful only from an authoritative server                                                          |
| 4     | NOTIMP   | The server does not support the requested kind of query                                                                        |
| 5     | REFUSED  | Refused for policy reasons - e.g. withholding data from a particular requestor, or declining zone transfer for particular data |
| 16    | BADVERS  | EDNS version not implemented [RFC6891, Section 6.1.3, Section 9]                                                               |

RFC 1035 marks 6-15 reserved; further values are registered with IANA.

Temporary failure MUST NOT be signalled to an application as name-nonexistence.
Conflating them is disruptive to users and can wreak havoc in mail systems
[RFC1034, Section 5.2.3]. Similarly, "the name does not exist" and "the name exists but holds
no data of this type" are distinct conditions; the general lookup interface should
not merge them, since doing so causes applications to issue useless follow-up
queries [RFC1034, Section 5.2.1].

### 2.2 Question Section [RFC1035, Section 4.1.2]

```
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 /                     QNAME                     /
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                     QTYPE                     |
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                    QCLASS                     |
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

QDCOUNT is usually 1. QNAME may occupy an odd number of octets; no padding is used.
QTYPE and QCLASS are each two octets, their value spaces supersets of TYPE and CLASS
[RFC1034, Section 3.7.1].

A responder MUST reproduce the question section in the response without alteration.

### 2.3 Resource Record [RFC1035, Section 4.1.3, Section 3.2.1]

```
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 /                     NAME                      /
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                     TYPE                      |
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                     CLASS                     |
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                      TTL                      |
 |                                               |
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                   RDLENGTH                    |
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 /                     RDATA                     /
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

**NAME** - the owner name, i.e. the node to which the record pertains.

**TYPE** (16 bits) - determines the interpretation of RDATA.

**CLASS** (16 bits) - the class of the data in RDATA.

**TTL** - the interval in seconds for which the record may be cached before the
source should again be consulted. Zero means the record may be used only for the
transaction in progress and MUST NOT be cached; SOA records are conventionally
distributed with a zero TTL for this reason [RFC1035, Section 3.2.1].

> **Conflict.** RFC 1035 defines this field three ways: Section 2.3.4 as positive values of
> a signed 32-bit number, Section 3.2.1 as a 32-bit signed integer, and Section 4.1.3 as a 32-bit
> unsigned integer. RFC 2181, Section 8 exists to settle it and specifies unsigned, minimum
> 0, maximum 2^31-1, transmitted with the sign bit zero. RFC 8767, Section 4 further amends
> the definition to a 32-bit unsigned integer and directs that received values with
> the high-order bit set be interpreted as positive rather than as zero - reversing
> the practice that followed from RFC 2181 - and recommends capping at 604800
> seconds (7 days).

The TTL bounds caching only; it does not apply to authoritative zone data, which is
aged instead by the zone's refresh policy [RFC1034, Section 3.6].

**RDLENGTH** (16 bits, unsigned) - the length in octets of RDATA. Where a name inside
RDATA is compressed, the compressed length is what counts [RFC1035, Section 4.1.4].

**RDATA** - a variable-length octet string whose format varies with TYPE and CLASS.

The order of records within an RRset is not significant and need not be preserved by
servers, resolvers, or any other part of the DNS [RFC1034, Section 3.6].

---

## 3. Domain Name Representation

### 3.1 Encoding [RFC1035, Section 3.1; RFC1034, Section 3.1]

A name is a sequence of labels, each a one-octet length field followed by that many
octets. Since every name ends at the root, whose label is the null string, a name is
terminated by a length octet of zero.

```
example.com encodes as:
07 65 78 61 6d 70 6c 65  03 63 6f 6d  00
 7  e  x  a  m  p  l  e   3  c  o  m  root
```

RFC 1035, Section 3.1 states that the high-order two bits of every length octet must be
zero, which is what confines a label to 63 octets. Section 4.1.4 then carves out the value
`11` for compression pointers and notes that `01` and `10` are reserved:

| Bits | Interpretation                                                 | Source                                                    |
| ---- | -------------------------------------------------------------- | --------------------------------------------------------- |
| 00   | The remaining six bits give the length of the label following  | [RFC1035, Section 3.1]                                    |
| 01   | Reserved; later used for extended label types, now discouraged | [RFC1035, Section 4.1.4; RFC6891, Section 4.2, Section 5] |
| 10   | Reserved                                                       | [RFC1035, Section 4.1.4]                                  |
| 11   | A compression pointer (Section 3.5)                            | [RFC1035, Section 4.1.4]                                  |

RFC 2671 defined `01` as an extended-label indicator, selecting a specific extended
type from the low six bits, i.e. first-octet values 64-127. RFC 6891 retains the
allocation but discourages its use, deprecates Binary Labels outright, and requires
that implementations neither generate nor pass them [RFC6891, Section 5]. A simple
implementation should reject `01` and `10`.

### 3.2 Size Limits [RFC1035, Section 2.3.4, Section 3.1; RFC1034, Section 3.1]

| Object                                         | Limit                                                                                |
| ---------------------------------------------- | ------------------------------------------------------------------------------------ |
| Label                                          | 63 octets or less                                                                    |
| Name (all label octets plus all length octets) | 255 octets or less                                                                   |
| UDP message                                    | 512 octets or less, absent EDNS(0)                                                   |
| `<character-string>`                           | up to 256 octets including its length octet, i.e. 255 of data [RFC1035, Section 3.3] |
| NULL RDATA                                     | 65535 octets or less [RFC1035, Section 3.3.10]                                       |

The 255-octet name limit exists to simplify implementations. Both it and the label
limit MUST be enforced on receipt. **[convention]** - RFC 1035 states the limits but
does not spell out receiver enforcement; enforcing them is what prevents unbounded
allocation from a crafted message.

### 3.3 Character Case [RFC1035, Section 2.3.3; RFC1034, Section 3.1]

All comparisons between character strings - labels, names - are case-insensitive
throughout the official protocol, without exception, assuming ASCII with zero
parity. Non-alphabetic codes must match exactly. Labels may contain any 8-bit
values, so storing names in 7-bit ASCII or using special octets to terminate labels
should be avoided against future binary-name use.

Original case should be preserved wherever possible. It may be discarded only where
data defines structure in a database and two names compare identical
case-insensitively. In general this preserves the case of the first label while
standardising interior labels.

RFC 3597, Section 3 adds that the case of embedded domain names not subject to compression
MUST be preserved exactly, so that equality comparison behaves identically whether
or not a server knows the type.

### 3.4 Preferred Name Syntax [RFC1035, Section 2.3.1; RFC1034, Section 3.5]

The protocol constrains label content only by length. For compatibility with
applications such as mail and TELNET, host names should follow:

```
<domain>      ::= <subdomain> | " "
<subdomain>   ::= <label> | <subdomain> "." <label>
<label>       ::= <letter> [ [ <ldh-str> ] <let-dig> ]
<ldh-str>     ::= <let-dig-hyp> | <let-dig-hyp> <ldh-str>
<let-dig-hyp> ::= <let-dig> | "-"
<let-dig>     ::= <letter> | <digit>
```

Begin with a letter, end with a letter or digit, use only letters, digits and hyphen
internally. This is guidance for assigning names, not a constraint a parser may
impose on received ones - RFC 1035, Section 3.1 is explicit that labels can contain any
octet values.

### 3.5 Message Compression [RFC1035, Section 4.1.4]

To reduce message size, an entire name or a trailing list of its labels may be
replaced by a pointer to a prior occurrence of the same name in that message.

```
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 | 1  1|                OFFSET                   |
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

The leading two bits are ones, which distinguishes a pointer from a label since a
label must begin with two zero bits. OFFSET counts from the start of the message.

A name in a message may therefore take one of exactly three forms:

1. a sequence of labels ending in a zero octet;
2. a pointer;
3. a sequence of labels ending with a pointer.

Because a pointer's meaning depends on the surrounding message, resolving one
requires access to the whole message, not merely the octets following the current
parse position.

**Where compression may be emitted.** RFC 1035, Section 4.1.4 permits pointers only where
the format of the name's position is not class-specific, so that a server need not
know the format of every type it handles. Section 3.3 states that because their RDATA
format is known, names in the RDATA of NS, SOA, CNAME and PTR - and the other types
defined in that section, which includes MX - may be compressed.

RFC 3597, Section 4 tightens this. Copying RDATA containing pointers into a new message
would leave those pointers aimed at unrelated data, corrupting the name. Therefore:

- Servers MUST NOT compress names embedded in the RDATA of types that are
  class-specific or not well-known. "Well-known" is defined to mean exactly the
  types defined in RFC 1035.
- Receiving servers MUST decompress names in RDATA of well-known types, and SHOULD
  also decompress RP, AFSDB, RT, SIG, PX, NXT, NAPTR and SRV. **SRV is the trap**:
  RFC 2782 prohibits compressing it, but the earlier RFC 2052 mandated it, and
  servers following the earlier specification remain in use. A receiver should
  handle a compressed SRV target even though it must never emit one.
- New RR types containing names MUST NOT permit compression of them.
- The owner name of a record is always eligible for compression.

RFC 3597, Section 4 also updates RFC 2163 and RFC 2535, which had explicitly allowed
compression for PX, SIG and NXT.

**Emitting is optional.** Programs are free to avoid pointers in messages they
generate, at the cost of datagram capacity and possible truncation. All programs are
required to understand arriving messages that contain them [RFC1035, Section 4.1.4].

**Loop and direction constraints.** **[convention]** RFC 1035 says a pointer refers
to a _prior_ occurrence but does not state a receiver requirement to reject forward
or self-referential pointers, nor a bound on pointers followed while resolving one
name. Neither appears in RFC 1034, 1035, 3597 or 6891. Both are nonetheless
necessary: without them a single crafted datagram drives an unbounded loop. Require
that each pointer target lie strictly below the offset of the pointer itself, and
cap the number followed. The closest RFC analogue is the resolver's global
per-request work counter of [RFC1035, Section 7.1], which exists for the same class of reason.

**Worked example [RFC1035, Section 4.1.4].** A message needing F.ISI.ARPA, FOO.F.ISI.ARPA, ARPA
and the root might encode them as:

```
offset 20:  01 'F' 03 'I' 'S' 'I' 04 'A' 'R' 'P' 'A' 00
offset 40:  03 'F' 'O' 'O' [pointer to 20]
offset 64:  [pointer to 26]
offset 92:  00
```

F.ISI.ARPA sits at offset 20. FOO.F.ISI.ARPA at offset 40 prefixes one label and
points at 20. ARPA at offset 64 points at **offset 26** - into the middle of the name
at 20, which works only because ARPA is that name's final label. The root at offset
92 is a single zero octet and has no labels. This example is the reason a decoder
must accept pointers into name interiors, not merely to name starts.

---

## 4. Type and Class Values

### 4.1 TYPE [RFC1035, Section 3.2.2]

| Value | Mnemonic | Meaning / RDATA                                        |
| ----- | -------- | ------------------------------------------------------ |
| 1     | A        | Host address - Section 5.1                             |
| 2     | NS       | Authoritative name server - Section 5.3                |
| 3     | MD       | Mail destination; obsolete, use MX                     |
| 4     | MF       | Mail forwarder; obsolete, use MX                       |
| 5     | CNAME    | Canonical name for an alias - Section 5.3              |
| 6     | SOA      | Start of a zone of authority - Section 5.7             |
| 7     | MB       | Mailbox domain name; experimental                      |
| 8     | MG       | Mail group member; experimental                        |
| 9     | MR       | Mail rename domain name; experimental                  |
| 10    | NULL     | Null record; experimental, not allowed in master files |
| 11    | WKS      | Well known service description                         |
| 12    | PTR      | Domain name pointer - Section 5.3                      |
| 13    | HINFO    | Host information: CPU and OS character strings         |
| 14    | MINFO    | Mailbox or mail list information; experimental         |
| 15    | MX       | Mail exchange - Section 5.4                            |
| 16    | TXT      | Text strings - Section 5.5                             |
| 28    | AAAA     | IPv6 address [RFC3596] - Section 5.2                   |
| 33    | SRV      | Service location [RFC2782] - Section 5.6               |
| 41    | OPT      | EDNS(0) pseudo-record [RFC6891] - Section 8            |

### 4.2 QTYPE [RFC1035, Section 3.2.3]

All TYPE values are valid QTYPEs. In addition:

| Value | Mnemonic | Meaning                                            |
| ----- | -------- | -------------------------------------------------- |
| 252   | AXFR     | Request for transfer of an entire zone             |
| 253   | MAILB    | Request for mailbox-related records (MB, MG or MR) |
| 254   | MAILA    | Request for mail agent records; obsolete, see MX   |
| 255   | \*       | Request for all records                            |

Note that RFC 1035 marks MAILA obsolete but not MAILB.

### 4.3 CLASS and QCLASS [RFC1035, Section 3.2.4, Section 3.2.5]

| Value | Mnemonic | Meaning                                                           |
| ----- | -------- | ----------------------------------------------------------------- |
| 1     | IN       | The Internet                                                      |
| 2     | CS       | The CSNET class; obsolete, used only in examples in obsolete RFCs |
| 3     | CH       | The CHAOS class                                                   |
| 4     | HS       | Hesiod                                                            |
| 255   | \*       | Any class; QCLASS only                                            |

Classes partition the database completely: data for each class is organised,
delegated and maintained separately, over what is by convention the same name space
[RFC1034, Section 4.2].

TYPE and CLASS values must be a proper subset of QTYPEs and QCLASSes respectively
[RFC1035, Section 3.6]. The authoritative registries are maintained by IANA under "Domain Name
System (DNS) Parameters"; the tables above are the RFC 1035 set plus the three later
types in common use.

---

## 5. RDATA Formats

### 5.1 A [RFC1035, Section 3.4.1]

For class IN, a 32-bit Internet address. RDLENGTH is 4. Hosts with several addresses
have several A records. A causes no additional section processing.

For class CH the format differs: a name followed by a 16-bit octal Chaos address
[RFC1034, Section 3.6].

### 5.2 AAAA [RFC3596, Section 2.2] - _not verified against source_

Sixteen octets, an IPv6 address. RDLENGTH is 16.

### 5.3 NS, CNAME, PTR [RFC1035, Section 3.3.11, Section 3.3.1, Section 3.3.12]

A single `<domain-name>`.

**NS** asserts that the named host should have a zone starting at the owner name for
the given class. It causes the usual additional section processing to locate an A
record and, when used in a referral, a special search of the zone it resides in for
glue. NS records appear only at the apex of a zone, where they are authoritative,
and at cuts around its bottom, where they are not - never in between [RFC1034, Section 4.2.1].

**CNAME** causes no additional section processing, though servers may restart the
query at the canonical name (Section 6.1).

**PTR** causes no additional section processing. Unlike CNAME it is simple data and
implies no special handling.

### 5.4 MX [RFC1035, Section 3.3.9]

```
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                  PREFERENCE                   |
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 /                   EXCHANGE                    /
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

PREFERENCE is a 16-bit integer ranking this record among others at the same owner;
lower is preferred. EXCHANGE is a name willing to act as mail exchange for the owner.
MX causes type A additional section processing for EXCHANGE.

### 5.5 TXT [RFC1035, Section 3.3.14, Section 3.3]

One or more `<character-string>`s, each a length octet followed by that many
characters, packed to fill exactly RDLENGTH. A character string is treated as binary
and may be up to 256 octets including its length octet. The RDATA is a _sequence_ of
strings, not one string.

### 5.6 SRV [RFC2782] - _not verified against source_

```
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                   PRIORITY                    |
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                    WEIGHT                     |
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                     PORT                      |
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 /                    TARGET                     /
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

TARGET MUST NOT be compressed on emission, but see Section 3.5 on receiving
compressed SRV from implementations following RFC 2052.

### 5.7 SOA [RFC1035, Section 3.3.13]

```
 /                    MNAME                      /
 /                    RNAME                      /
 |                    SERIAL                     |  32 bits
 |                   REFRESH                     |  32 bits
 |                    RETRY                      |  32 bits
 |                   EXPIRE                      |  32 bits
 |                   MINIMUM                     |  32 bits
```

All times are in seconds. SOA causes no additional section processing.

**MNAME** - the name server that was the original or primary source of data for the
zone. RFC 2181, Section 7.3 emphasises that this is the primary server's name, not the zone's
own name, which is already known from the record's owner.

**RNAME** - the mailbox of the person responsible for the zone, the local part
encoded as a single label regardless of dots within it [RFC1035, Section 8].

**SERIAL** - the unsigned 32-bit version number of the zone, preserved across zone
transfers. It wraps and must be compared using sequence space arithmetic. It is
advanced on every change to the zone [RFC1034, Section 4.3.5].

**REFRESH** - the interval before the zone should be refreshed.
**RETRY** - the interval before a failed refresh is retried.
**EXPIRE** - the upper limit on how long the zone may go unchecked before it is no
longer authoritative and must be discarded.

**MINIMUM** - as originally defined, the minimum TTL exported with any record from
the zone: on sending a record in a response, the TTL is set to the maximum of the
record's own TTL and the SOA MINIMUM, making MINIMUM a floor. RFC 1035 is explicit
that this floor is applied when records are copied into a response, not when the
zone is loaded, so that dynamic update can change the SOA with known semantics
[RFC1035, Section 3.3.13, Section 6.2].

> **Amendment.** RFC 2308, Section 4 redefines MINIMUM as solely the TTL to be used for
> negative responses, discarding the floor function.

### 5.8 Unrecognised Types [RFC3597]

An RR of unknown type is one whose RDATA format the implementation does not know and
whose type is not an assigned QTYPE or Meta-TYPE. Where a type's RDATA format is
class-specific, a record counts as unknown when the format for that type-and-class
combination is unknown.

Servers and resolvers MUST handle such records transparently, treating the RDATA as
unstructured binary data and storing and transmitting it unchanged. Servers MUST
also preserve exactly the RDATA of records of _known_ type, except for changes due
to compression or decompression where Section 3.5 permits them.

Unknown types cause no additional section processing.

---

## 6. Aliases and Wildcards

### 6.1 CNAME [RFC1034, Section 3.6.2]

A CNAME record identifies its owner as an alias and gives the canonical name in its
RDATA. Where a CNAME is present at a node, no other data should be present. This
ensures the data for a canonical name and its aliases cannot differ, and lets a
cached CNAME be used without consulting an authority for other types.

When a server fails to find the requested type at a name and the resource set there
consists of a CNAME of matching class, it includes the CNAME in the response and
restarts the query at the canonical name. Queries matching type CNAME are not
restarted - a CNAME or `*` query returns just the CNAME.

Records pointing at other names should point at the canonical name rather than an
alias, avoiding extra indirection. By the robustness principle, software must not
fail when presented with CNAME chains or loops: chains are followed, loops signalled
as an error. Multiple levels of aliasing are inefficient but are not an error
[RFC1034, Section 5.2.2].

### 6.2 Wildcards [RFC1034, Section 4.3.3]

A record whose owner begins with the label `*` is a wildcard - an instruction for
synthesising records rather than a record served as such. Its owner has the form
`*.<anydomain>`, which should contain no other `*` labels and should lie in the
zone's authoritative data. The wildcard applies to descendants of `<anydomain>` but
not to `<anydomain>` itself; `*` matches at least one whole label and sometimes
more, but always whole labels.

Wildcards do not apply:

- when the query is in another zone - delegation cancels the wildcard default;
- when the query name, or any name between the wildcard domain and the query name,
  is known to exist. If a wildcard owns `*.X` and the zone also holds records at
  `B.X`, the wildcard applies to `Z.X` but not to `B.X`, `A.B.X` or `X`.

When synthesising, the owner of the produced record is set to QNAME rather than to
the wildcard node, and the wildcard's RDATA is used unmodified.

A `*` in a _query_ name has no special effect, but is the only way to retrieve
wildcard records as such; results of such a query should not be cached.

---

## 7. Transport [RFC1035, Section 4.2]

Messages travel as datagrams or over a virtual circuit. Datagrams are preferred for
queries owing to lower overhead; zone refresh must use virtual circuits for reliable
transfer. Both TCP and UDP use port 53.

### 7.1 UDP [RFC1035, Section 4.2.1]

Messages are restricted to 512 octets, not counting the IP or UDP headers. Longer
messages are truncated and TC is set.

UDP is not acceptable for zone transfers but is the recommended method for standard
queries. Queries may be lost, so a retransmission strategy is required. Queries and
responses may be reordered by the network or by server processing, so resolvers must
not depend on ordering.

Recommended retransmission policy: try other servers and other addresses of a server
before repeating a query to one address; base the interval on prior statistics where
possible; keep the minimum interval at 2-5 seconds. Too aggressive retransmission
easily degrades response times for the community at large.

### 7.2 TCP [RFC1035, Section 4.2.2]

Each message is prefixed with a two-octet length field giving the message length,
excluding the two length octets themselves. The prefix lets low-level processing
assemble a complete message before parsing begins.

Recommended connection management: do not block other activity waiting for TCP data;
support multiple connections; assume the client initiates closing and delay closing
until outstanding requests are satisfied; if reclaiming a dormant connection, wait
until it has been idle on the order of two minutes, and in particular allow the
SOA-then-AXFR sequence on a single connection.

### 7.3 Truncation [RFC1035, Section 4.1.1, Section 6.2]

Where a response is too long, truncation should begin at the end of the response and
work forward through the datagram. Consequently, if any data remains for the
authority section, the answer section is guaranteed to be complete.

**[convention]** RFC 1035 does not spell out that a partial record must never be
transmitted, but truncating record-by-record is what the "work forward from the end"
rule amounts to in practice, and header counts must reflect what was actually sent.

A requestor receiving TC may reissue the query over TCP.

---

## 8. EDNS(0) and the OPT Pseudo-RR [RFC6891]

EDNS is a hop-by-hop extension, negotiated between each pair of hosts in a resolution
chain - stub to recursive, recursive to authoritative [RFC6891, Section 1]. Any implementation
that forwards queries upstream will encounter it.

### 8.1 Wire Format [RFC6891, Section 6.1.2]

An OPT record has type 41 and reuses the standard record fields with entirely
different meanings:

| Field    | Meaning in OPT                               |
| -------- | -------------------------------------------- |
| NAME     | MUST be 0 - the root domain                  |
| TYPE     | 41                                           |
| CLASS    | The requestor's UDP payload size             |
| TTL      | Extended RCODE and flags (Section 8.2)       |
| RDLENGTH | Length of all RDATA                          |
| RDATA    | Zero or more {attribute, value} option pairs |

Each option in RDATA is encoded as:

```
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                  OPTION-CODE                  |  16 bits
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |                 OPTION-LENGTH                 |  16 bits, octets of data
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 /                  OPTION-DATA                  /
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

Option order is not defined and carries no meaning. Any OPTION-CODE not understood
MUST be ignored.

### 8.2 The TTL Field [RFC6891, Section 6.1.3, Section 6.1.4]

```
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |         EXTENDED-RCODE        |    VERSION    |   octets 0-1
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
 |DO|                   Z                        |   octets 2-3
 +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```

**EXTENDED-RCODE** forms the upper 8 bits of a 12-bit RCODE, the lower 4 coming from
the header. A value of 0 means an unextended RCODE (0-15) is in use.

**VERSION** - full conformance with RFC 6891 is version 0. A responder not
implementing the requested version MUST reply with BADVERS (extended RCODE 16).

**DO** - DNSSEC OK [RFC3225]. **Z** - set to zero by senders, ignored by receivers.

### 8.3 Behaviour [RFC6891, Section 6.1.1, Section 6.2, Section 7]

- OPT MAY be added to the additional section of a request, and MAY sit anywhere
  within that section. It carries no DNS data - only control information for one
  transaction.
- OPT MUST NOT be cached, forwarded, or stored in or loaded from master files.
- At most one OPT per message. A query with more than one MUST get FORMERR.
- If a request contains OPT, a compliant responder MUST include OPT in its response.
- If a request contains **no** OPT, the responder MUST NOT include one.
- A responder not implementing EDNS MUST return FORMERR to a message containing OPT
  and MUST NOT include OPT in that response. A responder that _does_ implement EDNS
  but finds the OPT record itself malformed MUST return FORMERR _with_ an OPT record
  - the presence or absence of OPT in the error is how a requestor distinguishes the
    two cases.
- The minimal response is header, question section, and OPT - including when
  returning a truncated response with TC set.

### 8.4 Payload Size [RFC6891, Section 6.2.3, Section 6.2.5]

The requestor's payload size in the CLASS field is the largest UDP payload its
network stack can reassemble and deliver. Values below 512 MUST be treated as 512.
It MUST NOT be cached beyond the transaction advertising it.

4096 octets is a reasonable starting point. A requestor may fall back: start large
(4096), then 1280-1410 which has a fair chance of fitting one Ethernet frame, then
512 which may force TCP retry. Very large values guarantee IP fragmentation and can
cause answers to be lost to a single dropped fragment or a firewall that rejects
fragments.

Conformant middleboxes MUST NOT limit DNS over UDP to 512 bytes, and forwarding
middleboxes MUST NOT modify or delete OPT contents in either direction [RFC6891, Section 6.2.6].

---

## 9. Server Behaviour

### 9.1 Modes [RFC1034, Section 4.3.1]

Non-recursive operation is the simplest mode for a server: the response is generated
from local information alone and contains an error, the answer, or a referral to a
server closer to the answer. All name servers must implement non-recursive queries.

Recursive operation is the simplest mode for a client: the server acts as a resolver
and returns an error or an answer, never a referral. It is optional, and a server may
restrict which clients may use it.

### 9.2 Concurrency [RFC1035, Section 6.1.1]

A server must run multiple concurrent activities, whether as OS tasks or multiplexed
within one program. It is not acceptable to block UDP service while waiting for TCP
data for refresh or query activity. Recursive service must likewise be processed in
parallel, though a server may serialise requests from a single client or treat
identical repeats from one client as duplicates. Requests must not be substantially
delayed while a zone is reloaded or a refreshed zone is incorporated.

### 9.3 Query Processing Algorithm [RFC1034, Section 4.3.2]

The algorithm assumes records are held in one tree per zone plus a tree for the
cache; any equivalent organisation is permitted.

1. Set RA in the response according to whether recursive service is offered. If
   recursive service is available and requested via RD, go to step 5; otherwise go
   to step 2.

2. Search the available zones for the one that is the nearest ancestor of QNAME. If
   found, go to step 3; otherwise go to step 4.

3. Match down the zone label by label. Matching terminates in one of three ways:

   a. **QNAME matched in full.** If the data at the node is a CNAME and QTYPE is not
   CNAME, copy the CNAME into the answer section, change QNAME to the canonical
   name, and go back to step 1. Otherwise copy all records matching QTYPE into
   the answer section and go to step 6.

   b. **The match would leave authoritative data** - a node carrying NS records that
   mark a cut along the bottom of the zone. This is a referral. Copy the subzone's
   NS records into the authority section; put whatever addresses are available
   into the additional section, using glue where they are not available from
   authoritative data or the cache; go to step 4.

   c. **A label cannot be matched.** Look for a `*` label at that point. If it does
   not exist, check whether the name sought is the original QNAME or one reached
   via a CNAME: if original, set an authoritative name error and exit; otherwise
   just exit. If it does exist, match the records there against QTYPE and copy any
   that match into the answer section, setting the owner to QNAME rather than to
   the wildcard node; go to step 6.

4. Match down in the cache. If QNAME is found, copy all records attached to it that
   match QTYPE into the answer section. If no delegation came from authoritative
   data, take the best one from the cache and put it in the authority section. Go to
   step 6.

5. Answer the query using the local resolver or a copy of its algorithm, storing the
   results - including intermediate CNAMEs - in the answer section.

6. Using local data only, attempt to add other useful records to the additional
   section. Exit.

### 9.4 Composing Responses [RFC1035, Section 6.2]

Records destined for the additional section that duplicate records already in the
answer or authority sections may be omitted.

For QCLASS=\* or any QCLASS matching several classes, the response should never be
authoritative unless the server can guarantee it covers all classes.

A server should volunteer records it judges useful: a response carrying MX records
conventionally carries the exchanges' addresses in the additional section, on the
assumption the requestor will want them shortly [RFC1034, Section 3.7.1].

Where a server cannot load a zone from its master file, or cannot refresh it within
its expiration parameter, it should answer queries as though it were not supposed to
hold that zone at all [RFC1035, Section 6.3].

### 9.5 Negative Response Caching [RFC1034, Section 4.3.4; RFC2308]

A server may add the SOA of the zone that was the source of the authoritative data,
or of the name error, to the authority section of an authoritative response. The
MINIMUM field of that SOA bounds how long the negative result may be cached
[RFC2308, Section 4].

Where the answer section carries several owner names, the mechanism applies only to
data matching QNAME, that being the only authoritative data there.

Servers and resolvers must never add SOA records to a non-authoritative response, or
infer results not directly stated in an authoritative one: cached information is
generally insufficient to match records to their zone names, SOA records may be
cached from direct SOA queries, and servers are not required to emit SOAs in the
authority section at all. All resolvers and recursive servers are required at
minimum to be able to ignore an SOA present in a response.

---

## 10. Resolver Behaviour

### 10.1 State [RFC1034, Section 5.3.2; RFC1035, Section 7.1]

A request in progress is described by **SNAME**, the name sought; **STYPE** and
**SCLASS**; **SLIST**, the current best estimate of which servers hold the data,
including per-address history and a match count of labels shared between SNAME and
the zone being queried as a measure of closeness; **SBELT**, a safety belt of the
same form initialised from configuration with match count -1; and **CACHE**.

Authoritative data must always take precedence over cached data where both are held.

Each pending request also carries a **timestamp** marking when it began - used to
decide whether cached records are still timely, and superior to comparing against
current time because it lets zero-TTL records be cached normally yet still serve the
request that fetched them - and a **global per-request work counter**, decremented on
every action, which terminates the request with a temporary error on reaching zero.
Where one request spawns another in parallel, such as resolving a server's address,
the spawned request starts with a lower counter. This is what stops circular
references in the database from starting a chain reaction [RFC1035, Section 7.1].

SLIST is reinitialised whenever a delegation is followed [RFC1035, Section 7.2]. Where a
resolver finds no addresses for any server in SLIST, and those servers are precisely
the ones that would normally be used to look up their own addresses - which happens
when glue has a shorter TTL than the delegating NS records - it should detect the
condition and restart at the next ancestor zone or at the root.

### 10.2 Resolution Algorithm [RFC1034, Section 5.3.3]

1. See if the answer is in local information; if so return it.

2. Find the best servers to ask. Look for NS records starting at SNAME and working
   toward the root, copy the names into SLIST, and set up their addresses from local
   data. If the search for NS records fails, initialise SLIST from SBELT.

3. Send queries until one returns a response, cycling through all addresses of all
   servers with a timeout between transmissions. Each address chosen is then marked
   so it is not selected again until all others have been tried. Timeouts should be
   50-100% greater than the predicted average.

4. Analyse the response:

   a. If it answers the question or contains a name error, cache the data and return
   it to the client.
   b. If it contains a better delegation, cache the delegation information and go to
   step 2.
   c. If it shows a CNAME that is not itself the answer, cache the CNAME, set SNAME
   to the canonical name, and go to step 1.
   d. If it shows a server failure or otherwise bizarre contents, delete that server
   from SLIST and go back to step 3.

A resolver should be highly paranoid in parsing responses. It should check the header
for reasonableness and **discard datagrams that are queries when responses are
expected**; parse all sections and verify every record is correctly formatted; and
optionally check for excessively long TTLs, discarding the response or capping all
TTLs at one week [RFC1035, Section 7.3].

Match responses to requests first by the ID field, then by verifying the question
section corresponds to what is currently wanted - which requires devoting several
bits of the ID to a request identifier. Note that some servers reply from an address
other than the one queried, so a resolver cannot rely on source address matching
[RFC1035, Section 7.3].

A delegation must be checked to be closer to the answer than the servers already in
SLIST, by comparing match counts. A delegation that is not closer is bogus and must
be ignored [RFC1034, Section 5.3.3].

Data is entered in the cache only where its TTL exceeds zero.

### 10.3 What Not To Cache [RFC1035, Section 7.4]

- A partial set of records for one owner and type. When a response is truncated and
  the resolver cannot tell whether the set is complete, cache none of it.
- Anything that would end up taking precedence over authoritative data.
- The results of inverse queries.
- Results of queries whose QNAME contains `*` labels, where the data might be used
  to construct wildcards - the cache does not necessarily hold the zone boundary
  information needed to restrict wildcard application.
- Unsolicited responses, or record data other than what was requested. All sanity
  checks must be performed before anything is cached.

Where a resolver holds records for a name and receives more, it should check the
cache first: depending on circumstances either the response or the cache is
preferred, but the two must never be combined. Authoritative data from the answer
section is always preferred.

### 10.4 Design Priorities [RFC1034, Section 5.3.3]

In order of precedence:

1. Bound the work performed - packets sent, parallel processes started - so that a
   request cannot enter an infinite loop or start a chain reaction of requests with
   other implementations, _even if someone has incorrectly configured some data_.
2. Get back an answer if at all possible.
3. Avoid unnecessary transmissions.
4. Get the answer as quickly as possible.

The ordering is deliberate: bounding work outranks answering at all.

---

## 11. Security Considerations

The name and label limits of Section 3.2 and the pointer constraints of Section 3.5
are load-bearing. Note that the pointer direction and jump-count requirements are
**[convention]** rather than RFC text - they are among the few places where correct
implementation requires going beyond what RFC 1035 states.

The protocol as specified here carries no authentication. Over UDP both source
address and content are trivially forged; the ID field and the source port are the
only entropy available for rejecting off-path forgeries, and both should be selected
unpredictably. RFC 1035, Section 7.3 already notes that a resolver cannot rely on the
response arriving from the address it queried, which removes one plausible check.

Requestor-side specification of a maximum buffer size can open a denial-of-service
avenue if responders can be induced to send messages too large for intermediate
gateways, leading to ICMP storms; and announcing very large buffer sizes can cause
middleboxes to drop messages, producing retransmissions with no hope of success
[RFC6891, Section 8]. A server offering recursive service to arbitrary requestors may be used to
amplify reflection attacks, responses being typically far larger than the queries
eliciting them.

The priority ordering of Section 10.4 is a security property as much as a robustness
one: RFC 1034 treats misconfiguration elsewhere in the system as an expected
condition, not an exceptional one.

---

## Appendix A. Examples

> **Caution.** The messages in A.1 and A.2 were constructed for this document and are
> internally consistent, but the address shown is not example.com's current address.
> Use them as fixed test vectors for a codec, not as an expectation about live
> traffic. The compression example in Section 3.5 is from RFC 1035, Section 4.1.4 itself.

### A.1 Query

A standard query for the address records of `example.com`, ID 0x1234, recursion
desired. Length 29 octets.

```
0000  12 34 01 00 00 01 00 00  00 00 00 00
000c  07 65 78 61 6d 70 6c 65  03 63 6f 6d 00
0019  00 01 00 01
```

- `12 34` - ID
- `01 00` - QR=0, OPCODE=0, RD=1, all other flags zero
- `00 01` - QDCOUNT=1; remaining counts zero
- offset 0x0c - QNAME: `example`, `com`, root
- `00 01 00 01` - QTYPE=A, QCLASS=IN

### A.2 Response

The corresponding response with one answer record. Length 45 octets.

```
0000  12 34 81 80 00 01 00 01  00 00 00 00
000c  07 65 78 61 6d 70 6c 65  03 63 6f 6d 00
0019  00 01 00 01
001d  c0 0c 00 01 00 01 00 00  0e 10 00 04 5d b8 d8 22
```

- `81 80` - QR=1, RD=1, RA=1, RCODE=0
- `00 01 00 01` - QDCOUNT=1, ANCOUNT=1
- offsets 0x0c-0x1c reproduce the question section unaltered
- `c0 0c` - owner name as a pointer to offset 12, the start of the question
- `00 01 00 01` - TYPE=A, CLASS=IN
- `00 00 0e 10` - TTL 3600
- `00 04` - RDLENGTH
- `5d b8 d8 22` - 93.184.216.34

### A.3 Response Shapes [RFC1034, Section 6.2]

| Answer          | AA    | Authority    | Interpretation                              |
| --------------- | ----- | ------------ | ------------------------------------------- |
| Records present | set   | empty        | Authoritative answer                        |
| Records present | clear | empty        | Answer from cache; TTLs will have aged down |
| Empty           | set   | empty or SOA | Name exists, no data of this type           |
| Empty           | set   | SOA          | Name error, with negative-caching SOA       |
| Empty           | clear | NS records   | Referral; addresses in additional section   |

---

## Appendix B. References

### B.1 Normative

RFC 1034 Domain Names: Concepts and Facilities - _read in full_
RFC 1035 Domain Names: Implementation and Specification - _read in full_
RFC 2119 Key Words for Use in RFCs to Indicate Requirement Levels
RFC 2181 Clarifications to the DNS Specification - _Section 5.2, Section 7.3, Section 8 verified_
RFC 2308 Negative Caching of DNS Queries - _Section 4 via secondary sources_
RFC 2782 A DNS RR for Specifying the Location of Services (SRV) - _not read_
RFC 3596 DNS Extensions to Support IP Version 6 - _not read_
RFC 3597 Handling of Unknown DNS Resource Record Types - _read in full_
RFC 4035 Protocol Modifications for the DNS Security Extensions - _not read_
RFC 6891 Extension Mechanisms for DNS (EDNS(0)) - _read in full_
RFC 8767 Serving Stale Data to Improve DNS Resiliency - _Section 4 verified_

### B.2 Informative

RFC 1123 Requirements for Internet Hosts (compression, owner names)
RFC 1996 Prompt Notification of Zone Changes
RFC 2136 Dynamic Updates in the Domain Name System
RFC 3225 Indicating Resolver Support of DNSSEC
RFC 3425 Obsoleting IQUERY
RFC 6895 DNS IANA Considerations
RFC 9499 DNS Terminology
IANA Domain Name System (DNS) Parameters
