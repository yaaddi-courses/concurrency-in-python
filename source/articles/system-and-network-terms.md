# System and network terms used in concurrency

**Covers:** RAM, API, HTTP, URLs, TCP, DB, GUI, EOF

## Words the rest of the course takes for granted {#overview}
Concurrency is about waiting on things: memory, networks, databases, users. The examples in this course use a handful of short terms for them. This article explains each one so that the later articles can use it without stopping.

## RAM {#ram}
RAM, random-access memory, is the fast working memory in which a computer keeps running programs and their data. When a program holds a large list, it lives in RAM. Several processes all use the same RAM, so ten workers that each load a 1 GB file take 10 GB, unless they share it.

## API {#api}
An API, application programming interface, is a defined way for one program to ask another for something. A web service's API says which requests it accepts and what it answers. Calling an API over the network is a typical I/O-bound job, because most of the time is spent waiting for the answer.

## HTTP {#http}
HTTP, HyperText Transfer Protocol, is the protocol that web browsers and servers use to exchange pages and data. Most web APIs speak it. A request names an address and a method such as GET, and the reply carries a status code, 200 for success.

## URLs {#urls}
URLs, uniform resource locators, are the web addresses of pages and services, such as `https://example.com/data`. A concurrent program often has a list of URLs to fetch, and the point of concurrency is to fetch many of them at the same time.

## TCP {#tcp}
TCP, the Transmission Control Protocol, is the reliable connection underneath most of the internet, including HTTP. It delivers bytes in order and resends lost ones. A TCP connection must be opened and closed, and waiting for it is where time goes.

## DB {#db}
A DB, short for database, is a program that stores data and answers queries about it. A query to a database server is another waiting job: the program sends it and waits for rows to come back. Opening many connections at once can overload the server, which is why a limit is often needed.

## GUI {#gui}
A GUI, graphical user interface, is a program's windows, buttons and menus. A GUI runs an event loop that must stay free to react to clicks. If long work runs in that loop, the window freezes, so slow work is moved to a thread or process.

## EOF {#eof}
EOF, end of file, is the signal that a stream has no more data to read. Reading from a closed connection or a finished file ends with EOF, as an empty result or an `EOFError`.

## A worked example {#example}
Reading a web address into its parts, and spotting the end of data. This needs no network:

```python
from urllib.parse import urlparse
import io

url = "https://example.com:8080/data?id=7"
parts = urlparse(url)
print(parts.scheme)    # https
print(parts.hostname)  # example.com
print(parts.port)      # 8080
print(parts.path)      # /data

stream = io.StringIO("line one\nline two\n")
print(repr(stream.readline()))  # 'line one\n'
print(repr(stream.readline()))  # 'line two\n'
print(repr(stream.readline()))  # ''
```

The URL splits into the scheme (`https`, which means HTTP over a secure connection), the host, the port and the path. The third `readline()` returns an empty string. That is how EOF looks when reading text: nothing is left.

Why these terms matter for speed. Suppose a program fetches 100 URLs, and each API call takes 0.5 seconds, nearly all of it waiting for the network. One after another: 100 × 0.5 = 50 seconds. With 20 at the same time: 100 / 20 = 5 batches of 0.5 seconds, so about 2.5 seconds. Memory grows only slightly, since the waiting calls hold little RAM.

Now a limit. The same program talks to a DB that allows 10 connections. Twenty simultaneous queries would be refused, so the program has to cap itself at 10, with a semaphore. The speed-up is then 10 times, not 20.

| Term | Plain meaning | Where it costs time |
|---|---|---|
| RAM | Working memory | Too much use slows or crashes |
| API, HTTP, URL | Asking a web service | Waiting for the network |
| TCP | Reliable connection | Opening and closing |
| DB | Stored data | Waiting for query answers |
| GUI | Windows and buttons | Must not be blocked |
| EOF | No more data | Marks the end of a stream |

## A common mistake {#mistake}
Opening more connections than the other side accepts, so the extra requests fail. Read the limit and respect it. Another is running slow work inside a GUI's event loop and freezing the window.

## Key points {#key-points}
- RAM is shared by all workers, so many large workers can run out of it.
- API, HTTP, URL, TCP and DB calls are mostly waiting, so concurrency helps them.
- A GUI must stay responsive: move slow work out of its loop.
- EOF means a stream has ended.
