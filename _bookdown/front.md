## Installation

Install _Capsule_ via [Composer](https://getcomposer.org):

```
composer require capsule/di ^3.0
```

The Github repository is at [capsulephp/di](https://github.com/capsulephp/di).

## Autowiring Container

_Capsule_ will auto-inject typehinted constructor parameters.

```php
use Capsule\Di\Container;
use Capsule\Di\Definitions;

class Foo implements FooInterface
{
    public function __construct(
        protected Bar $bar,
        protected string $baz = 'dib'
    ) {
    }

    public function getBaz()
    {
        return $this->baz;
    }

    public function setBaz(string $baz)
    {
        $this->baz = $baz;
    }

    // ... other methods ...
}

$def = new Definitions();
$container = new Container($def);

$foo = $container->get(Foo::CLASS);
echo $foo->getBaz(); // 'dib'
```

Learn more about [Definitions](/3.x/definitions/overview.html)
and the [Container](/3.x/container.html).

## Initial Construction

_Capsule_ definitions are written separately from the _Container_ itself.

You can configure constructor arguments by position, name, or type ...

```php
$def = new Definitions();

$def->{Foo::CLASS}
    ->arguments([
        'baz' => 'zim'
    ]);

$container = new Container($def);

$foo = $container->get(Foo::CLASS);
echo $foo->getBaz(); // zim
```

... or you can take over construction completely using a factory callable:

```php
$def->{Foo::CLASS}
    ->factory(function (Container $container) {
        return new Foo(new Bar(), 'irk');
    });

$container = new Container($def);

$foo = $container->get(Foo::CLASS);
echo $foo->getBaz(); // irk
```

Learn more about [class definitions](/3.x/definitions/classes.html).

## Extended Construction

You can call object methods after construction ...

```php
$def->{Foo::CLASS}
    ->method('setBaz', 'baz-setter');
```

... modify the object yourself via a callable ...

```php
$def->{Foo::CLASS}
    ->modify(function (Container $container, Foo $foo) {
        $foo->doSomething();
    });
```

... or decorate it via another callable, as you see fit:

```php
$def->{Foo::CLASS}
    ->decorate(function (Container $container, Foo $foo) {
        return new DecoratedFoo($foo);
    });
```

These post-construction methods can be combined in any order any number of
times.

Learn more about [class definitions](/3.x/definitions/classes.html).

## Interface Implementation

Binding an interface to an implementation is trivial ...

```php
$def->{FooInterface::CLASS}
    ->class(Foo::CLASS);
```

... or you can specify a factory callable over the interface:

```php
$def->{FooInterface::CLASS}
    ->factory(function (Container $container) {
        return new Foo(new Bar())
    });
```

Learn more about [interface definitions](/3.x/definitions/interfaces.html).


## Lazy Resolution

Anywhere you need a parameter argument, you can use a _Defintions_ method to
specify lazy-resolution for that argument.

For example, reading from the environment at construction ...

```php
$def->{Foo::CLASS}
    ->argument('bar', $def->env('FOO_BAR'));
```

... as well as retrieving shared or new objects from the container, making
calls to those objects after retriving them, making function or static method
calls, invoking any other callable, or capturing the return of an included or required file.

Lazy resolution can be used almost anywhere in a definition, including in
`argument()` and  `method()` calls, as well as for primitive values.

Learn more about [lazy resolution](/3.x/lazy.html).

## Named Objects and Values

Defining named objects and primitive values is trivial:

```php
$def->{'db.host'} = $def->env('DB_HOST');
$def->{'db.user'} = $def->env('DB_USER');
$def->{'db.pass'} = $def->env('DB_PASS');

$def->db = $def->newDefinition(Database::CLASS)
    ->arguments([
        'host' => $def->get('db.host'),
        'user' => $def->get('db.user'),
        'pass' => $def->get('db.pass')
    ]);
```

Learn more about [value definitions](/3.x/definitions/values.html).

## Providing Capsule Definitions

A _Capsule_ container can receive any number of defintion providers, each of
which can be implemented at any layer of your system. Writing a provider
implementation is straightforward:

```php
use Capsule\Di\Provider;
use Capsule\Di\Definitions;

class MyProvider implements Provider
{
    public function provide(Definitions $def) : void
    {
        // work with $def to add definitions
    }
}
```

You can then compose any series of providers as you like, in any iterable you
like, when creating the _Capsule_ container. For example, in production, you
might do this ...

```php
$providers = [
    new DomainProvider(),
    new ApplicationProvider(),
    new InfrastructureProvider(),
    new HttpProvider(),
];

$def = new Definitions();

$container = new Container($def, $providers);
```

... but in testing, you might alternatively compose your providers this way instead:

```php
$providers = [
    new DomainProvider(),
    new ApplicationProvider(),
    new IntegrationTestingProvider(),
];
```

Learn more about [definition providers](/3.x/providers.html).

* * *

Using a previous version of Capsule? [Upgrade!](/3.x/upgrading.html)
