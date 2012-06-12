===============================================================================
    Clang C++ Services
===============================================================================

This is a design document, and a request for comments on a proposed Clang C++
service model.

The overarching goal is to build support for running a persistent, caching
service layer adjacent to a users' editor(s) of choice. This layer would
provide much of the functionality that might traditionally be found in an IDE,
but designed to work with different, very "unintegrated" editors and the widely
used and popular commandline development tools on unix-ish operating systems.
It would be implemented in terms of Clang/LLVM's libraries to support
C/C++/Obj-C/Obj-C++ development. It should be heavily integrated into existing
Clang layers such as the Tooling library, libclang, and potentially the plugin
architecture.


Concrete Goals and Non-Goals
============================
Goals:

- Provide a restartable, long-lived background process which manages caching,
  compilation, indexes, and performs the business logic.
- Define an inter-process communication protocol to allow command line tools
  and libraries to communicate w/ service layer.

  - This IPC layer should enable cross-machine usage in theory (so we would
    like to avoid shared memory), but it's not likely to be implemented in the
    initial round.

- Take advantage of multiple cores to parallelize tasks across multiple files
  automatically.
- Support very low-latency queries for UI-interactive modes: code-completion.

  - The crazy stretch goal for this is O(1ms) for code-completion with fully
    warm and primed caches.

- Provide basic command-line tools for interacting with the service layer via
  IPC.
- Provide a stable C API (much like libclang) for interacting with the service
  layer via IPC.

  - Should be a strict subset of the existing libclang API.
  - Clients using only the narrow API should be able to switch trivially
    between the two libraries to get IPC vs. internal process behavior.

- Provide Python bindings around the stable C API for the IPC layer.

  - Same constraints as the C API w.r.t. existing Python bindings.

- Share all implementation with libclang. These should be two interfaces to the
  same core functionality.
- Effective interface strategy for generic open source editors.

  - At least VIM and Emacs bust be easily supported as 1st class citizens.
  - I'd really like to have a Mac and Windows editor as well.

Non-Goals:

- Support languages outside of those supported by Clang already.
- Support code generation or other uses of Clang. (This might be interesting at
  some point, but it's not something we want to be constrained by in the
  initial design.)
- Anything remotely related to actually building an editor or a GUI or an
  actual IDE. This is a service platform proposal and will not creep toward an
  IDE or join in the editor wars. ;]


High-Level Architecture
=======================
The high-level architecture of the service platform follows a traditional
client/server model, with the exception that the client is expected to be able
to start an initial server if one is missing, and for the server to (primarily)
run on the same machine is the user's editor and any other clients. It is
designed to use a socket-like message oriented IPC mechanism for communication
between the client and the server.

Protocol Design
---------------
The communication protocol will take the form of serialized messages encoded
using the LLVM bitcode system. We will define bitcode readers and writers for
each message type. We will build an extremely simplified socket-oriented IPC
mechanism which handles the following tasks:

- Discovering the existing of a running server for a particular project/user
  combination.
- Establishing a connection from the client to the server.

  - Writing a request message to the server in serialized form.
  - Wait for a response message to arrive from the server.
  - Read the response message in serialized form.

This is intended to map cleanly onto an implementation using TCP or Unix domain
sockets, as well as other socket-like libraries.

The types of requests and response provided by the protocol is sketched here to
give a rough idea. The specifics of these will be hashed out as the
implementation and specific tools evolve and mature. We expect the requests to
be *extremely* high level in nature. Here are some example request/response
pairs::

  Request: {
    Kind: Syntax Check Request
    File: src/foo.cpp
    Dirty Buffers: {
      src/foo.cpp: /tmp/xyz123
      src/foo.h: /tmp/abc123
    }
  }
  Response: {
    Kind: Syntax Check Response
    File: src/foo.cpp
    Errors: {
      # A Serialized Diagnostic object, or some similar representation.
    }
    Warnings: {
      # ...
    }
  }

  Request: {
    Kind: Code Complete Request
    Token: <...> # An optional token to resume a previous code-completion
    Location: {
      # Serialized canonical source location
    }
    Dirty Buffers: {
      src/foo.cpp: /tmp/xyz123
      src/foo.h: /tmp/abc123
    }
  }
  Response: {
    Kind: Code Complete Response
    Token: .... # Arbitrary token used to resume this with a new request
    Completions {
      # A list of code completion suggestions
    }
  }

Likely one of the more interesting parts of the protocol is the dirty buffers
section. The goal is to avoid sending large chunks of data across the IPC
mechanism where possible, and instead to allow normal file I/O in most cases.
The idea is that the editor can stash copies of dirty edit buffers into
temporary storage, potentially in-memory temporary storage, and provide
a mapping from the source file to the storage location for the dirty buffer to
allow the clang service to act as-if the currently in-progress edits were saved
when operating on the file.

Each of these nested groups will be implemented with re-usable serialization
and de-serialization logic built on top of the bitcode reader/writer so that we
can build up a collection of common message data types that can be quickly
combined by clients to form particular protocols.

The intent is that the set af protocol interfaces exposed closely resembles the
most high-level of the libclang interfaces. These are necessarily stable,
long-lived interfaces, and so they share many design constraints with libclang.
There should be close parity in design, structure, and available functionality
between the two. It is entirely possible that we will naturally converge on
supporting the full width of libclang's API here, but it's not necessary
initially.


Clang Server
------------
The primary role of the server is to combine two existing constructs in Clang:
the libclang/ASTUnit translation unit caching and management system, and the
lib/Tooling tool running and compilation database management system. It also
includes the functionality to listen for incoming requests, and respond to them
potentially in parallel.

Several changes are needed to the core Clang libraries to support the server
model:

- Support arbitrarily complex file remappings, including in the driver and
  header search logic. This is essential to fully support dirty buffers.
- Thread safety when two threads are concurrently parsing different TUs.

  - One potential requirement will be the ability to share the cached open
    files between different file managers with different file remappings. This
    will reduce the memory overhead significantly, but introduces
    synchronization complexity.

- More optional callback instrumentation of the compilation, for example to
  pause and resume parsing or other operations while communicating with the
  client.
- Factoring some of the logic currently in libclang to implement high-level
  operations into C++ APIs that can be shared by libclang and the server.

The Clang server will operate on one or more compilation databases [#]_
associated with a project. These will be found either by an explicit database
file or using a '.clangrc' file [#]_.  If the RC file specifies multiple
compilation databases for a given project, potentially for different build
configurations, the server will utilize the union of them. These databases will
provide the basis for running tools over source code. The server will expose an
interface for directly adding a compilation to the set managed by the server as
well in order to match expected libclang functionality. This will also allow
IDEs to potentially dynamically update the compilation database as new build
information is available.

The server will also monitor the compilation database and refresh its view if
the database file changes underneath it. This will be accomplished through the
filesystem if supported, or through polling and checksumming if not.

.. [#] TODO: Link to compilation database documentation on clang.llvm.org.

.. [#] TODO: Specify the format of this file, and add support for it to the
       Tooling library as it is generally useful. Current plan is a flat YAML
       block of key-value mappings. Turn this into a link to that
       documentation.

The Clang server will when started will expose its connection through a file.
This will be a platform-specific file allowing a connection to be made. It is
expected to Unix Domain Socket on Linux at least, and likely Mac. Windows
support mechanism here is TBD. If the server is started around a single
compilation database, the connection file will be placed adjacent to it, and
called '.clang_server_connection' [#]_. If the server is started using an RC
file, that RC file will specify a location for the connection file.

.. [#] TODO: Is there a better name for this?

If the connection file is relocated or removed at any point, the server is
required to detect this eventually and shut down. It is expected that in the
event of failure modes multiple servers will be running concurrently for
a particular project for a brief period of time. TThe server should be able to
gracefully cope with the this, and thus avoid holding locks on shared files for
long periods of time, but use filesystem locking whenever updating files.

Clang C++ Client Libraries
--------------------------
The client code will have at its core a set of C++ libraries that use the IPC
mechanism to communicate with a running server, and include code to
automatically launch a server if it is not currently running. The strategy to
locate or launch a server follows:

#) Starting from the working directory of the client (or a given directory),
   walk up from that directory through parent directories looking for one of
   three files:

   #) A '.clangrc' file [#]_ which specifies the location of compilation
      database(s) and (optionally) the means of connecting to a running Clang
      server, or
   #) A file [#]_ which allows connecting to the running Clang server, or
   #) A compilation database.

#) If a means of connecting to a Clang server has been established, the client
   will attempt to connect to the server. If that connection fails for any
   reason, it will remove or relocate the connection file to allow forward
   progress and continue as if it had not been found.
#) If no connection is available, the client code will spawn a new server for
   the given RC file and/or compilation database.

.. [#] TODO: Link to .clangrc docs as above.
.. [#] This file is a platform-specific construct. The current plan is to use
       Unix domain sockts and allow Windows developers to suggest that
       platform's implementation.

