# Does HTTP/3 really fix Head-of-Line blocking? Perhaps

*In this blogpost, we look at the different variations of the Head-of-Line blocking problem from HTTP/1.1 over HTTP/2 to the new HTTP/3. We dive deep into the basic building blocks of the protocols, but I use lots of images and examples to hopefully keep things tangible, even for novices. The goal is to help people make correct assumptions about HTTP/3's performance improvements, which might not be as amazing as sometimes claimed in marketing materials.*

As you may have heard, after 4 years of work, the new HTTP/3 and QUIC protocols are finally approaching official standardization. Preview versions are now [available for testing in servers and browsers alike][h3WithCaddy]. 

[h3WithCaddy]: https://ma.ttias.be/how-run-http-3-with-caddy-2/

HTTP/3 is promising major performance improvements compared to HTTP/2, mainly because it [changes its underlying transport protocol from TCP to QUIC][quicIntro]. In this post, we'll be taking an in-depth look at just one of these improvements, namely the promised removal of the **"Head-of-Line blocking" (HOL blocking) problem**. This is useful because I've read a lot of misconceptions on what this actually means and how much it helps in practice. Solving HOL blocking was also one of the main motivations behind not just HTTP/3 and QUIC but also HTTP/2. 

[quicIntro]: https://ma.ttias.be/googles-quic-protocol-moving-web-tcp-udp/

I'll first introduce the problem and then track different forms of it throughout HTTP's history. Spoiler: much like the [cake][cake], HOL blocking removal in HTTP/3 is probably a lie. 

[cake]: https://knowyourmeme.com/memes/the-cake-is-a-lie

## What is Head-of-Line blocking?

It's difficult to give you a single technical definition of HOL blocking, as this blogpost alone describes four different variations of it. A simple, conceptual definition however would be:

> When a single (slow) object prevents other/following objects from making progress

A good real-life metaphore would be a grocery store with just a single check-out counter. One customer who's buying a lot of items can end up delaying everyone behind them, as customers are served in a First In, First Out manner. Another example could be a highway with just a single lane. One car crash on this road can end up jamming the entire passage for a long time. As such, even a single issue at he "head of the line" can "block" the entire line. 

As it turns out, that concept has been one of the hardest Web performance problems to solve. To understand this, let's start at its incarnation in our trusted workhorse: HTTP version 1.1 

## HOL blocking in HTTP/1.1

HTTP/1.1 is a protocol from a simpler time. A time when protocols could still be text-based and readable on the wire. This is illustrated in Figure 1 below:

[TODO: Figure 1]
*Figure 1: server HTTP/1.1 response for `script.js`*

In this case, the browser requested the simple `script.js` file (green) over HTTP/1.1, and Figure 1 shows the server's response to that request. We can see that the HTTP aspect itself is really straightforward: it just adds some textual "headers" (red) directly in front of the plaintext file content, that's it. Headers + content are then passed down to the underlying TCP (orange) for actual transport to the receiver. For this example, let's pretend we cannot fit the entire file into 1 TCP packet and it has to be split up into two parts.

*Note: in reality, when using HTTPS, there is another layer in between HTTP and TCP for security with the TLS protocol. However, we omit that in our discussion here for clarity. I did include a [bonus section at the end of this post](#sec_tls) that details a TLS-specific HOL blocking variant and how QUIC prevents it.* 

Now let's see what happens when the browser also requests `style.css` in Figure 2:

[TODO: Figure 2]
*Figure 2: server HTTP/1.1  response for `script.js` and `style.css`*

In this case, we are sending `style.css` (purple) after the response for `script.js` has been transmitted. The headers and content for `style.css` are simply appended after the JavaScript (JS) file. The receiver uses the **Content-Length** header for each response to know where one ends and the other starts (in our simplified example, let's pretend `script.js` is 1000 bytes large, while `style.css` is just 600 bytes).  

All of that seems sensible enough in this simple example with two small files. However, let's imagine a scenario in which the JS file is much larger than CSS (say 1MB instead of 1KB). In this case, the CSS would have to wait before the entire JS file was downloaded, even though it is much smaller and thus could be parsed/used earlier, which can be better for Web performance. If we visualize this a bit more directly, using the number 1 for `large_script.js` and 2 for `style.css`, we would get something like this:

> 11111111111111111111111111111111111111122

You can see this is an instance of the Head-of-Line blocking problem! Now you might think: that's easy to solve! Just have the browser request the CSS file before the JS file! Crucially however, the browser has **no way of knowing up-front which of the two will end up being the larger file at request time**, as there is no way to for instance indicate in the HTML how large a file is (this would be lovely, HTML working group: `<img src="thisisfine.jpg" size="15000" />`).  

The "real" solution to this problem would be to employ **multiplexing**. If we can cut up each file into smaller pieces (instead of sending them as one big blob), we can "interleave" those chunks on the wire (send a chunk for the JS, one for the CSS, then another for the JS again, etc. until the files are downloaded). With this approach, the smaller CSS file will be downloaded (and usable) much earlier, while only delaying the larger JS file by a bit. Visualized with numbers we would get:

> 12121111111111111111111111111111111111111

Sadly however, this multiplexing is not possible in HTTP/1.1 due to some fundamental limitations with the protocol's assumptions. To understand this, we don't even need to keep looking at the large-vs-small resource scenario, as it already shows up in our example with the two smaller files. Consider Figure 3, where we interleave just 4 chunks for the two resources:

[TODO: Figure 3]
*Figure 3: server HTTP/1.1 multiplexing for `script.js` and `style.css`*
 
The main problem here is that HTTP/1.1 is a purely textual protocol that only appends headers to the front. It does nothing further to differentiate individual (chunks of) resources from one another. As such, in Figure 3, the browser starts parsing the headers for `script.js` and expects 1000 bytes of response body (the Content-Length). It however only receives 450 JS bytes (the first JS chunk) and then starts reading the headers for `style.css`. It ends up interpreting the CSS headers and the first CSS chunk as part of the JS, as the file contents and headers for both files are just plaintext. Making matters worse, it stops after reading 1000 bytes, ending up somewhere halfway through the second `script.js` chunk. At this point, it doesn't see valid new headers and has to drop the rest of TCP packet 3. The browser then passes this weird text blob to the JS parser, which fails because it's not valid JavaScript:

```
function first() { return "hello"; }
HTTP/1.1 200 OK
Content-Length: 600

.h1 { font-size: 4em; }
func
```

Again, you could say there's an easy solution: have the browser look for the `HTTP/1.1 {statusCode} {statusString}\n` pattern to see when a new header starts. That might work for TCP packet 2, but will fail in packet 3: how would the browser know where the green `script.js` chunk ends and the purple `style.css` chunk begins? 

This all is a fundamental limitation of the way the HTTP/1.1 protocol was designed. It means that, if you have a single HTTP/1.1 connection, resource responses always have to be delivered **in-full** before you can switch to sending a new resource. This can lead to severe HOL blocking issues if earlier resources are slow to create (for example a dynamically generated `index.html` that is filled from database queries) or, as above, if earlier resources are simply quite large.

This is why browsers [started opening multiple parallel TCP connections][parallelConnections] (typically 6 nowadays) for each page load over HTTP/1.1. That way, requests could be distributed across those individual connections and there is no more HOL blocking. That is, unless you have more than 6 resources per page... That's where the practice of "sharding" your resources over multiple domains and Content Delivery Networks (CDNs) comes from: as each sharded domain gets 6 connections, browsers will open up to 30-ish TCP connections in total for each page load. This works, but has considerable overhead: setting up a new TCP connection can be expensive (for example in terms of state and memory at the server) and takes some time (especially for an HTTPS connection). 

[parallelConnections]: http://www.stevesouders.com/blog/2008/03/20/roundup-on-parallel-connections/

As this problem cannot be solved with HTTP/1.1 and the patchwork solution of parallel TCP connections didn't scale too well over time, it was clear a totally new approach was needed, which is what became HTTP/2. 

*Note: the old guard reading this might wonder about HTTP/1.1 pipelining. I decided not to discuss that here to keep the overall story flowing, but people interested in even more technical depth can read the [bonus section at the end of this post](#sec_pipelining).

<a name="sec_http2"></a>
## HOL blocking in HTTP/2 over TCP

So, let's recap. We now know that HTTP/1.1 has a HOL blocking problem where a large or slow response can end up delaying other and smaller responses behind it. This is mainly because the protocol is purely textual in nature and doesn't use delimiters between resource chunks. As a workaround, browsers open many parallel TCP connections, which is not efficient and doesn't scale.

As such, the goal for HTTP/2 was quite clear: make it so that we can **move back to a single TCP connection by solving the HOL blocking problem**. Stated differently: we want to enable proper multiplexing of resource chunks. This wasn't possible in HTTP/1.1 because there was no way to discern to which resource a chunk belonged, or where it ended and another began. HTTP/2 solves this quite elegantly by prepending small control messages, called **frames**, before the resource chunks. This can be seen in Figure 5:

[TODO: Figure 5]
*Figure 5: server HTTP/1.1 vs HTTP/2 response for `script.js`*

Unlike HTTP/1.1, HTTP/2 puts a so-called DATA frame in front of each chunk. These DATA frames mainly indicate two critical pieces of metadata: which resource the following chunk belongs to (each resource "stream" is assigned a unique number, the **stream id**) and how large this chunk is. The protocol has many other frame types as well, of which Figure 5 also shows the HEADERS frame. This again uses a stream id to indicate which response these headers belong to (so that headers can even be split up from their actual response data).

Using these frames, it follows that HTTP/2 indeed allows proper multiplexing of several resources on one connection, see Figure 6:

[TODO: Figure 6]
*Figure 6: multiplexed server HTTP/2 responses for `script.js` and `style.css`*

Unlike our example for Figure 3, the browser can now deal with this situation perfectly. It first processes the HEADERS frame for `script.js` (which also has a length built-in) and then the DATA frame for the first JS chunk. Because the DATA frame includes length of the chunk and this is perfectly contained in TCP packet 1, it knows that it needs to look for a completely new frame starting in TCP packet 2, where it indeed finds the HEADERS for `style.css`. The next DATA frame has a different stream id (2) than the first DATA frame (1), so the browser knows this belongs to a different resource and can process it independently. The same happens for TCP packet 3, where the DATA frame stream ids are used to "de-multiplex" the response chunks to their correct resource "streams". 

We can see that, by "framing" individual messages, HTTP/2 is much more flexible than HTTP/1.1 and allows for many resources to be sent on a single TCP connection at once. We can thus say: HTTP/2 solves HTTP/1.1's HOL blocking problem, our work here is done, let's go home. 

Well, not so fast there bucko. [We've solved HTTP/1.1 HOL blocking, yes, but what about TCP HOL blocking](https://4.bp.blogspot.com/-n4LJF-HJfS4/VuhfpUkYOxI/AAAAAAAAPTQ/H0I9ZU-lJGMY0dURTJZW-DwE_WenWooqQ/s1600/hobbit%2Bsecond.gif)? 

- protocols have different perspectives. see Figure 7. 

[TODO: Figure 7]
*Figure 7: bytestream perspectives of HTTP/2 vs TCP*

HTTP/2 doesn't know about JS or CSS, just the stream ids. Similarly, TCP doesn't know about these streams: it just sees data coming from above that it needs to send in packets. 
- As such, there is a mismatch: TCP doesn't know about different streams. Look at the byte counts: they go up over time. 
- Seems like unnecessary detail, but is important when we remember that TCP packets can be "lost" in the network and then need to be retransmitted. 

- what happens if packet 2 is lost but packet 3 is not...
- Maybe even clearer if packet 1 is lost: packet 2 and 3 need to wait, despite being imminently usable




- single connection
- binary + framing

- H2 introduces framing, so yay, multiplexing!
- However, still on 1 TCP stream, so vulnerable to packet loss and re-ordering

- mention prioritization

- However: much less of a problem in practice: low % is packet loss + bursty (important later)

- make a point of performance: multiple parallel connecttions is faster than a single connection? Complex, has to do with Congestion Control etc. something for another blogpost. In general: it depends, doesn't mean H1 or H2 (or H3) is always slower/faster. 

<a name="sec_http3"></a>
## HOL blocking in HTTP/3 over QUIC

- QUIC takes H2's streams over to the transport (put differently: each QUIC stream is its own TCP connection. like setting up as many TCP connections as you want with 1 handshake)
- This does get rid of Transport-layer HOL blocking due to packet loss, because each stream does retransmits individually
- Or does it...

- Note: we abstract away a lot. Normally, the sizes of the FRAMES would count as well, but we omit those here to get nice round numbers.

- Consider that, per stream, data also needs to be delivered in-order of course (put differently: previously we were HOL blocked inter-stream, now only intra-stream)
- Now, resource multiplexing comes into play: sequential vs RR 
- we can see: if sequential = still holblocking, little benefit from QUIC

- Why does this matter? Sequential is thought to be better for web performance...

<a name="sec_conclusion"></a>
## Conclusion

- So... in theory: it's solved
- In practice: the jury is still out. 
- Either way: it's not much of an issue in practice, especially on fast networks

<a name="sec_pipelining"></a>
## Bonus: HTTP/1.1 pipelining

HTTP/1.1 includes a feature called "pipelining" which is in my opinion often misunderstood. I've seen many posts and even books where people claim that HTTP/1.1 pipelining solves the HOL blocking issue. I've even seen some people saying that pipelining is the same as proper multiplexing. Both statements are [false](https://theofficeanalytics.files.wordpress.com/2017/11/dwight.jpeg?w=1200). 

I find it easiest to explain HTTP/1.1 pipelining with an illustration like the one in Bonus Figure 1:

[TODO: Bonus Figure 1]
*Bonus Figure 1: HTTP/1.1 pipelining*

Without pipelining (left side of Bonus Figure 1), the browser has to wait to send the second resource request until the response for the first request has been completely received (again using Content-Length). This adds one Round-Trip-Time (RTT) of delay for each request, which is bad for Web performance.  

With pipelining then (middle of Bonus Figure 1), the browser does not have to wait for any response data and can now send the requests back-to-back. This way we save some RTTs during the connection, making the loading process faster. *As a side-note, look back at Figure 2: you see that pipelining is actually used there as well, as the server bundles data from the `script.js` and `style.css` responses in TCP packet 2. This is of course only possible if the server received both requests around the same time.*

Crucially however, this pipelining only applies to the **requests** from the browser. As the [HTTP/1.1 specification][h1spec] says:

> A server MUST send its responses to those [pipelined] requests in the same order that the requests were received.

As such, we see that actual multiplexing of response chunks (illustrated on the right side of Bonus Figure 1) is still not possible with HTTP/1.1 pipelining. Put differently: **pipelining solves HOL blocking for requests, but not for responses**. Sadly, it's arguably the response HOL blocking that causes the most issues for Web performance. 

To make matters worse, most browsers actually do not use HTTP/1.1 pipelining in practice because it can make HOL blocking more unpredictable in the setup with multiple parallel TCP connections. To understand this, let's imagine a setup where three files A (large), B (small) and C (small) are being requested from the server over two TCP connections. A and B are each requested on a different connection. Now, on which connection should the browser pipeline the request for C? As we said before, it doesn't know up-front whether A or B will turn out to be the largest/slowest resource. 

If it guesses correctly (B), it can download both B and C in the time it takes to transfer A, resulting in a nice speedup. However, if the guess is wrong (A), the connection for B will be idle for a long time, while C is HOL blocked behind A. This is because HTTP/1.1 also does not provide a way to "abort" a request once it's been sent (something which HTTP/2 and HTTP/3 do allow). The browser can thus not simply request C over B's connection once it becomes clear that is going to be the faster one, as it would end up requesting C twice. 

To get around all this, modern browsers do not employ pipelining and will even actively delay requests of certain discovered resources (for example images) for a little while to see if more important files (say JS and CSS) are found, to make sure the high priority resources are not HOL blocked. 

It is clear that the failure of HTTP/1.1 pipelining was another motivation for HTTP/2's radically different approach. Yet, as HTTP/2's prioritization system to steer multiplexing often fails to perform in practice, some browsers have even taken to delaying HTTP/2 resource requests as well to get optimal performance. 

**TODO: add pipelining-in-browsers + prioritization-delaying-in-chrome refs (dig through my papers for these)**

<a name="sec_tls"></a>
## Bonus: TLS HOL blocking

- TLS encrypts in blocks of up to 16KB for efficiency
- if the last X of those blocks are lost, we cannot decrypt the first N - X blocks
- so there is a kind of TLS-specific HOL blocking happening if the final chunks of a TLS record are lost

- this is one of the reasons that QUIC encrypts one packet at a time
- This is less efficient (and one of the main reasons QUIC requires more CPU than TCP), but prevents QUIC-level HOL blocking even in these edge cases.


[tlsSizing]: https://www.igvita.com/2013/10/24/optimizing-tls-record-size-and-buffering-latency/
[tlsSizing2]: https://blog.cloudflare.com/optimizing-tls-over-tcp-to-reduce-latency/


## References


## original email for reference while writing

HTTP/1.1's HOL problem was at the application layer. You can't start sending response data for request 2 before the response to request 1 is fully downloaded (not even when using "pipelining")
HTTP/2 solves this application-layer HOL problem. This means that you now -can- start sending response data for request 2 before 1 is finished. 

This comes up in a few different ways:
- if request 1 becomes outdated you can simply stop sending its response and switch to only response 2 (e.g., this happens in video streaming, where old sections of the video might no longer be needed)
- if you want to send request 1 and 2 at the same time (multiplexing/bandwidth sharing. Mainly useful if the file can be used incrementally, like progressive jpegs/HTML/video. This is the typical example)
- If request 2 is of a higher priority than request 1 (in HTTP/1.1, to get around this, browsers often delay sendings requests until they can determine the optimal order by reading the HTML)
- If you want to use Server Push and the pushed stream is more important than the requesting stream
- (there are probably others)

Put differently, in HTTP/1.1, you always have to do:  111111222222, while in HTTP/2, you can do: 121212121212, 111222222, 222222111111, 221221221221, etc.
That is already a massive benefit to HTTP/1.1.


Now, you're correct in stating that this does not entirely solve all HOL problems, as they also occur on the TCP level. 
Because TCP doesn't know anything about HTTP/2, it just sees everything as a single bytestream, say: 12345678 (instead of HTTP/2, which knows it's really 11112222), and so TCP just knows that bytes have to be delivered in that exact order. 
That indeed also gives a HOL problem in the case of packet loss or heavy packet re-ordering: if say packet 4 is lost, but 5678 do arrive, they have to wait until the retransmission of 4 arrives. 
H2 knows that is not needed (as 5678 contain data for stream 2 and 4 is for stream 1 and they are independent) but TCP does not. 

Crucially though: that's only if you have packet loss. And while that -does- happen, it's not as much of a problem as you might think. 
Especially on wired connections, packet loss is typically extremely low (to the point that researchers sometimes have to manually drop packets when testing over the internet to observe this kind of behaviour). 
Another aspect is that packet loss is often bursty: it's not 1 in every 200 packets that is lost, but say groups of 10-20 every 2000-4000.
As such, TCP HOL blocking is real, but much less of an issue in practice and thus HTTP/2's solution at the application layer is still quite useful when compared to HTTP/1.1, as that has both the application- and transport-layer HOL problem *combined*. 

------------------

So, that's then one of the main reasons for doing QUIC: there, they took HTTP/2's concept of independent streams and brought it down to the transport layer. 
Now, QUIC also knows it's actually 11112222 and if the 4th packet is lost, it can still deliver packets 5 to 8 to for example the browser for processing (meaning image 1 might be stalled a bit, but image 2 keeps loading, which it wouldn't on TCP+HTTP/2).
HTTP/3 makes use of QUIC's streams, and so benefits from this as well.

Now, it's time to be critical about how much QUIC actually helps in practice though... As I said, it's only useful in situations with (a lot of) packet loss, which are uncommon.
A second problem is that for websites, most resources are NOT progressively load-able. For example JavaScript, CSS, PNGs, etc. all need to be downloaded fully to be usable. 
As such, you typically don't want to send data multiplexed like this: 11111111111222222222222222222111111111111111111111112222222222222222222221111111111111111111222222222222222222222, 
but rather sequentially like this: 1111111111111111111111111111111111111111111111111111111111111111222222222222222222222222222222222222222222222222222.

However, not all of those packets are typically "in flight" at the same time: due to congestion control, you'll first send only 10 packets, then 20, then 40, and then maybe go up linearly by 1 (41, 42, 43 packets).
So, when you're worried about web performance and loading resources sequentially (which e.g., Google Chrome does), you'd have say 20 packets on the wire, but they are all of resource 1... 
This means that if packet 11 of those 20 is lost, you'd still have HOL blocking for packet 12-20, because they are of the same stream as packet 11... 
(and per-stream, data of course still needs to be delivered in-order) basically undoing QUIC's "solution" to the HOL problem.

As such, QUIC will mainly help if in those 20 packets, you for example first have 10 packets of resource 1, and the next 10 of resource 2. Then, if a packet is lost, you can still make progress on the other stream. 
However, as I said, mixing stream data like that is typically worse for web performance anyway, so anything you gain from HOL removal might be undone by that fact alone.
This last aspect is however difficult to measure and estimate. We've done initial testing on this in our paper (see below), but it's not yet clear what the best approach is there.

-----------

Wew, what a wall of text :) I hope this makes things clearer. 
If you would like to know more about HOL blocking nuances, I have a paper that focuses on these aspects for QUIC and HTTP/3: https://h3.edm.uhasselt.be/files/ResourceMultiplexing_H2andH3_Marx2020.pdf
If you would like to know more about why sending resources sequentially is better for web performance, I have another YouTube talk: https://www.youtube.com/watch?v=nH4iRpFnf1c
