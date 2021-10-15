---
title: Streams
permalink: /streams
layout: base
---
# Streams

Stream callbacks are how we know when we can read and write to sockets and other streams. These events are how we're able
to actually create things like HTTP servers and asynchronous database libraries using the event loop. As such, stream IO
callbacks form the backbone of any useful non-blocking, concurrent application.

There are two types of IO callbacks:

- Readability callbacks
- Writability callbacks

### Readability Callbacks

{:.note}
> This is an advanced low-level API. Most users should use a stream abstraction instead.

Callbacks registered via `EventLoop::onReadable()` are invoked in the following situations:

- When data is available to read on the stream under observation
- When the stream is at EOF (for sockets, this means the connection is broken)

A common usage pattern for reacting to readable data looks something like this example:

```php
<?php

use Revolt\EventLoop;

const IO_GRANULARITY = 32768;

function isStreamDead($socket): bool {
    return !is_resource($socket) || @feof($socket);
}

EventLoop::onReadable($socket, function ($callbackId, $socket) {
    $socketId = (int) $socket;
    $newData = @fread($socket, IO_GRANULARITY);
    if ($newData != "") {
        // There was actually data and not an EOF notification. Let's consume it!
        parseIncrementalData($socketId, $newData);
    } elseif (isStreamDead($socket)) {
        EventLoop::cancel($callbackId);
    }
});

EventLoop::run();
```

In the above example we've done a few very simple things:

- Register a readability callback for a socket that will be triggered when there is data available to read.
- When we read data from the stream in our callback we pass that to a stateful parser that does something
  domain-specific when certain conditions are met.
- If the `fread()` call indicates that the socket connection is dead we clean up any resources we've allocated for the
  storage of this stream. This process should always include calling `EventLoop::cancel()` on any event loop callbacks we
  registered in relation to the stream.

> You should always read a multiple of the configured chunk size (default: 8192), otherwise your code might not work as expected with loop backends other than `stream_select()`, see [amphp/amp#65](https://github.com/amphp/amp/issues/65) for more information.

### Writabilty Callbacks

{:.note}
> This is an advanced low-level API. Most users should use a stream abstraction instead.

Callbacks registered via `EventLoop::onWritable()` are invoked in the following situations:

- There's enough free space in the buffer to accept new data to be written.
- Streams are essentially *"always"* writable. The only time they aren't is when their respective write buffers are
  full.

A common usage pattern for reacting to writability involves initializing a writability callback without enabling it when
a client first connects to a server. Once incomplete writes occur we're then able to "unpause" the write callback
using `EventLoop::enable()` until data is fully sent without having to create and cancel new callbacks on the same
stream multiple times.

Because streams are essentially *"always"* writable you should only enable writability callbacks while you have data to
send. If you leave these callbacks enabled when your application doesn't have anything to write the callback will trigger
endlessly until disabled or canceled. This will max out your CPU. If you're seeing inexplicably high CPU usage in your
application it's a good bet you've got a writability callback that you failed to disable or cancel after you were
finished with it.

A standard pattern in this area is to initialize writability callbacks in a disabled state before subsequently enabling
them at a later time as shown here:

```php
<?php

use Revolt\EventLoop;

$callbackId = EventLoop::onWritable(STDOUT, function (): void {});
EventLoop::disable($callbackId);
// ...
EventLoop::enable($callbackId);
// ...
EventLoop::disable($callbackId);
```