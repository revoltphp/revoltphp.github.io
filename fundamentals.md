---
title: Fundamentals
permalink: /fundamentals
layout: base
---
# Fundamentals

Every application making use of cooperative multitasking can only have one scheduler.
It doesn't make sense to have two event loops running at the same time, as they would just have to schedule each other in a busy waiting manner, wasting CPU cycles.

Revolt provides global access to the scheduler using methods on the `Revolt\EventLoop` class.
On the first use of the class, it will automatically create the [best available driver](/extensions).
`Revolt\EventLoop::setDriver()` can be used to set a custom driver.

There's no way to reset the scheduler.
While this might seem useful for test isolation, it would break as soon as test frameworks start to make use of Revolt themselves.
Test frameworks and test code need to share the same scheduler, as there can only be one.

Cooperative multitasking works by telling the scheduler which events we're interested in.
The scheduler will invoke a callback once the event happened.
Revolt supports the following events:

 - **Defer**
 
   {:.small-hint .mt-n2 .mb-1}
   The callback is executed in the next iteration of the event loop. If there are defers scheduled, the event loop won't wait between iterations.
 - **Delay**
 
   {:.small-hint .mt-n2 .mb-1} 
   The callback is executed after the specified number of seconds. Fractions of a second may be expressed as floating point numbers.
 - **Repeat**
 
   {:.small-hint .mt-n2 .mb-1}
   The callback is executed after the specified number of seconds, repeatedly. Fractions of a second may be expressed as floating point numbers.
 - **Stream readable**
 
   {:.small-hint .mt-n2 .mb-1}
   The callback is executed when there's data on the stream to be read, or the connection closed.
 - **Stream writable**
 
   {:.small-hint .mt-n2 .mb-1}
   The callback is executed when there's enough space in the write buffer to accept new data to be written.
 - **Signal**

   {:.small-hint .mt-n2 .mb-1}
   The callback is executed when the process received a specific signal from the OS.

We'll give up control to the scheduler until the events we're interested in happened.
The scheduler runs a loop that does a few things in each iteration:

 - Check for any deferred callbacks
 - Check for actionable timer / stream / signal events
 - Wait until the next timer watcher expires (unless there are `defer` events to be executed)

The event loop controls the program flow as long as it runs.
Once we tell the event loop to run it will maintain control until it is suspended, the application errors out, has nothing left to do, or is explicitly stopped.

Consider this very simple example:

```php
<?php

require __DIR__ . '/vendor/autoload.php';

use Revolt\EventLoop;

$suspension = EventLoop::createSuspension();

$repeatId = EventLoop::repeat(1, function (): void {
    print "++ Executing callback created by EventLoop::repeat()" . PHP_EOL;
});

EventLoop::delay(5, function () use ($suspension, $repeatId): void {
    print "++ Executing callback created by EventLoop::delay()" . PHP_EOL;

    EventLoop::cancel($repeatId);
    $suspension->resume(null);

    print "++ Executed after script ended" . PHP_EOL;
});

$suspension->suspend();

print '++ Script end' . PHP_EOL;
```

Upon execution of the above example you should see output like this:

```plain
++ Executing callback created by EventLoop::repeat()
++ Executing callback created by EventLoop::repeat()
++ Executing callback created by EventLoop::repeat()
++ Executing callback created by EventLoop::repeat()
++ Executing callback created by EventLoop::delay()
++ Script end
++ Executed after script ended
```

This output demonstrates that what happens inside the event loop is like its own separate program.
Your script will not continue past the point of `$suspension->suspend()` unless the suspension point is resumed with `$suspension->resume()` or `$suspension->throw()`.

While an application can and often does take place almost entirely inside the confines of the event loop, we can also use the event loop to do things like the following example which imposes a short-lived timeout for interactive console input:

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

Obviously we could have simply used `fgets(STDIN)` synchronously in this example.
We're just demonstrating that it's possible to move in and out of the event loop to mix synchronous tasks with non-blocking tasks as needed.