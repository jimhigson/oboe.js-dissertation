Introduction
============

HTTP was originally designed for the transfer of hypertext documents.
REST [@rest] introduces no fundamentally new methods but extends the
scope of HTTP to include the transfer of arbitrary data. Whereas the
rival technology SOAP [@soap] largely disregards HTTP's principled
design by adopting the protocol as a transport on which to bootstrap its
own semantics, REST adopts all of HTTP's core phrasing. This includes
the HTTP methods for fetching, creating and modifying resources: GET,
POST, PUT, PATCH, DELETE, and the locating of resources using URLs.
Under HTTP's original design hierarchical URLs are used to locate
documents without reference to the services which produce them. REST
advances this same naming strategy by likewise using URLs to locate data
resources, not services. As with HTTP, REST is stateless and therefore
cacheable, allowing large-scale content distribution networks to be
built. Because HTTP's inbuilt headers for content type, language
negotiation, and resource expiry are used according to the originally
intended meanings [@headers], existing intermediaries such as load
balancing proxies, gateways, and caches need make no special
accommodation for REST resources.

Despite REST adopting the mechanisms and semantics of HTTP, whereas
documents received over the web are often interpreted in a streaming
fashion, to date REST resources are not commonly examined in this way.
For most practical cases where we wish to be increase the speed of a
system there is no reasonable distinction between acting *earlier* and
being *quicker*. In the interest of creating efficient software we
should use data at the first possible opportunity: examining content
*while it streams* rather than holding it unexamined until it is wholly
available. The purpose of this dissertation is to explore tangible
benefits which may be realised if we fold HTTP streaming into the REST
paradigm.

Natural languages encourage our thinking to follow patterns that they
easily support [@whorf56]. This idea has been applied to programming,
for example Ruby is intentionally designed to discourage global
variables by using a less attractive notation [@rubylang]. It may be
useful when exploring new techniques to question which established
constructs are as they are because of languages which unintentionally
suggest that formulation; it is perhaps significant that REST clients
tend to style the calling of remote resources similarly to the call
style of the host programming language. In practice one of two schemas
is generally followed: a synchronous, blocking style in which an
invocation halts execution for the duration of the request before
evaluating to the fetched resource; or an asynchronous, non-blocking
form in which some logic is specified to be applied to the response once
it is available. Languages which promote concurrency though threading
generally consider blocking in a single thread to be acceptable and will
prefer the synchronous mode whereas languages with first class functions
are naturally conversant in callbacks and will prefer asynchronous I/O.
In programming the language limits the patterns that we readily see and
the schemes which map most easily onto our languages are not necessarily
the best possible organisation. For any multi-packet message sent via a
network some parts will arrive before others, at least approximately
in-order, but viewed from inside a language whose phasing encourages
statements to yield single, wholly evaluated results it is comfortable
to conceptualise the REST response as a discrete event. This
establishment of a 'limiting comfort' extends to graphical notations
such as UML whose constructs strongly reflect the textual languages of
the day. UML sequence diagrams include the syntax for instantaneously
delivered return values but, despite being commonly used to draw network
data transfer, provide no corresponding notation for a resource whose
data is progressively revealed.

No new computing techniques need be invented before this dissertation
can be implemented. As a minimum it requires an HTTP client which
reveals the response whilst it is in progress and a parser which can
begin to interpret that response before it sees all of it. Nor is it
novel to use these parts together to produce a streaming interpretation.
Every current web browser already implements a similar pattern. Load any
complex webpage -- essentially an aggregation of hypertext and other
resources -- and the HTML will be parsed and displayed incrementally
while it is downloading, with resources such as images requested in
parallel as soon as they are referenced. In the case of progressive JPEG
or SVG[^1] the images themselves will also be presented incrementally.
The incremental display is achieved through specialised programming
which applies only to displaying web pages. The contribution of this
dissertation is to provide a generic analogue, applicable to any problem
domain.

REST aggregation could be faster
--------------------------------

![**Sequence diagram showing the aggregation of low-level REST resources
by an intermediary.** A client fetches an author's publication list and
then their first three articles. This sequence represents the most
commonly used technique in which the client does not react until the
response is complete.
\label{rest_timeline_1}](images/rest_timeline_1.png)

![**Revised aggregation sequence showing a client which progressively
interprets the resources.** Because UML sequence diagrams draw returned
values as a one-off happening rather than a continuous process, the
lighter arrow notation is added to represent fragments of an incremental
response. Each request for an individual publication is made as soon as
its URL can be extracted from the publications list and once all
required links have been read from the original response it is aborted
rather than continuing to download unnecessary data.
\label{rest_timeline_2}](images/rest_timeline_2.png)

Figures \ref{rest_timeline_1} and \ref{rest_timeline_2} comparatively
illustrate how a progressive client may, without adjustments to the
server, be used to produce an aggregated resource sooner. This results
in a moderate improvement in the time taken to show the complete
aggregation but a dramatic improvement in the time to show the first
content. The ability to present the first content as early as possible
is a desirable trait for system usability because it allows the user to
start reading earlier and a progressively rendered display in itself
increases the human perception of speed [@perceptionFaxSpeed]. Note also
how the cadence of requests is more steady in Figure
\ref{rest_timeline_2} with four connections opened at roughly equal
intervals rather than a single request followed by a rapid burst of
three. Both clients and servers routinely limit the number of
simultaneous connections per peer so avoiding bursts is further to our
advantage. [Appendix i](#appendix_http_limits) lists some actual limits.

Nodes in an n-tier architecture defy categorisation as 'client' or
'server' in a way that is appropriate from all frames of reference. A
node might be labeled as the 'server' from the layer below and 'client'
from the layer above. Although the "client software" labels in the
figures \ref{rest_timeline_1} and \ref{rest_timeline_2} hint at
something running directly on a user's own device, the same benefits
apply if this layer is running remotely. If this layer were generating a
web page on the server-side to be displayed by the client's browser, the
same perceptual speed improvements apply because of HTTP chunked
encoding [@perceptionHttpChunkedSpeed]. If this layer were a remote
aggregation service, starting to write out the aggregated response early
provides much the same benefits to the client able to interpret it
progressively and, even if it is not, the overall delivery remains
faster. Whilst HTTP servers capable of streaming are quite common, there
seems to be no published general-purpose, streaming-receptive REST
client library.

Stepping outside the big-small tradeoff
---------------------------------------

Where a domain model contains a series of data from which continuous
ranges are requestable via REST there is often a tradeoff in the client
design with regards to how much should be requested with each call.
Because at any time it shows only a small window into a much larger
model, the social networking site Twitter might be a good example. The
Twitter interface designers adopted a popular interface pattern,
Infinite Scrolling [@infinitescroll]. Starting from an initial page
showing some finite number of tweets, once the user scrolls and reaches
the end of the list the next batch is automatically requested. When
loaded, this new batch is converted to HTML and added to the bottom of
the page. Applied repeatedly the illusion of an infinitely long page is
maintained, albeit punctuated with pauses whenever new content is
loaded. There is a tradeoff in the presentation layer between
sporadically requesting many items and frequently requesting a few. At
one extreme the interface would occasionally falter for a longer time,
whereas at the other it would pause momentarily but with greater
regularity.

Progressive loading could render this tradeoff unnecessary by
simultaneously delivering the best of both strategies. In the Twitter
example this could be achieved by making large requests but instead of
deferring all rendering until the request completes, add the individual
tweets to the page as they are incrementally parsed out of the ongoing
response. With a streaming transport, the time taken to receive the
first tweet should not vary depending on the total number that are also
being sent so there is no relationship between the size of the request
and the interval before the interface starts updating.

Staying fast on a fallible network
----------------------------------

REST operates over networks whose reliability varies widely. On
unreliable networks connections are abruptly dropped and existing HTTP
clients handle unexpected terminations wastefully. Consider the everyday
situation of a person using a smartphone browser to check their email.
Mobile data coverage is often weak outside of major cities [@opensignal]
so while travelling the signal will be lost and reestablished many
times. The web developer's standard toolkit is structured in a way that
encourages early terminated connections to be considered as wholly
unsuccessful rather than as partially successful. For example, the
popular AJAX library jQuery automatically parses JSON or XML responses
before passing back to the application but given an early disconnection
there is no attempt to hand over the partial response. To the programmer
who knows where to look the partial responses can be extracted as raw
text but handling them involves writing a special case and is not
possible using standard parsers which are not amenable to incomplete
markup. Because of this difficulty the canonical webapp drops partial
messages without inspection. For the user checking her email, even if
90% of her inbox had been retrieved before the signal was lost, the web
application will behave as if it received none and show her nothing.
Later, when the network is available the inbox will be downloaded from
scratch including the 90% which has already been successfully delivered.
A more efficient system would allow the 90% from the aborted first
request to be used straight away and when the signal later returns fetch
only the lost 10%.

The delivered part of a partially successful message may be used if we
turn away from this polarised view of wholly successful and unsuccessful
requests. A message may be conceptualised as having many parts which are
useful in themselves, in which the successful delivery of each part is
handled independently before knowing if the next part will also arrive.
As well as allowing partially successful messages to be used, seeing the
resource as a stream of small parts allows those parts to be used
earlier if they are made available to the application while the
streaming is ongoing. Should an early disconnection occur, the content
delivered up to that point will have already been handled so no special
case is required to salvage it. In most cases the only recovery
necessary will be to make a new request for just the part that was
missed. This approach is not incompatible with a problem domain where
the usefulness of an earlier part is dependent on the correct delivery
of the whole providing optimistic locking is used. In this case earlier
parts may be used immediately but their effect rolled back should the
transfer later fail.

Agile methodologies, frequent deployments, and compatibility today with versions tomorrow
-----------------------------------------------------------------------------------------

In most respects a SOA architecture fits well with the fast release
cycle encouraged by Agile methodologies. Because in SOA we may consider
that all data is local and that the components are loosely coupled and
autonomous, frequent releases of any particular sub-system should pose
no problem to the correct operation of the whole. In allowing a design
to emerge organically it should be possible for the structure of
resource formats to be realised slowly and iteratively while a greater
understanding of the problem is gained. Unfortunately in practice the
ability to change often is hampered by tools which encourage programming
against rigidly specified formats. If a data consumer is tightly coupled
to the format it consumes it will resist changes to the programs which
produce data in that format. As an anecdote, working in enterprise I
have seen the release of dozens of components cancelled because of a
single unit that failed to meet acceptance criteria. By insisting on
exact data formats, subsystems become tightly coupled and the perfect
environment is created for contagion whereby the updating of any single
unit may only be done as part of the updating of the whole. An effective
response to this problem would be REST clients which are able to use a
resource whilst being only loosely coupled to the overall shape of the
message.

Deliverables
------------

To avoid feature creep the scope of the software deliverables is pared
down to the smallest work which can be said to realise the goals of the
project, the guiding principle being that it is preferable to produce a
little well than more badly. Amongst commentators on start-up companies
this is known as a *zoom-in pivot* [@lean p172] and the work it produces
should be the *Minimum Viable Product* or MVP [@lean p106-110]. It would
not be feasible to deliver a full stack so we are obliged to focus on
solutions which interoperate with existing deployments. To a third party
wishing to adopt the technology it is also more inviting to add small
enhancements to the existing architecture than to action a shift which
requires wholesale change.

Although an explicitly streaming server would improve the situation
further, because all network data transfer may be thought of as a
stream, it is not required to start taking advantage of progressive
REST. A streaming client is the MVP and is the programming deliverable
for this project.

Criteria for success
--------------------

In evaluating this project we may say it has been a success if
non-trivial improvements in speed can be made without a corresponding
increase in the difficulty of programming the client. This improvement
may be in terms of the absolute total time required to complete a
representative task or in a user's perception of application
responsiveness. Because applications in the target domain are much more
I/O-bound than CPU-bound, optimisation in terms of the execution time of
algorithms will be de-emphasised unless especially egregious. The
delivered library should allow looser coupling to the format of consumed
resources than is possible with the current best tools so that it is
less disruptive when services are upgraded.

[^1]: for quite an obviously visible example of progressive SVG loading,
    try loading this SVG using a recent version of Google Chrome:
    <http://upload.wikimedia.org/wikipedia/commons/0/04/Marriage_(Same-Sex_Couples)_Bill,_Second_Reading.svg>
    For the perfectionist SVG artist, not just the final image should be
    considered but also the XML source order, for example in this case
    it would be helpful if the outline of the UK appeared first and the
    exploded sections last.
