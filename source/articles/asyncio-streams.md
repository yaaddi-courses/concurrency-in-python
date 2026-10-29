# Streams: reading and writing network data

**Covers:** StreamReader, StreamWriter, open_connection(), start_server(), drain(), write(), readline(), read()

## Network programs in a few lines {#overview}
A server must handle many connections at once, each of them mostly waiting for data. That is exactly what asyncio is good at. The streams interface hides the low-level details, and gives you a reader and a writer for every connection.

## StreamReader {#streamreader}
A `StreamReader` is the object you read from. Its methods are awaitable: `await reader.read(n)` waits for up to n bytes, `await reader.readline()` waits for one line.

## StreamWriter {#streamwriter}
A `StreamWriter` is the object you write to. `writer.write(data)` puts bytes in a buffer and returns at once. `await writer.drain()` waits until the buffer has been sent. `writer.close()` ends the connection.

## open_connection() {#open-connection}
`asyncio.open_connection(host, port)` opens a connection to a server and returns a reader and a writer. It is the other side side.

## start_server() {#start-server}
`asyncio.start_server(callback, host, port)` starts a server. For every new connection, it calls your callback with a reader and a writer. The callback is a coroutine, so thousands of connections can be handled at once.

## write() {#write}
`write()` adds data to the writer's buffer, or writes to a file. On a stream writer, it does not wait, so the data may not have left yet.

## drain() {#drain}
`drain()` waits until the write buffer of the stream writer has been flushed. Awaiting it after writing keeps the memory use low when the other side is slow, since it pauses a fast writer.

## readline() {#readline}
`readline()` waits for one full line, up to and including the newline character. Once the stream is over, it gives back empty `bytes`, which is the signal to stop looping.

## read() {#read}
`read(n)` reads up to n bytes from a stream. With no argument, it reads until the end of the stream.

## A worked example {#example}
A small echo server and a connecting program, inside one program:

```python
import asyncio

async def handle(reader, writer):
    line = await reader.readline()
    writer.write(line.upper())
    await writer.drain()
    writer.close()

async def main():
    server = await asyncio.start_server(handle, "127.0.0.1", 0)
    port = server.sockets[0].getsockname()[1]
    reader, writer = await asyncio.open_connection("127.0.0.1", port)
    writer.write(b"hello\n")
    await writer.drain()
    reply = await reader.readline()
    print(reply)  # b'HELLO\n'
    writer.close()
    server.close()
    await server.wait_closed()

asyncio.run(main())
```

Port 0 asks the system to pick a free port, which the program reads from the server socket. The other side sent "hello", the server made it uppercase and sent it back. Both sides awaited at each step, so neither blocked the loop. The data is `bytes`, not text, so `b"hello\n"` and a `b'HELLO\n'` reply.

Reading until the end:

```python
import asyncio

async def main():
    reader = asyncio.StreamReader()
    reader.feed_data(b"abc")
    reader.feed_eof()
    print(await reader.read(2))   # b'ab'
    print(await reader.read())    # b'c'
    print(await reader.readline())  # b''

asyncio.run(main())
```

`read(2)` returned two bytes. `read()` returned the rest. After the end, `readline()` returned empty bytes, which is how the end of the stream shows.

Why `drain()` matters. If a server writes a 10 MB reply to a connecting program that reads slowly, without `drain()` the whole reply sits in memory. With 1,000 such clients that is 10 GB. Awaiting `drain()` makes each writer pause until the other side catches up, so the memory use stays small.

Count the connections. A thread per connection needs about 8 MB of stack in the worst case, so 10,000 connections need tens of gigabytes. Tasks use a few kilobytes each, so 10,000 connections use some tens of megabytes. That is why asyncio suits servers with many idle connections.

| Call | Waits? | Purpose |
|---|---|---|
| `write(data)` | No | Buffer bytes |
| `await drain()` | Yes | Wait for the buffer to empty |
| `await readline()` | Yes | One line |
| `await read(n)` | Yes | Up to n bytes |

## A common mistake {#mistake}
Writing without ever awaiting `drain()`, so memory grows with a slow reader. Another is mixing up bytes and text: streams carry bytes, so encode and decode explicitly.

## Key points {#key-points}
- A reader and a writer represent each connection. `start_server` and `open_connection` make them.
- `write()` buffers; `await drain()` waits for it to be sent.
- Streams carry bytes. An empty result means the end.
- Tasks make thousands of connections cheap.
