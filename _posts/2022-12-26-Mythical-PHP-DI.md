---
layout: post
title: Mythical PHP DI
categories: [PHP, DI, Idea]
---

What if PHP DI[^1] will look like this?:

```php
class Application
{
  public function __construct(
    private readonly CacheInterface $cache,
    private readonly LoggerInterface $logger,
  ) {
  }
...
}

$object = new Application(cache: new DbCache());
```

You don't need to use container **explicitly** like `$container->create(...` instead of that you can use operator `new`[^2] as native PHP construction.

---


## Beginnings

I am _overwhelmed_ to use divers PHP DI tools. Each of them have some individualities, however we all know what they do under the hood - they inject dependencies.
It doesn't matter what DI you use, basically - if you need create class by hand - you cannot just write `new ClassName()` because the class might have additional **dependencies**.
With codebase expansion you more often find some strange construction in the same class (or in the method even):

```php
public function methodName(string $iAmParameter): void
{
  $object = Yii::createObject(CreatedByDI::class, ['param' => $iAmParameter]);
  ...
  $entity = new SimpleEntity($object->getRequiredParameterForSimpleEntity());
  ...
}
```

Maybe, after stumbling a couple times to that _misconception_, you decided to use only DI construction.
And it is good.

```php
public function methodName(string $iAmParameter): void
{
  $object = Yii::createObject(CreatedByDI::class, ['param' => $iAmParameter]);
  ...
  $entity = Yii::createObject(SimpleEntity::class, ['param' => $object->getRequiredParameterForSimpleEntity()]);
  ...
}
```

However do you really excited of the `Yii::createObject(`?

> _(it's just a coincidence that I used Yii[^3] for example)_

I don't.


## Continuing

Some of the DIs are really *rich*[^4] and help you create factories on the fly, or put object into any method/function without additional string of code.
Nevertheless, on practical usage it is not really what I want in my business code (except configuration), I need a simplicity. One mechanism to create an object without rethought all dependencies.

Surprisingly, PHP already have it - `new` operator.
Let's edit the example above:

```php
public function methodName(string $iAmParameter): void
{
  $object = new CreatedByDI(param: $iAmParameter);
  ...
  $entity = new SimpleEntity($object->getRequiredParameterForSimpleEntity());
  ...
}
```

The code is straight and solid, however the power of `new` operator with DI abilities appears in the deep.


## The Deep

Let's imagine you are using framework with DI and have a huge codebase that includes many different modules and one monolith with bussiness logic as glue between modules installed by composer.

⚠️ I'm going to use `$container` variable like alias to any DI you know.

ℹ️ Yes, maybe the examples isn't really good, however you can find better in a real codebase.


```php
// Order module
class OrderConvertService implements OrderConvertServiceInterface
{
  public function __construct(
    private readonly UserConvertServiceInterface $userService,
    private readonly ProductConvertServiceInterface $productService,
    private readonly DeliveryConvertServiceInterface $deliveryService,
  ) {
  }

  public function convert(OrderEntity $order): OrderDto
  {
    return $container->get(OrderDto::class, [
      'id' => $order->getId(),
      'date' => $order->getDate(),
      'status' => $order->getStatus(),
      'user' => $this->userService->convert($order->user ?? $container->get(AnonymousUserEntity::class)),
      'products' => $this->productService->convert($order->products),
      'delivery' => $this->deliveryService->convert($order->delivery),
    ]);
  }
}

...

// Global order event
class EventOrderCreated implements EventInterface
{
  public function __construct(
    private readonly OrderDto $order,
  ) {
  }
  
  public function getOrder(): OrderDto
  {
    return $this->order;
  }
}

...

// Mail module
class OrderCreatedListener implements ListenerInterface
{
  public function __construct(
    private readonly SendComponentInterface $sendComponent,
  ) {
  }

  public function handle(EventInterface $event): void
  {
    $order = $event->getOrder();
    if ($order->getUser() instanceof AnonymousUserEntity::class) {
      $phoneComponent = $container->get(PhoneSendComponent::class, [
        // some params here, but nobody know what exactly
      ]);
      $phoneComponent->notifyClient($order);
    
      return;
    }
  
    $this->sendComponent->notifyClient($order);
  }
}

...

// DI config
return [
  OrderConvertServiceInterface::class => OrderConvertService::class,
  UserConvertServiceInterface::class => UserConvertService::class,
  ProductConvertServiceInterface::class => ProductConvertService::class,
  DeliveryConvertServiceInterface::class => DeliveryConvertService::class,
  AnonymousUserEntity::class => static fn(ContainerInterface $c) => $c->create(AnonymousUserEntity::class, [
    'phone' => $c->get(OrderSessionInterface::class)->getPhone(),
    'name' => $c->get(OrderSessionInterface::class)->getName(),
  ]),
  OrderCreatedListener::class => static fn(ContainerInterface $c) => $c->get(OrderCreatedListener::class, [
    'sendComponent' => $c->get(MailSendComponent::class),
  ]),
];
```

First of all:

```php
return $container->get(OrderDto::class, [ ...
```

can be changed into `new OrderDto(` because DTO objects should be simple, however in the name of convenience developers often put inherritance here as well.
Consequently, if we use `new` with DI the line will turn into :

```php
return new OrderDto(
  id: $order->getId(),
  date: $order->getDate(),
  status: $order->getStatus(),
  user: $this->userService->convert($order->user ?? new AnonymousUserEntity()),
  products: $this->productService->convert($order->products),
  delivery: $this->deliveryService->convert($order->delivery),
);
```

Of course, `new AnonymousUserEntity()` should be created by some _factory_, in that case we can use `$config` settings as well, even with `new` operator.

Therefore *Listener* will look like this:

```php
class OrderCreatedListener implements ListenerInterface
{
  public function handle(EventInterface $event): void
  {
    $order = $event->getOrder();
    if ($order->getUser() instanceof AnonymousUserEntity::class) {
      (new PhoneSendComponent())->notifyClient($order);
    
      return;
    }
  
    (new MailSendComponent())->notifyClient($order);
  }
}
```

Hence, we get PHP native `new` operator that works like DI and can use usually `$config` DI settings for classes injection.


## Implementation

I don't have a work example, it is only just an idea that looks interesting for me.
However I made a couple experiments.

Let's imagine we have simple app, where directory `src` under `App` namespace:

```
.
├── .gitignore
├── composer.json
├── composer.lock
├── src
│   ├── Cache
│   │   └── ArrayCache.php
│   ├── Config
│   │   └── config.php
│   ├── Interfaces
│   │   ├── ApplicationInterface.php
│   │   ├── CacheInterface.php
│   │   └── LoggerInterface.php
│   ├── Logger
│   │   └── ConsoleLogger.php
│   └── Application.php
├── vendor
│   ├── di
│   ├── ... other packages ...
│   └── autoload.php
├── tmp
│   └── .gitignore
└── index.php
```

### *config.php*:

```php
return [
  CacheInterface::class => ArrayCache::class,
  LoggerInterface::class => ConsoleLogger::class,
  ArrayCache::class => static fn(ContainerInterface $c) => $c->singletone(ArrayCache::class),
];
```

### *ArrayCache.php*

```php
class ArrayCache implements CacheInterface
{
  private array $data = [];

  public function set(string $key, mixed $value): void
  {
    $this->data[$key] = $value;
  }
  
  public function get(string $key): mixed
  {
    return $this->data[$key];
  }
}
```

### *ConsoleLogger.php*

```php
class ConsoleLogger implement LoggerInterface
{
  public function log(string $message)
  {
    echo $message, PHP_EOL;
  }
}
```

### *Application.php*

```php
class Application implements ApplicationInterface
{
  public function _construct(
    private readonly CacheInterface $cache,
    private readonly LoggerInterface $logger,
  ) {
  }
  
  public function run(): int
  {
    $this->logger->log("Start Application");
    
    $this->logger->log($this->cache->get('message'));
    
    $this->logger->log("End Application");
    
    return 0;
  }
}
```

### *index.php*

```php
require __DIR__ '/vendor/autoload.php';

$cache = new ArrayCache();
$cache->set('message', $argv[1] ?? '--empty--');

$app = new Application();
if (0 !== $app->run()) {
  echo "ERROR!";
}
```

In the case when we use `new ArrayCache();` we **already** use DI and object will be created even it has _dependencies_, as well as `$app = new Application();` will be created with **default** cache and logger object.
In any line of code we can change our logger:

```php
...
$app = new Application(
  logger: new DbLogger(),
);
...
```

Or use `$config` **settings** like in any other DI:

```php
...
Application::class => static fn(ApplicationParams $p) => $p->isCli() ? new Application() : new Application(logger: new DbLogger()),
...
```

In our case `ContainerInterface` it is a **wrapper** for method like: `singletone` or some kind of `factory` if we need it in configuration file.
In other cases operator `new` is the really one solution for the code.

The idea is that we register _custom autoload function_[^5]:

```php
spl_autoload_register(static function (string $class) {
  $diNamespacePrefix = "DI\\";
  $diDirectory = "tmp/DI";
  $fileName = $diDirectory . DIRECTORY_SEPARATOR . str_replace("\\", DIRECTORY_SEPARATOR, $class) . '.php';

  if (!str_starts_with($class, $diNamespacePrefix) && file_exists($fileName)) {
    include $fileName;
  } else {
    include $class;
  }
});
```

So we literally, check if we can load _substitution_ class instead of specific in application or we just load **native** class instead.

ℹ️ This approach isn't multipurposes so we can load same _Flyweight_ class for each **native** class so _Flyweight_ will return **native** class in the runtime.

For example, this is Application's class wrapper:

```php
namespace \DI\Application;

class Application implements ApplicationInterface
{
  public function _construct(
    private readonly ?CacheInterface $cache = null,
    private readonly ?LoggerInterface $logger = null,
  ) {
  }
  
  // Or use magic method `__call`
  public function run(): int
  {
    return $globalContainer->create(\App\Application)->run();
  }
}
```

However it doesn't work now (while I'm writing the article). There are couple reasons:

1. Default composer loader override our loader or vise-versa.
2. We cannot distinguish from what namespace the class is loading, `spl_autoload_register` callable receives only classname, so when we load `\App\Application` from `DI` namespace the loader loads `\DI\Application` instead **original** recusrively.


## Summary

In my opinion it is interesting **idea** migrate from some additional tools, like DI, to `new`.
Of course is isn't super goal or something like a revolution, for PHP's sake.
It is just **an opinion** and **thinkings** over DI situation, because - as you saw - we still use DI under the `new`, and it may be written differently under the hood, the article more about how PHP class loader may works and gives us an ability to manipulate with classes.
On the other hand, somebody may has an opinion that `new` must stay _pure_ as it is, without any unusual behaviors and they will _right_.

Everyone chooses their side.

And as always: _it is all about fun_.

---

[^1]: [DI](https://en.wikipedia.org/wiki/Dependency_injection)
[^2]: [`new`](https://www.php.net/manual/en/language.oop5.basic.php#language.oop5.basic.new)
[^3]: [Yii2](https://www.yiiframework.com/doc/guide/2.0/en/concept-di-container)
[^4]: [PHP-DI](https://php-di.org/)
[^5]: [spl_autoload_register](https://www.php.net/manual/en/function.spl-autoload-register.php)
