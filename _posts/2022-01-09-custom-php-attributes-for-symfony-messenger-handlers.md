---
layout: post
title:  "Using custom PHP attributes for registering and configuring Symfony Messenger handlers" 
date:   2022-01-09 10:41:00
author: Dejan Angelov
image:  boxes.jpg
categories:
---

Symfony Messenger provides multiple ways for registration and configuration of message handlers. Nowadays, the handlers 
are usually registered by implementing the component's `MessageHandlerInterface` interface, which makes the handlers 
auto-configurable by default, or by implementing some interface of ours which we can register for autoconfiguration 
manually. However, with the addition of the attributes functionality in PHP 8.0, it doesn't really make sense anymore to 
use interfaces for the purpose of "describing" our classes. In this article we'll explore how we can use our own PHP 
attributes to register and configure the message handlers.

The attributes are more suitable for this purpose as, besides describing the classes, they can also serve as simple data 
transfer objects that can hold any data we need to pass. Since 
[Symfony 5.4](https://symfony.com/blog/new-in-symfony-5-4-messenger-improvements#configurable-handlers-with-php-attributes){:target="_blank"}, 
the Messenger component comes with an `#[AsMessageHandler]` attribute that serves as a replacement for the mentioned 
interface, and is great for quickly registering any class as message handler. However, it has a limited set of 
properties and can be declared only on classes.

If we want to have more flexibility, we can create and use our own attributes. With them, we'll have bigger control, and 
we can decouple our classes from the framework. Also, with attributes that can be declared on methods, we can easily 
define multiple handlers in a same class without the need of additional framework code into it, making it more focused 
on our business logic.

Let's begin by saying that whenever a new member joins a community in our application, we're dispatching an 
`MemberJoinedEvent` event:

{% highlight php startinline %}
class MemberJoinedEvent
{
    // ...
}
{% endhighlight %}

<em><small>(I'll be omitting the namespaces here for simplicity. You can organize the code and place the classes 
according to your needs.)</small></em>

The event is being handled by two listeners for taking the following actions:
* Start their trial period
* Send them a welcome e-mail

And we have the following message bus through which we're dispatching all the events:

{% highlight yaml %}
framework:
    messenger:
        buses:
            event_bus: ...
{% endhighlight %}

Now we need to register our events' listeners as services in the Symfony's container, and tag them as message handlers, 
so they can be reached by the dispatched events.

### Registering the handlers for autoconfiguration

As mentioned above, we'll create a new attribute with which we'll mark the listeners as handlers. 
For now, we can specify the classes as the only target of the attribute. Later in the article we'll see how we can use 
the attribute for methods as well.

{% highlight php startinline %}
#[Attribute(Attribute::TARGET_CLASS)]
class Listener
{
}
{% endhighlight %}

Next, we'll declare the attribute on the listener classes:

{% highlight php startinline %}
#[Listener]
final class StartTrialPeriod
{
    public function __invoke(MemberJoinedEvent $event): void { ... }
}
{% endhighlight %}

{% highlight php startinline %}
#[Listener]
final class SendWelcomeEmail
{
    public function __invoke(MemberJoinedEvent $event): void { ... }
}
{% endhighlight %}

As a final step, we need to tell Messenger that these listener classes are message handlers by tagging each of them with 
the `messenger.message_handler` tag.

Similar to the functionality for autoconfiguring services that implement a given interface, since
[Symfony 5.3](https://github.com/symfony/symfony/pull/39897){:target="_blank"} there's an option to register attributes 
for autoconfiguration as well.

For that purpose, there's the `registerAttributeForAutoconfiguration` method in the `ContainerBuilder` class, which 
we'll use to register the attribute when building the container. As arguments to this method we'll be passing the 
classpath of the attribute we want to register, and a configurator - an anonymous function that will be executed for 
each of the attribute's declarations (i.e. for each of the listener classes).

As we want our handlers to work on the `event_bus` message bus, we'll also set the `bus` property on the 
`messenger.message_handler` tag we'll be adding to the listener classes.

{% highlight php startinline %}
class Kernel extends BaseKernel
{
    // ...

    protected function build(ContainerBuilder $container): void
    {
        // ...

        $container->registerAttributeForAutoconfiguration(
            Listener::class,
            static function (
                ChildDefinition $definition,
                Listener $attribute,
                ReflectionClass $reflector
            ): void {
                $definition->addTag('messenger.message_handler', [
                    'bus' => 'event_bus'
                ]);
            }
        );
    }
}
{% endhighlight %}

We can now use the `debug:messenger` console command to check the list of messages and handlers:

{% highlight text %}
MemberJoinedEvent
    handled by SendWelcomeEmail
    handled by StartTrialPeriod
{% endhighlight %}

With the work we've done so far, whenever we declare the `#[Listener]` attribute on a class, the Messenger component
will register that class as a message handler in the `event_bus` message bus. The new class will also be automatically
configured to handle events of the class type-hinted as a first argument in its `__invoke` method.

### Changing the handler method name

Next, let's try to make it possible to use a different method name (eg. `handle`) for handling the events, instead of
`__invoke`.

{% highlight php startinline %}
#[Listener]
final class SendWelcomeEmail
{
    public function handle(MemberJoinedEvent $event): void { ... }
}
{% endhighlight %}

To specify which method should be called when the event is passed to the handler, we need to set the `method` property 
when tagging the handler.

{% highlight php startinline %}
$container->registerAttributeForAutoconfiguration(
    Listener::class,
    static function (
        ...
    ): void {
        $definition->addTag('messenger.message_handler', [
            'bus' => 'event_bus',
            'method' => 'handle'
        ]);
    }
);
{% endhighlight %}

However, if we try running this, we'll get the following error:

> Invalid handler service "SendWelcomeEmail": class "SendWelcomeEmail" must have an "__invoke()" method.<br />
> <sup>500 Internal Server Error - RuntimeException</sup>

At the moment, Messenger will try to resolve the handled message type (in our case the event class) automatically only 
when the method is not specified explicitly. As a result, in our case, we also need to manually specify the handled 
event for each of the registered handlers, using the `handles` property of the tag.

As we will have multiple events, we need to somehow pass the value to the configurator for each of the handlers we want 
to configure.

For that reason, we'll add a new property in the attribute:

{% highlight php startinline %}
#[Attribute(Attribute::TARGET_CLASS)]
class Listener
{
    public function __construct(public readonly string $event) { ... }
}
{% endhighlight %}

And we'll provide the value when declaring the attribute on the listener classes:

{% highlight php startinline %}
#[Listener(event: MemberJoinedEvent::class)]
final class SendWelcomeEmail
{
    public function handle(MemberJoinedEvent $event): void { ... }
}
{% endhighlight %}

Now, when registering the handlers, in the configurator we'll read the `event` property of the attribute instance, and 
we'll use that value for the `handles` property of the tag:

{% highlight php startinline %}
$container->registerAttributeForAutoconfiguration(
    Listener::class,
    static function (
        ChildDefinition $definition,
        Listener $attribute,
        ReflectionClass $reflector
    ): void {
        $definition->addTag('messenger.message_handler', [
            'bus' => 'event_bus',
            'method' => 'handle',
            'handles' => $attribute->event
        ]);
    }
);
{% endhighlight %}

If we check the list of messages and handlers now, we'll see that the listeners will handle the events with the new 
method name:

{% highlight text %}
MemberJoinedEvent
    handled by SendWelcomeEmail (when method=handle)
    handled by StartTrialPeriod (when method=handle)
{% endhighlight %}

### Handling multiple messages (events) in the same class

Now that we're able to use a different method names for the handlers, we can make it more dynamic and also make it 
possible for us to have multiple methods in the same class that will handle different events. By using the attribute, 
we're able to achieve this without implementing any additional interfaces or adding additional methods to our classes.

Let's say that we have another similar `MemberLeftEvent` event, and we want to have some service that will keep track of 
the number of members in a community. In this case, it makes sense to have a class that will contain the listeners for 
both of the events, which will adjust the number respectively.

As we want the listeners in this case to be represented by the methods, we need to declare the `#[Listener]` attribute 
to each of them. First, we'll make it possible for the attribute to be declared on methods too:

{% highlight php startinline %}
#[Attribute(Attribute::TARGET_CLASS | Attribute::TARGET_METHOD)]
class Listener
{
    ...
}
{% endhighlight %}

Next we'll declare the attributes on each of the methods we want to handle an event:

{% highlight php startinline %}
final class CountMembers
{
    #[Listener(event: MemberJoinedEvent::class)]
    public function increaseNumber(MemberJoinedEvent $event): void { ... }

    #[Listener(event: MemberLeftEvent::class)]
    public function decreaseNumber(MemberLeftEvent $event): void { ... }
}
{% endhighlight %}

The third argument in the configurator anonymous function was added in
[Symfony 5.4](https://symfony.com/blog/new-in-symfony-5-4-dependencyinjection-improvements#autoconfigurable-methods-properties-and-parameters){:target="_blank"}, 
and provides us some reflection info about the place (class, method, property, etc.) where the attribute was declared.

As we're allowing our attribute to be declared on classes and methods, we're expecting that argument to be an instance
of either `ReflectionClass` or `ReflectionMethod`, so we can use an union of these two types when type-hinting the 
argument.

For the cases where the attribute is declared on a specific method, we'll fetch the method name from the 
`ReflectionMethod` object, and we'll use it instead of the hardcoded value we had until now. For the cases where the 
attribute is declared on the class, we'll fail-back to the `handle` method as before. If you want to make it more 
flexible for these cases too, you can add another property to the attribute, which you'll set when declaring the 
attribute on the classes.

{% highlight php startinline %}
$container->registerAttributeForAutoconfiguration(
    Listener::class,
    static function (
        ChildDefinition $definition,
        Listener $attribute,
        ReflectionClass | ReflectionMethod $reflector
    ): void {
        $method = ($reflector instanceof ReflectionMethod)
            ? $reflector->getName()
            : 'handle';

        $definition->addTag('messenger.message_handler', [
            'bus' => 'event_bus',
            'method' => $method,
            'handles' => $attribute->event
        ]);
    }
);
{% endhighlight %}

If we check the list now, we'll see both the classes and methods handling the events:

{% highlight text %}
MemberJoinedEvent
    handled by CountMembers (when method=increaseNumber)
    handled by SendWelcomeEmail (when method=handle)
    handled by StartTrialPeriod (when method=handle)

MemberLeftEvent
    handled by CountMembers (when method=decreaseNumber)
{% endhighlight %}

### Passing additional configuration

As we saw before, we own the attribute class, so we can use them to pass any metadata or configuration options we want.

Now, let's say that we have two transports - `sync` for the actions we want to do synchronously, and `async` for the 
actions we want to be done asynchronously in the background.

{% highlight yaml %}
framework:
    messenger:
        # ...

        transports:
             async: '%env(MESSENGER_TRANSPORT_DSN)%'
             sync: 'sync://'
{% endhighlight %}

From the handlers, we'd like the `SendWelcomeEmail` one to be working only in the background, so it'll handle only 
events that will come through the `async` transport.

Instead of specifying the transport inside the listener classes, thus adding more framework code to them, we can add a 
new boolean property in the attribute which will indicate whether we want the concrete handler to be executed 
synchronously or asynchronously.

{% highlight php startinline %}
#[Attribute(Attribute::TARGET_CLASS | Attribute::TARGET_METHOD)]
class Listener
{
    public function __construct(
        public readonly string $event,
        public readonly bool $async = false
    ) {
    }
}
{% endhighlight %}

And we'll update the attribute declaration in the listener:

{% highlight php startinline %}
#[Listener(event: MemberJoinedEvent::class, async: true)]
final class SendWelcomeEmail
{
    public function handle(MemberJoinedEvent $event): void { ... }
}
{% endhighlight %}

In the configurator for the attribute, we can use the `from_transport` property of the tag to register the handlers 
properly:

{% highlight php startinline %}
$container->registerAttributeForAutoconfiguration(
    Listener::class,
    static function (
        ChildDefinition $definition,
        Listener $attribute,
        ReflectionClass | ReflectionMethod $reflector
    ): void {
        $method = ($reflector instanceof ReflectionMethod)
            ? $reflector->getName()
            : 'handle';

        $definition->addTag('messenger.message_handler', [
            'bus' => 'event_bus',
            'method' => $method,
            'handles' => $attribute->event,
            'from_transport' => $attribute->async ? 'async' : 'sync'
        ]);
    }
);
{% endhighlight %}

If we check the list now, we'll see that the welcome e-mail messages will be sent asynchronously, while the rest of the 
listeners will be executed synchronously:

{% highlight text %}
MemberJoinedEvent
    handled by CountMembers (when method=increaseNumber, from_transport=sync)
    handled by SendWelcomeEmail (when method=handle, from_transport=async)
    handled by StartTrialPeriod (when method=handle, from_transport=sync)

MemberLeftEvent
    handled by CountMembers (when method=decreaseNumber, from_transport=sync)
{% endhighlight %}

<em><small>(Note that in this case the same message is being expected from multiple transports, so it'll have to be
routed to all of them.)</small></em>

### Next steps

If we have other busses for another types of messages in our application, we can create more attribute classes - one
for each of them. For example, if we have a command bus too, we can create a `#[CommandHandler]` attribute. That way, 
we'll provide more context, we can specify different configuration options per handler type, and we'll better 
distinguish the different types of message handlers.

<br />
<small><small><small>
    Photo for social media by [Tima Miroshnichenko](https://www.pexels.com/photo/parcels-inside-a-delivery-van-6170458/){:target="_blank"}.
</small></small></small>
