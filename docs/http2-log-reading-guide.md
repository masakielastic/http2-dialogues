# Understanding HTTP/2 by Reading Real Tool Logs

## Introduction

HTTP/1 is relatively easy to inspect because the protocol is text-based. You can often look at a request like `GET / HTTP/1.1` and immediately understand what is happening on the wire. HTTP/2 is different: it is a binary framing protocol, so the interesting parts are no longer visible as plain text. That is why learning HTTP/2 usually requires a different workflow: observe logs, interpret what they mean, connect them to the protocol model, and then map that model to library APIs.

## Dialogue

**Learner**

I could read a lot of HTTP/1 traffic just by looking at raw requests and responses. Why does HTTP/2 feel so much harder to see?

**Engineer**

Because HTTP/1 exposes most of its structure directly in text. A request looks like this:

```
GET / HTTP/1.1  
Host: example.com  
User-Agent: curl/8.x  
Accept: \*/\*
```

You can inspect that with `telnet`, `nc`, packet captures, proxy tools, or application logs.

HTTP/2 does not work like that. It sends binary frames over a single connection. So even though the application is still doing familiar HTTP things such as sending requests, headers, and bodies, the transport format is no longer human-readable.

That is why tool logs matter. If you want to learn HTTP/2, the practical path is:

```
logs -> interpretation -> protocol understanding -> library design
```

Three tools are especially useful here:

```
curl \-v \--http2 https://example.com  
nghttp \-nv https://example.com  
nghttpd 8443 server.key server.crt
```

**Architect**

HTTP/2 moved complexity away from the text syntax of HTTP/1 and into a structured binary protocol. That design enables multiplexing, flow control, header compression, and better connection reuse, but it also means the protocol is less directly observable.

So the learning problem changes.

With HTTP/1, you can often ask, “What text was sent?”

With HTTP/2, you ask, “What frames were exchanged, on which stream, in what order, and why?”

That is why logs are not just debugging output. For HTTP/2, logs are one of the best teaching surfaces.

---

**Learner**

You mentioned `nghttp` and `nghttpd`. What exactly are they, and why are they so useful for learning?

**Engineer**

They come from the `nghttp2` project, which is one of the best-known open-source implementations of HTTP/2.

The project includes:

- `nghttp`: an HTTP/2 client
    
- `nghttpd`: an HTTP/2 server
    
- `nghttp2`: the underlying library
    

That already makes them useful, but there is an additional reason they are especially good learning tools: the author of `nghttp` and `nghttpd` is also the author of the `nghttp2` library itself.

So these tools are not accidental wrappers around the library. They reflect the mental model of the library author and expose the protocol in a very direct way.

For learning and debugging, that is extremely valuable.

**Architect**

This matters because observability tools are best when they are close to the implementation model.

If the same project provides both the library and the tools, the logs are often aligned with the internal protocol representation. That makes it easier to move from “I saw this line in a log” to “I now understand what the protocol stack is doing.”

In other words:

- `curl` is useful for seeing negotiation and general client behavior
    
- `nghttp` is useful for seeing HTTP/2 frame activity explicitly
    
- `nghttpd` is useful for seeing the server side of the same exchange
    

That combination gives you multiple viewing angles on the same protocol.

---

**Learner**

Before reading logs, I want a simple mental model. What is the basic structure of HTTP/2?

**Engineer**

A good starting point is this:

```
Connection  
  ├ Stream  
  │   ├ HEADERS  
  │   └ DATA  
  ├ Stream  
  │   ├ HEADERS  
  │   └ DATA  
  └ Stream  
      ├ HEADERS  
      └ DATA
```

There are three core layers to keep in mind.

**Connection**

This is the underlying TCP connection, usually protected by TLS. HTTP/2 runs over that single connection.

**Stream**

A stream is a logical HTTP exchange inside that connection. One request/response pair usually maps to one stream.

**Frame**

A frame is the smallest protocol unit actually sent on the wire. HEADERS, DATA, and SETTINGS are all frame types.

So when you read logs, you are usually reading frame-level observations. From those frames, you reconstruct stream-level behavior. And from the set of streams, you understand what the connection is doing.

**Architect**

This layered view is critical.

An HTTP/1 mindset often assumes:

```
one request <-> one connection
```

HTTP/2 breaks that expectation. It is closer to:

```
one connection <-> many concurrent streams  
one stream <-> one logical exchange  
one exchange <-> many frames
```

If you do not separate those layers, logs will feel chaotic. You will see multiple frame types interleaved and assume the system is disorganized. In reality, the protocol is highly structured. The structure is just no longer text-first.

---

**Learner**

Let’s start with `curl`. What can I actually learn from `curl -v --http2`?

**Engineer**

A typical command looks like this:

```
curl \-v \--http2 https://example.com
```

A typical snippet might look like this:


```
\* ALPN, offering h2  
\* ALPN, offering http/1.1  
\* TLSv1.3 handshake completed  
\* ALPN, server accepted h2  
\* Using HTTP2  
\> GET / HTTP/2  
\> Host: example.com  
\> user-agent: curl/8.x  
\> accept: \*/\*
```

From this, you can learn several important things.

First, **ALPN negotiation**.

```
\* ALPN, offering h2  
\* ALPN, server accepted h2
```

This means the client offered HTTP/2 during the TLS handshake, and the server agreed.

Second, **TLS matters**.

HTTP/2 over HTTPS is typically negotiated during TLS setup, not after the request has already started.

Third, **protocol selection**.

```
\* Using HTTP2
```

This confirms that the actual application layer is now HTTP/2.

What `curl -v` does well is show the transition into HTTP/2. It is excellent for confirming:

- whether the server supports HTTP/2
    
- whether ALPN negotiation succeeded
    
- whether the client is really using HTTP/2
    

But `curl` is not mainly a frame inspection tool. It tells you the protocol choice and some high-level behavior, not the full frame-by-frame story.

**Architect**

That distinction is important.

`curl` is primarily an HTTP client tool. Its logs are oriented toward client behavior and troubleshooting, not protocol pedagogy.

So when you read `curl -v`, treat it as the connection and negotiation view:

```
transport setup -> TLS -> ALPN -> protocol selected
```

Then move to `nghttp` when you want to observe the frame machinery itself.

---

**Learner**

So `curl` helps me see how HTTP/2 starts, but not really how the frames behave. Is that where `nghttp` becomes more useful?

**Engineer**

Exactly.

Run:

```
nghttp \-nv https://example.com
```

A typical simplified log might look like this:

```
\[  0.020\] send HEADERS frame <length=36, flags=0x25, stream\_id=1>  
          ; END\_STREAM | END\_HEADERS | PRIORITY  
\[  0.021\] recv SETTINGS frame <length=18, flags=0x00, stream\_id=0>  
\[  0.021\] recv WINDOW\_UPDATE frame <length=4, flags=0x00, stream\_id=0>  
\[  0.022\] recv HEADERS frame <length=120, flags=0x04, stream\_id=1>  
          ; END\_HEADERS  
\[  0.022\] recv DATA frame <length=512, flags=0x00, stream\_id=1>  
\[  0.023\] recv DATA frame <length=128, flags=0x01, stream\_id=1>  
          ; END\_STREAM
```

This is much closer to the protocol.

Here are the big pieces.

### HEADERS frames

These carry HTTP header fields, such as method, path, status, and response headers.

Example:

```
send HEADERS frame ... stream\_id=1  
recv HEADERS frame ... stream\_id=1
```

That usually means:

- client sent request headers on stream 1
    
- server responded with response headers on stream 1
    

### DATA frames

These carry the message body.

Example:

```
recv DATA frame ... stream\_id=1  
recv DATA frame ... stream\_id=1 ; END\_STREAM
```

That means the response body arrived in one or more chunks, and the final frame marked the end of the stream.

### SETTINGS frames

These configure connection-level behavior.

Example:

```
recv SETTINGS frame ... stream\_id=0
```

Notice `stream_id=0`. That is because SETTINGS is a connection-level frame, not a stream-specific one.

This is one of the first important log-reading habits in HTTP/2:

- some frames belong to a specific stream
    
- some frames belong to the whole connection
    

**Architect**

This is where the learning becomes architectural.

HTTP/2 is not “a request with some headers and some bytes.” It is “a connection that hosts a structured exchange of framed messages.”

That changes how you think about APIs too.

A library that exposes HTTP/2 naturally tends to distinguish between:

- connection-level activity
    
- stream-level activity
    
- frame-level activity
    

The logs are your first introduction to that layered model.

---

**Learner**

Can you show me how to interpret a frame sequence as an actual request/response?

**Engineer**

Yes. Start with a minimal sequence:

```
send HEADERS stream=1  
recv HEADERS stream=1  
recv DATA stream=1  
recv DATA stream=1 END\_STREAM
```

Read it like this:

1. The client opens stream 1 and sends request headers.
    
2. The server replies on the same stream with response headers.
    
3. The server sends part of the response body as DATA.
    
4. The server sends the final DATA frame with `END_STREAM`.
    

That corresponds to one logical HTTP exchange.

You can rewrite that mentally as:

```
Stream 1:  
  request headers sent  
  response headers received  
  response body received  
  response finished
```

The important point is that the wire-level units are frames, but the application-level story is still a request/response conversation.

**Architect**

This is the core interpretive skill.

A beginner sees:

```
HEADERS  
DATA  
DATA  
END\_STREAM
```

An experienced reader sees:

```
one logical HTTP transaction progressed and completed
```

That is the bridge from protocol syntax to protocol meaning.

When teaching HTTP/2, do not stop at naming frame types. Show how frame sequences form higher-level events in the lifecycle of a stream.

---

**Learner**

What about multiplexing? That is one of the features people always mention, but I still do not feel it intuitively.

**Engineer**

Logs make multiplexing much easier to see.

Look at this:

```
send HEADERS stream=1  
send HEADERS stream=3  
recv HEADERS stream=1  
recv DATA stream=1  
recv HEADERS stream=3  
recv DATA stream=3  
recv DATA stream=1 END\_STREAM  
recv DATA stream=3 END\_STREAM
```

This means two streams are active on the same connection:

- stream 1
    
- stream 3
    

The server does not have to finish stream 1 before working on stream 3. Frames from different streams can be interleaved.

A more compact example is:

```
HEADERS stream=1  
HEADERS stream=3  
DATA stream=1  
DATA stream=3
```

This is the key observation:

```
one connection  
multiple concurrent streams  
interleaved frames
```

That is multiplexing.

```
In HTTP/1.1, parallelism often required multiple connections or pipelining with practical limitations. In HTTP/2, interleaving happens at the frame layer inside one connection.
```

**Architect**

Multiplexing is easier to understand when you stop imagining “a connection carries one request” and instead imagine “a connection is a transport container for many stream state machines.”

Each stream advances independently, but they share the same physical connection.

That is why stream IDs matter so much in logs. They are the label that lets you reconstruct each logical conversation out of an interleaved frame sequence.

When reading HTTP/2 logs, one of the best habits is:

```
group by stream\_id first  
then read frame order
```

Without that, multiplexed traffic looks noisy. With it, the design becomes legible.

---

**Learner**

So far we have looked from the client side. What changes when I use `nghttpd` and look from the server side?

**Engineer**

`nghttpd` gives you the opposite perspective.

A basic command looks like this:

```
nghttpd 8443 server.key server.crt
```

A simplified server-side log might look like this:

```
\[id=1\] recv HEADERS frame <stream\_id=1>  
\[id=1\] recv DATA frame <stream\_id=1>  
\[id=1\] send HEADERS frame <stream\_id=1>  
\[id=1\] send DATA frame <stream\_id=1>
```

Now the same exchange is being observed by the server.

From the client side, you might have seen:

```
send HEADERS stream=1  
recv HEADERS stream=1  
recv DATA stream=1
```

From the server side, the same activity becomes:

```
recv HEADERS stream=1  
send HEADERS stream=1  
send DATA stream=1
```

This is useful because it confirms that the protocol is symmetric at the frame-observation level, even though client and server roles differ.

It also helps when debugging. If the client says it sent something but the server never logs it, you know where to investigate next.

**Architect**

The server perspective teaches an important architectural lesson: protocol events are role-relative.

The same exchange can be described in two equally valid ways:

- client-centric: “I sent request headers”
    
- server-centric: “I received request headers”
    

That is why protocol libraries often separate transport observation from application semantics. A reusable HTTP/2 library must represent what happened in a way that works for either side of the connection.

---

**Learner**

I think I understand frames better now. But how do these logs connect to library APIs?

**Engineer**

This is where frame logs become software architecture.

Suppose your logs look like this:

```
recv HEADERS  
recv DATA  
stream closed
```

A library usually does not want application developers to parse raw frame logs manually. Instead, it often translates those protocol details into higher-level events such as:

```
HeadersReceived  
DataReceived  
StreamClosed
```

That is a more ergonomic API surface.

For example, the raw protocol may contain:

```
recv HEADERS frame <stream\_id=1>  
recv DATA frame <stream\_id=1>  
recv DATA frame <stream\_id=1 END\_STREAM>
```

A library might expose that as:

```
onHeaders(stream=1, headers=...)  
onData(stream=1, chunk=...)  
onData(stream=1, chunk=..., endStream=true)  
onStreamClosed(stream=1)
```

Or as objects:

```
HeadersReceived(streamId=1, headers=...)  
DataReceived(streamId=1, data=...)  
StreamClosed(streamId=1)
```

This is a key design idea: the library converts frame sequences into a higher-level event model.

**Architect**

And that conversion is not just convenience. It is architecture.

Most application developers do not want to think in raw protocol frames all the time. They want a model closer to the lifecycle of a request/response exchange.

So HTTP/2 libraries often define an event-driven API that:

- preserves protocol meaning
    
- hides unnecessary wire-level detail
    
- separates connection events from stream events
    
- gives applications a stable abstraction boundary
    

You can think of it like this:


```
wire protocol frames  
    ↓  
protocol state transitions  
    ↓  
library events  
    ↓  
application callbacks or handlers
```

That is why reading logs is such a strong learning method. The logs expose the lower layer. Once you understand them, higher-level APIs become much easier to design and evaluate.

---

**Learner**

Does that mean I should always think in events once I move from tools to library design?

**Engineer**

Not always, but it is usually a good mental shift.

When you use tools like `nghttp`, you are close to the wire and think in frames:

```
HEADERS  
DATA  
SETTINGS  
WINDOW\_UPDATE  
RST\_STREAM
```

When you design or use a library, you often think in lifecycle events:

```
request started  
headers received  
data received  
stream ended  
stream reset  
connection closed
```

Both views matter.

If you only know the event API, you may not understand why a bug happens.

If you only know raw frames, you may design an application API that is too low-level for normal use.

So the best learning path is to move between the two.

**Architect**

Exactly. Good protocol education should not stop at “here is the spec” or “here is the API.”

It should build a translation skill:

- from raw observations to protocol meaning
    
- from protocol meaning to software abstractions
    

That is the reason this learning path works so well:

```
logs -> interpretation -> protocol understanding -> library design
```

It trains the reader to see HTTP/2 not just as bytes on the wire, but as a system with layered representations.

## Concept Summary

HTTP/2 is harder to observe than HTTP/1 because it is a binary framing protocol rather than a text-based one. That means a reader cannot rely on raw packet text or simple request dumps to understand what is happening. Instead, practical learning depends on observability tools such as `curl`, `nghttp`, and `nghttpd`.

The most important conceptual structure is the relationship between connection, stream, and frame. The connection is the underlying transport channel. A stream is a logical HTTP exchange within that connection. A frame is the smallest protocol unit that actually travels over the wire. When reading logs, you usually observe frames directly and reconstruct stream behavior from them.

`curl -v --http2` is useful for understanding negotiation and protocol selection, especially ALPN and TLS-driven HTTP/2 activation. `nghttp -nv` is more useful for understanding actual frame activity, because it shows HEADERS, DATA, SETTINGS, and other frame types more directly. `nghttpd` complements that by showing the server-side view of the same exchange.

Frame logs become especially valuable when learning multiplexing. In HTTP/2, multiple streams share one connection, and their frames can be interleaved. This is one of the major conceptual differences from HTTP/1. When logs look confusing, grouping lines by stream ID is often the fastest way to recover the logical structure.

Finally, HTTP/2 libraries usually do not expose raw frame handling directly to application code. Instead, they convert protocol activity into higher-level events such as `HeadersReceived`, `DataReceived`, and `StreamClosed`. That is why log reading is not merely a debugging technique. It is also a path toward understanding how HTTP/2 libraries are designed.

## Key Takeaways

- HTTP/2 is not easily inspected as plain text; logs are one of the best ways to learn it.
    
- The key structure is **connection -> stream -> frame**.
    
- `curl` is useful for seeing negotiation; `nghttp` and `nghttpd` are better for frame-level observation.
    
- Frames are the wire-level units, but streams are the logical unit of an HTTP exchange.
    
- Multiplexing means multiple streams can share one connection and interleave their frames.
    
- Library APIs often translate frame sequences into higher-level events.
    

## Further Exploration

Readers who want to go deeper may explore:

- HTTP/2 frame types such as `RST_STREAM`, `PING`, and `GOAWAY`
    
- HPACK and how header compression affects what logs show
    
- flow control and `WINDOW_UPDATE`
    
- stream state transitions
    
- event-driven API design in HTTP/2 libraries such as `nghttp2`