==========================================================
ASGI (Asynchronous Server Gateway Interface) Specification
==========================================================

**Version**: 2.0 (2017-11-28)

Abstract
========

This document proposes a standard interface between network protocol
servers (particularly web servers) and Python applications, intended
to allow handling of multiple common protocol styles (including HTTP, HTTP/2,
and WebSocket).

This base specification is intended to fix in place the set of APIs by which
these servers interact and run application code;
each supported protocol (such as HTTP) has a sub-specification that outlines
how to encode and decode that protocol into messages.


Rationale
=========

The WSGI specification has worked well since it was introduced, and
allowed for great flexibility in Python framework and web server choice.
However, its design is irrevocably tied to the HTTP-style
request/response cycle, and more and more protocols are becoming a
standard part of web programming that do not follow this pattern
(most notably, WebSocket).

ASGI attempts to preserve a simple application interface, but provide
an abstraction that allows for data to be sent and received at any time,
and from different application threads or processes.

It also take the principle of turning protocols into Python-compatible,
asynchronous-friendly sets of messages and generalises it into two parts;
a standardised interface for communication and to build servers around (this
document), and a set of standard message formats for each protocol.

Its primary goal is to provide a way to write HTTP/2 and WebSocket code,
alongside normal HTTP handling code, however, and part of this design is
ensuring there is an easy path to use both existing WSGI servers and
applications, as a large majority of Python web usage relies on WSGI and
providing an easy path forwards is critical to adoption. Details on that
interoperability are covered in the ASGI-HTTP spec.


Overview
========

ASGI consists of two different components:

- A *protocol server*, which terminates sockets and translates them into
  connections and per-connection event messages.

- An *application*, which lives inside a *protocol server*, is instantiated
  once per connection, and handles event messages as they happen.

Like WSGI, the server hosts the application inside it, and dispatches incoming
requests to it in a standardized format. Unlike WSGI, however, applications
are instantiated objects that are fed events rather than simple callables,
and must run as ``asyncio``-compatible coroutines (on the main thread;
they are free to use threading or other processes if they need synchronous code).

Unlike WSGI, there are two separate parts to an ASGI connection:

- A *connection scope*, which represents a protocol connection to a user and
  survives until the connection closes.

- *Events*, which are sent to the application as things happen on the
  connection.

Applications are instantiated with a connection scope, and then run in an
event loop where they are expected to handle events and send data back to the
client.

Each application instance maps to a single incoming "socket" or connection, and
is expected to last the lifetime of that connection plus a little longer if
there is cleanup to do. Some protocols may not use traditional sockets; ASGI
specifications for those protocols are expected to define what the scope
(instance) lifetime is and when it gets shut down.


Specification Details
=====================

Connection Scope
----------------

Every connection by a user to an ASGI application results in an instance of
that application being created for the connection. How long this lives, and
what information it gets given upon creation, is called the *connection scope*.

For example, under HTTP the connection scope lasts just one request, but it
contains most of the request data (apart from the HTTP request body, as this
is streamed in via events).

Under WebSocket, though, the connection scope lasts for as long as the socket
is connected. The scope contains information like the WebSocket's path, but
details like incoming messages come through as Events instead.

Some protocols may give you a connection scope with very limited information up
front because they encapsulate something like a handshake. Each protocol
definition must contain information about how long its connection scope lasts,
and what information you will get inside it.

Applications **cannot** communicate with the client when they are
initialized and given their connection scope; they must wait until their
event loop is entered, and depending on the protocol spec, may have to
wait for an initial opening message.


Events
------

ASGI decomposes protocols into a series of *events* that an application must
react to. For HTTP, this is as simple as two events in order - ``http.request``
and ``http.disconnect``. For something like a WebSocket, it could be more like
``websocket.connect``, ``websocket.send``, ``websocket.receive``,
``websocket.disconnect``.

Each event is a ``dict`` with a top-level ``type`` key that contains a
unicode string of the message type. Users are free to invent their own message
types and send them between application instances for high-level events - for
example, a chat application might send chat messages with a user type of
``mychat.message``. It is expected that applications would be able to handle
a mixed set of events, some sourced from the incoming client connection and
some from other parts of the application.

Because these messages could be sent over a network, they need to be
serializable, and so they are only allowed to contain the following types:

* Byte strings
* Unicode strings
* Integers (within the signed 64 bit range)
* Floating point numbers (within the IEEE 754 double precision range, no ``Nan`` or infinities)
* Lists (tuples should be encoded as lists)
* Dicts (keys must be unicode strings)
* Booleans
* ``None``


Applications
------------

ASGI applications are defined as a callable::

    application(scope)

* ``scope``: The Connection Scope, a dictionary that contains at least a
  ``type`` key specifying the protocol that is incoming.

This first callable is called whenever a new connection comes in to the
protocol server, and creates a new *instance* of the application per
connection (the instance is the object that this first callable returns).

This callable is synchronous, and must not contain blocking calls (it's
recommended that all it does is store the scope). If you need to do
blocking work, you must do it at the start of the next callable, before you
application awaits incoming events.

It must return another, awaitable callable::

    coroutine application_instance(receive, send)

* ``receive``, an awaitable callable that will yield a new event dict when
  one is available
* ``send``, an awaitable callable taking a single event dict as a positional
  argument that will return once the send has been completed

This design is perhaps more easily recognised as one of its possible
implementations, as a class::

    class Application:

        def __init__(self, scope):
            self.scope = scope

        async def __call__(self, receive, send):
            ...

The application interface is specified as the more generic case of two callables
to allow more flexibility for things like factory functions or type-based
dispatchers.

Both the ``scope`` and the format of the messages you send and receive are
defined by one of the application protocols. ``scope`` must be a ``dict``.
The key ``scope["type"]`` will always be present, and can be used to work
out which protocol is incoming.

The protocol-specific sub-specifications cover these scope
and message formats. They are equivalent to the specification for keys in the
``environ`` dict for WSGI.


Protocol Specfications
----------------------

These describe the standardized scope and message formats for various protocols.

The one common key across all scopes and messages is ``type``, a way to indicate
what type of scope or message is being received.

In scopes, the ``type`` key must be a unicode string, like ``"http"`` or
``"websocket"``, as defined in the relevant protocol specification.

In messages, the ``type`` should be namespaced as ``protocol.message_type``,
where the ``protocol`` matches the scope type, and ``message_type`` is
defined by the protocol spec. Examples of a message ``type`` value include
``http.request`` and ``websocket.send``.

Current protocol specifications:

* `HTTP and WebSocket <https://github.com/django/asgiref/blob/master/specs/www.rst>`_


Middleware
----------

It is possible to have ASGI "middleware" - code that plays the role of both
server and application, taking in a scope and the send/receive awaitables,
potentially modifying them, and then calling an inner application.

When middleware is modifying the scope, it should make a copy of the scope
object before mutating it and passing it to the inner application, as otherwise
changes may leak upstream. In particular, you should not assume that the copy
of the scope you pass down to the application is the one that it ends up using,
as there may be other middleware in the way; thus, do not keep a reference to
it and try to mutate it outside of the initial ASGI constructor callable that
gets passed ``scope``.

It's notable that the part of ASGI applications that gets to modify the
``scope`` runs synchronously, as it's designed to be compatible with Python
class constructors. If you need to put objects into the scope that require
blocking/asynchronous work to resolve, then either make them awaitables
themselves, or make objects that you can fill in later during the coroutine
entry (remember, the objects must be modifiable; you cannot keep a reference
to the scope and try to add keys later).


Error Handling
--------------

If a server receives an invalid event dict - for example, with an unknown type,
missing keys a type should have, or with wrong Python types for objects (e.g.
unicode strings for HTTP headers), it should raise an exception out of the
``send`` awaitable back into the application.

If an application receives an invalid event dict from ``receive`` it should
raise an exception.

In both cases, presence of additional keys in the event dict should not raise
an exception. This is to allow non-breaking upgrades to protocol specifications
over time.

Servers are free to surface errors that bubble up out of application instances
they are running however they wish - log to console, send to syslog, or other
options - but they must terminate the application instance and its associated
connection if this happens.


Extensions
----------

There are times when protocol servers may want to provide server-specific
extensions outside of a core ASGI protocol specification, or when a change
to a specification is being trialled before being rolled in.

For this use case, we define a common pattern for ``extensions`` - named
additions to a protocol specification that are optional but that, if provided
by the server and understood by the application, can be used to get more
functionality.

This is achieved via a ``extensions`` entry in the ``scope`` dict, which is
itself a dict. Extensions have a unicode string name that
is agreed upon between servers and applications.

If the server supports an extension, it should place an entry into the
``extensions`` dict under the extension's name, and the value of that entry
should itself be a dict. Servers can provide any extra scope information
that is part of the extension inside this dict value, or if the extension is
only to indicate that the server accepts additional events via the ``send``
callable, it may just be an empty dict.

As an example, imagine a HTTP protocol server wishes to provide an extension
that allows a new event to be sent back to the server that tries to flush the
network send buffer all the way through the OS level. It provides an empty
entry in the extensions dict to signal that it can handle the event::

    scope = {
        "type": "http",
        "method": "GET",
        ...
        "extensions": {
            "fullflush": {},
        },
    }

If an application sees this it then knows it can send the custom event
(say, of type ``http.fullflush``) via the ``send`` callable.


Strings and Unicode
-------------------

In this document, and all sub-specifications, *byte string* refers to
the ``bytes`` type in Python 3. *Unicode string* refers to the ``str`` type
in Python 3.

This document will never specify just *string* - all strings are one of the
two exact types.

All dict keys mentioned (including those for *scopes* and *events*) are
unicode strings.


Version History
===============

* 2.0 (2017-11-28): Initial non-channel-layer based ASGI spec


Copyright
=========

This document has been placed in the public domain.
