---
title: Signals
permalink: /signals
layout: base
---
# Signals

Signals are [standardized messages in Unix-like operating systems](https://en.wikipedia.org/wiki/Signal_(IPC)).

## Signal Callbacks

`EventLoop::onSignal()` can be used to react to signals sent to the process.

```php
<?php

use Revolt\EventLoop;

// Let's tick off output once per second, so we can see activity.
EventLoop::repeat(1, function (): void {
    echo "tick: ", date('c'), "\n";
});

// What to do when a SIGINT signal is received
EventLoop::onSignal(SIGINT, function (): void {
    echo "Caught SIGINT! exiting ...\n";
    exit;
});

EventLoop::run();
```

As should be clear from the [fundamentals](/fundamentals), signal callbacks may be enabled, disabled and canceled like any other event callback.

Generally, if all callbacks are gone and only the signal callback still exists, you want to exit the event loop unless you're not actively waiting for that event to happen.

### Signal Number Availability

`ext-uv` exposes `UV::SIG*` constants for watchable signals. Applications using the `EventDriver` will need to manually
specify the [appropriate integer signal numbers](https://en.wikipedia.org/wiki/Signal_(IPC)#Default_action) when registering signal callbacks or rely on `ext-pcntl`.
