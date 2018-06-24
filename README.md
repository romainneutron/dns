# Dns

[![Build Status](https://travis-ci.org/reactphp/dns.svg?branch=master)](https://travis-ci.org/reactphp/dns)

Async DNS resolver for [ReactPHP](https://reactphp.org/).

The main point of the DNS component is to provide async DNS resolution.
However, it is really a toolkit for working with DNS messages, and could
easily be used to create a DNS server.

**Table of contents**

* [Basic usage](#basic-usage)
* [Caching](#caching)
  * [Custom cache adapter](#custom-cache-adapter)
* [Advanced usage](#advanced-usage)
  * [DatagramTransportExecutor](#datagramtransportexecutor)
  * [HostsFileExecutor](#hostsfileexecutor)
* [Install](#install)
* [Tests](#tests)
* [License](#license)
* [References](#references)

## Basic usage

The most basic usage is to just create a resolver through the resolver
factory. All you need to give it is a nameserver, then you can start resolving
names, baby!

```php
$loop = React\EventLoop\Factory::create();

$config = React\Dns\Config\Config::loadSystemConfigBlocking();
$server = $config->nameservers ? reset($config->nameservers) : '8.8.8.8';

$factory = new React\Dns\Resolver\Factory();
$dns = $factory->create($server, $loop);

$dns->resolve('igor.io')->then(function ($ip) {
    echo "Host: $ip\n";
});

$loop->run();
```

See also the [first example](examples).

The `Config` class can be used to load the system default config. This is an
operation that may access the filesystem and block. Ideally, this method should
thus be executed only once before the loop starts and not repeatedly while it is
running.
Note that this class may return an *empty* configuration if the system config
can not be loaded. As such, you'll likely want to apply a default nameserver
as above if none can be found.

> Note that the factory loads the hosts file from the filesystem once when
  creating the resolver instance.
  Ideally, this method should thus be executed only once before the loop starts
  and not repeatedly while it is running.

Pending DNS queries can be cancelled by cancelling its pending promise like so:

```php
$promise = $resolver->resolve('reactphp.org');

$promise->cancel();
```

But there's more.

## Caching

You can cache results by configuring the resolver to use a `CachedExecutor`:

```php
$loop = React\EventLoop\Factory::create();

$config = React\Dns\Config\Config::loadSystemConfigBlocking();
$server = $config->nameservers ? reset($config->nameservers) : '8.8.8.8';

$factory = new React\Dns\Resolver\Factory();
$dns = $factory->createCached($server, $loop);

$dns->resolve('igor.io')->then(function ($ip) {
    echo "Host: $ip\n";
});

...

$dns->resolve('igor.io')->then(function ($ip) {
    echo "Host: $ip\n";
});

$loop->run();
```

If the first call returns before the second, only one query will be executed.
The second result will be served from an in memory cache.
This is particularly useful for long running scripts where the same hostnames
have to be looked up multiple times.

See also the [third example](examples).

### Custom cache adapter

By default, the above will use an in memory cache.

You can also specify a custom cache implementing [`CacheInterface`](https://github.com/reactphp/cache) to handle the record cache instead:

```php
$cache = new React\Cache\ArrayCache();
$loop = React\EventLoop\Factory::create();
$factory = new React\Dns\Resolver\Factory();
$dns = $factory->createCached('8.8.8.8', $loop, $cache);
```

See also the wiki for possible [cache implementations](https://github.com/reactphp/react/wiki/Users#cache-implementations).

## Advanced Usage

### DatagramTransportExecutor

The `DatagramTransportExecutor` can be used to
send DNS queries over a datagram transport such as UDP.

This is the main class that sends a DNS query to your DNS server and is used
internally by the `Resolver` for the actual message transport.

For more advanced usages one can utilize this class directly.
The following example looks up the `IPv6` address for `igor.io`.

```php
$loop = Factory::create();
$executor = new DatagramTransportExecutor($loop);

$executor->query(
    '8.8.8.8:53', 
    new Query($name, Message::TYPE_AAAA, Message::CLASS_IN)
)->then(function (Message $message) {
    foreach ($message->answers as $answer) {
        echo 'IPv6: ' . $answer->data . PHP_EOL;
    }
}, 'printf');

$loop->run();
```

See also the [fourth example](examples).

Note that this executor does not implement a timeout, so you will very likely
want to use this in combination with a `TimeoutExecutor` like this:

```php
$executor = new TimeoutExecutor(
    new DatagramTransportExecutor($loop),
    3.0,
    $loop
);
```

Also note that this executor uses an unreliable UDP transport and that it
does not implement any retry logic, so you will likely want to use this in
combination with a `RetryExecutor` like this:

```php
$executor = new RetryExecutor(
    new TimeoutExecutor(
        new DatagramTransportExecutor($loop),
        3.0,
        $loop
    )
);
```

> Internally, this class uses PHP's UDP sockets and does not take advantage
  of [react/datagram](https://github.com/reactphp/datagram) purely for
  organizational reasons to avoid a cyclic dependency between the two
  packages. Higher-level components should take advantage of the Datagram
  component instead of reimplementing this socket logic from scratch.

### HostsFileExecutor

Note that the above `Executor` class always performs an actual DNS query.
If you also want to take entries from your hosts file into account, you may
use this code:

```php
$hosts = \React\Dns\Config\HostsFile::loadFromPathBlocking();

$executor = new Executor($loop, new Parser(), new BinaryDumper(), null);
$executor = new HostsFileExecutor($hosts, $executor);

$executor->query(
    '8.8.8.8:53', 
    new Query('localhost', Message::TYPE_A, Message::CLASS_IN)
);
```

## Install

The recommended way to install this library is [through Composer](https://getcomposer.org).
[New to Composer?](https://getcomposer.org/doc/00-intro.md)

This will install the latest supported version:

```bash
$ composer require react/dns:^0.4.13
```

See also the [CHANGELOG](CHANGELOG.md) for details about version upgrades.

This project aims to run on any platform and thus does not require any PHP
extensions and supports running on legacy PHP 5.3 through current PHP 7+ and
HHVM.
It's *highly recommended to use PHP 7+* for this project.

## Tests

To run the test suite, you first need to clone this repo and then install all
dependencies [through Composer](https://getcomposer.org):

```bash
$ composer install
```

To run the test suite, go to the project root and run:

```bash
$ php vendor/bin/phpunit
```

The test suite also contains a number of functional integration tests that rely
on a stable internet connection.
If you do not want to run these, they can simply be skipped like this:

```bash
$ php vendor/bin/phpunit --exclude-group internet
```

## License

MIT, see [LICENSE file](LICENSE).

## References

* [RFC 1034](https://tools.ietf.org/html/rfc1034) Domain Names - Concepts and Facilities
* [RFC 1035](https://tools.ietf.org/html/rfc1035) Domain Names - Implementation and Specification
