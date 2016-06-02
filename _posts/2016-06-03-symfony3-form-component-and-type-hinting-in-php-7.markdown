---
layout: post
title:  "Symfony3 Form Component and type hinting in PHP 7.0"
date:   2016-06-01 00:15:37
author: Dejan Angelov
categories:
---

Type hinting the methods' arguments and declaring their return types is one of the best improvements that came with PHP 7. When I finally got a chance to play with it in an experimental side project using Symfony, I got stuck trying to combine the type hinting and the Symfony3 Form component.

This post is an explaination of what happened in my case and what I did to make it work. It may not be the best solution and I still haven't tried to use this with a form that includes some more complex types (eg. EntityType).

Lets begin with a list of the two problems that occured:

1. Trying to access a property with a null value, using a setter declared to return strings.
2. Trying to pass a null value to a setter expecting a string.

For a better elaboration of the problems, let's imagine that we have a system for articles that only contain titles. Here's the simple entity class:

{% highlight php startinline %}
namespace AppBundle\Entity;

use Symfony\Component\Validator\Constraints as Assert;

class Article
{
    /**
     * @Assert\NotBlank(message="Each article must have a title.")
     */
    private $title;

    public function getTitle() : string
    {
        return $this->title;
    }

    public function setTitle(string $title)
    {
        $this->title = $title;
    }
}
{% endhighlight %}

And the custom form field type for the created entity:

{% highlight php startinline %}
namespace AppBundle\Form\Type;

use AppBundle\Entity\Article;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

class ArticleType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options)
    {
        $builder->add('title', TextType::class);
    }

    public function configureOptions(OptionsResolver $resolver)
    {
        $resolver->setDefault('data_class', Article::class);
    }
}
{% endhighlight %}

The first said problem occurs when we want to create the form and we provide the form factory an object of the entity class to read the default data from. 
The service then tries to read the properties of the object using its getter methods. If any of the properties has a null value, instead of a value whose type corresponds to the return type of the getter, and we don't have any logic in the getter (we just return the property's value as it is, like in the example above), we'd get an exception similar to the following one:

> Type error: Return value of AppBundle\Entity\Article::getTitle() must be of the type string, null returned<br />
> <sup>500 Internal Server Error - FatalThrowableError</sup>

To fix the problem in our example, we should set the default value of the `$title` property to be an empty string:

{% highlight php startinline %}
class Article
{
    ...
    private $title = "";
    ...
}
{% endhighlight %}

With what we've done by now, we fixed the first problem and we're getting the form rendered in our browser.

However, if we try to submit the form while leaving the "title" field empty, instead of having the validation component arguing about the field being blank (as we specified in the entity class), we'll get the following exception:

> Expected argument of type "string", "NULL" given<br />
> <sup>500 Internal Server Error - InvalidArgumentException</sup>

This is the second said problem, happening when the service tries to fill the object with the data submitted from the form. It will try to use the `setTitle(string $title)` method to set the value from the "title" field. Since we didn't fill anything in the field, it will try to pass null as a value to the setter that expects a string, so the system will throw the exception.

So, to solve the problem, we should have something to convert the value from null to an empty string before it's passed to the setter in the object.

When I looked for a solution around the web, the Stack Overflow user [HPierce](http://stackoverflow.com/users/3000068/hpierce) suggested that I can use a data transformer to fix the submitted data.

{% highlight php startinline %}
namespace AppBundle\Form\DataTransformer;

use Symfony\Component\Form\DataTransformerInterface;

class NullToEmptyStringDataTransformer implements DataTransformerInterface
{
    public function transform($value)
    {
        return $value;
    }

    public function reverseTransform($value)
    {
        return (is_null($value)) ? (string) $value : $value;
    }
}
{% endhighlight %}

As a final step, we should add the data transformer to the field in the method where we're building the form (in the ArticleType class):

{% highlight php startinline %}
public function buildForm(FormBuilderInterface $builder, array $options)
{
    ...

    $transformer = new NullToEmptyStringDataTransformer();
    $builder->get('title')->addModelTransformer($transformer);
}
{% endhighlight %}

If we now try to submit the form without any value in the "title" field, the data will be processed and we should get the specified validation error message.

If you have a comment or a better solution, please ping me on [Twitter](https://twitter.com/angelovdejan).

