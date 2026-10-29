# Client, session and request

**Covers:** Client, Session, Request

## Three words behind every web call {#overview}
When a program talks to a web service, three ideas appear in nearly every library: who is asking, how long the conversation lasts, and what exactly is asked. Their names are client, session and request. In concurrent programs they matter for a practical reason: how you share them decides how fast and how safe your program is.

## Client {#client}
A client is the program, or the object in it, that sends requests and receives replies. The other side is the server. In code, a client object often holds settings that all calls share, such as the base address, the time limits and the login details.

## Session {#session}
A session is a client's long-lived conversation with a server. It keeps connections open, so that the next request does not need a new handshake, and it remembers things such as cookies and headers. Reusing one session for many requests is much faster than starting a new connection each time.

## Request {#request}
A request is one message sent to a server: the method (such as GET), the address, the headers and sometimes a body. The server answers with a response. A request is the smallest unit of a web conversation.

## A worked example {#example}
A request is just data until it is sent. Build one and inspect it, with no network involved:

```python
from urllib.request import Request

request = Request(
    "https://example.com/data",
    headers={"Accept": "application/json"},
    method="GET",
)
print(request.get_method())        # GET
print(request.full_url)            # https://example.com/data
print(request.get_header("Accept"))  # application/json
```

The request knows its method, its address and its headers. Nothing has been sent. A client would send it, and a session would reuse a connection to do so.

Why sessions matter for speed. Opening a connection to a secure server takes several round trips: the address lookup, the connection, and the security handshake. Suppose that costs 150 milliseconds, and the request itself takes 50.

- 20 requests with a new connection each: 20 × (150 + 50) = 4,000 milliseconds.
- 20 requests over one session: 150 + 20 × 50 = 1,150 milliseconds.

The session is almost 3.5 times faster, with the same requests. This is the first thing to fix in a slow client.

In a concurrent program, the rules for sharing are:

- Share one session between many tasks or threads, if the library says it is safe. Asyncio clients are meant to be shared across tasks. Some thread-based clients are not safe to share, and need one session per thread.
- Create a request for each call. Requests are cheap and should not be reused with changed content.
- Limit how many requests are in flight, with a semaphore, to respect the server's limits.

A sketch of the asyncio pattern:

```text
async with SomeClient() as client:        # one client, one session
    tasks = [client.get(url) for url in urls]
    replies = await asyncio.gather(*tasks)
```

One client serves all the requests, and `gather` runs them together. With 100 URLs and a limit of 10 at once, each taking 0.2 seconds, the total is about 10 rounds × 0.2 = 2 seconds, instead of 20 seconds one by one.

| Word | Meaning | Lifetime |
|---|---|---|
| Client | The program or object that asks | The whole program, usually |
| Session | A reused conversation | As long as it is useful |
| Request | One message | One call |

## A common mistake {#mistake}
Creating a new client or session for every request, which pays the connection cost every time. Another is sharing a session between threads when the library does not allow it.

## Key points {#key-points}
- A client sends requests. A session keeps connections open and remembers state.
- A request is one message to the server.
- Reuse one session for many requests: it is often several times faster.
- Limit how many requests are in flight.
