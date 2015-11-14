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
  RFC6454:
  RFC7230:
  RFC7232:
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
In one form, a server MAY use a portion of a hash of the URL concatenated with the value of ETag response header as the key.
In another form, a server MAY pre-assign a small number to every resource it might push.

Additional considerations in making effective use of the header is described in {{considerations}}.

# The CACHE_FINGERPRINT HTTP/2 Frame {#cache-fingerprint}

The CACHE_FINGERPRINT HTTP/2 frame ([RFC7540], Section 4) indicates which responses (with "Cache-Fingerprint-Key" header fields) from same origin [RFC6454] are cached by the client.

The CACHE_FINGERPRINT frame is a non-critical extension to HTTP/2.
Endpoints that do not support this frame can safely ignore it.

It MUST occur on stream 0; a CACHE_FINGERPRINT frame on any other stream is invalid and MUST be ignored.
The frame does not define any flags.

When connecting to an HTTP/2 server, a user agent SHOULD send the frame before sending the first HEADERS frame for each origin the client tries to access, so that the peer can determine if the client implements this specification, and in case it does, the cache state of the client, prior to recieiving the first request to each origin.

A user agent MAY also send the frame at any later time, when it needs to notify the server it has received a cached response that originated from one of the previously specified origin using other methods than the current connection (e.g. when a cached response has been shared from a neigboring cache server).

The CACHE_FINGERPRINT frame type is 0xc (decimal 12).

~~~
+-------------------------------+-------------------------------+
|        Origin-Len (16)        |          Origin (*)         ...
+-------------------------------+-------------------------------+
|        Fingerprint (*)      ...
+-------------------------------+
~~~

The CACHE_FINGERPRINT frame MUST contain the following fields:

Origin-Len: An unsigned, 16-bit integer indicating the length, in octets, of the Origin field.

Origin: A sequence of characters containing the ASCII serialization of an origin ([RFC6454], Section 6.2) to which the sent fingerprint is associated.

Fingerprint: A sequence of octets containing a list of fingerprint keys found in the cached responses sent by specified origin.
The contents of the field is specified in the following section.

## Calculating the Fingerprint Field

The value of the Fingerprint field MUST match the output generated by the following steps:

1. collect the values of "Cache-Fingerprint-Key" header fields from all the cached responses of the same origin
2. if number of collected keys is zero (0), go to step 10
3. algebraically sort the collected keys
4. determine the parameter of Golomb-Rice coding to be used [Golomb].[Rice].  The value MUST be a power of two (2), between one (1) to 2147483648.
5. calculate log2 of the parameter determined in step 4 as a 5-bit value
6. encode the first key using Golomb-Rice coding with parameter determined in step 4
7. if number of collected keys is one (1), go to step 9
8. for every collected key expect for the first key, encode the delta from the previous key minus one (1) using Golom-Rice coding with parameter determined in step 4
9. concatenate the result of step 4, 6, 8
10. if number of bits contained in the result of step 9 is not a multiple of eight (8), append a bit set until the length becomes a multiple of eight (8)

As an example, when none of the cached responses from the same origin contained a "Cache-Fingerprint-Key" header, then the output will be a zero-length octet stream.

Or if two cached responses contained the header with values 115 and 923, the output will be following four (4) octets if 256 was selected as the parameter for Golom-Rice coding.

~~~
41 cf 89 ff
~~~

User agents MAY run the steps more than once with different values selected as the parameter used for Golomb-Rice coding, and send the shortest output as the value of the header.
Or it MAY use the result of the following equation rounded to the nearest power of two (2).  It can be shown that the parameter chosen using this equation will yield the shortest output when the keys are distributed geometrically.

~~~
  log2(maximum_value_of_collected_keys / number_of_collected_keys)
~~~

An implementation of the steps above can be found at https://github.com/kazuho/golombset/.

# Considerations

## Size of the CACHE_FINGERPRINT frame {#cache-fingerprint-size}

Length of the Fingerprint field of the CACHE_FINGERPRINT frame is proportional to the number of keys contained, and to log2 of the average distance between the keys.

Therefore, to avoid the frame field from becoming too long, a server SHOULD send "Cache-Fingerprint-Key" header only with a response containing a resource the server needs to track the cache state of.

And in case of using a hash function for deriving the value of "Cache-Fingerprint-Key" header, the hash value SHOULD be truncated to a small range of consecutive integers starting from zero (0) to a maximum calculated as the number of resources need to be tracked divided by the false positive probability.

For example, to track the cache state of 100 resources with probability of 1% false positive using a hash function, the key should be a remainder of the hashed value divided by 10000.
Using the existing implementation, it is estimated that the length of the frame field will be around 102 bytes when all the 100 resources are cached.

## Security Considerations

A server MAY ignore a CACHE_FINGERPRINT frame that contains unnecessarily huge amount of keys compared to the number of elements that the server is willing to track, to prevent attackers mounting denial-of-service attacks against the server.

## Privacy Considerations

Value of CACHE_FINGERPRINT frames can be used by a third party monitoring the network communication for tracking a user.
Therefore a CACHE_FINGERPRINT frame SHOULD NOT be sent over a non-encrypted HTTP/2 connection.

The Values can also be used by the origin server to track users; however since there are other existing methods that can be used for the same purpose (e.g. issue different ETag for every user), introduction of the new HTTP/2 frame is not considered that it would introduce new issues regarding privacy.
