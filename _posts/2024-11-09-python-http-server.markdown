---
layout: post
title:  "HTTP Servers and Related Concepts" 
date:   2024-11-09 19:05:00 -0800
categories: jekyll update
---
I generally try to build mental models of the tech that I work with.
Some (non mutually exclusive) ways people build mental models:
- Read blogs or books, watch youtube videos
- Take a course
- Build an application/mini-project
- Code things up from scratch (.. _scratch_ is subjective here)

I am a big _believer_ in the power of implementing ideas to understand a concept. The simpler our abstractions, the better. I'm not claiming that I always do it. I _wish_ I could..

The idea is not to build something production-grade. The objective of this effort is to build an _understanding_. When you set yourself on a path towards understanding, don't let yourself get distracted by tangential optimizations. It's good to identify them, but note them down and move on. Don't fall into a rabbit hole, unless your objective explicitly defines it.

Coming back to the point..

## Let's build an HTTP server.

An HTTP server generally runs on a different machine on the network (internet/LAN). We, therefore, need the ability to send and receive bytes from/to the server over a network. We will use sockets as our underyling OS abstraction to accomplish our objective.

FYI: This maybe news to you, it definitely was to me when I first learnt about it.
HTTP is *just* text. Yep - so much talk about HTTP/protocols, networking, and it's actually *just* text.

A simple HTTP request looks like this
```
GET /about.html HTTP/1.0  <--  Specify the resource that we want aka we want to load the about.html page on this server.
User-Agent: Mozilla/5.0   <-- Headers start from the next line.
```

Sample code for a HTTP server:
{% highlight ruby %}

import socket

# A machine has an IP address that is it's unique address.
# Think of a port as a specific sub-address on that machine, where our server process is actively waiting for requests. 

SERVER_HOST = '127.0.0.1'
SERVER_PORT = 12345

# create a socket
server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM) # IPv4, TCP
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
# set socket options to allow port re-use upon start-stop of the server.
# You do not want to change the port in the code everytime you start the server.

# bind the socket to our predefined host and port
server_socket.bind((SERVER_HOST, SERVER_PORT)) # bind takes a single tuple argument

server_socket.listen(2) # 2 here is the backlog size.
# It's the answer to the question - "How many clients are we okay to put in a waiting state before we start rejecting requests?"
# What's our incoming client queue size? If there are more clients than size, we start refusing connections

print(f"Server is now listening on port {SERVER_PORT}")

while True:
    # Infinitely wait for clients to connect
    incoming_client_connection, client_address = server_socket.accept()

    # Manage client request
    request = incoming_client_connection.recv(1024).decode()
    print(request)

    # Send HTTP response
    response = "HTTP/1.0 200 OK\n\nHello World "
    incoming_client_connection.sendall(response.encode())
    incoming_client_connection.close()

server_socket.close()
{% endhighlight %}


#### Notes
- How are files served?
    The GET request will basically mention the name of the file (aka resource eg /index.html). Our server will open the file, retrieve all contents, and send them over to the client using an encoded string.
    The client will basically render the file (via the looong decoded string) and show the output to the user.

- Blocking vs. Non-blocking:
    By default, accept() is blocking, meaning it waits until a client connects.
    For non-blocking sockets, you can use `server_socket.setblocking(False)` or use `select()`/`poll()` for multiplexing.

- Performance:
    A higher backlog does not guarantee better performance; it's just a queue size. If your server cannot handle many connections quickly enough, consider optimizing the server (e.g., using threading, multiprocessing, or asynchronous programming with asyncio).


**Sockets**:
A socket is basically an abstraction provided by the operating system that enables you to, both, *send* and *receive* bytes through a network. 
Our above exercise to build a simple server, allows us to build a mental model of what HTTP is, and how a simple server works (and handles client requests).
When you went through the code, you saw that we used an abstraction (sockets) to build basically the whole server infrastructure.
And this would then beg the question, what is a socket? Why use it?

Sockets (among others) are a primary method of IPC (Inter process communication). They are also a primary method of IPC over network.
Note: Every socket internally is basically associated with a file descriptor. (File descriptors are the next level of abstraction.)

A simple mental model of the problem that's being solved here is - think of a server process on machine serverMachine, and a client process on clientMachine. We want to enable communication between them (across some network).

Quick step by step walkthrough:
- Server machine starts up, loads the server program, allocates the given port, creates a "socket" and starts _listening_.
- Think of a socket as a file on the file system, that can be read by and written to by the server program (process).
- Client machine boots up, loads the client program, allocates an ephemeral port (and a socket) on the client machine, and sends out a connection request to the server.
- When the server gets a connection request, there's the general TCP 3-way handshake, and the client is then said to have an active connection.
- For every successful connection on the server, the server creates a new socket (with a new file descriptor) that's bound to the same port.
- On the server, therefore, every client has a separate socket (and transitively a FD) associated with it, that the server uses for communication.

I implore you to read more about this. Ask yourself questions. Then go and find answers to these questions. That's the only way to build true understanding.