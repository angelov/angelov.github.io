---
layout: post
title:  "Experimental: Unpacking message properties as handler arguments in Symfony Messenger" 
date:   2022-05-07 14:29:00
author: Dejan Angelov
image:  boxes-2.jpg
categories:
---

In my [previous post](/2022/01/09/custom-php-attributes-for-symfony-messenger-handlers.html), we saw that by using 
custom PHP attributes, we can have our Symfony Messenger message handlers placed in any service in our application. The 
only requirement for the message handlers is for them to be methods that are able to receive the message object as an 
argument. In this post we will see how we can avoid that requirement, and register any method as a message handler, by 
automatically passing any of the message's properties as values for the handler's arguments.

<div class="alert alert-danger">
    <small>
	    <strong>Warning:</strong> I got this idea while working on a hobby project that will never go to production. I 
        haven't tried it in a real-world project and haven't really thought of the long-term consequences of it.
        <br />
        My advice is to use what you'll read here only as a basis for future experimentation, and not as a final 
        solution.
    </small>
</div>

We can start by 
[declaring two message busses](https://symfony.com/doc/6.0/messenger/multiple_buses.html){:target="_blank"} in our 
application, called `event_bus` and `command_bus`:

{% highlight yaml %}
framework:
    messenger:
        buses:
            event_bus: ...
            command_bus: ...
{% endhighlight %}

Imagine that we have a store where people can buy and sell books. Whenever somebody buys a book, we'll be dispatching 
the following domain event:

{% highlight php startinline %}
class BookWasPurchased
{
    public function __construct(
        public readonly string $title,
        public readonly string $customerName,
        public readonly DateTimeInterface $purchasedAt
    ) {
    }
}
{% endhighlight %}

We have a `PurchaseHistory` service that keeps track who has bought which book, so we want its `recordPurchase`, which 
requires the title of the book and the name of the customer, to be called whenever this event occurs.

{% highlight php startinline %}
class PurchaseHistory
{
    public function recordPurchase(string $bookTitle, string $customerName): void
    {
        dd(sprintf(
            "%s bought the \"%s\" book.",
            $customerName,
            $bookTitle
        ));
    }
}
{% endhighlight %}

To achieve that, we need to register a listener that will be receiving and handling these events:

{% highlight php startinline %}
class RecordBookPurchase
{
    private PurchaseHistory $purchaseHistory;

    public function __construct(PurchaseHistory $purchaseHistory)
    {
        $this->purchaseHistory = $purchaseHistory;
    }

    #[Listener(event: BookWasPurchased::class)]
    public function handle(BookWasPurchased $event): void
    {
        $this->purchaseHistory->recordPurchase(
            $event->title,
            $event->customerName
        );
    }
}
{% endhighlight %}

As we can see here, we're not doing much in this listener besides getting the needed information from the event's 
properties and passing it to the service where the actual recording will happen.

Let's see how we can avoid having to create separate classes/methods for this type of listeners, and just register the 
actual service's method as a message handler (an event listener in our case) instead.

As a first step, we'll remove the `RecordBookPurchase` class and move the listener attribute declaration to the 
`PurchaseHistory::recordPurchase` method:

{% highlight php startinline %}
class PurchaseHistory
{
    #[Listener(event: BookWasPurchased::class)]
    public function recordPurchase(string $bookTitle, string $customerName): void
    {
        // ...
    }
}
{% endhighlight %}

And then dispatch an instance of the event:

{% highlight php startinline %}
$eventBus->dispatch(
    new BookWasPurchased('Example Book', 'John Smith', $now)
);
{% endhighlight %}

As expected, the handling of the event will fail because we broke the compatibility - the handler is expecting multiple
string arguments, but it receives an `BookWasPurchased` instance instead.

> Handling "BookWasPurchased" failed: PurchaseHistory::recordPurchase(): Argument #1 ($bookTitle) must be of type 
> string, BookWasPurchased given<br />
> <sup>TypeError > HandlerFailedException</sup>

So, we need to find a way and build some mechanism that will transform the data before it arrives to the handler.

### Replacing the handlers locator

When a message is dispatched to a Symfony Messenger message bus, it goes through a 
[list of middleware](https://symfony.com/doc/6.0/messenger.html#middleware){:target="_blank"} before eventually being 
handled by a handler. The last middleware in the list, `\Symfony\Component\Messenger\Middleware\HandleMessageMiddleware`, 
is the one that calls the appropriate handler(s).

This middleware uses a service that implements the `\Symfony\Component\Messenger\Handler\HandlersLocatorInterface` 
interface to find all the registered handlers for the current dispatched message.

If we take a look at that interface, its `getHandlers` method expects the message wrapped in an Envelope (that's how 
it's already being received in middleware) and returns an iterable that consists of 
`\Symfony\Component\Messenger\Handler\HandlerDescriptor` objects (we'll call them handler descriptors). Each of these 
handler descriptors wraps a found handler, as well as some additional metadata related to it.

{% highlight php startinline %}
interface HandlersLocatorInterface
{
    /**
    * Returns the handlers for the given message name.
    *
    * @return iterable<int, HandlerDescriptor>
    */
    public function getHandlers(Envelope $envelope): iterable;
}
{% endhighlight %}

Within the middleware, from each of the handler descriptors, the actual handler is being retrieved and called with the 
message object as an argument.

{% highlight php startinline %}
// from the \Symfony\Component\Messenger\Middleware\HandleMessageMiddleware class

foreach ($this->handlersLocator->getHandlers($envelope) as $handlerDescriptor) {
    // ...

    $handler = $handlerDescriptor->getHandler();

    // ...
    
    $result = $handler($message);
                
    // ...
}
{% endhighlight %}

If we take a look at the handler descriptor for the handler we defined here, we can see that it has the handler as a 
closure, with both the string arguments.

{% highlight php startinline %}
Symfony\Component\Messenger\Handler\HandlerDescriptor {#618 ▼
    -handler: PurchaseHistory::recordPurchase(string $bookTitle, string $customerName): void {#620 ▶}
    -name: "PurchaseHistory::recordPurchase"
    -batchHandler: null
    -options: array:1 [▼
        "method" => "recordPurchase"
    ]
}
{% endhighlight %}

To make it possible for the middleware to call our method as a handler, we can wrap it in another closure with the 
message as an argument, or with no arguments at all.

To do that, we will create a new implementation of the locator interface which will decorate the existing one provided 
by the component, and which will wrap the found handlers if needed.

For now, let's only create the decorator and return any result that we'll get from the decorated class.

{% highlight php startinline %}
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Handler\HandlersLocatorInterface;

class WrappingHandlersLocator implements HandlersLocatorInterface
{
    private HandlersLocatorInterface $decorated;

    public function __construct(HandlersLocatorInterface $decorated)
    {
        $this->decorated = $decorated;
    }

    public function getHandlers(Envelope $envelope): iterable
    {
        return $this->decorated->getHandlers($envelope);
    }
}
{% endhighlight %}

Next, we need to make the middleware use our decorator as a locator instead of the component's one (that we're 
decorating).

The Messenger component uses the `\Symfony\Component\Messenger\DependencyInjection\MessengerPass` compiler pass to 
register its internal services in the application's service container. There, we can see that it will register separate 
services of both the middleware and the handlers locator for each of the message busses that we have:

{% highlight php startinline %}
// from the \Symfony\Component\Messenger\DependencyInjection\MessengerPass class

// ...

foreach ($busIds as $bus) {
    $container->register($locatorId = $bus.'.messenger.handlers_locator', HandlersLocator::class)
        ->setArgument(0, $handlersLocatorMappingByBus[$bus] ?? [])
    ;

    if ($container->has($handleMessageId = $bus.'.middleware.handle_message')) {
        $container->getDefinition($handleMessageId)
            ->replaceArgument(0, new Reference($locatorId))
        ;
    }
}

// ...
{% endhighlight %}

As we created two message busses at the beginning of the article (`command_bus` and `event_bus`), we can confirm that we 
have two services for the handlers locator in the container:

{% highlight text %}
------------------------------------------- -------------------------------------------
Service ID                                  Class name
------------------------------------------- ------------------------------------------- 
command_bus.messenger.handlers_locator      Symfony\Component\Messenger\Handler\HandlersLocator
event_bus.messenger.handlers_locator        Symfony\Component\Messenger\Handler\HandlersLocator
...
{% endhighlight %}

Now we need a new compiler pass that will also register our locator implementation as a separate service for each of the 
busses. Each of the new services should also be marked as decorator that will decorate the existing locator service for 
the appropriate bus. We also need the new services marked as autowired, so they will get the appropriate locator 
instance injected when instantiated.

Each of the busses that we have registered in the container is tagged with the `messenger.bus` tag, so we can use that
to find the list of ids of the message bus services.

{% highlight php startinline %}
use Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface;
use Symfony\Component\DependencyInjection\ContainerBuilder;

class HandlersLocatorCompilerPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        $busses = $container->findTaggedServiceIds('messenger.bus');
        $busIds = array_keys($busses);
    
        foreach ($busIds as $busId) {
            $decoratorId = $busId . '.messenger.wrapping_handlers_locator';
            $originalLocatorId = $busId . '.messenger.handlers_locator';
            
            $container->register($decoratorId, WrappingHandlersLocator::class)
                ->setAutowired(true)
                ->setDecoratedService($originalLocatorId);
        }
    }
}
{% endhighlight %}

Finally, we need to register the compiler pass in the application's kernel:

{% highlight php startinline %}
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\HttpKernel\Kernel as BaseKernel;

class Kernel extends BaseKernel
{
    // ...

    protected function build(ContainerBuilder $container): void
    {
        // ...

        $container->addCompilerPass(new HandlersLocatorCompilerPass());
    }
}
{% endhighlight %}

Let's take a look at the list of services now:

{% highlight text %}
-------------------------------------------             -------------------------------------------
Service ID                                              Class name
-------------------------------------------             ------------------------------------------- 
command_bus.messenger.handlers_locator                  alias for "command_bus.messenger.wrapping_handlers_locator"
command_bus.messenger.wrapping_handlers_locator         WrappingHandlersLocator
command_bus.messenger.wrapping_handlers_locator.inner   Symfony\Component\Messenger\Handler\HandlersLocator

event_bus.messenger.handlers_locator                    alias for "event_bus.messenger.wrapping_handlers_locator"
event_bus.messenger.wrapping_handlers_locator           WrappingHandlersLocator
event_bus.messenger.wrapping_handlers_locator.inner     Symfony\Component\Messenger\Handler\HandlersLocator
...
{% endhighlight %}

As we saw in the `MessengerPass` compiler pass above, each service of the `HandleMessageMiddleware` class 
(`[bus id].middleware.handle_message`) will receive the appropriate `[bus id].messenger.handlers_locator` service as 
argument in the constructor. For example, when instantiating the `event_bus.middleware.handle_message` service, the 
container will pass the `event_bus.messenger.handlers_locator` service as an argument.

With the compiler pass that we just registered, we changed, for example, the `event_bus.messenger.handlers_locator` 
service to be an alias for our own implementation of the locator, meaning that the component's middleware will now be 
getting an instance of our locator.

We can confirm that by checking the instance received in the middleware:

{% highlight php startinline %}
WrappingHandlersLocator {#239 ▼
    -decorated: Symfony\Component\Messenger\Handler\HandlersLocator {#237 ▶}
}
{% endhighlight %}

### Resolving the values for the arguments

Now that we're in control of how the handlers will be found, we need to actually implement the `getHandlers` method.

We already saw that we can get the list of handlers for the dispatched message by using the decorated locator. After 
that, we can iterate over that list, wrap each of the handlers if needed and then yield it.

For now, we'll assume that the dispatched message/event contains enough data for the handler's arguments. Later we'll 
see what we can do for other cases as well.

{% highlight php startinline %}
// inside the WrappingHandlersLocator class:

public function getHandlers(Envelope $envelope): iterable
{
    /** @var iterable<int, HandlerDescriptor> $handlerDescriptors */
    $handlerDescriptors = $this->decorated->getHandlers($envelope);
    $message = $envelope->getMessage();

    foreach ($handlerDescriptors as $handlerDescriptor) {
        if ($this->shouldWrapHandler($handlerDescriptor, $message)) {
            yield $this->wrapHandler($handlerDescriptor, $message);
        } else {
            yield $handlerDescriptor;
        }
    }
}
{% endhighlight %}

To determine if a handler should we wrapped, we'll be checking its list of arguments. Basically, we won't wrap a 
handler only if it has a single argument type-hinted with the message's own class. These handlers should continue to be
called as before.

{% highlight php startinline %}
// inside the WrappingHandlersLocator class:

private function shouldWrapHandler(HandlerDescriptor $handlerDescriptor, object $message): bool
{
    $arguments = $this->getHandlerArguments($handlerDescriptor);

    return (count($arguments) == 0)
            || (count($arguments) > 1)
            || (!$arguments[0]->getType()->getName() == get_class($message));
}
{% endhighlight %}

We already saw that the handler, which we can get from the handler descriptor is a closure, so we can use it to 
create a `ReflectionFunction` object. From that object we can get a list of `ReflectionParameter` objects that will 
represent the arguments of the handler.

{% highlight php startinline %}
// inside the WrappingHandlersLocator class:

/** @return array<int, ReflectionParameter> */
private function getHandlerArguments(HandlerDescriptor $handlerDescriptor): array
{
    $function = new ReflectionFunction($handlerDescriptor->getHandler());

    return $function->getParameters();
}
{% endhighlight %}

To do the wrapping, we need the list of arguments expected by the handler, and values for each of them.
After that, we will create a new handler descriptor which, as we saw before, will be yielded instead of the original 
one. 

When creating the new handler descriptor, we'll pass a new closure with no arguments, which internally will call the 
original one with the resolved values as arguments.

{% highlight php startinline %}
// inside the WrappingHandlersLocator class:

private function wrapHandler(HandlerDescriptor $handlerDescriptor, object $message): HandlerDescriptor
{
    $handlerArguments = $this->getHandlerArguments($handlerDescriptor);
    $handlerArgumentValues = $this->resolveValuesForHandlerArguments($handlerArguments, $message);

    return new HandlerDescriptor(
        fn() => $handlerDescriptor->getHandler()(...$handlerArgumentValues),
        [
            'alias' => $this->generateHandlerAlias($handlerDescriptor)
        ]
    );
}
{% endhighlight %}

As we saw before, in each of the handler descriptors, we have a name for the handler. After a handler finishes handling a 
message, the `HandleMessageMiddleware` middleware adds a new `\Symfony\Component\Messenger\Stamp\HandledStamp` stamp to 
the envelope that wraps the message, with the name of the handler. The same middleware also checks the received messages 
for the same type of stamp in order to prevent any of the handlers to handle the same message twice.

In our case, if we wrap multiple handlers, all the newly created handlers will have "Closure" as a name by default. That 
will prevent the message to be handled by multiple handlers (e.g. if we have multiple listeners), even thought they are 
completely different ones.

It is not possible to change the name of the handler completely, but there's an option to submit an alias when creating
the handler descriptor. That alias is later used as part of the name, so each handler will be treated as unique.

{% highlight php startinline %}
// inside the WrappingHandlersLocator class:

private function generateHandlerAlias(HandlerDescriptor $handlerDescriptor): string
{
    $function = new ReflectionFunction($handlerDescriptor->getHandler());

    return sprintf(
        "%s::%s",
        $function->getClosureScopeClass()?->getShortName(),
        $function->getName()
    );
}
{% endhighlight %}

We already have the mechanism for getting the list of arguments, so now we need to resolve the values, by mapping each 
of them to some properties of the message.

{% highlight php startinline %}
// inside the WrappingHandlersLocator class:

/**
 * @param array<int, ReflectionParameter> $arguments
 * @return array<int, mixed>
 */
private function resolveValuesForHandlerArguments(array $arguments, object $message): array
{
    return array_map(
        fn(ReflectionParameter $argument) => $this->resolveValueForHandlerArgument($argument, $message),
        $arguments
    );
}
{% endhighlight %}

The event that we're dispatching has more properties than we need in the handler, and additionally, one of the 
properties has a different name than the handler's argument. Because of that, we need a way to help the locator to map
the properties to the arguments properly, so we'll create a new PHP attribute called, for example, `ExtractedValue`. 

We'll be declaring this attribute on those arguments in the handler's signature that have no matching property in the 
handled message with same name, or if we intentionally want to use another one. It will have a `messageProperty` string 
property that we'll use to indicate which property of the message should be used for that particular argument. As the 
property will be optional, we'll be able to declare the attribute on an argument even if we don't want to explicitly 
specify a property name.

{% highlight php startinline %}
use Attribute;

#[Attribute(Attribute::TARGET_PARAMETER)]
class ExtractedValue
{
    public function __construct(
        public readonly ?string $messageProperty = null
    ) {
    }
}
{% endhighlight %}

In our case, we'll be passing the event's `$title` property as a value for the `$bookTitle` argument, so we will declare
the attribute on the argument, and we'll specify the property name. As the `$customerName` argument can be automatically 
mapped to the `customerName` property of the event, we can choose whether to declare the attribute with no arguments, or 
to just omit it.

{% highlight php startinline %}
class PurchaseHistory
{
    #[Listener(event: BookWasPurchased::class)]
    public function recordPurchase(
        #[ExtractedValue(messageProperty: "title")] string $bookTitle,
        #[ExtractedValue] string $customerName // we could omit the attribute here
    ): void {
        // ...
    }
}
{% endhighlight %}

Back to the locator, where we need to resolve the values for each of the attributes. When resolving the value for a
single attribute, we'll first try to fetch the declared `ExtractedValue` attribute.

If there's such attribute declared, we'll check if it contains a specified property name. In the cases when there is no
attribute or no method name has been explicitly specified, we'll assume that we need to use the argument's name.

Now that we have the needed property name, we'll try to fetch its value from the message object. If there's no such 
property we'll throw an exception.

{% highlight php startinline %}
// inside the WrappingHandlersLocator class:

private function resolveValueForHandlerArgument(
    ReflectionParameter $argument,
    object $message
): mixed {
    $attribute = $this->getAttributeForArgument($argument);

    $messagePropertyName = $attribute?->messageProperty ?? $argument->getName();

    if (property_exists($message, $messagePropertyName)) {
        return $message->{$messagePropertyName};
    }

    throw new Exception(sprintf(
        "Missing handler argument mapping for the \"%s\" argument of \"%s::%s\".",
        $argument->getName(),
        $argument->getDeclaringClass()->getName(),
        $argument->getDeclaringFunction()->getName()
    ));
}
{% endhighlight %}

We already have the argument as a `ReflectionParameter` object, so getting the attribute is easy. We'll fetch the 
attributes from the argument, which will give us a (possibly empty) list of `ReflectionAttribute` objects. We've not 
flagged the attribute as repeatable, so we can assume that there can be up to one declaration per argument. 

If the list of attributes is not empty, calling `newInstance` on the first element will give an instance of the 
attribute with the properties we've specified when declaring it on the handler. Otherwise, we'll just return `null`.

{% highlight php startinline %}
// inside the WrappingHandlersLocator class:

private function getAttributeForArgument(ReflectionParameter $argument): ?ExtractedValue
{
    $attributes = $argument->getAttributes(ExtractedValue::class);

    if (!count($attributes)) {
        return null;
    }

    return $attributes[0]->newInstance();
}
{% endhighlight %}

With what we've done so far, whenever the component's locator finds a handler with a signature that is not suitable by
default, we'll wrap it within a closure that will act as some sort of adapter that will pass the correct arguments to 
the handler.

If we dispatch the event again, we'll get the expected result:

{% highlight php startinline %}
$eventBus->dispatch(
    new BookWasPurchased("Example Book", "John Smith", $now)
);
{% endhighlight %}

>"John Smith bought the "Example Book" book."

Additionally, when checking the list of messages and handlers, we'll still get the correct listener:

{% highlight text %}
BookWasPurchased                                                                                           
    handled by PurchaseHistory (when method=recordPurchase)   
{% endhighlight %}

<small><small>
    *The whole WrappingHandlersLocator class written up to this point is available 
    [here](https://gist.github.com/angelov/689ce0ffd1f9412e12a85b5c3903e697){:target="_blank"}.*
</small></small>

### Handlers with additional arguments with default values

Now that we have this working, let's add another argument to the handler that won't be present in the event, but will 
have a default value.

{% highlight php startinline %}
class PurchaseHistory
{
    #[Listener(event: BookWasPurchased::class)]
    public function recordPurchase(
        #[ExtractedValue(messageProperty: "title")] string $bookTitle,
        string $customerName,
        int $bookAge = 1
    ): void {
        dd(sprintf(
            "%s bought the \"%s\" book. The book is %d year(s) old.",
            $customerName,
            $bookTitle,
            $bookAge
        ));
    }
}
{% endhighlight %}

We've made the locator to be throwing exceptions if we try to map an argument that has no matching property with the 
same name in the message object, or a declared attribute with explicitly specified property name. Because of that, 
dispatching the event now will result with the following exception:

> Missing handler argument mapping for the "bookAge" argument of "PurchaseHistory::recordPurchase".<br />
> <sup>Exception</sup>

As the argument has a default value in the handler's signature, we should be able to pass that value when we cannot 
retrieve something else from the message.

To achieve that, when no property for a given argument exists in the message object, we'll check if the argument has a 
default value specified. If a default value is available, we'll use that one. Otherwise, we'll still throw the 
exception.

{% highlight php startinline %}
// inside the WrappingHandlersLocator class:

private function resolveValueForHandlerArgument(
    ReflectionParameter $argument,
    object $message
): mixed {
    $attribute = $this->getAttributeForArgument($argument);

    $messagePropertyName = $attribute?->messageProperty ?? $argument->getName();

    if (property_exists($message, $messagePropertyName)) { ... }

    if ($argument->isDefaultValueAvailable()) {
        return $argument->getDefaultValue();
    }

    throw new Exception(sprintf(
        "Missing handler argument mapping or default value for the \"%s\" argument of \"%s::%s\".",
        $argument->getName(),
        $argument->getDeclaringClass()->getName(),
        $argument->getDeclaringFunction()->getName()
    ));
}
{% endhighlight %}

If we dispatch the event again, we'll get the new expected result:

>"John Smith bought the "Example Book" book. The book is 1 year(s) old."

<small><small>
    *The whole WrappingHandlersLocator class written up to this point is available
    [here](https://gist.github.com/angelov/b57d76cc568ec83825897180a1623fad){:target="_blank"}.*
</small></small>

### Handlers with additional arguments with no default values

Next, let's add one more argument to the caller, called `$sellerName`. However, this time we won't specify a default 
value for it.

{% highlight php startinline %}
class PurchaseHistory
{
    #[Listener(event: BookWasPurchased::class)]
    public function recordPurchase(
        #[ExtractedValue(messageProperty: "title")] string $bookTitle,
        string $customerName,
        string $sellerName,
        int $bookAge = 1
    ): void {
        dd(sprintf(
            "%s bought the \"%s\" book. The book is %d year(s) old and was sold by %s.",
            $customerName,
            $bookTitle,
            $bookAge,
            $sellerName
        ));
    }
}
{% endhighlight %}

Just as before, as this argument has no mapped property in the message object and no default value, we'll end up with an
exception:

> Missing handler argument mapping or default value for the "sellerName" argument of "PurchaseHistory::recordPurchase".
> <br />
> <sup>Exception</sup>

One option that we have here is to leave the code as it is, and just disallow methods with such arguments to be 
registered as message handlers in our application.

Another option is to add a way to declare a default value for the arguments that will be used only when handling the 
message, but not when regularly calling the method. 

For the second option, we'll add a new property to the attribute, called `defaultValue` which will hold the "special" 
default value.

{% highlight php startinline %}
#[Attribute(Attribute::TARGET_PARAMETER)]
class ExtractedValue
{
    public function __construct(
        public readonly ?string $messageProperty = null,
        public readonly mixed $defaultValue = null
    ) {
    }
}
{% endhighlight %}

And we can declare the attribute to the problematic argument:

{% highlight php startinline %}
class PurchaseHistory
{
    #[Listener(event: BookWasPurchased::class)]
    public function recordPurchase(
        // ...
        #[ExtractedValue(defaultValue: "Unknown seller")] string $sellerName,
        // ...
    ): void {
        // ...
    }
}
{% endhighlight %}

When resolving the values for such arguments, we'll check if the attribute declared on it (if present) has a 
specified default value to be used. 

As this property can be specified on attributes applied to properties with default value, it's up to us to decide which 
default value we want to have a higher priority when choosing. Here, we'll just ignore the one specified in the 
attribute if a default value is specified in the handler's signature.

As a last resort, we'll still throw an exception, just as before.

{% highlight php startinline %}
// inside the WrappingHandlersLocator class:

private function resolveValueForHandlerArgument(
    ReflectionParameter $argument,
    object $message
): mixed {
    $attribute = $this->getAttributeForArgument($argument);

    $messagePropertyName = $attribute?->messageProperty ?? $argument->getName();

    if (property_exists($message, $messagePropertyName)) { ... }

    if ($argument->isDefaultValueAvailable()) { ... }

    if ($attribute?->defaultValue) {
        return $attribute->defaultValue;
    }

    throw new Exception(...);
}
{% endhighlight %}

Dispatching the event again will give us the expected output:

>"John Smith bought the "Example Book" book. The book is 1 year(s) old and was sold by Unknown seller."

<small><small>
    *The whole WrappingHandlersLocator class written up to this point is available
    [here](https://gist.github.com/angelov/f5cd2eb4ea21ed65fc82caa25c55ba8d){:target="_blank"}.*
</small></small>

### Handlers with no arguments

As a final example, let's try what will happen if we register a method with no arguments, as a listener for our event.

{% highlight php startinline %}
class PurchasesCounter
{
    #[Listener(event: BookWasPurchased::class)]
    public function increaseNumber(): void
    {
        dd("Number of purchases increased");
    }
}
{% endhighlight %}

If we dispatch the event now, we'll get the expected output:

>"Number of purchases increased"

Also, the list of message and handlers will now show us the both listeners for the event:

{% highlight text %}
BookWasPurchased                                                                                           
    handled by PurchaseHistory (when method=recordPurchase)
    handled by PurchasesCounter (when method=increaseNumber)
{% endhighlight %}

### Next steps

This is definitely not a complete solution, and most definitely does not cover all the possible cases. It can 
additionally be adjusted accordingly to the application's needs. 

For example, we can go further and also make it possible for the `ExtractedValue` attributes to be optionally nested as 
part of the `Listener` attribute declared on the method, which is possible 
[since PHP 8.1](https://www.php.net/releases/8.1/en.php#new_in_initializers){:target="_blank"}. That way we can still 
define everything we need, without bloating the methods' signatures.

As said before, please use this only as a basis for future experimentation, and make sure you won't hurt your 
application's performance or maintainability by that.

<br />
<small><small><small>
    Photo for social media by [Ketut Subiyanto](https://www.pexels.com/photo/unpacked-boxes-in-middle-of-room-4246091/){:target="_blank"}.
</small></small></small>
