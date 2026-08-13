# DNS server

Just another DNS server implementation(s) made for fun but tried to stick to the specification.

## Specification

[SPECIFICATION.md](SPECIFICATION.md) is a consolidated DNS specification assembled
for this project, drafted with AI assistance and validated manually against the
source RFCs: the message-format and behavioural provisions of RFC 1034 and RFC 1035
merged with the amendments in RFC 2181, 2308, 2782, 3596, 3597, 4035, 4592, 5452,
6891, 6895, 7766, 8482 and 8767, so the rules that actually govern an implementation
sit in one place instead of across sixteen documents.

Every provision carries its source, e.g. `[RFC1035, Section 4.1.1]`. Where documents
conflict, the later one governs and the conflict is called out rather than quietly
resolved. TTL signedness, the SOA MINIMUM redefinition, negative-response placement,
TC-bit rules and the AD/CD header bits all have amendment notes explaining what
changed. Requirements that are common implementation practice rather than RFC text
are marked **[convention]** so they are not mistaken for normative ones.

Every citation has been checked against the source RFC text. Master file syntax,
inverse queries, zone transfer mechanics and DNSSEC validation are out of scope;
consult the RFCs directly for those.

The specification is the only part of this repository produced with AI. The
implementations are written by hand.

## Implementations

Each implementation is independent and lives in its own directory

| Directory  | Language | Owner | Status |
| ---------- | -------- | ----- | ------ |
| `impl/TBD` | TBD      | TBD   | TBD    |
| `impl/TBD` | TBD      | TBD   | TBD    |

### Roadmap

1. Codec: header, names, question and records, round-tripped against the vectors
2. Compression pointer decoding, including pointers into the interior of a name
3. UDP server: listen, answer from a static zone, truncation and the TC bit
4. Forwarding upstream, with EDNS(0) OPT
5. TCP transport and compression pointer encoding

Steps 1 and 2 are the codec and are worth getting exactly right before anything is
built on top.

Step 2 is not optional: real resolvers send compressed names, so a
server that cannot read them cannot forward.

Step 5 is last but not optional either. RFC 7766 requires forwarders to support TCP,
and DNSSEC-signed responses routinely exceed what fits in a UDP datagram. It is also
cheap once the codec exists: a two-octet length prefix in front of the same parser.
