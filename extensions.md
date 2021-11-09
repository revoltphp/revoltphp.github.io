---
title: Extensions
permalink: /extensions
layout: base
---
# Extensions

The PHP standard library already has everything we need to write event-driven and non-blocking applications.
We only reach the limits of native PHP's functionality in this area when we ask it to poll thousands of file descriptors for IO activity at the same time.
Even in this case, though, the fault is not with PHP but the underlying system `select()` call which is linear in its performance degradation as load increases.
`select()` is usually limited to a fixed size of 1024 file descriptors.

For performance that scales out to high volume we require more advanced capabilities currently found only in extensions.
If you wish to, for example, service 10,000 simultaneous clients in an event loop backed socket server, you should use one of the event loop implementations based on a PHP extension.
However, if you're using the package in a strictly local program for non-blocking concurrency, or you don't need to handle more than a few hundred simultaneous clients in a server application, the native PHP functionality should be adequate.

Revolt currently ships with four built-in event loop implementations:

| Class                     | Extension                                              | Repository |
| ------------------------- | ------------------------------------------------------ | ---------- |
| `Revolt\EventLoop\Driver\StreamSelectDriver` | –                                                      | –          |
| `Revolt\EventLoop\Driver\EvDriver`           | [`pecl/ev`](https://pecl.php.net/package/ev)           | [`php-ev`](https://bitbucket.org/osmanov/pecl-ev) |
| `Revolt\EventLoop\Driver\EventDriver`        | [`pecl/event`](https://pecl.php.net/package/event)     | [`pecl-event`](https://bitbucket.org/osmanov/pecl-event) |
| `Revolt\EventLoop\Driver\UvDriver`           | [`pecl/uv`](https://pecl.php.net/package/uv)           | [`php-uv`](https://github.com/amphp/ext-uv) |

It's not important to choose one implementation for your application.
The package will automatically select the best available driver.
It's perfectly fine to have one of the extensions in production while relying on the `StreamSelectDriver` locally for development.

If you want to quickly switch implementations during development, e.g. for comparison or testing, you can set the `REVOLT_DRIVER` environment variable to one of the classes.
If you use a custom implementation, this only works if the implementation's constructor doesn't take any arguments.
