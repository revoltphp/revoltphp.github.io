---
title: Revolt - The rock-solid event loop for PHP
permalink: /
layout: base
---
# Revolt - The rock-solid event loop for PHP
It may surprise people to learn that the PHP standard library already has everything we need to write event-driven and non-blocking applications. We only reach the limits of native PHP's functionality in this area when we ask it to poll thousands of file descriptors for IO activity at the same time. Even in this case, though, the fault is not with PHP but the underlying system `select()` call which is linear in its performance degradation as load increases.

For performance that scales out to high volume we require more advanced capabilities currently found only in extensions. If you wish to, for example, service 10,000 simultaneous clients in an event loop backed socket server, you should use one of the event loop implementations based on a PHP extension. However, if you're using the package in a strictly local program for non-blocking concurrency, or you don't need to handle more than a few hundred simultaneous clients in a server application, the native PHP functionality should be adequate.

[Follow us on Twitter](https://twitter.com/revoltphp) for updates.