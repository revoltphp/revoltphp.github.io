---
title: Fibers
permalink: /fibers
layout: base
---
# Fibers

Revolt is designed to work well with fibers.
All event callbacks are run in a separate fiber and can suspend it at any time.
If there are no suspensions in an event callback, the fiber will be reused for future event callbacks to save resources.

## Suspensions

Suspensions allow awaiting an event by suspending the current execution context until the event in question happened.
They will suspend the current fiber and return to the event loop, or start running the event loop if called from outside a fiber, i.e. from `{main}`.

Fibers should be suspended and resumed using the `Revolt\EventLoop\Suspension` API.
`Suspension` objects can be created using `Revolt\EventLoop::createSuspension()`.
After obtaining a `Suspension` object, an event callback can be registered to schedule the resumption of the current fiber.
`$suspension->suspend()` will suspend the current execution context until it is resumed via `$suspension->resume()` or `$suspension->throw()`.

## Example

Let's suspend the main execution context until there's data to read from `STDIN` or a timeout expires:

```php
<?php

require __DIR__ . '/vendor/autoload.php';

use Revolt\EventLoop;

if (\stream_set_blocking(STDIN, false) !== true) {
    \fwrite(STDERR, "Unable to set STDIN to non-blocking" . PHP_EOL);
    exit(1);
}

print "Write something and hit enter" . PHP_EOL;

$suspension = EventLoop::createSuspension();

$readableId = EventLoop::onReadable(STDIN, function ($id, $stream) use ($suspension): void {
    EventLoop::cancel($id);

    $chunk = \fread($stream, 8192);

    print "Read " . \strlen($chunk) . " bytes" . PHP_EOL;

    $suspension->resume(null);
});

$timeoutId = EventLoop::delay(5, function () use ($readableId) {
    EventLoop::cancel($readableId);
    
    print "Timeout reached" . PHP_EOL;

    $suspension->resume(null);
});

$suspension->suspend();

EventLoop::cancel($readableId);
EventLoop::cancel($timeoutId);
```
