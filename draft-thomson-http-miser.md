---
title: Merkle-Integrity Signature Extension for Responses
abbrev: MISER
docname: draft-thomson-http-miser-latest
date: 2016
category: info

ipr: trust200902
area: ART
workgroup: httpbis
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: M. Thomson
    name: Martin Thomson
    organization: Mozilla
    email: martin.thomson@gmail.com

normative:
  RFC2119:
  RFC4648:
  RFC7230:
  RFC7231:
  X9.62:
     title: "Public Key Cryptography For The Financial Services Industry: The Elliptic Curve Digital Signature Algorithm (ECDSA)"
     author:
       - org: ANSI
     date: 1998
     seriesinfo: ANSI X9.62
  FIPS186:
    title: "Digital Signature Standard (DSS)"
    author:
      - org: National Institute of Standards and Technology (NIST)
    date: July 2013
    seriesinfo: NIST PUB 186-4
  FIPS180-2:
    title: NIST FIPS 180-2, Secure Hash Standard
    author:
      name: NIST
      ins: National Institute of Standards and Technology, U.S. Department of Commerce
    date: 2002-08
  I-D.thomson-http-mice:
  I-D.thomson-http-encryption:

informative:
  I-D.ietf-httpbis-encryption-encoding:
  I-D.reschke-http-oob-encoding:
  SRI:
    title: "Subresource Integrity"
    author:
      - ins: D. Akhawe
      - ins: F. Braun
      - ins: F. Marier
      - ins: J. Weinberger
    date: 2015-11-13
    seriesinfo: W3C CR
    target: https://w3c.github.io/webappsec-subresource-integrity/
  DAVIS:
    title: "Defective Sign & Encrypt in S/MIME, PKCS#7, MOSS, PEM, PGP, and XML"
    author:
      - ins: D. Davis
    date: 2001-05-05
    target: http://world.std.com/~dtd/sign_encrypt/sign_encrypt7.html


--- abstract

Use of a signature is described to provide integrity for the content of HTTP
responses.  Content can be progressively processed without losing integrity if a
client is prepared to accept the risk of truncation.


--- middle

# Introduction {#problems}

A scheme for providing integrity for the content of HTTP messages is described
in [I-D.thomson-http-mice].  That scheme permits a recipient to process the
content of a message without having to receive the entire message.  Though the
content might subsequently be truncated, the partial content that is received is
known to be correct.  This is similar to the integrity guarantee provided by
HTTPS [RFC2818].

The major drawback of any integrity scheme that it is not possible to prime a
recipient with any information that would allow them to make a decision
regarding content that isn't yet known.  Thus, for example, it is impossible to
construct an integrity protected reference such as the one described in [SRI]
without first knowing the precise details of the content that is being referred
to.

A signature-based integrity scheme avoids this problem by allowing a recipient
to be given a signature public key that can be used to verify the integrity of
message content.  This allows a reference to be constructed prior to generating
dynamic content.  More generally, it makes it possible to provide an
integrity-protected reference to content without having to know the specific
content of what is being referred to.


## Signature Limitations

Signature-based schemes do have several downsides that need to be carefully
considered.

* Signatures can be constructed in a way that permits unwanted portability of
  content.  If the same key is used to sign the content of multiple messages,
  the content of one message could be switched for the content of another
  message.

* Signatures are also not bound to a particular key; given a message signed with
  one key, it is possible that another key could be found that appears to have
  produced the signature.  This might be used to falsely claim the contents of a
  signed message.

* Signatures - when combined with encryption - create deceptive properties.  A
  message that is both signed by a particular sender and encrypted toward a
  particular recipient does not imply that the message was originally
  constructed by that sender to be sent to that recipient.  If separately
  applied, the outermost signature or encryption can be replaced [DAVIS].

This document defines a signature scheme with a narrow scope, the intent being
to provide the ability to use signatures for integrity protection without
running afoul of the many pitfalls of previous attempts at signing HTTP messages
or content.

In particular, this document only describes how the content of an HTTP response
is signed.

ISSUE: Should this further limit the signature to responses to GET requests?  In
looking at the various ways in which these signatures can be transplanted,
limiting in this way would avoid problems with different requests.  Since most
requests can be trivially parlayed into a GET request, this doesn't seem like an
unbearable restriction on use.

ISSUE: SHOULD this also prohibit content negotiation?  That the signatures on
XML and JSON variants of responses to the same resource are interchangeable
doesn't seem obviously exploitable, but not all changes in response to content
negotiation are so clearly delineated.


## Terminology          {#Terminology}

RFC 2119 [RFC2119] defines the terms MUST, SHOULD, and MAY.


## Example

The following shows an HTTP/1.1 response to a request to
`https://example.com/hello`.

~~~
HTTP/1.1 200 OK
Date: Wed, 29 Jun 2016 01:16:02 GMT
Content-Length: 15
Crypto-Key: keyid=a;
    p256ecdsa=BLqanccr81i6vKOl3cUpc5GyxeGP4A05DtsxNLMGtzqV
              KzIUmJz3rQNUgMZc8gv7fk0xQJ80BVnrjW4VifawAZ8
MI: keyid=a;
    p256ecdsa=iPn7MWjD7nCcnq6OJn2nTYbKyYhj3IB1ngmsiatFiARh
              zqVsuaH7KsB2et2PCX33SDX6ZlEges_wDiPkIp5zqA

Hello, World!
~~~

Line wrapping is added to this example to fit formatting constraints.  The
omitted `p` attribute for this example is included in [I-D.thomson-http-mice].


# Signature Usage {#sig}

A signature uses the MI header field defined in [I-D.thomson-http-mice].  A new
`p256ecdsa` parameter is defined that carries an ECDSA [X9.62] signature over the P-256
curve [FIPS186] using the SHA-2 hash [FIPS180-2].  The signature is encoded
in the parameter using base64url encoding [RFC7515].

The input to the signature is the concatenation of:

* the UTF-8 [RFC3629] encoded string "MI: p256ecdsa",
* a single zero-valued octet,
* the octets of the effective request URI (Section 5.5 of [RFC7230]),
* a single zero-valued octet, and
* the "mi-sha256" root integrity proof for the content [I-D.thomson-http-mice].

The `p256ecdsa` field contains the concatenated bitstrings of the R and S
outputs of the ECDSA signature algorithm.

If a recipient is expected to use the signature to validate content, the
`p256ecdsa` parameter can replace the `p` parameter of the MI header field.  Both
parameters MAY be included, though both should agree.

Signatures MUST NOT be used to protect the content of a request.  Clients
frequently make multiple requests of the same resource with different header
fields and content.  Relying on signatures for requests could allow attacks
where content from one request is interchanged with other requests to the same
resource.


## URI Normalization

The consequence of including the effective request URI under the signature is
that the signature is exposed to the vagaries of URI encoding.  Though two URIs
might be considered equivalent, differences in the encoding of those URIs will
produce different signatures.

A potential approach to this problem has the server signal the precise encoding
that it used to produce the signature.  However, this changes the problem from a
normalization problem to a comparison problem.  A client that fails to correctly
compare their effective request URI with the claimed URI could be exposed to
attacks where the content of resources with similar URIs are interchanged.  A
client cannot know that a server will consider two different URI encodings to be
identical.

Thus, a normalization scheme specific to `https://` URIs is defined, based on
the recommendations in [RFC7230] and [RFC3986].  Use of signatures with other
URI schemes is not defined.  The following transformations are made to the
effective request URI, with reference to the corresponding part of Section 6.2
of [RFC3986]:

* The `https://` scheme is rendered in lower case (S6.2.2.1)

* A domain name is encoded as lower case A-Labels (S6.2.2.1)

* An IPv4 literal is represented in dotted decimal notation without additional
  leading zeroes

* An IPv6 literal is represented according to the canonical form defined in
  [RFC5952]

* A port number is included without leading zeros; the colon and port are
  omitted entirely if the port number is 443 (S6.2.3)

* Percent-encoded of characters in the host component are decoded; non-reserved
  characters in the path and query components are decoded (S6.2.2.2)

* Path segments containing "." and ".." are normalized (S6.2.2.3)

* An empty path segment is replaced with a "/" (S6.2.3)

Note that an effective request URI does not contain a fragment.

Unfortunately, devising a perfect normalization scheme for all `https://` URIs
isn't possible, since servers could apply additional rules for determining URI
equivalance that are context-specific.

Clients that encounter signature failures MAY attempt to re-acquire resources by
passing the normalized effective request URI in a request that uses the
absolute-form (see Section 5.3.2 of [RFC7230]).

Servers that receive a request in the absolute-form SHOULD use the octets
provided by the client as input as input to the above normalization procedure
when producing a signature.  In other words, servers that permit a range of
possible URI encodings to identify the same logical resource might need to
produce multiple signatures for that resource depending on how a client requests
that resource.


# Identifying and Describing Signature Keys {#keys}

A `keyid` parameter is added for use with the MI header field.  This allows a
recipient to identify the key that can be used to validate the signature in the
`p256ecdsa` parameter.  The `keyid` SHOULD match a value in the Crypto-Key
header field [I-D.ietf-httpbis-encryption-encoding] if that header field is is
used.

Providing a signature key is typically only useful where the provision of the
key can be attributed a higher level of trust than the signature itself.  A
message sent using out-of-band content-encoding [I-D.reschke-http-oob-encoding]
is one situation that might benefit from the use of this header field.

This document defines a new parameter for use with the Crypto-Key header field.
The `p256ecdsa` parameter conveys an uncompressed P-256 public key [X.692].  The
`p256ecdsa` parameter for the Crypto-Key header field is encoded using base64url
encoding [RFC7515].


# Security Considerations {#security}

Determining whether a signature is valid is only a small part of authenticating
a response.  This document doesn't describe a complete solution for identifying
which signing keys are accepted.

The signature does not authenticate header fields or any part of the request.
The signature only covers the effective request URI, which is to prevent signed
content from being replayed for responses to other URIs.  A recipient of a
signed payload needs to be especially careful that decisions that rely on
authenticating the payload do not take any unauthenticated material as input.
In particular, the `Content-Type` header field are not authenticated by this
scheme.

No replay protection is offered for signatures.  This means that valid messages
can be captured and replayed.  The only restriction is that they are limited to
responses for the same URI.  The following changes are not detectable based on
the signature:

* The signature does not include details of the request, therefore contents of
  responses that depend on requests are interchangeable.

* The signature does not include header fields in the response, therefore any
  arrangement of header fields might be attached to signed content.

* The signature is not bound to a particular time, therefore responses can be
  replayed at other times.

In particular, if content negotiation (see Section 3.4 of [RFC7231]) is used,
then all alternative representations for a given URI will all have valid
signatures.  For this reason, servers SHOULD avoid combining content negotiation
with signatures.

A recipient of encrypted [I-D.ietf-httpbis-encryption-encoding] content that is
also signed, cannot assume that the encrypted content was originally generated
by the holder of the signing key.  A signature might be applied to encrypted
content without knowing its contents.  For a similar reason, an encrypted
message might not have been originally sent to the holder of the decryption
keys.  More generally, a recipient of encrypted HTTP response content cannot
assume that the message was encrypted specifically for them.

A key used to sign response content MUST NOT be used for other purposes.  A key
SHOULD also be used over a limited scope and time.  A key might be limited to
use with a single request.


# IANA Considerations {#iana}

## Additions to the MI Parameter Registry

This memo registers the `p256ecdsa` and `keyid` in the "Hypertext Transfer
Protocol (HTTP) MI Parameters" registry established in [I-D.thomson-http-mice].

For `keyid`:

* Parameter Name: keyid
* Purpose: Identify the key that is in use.
* Reference: {{keys}} of this document

For `p256ecdsa`:

* Parameter Name: p256ecdsa
* Purpose: Conveys a signature over the `p` parameter using P-256, ECDSA and SHA-256.
* Reference: {{sig}} of this document


## Additions to the Crypto-Key Parameter Registry

This memo registers the `p256ecdsa` and `keyid` in the "Hypertext Transfer
Protocol (HTTP) Crypto-Key Parameters" registry established in
[I-D.ietf-httpbis-encryption-encoding].

* Parameter Name: p256ecdsa
* Purpose: Conveys a signing key for use with the parameter of the same name on
  the `Content-Signature` header field.
* Reference: {{keys}} of this document


--- back
