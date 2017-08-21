---
layout: post
title:  "On creating cleaner and simpler commands"
date:   2017-08-21 09:10:00
author: Dejan Angelov
categories:
---

Commands, as part of the CQRS pattern, provide a clean way for representing intentions for changing some state in our application. We’ve been working with CQRS for some time and it helped us significantly improve the quality of our code base. Meanwhile, we learned a lot, but we also made some mistakes. In this article we’ll discuss some of the mistakes we were making when using commands and what we did to improve them.

Imagine that, for example, we want to promote an employee to another position. We would have something like the following:

{% highlight php startinline %}
class PromoteEmployeeCommand
{
    private $employeeId;
    private $position;

    public function __construct(string $employeeId, string $position)
    {
        $this->employeeId = $employeeId;
        $this->position = $position;
    }

    public function getEmployeeId() : string
    {
        return $this->employeeId;
    }

    public function getPosition() : string
    {
        return $this->position;
    }
}
{% endhighlight %}

Instead of handling the business logic needed for changing an employee’s position inside a controller, we can now call this command, making our application independent of the HTTP layer. At same time, we let the controller serve their right purpose – transforming the received data and sending it to the application’s model layer.

Taking the needed actions to actually promote the employee will be handled in a separate “handler” class:

{% highlight php startinline %}
class PromoteEmployeeCommandHandler
{
    public function handle(PromoteEmployeeCommand $command) : void
    {
        // ...
    }
}
{% endhighlight %}

If you're not already familiar with the concept, you can read about CQRS, different types of messages for communication within the application and more in [this article](https://www.ibuildings.nl/node/262) by Matthias Noback.

However, there are cases when, if we don’t model our commands properly, we can introduce mess and confusion into our application and even make it hard to scale and debug.

Such case is when we need to have commands for creating new resources. In that case, the amount of data we receive with the request may contain a long list of properties and values. In this article we’ll discuss some techniques for creating simpler and cleaner commands for this type of cases.

Let's imagine that we need to employ a new person in our company. Every employee would have *first name*, *last name*, *email address*, *phone number*, *position* and *salary*.

# Avoiding data arrays as only argument for new commands

Knowing that our HTTP library can easily provide us the needed data as an array extracted from the request, it may be tempting for us to make our command receive that array and let the appropriate handler deal with it:

{% highlight php startinline %}
class EmployNewPersonCommand
{
    private $data;

    public function __construct(array $data)
    {
        $this->data = $data;
    }

    public function getData() : array
    {
        return $this->data;
    }
}
{% endhighlight %}

To create a new instance of this command, we just ask the request for the data and then pass the same array to the command:

{% highlight php startinline %}
public function employAction(Request $request)
{
    $employeeData = $request->request->all();

    $command = new EmployNewPersonCommand($employeeData);
    // ...
}
{% endhighlight %}

Modeling the commands this way may seem really clean, allowing us to more rapidly create and execute new commands. However, as our application grows, this can become really messy and can cause a lot of confusion.

One of the main problems here is that, whenever we need to create a new instance of the command, there is no information about what should be put inside the expected array of data. And don’t forget that the purpose of the commands is not to be called just from one point in our application (eg. only one controller method).

With this problem, when somebody needs to create a new instance of the command, instead of just telling what they need to be done, they must find the appropriate handler for the needed command, dig inside it and determine what are the proper keys of the array and what should be put inside them.

Even then, some of the keys may be mistaken or missed, so we are given a possibility to create commands with incomplete or invalid data.

To ensure all the required information is passed to the command when it is created, we need to do some key existence checking in the commands’ constructors, or even worse, in their handlers.

We can avoid such problems by making sure we always list all required information as separate arguments with proper types in the commands’ constructors, making the commands impossible to instantiate if something is missing:

{% highlight php startinline %}
class EmployNewPersonCommand
{
    private $firstName;
    private $lastName;
    private $emailAddress;
    private $phoneNumber;
    private $position;
    private $salary;

    public function __construct(
        string $firstName,
        string $lastName,
        string $emailAddress,
        string $phoneNumber, 
        string $position, 
        float $salary
    ) {
        $this->firstName = $firstName;
        $this->lastName = $lastName;
        $this->emailAddress = $emailAddress;
        $this->phoneNumber = $phoneNumber;
        $this->position = $position;
        $this->salary = $salary;
    }

    public function getFirstName() : string
    {
        return $this->firstName;
    }
    
    public function getLastName() : string
    {
        return $this->lastName;
    }

    // other getters
}
{% endhighlight %}

# Using value objects for cleaner command arguments

You probably notice another problem in our command’s current design – it became bloated with long list of properties and getter methods.

We should also validate the information inside all of those properties, so our command can become even bigger and more messy.

To make our command simpler, we can create multiple value objects in which we can group and organize the values, and use them as arguments for our command.

For example, we can group the first and last name in a `PersonalDetails` value object. The email address and phone number would get a separate `EmailAddress` and `PhoneNumber` value objects. For the position and the salary we can create the `Employment` class. Note that this is just an example of how the information can be organized and you should think of your domain logic when creating the classes.

Each of the value object can be validated independently, which gives us even better separation.

With this reorganization, our command now expects three defined arguments:

{% highlight php startinline %}
class EmployPersonCommand
{
    private $personalDetails;
    private $contacts;
    private $employment;

    public function __construct(PersonalDetails $personalDetails, array $contacts, Employment $employment)
    {
        $this->personalDetails = $personalDetails;
        $this->contacts = $contacts;
        $this->employment = $employment;
    }

    public function getPersonalDetails() : PersonalDetails
    {
        return $this->personalDetails;
    }

    public function getContacts() : array
    {
        return $this->contacts;
    }

    public function getEmployment() : Employment
    {
        return $this->employment;
    }
}
{% endhighlight %}

To create a new instance, firstly we need to prepare the value objects and then give them to the command:

{% highlight php startinline %}
$personalDetails = new PersonalDetails(
    $request->get('firstName'), 
    $request->get('lastName')
);

$email = new EmailAddress($request->get('email'));
$phoneNumber = new PhoneNumber($request->get('phoneNumber'));

$employment = new Employment(
    $request->get('position'), 
    $request->get('salary')
);
        
$command = new EmployPersonCommand(
    $personalDetails, 
    [$email, $phoneNumber], 
    $employment
);
{% endhighlight %}

We can reuse the same value objects in multiple places in our application, for example to make the entities or form types cleaner as well.

# Using multiple commands instead of just one

If the domain logic we’re working on allows us, we can improve the command to have less responsibilities and to cause less implications by separating it into multiple smaller commands.

Currently, our command receives multiple types of information about our new employee – their personal details, contact details and details about the new employment.

If we need a possibility to add more contact details to an existing employee in future, we would create a new command for that purpose, possibly causing us to duplicate some existing code. If our domain logic allows for an employee to exist without providing us contact details, we can extract this into separate command which we can later use in another use case.

We may even be able to firstly store the person’s personal details before we employ them, allowing us to store details about persons related to us in a different ways (eg. interns).

So, instead of taking all this action by calling a single command, we can have the following three commands, which we can call from multiple places and in multiple cases:

{% highlight php startinline %}
class CreatePersonCommand { ... }
{% endhighlight %}

{% highlight php startinline %}
class EmployPersonCommand { ... }
{% endhighlight %}

{% highlight php startinline %}
class AddContactDetailsForEmployee { ... }
{% endhighlight %}

And to call them one by one:

{% highlight php startinline %}
public function employAction(Request $request)
{
    // ...

    $this->handle(new CreatePersonCommand(...));
    $this->handle(new EmployPersonCommand(...));
    $this->handle(new AddContactDetailsForEmployee(...));
}
{% endhighlight %}

Note that, since the commands are not supposed to be returning anything, in order to know the identity of the created person who we want to employ, we should generate their ID before the commands are called. Another thing we need to be careful is that these commands should be called synchronically, because we must be sure that the person we want to employ already exists in our application.

{% highlight php startinline %}
public function employAction(Request $request)
{
    // ...

    $id = $this->uuids->generate();

    $this->handle(new CreatePersonCommand($id, ...));
    $this->handle(new EmployPersonCommand($id, ...));
    $this->handle(new AddContactDetailsForEmployee($id, ...));
}
{% endhighlight %}

As you’ve read here in this article there are some ways to improve our commands’ design. You should always be aware of the requirements of the domain of your application before taking such decisions.

In general, always try to make your commands concise, simpler and reusable.

<div class="alert alert-info">
	<strong>Note:</strong> This article was originally published on my employer's web site. You can also read it <a href="http://gsix.me/post/creating-cleaner-simpler-commands/">here</a>. Any republishing of the article as a whole or in parts should be done in compliance with that site's content licensing.
</div>
