---
layout: post
title:  "Centralized exception handling with Symfony and custom PHP attributes" 
date:   2022-11-24 17:02:00
author: Dejan Angelov
image:  error-cover.jpg
categories:
---

Over time, as our application grows, the controllers can become cluttered with repeated exception handling code. In this 
article we'll see how we can have both cleaner controller methods when using the Symfony framework by centralizing the 
exception handling, and cleaner domain exceptions by only declaring custom PHP attributes on their classes.

Let's imagine that we are developing an e-commerce application that can process and manage orders made by the website's 
visitors. One of the functionalities is the ability to cancel a specific order. However, an order can be cancelled 
only if these three conditions are satisfied:

* it must be present (obviously)
* it must not be shipped yet
* only the customer who created it can cancel it

If there is an attempt to cancel an order without satisfying all the conditions, we are throwing one of the following 
exceptions:

{% highlight php startinline %}
class OrderNotFound extends Exception
{
    public function __construct(OrderId $id)
    {
        parent::__construct(
            sprintf('The order "%s" could not be found.', (string) $id)
        );
    }
}
{% endhighlight %}

{% highlight php startinline %}
class OrderAlreadyShipped extends Exception
{
    public function __construct(Order $order)
    {
        parent::__construct(
            sprintf('The order "%s" is already shipped.', (string) $order->getId())
        );
    }
}
{% endhighlight %}

{% highlight php startinline %}
class CustomerMismatch extends Exception
{
    public function __construct(Order $order, Customer $customer)
    {
        parent::__construct(sprintf(
            'The order "%s" is not created by the customer "%s".',
            (string) $order->getId(),
            (string) $customer->getId()
        ));
    }
}
{% endhighlight %}

### Cancelling the order and handling the exceptions

One way to handle the possibly thrown exceptions is to catch them in the controller where we initiate the cancellation
(either directly or through some other service, like a command handler). Then we can build the response, assign the 
suitable status code to it, and return it to the client.

{% highlight php startinline %}
class OrdersController
{
    // ...

    #[Route("/orders/{id}/cancel", ...)]
    public function cancel(string $id): Response
    {
        // ...

        try {
            $order = $this->orders->find($id);
            $order->cancel(by: $customer);

            $this->orders->store($order);
        } catch (OrderNotFound $e) {
            return new JsonResponse(["error" => $e->getMessage()], Response::HTTP_NOT_FOUND);
        } catch (CustomerMismatch $e) {
            return new JsonResponse(["error" => $e->getMessage()], Response::HTTP_FORBIDDEN);
        } catch (OrderAlreadyShipped $e) {
            return new JsonResponse(["error" => $e->getMessage()], Response::HTTP_UNPROCESSABLE_ENTITY);
        }

        return new JsonResponse(...);
    }
}
{% endhighlight %}

As we can see, most of the code in our method is used for handling the exceptions. This will likely happen with most
of the other controller methods in our application as well. Another negative implication is that whenever we start 
throwing a new similar exception somewhere in our domain layer, we'll also have to add another catch block in all 
controllers it may end up to. That can span over multiple API endpoints, thus we'll need to update multiple controller 
methods (for example, we may also throw an `OrderAlreadyShipped` exception if somebody tries to add a new item to an 
already shipped order.)

### Moving the exception handling

As a first step, let's remove the entire try-catch block from the controller method:

{% highlight php startinline %}
class OrdersController extends AbstractController
{
    // ...

    #[Route("/orders/{id}/cancel", ...)]
    public function cancel(string $id): Response
    {
        // ...

        $order = $this->orders->find($id);
        $order->cancel(by: $customer);

        $this->orders->store($order);

        return new JsonResponse(...);
    }
}
{% endhighlight %}

The controller method is more clear now, and we can easily see it's intention. Note that as a trade-off, we're losing 
some visibility over what can go wrong during the execution of the methods we're calling.

If we try to cancel an already shipped order now, as expected since we removed the exception handling, we'll get the 
Symfony's default error page.

> The order "213ba2c0-d82c-4805-8de4-773d20f3cbe3" is already shipped.<br />
> <sup>HTTP 500 Internal Server Error - OrderAlreadyShipped</sup>

When an exception is thrown during the handling of the request, and is not explicitly caught and handled, the Symfony's 
HttpKernel dispatches an 
[`kernel.exception`](https://symfony.com/doc/6.1/reference/events.html#kernel-exception){:target="_blank"} event.

To centralize the exception handling, we can create a listener that will listen to this type of events, and that will 
take care of preparing the response for the client. When such event is dispatched, all listeners that are supposed to
handle it receive an instance of the `ExceptionEvent` class, from which we can get the thrown exception object.

{% highlight php startinline %}
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\ExceptionEvent;
use Symfony\Component\HttpKernel\KernelEvents;

class ExceptionHandler implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [KernelEvents::EXCEPTION => 'handleException'];
    }

    public function handleException(ExceptionEvent $event): void
    {
        $exception = $event->getThrowable();

        // ...
    }
}
{% endhighlight %}

Now, whenever there's an unhandled exception in our application, we will receive an event in the listener we just 
created. However, we don't want all the exceptions handled here, as not everything is intended to be shown to the 
client, so we need a way to determine if the received exception should be handled or not.

The domain exceptions we have in our application so far can be divided in 3 groups, each associated with an appropriate
HTTP status code.

<table class="table table-bordered table-condensed table-striped" style="font-size: small; width: 45%">
    <thead>
        <tr>
            <th>Exception</th>
            <th>Status Code</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>OrderNotFound</td>
            <td>404 Not Found</td>
        </tr>
        <tr>
            <td>CustomerMismatch</td>
            <td>403 Forbidden</td>
        </tr>
        <tr>
            <td>OrderAlreadyShipped</td>
            <td>422 Unprocessable Entity</td>
        </tr>
    </tbody>
</table>

For each of the HTTP status codes that we want to support we'll create a separate custom PHP attribute. To make sure all
of them are providing a specific status code, the attributes will implement the following interface:

{% highlight php startinline %}
interface StatusCodeProvider
{
    public function getStatusCode(): int;
}
{% endhighlight %}

Here are the three attributes we need so far:

{% highlight php startinline %}
use Symfony\Component\HttpFoundation\Response;

#[Attribute(Attribute::TARGET_CLASS)]
class NotFound implements StatusCodeProvider
{
    public function getStatusCode(): int
    {
        return Response::HTTP_NOT_FOUND;
    }
}
{% endhighlight %}

{% highlight php startinline %}
use Symfony\Component\HttpFoundation\Response;

#[Attribute(Attribute::TARGET_CLASS)]
class UnprocessableEntity implements StatusCodeProvider
{
    public function getStatusCode(): int
    {
        return Response::HTTP_UNPROCESSABLE_ENTITY;
    }
}
{% endhighlight %}

{% highlight php startinline %}
use Symfony\Component\HttpFoundation\Response;

#[Attribute(Attribute::TARGET_CLASS)]
class AccessDenied implements StatusCodeProvider
{
    public function getStatusCode(): int
    {
        return Response::HTTP_FORBIDDEN;
    }
}
{% endhighlight %}

Next, we'll declare the new attributes on the existing exception classes:

{% highlight php startinline %}
#[NotFound]
class OrderNotFound extends Exception { ... }
{% endhighlight %}

{% highlight php startinline %}
#[UnprocessableEntity]
class OrderAlreadyShipped extends Exception { ... }
{% endhighlight %}

{% highlight php startinline %}
#[AccessDenied]
class CustomerMismatch extends Exception { ... }
{% endhighlight %}

Back to the listener, we can now use the Reflection API to get the attributes declared on the thrown exception's class. 
As any class can have multiple attributes declared on it, we can use the fact that all our attributes implement the 
`StatusCodeProvider` interface and filter out the non-related ones. If no such attribute is declared on the class, it 
means that we don't support handling this exception in the handler. If multiple such attributes are declared, we'll just 
use the first one.

{% highlight php startinline %}
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\ExceptionEvent;

class ExceptionHandler implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array { ... }

    public function handleException(ExceptionEvent $event): void { ... }

    private function getAttribute(Throwable $exception): ?StatusCodeProvider
    {
        $reflectionClass = new ReflectionClass($exception);
        $attributes = $reflectionClass->getAttributes();

        $instances = array_map(
            fn(ReflectionAttribute $attribute) => $attribute->newInstance(),
            $attributes
        );

        $supported = array_filter(
            $instances,
            fn(object $attribute) => $attribute instanceof StatusCodeProvider
        );

        if (count($supported) === 0) {
            return null;
        }

        return $supported[array_key_first($supported)];
    }

    private function supportedException(Throwable $exception): bool
    {
        return $this->getAttribute($exception) !== null;
    }
}
{% endhighlight %}

If the exception is not supported, we can simply ignore it and let it be handled somewhere else. Otherwise, we need to
extract the needed data and build the response we'd want to return to the client.

For the response's content we can use the message provided by the exception, and we can get the needed HTTP status code 
from the declared attribute that we fetched above.

When the response is prepared, we need to pass it along by updating the received event object using the `setResponse`
method.

<div class="alert alert-warning">
    <small>
	    <strong>Note:</strong> By default we can only set 3xx (Redirection), 4xx (Client error) and 5xx (Server error) 
        HTTP status codes on the response we're building in the listener.
        If we set a status code that is outside these ranges, the HttpKernel will override it and will use 500 instead.
        <br />
        This behaviour can be changed by calling 
        <code class="language-plaintext highlighter-rouge">allowCustomResponseCode()</code> on the event object, but 
        doing that is discouraged by the 
        <a href="https://symfony.com/doc/6.1/reference/events.html#kernel-exception" target="_blank">docs</a>.
    </small>
</div>

{% highlight php startinline %}
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\ExceptionEvent;

class ExceptionHandler implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array { ... }

    public function handleException(ExceptionEvent $event): void
    {
        $exception = $event->getThrowable();

        if (!$this->supportedException($exception)) {
            return;
        }

        $attribute = $this->getAttribute($exception);

        $response = new JsonResponse(
            [
                "error" => $exception->getMessage()
            ],
            $attribute->getStatusCode()
        );

        $event->setResponse($response);
    }

    private function getAttribute(Throwable $exception): ?StatusCodeProvider { ... }

    private function supportedException(Throwable $exception): bool { ... }
}
{% endhighlight %}

<div class="alert alert-info">
    <small>
	    <strong>Note:</strong> For simplicity, the data for the response's content in the examples is passed directly as 
        an array. An external serializer service can be injected and used to structure the data instead.
    </small>
</div>

If we try to cancel the same order again, this time we'll get the expected JSON data, with *422 Unprocessable Entity* as
status code.

{% highlight json %}
{
    "error": "The order \"213ba2c0-d82c-4805-8de4-773d20f3cbe3\" is already shipped."
}
{% endhighlight %}

### Next steps

Now that we have the attributes for adding the needed metadata, and the handler which will handle the marked exceptions, 
we can easily add more use cases.

For example, if we have a functionality for applying promo codes to the orders, and we check if the given code is still 
valid, we can have the following exception, and just declare the `UnprocessableEntity` attribute on it:

{% highlight php startinline %}
#[UnprocessableEntity]
class PromoCodeExpired extends Exception
{
    public function __construct(PromoCode $promoCode)
    {
        parent::__construct(sprintf(
            'The promo code "%s" has expired.',
            $promoCode->getCode()
        ));
    }
}
{% endhighlight %}

If we want to support more status codes, we only need to create a new attribute that will implement the 
`StatusCodeProvider` interface and start declaring it on the exceptions.

Sometimes, it may happen that we may want to return a different message to the client instead of the one we have in 
the exception (which we may still want to keep for other purposes). For such cases, we can extend the attributes by 
adding an optional `message` argument to each of them that can be specified when declaring it on an exception. Then in
the handler we can use this message if provided, or the exception's one if not.

The events of the `ExceptionEvent` class also contain the request object, which can be useful to decide in which format 
we should respond to the client. If we also have controllers that render templates and provide HTML content, we can 
create an additional error handler that will set a flash message and redirect the client to the previous page, if the 
exception is thrown while executing a request from those pages. This can be also useful if we have multiple bounded 
contexts, so we can have multiple similar handlers that will cover exceptions from different areas in the application.

<br />
<small><small><small>
    Photo for social media by 
    [Vie Studio](https://www.pexels.com/photo/word-error-on-white-surface-4439425/){:target="_blank"}.
</small></small></small>
