# Nanomsg FFI bindings for PicoLisp

[![GitHub release](https://img.shields.io/github/release/aw/picolisp-nanomsg.svg)](https://github.com/aw/picolisp-nanomsg) [![Build Status](https://travis-ci.org/aw/picolisp-nanomsg.svg?branch=master)](https://travis-ci.org/aw/picolisp-nanomsg) [![Dependency](https://img.shields.io/badge/%5Bdeps%5D%20Nanomsg-0.8--beta-ff69b4.svg)](https://github.com/nanomsg/nanomsg) [![Dependency](https://img.shields.io/badge/[deps] picolisp--unit-v1.0.0-ff69b4.svg)](https://github.com/aw/picolisp-unit.git)

[Nanomsg](http://nanomsg.org/index.html) FFI bindings for [PicoLisp](http://picolisp.com/).

The following protocols are supported:

  1. [REQ/REP](#example-reqrep)
  2. [PUB/SUB](#example-pubsub)
  3. [BUS](#example-bus)
  4. [PAIR](#example-pair)
  5. [PUSH/PULL (PIPELINE)](#example-pushpull---pipeline)
  6. [SURVEYOR/RESPONDENT](#example-survey)

# Requirements

  * PicoLisp 64-bit v3.1.9+
  * Git
  * UNIX/Linux development/build tools (gcc, make/gmake, etc..)

# Explanation

To learn more about PicoLisp and this Nanomsg library, please read the [EXPLAIN.md](EXPLAIN.md) document.

# Getting started

This binding relies on the _Official Nanomsg C Library_, compiled as a shared library.

  1. Type `make` to pull and compile the _Official Nanomsg C Library_.
  2. Include `nanomsg.l` in your project
  3. Try the example below

### Linking and Paths

Once compiled, the shared library is symlinked as:

    .lib/libnanomsg.so -> .modules/nanomsg/HEAD/src/.libs/libnanomsg.so

The `nanomsg.l` file searches for `.lib/libnanomsg.so`, relative to its current directory.

### Updating

To keep everything updated, type:

    git pull && make clean && make

# Usage

Only the following functions are exported publicly, and namespaced with `(symbols 'nanomsg)` (or the prefix: `nanomsg~`):

  * `protocol-bind`: bind a `REP, PUB, BUS, PAIR, PULL, or SURVEYOR` socket (inproc, ipc, tcp)
  * `protocol-connect`: connect to a `REQ, SUB, BUS, PAIR, PUSH, RESPONDENT` socket (inproc, ipc, tcp)
  * `end-sock`: shutdown and close a socket
  * `msg-recv`: receive a message (blocking/non-blocking)
  * `msg-send`: send a message (blocking/non-blocking)
  * `subscribe`: subscribe to a `PUB/SUB` topic
  * `unsubscribe`: unsubscribe from a `PUB/SUB` topic

> **Note:** These functions are not namespace [local symbols](http://software-lab.de/doc/refL.html#local), which means they would redefine symbols with the same name in the `'pico` namespace.

## Error handling

When an error occurs, `'InternalError` is thrown, along with the error (error type in `car`, message in `cdr`). The error will also be returned by the `(catch)` expression. Ensure your `(catch)` ends with `NIL`.

```lisp
pil +
(load "nanomsg.l")

(symbols 'nanomsg)
(let Error
  (catch 'InternalError
    (protocol-bind "REP" "tcpz://127.0.0.1:5560" "AF_SP_RAW")
    (prinl "you shouldn't see this") NIL)

  (when Error (println @)) )
-> (NanomsgError . "Protocol not supported")
```

# Example (REQ/REP)

## Server

```lisp
pil +
(load "nanomsg.l")

(symbols 'nanomsg)
(unless (fork)
  (let Sockpair
    (protocol-bind "REP" "tcp://127.0.0.1:5560")

    (prinl (msg-recv (car Sockpair)))
    (msg-send (car Sockpair) "Yep I can see it!" T) # non-blocking

    (end-sock Sockpair) )

  (bye) )

# => Can you see this?
```

## Client

```lisp
pil +
(load "nanomsg.l")

(symbols 'nanomsg)
(unless (fork)
  (let Sockpair
    (protocol-connect "REQ" "tcp://127.0.0.1:5560")
    (msg-send (car Sockpair) "Can you see this?")
    (prinl (msg-recv (car Sockpair)))
    (end-sock Sockpair) )
  (bye) )

# => Yep I can see it!
```

# Example (PUB/SUB)

## Server

```lisp
pil +
(load "nanomsg.l")

(symbols 'nanomsg)
(unless (fork)
  (let Sockpair
    (protocol-connect "SUB" "tcp://127.0.0.1:5560")
    (subscribe (car Sockpair) "test")
    (while T (prinl "RECEIVED: " (msg-recv (car Sockpair))) (wait 1000 (unsubscribe 0 "test")))
    (end-sock Sockpair) )
  (bye) )

# => RECEIVED: test Hello World!
```

## Client

```lisp
pil +
(load "nanomsg.l")

(symbols 'nanomsg)
(let Sockpair
  (protocol-bind "PUB" "tcp://127.0.0.1:5560")
  (while T (msg-send (car Sockpair) "test Hello World!"))
  (end-sock Sockpair) )
```

# Example (BUS)

## Server

```lisp
pil +
(load "nanomsg.l")

(symbols 'nanomsg)
(unless (fork)
  (let Sockpair
    (protocol-connect "BUS" "tcp://127.0.0.1:5560")
    (prinl (msg-recv (car Sockpair)))
    (end-sock Sockpair) )
  (bye) )

# => Hello World!
```

## Client

```lisp
pil +
(load "nanomsg.l")

(symbols 'nanomsg)
(unless (fork)
  (let Sockpair
    (protocol-bind "BUS" "tcp://127.0.0.1:5560")
    (msg-send (car Sockpair) "Hello World!")
    (end-sock Sockpair) )
  (bye) )
```

# Example (PAIR)

## Server

```lisp
pil +
(load "nanomsg.l")

(symbols 'nanomsg)
(unless (fork)
  (let Sockpair
    (protocol-connect "PAIR" "tcp://127.0.0.1:5560")
    (prinl (msg-recv (car Sockpair)))
    (end-sock Sockpair) )
  (bye) )

# => Hello World!
```

## Client

```lisp
pil +
(load "nanomsg.l")

(symbols 'nanomsg)
(unless (fork)
  (let Sockpair
    (protocol-bind "PAIR" "tcp://127.0.0.1:5560")
    (prinl (msg-send (car Sockpair) "Hello World!"))
    (end-sock Sockpair) )
  (bye) )
```

# Example (PUSH/PULL) - PIPELINE

## Server

```lisp
pil +
(load "nanomsg.l")

(symbols 'nanomsg)
(unless (fork)
  (let Sockpair
    (protocol-bind "PULL" "tcp://127.0.0.1:5560")
    (prinl (msg-recv (car Sockpair)))
    (end-sock Sockpair) )
  (bye) )

# => Hello Pipeline
```

## Client

```lisp
pil +
(load "nanomsg.l")

(symbols 'nanomsg)
(unless (fork)
  (let Sockpair
    (protocol-connect "PUSH" "tcp://127.0.0.1:5560")
    (prinl (msg-send (car Sockpair) "Hello Pipeline"))
    (end-sock Sockpair) )
  (bye) )
```

# Example (SURVEYOR/RESPONDENT)

> **Note:** The _Surveyor_ protocol in Nanomsg is buggy, it's possible for this not to
work as expected.

## Server

```lisp
pil +
(load "nanomsg.l")

(symbols 'nanomsg)
(unless (fork)
  (let Sockpair (protocol-bind "SURVEYOR" "tcp://127.0.0.1:5560")
    (msg-send (car Sockpair) "Knock knock.")
    (prinl (msg-recv (car Sockpair)))
    (end-sock Sockpair) )
  (bye) )

# => Who's there?
```

## Client

```lisp
pil +
(load "nanomsg.l")

(symbols 'nanomsg)
(unless (fork)
  (let Sockpair
    (protocol-connect "RESPONDENT" "tcp://127.0.0.1:5560")
    (prinl (msg-recv (car Sockpair)))
    (msg-send (car Sockpair) "Who's there?")
    (end-sock Sockpair) )
  (bye) )

# => Knock knock.
```

# Non-blocking I/O

Some situations require non-blocking I/O. You can call `msg-recv` or `msg-send` with a last argument `T` to enable non-blocking mode. Be aware `NIL` will be returned if `EAGAIN` is received during a non-blocking call. You need to manually poll/loop over the socket in this situation.

Usage example:

```lisp
...
(let Msg (msg-recv (car Sockpair) T)
  (when Msg (fifo '*Messages Msg)) )
...
```

# Receive buffer size

A fixed amount of memory is allocated for each receive buffer. The default setting is `8192` Bytes (8 KiB).

This can be changed with the environment variable `NANOMSG_MAX_SIZE`. You can also overwrite the `MSG_MAX_SIZE` global constant at runtime.

# Testing

This library now comes with full [unit tests](https://github.com/aw/picolisp-unit). To run the tests, type:

    make check

# Contributing

If you find any bugs or issues, please [create an issue](https://github.com/aw/picolisp-nanomsg/issues/new).

If you want to improve this library, please make a pull-request.

# License

[MIT License](LICENSE)
Copyright (c) 2015-2016 Alexander Williams, Unscramble <license@unscramble.jp>
