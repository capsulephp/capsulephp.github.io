<div style="text-align: center;">
  <img style="border: none; width: 128px; height: 128px;" src="/capsule.png" />
</div>

Most dependency injection containers work through public configuration, are
intended for use at the application level, and use "stringly"-typed retrieval
methods.

**Capsule**, on the other hand, is intended for use at the library, module,
component, or package level; has no public methods for configuration; and
encourages typehinted retrieval methods.

This means that you `composer require capsule/di` into your package, extend
the `AbstractContainer` for your library or module, and `init()`-ialize your
component objects inside that extended container. You then add typehinted
public methods for object retrieval.

## An Example

The following container provides a shared instance of a hypothetical data mapper
using a shared instance of PDO.

{% highlight php %}
class MyCapsule extends \Capsule\Di\AbstractContainer
{
    protected function init()
    {
        parent::init();

        $this->provide(\PDO::CLASS)->args(
            $this->env('DB_DSN'),
            $this->env('DB_USERNAME'),
            $this->env('DB_PASSWORD')
        );

        $this->provide(MyDataMapper::CLASS)->args(
            $this->service(\PDO::CLASS)
        );
    }

    public function getMyDataMapper() : MyDataMapper
    {
        return $this->serviceInstance(MyDataMapper::CLASS);
    }
}

class MyDataMapper
{
    public function __construct(PDO $pdo) { ... }
}

$capsule = new MyCapsule([
    'DB_DSN' => 'mysql:host=localhost;dbname=mydb',
    'DB_USERNAME' => 'my_user',
    'DB_PASSWORD' => 'my_pass',
]);

$mapper = $capsule->getMyDataMapper(); // instanceof MyDataMapper
{% endhighlight %}

## Initialization Methods

Call these methods within `init()` to configure the container. Note that
they are all protected; they cannot be called from outside the container.

### default(*string* $class) : *Config*

Returns a Config object (see below) so you can set the default constructor
arguments, post-instantiation method calls, and instantiation factory for a
particular class.  All object instantiations will use this configuration by
default.

{% highlight php %}
protected function init()
{
    // ...
    $this->default(Foo::CLASS)->args(
        'foo',
        'bar',
        'baz',
    );
}
{% endhighlight %}

### provide(*string* $id, *LazyInterface* $lazy = null) : *?LazyNew*

Use this to provide a shared instance of a class that will be reused after it is
instantiated.

By default, the `provide()` method presumes the service `$id` is a class name,
and the shared instance is registered under that class name.

The returned LazyNew object extends Config, so you can further configure the
object prior to its instantiation, overriding the class defaults.

{% highlight php %}
protected function init()
{
    // ...

    // uses default arguments
    $this->provide(Foo::CLASS);

    // overrides default arguments
    $this->provide(Bar::CLASS)->args(
        'zim',
        'dib',
        'gir',
    );
}
{% endhighlight %}

Use this to provide a shared instance of a class that will be reused after it is
instantiated. The shared instance is registered under the `$id`, which may be
any class name or any other identifying string.

If you pass a `$lazy` argument to `provide()`, the `$lazy` will be used to
create the service instance, in which case the `$id` may or may not match the
class of the lazy-loaded instance. When passing an explicit `$lazy`, `provide()`
will return `null`.

### service(*string* $id) : *LazyService*

Use this to specify that a dependency should should be a shared service instance
registered using `provide()`. (The service instance may not be defined yet.)

{% highlight php %}
protected function init()
{
    // ...

    $this->default(Foo::CLASS)->args(
        $this->service(Bar::CLASS)
    );
}
{% endhighlight %}

### serviceCall(*string* $id, *string* $func, ...$args) : *LazyCall*

Use this to specify that a dependency should be the result of a method call to
a shared service instance.

{% highlight php %}
protected function init()
{
    // ...

    // provide a MapperLocator
    $this->provide(MapperLocator::CLASS);

    // for the Wiki application service, get the WikiMapper
    // out of the MapperLocator
    $this->default(WikiService::CLASS)->args(
        $this->serviceCall(MapperLocator::CLASS, 'get', WikiMapper::CLASS)
    );
}
{% endhighlight %}

### new(*string* $class) : *LazyNew*

Use this to specify that a dependency should be a **new** instance of a class,
not a shared instance.

{% highlight php %}
protected function init()
{
    // ...

    $this->default(Service::CLASS)->args(
        $this->new(SupportingObject::CLASS)
    );
}
{% endhighlight %}

The returned LazyNew object extends Config, so you can further configure the
object prior to its instantiation, overriding the class defaults.

### call(*callable* $func, ...$args) : *LazyCall*

Use this to specify that a dependency should be the results of invoking a
callable, whether an instance method, static method, or function. (The PHP
keywords `include` and `require` are also supported.)

{% highlight php %}
protected function init()
{
    // ...

    $this->default(Service::CLASS)->args(
        $this->call('include', '/path/to/file.php')
    );
}
{% endhighlight %}

### closure(*string* $func, ...$args) : *\Closure*

This returns a Closure that makes a call to a Capsule container method. It is
useful for providing new-instance and service-instance callables to other
containers, locators, registries, and factories. It will not be invoked as a
lazy-loaded dependency at instantiation time.


{% highlight php %}
protected function init()
{
    // ...

    // presuming the the MapperLocator takes an array of callable factories
    // as its first argument
    $this->default(MapperLocator::CLASS)->args(
        [
            $this->closure('newInstance', ThreadMapper::CLASS),
            $this->closure('newInstance', AuthorMapperMapper::CLASS),
            $this->closure('newInstance', ReplyMapper::CLASS),
        ]
    );
}
{% endhighlight %}

### env(*string* $key) : *mixed*

Returns the value of the `$env[$key]` property (if it is set), then the value of
`getenv($id)` (if it is not `false`), and then `null`. (The `$env` property
values are populated at `__construct()` time.)

{% highlight php %}
protected function init()
{
    // ...
    $this->provide(\PDO::CLASS)->args(
        $this->env('DB_DSN'),
        $this->env('DB_USERNAME'),
        $this->env('DB_PASSWORD')
    );
}
{% endhighlight %}

### alias(*string* $from, *string* $to) : *void*

Use this to alias a class `$from` its original name `$to` another name. All
requests for `$from` will receive an instance of `$to`.

This can be useful for specifying default implementations for interfaces.

{% highlight php %}
$this->alias('FooInterface', 'FooImplementation');
$instance = $this->newInstance('FooInterface'); // instanceof FooImplementation
{% endhighlight %}

## Kickoff Methods

These methods return an actual instance of the requested class, thus invoking
all the lazy-loading configurations for that class and its dependency.

These are protected methods, and should be called from typehinted
methods on your extended container class to provide objects from your library,
package, module, or component.

### newInstance(*string* $class, ...$args) : *mixed*

This returns a new instance of the specified class, with additional arguments
that override the default for that class. Multiple calls to `newInstance()`
return different new instances.

{% highlight php %}
class MyCapsule extends \Capsule\Di\AbstractContainer
{
    public function newPdo() : PDO
    {
        return $this->newInstance(PDO::CLASS);
    }
}
{% endhighlight %}

### serviceInstance(*string* $id) : *mixed*

This returns a shared service instance from the Capsule. Multiple calls to
`serviceInstance()` return the same instance.

{% highlight php %}
class MyCapsule
{
    public function getPdo() : PDO
    {
        return $this->serviceInstance(PDO::CLASS);
    }
}
{% endhighlight %}

## Config Object Methods

These fluent methods are available on the return of `default()`, `new()`, and
`provide()`. They allow you to further configure the class specified by those
methods.

### args(...$args) : *self*

Sets the constructor arguments for the class. These are sequential; lazy-loaded
dependencies will be invoked when the object is actually created. Multiple
calls to `args()` will **reset and replace** the previous constructor arguments.

To reset the args, call `resetArgs()`.

### call(*string* $func, ...$args) : *self*

Adds one post-instantiation method call on the object. This is good for setter
injection, and for post-instantiation setup methods. Multiple calls to `call()`
will **add to** the previous calls.

To reset the calls, use `resetCalls()`.

### factory(*callable* $factory) : *self*

Sets a custom factory callable to use for instantiating the object. This factory
will be used **in place of** the Capsule factory. The `$args` values will
be passed to the custom factory when it is called, and the `$calls` will be made
on the object returned from the custom factory.

To reset the factory so that Capsule factory is used, call `resetFactory()`.

## Automated Configuration

If you do not define the `default()` configuration for a class, the Capsule
will reflect on the requested class instance and examine its constructor
arguments. The Capsule will then populate each argument with:

- a shared service instance with a matching class name, if one has been
  `provide()`-ed; or,
- a new instance of that class

Non-class typehints cannot be configured automatically.

## Implementing PSR-11

PSR-11 is a "stringly"-typed interface. The Capsule container does not implement
it by default, but you can do so easily.

{% highlight php %}
class Psr11Capsule implements \Psr\Container\ContainerInterface
{
    public function get($id)
    {
        return $this->getRegistry()->get($id);
    }

    public function has($id)
    {
        return $this->getRegistry()->has($id);
    }
}
{% endhighlight %}

## Serving From Other Containers

TBD.

## Providing To Other Containers

TBD.
