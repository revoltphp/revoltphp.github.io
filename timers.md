---
title: Timers
permalink: /timers
layout: base
---
# Timers

The event loop exposes several ways to schedule timers. Let's look at some details for each function.

## Deferred Callbacks

- `defer()` schedules a callback to execute in the next iteration of the event loop.
- This method guarantees a clean call stack to avoid starvation of other events in the current iteration of the loop.
  A `defer()` callback is *always* executed in the next tick of the event loop.
- After a `defer()` timer executes, it is automatically garbage collected by the event loop so there is no need
  for applications to manually cancel the associated callback.
- Like all event callbacks, `defer()` timers may be disabled and re-enabled. If you disable this callback between the time you
  schedule it and the time that it actually runs the event loop *will not* be able to garbage collect it until it
  executes. Therefore, you must manually cancel a `defer()` callback yourself if it never actually executes to free any
  associated resources.

### Example

```php
<?php // using EventLoop::defer()

use Revolt\EventLoop;

echo "line 1\n";

EventLoop::defer(function (): void {
    echo "line 3\n";
});

echo "line 2\n";

EventLoop::run();
```

### Callback Signature

`function (string $callbackId)`

## Delayed Callbacks

- `delay()` schedules a callback to execute after a delay of `n` seconds
- A `delay()` callback is also automatically garbage collected by the event loop after execution and applications should not
  manually cancel it unless they wish to discard the callback entirely prior to execution.
- A `delay()` callback that is disabled has its delay time reset so that the original delay time starts again from zero
  once re-enabled.
- Like `defer()` callbacks, a timer scheduled for one-time execution must be manually canceled to free resources if it
  never runs due to being disabled by the application after creation.

### Example

```php
<?php // using EventLoop::delay()

use Revolt\EventLoop;

EventLoop::delay(3, function (): void {
    print '3 seconds passed';
});

EventLoop::run();
```

### Callback Signature

`function (string $callbackId)`

## Periodic Callbacks

- `repeat()` schedules a callback to repeatedly execute every `n` seconds.
- Like all other event callbacks, `repeat()` timers may be disabled/re-enabled at any time.
- Unlike `defer()` and `delay()` callbacks, `repeat()` callbacks must be explicitly canceled to free associated resources.
  Failure to free `repeat()` callbacks via `cancel()` once their purpose is fulfilled will result in memory leaks in your
  application. It is not enough to simply disable `repeat()` callbacks as their data is only freed upon cancellation.

### Example

```php
<?php // using EventLoop::repeat()

use Revolt\EventLoop;

EventLoop::repeat(0.1, function ($callbackId): void {
    static $i = 0;

    if ($i++ < 3) {
        echo "tick\n";
    } else {
        EventLoop::cancel($callbackId);
    }
});

EventLoop::run();
```

### Callback Signature

`function (string $callbackId): void`

### Timer Drift

Repeat timers are basically simple delay timers that are automatically re-scheduled right before the appropriate handler
is triggered. They are subject to timer drift. Multiple timers might stack up in case they execute as coroutines.
