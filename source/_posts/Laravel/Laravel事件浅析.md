---
title: Laravel事件浅析
date: 2021-03-15 10:49:38
tags: 
  - Laravel
  - PHP
categories: Laravel
summary: 解析Laravel中是如何实现观察者模式的
---

## Laravel 事件源码浅析
<!-- more -->
### 一.前言

需要了解以下基础知识：

- Laravel 事件系统：https://learnku.com/docs/laravel/5.5/events/1318
- Facades：https://learnku.com/docs/laravel/5.5/facades/1291
- 服务提供者：https://learnku.com/docs/laravel/5.5/providers/1290
### 二. 监听者注册和事件触发方式

#### 注册监听

1. 在`App\Providers\EventServiceProvider` 的 `$listen` 数组中增加事件类和监听者类的映射。
2. 在`App\Providers\EventServiceProvider` 的 `boot()` 方法中设置基于字符串（事件名）和闭包（监听方法）的监听。
3. 在自定义的订阅者中可以监听多个事件，订阅者需要在 `App\Providers\EventServiceProvider` 的 `$subscribe` 属性中注册。

#### 事件触发

使用辅助方法 `event()` 触发事件。

### 三. 核心事件类的注册

通过追踪 `\App\Providers\EventServiceProvider` 类，发现其继承 `Illuminate\Foundation\Support\Providers\EventServiceProvider`

`Illuminate\Foundation\Support\Providers\EventServiceProvider` 类通过调用 `Event` 门面的 `listen()` 和 `subscribe()`方法完成对事件的订阅。

```php
public function boot()
{
    foreach ($this->listens() as $event => $listeners) {
        foreach ($listeners as $listener) {
            Event::listen($event, $listener);
        }
    }

    foreach ($this->subscribe as $subscriber) {
        Event::subscribe($subscriber);
    }
}
```

##### Event 门面核心代码如下：

```php
/**
 * @see \Illuminate\Events\Dispatcher
 */
class Event extends Facade
{
  	//........
    /**
     * Get the registered name of the component.
     *
     * @return string
     */
    protected static function getFacadeAccessor()
    {
        return 'events';
    }
}
```

这里的 `events` 就是 Laravel 的事件监听器， 注册于 `\Illuminate\Foundation\Application:193`:

```php
protected function registerBaseServiceProviders()
{
    $this->register(new EventServiceProvider($this));
}
```

`\Illuminate\Events\EventServiceProvider` :

```php
class EventServiceProvider extends ServiceProvider
{
    /**
     * Register the service provider.
     *
     * @return void
     */
    public function register()
    {
        $this->app->singleton('events', function ($app) {
            return (new Dispatcher($app))->setQueueResolver(function () use ($app) {
                return $app->make(QueueFactoryContract::class);
            });
        });
    }
}
```

去除掉绑定队列实现的操作，核心代码如下：

```php
$this->app->singleton('events', function ($app) {
    return (new Dispatcher($app));
});
```

可见，事件服务的核心类就是 `\Illuminate\Events\Dispatcher` 。

### 四. 事件监听的注册

结合上边 `Illuminate\Foundation\Support\Providers\EventServiceProvider` 的内容:

```php
foreach ($this->listens() as $event => $listeners) {
    foreach ($listeners as $listener) {
      	Event::listen($event, $listener);
    }
}

foreach ($this->subscribe as $subscriber) {
    Event::subscribe($subscriber);
}
```

可以看出事件的监听是依赖 `Dispatcher` 的 `listen()` 方法 和 `subscribe()` 方法实现的：

```php
public function listen($events, $listener)
{
    foreach ((array) $events as $event) {
        if (Str::contains($event, '*')) {
          	$this->setupWildcardListen($event, $listener);
        } else {
          	$this->listeners[$event][] = $this->makeListener($listener);
        }
    }
}


public function subscribe($subscriber)
{
    $subscriber = $this->resolveSubscriber($subscriber);

    $subscriber->subscribe($this);
}
```

根据监听事件的模式不同，储存至不同的属性中：

1. 当 事件名 中包含 `*` 时，认为是 通配符事件监听，将会把事件监听映射存储至 `$wildcards` 属性。

```php
protected function setupWildcardListen($event, $listener)
{
    $this->wildcards[$event][] = $this->makeListener($listener, true);
}
```

2. 否则会将事件监听映射存储至 `$listeners` 属性。

此处通过 `makeListener` 解析出监听器：

```php
/**
 * Register an event listener with the dispatcher.
 *
 * @param  \Closure|string  $listener
 * @param  bool  $wildcard
 * @return \Closure
 */
public function makeListener($listener, $wildcard = false)
{
    if (is_string($listener)) {
        return $this->createClassListener($listener, $wildcard);
    }

    return function ($event, $payload) use ($listener, $wildcard) {
        if ($wildcard) {
            return $listener($event, $payload);
        }

      return $listener(...array_values($payload));
    };
}
```

如果 `$listener` 是字符串 ：那么会通过`createClassListener` 来 将监听方法解析为闭包

如果 `$listener` 是闭包，那么也会封装为统一的闭包

`createClassListener` :

```php
/**
 * Create a class based listener using the IoC container.
 *
 * @param  string  $listener
 * @param  bool  $wildcard
 * @return \Closure
 */
public function createClassListener($listener, $wildcard = false)
{
    return function ($event, $payload) use ($listener, $wildcard) {
        if ($wildcard) {
            return call_user_func($this->createClassCallable($listener), $event, $payload);
        }

        return call_user_func_array(
            $this->createClassCallable($listener), $payload
        );
    };
}
```

通过 `createClassCallable` 从服务容器中解析出 对应的类方法:

```php
/**
 * Create the class based event callable.
 *
 * @param  string  $listener
 * @return callable
 */
protected function createClassCallable($listener)
{
    list($class, $method) = $this->parseClassCallable($listener);
		/**队列代码  忽略
    if ($this->handlerShouldBeQueued($class)) {
        return $this->createQueuedHandlerCallable($class, $method);
    }
		**/
    return [$this->container->make($class), $method];
}

protected function parseClassCallable($listener)
{
    return Str::parseCallback($listener, 'handle');
}

public static function parseCallback($callback, $default = null)
{
  	return static::contains($callback, '@') ? explode('@', $callback, 2) : [$callback, $default];
}
```

`subscribe()` 方法的实现依赖于 `listen()`方法

```php
public function subscribe($subscriber)
{
    $subscriber = $this->resolveSubscriber($subscriber);

    $subscriber->subscribe($this);
}

protected function resolveSubscriber($subscriber)
{
    if (is_string($subscriber)) {
        return $this->container->make($subscriber);
    }

    return $subscriber;
}

/**
自定义的订阅者 subscribe() 方法 示例
*/
public function subscribe($events)
{
  //接口响应事件
  $events->listen(
    RequestHandled::class,
    'App\Listeners\AlertEventSubscriber@onApiResponse'
  );

  //提交服务单事件
  $events->listen(
    CommitWorkOrder::class,
    'App\Listeners\AlertEventSubscriber@onCommitInsurance'
  );
}
```

### 四. 事件触发

`event()` 方法实现：

```php
/**
* Dispatch an event and call the listeners.
*
* @param  string|object  $event
* @param  mixed  $payload
* @param  bool  $halt
* @return array|null
*/
function event(...$args)
{
		return app('events')->dispatch(...$args);
}
```

`dispatch()` 方法：

```php
/**
* Fire an event and call the listeners.
*
* @param  string|object  $event
* @param  mixed  $payload
* @param  bool  $halt
* @return array|null
*/
public function dispatch($event, $payload = [], $halt = false)
{
    //当给定的“事件”实际上是一个对象时，我们将假定它是一个事件
    //对象，并使用类作为事件名称，并使用该事件本身作为处理程序的有效负载，这使基于对象的事件非常简单
    list($event, $payload) = $this->parseEventAndPayload(
        $event, $payload
    );

    if ($this->shouldBroadcast($payload)) {
        $this->broadcastEvent($payload[0]);
    }

    $responses = [];

    foreach ($this->getListeners($event) as $listener) {
        $response = $listener($event, $payload);

        //如果侦听器返回了响应，并且启用了事件暂停功能，
        //我们将仅返回此响应，而不会调用事件的其余部分
        //。否则，我们会将响应添加到响应列表中。
        if ($halt && ! is_null($response)) {
            return $response;
        }

        //如果侦听器返回了布尔值false，我们将停止将该事件传播到链中的其他侦听器，否则我们将继续
        //遍历侦听器并触发序列中的每个侦听器。
        if ($response === false) {
            break;
        }

        $responses[] = $response;
    }

    return $halt ? null : $responses;
}
```

这里通过 `parseEventAndPayload()` 解析出 事件和参数，如果是对象事件，将对象的类名作为事件名，事件本身作为参数

```php
protected function parseEventAndPayload($event, $payload)
{
  	if (is_object($event)) {
    	list($payload, $event) = [[$event], get_class($event)];
  	}

  	return [$event, Arr::wrap($payload)];
}
```

`getListeners()` 方法：

```php
public function getListeners($eventName)
{
    $listeners = $this->listeners[$eventName] ?? [];

    $listeners = array_merge(
        $listeners, $this->getWildcardListeners($eventName)
    );

    return class_exists($eventName, false)
    						? $this->addInterfaceListeners($eventName, $listeners)
    						: $listeners;
}

protected function getWildcardListeners($eventName)
{
    $wildcards = [];

    foreach ($this->wildcards as $key => $listeners) {
        if (Str::is($key, $eventName)) {
            $wildcards = array_merge($wildcards, $listeners);
        }
    }

    return $wildcards;
}

/**
* 基于接口的事件冒泡
* Add the listeners for the event's interfaces to the given array.
*
* @param  string  $eventName
* @param  array  $listeners
* @return array
*/
protected function addInterfaceListeners($eventName, array $listeners = [])
{
    foreach (class_implements($eventName) as $interface) {
        if (isset($this->listeners[$interface])) {
            foreach ($this->listeners[$interface] as $names) {
                $listeners = array_merge($listeners, (array) $names);
            }
        }
    }

    return $listeners;
}
```
