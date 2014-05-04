% Book Proposal: REST and Streaming
% Jim Higson, May 2014

Make it fast
============

There is a common belief, sometimes true, that creating fast systems requires writing fast algorithms.
Servicing a request in this age of cheap, fast CPUs and architectures composed from loosely coupled, remote 
resources will normally consume more time waiting for I/O than any other activity.
Can it be right that we create processes only to spend their lives waiting?

We have now a generation of highly asynchronous servers: Node.js, Netty, NginX. In Node the stream,
not the whole resource, is the primary abstraction. However, today's popular REST clients
are yet to embrace streaming and still wait until a resource arrives completely
before using it. We should want to act early: in most situations there is no real-world difference
between being reacting *earlier* and being *faster*.

A Journey
------

The passenger checks her email using a smartphone. As the train progresses
through the countryside the reception is lost and reestablished many times.
Let's look inside the email webapp. 
The web developer's standard toolkit is structured so that
connections that were terminated early but were partially successful are considered wholly
unsuccessful. The
popular AJAX library jQuery automatically parses JSON or XML responses
before passing back to the application but given an early disconnection
there are no means to parse the partial response. 
With applications following the path set out by the tooling,
our canonical webapp disregards partial
data without inspection. For our passenger, even if
90% of her inbox had been retrieved, if her signal is lost, the
web applications behaves as if it received nothing.
Later, when the network returns her inbox will be downloaded from
scratch, repeating the 90% which has already been successfully delivered.

By embracing streaming in REST we
stop seeing messages as wholly successful or unsuccessful.
The data may be conceptualised as having many parts which are
useful in themselves, in which the successful delivery of each part is
handled independently as soon as it arrives.
On early disconnection the content
delivered up to that point will have already been handled so no special
case is required to handle the partial data.

A Vote
-------

Consider the REST service provided by a news organisation for per-ward results in
an important election.
Where historic results are requested the data is delivered in JSON format much
as usual. But requests for ongoing vote results, an incomplete JSON with the wards known so far
would be immediately sent, followed by the remainder dispatched
live as the results are called. While the data is sent as a stream,
when all results are known, after-the-fact the
JSON closes as usual and leaves a standard, cacheable complete resource.
After the polls have closed somebody wishing to fetch the results would use the *same URL for the
historic data as was used during the event for the live stream*. This is
possible because cool URLs locate data by its meaning, indifferent to
the time when the request is made.

An application developer receiving streaming REST does not have to handle
live and historic data as separate cases, they can concentrate on
handling data regardless of the timing. Without branching, the code which
displays results as they are announced would automatically be able to
show historic data.
