# Transports, protocols and signals

**Covers:** Transport, Protocol, connection_made(), data_received(), add_signal_handler(), SIGINT, SIGTERM, SIGKILL

## The low-level layer under streams {#overview}
Streams are built on a lower layer made of transports and protocols. It is callback-based, with no `await` in your code. Most programs use streams, but knowing the layer explains how asyncio works, and some libraries expose it. The same chapter also covers how a program should react to signals from the operating system.

## Transport {#transport}
A Transport is the object that moves bytes for a connection: it does the actual sending and receiving on the socket. You do not read from a transport. You give it data with `transport.write()` and it reports events to a protocol.

## Protocol {#protocol}
A Protocol is the lower-level, callback-based counterpart to streams. Instead of `async def` and `await`, it is a class whose methods the loop calls when things happen: a connection opens, data arrives, the connection closes. You write the reactions.

## connection_made() {#connection-made}
`connection_made(transport)` is the protocol method that the loop calls when a connection is established. It receives the transport, which you usually store for later writing.

## data_received() {#data-received}
`data_received(data)` is the protocol method that the loop calls each time bytes arrive. The bytes may be a part of a message, or several messages together, so a protocol must handle both.

## add_signal_handler() {#add-signal-handler}
`loop.add_signal_handler(signal, callback)` registers a callback on the event loop for an operating-system signal. It is the safe way to react to signals in asyncio programs on Unix systems, because the callback runs inside the loop, where it can touch tasks.

## SIGINT {#sigint}
`SIGINT` is the signal sent when the user presses Ctrl+C. By default, Python turns it into a `KeyboardInterrupt`.

## SIGTERM {#sigterm}
`SIGTERM` is the polite request to end, sent by `kill`, by process managers and by container platforms when they stop a service. A program should catch it and shut down cleanly.

## SIGKILL {#sigkill}
`SIGKILL` ends a process at once. It cannot be caught, blocked or ignored, so the program gets no chance to clean up. It is the last resort when SIGTERM is ignored.

## A worked example {#example}
A protocol that echoes what it receives.

```python
import asyncio

class Echo(asyncio.Protocol):
    def connection_made(self, transport):
        self.transport = transport

    def data_received(self, data):
        self.transport.write(data.upper())
        self.transport.close()

async def main():
    loop = asyncio.get_running_loop()
    server = await loop.create_server(Echo, "127.0.0.1", 0)
    port = server.sockets[0].getsockname()[1]
    reader, writer = await asyncio.open_connection("127.0.0.1", port)
    writer.write(b"ping")
    await writer.drain()
    print(await reader.read())  # b'PING'
    writer.close()
    server.close()
    await server.wait_closed()

asyncio.run(main())
```

The loop created an `Echo` object for the connection and called `connection_made` with the transport. When the client sent "ping", the loop called `data_received`, which wrote the uppercase reply and closed. No `await` appears inside the protocol, because it is all callbacks.

A common care: data may arrive in pieces. If a client sends "hel" and then "lo", `data_received` is called twice. A protocol that expects a full message in one call fails then, so it must collect pieces in a buffer and look for the end of a message.

Signals and a clean shutdown. This part needs Unix, and shows the pattern:

```text
import asyncio, signal

async def main():
    loop = asyncio.get_running_loop()
    stop = asyncio.Event()
    loop.add_signal_handler(signal.SIGTERM, stop.set)
    loop.add_signal_handler(signal.SIGINT, stop.set)
    await stop.wait()
    print("shutting down cleanly")

asyncio.run(main())
```

Both signals set an event. The main coroutine waits for it, and then runs its clean-up. Ctrl+C or a `kill` from a process manager lead to the same orderly stop. SIGKILL cannot be handled, so nothing runs: that is why a service should respond to SIGTERM promptly, or the manager will send SIGKILL after its grace period, often 10 or 30 seconds.

| Signal | Meaning | Catchable? |
|---|---|---|
| `SIGINT` | Ctrl+C | Yes |
| `SIGTERM` | Please stop | Yes |
| `SIGKILL` | Stop now | No |

Protocols against streams: streams are easier to write, as straight code with `await`. Protocols have slightly less overhead per message, which matters only for very high rates, such as hundreds of thousands of small messages a second.

## A common mistake {#mistake}
Assuming one `data_received` call holds one whole message. Buffer and split it yourself. Another is handling SIGTERM by exiting abruptly, instead of finishing the work in progress.

## Key points {#key-points}
- A transport moves bytes. A protocol reacts with callbacks such as `connection_made` and `data_received`.
- Data may arrive in pieces, so buffer it.
- `add_signal_handler` lets a loop react to SIGINT and SIGTERM. SIGKILL cannot be caught.
- Handle SIGTERM by finishing work and exiting.
