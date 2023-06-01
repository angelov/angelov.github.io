---
layout: post
title:  "Resolving values for unmapped properties when mapping request data to DTOs in Symfony 6.3+" 
date:   2023-06-01 19:00:00
author: Dejan Angelov
image: pexels-daria-obymaha-1684149.jpg
categories:
---

Ever since it was [introduced](https://symfony.com/blog/new-in-symfony-6-3-mapping-request-data-to-typed-objects){:target="_blank"},
the functionality for mapping request data to DTOs has been my favourite Symfony 6.3 new feature. The new 
`#[MapRequestPayload]` and `#[MapQueryString]` attributes allow us to delegate the DTO instantiation to the framework, 
instead of doing it manually in the controllers - for example when instantiating commands to be dispatched to a command 
bus. However, I realized that most of the commands in my application contain additional properties which are not 
directly mapped to the request data - such as the ID of the currently authenticated user. In this article, we'll see how
we can enhance the process to be able to resolve the unmapped values in such cases.

Let's start with this command:

{% highlight php startinline %}
readonly class ReserveRoomCommand
{
    public function __construct(
        public string $customerId,
        public string $roomId,
        public DateTimeImmutable $reservationFrom,
        public DateTimeImmutable $reservationTo
    ) {
    }
}
{% endhighlight %}

Usually, when we want to dispatch an instance of the command from a controller, we'd manually retrieve the values from 
the request object, and use them to initiate the command object:

{% highlight php startinline %}
#[Route("/reservations")]
class ReservationsController extends AbstractController
{
    #[Route("/", methods: "POST")]
    public function reserve(Request $request): Response
    {
        $command = new ReserveRoomCommand(
            $request->request->get('customerId'),
            $request->request->get('roomId'),
            new \DateTimeImmutable($request->request->get('from')),
            new \DateTimeImmutable($request->request->get('to'))
        )

        // ... dispatch the command, etc.
    }
}
{% endhighlight %}

With the new `#[MapRequestPayload]` attribute provided by Symfony, this process is now automated, and we can replace the
`Request` argument of the controller method with an argument of the command class. Additionally, we need to declare the
new attribute on the argument.

<div class="alert alert-info">
    <small>
	    <strong>Note:</strong>
        We will only use <code>#[MapRequestPayload]</code> in this article, but the same applies when using the 
        <code>#[MapQueryString]</code> attribute too. The framework uses the same value resolver for attributes with 
        any of these attributes.
    </small>
</div>

{% highlight php startinline %}
#[Route("/reservations")]
class ReservationsController extends AbstractController
{
    #[Route("/", methods: "POST")]
    public function reserve(
        #[MapRequestPayload] ReserveRoomCommand $command
    ): Response {
        // ... dispatch the command, etc.
    }
}
{% endhighlight %}

So, the framework will instantiate the object for us and populate it with the data from the request. But what if we 
don't want to read the `customerId` value form the request, but use the ID of the authenticated customer instead?

Although customizable, by default the framework uses the 
`\Symfony\Component\HttpKernel\Controller\ArgumentResolver\RequestPayloadValueResolver`
[value resolver](https://symfony.com/doc/6.3/controller/value_resolver.html) when resolving controller action arguments 
like the command object above.

If we take a look at that resolver, we can see that it simply reads the request content (either from the query string 
or the payload) and passes it to the Serializer component. The serializer decodes the data if needed (e.g. if the 
data is sent as a JSON object) - turning it into an associative array, and then denormalizes it - turning it into an 
object of the desired class, in our case `ReseveRoomCommand`.

In our case, the `customerId` property of the command doesn't have a default value, so if we don't supply it within the 
request, the denormalization process will fail.

> Failed to create object because the class misses the "customerId" property.<br />
> <sup>PartialDenormalizationException</sup>

To avoid this, but to also protect the value from being set from outside, we'll need to hook into the denormalization 
process and modify the data by adding/replacing the needed items.

First, let's create a new denormalizer.

{% highlight php startinline %}
use Symfony\Component\Serializer\Normalizer\DenormalizerInterface;

class PropertyValueResolvingDenormalizer implements DenormalizerInterface
{
    public function denormalize(mixed $data, string $type, string $format = null, array $context = []): mixed { ... }

    public function supportsDenormalization(mixed $data, string $type, string $format = null): bool { ... }

    public function getSupportedTypes(?string $format): array { ... }
}
{% endhighlight %}

We only want to resolve values for some properties, and not all of them can be resolved in the same way. To achieve the
differentiation, we can create new (or reuse some existing) custom attributes for each case, e.g. for resolving the ID
of the authenticated customer.

{% highlight php startinline %}
#[\Attribute(\Attribute::TARGET_PROPERTY)]
class Authenticated
{
}
{% endhighlight %}

And declare it on the argument in the command:

{% highlight php startinline %}
readonly class ReserveRoomCommand
{
    public function __construct(
        #[Authenticated] public string $customerId,
        public string $roomId,
        public DateTimeImmutable $from,
        public DateTimeImmutable $to,
    ) {
    }
}
{% endhighlight %}

As already mentioned, we can have multiple similar attributes and multiple ways for resolving values, so we can define 
the following interface and have all the resolves implement it:

{% highlight php startinline %}
use Symfony\Component\DependencyInjection\Attribute\AutoconfigureTag;

#[AutoconfigureTag]
interface PropertyValueResolverInterface
{
    public function resolve(ReflectionAttribute $attribute): mixed;

    /** @return class-string */
    public function supportedAttribute(): string;
}
{% endhighlight %}

<div class="alert alert-info">
    <small>
	    <strong>Note:</strong> For simplicity, we'll use attributes to interact with the Symfony's Dependency Injection
        container. You can still use your preferred way for configuring it.
    </small>
</div>

To resolve values for arguments marked with our `#[Authenticated]` attribute, we'll have an implementation that will 
support that attribute, and which will be fetching the current customer and returning its ID.

{% highlight php startinline %}
class AuthenticatedUserResolver implements PropertyValueResolverInterface
{
    // ...

    public function resolve(ReflectionAttribute $attribute): mixed
    {
        $uuid = // resolve the authenticated customer's uuid using some other service

        return $uuid;
    }

    public function supportedAttribute(): string
    {
        return Authenticated::class;
    }
}
{% endhighlight %}

Back to the denormalizer, we'll be injecting an array of all registered value resolvers implementing the interface from 
above. Then in the `supportsDenormalization` we can use this list to determine if the denormalizer should be applied 
when mapping some request data.

Among the other arguments we're receiving in the `supportsDenormalization` method is the type i.e. the class of the 
object we'll be getting at the end of the process - in our case the command. From that class, we can get the list of 
properties and then iterate over it to find if any of the properties has a declared attribute for which we have a 
suitable value resolver. If we find such property, we'll let the serializer know that it should use this denormalizer.

{% highlight php startinline %}
use Symfony\Component\DependencyInjection\Attribute\TaggedIterator;
use Symfony\Component\Serializer\Normalizer\DenormalizerInterface;

class PropertyValueResolvingDenormalizer implements DenormalizerInterface
{
    /** @var array<class-string, object<PropertyValueResolverInterface>> */
    private array $resolvers = [];

    /** @param iterable<PropertyValueResolverInterface> $resolvers */
    public function __construct(
        #[TaggedIterator(PropertyValueResolverInterface::class)] iterable $resolvers
    ){
        foreach ($resolvers as $resolver) {
            $this->resolvers[$resolver->supportedAttribute()] = $resolver;
        }
    }

    public function supportsDenormalization(mixed $data, string $type, string $format = null): bool
    {
        $properties = (new ReflectionClass($type))->getProperties();

        foreach ($properties as $property) {
            if ($this->canResolveValueForProperty($property)) {
                return true;
            }
        }

        return false;
    }

    private function canResolveValueForProperty(ReflectionProperty $property): bool
    {
        $attributes = $property->getAttributes();

        foreach ($attributes as $attribute) {
            if (array_key_exists($attribute->getName(), $this->resolvers)) {
                return true;
            }
        }

        return false;
    }

    // ...
}
{% endhighlight %}

In the `denormalize` method we'll be doing something similar - we can iterate over the class's properties as we did 
above, and use a suitable value resolver for each of the properties that need resolving. In the `data` argument we'll 
have the existing request data - which have previously been decoded by the serializer - as an array that we can modify 
to add/replace the resolved values.

As our intention is not to completely replace the denormalization process, but just to enhance it, we'll pass the final
modified data array to the serializer's `ObjectNormalizer` which will then transform it into an object.

{% highlight php startinline %}
use Symfony\Component\Serializer\Normalizer\DenormalizerInterface;
use Symfony\Component\Serializer\Normalizer\ObjectNormalizer;

class PropertyValueResolvingDenormalizer implements DenormalizerInterface
{
    // ...

    private ObjectNormalizer $objectNormalizer;

    public function __construct(
        // ...
        ObjectNormalizer $objectNormalizer,
    ) {
        // ...
        $this->objectNormalizer = $objectNormalizer;
    }

    public function denormalize(mixed $data, string $type, string $format = null, array $context = []): mixed
    {
        $properties = (new ReflectionClass($type))->getProperties();

        foreach ($properties as $property) {
            $name = $property->getName();

            if (!$this->canResolveValueForProperty($property)) { // shown in code above
                continue;
            }

            $data[$name] = $this->resolveValueForProperty($property);
        }

        return $this->objectNormalizer->denormalize($data, $type, $format, $context);
    }

    private function resolveValueForProperty(ReflectionProperty $property): mixed
    {
        $attributes = $property->getAttributes();

        foreach ($attributes as $attribute) {
            if (array_key_exists($attribute->getName(), $this->resolvers)) {
                return $this->resolvers[$attribute->getName()]->resolve($attribute);
            }
        }

        return null;
    }

    // ...
}
{% endhighlight %}

Finally, let's also look at the `getSupportedTypes` method. It has been introduced in Symfony 6.3 too as a performance 
improvement, but due to BC reasons is still not enforced as part of the `DenormalizerInterface` interface. It is also a 
replacement for the now-deprecated `\Symfony\Component\Serializer\Normalizer\CacheableSupportsMethodInterface` interface 
and can be used to tell the serializer whether it can cache the result of the `supportsDenormalization` method when 
denormalizing into a certain type.

As our denormalizer can process data for multiple types - objects of multiple classes, and the declared attributes are 
not changing during the execution, we can use the generic `object` type and always allow the caching.

You can read more about the `getSupportedTypes` in 
[this article](https://symfony.com/blog/new-in-symfony-6-3-performance-improvements#improved-performance-of-serializer-normalizers-denormalizers){:target="_blank"}.

{% highlight php startinline %}
use Symfony\Component\Serializer\Normalizer\DenormalizerInterface;

class PropertyValueResolvingDenormalizer implements DenormalizerInterface
{
    // ...

    /** @return array<class-string, bool> */
    public function getSupportedTypes(?string $format): array
    {
        return ['object' => true];
    }
}
{% endhighlight %}

<small><small>
*The whole PropertyValueResolvingDenormalizer class is available
[here](https://gist.github.com/angelov/9f0beae2c415771931ab9221212a1b5f){:target="_blank"}.*
</small></small>

To try the denormalizer out, let's say that the ID of the authenticated customer is "customer-uuid". If we send a 
request with the following JSON body to our endpoint:

{% highlight json %}
{
    "roomId": "room-uuid",
    "from": "2023-06-10",
    "to": "2023-06-15"
}
{% endhighlight %}

We'll get the following object as a result:

{% highlight php startinline %}
^ ReserveRoomCommand {#181
    +customerId: "customer-uuid"
    +roomId: "room-uuid"
    +from: DateTimeImmutable @1686355200 {#182
        date: 2023-06-10 00:00:00.0 UTC (+00:00)
    }
    +to: DateTimeImmutable @1686787200 {#209
        date: 2023-06-15 00:00:00.0 UTC (+00:00)
    }
}
{% endhighlight %}

As another example, let's imagine that for some reason we also want to pass the reservation time within the command, 
which will be the current time.

Now we just need to create a new (or re-use an existing) attribute:

{% highlight php startinline %}
#[\Attribute(\Attribute::TARGET_PROPERTY)]
class CurrentDateTime
{
}
{% endhighlight %}

And a resolver for it:

{% highlight php startinline %}
use Psr\Clock\ClockInterface;

class CurrentDateTimeResolver implements PropertyValueResolverInterface
{
    private ClockInterface $clock;

    public function __construct(ClockInterface $clock)
    {
        $this->clock = $clock;
    }

    public function resolve(ReflectionAttribute $attribute): mixed
    {
        return $this->clock->now()->format(DATE_ATOM);
    }

    public function supportedAttribute(): string
    {
        return CurrentDateTime::class;
    }
}
{% endhighlight %}

Note that here we're returning a string as a resolved value, and not a `DateTimeImmutable` object. That's because after 
we pass the data to the `ObjectNormalizer` to continue with the denormalization, it will use the 
`\Symfony\Component\Serializer\Normalizer\DateTimeNormalizer` denormalizer to instantiate the final `DateTimeImmmutable` 
object - and that denormalizer requires the value to be sent as a string. So, in general, our value resolvers should 
be returning the resolved values just as if they've been send within a request.

Now if we add a new property to the command and declare the attribute on it:

{% highlight php startinline %}
readonly class ReserveRoomCommand
{
    public function __construct(
        #[Authenticated] public string $customerId,
        public string $roomId,
        public DateTimeImmutable $from,
        public DateTimeImmutable $to,
        #[CurrentDateTime] public DateTimeImmutable $reservedAt
    ) {
    }
}
{% endhighlight %}

When we send the same JSON object as a request as above, we'll get the current time as expected:

{% highlight php startinline %}
^ ReserveRoomCommand {#184
    +customerId: "customer-uuid"
    +roomId: "room-uuid"
    +from: DateTimeImmutable @1686355200 {#185
        date: 2023-06-10 00:00:00.0 UTC (+00:00)
    }
    +to: DateTimeImmutable @1686787200 {#213
        date: 2023-06-15 00:00:00.0 UTC (+00:00)
    }
    +reservedAt: DateTimeImmutable @1685054767 {#214
        date: 2023-05-25 22:46:07.0 +00:00
    }
}
{% endhighlight %}

As a next step, we can declare the attributes on all the commands or other objects we want to be mapped in the 
controllers. If we face another unmapped property, we saw how we can add a custom resolver for it.

<br />
<small><small><small>
    Photo for social media by 
    [Daria Obymaha](https://www.pexels.com/photo/brown-framed-eyeglasses-on-white-printer-paper-beside-white-ceramic-mug-1684149/){:target="_blank"}.
</small></small></small>
