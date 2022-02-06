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

 - [**Defer**](/timers)
 
   {:.small-hint .mt-n2 .mb-1}
   The callback is executed in the next iteration of the event loop. If there are defers scheduled, the event loop won't wait between iterations.
 - [**Delay**](/timers)
 
   {:.small-hint .mt-n2 .mb-1} 
   The callback is executed after the specified number of seconds. Fractions of a second may be expressed as floating point numbers.
 - [**Repeat**](/timers)
 
   {:.small-hint .mt-n2 .mb-1}
   The callback is executed after the specified number of seconds, repeatedly. Fractions of a second may be expressed as floating point numbers.
 - [**Stream readable**](/streams)
 
   {:.small-hint .mt-n2 .mb-1}
   The callback is executed when there's data on the stream to be read, or the connection closed.
 - [**Stream writable**](/streams)
 
   {:.small-hint .mt-n2 .mb-1}
   The callback is executed when there's enough space in the write buffer to accept new data to be written.
 - [**Signal**](/signals)

   {:.small-hint .mt-n2 .mb-1}
   The callback is executed when the process received a specific signal from the OS.

We'll give up control to the scheduler until the events we're interested in happened.
The scheduler runs a loop that does a few things in each iteration:

 - Check for any deferred callbacks
 - Check for actionable timer / stream / signal events
 - Wait for stream activity until the next timer callback expires (unless there are `defer` events to be executed)

The event loop controls the program flow as long as it runs.
Once we tell the event loop to run it will maintain control until it is suspended, the application errors out, has nothing left to do, or is explicitly stopped.

## Examples

Consider this very simple example:

```php
<?php

require __DIR__ . '/vendor/autoload.php';

use Revolt\EventLoop;

$suspension = EventLoop::createSuspension();

$repeatId = EventLoop::repeat(1, function (): void {
    print '++ Executing callback created by EventLoop::repeat()' . PHP_EOL;
});

EventLoop::delay(5, function () use ($suspension, $repeatId): void {
    print '++ Executing callback created by EventLoop::delay()' . PHP_EOL;

    EventLoop::cancel($repeatId);
    $suspension->resume(null);

    print '++ Suspension::resume() is async!' . PHP_EOL;
});

print '++ Suspending to event loop...' . PHP_EOL;

$suspension->suspend();

print '++ Script end' . PHP_EOL;
```

Upon execution of the above example you should see output like this:

```plain
++ Suspending to event loop...
++ Executing callback created by EventLoop::repeat()
++ Executing callback created by EventLoop::repeat()
++ Executing callback created by EventLoop::repeat()
++ Executing callback created by EventLoop::repeat()
++ Executing callback created by EventLoop::delay()
++ Suspension::resume() is async!
++ Script end
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

$timeoutId = EventLoop::delay(5, function () use ($readableId, $suspension) {
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

## Events

Event callbacks are registered using the methods on `Revolt\EventLoop` and are invoked using the following standardized parameter order:

| Method                                | Callback Signature                      |
| ------------------------------------- | --------------------------------------- |
| [`EventLoop::defer()`](/timers)       | `function(string $callbackId)`          |
| [`EventLoop::delay()`](/timers)       | `function(string $callbackId)`          |
| [`EventLoop::repeat()`](/timers)      | `function(string $callbackId)`          |
| [`EventLoop::onReadable()`](/streams) | `function(string $callbackId, $stream)` |
| [`EventLoop::onWritable()`](/streams) | `function(string $callbackId, $stream)` |
| [`EventLoop::onSignal()`](/signals)   | `function(string $callbackId, $signal)` |

### Pausing, Resuming and Canceling Callbacks

All event callbacks, regardless of type, can be temporarily disabled and enabled in addition to being cancelled via `EventLoop::cancel()`.
This allows for advanced capabilities such as disabling the acceptance of new socket clients in server applications when simultaneity limits are reached.
In general, the performance characteristics of event callback reuse via `enable()`/`disable()` are favorable by comparison to repeatedly canceling and re-registering callbacks.

#### Pausing Callbacks

A simple disable example:

```php
<?php

use Revolt\EventLoop;

// Register a callback we'll disable
$callbackIdToDisable = EventLoop::delay(1, function (): void {
    echo "I'll never execute in one second because: disable()\n";
});

// Register a callback to perform the disable() operation
EventLoop::delay(0.5, function () use ($callbackIdToDisable) {
    echo "Disabling callback: ", $callbackIdToDisable, "\n";
    EventLoop::disable($callbackIdToDisable);
});

EventLoop::run();
```

After our second event callback executes, the event loop exits because there are no longer any enabled event callbacks registered.

#### Resuming Callbacks

`enable()` is the diametric analog of the `disable()` example demonstrated above:

```php
<?php

use Revolt\EventLoop;

// Register a repeating timer callback
$callbackId = EventLoop::repeat(1, function(): void {
    echo "tick\n";
});

// Disable the callback
EventLoop::disable($callbackId);

EventLoop::defer(function () use ($callbackId): void {
    // Immediately enable the callback when the event loop starts
    EventLoop::enable($callbackId);
    // Now that it's enabled we'll see tick output in our console every second.
});

EventLoop::run();
```

#### Cancelling Callbacks

It's important to *always* cancel persistent event callbacks once you're finished with them, or you'll create memory leaks in  your application.
This functionality works in exactly the same way as the above `enable` / `disable` examples:

```php
<?php

use Revolt\EventLoop;

$callbackId = EventLoop::repeat(1, function (): void {
    echo "tick\n";
});

// Cancel $callbackId in five seconds and exit the event loop
EventLoop::delay(5, function () use ($callbackId): void {
    EventLoop::cancel($callbackId);
});

EventLoop::run();
```

#### Cancellation Safety

It is always safe to cancel a callback from within itself. For example:

```php
<?php

use Revolt\EventLoop;

$increment = 0;

EventLoop::repeat(0.1, function ($callbackId) use (&$increment): void {
    echo "tick\n";
    if (++$increment >= 3) {
        EventLoop::cancel($callbackId); // <-- cancel myself!
    }
});

EventLoop::run();
```

It is also always safe to cancel a callback from multiple places. A double-cancel will simply be ignored.

### Referencing Callbacks

Callbacks can either be referenced or unreferenced. An unreferenced callback doesn't keep the event loop alive.
All callbacks are referenced by default.

One example to use unreferenced callbacks is when using signal callbacks.
Generally, if all callbacks are gone and only the signal callback still exists, you want to exit the event loop unless you're actively waiting for that event to happen.

#### Referencing Callbacks

`reference()` marks a callback as referenced. Takes the `$callbackId` as first and only argument.

#### Unreferencing Callbacks

`unreference()` marks a callback as unreferenced. Takes the `$callbackId` as first and only argument.
