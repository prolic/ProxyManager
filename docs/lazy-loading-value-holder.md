# Lazy Loading Value Holder Proxy

A lazy loading value holder proxy is an object that is wrapping a lazily initialized "real" instance of the proxied
class.

## What is lazy loading?

In pseudo-code, in userland, [lazy loading](http://www.martinfowler.com/eaaCatalog/lazyLoad.html) looks like following:

```php
class MyObjectProxy
{
    private $wrapped;

    public function doFoo()
    {
        $this->init();

        return $this->wrapped->doFoo();
    }

    private function init()
    {
        if (null === $this->wrapped) {
            $this->wrapped = new MyObject();
        }
    }
}
```

This code is problematic, and adds a lot of complexity that makes your unit tests' code even worse.

Also, this kind of usage often ends up in coupling your code with a particular
[Dependency Injection Container](http://martinfowler.com/articles/injection.html)
or a framework that fetches dependencies for you.
That way, further complexity is introduced, and some problems related
with service location raise, as I've explained
[in this article](http://ocramius.github.com/blog/zf2-and-symfony-service-proxies-with-doctrine-proxies/).

Lazy loading value holders abstract this logic for you, hiding your complex, slow, performance-impacting objects behind
tiny wrappers that have their same API, and that get initialized at first usage.

## When do I use a lazy value holder?

You usually need a lazy value holder in cases where following applies

 * your object takes a lot of time and memory to be initialized (with all dependencies)
 * your object is not always used, and the instantiation overhead can be avoided

## Usage examples

[ProxyManager](https://github.com/Ocramius/ProxyManager) provides a factory that eases instantiation of lazy loading
value holders. To use it, follow these steps:

First of all, define your object's logic without taking care of lazy loading:

```php
namespace MyApp;

class HeavyComplexObject
{
    public function doFoo()
    {
        // ... do foo
        echo 'OK!';
        // just write your business logic
        // don't worry about how heavy this object will be!
    }
}
```

Then use the proxy manager to create a lazy version of the object (as a proxy):

```php
namespace MyApp;

use ProxyManager\Configuration;
use ProxyManager\Factory\LazyLoadingValueHolderFactory;
use ProxyManager\Proxy\LazyLoadingInterface;

require_once __DIR__ . '/vendor/autoload.php';

$config      = new Configuration();
$factory     = new LazyLoadingValueHolderFactory($config);
$initializer = function (& $wrappedObject, LazyLoadingInterface $proxy, $method, array $parameters) {
    $proxy->setProxyInitializer(null); // disable initialization

    $wrappedObject = new HeavyComplexObject(); // fill your object with values here

    return true; // confirm that initialization occurred correctly
};

$instance = $factory->createProxy('MyApp\HeavyComplexObject', $initializer);
```

You can now simply use your object as before:

```php
// this will just work as before
$proxy->doFoo(); // OK!
```

## Lazy Initialization

As you can see, we use a closure to handle lazy initialization of the proxy instance at runtime.
The initializer closure signature should be as following:

```php
/**
 * @var object $wrappedObject the instance (passed by reference) of the wrapped object,
 *                            set it to your real object
 * @var object $proxy         the instance proxy that is being initialized
 * @var string $method        the name of the method that triggered lazy initialization
 * @var string $method        an ordered list of parameters passed to the method that
 *                            triggered initialization, indexed by parameter name
 *
 * @return bool true on success
 */
$initializer = function (& $wrappedObject, $proxy, $method, $parameters) {};
```

The
[`ProxyManager\Factory\LazyLoadingValueHolderFactory`](https://github.com/Ocramius/ProxyManager/blob/master/src/ProxyManager/Factory/LazyLoadingValueHolderFactory.php)
produces proxies that implement both the
[`ProxyManager\Proxy\ValueHolderInterface`](https://github.com/Ocramius/ProxyManager/blob/master/src/ProxyManager/Proxy/ValueHolderInterface.php)
and the
[`ProxyManager\Proxy\LazyLoadingInterface`](https://github.com/Ocramius/ProxyManager/blob/master/src/ProxyManager/Proxy/LazyLoadingInterface.php).

At any point in time, you can set a new initializer for the proxy:

```php
$proxy->setProxyInitializer($initializer);
```

In your initializer, you currently **MUST** turn off any further initialization:

```php
$proxy->setProxyInitializer(null);
```

## Triggering Initialization

A lazy loading proxy is initialized whenever you access any property or method of it.
Any of the following interactions would trigger lazy initialization:

```php
// calling a method
$proxy->someMethod();

// reading a property
echo $proxy->someProperty;

// writing a property
$proxy->someProperty = 'foo';

// checking for existence of a property
isset($proxy->someProperty);

// removing a property
unset($proxy->someProperty);

// cloning the entire proxy
clone $proxy;

// serializing the proxy
$unserialized = serialize(unserialize($proxy));
```

Remember to call `$proxy->setProxyInitializer(null);` to disable initialization of your proxy, or it will happen more
than once.

## Tuning performance for production

See [Tuning ProxyManager for Production](https://github.com/Ocramius/ProxyManager/blob/master/docs/tuning-for-production.md).