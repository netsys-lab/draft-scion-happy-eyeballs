---
title: "SCION in Happy Eyeballs Version 3"
abbrev: "SCION Happy Eyeballs"
category: exp
docname: draft-john-scion-happy-eyeballs-latest
submissiontype: independent
number:
date:
v: 3
keyword:
  - Happy Eyeballs
  - SCION
  - QUIC
  - path-aware networking
venue:
  github: "netsys-lab/draft-scion-happy-eyeballs"
author:
  - fullname: Tony John
    initials: T.
    surname: John
    organization: OVGU Magdeburg
    email: tony.john@ovgu.de
  - fullname: Jelte van Bommel
    initials: J.
    surname: van Bommel
    organization: ETH Zurich
    email: jelte.vanbommel@inf.ethz.ch
  - fullname: Tilmann Zäschke
    asciiFullname: Tilmann Zaeschke
    initials: T.
    surname: Zäschke
    asciiSurname: Zaeschke
    organization: ETH Zurich
    email: tilmann.zaeschke@inf.ethz.ch
  - fullname: Lars-Christian Schulz
    initials: L.
    surname: Schulz
    organization: OVGU Magdeburg
    email: lschulz@ovgu.de
  - fullname: Nicola Rustignoli
    initials: N.
    surname: Rustignoli
    organization: SCION Association
    email: nic@scion.org

normative:
  RFC6724:
  RFC9000:
  RFC9001:
  RFC9460:
  RFC9846:
  I-D.ietf-happy-happyeyeballs-v3:
  I-D.dekater-scion-dataplane:
  I-D.dekater-scion-controlplane:
  I-D.john-scion-svcb:

informative:
  RFC7942:
  RFC9217:
  RFC9849:
  RFC10069:
  SCION-HEV3:
    title: "scion-hev3: Go client library for SCION and IP endpoints, commit d461073"
    target: https://github.com/netsys-lab/scion-hev3/tree/d46107329e06afba4fcd76a99b1cfef3810c8c73
    author:
      - org: netsys-lab
    date: 2026-11
  SCIERA:
    title: "SCION Education, Research and Academic Infrastructure (SCIERA)"
    target: https://sciera.readthedocs.io/en/latest/
    author:
      - org: SCIERA
    date: 2026
  SCION-V0.15.1:
    title: "SCION reference implementation, release v0.15.1"
    target: https://github.com/scionproto/scion/releases/tag/v0.15.1
    author:
      - org: SCION Association
    date: 2026-08

--- abstract

Happy Eyeballs Version 3 uses the choice of addresses that a service offers: it races connection attempts to IPv6 and IPv4 addresses. SCION (Scalability, Control, and Isolation On Next-generation networks) is a path-aware inter-domain network architecture in which a client also has a choice of paths to an address. This document specifies how a Happy Eyeballs client uses SCION addresses published in DNS Service Binding (SVCB) and HTTPS records. The client treats SCION as a third address family. It selects a small number of paths that differ from each other and attempts a SCION address on the first. Each time a short delay passes and no attempt has succeeded, it starts an attempt on the next path, while the earlier attempts continue. The number of paths (2 by default) and the delay (50 milliseconds by default) are configurable. Each attempt is a separate connection, and the client uses the first attempt that succeeds. The attempts on further paths do not delay attempts to other addresses.

--- middle

# Introduction

SCION {{I-D.dekater-scion-controlplane}} {{I-D.dekater-scion-dataplane}} is a path-aware inter-domain network architecture {{RFC9217}}. A source endpoint looks up paths to the destination Autonomous System (AS) and puts the chosen path into each packet. {{I-D.john-scion-svcb}} defines the "scion" SvcParamKey, which publishes the SCION addresses of a service endpoint in SVCB and HTTPS records {{RFC9460}}. Happy Eyeballs Version 3 (HEv3) {{I-D.ietf-happy-happyeyeballs-v3}} does not consider SCION addresses.

HEv3 uses the choice of addresses that a service offers. SCION adds a second choice, the choice of paths. A client usually knows several paths to one SCION address, and these paths cross different ASes and links. One path can work when another does not.

This document lets an HEv3 client use both choices. A SCION address is one more endpoint in the HEv3 list of endpoints. When its turn comes, the client selects a small number of paths that differ from each other, at most the Path Attempt Count, and starts an attempt on the first. Each time the Path Attempt Delay passes and no attempt has succeeded, the client starts an attempt on the next of these paths, while the earlier attempts continue. Each attempt is a separate connection, and the client uses the first attempt that succeeds. When the first path works, the client normally makes only one handshake. The attempts on further paths do not delay attempts to other endpoints.

This document covers connection establishment only ({{after-establishment}}). The rules of {{I-D.ietf-happy-happyeyeballs-v3}} apply unless this document says otherwise. This document does not update HEv3. A client without SCION connectivity ignores the "scion" key.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the terms of {{I-D.ietf-happy-happyeyeballs-v3}} and the term "SCION address" of {{I-D.john-scion-svcb}}. In this document, "AS" means a SCION AS, and "path" means a SCION path, as in {{I-D.dekater-scion-controlplane}}. This document also uses the following terms:

ISD-AS:
: The pair of the Isolation Domain (ISD) number and the AS number that together identify an AS.

SCION connectivity:
: A client has SCION connectivity if it can look up paths (Section 4 of {{I-D.dekater-scion-controlplane}}) and send and receive SCION packets on a chosen path.

SCION endpoint:
: An endpoint whose address is a SCION address.

Eligible path:
: A path to the AS of a SCION endpoint on which the client can attempt that endpoint ({{path-lookup}}).

Candidate paths:
: The eligible paths that the client selects when it attempts a SCION endpoint ({{path-selection}}). The client makes at most one connection attempt on each.

Path Attempt Count:
: The largest number of candidate paths that the client selects for one SCION endpoint.

Path Attempt Delay:
: The time between the starts of two attempts on candidate paths of one SCION endpoint.

Hop:
: The traversal of one AS by a path. A path is a sequence of hops. Each hop has the ISD-AS of its AS and the Interface IDs (Section 1.1 of {{I-D.dekater-scion-dataplane}}) through which the path enters and leaves that AS. An AS in which two path segments join is one hop (Section 2.4.2 of {{I-D.dekater-scion-dataplane}}).

Inter-AS link:
: A link between two ASes (Section 1.3 of {{I-D.dekater-scion-controlplane}}), including a peering link. It is identified by the ISD-AS and the Interface ID at each of its two ends. A path traverses one inter-AS link between each hop and the next.

Pair of adjacent ASes:
: The ASes of two consecutive hops of a path. More than one inter-AS link can connect the same pair of adjacent ASes.

Transit AS:
: An AS of a hop that is neither the first nor the last hop of a path. All paths to one SCION endpoint have the same first and last AS.

AS hops:
: The number of inter-AS links that a path traverses.

Expiry of a path:
: The earliest expiration time of the Hop Fields of the path (Section 2.2.2.5 of {{I-D.dekater-scion-controlplane}}). A path has expired when this time has passed.

# Hostname Resolution and Path Lookup {#resolution}

## Sending DNS Queries

A client with SCION connectivity SHOULD send an SVCB or HTTPS query (Section 4.1 of {{I-D.ietf-happy-happyeyeballs-v3}}) for any application protocol that has an SVCB mapping, even if the client has no IPv4 or IPv6 connectivity, because SCION addresses are published only in these records. This document adds no other DNS queries.

## SCION Endpoints {#scion-endpoints}

Each distinct usable SCION address (Section 3.3.1 of {{I-D.john-scion-svcb}}) of a ServiceMode record yields one SCION endpoint with the record's SvcPriority and port. Its application protocols are those of the record's SVCB ALPN set (Section 7.1 of {{RFC9460}}) whose transport the client can use over SCION (Section 3.3 of {{I-D.john-scion-svcb}}). If there are none, the address yields no endpoint.

## Path Lookup {#path-lookup}

An IP address, whether from an address record or from a hint, is usable as soon as the client receives it. A SCION address is usable only when the client has a path to its AS. When the client creates a SCION endpoint, it therefore starts a path lookup for the endpoint's ISD-AS. SCION endpoints with the same ISD-AS share one lookup. A lookup can be answered from a cache, for example that of a path service on the client's host (a local path service). The limit on the number of destination ASes for which a client looks up paths (Section 5.4 of {{I-D.john-scion-svcb}}) applies.

A path is eligible if it has not expired and its MTU (Section 3.4 of {{I-D.dekater-scion-controlplane}}) is at least the size of the largest SCION packet that the client sends on it before the handshake completes, without underlay encapsulation. For QUIC, the largest of these packets carries a UDP payload of at least 1200 octets (Section 14.1 of {{RFC9000}}).

A SCION endpoint enters the list of endpoints when its path lookup has returned at least one eligible path. If connection attempts have already started, the endpoint is added as Section 7 of {{I-D.ietf-happy-happyeyeballs-v3}} describes. A SCION endpoint in the client's own AS needs no lookup and enters the list at once. Its only path is the empty path (Section 2.4.1 of {{I-D.dekater-scion-dataplane}}), which has no expiry and has the MTU of the client's AS.

A client gives up on a path lookup that has not completed within 2 seconds. This limit bounds how long a client with no other endpoint waits ({{starting}}). If the lookup fails or returns no eligible path, the client discards the SCION endpoint.

# Sorting Endpoints {#sorting}

SCION endpoints are grouped by application protocol and security requirements (Section 5.1 of {{I-D.ietf-happy-happyeyeballs-v3}}) and by service priority (Section 5.2 of {{I-D.ietf-happy-happyeyeballs-v3}}) in the same way as the IP endpoints of the same record.

Within a group, the SCION endpoints form a third address family. Destination Address Selection {{RFC6724}} (Section 5.3 of {{I-D.ietf-happy-happyeyeballs-v3}}) orders the IP endpoints only. A client orders the SCION endpoints among themselves at random, as Section 3.3 of {{I-D.john-scion-svcb}} recommends, unless local policy sets another order.

By default, a client orders the address families as SCION first, then the family of the first IP endpoint after Destination Address Selection, then the other IP family. Only SCION attempts give the application a choice of path; {{results}} reports what this order costs. A client whose local policy or application prefers IP orders SCION last instead. In both cases, the client interleaves the three families as Section 5.3 of {{I-D.ietf-happy-happyeyeballs-v3}} interleaves two, starting with as many endpoints of the first family as the Preferred Address Family Count.

For example, take SCION endpoints S1 and S2, IPv6 endpoints x1 and x2, and IPv4 endpoints y1 and y2, all in one group, with x1 first after Destination Address Selection and a Preferred Address Family Count of 1. The list is then S1, x1, y1, S2, x2, y2.

# Starting Connection Attempts {#starting}

A client that orders SCION first ({{sorting}}) treats a path lookup as HEv3 treats a DNS query for the preferred address family. It applies the conditions of Section 4.2 of {{I-D.ietf-happy-happyeyeballs-v3}} for starting connection attempts as follows:

* SCION is the preferred address family while the client has a SCION endpoint whose path lookup has not failed.

* A SCION endpoint in the list of endpoints counts as a positive answer for the SCION family. The client then starts connection attempts without waiting for other answers.

* A path lookup in progress is an answer that has not been received. Once the client has a positive answer for an IP address family, it waits for the lookup no longer than the Resolution Delay and then starts with the IP endpoints. A SCION endpoint whose lookup completes later is added to the list ({{path-lookup}}). A client that wants to wait longer for path lookups can configure a longer Resolution Delay (Section 10 of {{I-D.ietf-happy-happyeyeballs-v3}}), which also lengthens the wait for DNS answers.

A client that orders SCION last does not wait for path lookups.

# Connection Attempts {#attempts}

A SCION endpoint takes its turn in the list of endpoints as an IP endpoint does. When its turn comes, the client selects candidate paths ({{path-selection}}) and attempts the endpoint on them ({{path-attempts}}).

## Candidate Paths {#path-selection}

When the client attempts a SCION endpoint, it selects the candidate paths, in an order set by local policy, from the paths to the endpoint's AS that are eligible at that time. It selects no more than the Path Attempt Count. The default Path Attempt Count is 2, and values above 3 are NOT RECOMMENDED ({{load}}). If no path is eligible, the client skips the endpoint and attempts the next one at once.

Without a more specific policy, a client SHOULD first select a path with the fewest AS hops, at random among equals. For each further candidate path, up to the Path Attempt Count, it SHOULD select the eligible path that has the least overlap with the paths already selected ({{design-rationale}}). Overlap is compared by the number of shared transit ASes, then of shared pairs of adjacent ASes, then of shared inter-AS links. A transit AS, a pair, or a link counts once, however many selected paths have it. Among paths of equal overlap, the client selects one with the fewest AS hops, at random among equals. Paths that traverse the same inter-AS links count as one path, and the client uses the one with the latest expiry.

Router clocks need to agree with those of core ASes only to within minutes (Section 4.2.2.3 of {{I-D.dekater-scion-dataplane}} and Section 7.3 of {{I-D.dekater-scion-controlplane}}), so a router can treat a path as expired before the client does. A client therefore SHOULD NOT select a path that expires within 5 minutes unless no other path is left.

The candidate paths of a SCION endpoint do not change once the client has started to attempt it. A client MUST NOT send on an expired path: it does not start an attempt on a candidate path that has expired, and an attempt in progress fails when its path expires.

## Attempts over Paths {#path-attempts}

Each connection attempt to a SCION endpoint uses one candidate path: the client sends every packet of the attempt on that path. In all other respects it is a connection attempt as in Section 6 of {{I-D.ietf-happy-happyeyeballs-v3}}.

The client starts an attempt on the first candidate path. Each time the Path Attempt Delay passes and no attempt has succeeded, it starts an attempt to the same endpoint on the next candidate path, until it has started an attempt on each candidate path. Each attempt is a separate connection (Section 6 of {{I-D.ietf-happy-happyeyeballs-v3}}).

The default Path Attempt Delay is 50 milliseconds ({{design-rationale}}). A client MAY use another value, for example one that it derives from historical round-trip time data (Section 6 of {{I-D.ietf-happy-happyeyeballs-v3}}). A client MUST NOT use a Path Attempt Delay below 10 milliseconds, the lower bound of Section 6 of {{I-D.ietf-happy-happyeyeballs-v3}} for the time between two attempts, and SHOULD NOT use one above the Connection Attempt Delay. The lower bound also applies between any of these attempts and an attempt to another endpoint.

The attempts on candidate paths do not change the timing of other endpoints. Only the first attempt to a SCION endpoint sets the Next Connection Attempt Timer, and the attempts to one SCION endpoint count as one attempt for the Connection Attempt Delay Multiplier.

An attempt to a SCION endpoint succeeds as described in Section 6.1 of {{I-D.ietf-happy-happyeyeballs-v3}}. The client then cancels the other attempts, including those to the same SCION endpoint, as Section 6 of {{I-D.ietf-happy-happyeyeballs-v3}} specifies.

## Transport Requirements {#transport}

Attempts to one SCION endpoint share the client's SCION address and the destination SCION address and port. A client whose transport identifies connections by addresses and ports MUST use a different local port for each concurrent attempt to the same destination SCION address and port. TCP identifies connections in this way, and so does QUIC for a client that uses zero-length connection IDs (Sections 5.1 and 5.2.1 of {{RFC9000}}).

The server chooses the paths of its replies, which need not be the reverse (Section 2.4.4 of {{I-D.dekater-scion-dataplane}}) of the path of the attempt. The client MUST accept replies from the SCION address of the endpoint on any path. It MUST present the same server address to the transport regardless of the path a reply arrives on (Section 9 of {{RFC9000}}). A reply does not show that the reverse of its path carries the client's packets. The client therefore MUST continue to send on the path of the attempt until the attempt succeeds or fails.

When a QUIC client cancels an attempt, it SHOULD send a CONNECTION_CLOSE frame on the path of that attempt so that the server can release its state. Section 10.2.3 of {{RFC9000}} permits this frame in an Initial packet, which the client uses as long as it has not sent a Handshake packet of that attempt (Section 17.2.2.1 of {{RFC9000}}).

## Early Data {#early-data}

Early data (QUIC 0-RTT data {{RFC9001}} or other TLS early data {{RFC9846}}) that is sent in more than one attempt can be processed more than once (Section 8 of {{RFC9846}}). A client SHOULD NOT send the same early data in more than one attempt of a connection establishment. If the attempt that carried early data does not succeed, the client treats the early data as possibly processed. The application protocol decides whether the client sends that data again, as for rejected early data (Section 4.3.10 of {{RFC9846}}).

## SCMP Error Messages {#scmp}

A client SHOULD NOT end an attempt because of a SCION Control Message Protocol (SCMP) error message (Section 6 of {{I-D.dekater-scion-controlplane}}). Section 8.3.4 of {{I-D.dekater-scion-controlplane}} specifies no authentication for these messages and advises endpoints to treat them as weak hints. An AS on the path of an attempt could otherwise end that attempt with a forged message. A client whose attempts share a local port needs to ensure that such a message does not close the socket that they share.

## Applications That Require SCION {#scion-required}

An application can require that its connection use SCION. The client then MUST NOT attempt IP endpoints. It waits for path lookups up to the limit of {{path-lookup}}.

## After Establishment {#after-establishment}

The procedure of this document ends when a connection is established. Its result is one connection on one path. Finding further paths, moving to another path, and using several paths at once belong to the transport. For QUIC, the multipath extension {{RFC10069}} provides the means to use several paths in one connection, and leaves finding paths and the decision to open or close them to the application. Its use over SCION is not specified here.

# Summary of Configurable Values {#config}

In addition to the values of Section 10 of {{I-D.ietf-happy-happyeyeballs-v3}}, this document defines the following values and their defaults. As that section allows, a client MAY use other values, within the limits stated in the sections referenced.

* Path Attempt Count ({{path-selection}}): 2.

* Path Attempt Delay ({{path-attempts}}): 50 milliseconds.

* Address family order ({{sorting}}): SCION first.

# Experiment Goals {#experiment}

This document is Experimental because attempting SCION addresses on several paths within HEv3 has not been deployed. The experiment affects clients that implement this document, the servers that they connect to over SCION, and the control services that answer their path lookups. A participating client that orders SCION first can delay its first attempt to an IP endpoint by up to the Resolution Delay, and by one Connection Attempt Delay for each SCION endpoint ordered before that IP endpoint.

On a SCION testbed with failure injection and on a production SCION network, the experiment is to answer the following questions:

* How long do path lookups take, with and without cached paths, and how often does the first attempt to an IP endpoint start before the lookup completes?

* What does ordering SCION first cost a connection that is established over IP, compared with ordering IP first?

* Does a second path pay off? Compare a Path Attempt Count of 1, 2, and 3 by the median and the 99th percentile of the connection establishment time, with all paths working and with the first path failing silently, that is, dropping all packets of the attempt. Also compare, under partial packet loss on the first path, a second attempt on a different path with a second attempt on the same path. Measure in production how often a first path fails silently.

* What is the right Path Attempt Delay? Compare 10, 50, 100, and 250 milliseconds by the same two times, and by the number of handshakes, the octets sent, and the server connection state for each established connection. Also test whether a delay derived from round-trip time data does better than a fixed one.

* Do the candidate paths fail together? Measure in production how many transit ASes, pairs of adjacent ASes, and inter-AS links the candidate paths share. Failures were too rare in production to count joint failures there ({{results}}), so measure on the testbed, with a failure injected at a transit AS and at an inter-AS link, how often no candidate path works.

For each connection establishment, the experiment records which attempt succeeded, how many attempts the client started and how many it cancelled, and how many handshakes the server completed.

A Path Attempt Count of 2 is kept as the default if, compared with a Path Attempt Count of 1, it meets two conditions. First, with the first path failing silently, it lowers the 99th percentile of the connection establishment time by at least 100 milliseconds, the Minimum Connection Attempt Delay that Section 6 of {{I-D.ietf-happy-happyeyeballs-v3}} recommends. Second, with all paths working, it raises the median by no more than 2 milliseconds, which is about the variation between repeated runs on the testbed. The default Path Attempt Delay is kept if, in addition, the client starts no more than 1.1 attempts on average for each connection that it establishes over SCION with all paths working, that is, at most one further handshake for every ten connections. Otherwise these defaults change. The authors chose these three limits; they are not derived from measurements. The order of the address families is not subject to such a condition: only SCION attempts give the application a choice of path ({{sorting}}), and the experiment reports what ordering SCION first costs. The authors will report further results, and any change to the defaults, in a revision of this document.

## Results So Far {#results}

The testbed results come from the prototype of {{implementation-status}} in a testbed of 12 SCION ASes, with 400 connection establishments for each configuration, and 200 for each configuration with an inter-AS link or an AS dropping packets. A handshake on a working path took about 28 milliseconds.

A path lookup took a median of 5 milliseconds with cached paths. It took 35 milliseconds when the client forced the path service to fetch the paths again, which is what "uncached" means in this section. The first attempt to an IP endpoint started before the lookup completed in at most 5.5% of connection establishments with cached paths and in 95.5% without.

Ordering SCION first was compared with ordering IP first by the median connection establishment time. With cached paths, the connection was established over SCION and took 7 milliseconds longer than with IP first. With uncached paths, the client waited the Resolution Delay of 25 milliseconds for the lookup and then established the connection over IP, 25 milliseconds later than with IP first. When no SCION attempt could succeed, the client established the connection over IP after one Connection Attempt Delay, 257 milliseconds later.

The times in {{tab-count}} and {{tab-delay}} are connection establishment times in milliseconds.

| Path Attempt Count | 1 | 2 | 3 |
|---|---|---|---|
| Median, all paths working | 35.6 | 34.9 | 34.5 |
| 99th percentile, first path failing silently | 297.4 | 96.4 | 93.7 |
{: #tab-count title="Path Attempt Count, with a Path Attempt Delay of 50 milliseconds"}

A count of 2 met both conditions above. A third path gained nothing.

| Path Attempt Delay (ms) | 10 | 25 | 35 | 50 | 100 | 250 |
|---|---|---|---|---|---|---|
| Median, all paths working | 34.7 | 34.4 | 34.8 | 34.9 | 34.9 | 34.3 |
| Median, first path failing silently | 45.6 | 60.2 | 71.0 | 85.4 | 135.3 | 285.6 |
| Attempts started for each connection, all paths working | 2.0 | 1.7 | 1.0 | 1.0 | 1.0 | 1.0 |
{: #tab-delay title="Path Attempt Delay, with a Path Attempt Count of 2"}

The server completed as many handshakes as the client started attempts. A delay of 10 milliseconds thus cost a second handshake for every connection and did not lower the median. With the first path failing silently, the median was the delay plus about 36 milliseconds. A delay of 50 milliseconds met the condition above for the delay.

A delay derived from round-trip time data was tested as the smoothed round-trip time plus twice its variation, taken from earlier connections on the same path, and 50 milliseconds without such data. It had a median of 38 milliseconds. With all paths working, the client started no second attempt and the median did not change. With the first path failing silently, the median was 73.4 milliseconds, 12 milliseconds lower than with the fixed delay.

A second attempt on the same path was compared with one on a different path, with a Path Attempt Count of 2. With all paths working, the times were the same. A path that drops all packets fails every attempt on it, so the two were compared with 30% packet loss on the first inter-AS link of the first path: the 99th percentile was 204 milliseconds on the same path and 98 milliseconds on a different path.

To test whether the candidate paths fail together, one of the three inter-AS links of the client's AS was set to drop all packets. The first path used that link in about one third of connection establishments. With a Path Attempt Count of 1, these (70 of 195) were established over IP after one Connection Attempt Delay. With a count of 2 or 3, a candidate path worked in every connection establishment, because the candidate paths shared no transit AS and no inter-AS link. Dropping all packets at a neighboring transit AS gave the same result, because every path through that AS uses the same link of the client's AS. A failure further along a path has not been tested.

The production results come from one AS of the SCIERA network {{SCIERA}} in November 2026.

* 78 ASes were reachable, over 19,786 paths in total and between 1 and 1,600 paths each. The round-trip time to the last router of the first path selected ({{path-selection}}) had a median of 39 milliseconds and ranged from under 1 to 155 milliseconds. For 87% of these paths, the round-trip time plus 5 milliseconds was at most 50 milliseconds.

* In two passes of probes about 70 minutes apart, every path that was probed answered. In a separate measurement of up to 6 paths to each AS, 2 of 1,272 measurements got no answer, and both paths answered before and after. Silent failure of a path was thus rare.

* A second path was faster than the first by more than 10 milliseconds in fewer than 1% of selections. Attempts on several paths therefore do not find a faster path; they protect against a path that does not work.

* When the second path was selected by shared inter-AS links alone, it traversed on average 94% of the pairs of adjacent ASes of the first. With the selection of {{path-selection}}, it traversed 72% of them and 93% of the transit ASes of the first, had more AS hops in 71% of selections, and had a median round-trip time 0.5 milliseconds higher. A second path that shared no inter-AS link with the first existed for 2 of the 78 ASes.

* Path lookups took a median of 2 milliseconds with cached paths. Lookups that the client forced the path service to repeat were measured for 14 ASes. Their median was between 8 and 19 milliseconds for 7 of these ASes, 377 milliseconds for one, and 9 seconds for 6, which exceeds the limit of {{path-lookup}}.

# Security Considerations {#security}

The security considerations of {{I-D.ietf-happy-happyeyeballs-v3}} and {{I-D.john-scion-svcb}} apply.

This document relies on SCION path authorization (Section 7.1 of {{I-D.dekater-scion-dataplane}}), which ensures that packets follow paths that the ASes on them authorized, and on TLS to authenticate the server. Attacks on the SCION control plane are out of scope except as they affect path selection.

## Attempts over Several Paths {#several-paths}

Each attempt exposes the client's first flight to the ASes on its path. QUIC Initial packets are protected with keys that anyone who sees the client's first Initial packet can derive (Section 5.2 of {{RFC9001}}). Each of these ASes can therefore read the TLS ClientHello, including the server name unless Encrypted Client Hello {{RFC9849}} is used. The client starts an attempt on a further candidate path whenever the earlier attempt is late, so an AS on the earlier path can extend this exposure to a further path by delaying packets.

An AS on the path of an attempt, or on a path that the server chooses for its replies, can make that attempt fail by dropping its packets. An AS on the path of an attempt can also race forged Initial packets (Section 21.1.1.3 of {{RFC9000}}) or forge SCMP error messages ({{scmp}}). None of these actions can make an attempt succeed if success requires a completed QUIC or TLS handshake (Section 6.1 of {{I-D.ietf-happy-happyeyeballs-v3}}), because that handshake authenticates the server. By making the other attempts fail, an AS can make the client use a path that the AS is on.

An AS on the path of an attempt can replace the remaining path in the header of the attempt's packets with other authorized segments, and restore it on replies (Section 7.2.1 of {{I-D.dekater-scion-dataplane}}). The success of an attempt therefore shows that the server was reached, not that the packets followed the candidate path.

## Path Selection {#selection-security}

The hops and inter-AS links that the default selection counts are declared by the ASes on the path. ASes can announce many parallel links or fake peering links (Section 8.3.4 of {{I-D.dekater-scion-controlplane}}) so that their paths appear short or disjoint from others and are selected. Paths that differ only in the links between the same two ASes can fail together, which is why the default selection counts transit ASes and pairs of adjacent ASes before links.

## Load {#load}

Each attempt costs the client and the server a handshake: a key exchange, connection state, and the first flights of both. A Path Attempt Delay shorter than the time in which the first attempt succeeds makes every connection over SCION cost more than one handshake, up to one for each candidate path ({{design-rationale}}). An attempt that the client cancels holds server state until the server receives the client's CONNECTION_CLOSE frame ({{transport}}) or abandons the handshake. A server that limits the concurrent handshakes of one client address needs to allow for one handshake for each candidate path.

Attempts to one SCION endpoint share the client's and the server's hosts and can share links within both ASes. When packets are lost there, more attempts add load and are no more likely to succeed. This is why the Path Attempt Count is small.

## Downgrade {#downgrade}

An attacker that blocks SCION packets or path lookups, or that makes the attempts on all paths fail, can make the client connect over IP and lose the choice of path that SCION provides. The client's own control service can do the same by withholding paths, or by delaying its answer beyond the Resolution Delay ({{starting}}). Applications that require SCION exclude IP endpoints ({{scion-required}}).

# Privacy Considerations

Path lookups disclose the destination ISD-AS to the client's control service and, for uncached paths, to core ASes of the destination ISD; core ASes of the client's ISD learn the destination ISD (Section 6 of {{I-D.john-scion-svcb}}). Because a client looks up paths when it creates a SCION endpoint ({{path-lookup}}), it also discloses the destination for endpoints that it does not attempt and for services that it then reaches over IP.

Each attempt exposes the client's SCION address and the ClientHello to the ASes on its path ({{several-paths}}). A Path Attempt Count of 1 limits this exposure to one path for each SCION endpoint. Attempts to one SCION endpoint carry the same two SCION addresses, so an observer on more than one of their paths can link them.

Historical data about paths that the client keeps is partitioned as Sections 5.3 and 9 of {{I-D.ietf-happy-happyeyeballs-v3}} specify. A path cache shared with other applications, such as that of a local path service, is outside the client's control.

# IANA Considerations {#iana}

This document has no IANA actions.

--- back

# Design Rationale {#design-rationale}

Separate connections:
: Each attempt on a path is an ordinary connection, so the rules of HEv3 for success and cancellation apply as they are, and neither the transport nor the server needs to change. The cost is a handshake and server state for each attempt after the first. Racing paths within one QUIC connection, by copying its datagrams to further paths below the transport, would avoid that cost. That approach is contrary to Section 6 of {{I-D.ietf-happy-happyeyeballs-v3}}, hides the copies from congestion control and round-trip time estimation, and lets the server's choice of reply path decide which path completes the handshake.

Paths that differ:
: The first path has the fewest AS hops because each AS and each inter-AS link adds delay. A second attempt adds little if its path shares most links with the first. The overlap is counted from the widest scope of failure to the narrowest. The failure of a transit AS affects every path through it. Parallel links between the same pair of adjacent ASes can fail together ({{selection-security}}). An inter-AS link is the narrowest scope. Section 2.3.3 of {{I-D.dekater-scion-controlplane}} makes a similar distinction, between AS-disjoint and link-disjoint paths, for the selection of path segments. The selection ranks partial overlap because disjoint paths were rare in production, where selection by links alone mostly chose parallel links ({{results}}). The default Path Attempt Count of 2 gives the client one alternative; a third gained nothing in the testbed ({{results}}).

A further path after 50 milliseconds:
: An attempt on a working path succeeds after about one round-trip time. A client that waits longer than that before it starts an attempt on a further path makes one handshake when the first path works, and loses little when it does not: in the testbed, the connection establishment time with a failing first path was the Path Attempt Delay plus the time on a working path ({{results}}). Starting the attempts 10 milliseconds apart cost a second handshake for every connection in the testbed, and in production the selected paths differed too little in round-trip time for this to find a faster one ({{results}}). The default is a fixed value because a fixed value needs no data about the destination. A delay of 50 milliseconds exceeded the round-trip time plus 5 milliseconds for 87% of the first paths selected in production. To a destination with a longer round-trip time, the client starts the second attempt even when the first path works; the Path Attempt Count bounds that cost, and a client can avoid it with a delay that it derives from round-trip time data. Section 6 of {{I-D.ietf-happy-happyeyeballs-v3}} permits a delay below the Minimum Connection Attempt Delay, down to 10 milliseconds. That minimum guards against congestion collapse under high packet loss; here the Path Attempt Count bounds the attempts started together.

Path lookups within the Resolution Delay:
: A lookup answered from the cache of a local path service took a median of 5 milliseconds in the testbed and 2 milliseconds in production. An uncached lookup took a median of 35 milliseconds in the testbed and, in production, between 8 milliseconds and 9 seconds depending on the destination AS ({{results}}). Waiting up to the Resolution Delay, 25 milliseconds by default, adds no parameter and covers most cached lookups when the SVCB or HTTPS answer arrives no later than the first IP answer. When paths are not cached, the first connection can go over IP and later ones over SCION.

# Implementation Status {#implementation-status}

(RFC Editor: please remove this section before publication.)

This section records the status of known implementations as of November 2026, following {{RFC7942}}.

scion-hev3 {{SCION-HEV3}}:
: A research prototype in Go on release v0.15.1 of the SCION reference implementation {{SCION-V0.15.1}}. Contact: Tony John. Last updated: November 2026.
