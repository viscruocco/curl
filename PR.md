# PR

## Summary

1. Fix a (presumed) regression introduced in #16512 / fa3d1e7.
2. Extend the handling of fragmented WebSocket messages to fix the issues
   described in #15298.

The work in #16512 / fa3d1e7 set out to address the handling of the CURLWS_CONT
flag, but did not modify the handling of the accompanying CURLWS_TEXT and 
CURLWS_BINARY flag. This would probably be sufficient to allow an application to
correctly implement fragmented message handling. With regard to fragmented
messages it also seems compatible with the existing interface documentation,
although the  documentation appears underspecified and allows for varying
interpretations whether the CURLWS_TEXT/CURLWS_BINARY should be present in
following fragments or not (as described in #15298).

Unfortunately #16512 / fa3d1e7 also modified the CURLWS_CONT flag handling for
partially processed frames occuring for non-blocking interfaces like
`curl_ws_send()`/`curl_ws_recv()`. The new behavior seems unhelpful, may break
existing applications and is in violation of the documentation.

## Terminology

Because the discussion in #16503 shows that the terminology around fragmented
WebSocket messages is not very clear. Hence, let me fist pin down the most
important terms before carrying on:

* Frame:
  A "packet" of data encoded with the header and payload structure as described in
  [RFC 6455 Section 5 "Data Framing"](https://datatracker.ietf.org/doc/html/rfc6455#section-5).

* Message:
  An abstract unit of information. From [RFC 6455 Section 1.2 "Protocol Overview"](https://datatracker.ietf.org/doc/html/rfc6455#section-1.2):
  > [...] After a successful handshake, clients and servers transfer data back
    and forth in conceptual units referred to in this specification as
    "messages". On the wire, a message is composed of one or more frames. [...]
    Each frame belonging to the same message contains the same type of data.

  According to [RFC 6455 Section 5.4 "Fragmentation"](https://datatracker.ietf.org/doc/html/rfc6455#section-5.4)
  a message delivered in a single frame is called an "unfragmented message".
  In contrast, a message split over multiple frames is called a "fragmented message".

* Fragment:
  According to [RFC 6455 Section 5.4 "Fragmentation"](https://datatracker.ietf.org/doc/html/rfc6455#section-5.4)
  this refers to the part of a "fragmented message" which is delivered within a single frame.
  (I try to avoid using the term "fragment" to refer to the frame containing a "message fragment",
  but occasionally I might accidentally do so.)

* Chunk:
  When `curl_ws_send()`/`curl_ws_recv()` are used in conjunction with the `CURLWS_OFFSET`
  flag or are not able to process enough data in a single call, then they will
  only consume/produce a piece of the data and indicate this via their
  `sent`/`meta->bytesleft` output parameter. The libcurl documentation does not introduce an
  official term for this slice of partial data, but I will refer to it as "chunk"
  to avoid confusion with the term "fragment".

  The difference between fragments and "chunks" seemed to be a clear
  source of confusion in the discussion about #16503.

* CONT:
  There is a notable difference between the usage of the term CONT in the
  WebSocket RFC and libcurl.
  RFC 6455 uses CONT with a semantic of "this is the continuation of a prior fragment".
  Curl instead uses CONT with a semantic of "this will be continued in a following fragment".
  Unfortunately it's too late to amend this naming since WebSocket support is not
  experimental anymore, so we'll have to live with it.
  
  Hence, the RFC marks the frames of a fragmented message in the following way.
  1. In the first frame the OPCODE is set according to the message type
     (WSBIT_OPCODE_TEXT or WSBIT_OPCODE_BIN) and the WSBIT_FIN bit is cleared.
  2. In intermediate frames (neither first nor last fragment; might not be present)
     the OPCODE is set to WSBIT_OPCODE_CONT and the WSBIT_FIN bit is cleared.
  3. In the final frame the OPCODE is set to WSBIT_OPCODE_CONT and the
     WSBIT_FIN bit is set.

  While libcurl marks the frames of a fragmented message in the following way.
  1. In the first frame the CURLWS_CONT flag is set in conjunction with the
     CURLWS_TEXT/CURLWS_BINARY flag.
  2. In intermediate frames the CURLWS_CONT flag is set and the documentation
     does not specify whether the CURLWS_TEXT/CURLWS_BINARY flag is set or cleared.
  3. In the final frame the CURLWS_CONT flag is cleared and the documentation
     does not specify whether the CURLWS_TEXT/CURLWS_BINARY flag is set or cleared.

## Problem statement

The yet unspecified portion whether the CURLWS_TEXT/CURLWS_BINARY flag is set or cleared
in the intermediate frames and final frame is one of two motivators behind this PR.

The other - more important - motivator is to the adjust the way the CURLWS_CONT
flag is handled for chunks.

Before #16512 / fa3d1e7 the CURLWS_CONT flag would _never_ show up for
unfragmented messages, because it was solely based on the frames opcode.
This was changed (and arguably broken) in #16512 / fa3d1e7.

The discussion in #16503, which was probably influenced by a confusion of fragments
and chunks, prompted that in #16512 / fa3d1e7 the CURLWS_CONT flag is now set
on all but the final chunks of a frame. Importantly this includes (but is
not limited to) the case where an unfragmented message is delivered in a single
frame but received as several chunks. Contrary to the old behavior this will
now set the CURLWS_CONT flag.

For reference: the currently documented method to check whether a non-blocking
call was able to process a frame as a single chunk is to evaluate the `sent`
output parameter of `curl_ws_send()` respectively the `meta->bytesleft`
output parameter of `curl_ws_recv()`. In turn the current documentation does not
mention any other use for the CURLWS_CONT flag than the fragmented message
signalling I have described previously.

The change may break existing application logic which interpret the occurence
of CURLWS_CONT as a sign that more frames with subsequent message fragments will
follow. They could detect the discrepancy by explicitly paying attention the 
presence of the CURLWS_CONT flag in the last chunk of a frame (identified
through the value of `sent`/`meta->bytesleft`) but it is very unlikely that any
existing implementation would bother to verify this. Instead, applications
may just read the flags on the first chunk and assume they will either remain
the same or be completely unset for all subsequent chunks, since the
documentation did not specify anything else.

Besides from potentially breaking current applications and violating the
documentation, the newly introduced behavior does not offer an additional
benefit over the existing method to check the `sent`/`meta->bytesleft` value.
On the contrary: checking `sent`/`meta->bytesleft` would be absolutely mandatory
to identify the last chunk of the frame, which would then carry the CURLWS_CONT
flag describing the whole frame. In this last chunk the CURLWS_CONT could either
be set flag to indicate that more fragments will follow in the next frames or be
cleared to indicate that the message is complete.

## Proposed solution

This PR attempts to implement and document the behavior as follows:

For consecutive frames of a fragmented message, the behavior shall be as
already perfectly summarized in #15298:
1. First frame: Both CURLWS_TEXT/BINARY and CURLWS_CONT set.
2. Intermediate frames: Both CURLWS_TEXT/BINARY and CURLWS_CONT set.
3. Final frame: Only CURLWS_TEXT/BINARY set.

For consecutive data chunks the flags shall be guaranteed to be identical.
Hence, the CURLWS_CONT flag shall either be present in all chunks of a frame
(for first/intermediate frame of a fragmented message) or none of the chunks of
a frame (for final frame or unfragmented message).