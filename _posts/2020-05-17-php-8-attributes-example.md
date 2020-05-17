---
layout: post
title:  "PHP 8 Attributes example: Injecting the current user in Symfony controllers without the security component" 
date:   2020-05-17 21:31:00
author: Dejan Angelov
image:  pmappflow.png
categories:
---

Although we've been using annotations in PHP for years, this functionality was not part of the language itself. Instead, 
we had to rely on parsing the comments formatted as docblocks above the classes, properties or methods, usually with the 
`doctrine/annotations` library. With the acceptance of the ["Attributes v2" RFC](https://wiki.php.net/rfc/attributes_v2) 
for the next major PHP version, the concept will become part of the PHP core, called "Attributes" and with slightly 
different syntax: `<<Example>>` instead of `@Example`. In this article we'll see how we can use a custom attribute to 
tell Symfony how to resolve the currently authenticated user as controller action argument even if we don't use the 
security component.

<div class="alert alert-warning">
    <small>
	    <strong>Note:</strong> Even though the RFC has been accepted, please consider that at the moment of publishing 
	    of this article the functionality is still under development and is not yet merged to the master branch. If you 
	    want to try the code presented here now, you will have to compile the PHP source by yourself using the branch 
	    from <a href="https://github.com/php/php-src/pull/5394" target="_blank">this PR</a>.
	</small>
</div>

### The story

Let's imagine that we're developing a project that, among others, includes two bounded contexts called 
**"Project Management"** and **"Authentication"**, working as separate applications. Our focus in this article will be 
on the "Project Management" one that we're developing as a Symfony application.

As actors in the application we have managers and developers, represented with the `Manager` and `Developer` entities 
respectively (both implementing the `EmployeeInterface` interface). The managers will be able to create and assign tasks 
(`Task` entity) to developers and the developers can mark those tasks as completed. To do their job, both managers and 
developers are able to authenticate in the system. 

The application provides a HTTP API, used by the front-end web interface. To authenticate the employees when we receive 
a request, we're passing the `Authorization` header value to the other "Authentication" application, which then gives us 
the UUID of the authenticated user. We have those UUIDs assigned the managers and developers in our application, and we 
can know which manager or developer has sent the request.

As example, here's the flow for creating a new task:

![Example](/images/pmappflow.png)

### Resolving the authenticated developer/manager

For assigning the tasks, we have the `assignTask` controller action in the `TasksController`, which is mapped to the 
`POST /tasks/{task}/assign/{developer}` API endpoint.

In order to perform the task assignment, we need to fetch the task, the developer to which the task will be assigned 
and the manager who will do the assignment.

For simplicity, we'll imagine that we use  *Doctrine ORM* and *SensioFrameworkExtraBundle*, so Symfony will 
automatically know how to inject the proper task and developer objects in the action, based on the URL. If we don't want 
to use that bundle, we can use similar approach as the one discussed later in this article to fetch those objects as 
well. You may even not need those objects here and just use the IDs.

Here is the general structure of the controller and the action:

{% highlight php startinline %}
class TasksController
{
    /** @Route("/tasks/{task}/assign/{developer}", methods={"POST"}) */
    public function assignTask(
        Request $request,
        Task $task,
        Developer $developer
    ): Response {
        // ...

        return new JsonResponse(/** ... */);
    }
}
{% endhighlight %}

The only thing that we miss now is the manager who does the assignment. So, we need to transform the received token from 
the request to the proper `Manager` object from the storage. To achieve that, we'll have the following abstraction:

{% highlight php startinline %}
interface AuthenticatedEmployeeResolverInterface
{
    /** @throws InvalidTokenException */
    public function fromToken(string $token): EmployeeInterface;
}
{% endhighlight %}

If the token is invalid, it will throw an `InvalidTokenException` exception.

As the "Authentication" application will give us only the id of the employee, we'll also have an 
`EmployeesRepositoryInterface` abstraction that will know how to fetch the proper manager or developer from the storage.

{% highlight php startinline %}
interface EmployeesRepositoryInterface
{
    /** @throws EmployeeNotFoundException */
    public function fetchById(string $id): EmployeeInterface;
}
{% endhighlight %}

So, in the authenticated employee resolver, first we'll use a PSR-18 HTTP client in order to exchange the token for the 
UUID of the user by communicating to the API of the "Authentication" application, and then we'll fetch the employee from 
the storage.

{% highlight php startinline %}
class AuthenticatedEmployeeResolver implements AuthenticatedEmployeeResolverInterface
{
    private ClientInterface $httpClient;
    private EmployeesRepositoryInterface $employeesRepository;

    // constructor omitted

    /** @throws InvalidTokenException */
    public function fromToken(string $token): EmployeeInterface
    {
        try {
            $uuid = // send the request and parse the response

            $employee = $this->employeesRepository->findById($uuid);
        } catch (ClientExceptionInterface | EmployeeNotFoundException $exception) {
            throw new InvalidTokenException();
        }
        
        return $employee;
    }
}
{% endhighlight %}

Now, let's first see the approach with injecting the `AuthenticatedEmployeeResolverInterface` service as constructor 
argument in the controller, and explicitly using it to fetch the currently authenticated manager when assigning the 
task.

In the `assignTask` method, we'll retrieve the token from the request and then try to resolve the authenticated employee 
using it. Additionally, we'll also check if the employee is actually a manager inside our bounded context.

{% highlight php startinline %}
class TasksController
{
    private AuthenticatedEmployeeResolverInterface $employeeResolver;

    // constructor omitted 

    /** @Route("/tasks/{task}/assign/{developer}", methods={"POST"}) */
    public function assignTask(
        Request $request,
        Task $task,
        Developer $developer
    ): Response {
        $token = str_replace(
            'Bearer ',
            '',
            $request->headers->get('Authorization', '')
        );

        try {
            $employee = $this->employeeResolver->fromToken($token);
        } catch (InvalidTokenException $exception) {
            return $this->createUnauthorizedResponse();
        }

        if (!$employee instanceof Manager) {
            return $this->createUnauthorizedResponse();
        }

        // ... do the assignment by sending a command or something similar

        return new JsonResponse(/** ... */);
    }

    private function createUnauthorizedResponse(): Response
    {
        return new JsonResponse('Unauthorized', Response::HTTP_UNAUTHORIZED);
    }
}
{% endhighlight %}

With this, we've managed to fetch the proper manager and now we can use it to do the task assignment. However, this 
bloats our controllers, as we need to inject the `AuthenticatedEmployeeResolverInterface` abstraction and parse the 
token in all controllers in which we need the authenticated employee.

Luckily, Symfony provides a mechanism called [**"Action Argument Resolving"**](https://symfony.com/doc/current/controller/argument_value_resolver.html) 
which will help us replace the whole authentication related code from the controllers with a single PHP attribute.

As we can see in the Symfony documentation, there's an existing `UserValueResolver` resolver, but using it will require 
us to install the whole Symfony Security component and make our `Manager` and `Developer` entities implement the 
component's `UserInterface` which is too much for our case.

So, we need to extend the argument resolving mechanism by creating a new argument value resolver which will implement
the `Symfony\Component\HttpKernel\Controller\ArgumentValueResolverInterface` interface.

Even though both types of employees implement the `EmployeeInterface` we can't just make it resolve any argument 
implementing this interface, or each of the entity classes separately, because as we saw, we can have controller action 
methods that require multiple such objects, of which only one is the authenticated and the others should be fetched from 
the storage. That's why we need to mark the authenticated one with an attribute.

Our final result will look like this:

{% highlight php startinline %}
class TasksController
{
    /** @Route("/tasks/{task}/assign/{developer}", methods={"POST"}) */
    public function assignTask(
        <<Authenticated>> Manager $manager,
        Task $task,
        Developer $developer
    ): Response {

        // ... send a command to the command bus or something similar

        return new JsonResponse(/** ... */);
    }
}
{% endhighlight %}

So, we'll create a custom PHP attribute called `Authenticated`. To do that, as defined in the RFC, we need to create a 
class with that name and add a `PhpAttribute` attribute on it. Note that the PHP's `PhpAttribute` class is in the global 
space, so you will probably have to import it or use it as `\PhpAttribute`.

{% highlight php startinline %}
<<PhpAttribute>>
class Authenticated
{
}
{% endhighlight %}

As defined in the interface, the argument value resolver will have two methods. One (`supports`) that receives the 
request and an attribute metadata and tells Symfony whether this resolver should be used for the specific attribute. 
When doing the resolving, Symfony goes through all the registered resolvers for each of the controller action attributes 
and by calling this methods checks which resolvers should it use. The other one (`resolve`) is the one that will 
actually provide the proper value that should be passed to the controller action method.

Let's see each of the two methods separately.

{% highlight php startinline %}
class AuthenticatedEmployeeArgumentValueResolver implements ArgumentValueResolverInterface
{
    // constructor and properties omitted, but will be needed later

    public function supports(Request $request, ArgumentMetadata $argument): bool
    {
        $controller = $request->attributes->get('_controller');
        [$className, $methodName] = explode("::", $controller);

        $parameter = new ReflectionParameter(
            [$className, $methodName],
            $argument->getName()
        );

        $attributes = $parameter->getAttributes();

        return in_array(
            Authenticated::class,
            array_map(
                fn(ReflectionAttribute $attribute): string => $attribute->getName(), 
                $attributes
            )
        );
    }
    
    public function resolve(Request $request, ArgumentMetadata $argument): iterable { ... }
}
{% endhighlight %}

The controller resolving happens before the argument resolving, so we can take that information from the request's 
attributes. With it, we can find out the needed controller class and action method. Note that the code above can be 
slightly changed if we were using invokable controller classes as actions, but the main logic is still same. Having that 
information, we can now use the reflection API to get more information about the method's argument/property.

[As defined in the RFC](https://wiki.php.net/rfc/attributes_v2#reflection), the reflection classes that represent 
classes, functions, properties and class constants now come with a new `getAttributes` method which returns an array of 
`ReflectionAttribute` objects. We'll use that method on our `ReflectionParameter` object that represents the current 
argument to retrieve all attributes placed on it.

Then we can use the `getName` method for each of the found attributes to find out their FQCNs and check if any of it 
matches the custom attribute that we created.

If our attribute is matched in the list of attributes for the argument, the next method from our value resolver is 
called - the `resolve` one.

{% highlight php startinline %}
class AuthenticatedEmployeeArgumentValueResolver implements ArgumentValueResolverInterface
{
    private AuthenticatedEmployeeResolverInterface $employeeResolver;

    // constructor omitted

    public function supports(Request $request, ArgumentMetadata $argument): bool { ... }

    public function resolve(Request $request, ArgumentMetadata $argument): iterable
    {
        $token = str_replace(
            'Bearer ',
            '',
            $request->headers->get('Authorization', '')
        );

        try {
            $employee = $this->employeeResolver->fromToken($token);
        } catch (InvalidTokenException $exception) {
            throw new UnauthenticatedException();
        }

        $argumentType = $argument->getType();

        if (!$employee instanceof $argumentType) {
            throw new UnauthorizedException();
        }

        yield $employee;
    }
}
{% endhighlight %}

We can see that the logic in the `resolve` method is almost identical to what we previously had in the controller. One 
of the differences here is that we don't just create a HTTP response in the invalid cases, but throw custom exceptions 
instead. Later, we'll tell Symfony how to convert those exceptions to proper HTTP responses.

Another difference is that we don't explicitly check if the employee is of a specific type. We do that in a more 
flexible way, by getting the argument type from the argument metadata. That allows us to specify the employee type as a 
type hint in the controller action, making all of the following cases supported:

{% highlight php startinline %}
class EmployeesController
{
    /** @Route("/profile", methods={"GET"}) */
    public function profile(
        <<Authenticated>> EmployeeInterface $employee
    ): Response {
        // Any type of employee is allowed to access this route
    }
}

class TasksController
{
    /** @Route("/tasks/{task}/assign/{developer}", methods={"POST"}) */
    public function assignTask(
       <<Authenticated>> Manager $manager
        Task $task,
        Developer $developer
    ): Response {
        // This is allowed only for managers
    }

    /** @Route("/tasks/{task}/complete", methods={"POST"}) */
    public function completeTask(
           <<Authenticated>> Developer $developer,
            Task $task
        ): Response {
        // This is allowed only for developers
    }
}
{% endhighlight %}

As a final step, we just have to let Symfony know how to transform the `UnauthenticatedException` and 
`UnauthorizedException` exceptions to a HTTP response. Here's just a simple example how this can be done:

{% highlight php startinline %}
class ExceptionsSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [KernelEvents::EXCEPTION => 'handleException'];
    }

    public function handleException(ExceptionEvent $event)
    {
        $statusCodes = [
            UnauthenticatedException::class => Response::HTTP_UNAUTHORIZED,
            UnauthorizedException::class => Response::HTTP_FORBIDDEN
        ];

        $exceptionClass = get_class($event->getThrowable());

        if (!isset($statusCodes[$exceptionClass])) {
            return;
        }

        $event->setResponse(
            new JsonResponse(/* ... */, $statusCodes[$exceptionClass])
        );
    }
}
{% endhighlight %}

### Conclusion

Annotations are widely used in other popular languages (such as Java) and having them now in the PHP core (as attributes) 
opens many new possibilities. As previously noted, the feature is accepted for PHP 8 and although it is working for the 
cases discussed here, it is still under development. The releasing of PHP 8 is currently scheduled for December 2020, 
more than half a year from now, a long period in which many things may happen or change. However, I believe that the 
general ideas discussed here will remain applicable.

Also, since PHP 8 is still in development, many of the libraries (including Doctrine) may not work with it at this 
moment, so making all the code presented here work requires switching between versions and doing some improvisations. 
Please do not focus too much on the details, as the important thing here is to understand the general idea.
