---
layout: documentation.twig
title: IoC Container
redirect:
    3.x: core/ioc-container

---

## What is Ioc Container

Windwalker DI is a [dependency injection](http://en.wikipedia.org/wiki/Dependency_injection) tools,
provide us an [IOC](http://en.wikipedia.org/wiki/Inversion_of_control) container to manage objects and data.
We also support service provider to help developers build their service in a universal interface.

> For more information about IOC and DI, please see
[Inversion of Control Containers and the Dependency Injection pattern](http://martinfowler.com/articles/injection.html) by Martin Fowler.

## Get IoC Container

Use internal container in controller and Application:

``` php
$session = $this->container->get('system.session');
```

Or use static method in `Ioc` class to get it:

``` php
$container = \Windwalker\Ioc::factory(); // This container is singleton
```

Now we can store objects into it.

``` php
$input = new Input;

$container->set('my.input', $input);

$input = $container->get('my.input');
```

Or get it by `Ioc:get()` statically:

``` php
\Windwalker\Ioc::get('my.input');
```

### Get Sub Container

Every package will use a child container, if the key in child container not found, container will search from parent:

``` php
// Get FlowerPackage container
$container = Ioc::factory('flower');

// Set something
$container->set('sakura', 'Sakura');

// sakura exists
$sakura = $container->get('sakura');

// rose not exists, will find from parent.
$rose = $container->get('rose');
```

Directly get object from sub container by `Ioc`:

``` php
\Windwalker\Ioc::get('sakura', 'flower');
```

## Lazy Loading

Use callback function as input value if we do not want to create object instantly.

``` php
// Set a closure into it
$container->set('input', function(Container $container)
{
    return new Input;
});

// Will call this closure when we get it
$input = $container->get('input');
```

But if we use `set()` method to set callback, this object will be re-created everytime.

## Shared Object (Singleton)

Use `set('foo', $object, true)` or `share('foo', $object)` to make an object singleton, we'll always get the same instance.

``` php
// Share a closure
$container->share('input', function(Container $container)
{
    return new Input;
});

// Will will always get same instance
$input = $container->get('input');

// The second argument of get() is whether forced to create new instance or not
$newInput = $container->get('input', true);

// Use readable constant
$newInput = $container->get('input', Container::FORCE_NEW);
```

## Protect Object

Use `protect()` to prevent others from overriding your important object.

``` php
$container->protect(
    'input',
    function(Container $container)
    {
        return new Input;
    },
    true // Shared or not
);

// We can still get this object
$input = $container->get('input');

// @Throws OutOfBoundsException
$container->set('input', $otherInput);
```

## Alias

It is convenience to set an alias to key of objects which we often use.

``` php
$container->share('system.application', $app)
    ->alias('app', 'system.application');

// Same as system.application
$app = $container->get('app');
```

## Ioc Class

`\Windwalker\Ioc` provides a easy way to get system objects, the benefit to use these methods is that IDE can identify
what object we get, and provides auto-complete:

``` php
$db    = \Windwalker\Ioc::getDatabase();
$input = \Windwalker\Ioc::getInput();
$app   = \Windwalker\Ioc::getApplication();
```

![160331-0001](https://cloud.githubusercontent.com/assets/1639206/14169419/38066a96-f75a-11e5-91da-a3c433bc4771.jpg)

If you want to add your own object in `Ioc`, edit the `/src/Windwalker/Ioc.php` file:

``` php
<?php
// /src/Windwalker/Ioc.php`

namespace Windwalker;

abstract class Ioc extends \Windwalker\Core\Ioc
{
    /**
     * Add this docblock that your IDE can identify what you get.
     *
     * @return  MyObject
     */
	public static function getMyObject()
	{
		return static::get('my.object');
	}
}
```

Now you can use this method in everywhere.

``` php
\Windwalker\Ioc::getMyObject();
```

## Build Object

Container can build an object and automatically inject all the necessary dependencies.

``` php
use Windwalker\IO\Input;
use Windwalker\Registry\Registry;

class MyClass
{
    public $input;
    public $config;

    public function __construct(Input $input, Registry $config)
    {
        $this->input = $input;
        $this->config = $config;
    }
}

$myObject = $container->createObject('MyClass');

$myObject->input; // Input
$myObject->config; // Registry
```

### Binding Classes

Sometimes we hope to inject particular object we want, we can bind a class as key to let Container know what you want to
instead the dependency object.

Here is a class but dependency to an abstract class, we can bind a sub class to container for use.

``` php
use Windwalker\Model\AbstractModel;
use Windwalker\Registry\Registry;

class MyClass
{
    public $model;
    public $config;

    public function __construct(AbstractModel $model, Registry $config)
    {
        $this->model = $model;
        $this->config = $config;
    }
}

class MyModel extends AbstractModel
{
}

// Bind MyModel as AbstractModel
$container->share('Windwalker\\Model\\AbstractModel', function()
{
    return new MyModel;
});

$myObject = $container->createObject('MyClass');

$myObject->model; // MyModel
```

## Extending

Container allows you to extend an object, the new instance or closure will override the original one, this is a sample:

``` php
// Create an item first
$container->share('flower', function()
{
    return new Flower;
});

$container->extend('flower', function($origin, Container $container)
{
    $origin->name = 'sakura';

    return $origin;
});

$flower = $container->get('flower');

$flower->name; // sakura
```

## Container Aware

The `ContainerAwareInterface` defines getter & setter of Container as a system, constructor, we often use it on application or controller classes.

``` php
use Windwalker\DI\ContainerAwareInterface;

class MyController implements ContainerAwareInterface
{
    protected $container;

    public function getContainer()
    {
        return $this->container;
    }

    public function setContainer(Container $container)
    {
        $this->container = $container;
    }

    public function execute()
    {
        $container = $this->getContainer();
    }
}
```

### Using Trait

In PHP 5.4, you can use `ContainerAwareTrait` to create an aware object.

``` php
class MyController implements ContainerAwareInterface
{
    use ContainerAwareTrait;

    public function execute()
    {
        $container = $this->getContainer();
    }
}
```

## Service Providers

Service providers is an useful way to encapsulate logic of creating objects and services.
Just implements the `Windwalker\DI\ServiceProviderInterface`.

``` php
use Windwalker\DI\Container;
use Windwalker\DI\ServiceProviderInterface;

class DatabaseServiceProvider implements ServiceProviderInterface
{
    public function register(Container $container)
    {
        $container->share('db', function (Container $container)
        {
            $options = $container->get('config')->get('database');

            return DatabaseFactory::getDbo($options['driver'], $options);
        });

        // Or use callable
        $container->share('query', array($this, 'getQuery'));
    }

    public function getQuery(Container $container)
    {
        return new MysqlQuery;
    }
}

$container->registerServiceProvider(new DatabaseServiceProvider);
```

## Facade

Windwalker has a Facade class to help us use proxy pattern to call object methods statically

``` php
// Create a object into conteiner
$container->share('my.router', new Router);
```

``` php
use Windwalker\Core\Facade\AbstractProxyFacade;

class MyRouter extends AbstractProxyFacade
{
    // Same as the container key, then we can auto match the object in container.
    protected static $_key = 'my.router';
}
```

Now just call the Router methods statically from `MyRouter`.

``` php
MyRouter::build('route');

// Same as...

$container->get('my.router')->build('route');
```

To make IDE support auto-complete, we can add PhpDoc to class.

``` php
/**
 * The Router class.
 *
 * @method  static  string  build($route, $queries = array(), $type = RestfulRouter::TYPE_RAW, $xhtml = false)
 *
 * ...
 *
 * @see \Windwalker\Router\Router
 */
class MyRouter extends AbstractProxyFacade
{
    protected static $_key = 'my.router';
}
```
