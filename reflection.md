---
title: Reflection
permalink: /reflection
layout: base
---
# Reflection

> In computer science, reflective programming or reflection is the ability of a process to examine, introspect, and modify its own structure and behavior.
> 
> [Wikipedia](https://en.wikipedia.org/wiki/Reflective_programming)

The event loop allows its own state to be inspected for various reasons, e.g. to build monitoring and observability tooling.

All currently known event loop identifiers can be queried using `EventLoop::getIdentifiers()`.
It'll return an array of string identifiers, which can be used for further inspection:

 - `EventLoop::getType()` returns the callback type as an instance of `CallbackType`.
 - `EventLoop::isEnabled()` returns whether the given callback is currently enabled.
 - `EventLoop::isReferenced()` returns whether the given callback is currently referenced, i.e. keeps the event loop running.

Let's say we want to monitor the number of enabled callbacks by type, then we can periodically collect these counts with the reflection API and report them to some kind of time series service:

```php
$stats = [];

foreach (EventLoop::getIdentifiers() as $identifier) {
    if (EventLoop::isEnabled($identifier)) {
        $type = EventLoop::getType($identifier);
    
        $stats[$type->name] ??= 0;
        $stats[$type->name]++;
    }
}

// report $stats somewhere
```