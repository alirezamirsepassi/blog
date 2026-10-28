---
layout: post
title: Service Locator design pattern
date:   2022-11-01
categories: [software, design-pattern]
---

During creating and designing the architechture of applications, we create a lot of objects ("Services") and managing these objects in a big application without a service container will become a nightmare eventually.

<!-- more -->

## What is service container?
A service container is a form of inversion of control ("IoC") which is just an array or in other terms registry of instantiated services which uses service locator pattern to give you access to registered services.

Below you can find a very simple implementation of service container compatible with [PSR-11](https://www.php-fig.org/psr/psr-11/).

```php
use Psr\Container\ContainerInterface;

class Container implements ContainerInterface {
    private readonly $registry = [];

    public function get($id): object
    {
        if (!$this->has($id)) {
            throw new RuntimeException('Service does not exist!');
        }
    
        return $this->registry[$id];
    }
    
    public function set($id, object $object): void
    {
        $this->registry[$id] = $object;
    }
    
    public function has($id): bool
    {
        return array_key_exists($id, $this->registry);
    }
}
```

And a simple usage of this container would be like:

```php
$container = new Container();

$serviceA = new ServiceA();
$serviceB = new ServiceB($serviceA);

$container->set('service_a', $serviceA);
$container->set('service_b', $serviceB);

// Somewhere in the application
$container->get('service_b')->doSomethingUsingServiceA(...);
```

But in most cases we won't end up using this container as it's not memory and time efficient because we have to instantiate every service we want to be registered in the container, instead, we use an advanced container with the ability to learn how to create a service like [Symfony DependencyInjection component](https://symfony.com/doc/current/components/dependency_injection.html).

Furthermore, using service locator design pattern in most cases is considered an anti-pattern and has very concerning drawbacks that in my opinion, every developer should avoid using when dependency injection is available.

## What is dependency injection?
![Coupling VS Cohesion](/assets/images/CouplingVsCohesion.svg)

As [WikiPedia](https://en.wikipedia.org/wiki/Dependency_injection) describes it:

> dependency injection is a design pattern in which an object or function receives other objects or functions that it depends on. A form of inversion of control, dependency injection aims to separate the concerns of constructing objects and using them, leading to loosely coupled programs.

I describe it as a way for services to talk to each other without knowing each other's implementation.


## What are the drawbacks?
The main drawbacks of using service locator fall into two categories:

- Types are not known in runtime which makes the code error-prone.
- Dependencies of class are hidden and to write unit tests you have to go through the code and find the dependencies.

```php
# Good
class Service {
    public function __constructor(private readonly Dummy $dummy)
    {
    }

    public function doSomething()
    {
        $this->dummy->something();
    }
}
```

```php
# Bad
class Service {
    public function __constructor(private readonly ContainerInterface $container)
    {
    }

    public function doSomething()
    {
        // We can't be sure this return dummy or any service!
        $dummy = $this->container->get(Dummy::class);

        // We don't know method `something` exists in the $dummy variable!
        $dummy->something();
    }
}
```

## What is wrong with Laravel?
Laravel like many other frameworks uses dependency injection design pattern but the problem is that it promotes and uses service locator more!

### Helpers
There are many helpers in Laravel that use service locator. For instance, take a look at how `config(...)` works:

```php
/**
 * Get / set the specified configuration value.
 *
 * If an array is passed as the key, we will assume you want to set an array of values.
 *
 * @param  array|string|null  $key
 * @param  mixed  $default
 * @return mixed|\Illuminate\Config\Repository
 */
function config($key = null, $default = null)
{
    if (is_null($key)) {
        return app('config');
    }

    if (is_array($key)) {
        return app('config')->set($key);
    }

    return app('config')->get($key, $default);
}
```

The ironic part of these helpers is that in most cases, you can inject the implementation directly into your class but many developers don't know or even decide not to!

```php
use Illuminate\Config\Repository;

class Service {
    public function __construct(private readonly Repository $config)
    {
    }
}
```

### Facades
Facade is another of many idiotic ideas that was introduced in Laravel which made naive developers use it a lot and never learn that swapping the implementation of a dependency can be easily done using an interface!

> Facades provide a "static" interface to classes that are available in the application's service container.

```php
/**
 * Resolve the facade root instance from the container.
 *
 * @param  string  $name
 * @return mixed
 */
protected static function resolveFacadeInstance($name)
{
    if (isset(static::$resolvedInstance[$name])) {
        return static::$resolvedInstance[$name];
    }

    if (static::$app) {
        if (static::$cached) {
            return static::$resolvedInstance[$name] = static::$app[$name];
        }

        return static::$app[$name];
    }
}
```

In some cases I even see some developers using facades without having any intention of swapping the underlying implementation in future.

[Facade VS Dependency Injection](https://laravel.com/docs/9.x/facades#facades-vs-dependency-injection)


## Conculusion
If you want to write maintainable and testable code, you should avoid using service locator and facades and helpers ... . Dependencies of a class are better to be exposed and visible rather than hidden.