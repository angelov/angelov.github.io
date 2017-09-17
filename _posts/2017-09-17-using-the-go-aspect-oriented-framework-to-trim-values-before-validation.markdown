---
layout: post
title:  "Using the Go! Aspect-Oriented Framework to trim values before validation"
date:   2017-09-17 12:15:00
author: Dejan Angelov
categories:
---

Few weeks ago, while using the `NotBlank` constraint from the Symfony Validator to validate a [command]({% post_url 2017-08-21-on-creating-cleaner-and-simpler-commands %}) before executing it, a co-worker of mine noticed that the values are not automatically trimmed, so it would allow strings of empty spaces to be passed as valid values.

We were already using the Symfony Forms component with the command classes set for the `data_class` property in the forms, and the trimming was [done there by default](http://symfony.com/doc/current/reference/forms/types/text.html#trim). As the commands can be initialized from another places in the applications as well, this is not enough and we must be sure that the data is still properly validated.

While looking for multiple approaches how this can be solved, it came to my mind that this is a case where we can finally give [Aspect-Oriented Programming](https://en.wikipedia.org/wiki/Aspect-oriented_programming) a try. 

My first contact with the AOP paradigm was few years ago while exploring the Spring framework in Java. My impression was that the AOP way of connecting things is too messy and can easily lead us to having a big mess in our applications, so I didn't dig much deeper into the approach.

However, after listening to Alexander Lisachenko's ["Solving Cross-Cutting Concerns in PHP"](https://www.youtube.com/watch?v=CIXfYbnom6s) talk at the PHP Serbia Conference this spring, although still suspicous about relying too much on the concept, I was kind of convinced that AOP can be useful for some simpler tasks in our applications.

The [Go! Aspect-Oriented Framework](https://github.com/goaop/framework) is a popular AOP framework in the PHP ecosystem (and probably the only one I had heard of before), so I chose it for the experiment.

Let's imagine that we want to validate the values passed in a `PostCommentCommand` command before executing it.

{% highlight php startinline %}
use Symfony\Component\Validator\Constraints as Assert;

class PostCommentCommand
{
    /** @Assert\NotBlank() */
    private $author;

    /** @Assert\NotBlank() */
    private $content;

    public function __construct(string $author, string $content)
    {
        $this->author = $author;
        $this->content = $content;
        // other stuff if needed...
    }

    // getters...
}
{% endhighlight %}

If we create an instance of the command and pass strings of empty spaces as values for the `$author` and `$content` arguments, we can see that no constraints are violated.

{% highlight php startinline %}
use Symfony\Component\Validator\Validation;

$validator = Validation::createValidatorBuilder()
    ->enableAnnotationMapping()
    ->getValidator();

$command = new PostCommentCommand('  ', '  ');

$violations = $validator->validate($command);

dump($violations);
{% endhighlight %}

{% highlight php startinline %}
ConstraintViolationList {#20 ▼
  -violations: []
}
{% endhighlight %}

When using the Go! AOP framework, in order to trim the values of the constructor arguments before validating them, we need to create an Aspect class where the actual trimming would take place.

> **Aspect**: A modularization of a concern that cuts across multiple objects. Logging, caching, transaction management are good examples of a crosscutting concern in PHP applications. Go! defines aspects as regular classes implemented empty Aspect interface and annotated with the @Aspect annotation.

In the aspect class, we'll have an method that we'd like to be called whenever we call a method whose string arguments need to be trimmed. When called, this method will receive an instance of the `MethodInvocation` class that will contain details about the originally called method, including a list of the values given as arguments in the call. We can then take the string arguments from the list and trim their values.

{% highlight php startinline %}
use Go\Aop\Aspect;
use Go\Aop\Intercept\MethodInvocation;

class TrimStringValuesAspect implements Aspect
{
    public function trimStringArgumentValues(MethodInvocation $invocation)
    {
        $invocation->setArguments(array_map(function($argument) {
            return is_string($argument) ? trim($argument) : $argument;
        }, $invocation->getArguments()));
    }
}
{% endhighlight %}

The Go! framework needs to know when it should call the method we just created. To tell it, we should define the method as an `Advice` and give it a properly defined way to match join points.

> **Advice**: Action taken by an aspect at a particular join point. There are different types of advice: @Around, @Before and @After advice.

In our concrete example, we need to call the aspect before the constructor of the command class is called, so we'll use the `@Before` advice annotation.

> **Before advice**: Advice that executes before a join point, but which does not have the ability to prevent execution flow proceeding to the join point (unless it throws an exception).

The join points matchers are defined using a concept called Pointcuts.

> **Pointcut**: A regular expression that matches join points. Advice is associated with a pointcut expression and runs at any join point matched by the pointcut (for example, the execution of a method with a certain name).

The first way for defining a pointcut we'll take a look at is to use a regular expression to define a pattern that will be used to check if the given advice should be called when a method in another place is being executed.

We would like the `trimStringArgumentValues()` method of our `TrimStringValuesAspect` class to be called before an instance of our command classes (all classes whose name ends with the word Command) is created, so we'll have the following advice definition:

{% highlight php startinline %}
/**
 * @Before("execution(public **\*Command->__construct(*))")
 */
public function trimStringArgumentValues(MethodInvocation $invocation){ ... }
{% endhighlight %}

Another thing we need to do is to register the aspect in the framework. We'll see how we can do that bellow in this article.

Having the aspect defined, if we validate the created command object again, we get the two expected violations:

{% highlight php startinline %}
ConstraintViolationList {#284 ▼
  -violations: array:2 [▼
    0 => ConstraintViolation {#305 ▼
      -message: "This value should not be blank."
      -root: PostCommentCommand {#279 ▶}
      -propertyPath: "author"
      -invalidValue: ""
      -constraint: NotBlank {#291 ▶}
      // ...
    }
    1 => ConstraintViolation {#306 ▼
      -message: "This value should not be blank."
      -root: PostCommentCommand {#279 ▶}
      -propertyPath: "content"
      -invalidValue: ""
      -constraint: NotBlank {#300 ▶}
      // ...
    }
  ]
}
{% endhighlight %}

Another way for defining when the advice should be called is to use an annotation pointcut. Defining it this way, it will allow the `trimStringArgumentValues()` method to be called before execution of any method in any class annotated with a custom annotation we'd create.

We'll create a new annotation class called, for example, `TrimmableArgumentValues`:

{% highlight php startinline %}
use Doctrine\Common\Annotations\Annotation;

/**
 * @Annotation
 * @Target("METHOD")
 */
class TrimmableArgumentValues extends Annotation
{
}
{% endhighlight %}

We'd now change the definition of the advice in our aspect to match only the methods annotated with our annotation:

{% highlight php startinline %}
/**
 * @Before("@execution(Acme\TrimmableArgumentValues)")
 */
public function trimStringArgumentValues(MethodInvocation $invocation) { ... }
{% endhighlight %}

The constructor method of our command class will not be automatically associated with the aspect anymore, so we'll annotate it with the `TrimmableArgumentValues` annotation:

{% highlight php startinline %}
class PostCommentCommand
{
    // ...

    /**
     * @TrimmableArgumentValues
     */
    public function __construct(string $author, string $content)
    {
        $this->author = $author;
        $this->content = $content;
        // other stuff if needed...
    }

    // ...
}
{% endhighlight %}

If we validate the command object again, we'll still get the same expected violations.

Integrating the Go! AOP framework into our applications is well described in the framework's documentation.

We should create a new kernel class that would extends the `Go\Core\AspectKernel` class. In this kernel we'll register our aspects.

{% highlight php startinline %}
use Go\Core\AspectContainer;
use Go\Core\AspectKernel;

class ApplicationAspectKernel extends AspectKernel
{
    protected function configureAop(AspectContainer $container)
    {
        $container->registerAspect(new TrimStringValuesAspect());
    }
}
{% endhighlight %}

Then, in our front controller (or some other proper place in the application) we'll initialize and configure the kernel:

{% highlight php startinline %}
$applicationAspectKernel = ApplicationAspectKernel::getInstance();
$applicationAspectKernel->init([
    'debug' => true,
    'cacheDir'  => __DIR__ . '/cache',
    'includePaths' => [
        __DIR__ . '/src/'
    ]
]);
{% endhighlight %}

If we want to use it withing the Symfony Framework, we can use the official [GoAopBundle](https://github.com/goaop/goaop-symfony-bundle) bundle.

Having this bundle installed, we only need to register our aspects as a services and tag them with a `goaop.aspect` tag:

{% highlight yaml %}
services:
    Acme\TrimStringValuesAspect:
        tags:
            - { name: goaop.aspect }
{% endhighlight %}

As I said before, this is a first time I'm giving the AOP approach a try, so I can't guarantee if this way of dealing with the argument values is actually good on a long term basis.

If you want to share any opinions, ping me on [Twitter](https://twitter.com/angelovdejan).
