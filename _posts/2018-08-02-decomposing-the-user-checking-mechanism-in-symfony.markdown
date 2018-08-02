---
layout: post
title:  "Decomposing the user checking mechanism in Symfony" 
date:   2018-08-2 10:07:00
author: Dejan Angelov
categories:
---

The user checkers in the Symfony Security component allow us to control whether an user can be authenticated in our system, 
even if they have entered their correct credentials. In this article we'll see how we can make the process more flexible by 
splitting our checking logic from one to multiple checkers.

As written in the documentation, we can set a custom user checker for each of the firewalls we have in our security configuration:

{% highlight yaml %}
security:
    firewalls:
        main:
            pattern: ^/
            user_checker: App\Security\UserChecker
{% endhighlight %}

Let's imagine that we are developing an application that has the following requirements for the users in order for them to be 
allowed to login into the system:
    
* The user has an approved account
* The user is not banned
* The user has a valid subscription

### Using a general User Checker

As the configuration allows us to configure only one service as user checker per firewall, it may seem convenient for us to put all 
those checks in a single user checker class:

{% highlight php startinline %}
class UserChecker implements UserCheckerInterface
{
    private $bansChecker;
    private $subscriptionValidator;

    public function __construct(
        UserBansChecker $bansChecker,
        SubscriptionValidator $subscriptionValidator
    ) {
        $this->bansChecker = $bansChecker;
        $this->subscriptionValidator = $subscriptionValidator;
    }

    public function checkPreAuth(UserInterface $user) : void
    {
        if (!$user->isApproved()) {
            throw new UserNotApprovedAuthenticationException();
        }
    }

    public function checkPostAuth(UserInterface $user) : void
    {
        if ($this->bansChecker->isBanned($user)) {
            throw new UserBannedAuthenticationException();
        }

        if ($this->subscriptionValidator->hasValidSubscription($user)) {
            throw new SubscriptionExpiredAuthenticationException($user);
        }
    }
}
{% endhighlight %}

If we're working with a bigger and more complex application, the logic of the checkers may be distributed along multiple 
modules in the system. Although it's simple, in the example above we can see that the rules for the checking 
can come from different modules like "Subscriptions", "Users", "User Banning", etc.

Each of those modules may have its own business logic, and having this one user checker, we're moving pieces of that logic 
to a single place, perhaps in some "Authentication" module.

We already know that business requirements always change, so we can certainly be sure that the business logic we're working 
with in our application will have to change as well.

Let's imagine that we've stopped to support our services in some countries, so we'll have to disallow the users from those 
countries to login into the application. In such case we'll have to modify our existing class, clearly violating the
*"Open–closed principle"*. Besides having to modify our checker class, we'll also have to modify our tests for this class, 
to mock more services there and possibly to make the existing cases more complicated. And the same applies for removing a check 
too.

For most of the rules included in the checker, we'll have to inject some service. As we'll be adding more and more rules,
this list will grow too, making the checker depending on more and more services.

### Decomposing the checking logic

Instead of having all the checks combined in a single class, we can split them and define the following three checkers:

{% highlight php startinline %}
class UserApprovalStatusUserChecker implements UserCheckerInterface
{
    public function checkPreAuth(UserInterface $user)
    {
        if (!$user->isApproved()) {
            throw new UserNotApprovedAuthenticationException();
        }
    }

    public function checkPostAuth(UserInterface $user)
    {
    }
}
{% endhighlight %}

{% highlight php startinline %}
class BannedUsersUserChecker implements UserCheckerInterface
{
    private $bansChecker;

    public function __construct(UserBansChecker $bansChecker)
    {
        $this->bansChecker = $bansChecker;
    }

    public function checkPreAuth(UserInterface $user): void
    {
    }

    public function checkPostAuth(UserInterface $user): void
    {
        if ($this->bansChecker->isBanned($user)) {
            throw new UserBannedAuthenticationException();
        }
    }
}
{% endhighlight %}

{% highlight php startinline %}
class SubscriptionUserChecker implements UserCheckerInterface
{
    private $subscriptionValidator;

    public function __construct(SubscriptionValidator $subscriptionValidator)
    {
        $this->subscriptionValidator = $subscriptionValidator;
    }

    public function checkPreAuth(UserInterface $user)
    {
    }

    public function checkPostAuth(UserInterface $user)
    {
        if ($this->subscriptionValidator->hasValidSubscription($user)) {
            throw new SubscriptionExpiredAuthenticationException($user);
        }
    }
}
{% endhighlight %}

Having them separated, we can organize the classes better by moving them to the proper modules they belong to.

Each of the checkers can now be tested separately, leading to cleaner, simpler and more maintainable tests.

Let's go back to the case when we need to add an additional user checker. To achieve the same, now we only have to write the 
new user checker class, without touching the existing ones. As the new class can be tested separately, the unit tests for the other 
checkers can remain untouched as well.

Removing an user checker now is just a matter of removing the checker class.

As the checkers are now defined per module instead of a central place, we can easily extend the checking process by adding 
new modules which will have their own checkers defined.

### Wiring the checkers together

We remember that the configuration of the Symfony Security component allows us to define only one user checker per firewall. 
As we now have more than one checker, we'll create a composite user checker which will hold the other checkers and call them 
when needed. We'll register that checker as a service and configure it as checker for our firewall.

{% highlight php startinline %}
class CompositeUserChecker implements UserCheckerInterface
{
    /** @var UserCheckerInterface[] */
    private $checkers = [];

    public function addChecker(UserCheckerInterface $userChecker)
    {
        $this->checkers[] = $userChecker;
    }

    public function checkPreAuth(UserInterface $user)
    {
        foreach ($this->checkers as $checker) {
            $checker->checkPreAuth($user);
        }
    }

    public function checkPostAuth(UserInterface $user)
    {
        foreach ($this->checkers as $checker) {
            $checker->checkPostAuth($user);
        }
    }
}
{% endhighlight %}

{% highlight yaml %}
security:
    firewalls:
        main:
            pattern: ^/
            user_checker: Acme\Core\Security\CompositeUserChecker
{% endhighlight %}

The next thing we need is the mechanism of adding our checkers to the composite one. 

We'll register all the checkers as services and tag them with a `user_checker` tag.

{% highlight yaml %}
services:

    Acme\Users\Authentication\UserApprovalStatusUserChecker:
        tags: ['user_checker']

    Acme\Users\Banning\Authentication\BannedUsersUserChecker:
        tags: ['user_checker']

    Acme\Subscriptions\Authentication\SubscriptionUserChecker:
        tags: ['user_checker']
{% endhighlight %}

Alternatively, since all of our checkers implement the `Symfony\Component\Security\Core\User\UserCheckerInterface` interface, instead of 
manually adding the tags, we can register the interface for autoconfiguration in our Kernel.

{% highlight php startinline %}
class Kernel extends BaseKernel
{
    // ...

    protected function build(ContainerBuilder $container)
    {
        $container
            ->registerForAutoconfiguration(UserCheckerInterface::class)
            ->addTag('user_checker');
    }
}
{% endhighlight %}

We now have our checkers registered and configured. The final step is to add them to the composite checker. To do that, 
we'll have a compiler pass that will find the tagged checkers when compiling the container, and add them to the composite 
checker:

{% highlight php startinline %}
class UserCheckersCompilerPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container)
    {
        $mainChecker = $container->findDefinition(CompositeUserChecker::class);

        $userCheckers = $container->findTaggedServiceIds('user_checker');

        foreach ($userCheckers as $checker => $tags) {
            if ($checker === $mainChecker->getClass()) {
                continue;
            }

            $mainChecker->addMethodCall('addChecker', [new Reference($checker)]);
        }
    }
}
{% endhighlight %}

### Further ideas

In bigger and more complex applications, it is very likely that we'll have multiple firewall with different authentication 
requirements. 

As the checks for all the requirements are now represented as separate classes, we can easily combine any of them in multiple 
instances of the composite checker and use them for the proper fiewalls.

As a final note, I would like to note that even though we've made the checking process more flexible, we're still coupling part 
of our business logic to the Symfony Security component. Let's have that topic for a possible future article :)
