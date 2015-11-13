---
title: Cache fingerprinting for HTTP
abbrev: HTTP cache fingerprinting
docname: draft-kazuho-http-cache-fingerprint-latest
date: 2015-11-12
category: info

ipr: trust200902
area: General
workgroup:
keyword: Internet-Draft

stand_alone: yes
pi: [toc, docindent, sortrefs, symrefs, strict, compact, comments, inline]

author:
 -
    ins: K. Oku
    name: Kazuho Oku
    organization: DeNA Co., Ltd.
    email: kazuhooku@gmail.com

normative:
  RFC2119:
  RFC4648:
  RFC7230:
  RFC7540:
  Golomb:
    title: Run-length codings
    author:
      ins: S. Golomb
      name: Solomon Wolf Golomb
    date: 1966-07
    seriesinfo: IEEE Transactions on Information Theory 12(3)
  Rice:
    title: Some Practical Universal Noiseless Coding Techniques
    author:
      ins: R. Rice
      name: Robert F. Rice
      organization: Jet Propulsion Laboratory
    date: 1979-03
    seriesinfo: JPL Publication 79-22

informative:

--- abstract

This memo introduces HTTP headers for cache fingerprinting.

The fingerprint can be used by the server to determine the responses it should pre-emptively send (or "push") to client.

--- middle

# Introduction

Server push introduced in HTTP/2 [RFC7540] allows a server to speculatively send data to a client that the server anticipates the client will need.
For example, when receiving a request for an HTML document over a high-latency network, a server might want to send to the client resources that are required for rendering the HTML document (e.g. style-sheets and script files) in addition to the HTML itself in order to reduce the number of round-trips necessary for the client gathering all necessary responses.

But in case the client is already in posession of such additional resources, there is no reason to push them to the client; doing so is just waste of bandwidth and time.

Therefore, it is desirable to define a method for endpoints to communicate the cache state of a client, so that a server can determine what it should push with knowledge of the peer's cache state.  This document specifies a set of HTTP headers that can be used for such purpose.

By using the headers it is possible for a server to make a good guess of what is already cached on a client, and only push the responses that have not yet been cached.

## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC2119].

# The Cache-Fingerprint-Key Header Field {#cache-fingerprint-key}

The "Cache-Fingerprint-Key" HTTP response header field describes the fingerprint key of the content as a decimal number between zero (0) and 4294967295.

~~~
  Cache-Fingerprint-Key = 1*DIGIT
~~~

An example is

~~~
  Cache-Fingerprint-Key: 12345
~~~

A server MAY send the header field in a cacheable response.

This document does not specify how the key should be derived.
In one form, a server MAY use a portion of a hash of the URL as the key.
In another form, a server MAY pre-assign a small number to every resource it might push.

# The Cache-Fingerprint Header Field {#cache-fingerprint}

A user agent sends an aggregation of fingerprint keys found in cached responses sent from the origin server using the "Cache-Fingerprint" header.

~~~
  Cache-Fingerprint: *( ALPHA / DIGIT / "-" / "_") *"="
~~~

A user agent SHOULD send the header even if no response with fingerprint keys are being cached, as an indication to the server that it is capable of collecting and sending a cache fingerprint.

When a user agent sends a "Cache-Fingerprint" header field, the value MUST be computed using the following steps.

1. collect the values of "Cache-Fingerprint-Key" header fields in the cached HTTP responses sent from the origin server to which the header field is going to be sent
2. if number of collected keys is zero (0), go to step 9
3. algebraically sort the collected keys
4. determine the parameter of Golomb-Rice coding to be used [Golomb].[Rice].  The value MUST be a power of two (2), between one (1) to 2147483648.
5. calculate log2 of the parameter determined in step 4 as a 5-bit value
6. encode the first key using Golomb-Rice coding with parameter determined in step 4
7. if number of collected keys is one (1), go to step 9
8. for every collected key expect for the first key, encode the delta from the previous key minus one (1) using Golom-Rice coding with parameter determined in step 4
9. concatenate the result of step 4, 6, 8 and encode the result using base64url [RFC4648].  Padding of base64url MAY be omitted.

As an example, when none of the cached responses from the origin server contained a "Cache-Fingerprint-Key" header, then the  "Cache-Fingerprint" header field will be:

~~~
  Cache-Fingerprint:
~~~

Or if two cached responses contained the header with values 115 and 923, the header field will be as follows, if 512 was selected as the parameter for Golom-Rice coding and if padding of base64url was omitted.

~~~
  Cache-Fingerprint: Qc+J/w
~~~

User agents MAY run the steps more than once with different values selected as the parameter used for Golomb-Rice coding, and send the shortest output as the value of the header.
Or it MAY use the result of the following equation rounded to the nearest power of two (2).  It can be shown that the parameter chosen using this equation will yield the shortest output when the keys are distributed geometrically.

~~~
  log2(maximum_value_of_collected_keys / number_of_collected_keys)
~~~

An implementation of the steps above can be found at https://github.com/kazuho/golombset/.
