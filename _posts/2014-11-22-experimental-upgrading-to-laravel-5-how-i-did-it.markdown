---
layout: post
title:  "Experimental upgrading to Laravel 5: How I did it"
date:   2014-11-22 20:14:35
author: Dejan Angelov
categories:
---

When I started working on my biggest side project, the "EESTEC Platform for Local Committees" I introduced [a few months ago]({% post_url 2014-08-05-introducing-eestec-platform-for-local-committees %}), the latest stable version of Laravel was 4.2. Over the past weeks, Taylor introduced many great changes and new features that we'll be able to use in the new version, firstly numbered 4.3 and later 5. According to the framework's six month release cycle, it should had hit stable late this month or in early December. Because of that, I started to play with it and to apply the changes to make my application use it.

However, a couple of days ago, Taylor wrote a [blog post](http://blog.laravel.com/laravel-5-release-dates/) on the Laravel's blog saying that because of the importance of this release, the release date will be postponed to January. Considering this, everything you'll read here MUST NOT be applied to applications that are currently in production. There isn't any documentation for many of the new features yet and to implement them I had to dig into the framework's source code. Some of them even didn't work properly and had bugs. The framework is still in alpha state and it is possible for some of the things written here to be changed at the time you read this article. Some of the things may not be aplicable to your application. Consider this more as a report of how I did everything and not how you must do the same. The latest commit to the develop branch of the "laravel/laravel" repository when this article was edited last time was [this one](https://github.com/laravel/laravel/commit/b73e127ed07034a2a9aa6b8598c00b918287b181).

I also didn't want to write a deeper explaination on some changes because there are already many great sources where you can read more about them. For example, you can take a look on the Matt Stauffer's [series of blog posts](http://mattstauffer.co/blog/laravel-5.0-form-requests) about the new Laravel features, or watch some videos about them on [Laracasts](https://laracasts.com/series/whats-new-in-laravel-5).

And again, **do not use Laravel 5 in production yet**.

Dependencies
---

The starting point of the upgrading is the 3rd party dependencies we use in our application. That means that our first step will be to open "composer.json" and change the required version of the "laravel/framework" package to "5.0.*@dev" and hit `composer update`.

Personally, I’ve never been a fan of the Blade templating engine and I love Twig instead. To be able to use Twig for my presentation layer, I had to use some bridge library. When I hit update for the first time, I got an error saying that the "[barryvdh/laravel-twigbridge](https://github.com/barryvdh/laravel-twigbridge)" library requires the "illuminate/view 4.x" component and is not suited to be used with Laravel 5. Then I realized that this library is now deprecated in favor of another one and now I'm requiring "[rcrowe/twigbridge](https://github.com/rcrowe/TwigBridge)": "0.6.\*" instead. Another thing I needed to change was the required version of "[way/generators](https://github.com/JeffreyWay/Laravel-4-Generators)" from "2.\*" to "~3.0". When I hit update this time it finished with success.

“Turn On The Lights”
----

When I tried to open the application in browser or to use Artisan, I was getting an error saying "Call to undefined method Illuminate\Foundation\Application::bindInstallPaths()". This happens because in the previous versions of Laravel, there was a "start.php" file in the "bootstrap/" folder that was included every time the application was starting. The file no longer exists in Laravel 5 so we need to change the "Turn On The Lights" section in the "public/index.php" and "./artisan" files. Open those files and find the sections that looks like this:

{% highlight php startinline %}
/*
|--------------------------------------------------------------------------
| Turn On The Lights
|--------------------------------------------------------------------------
|
| We need to illuminate PHP development, so let's turn on the lights.
| This bootstraps the framework and gets it ready for use, then it
| will load up this application so that we can run it and send
| the responses back to the browser and delight these members.
|
*/
 
$app = require_once __DIR__.'/bootstrap/start.php';
{% endhighlight %}

If we take a look on [GitHub](https://github.com/laravel/laravel/blob/develop/artisan#L18), we can see that instead of bootstrap/start.php, we should require boostrap/app.php.

{% highlight php startinline %}
$app = require_once __DIR__.'/bootstrap/app.php';
{% endhighlight %}

Note that the path to the "app.php" file is a little different in each of the two files and make sure you wrote the correct one. And here comes an another problem. There's no such file as "bootstrap/app.php", so we need to create it by ourselves. Peek into the laravel/laravel repository, grab the "app.php" [content](https://github.com/laravel/laravel/blob/develop/bootstrap/app.php) and put it in your new file.

Starting and shutting down the application
---

The next thing that I saw was that there’re also changes in the ways the application is being started and shutted down.
The "Load The Artisan Console Application" section in the ./artisan file is now removed and to run the application, instead of calling `$artisan->run()` we now have:

{% highlight php startinline %}
$status = $app->make('Illuminate\Contracts\Console\Kernel')->handle(
    new Symfony\Component\Console\Input\ArgvInput,
    new Symfony\Component\Console\Output\ConsoleOutput
);
{% endhighlight %}

To shut down the application we no longer call `$app->shutdown()` so we can remove that line and just leave `exit($status)`.

We need to make similar changes in "public/index.php" as well. To run the application, until now, we were just calling `$app->run()`. From now on, this section is changed to:

{% highlight php startinline %}
$kernel = $app->make('Illuminate\Contracts\Http\Kernel');
 
$response = $kernel->handle(
    $request = Illuminate\Http\Request::capture()
);
 
$response->send();
 
$kernel->terminate($request, $response);
{% endhighlight %}

“Register The Auto Loader”
---

Handling the auto loading in Laravel is configured in the bootstrap/autoload.php file. As described in this [article](https://medium.com/rebuilding-laravel/rebuilding-laravel-part-1-autoloading-a7bc8029ca0b), in order to load the needed classes, Laravel was doing a  few steps. It was including the Composer’s autoload.php file, checking if there’s compiled class file which contains all of the classes commonly used by a request, registering its own ClassLoader in order to “catch” all the classes not yet mapped by Composer and registering some workbench loaders.

In the new version, we can see that the file only contains the "Register The Composer Auto Loader" and "Include The Compiled Class File" sections. The "Register The Laravel Auto Loader" and "Register The Workbench Loaders" sections are now removed.

The path of the compiled class file is now changed and the file is placed in the storage folder, so we need to change the way it is loaded:

{% highlight php startinline %}
$compiledPath = __DIR__.'/../storage/framework/compiled.php';
 
if (file_exists($compiledPath))
{
    require $compiledPath;
}
{% endhighlight %}

The file also contained a "Setup Patchwork UTF-8 Handling" section which is now removed as well.

If you try to open the application now, you'll probably get an "Class App\Http\Kernel does not exist" exception. That's because Laravel now autoloads almost everything using PSR-4 and not classmap. We will fix this error soon.

New directory structure
---

One of the main reasons why the planned Laravel version 4.3 became 5 was the big changes in the directory structure. Let's take a look at the base folders of the two versions, 4.2 and 5.

<pre style="float:left;width:45%;display:inline-block;">Laravel 4.2:
.
├── app
├── bootstrap
├── public
├── vendor
├── artisan
├── composer.json
├── composer.lock
├── CONTRIBUTING.md
├── phpunit.xml
├── readme.md
└── server.php
</pre>
<pre style="float:right;width:45%;display:block; clear: right;">Laravel 5:
.
├── app
├── bootstrap
├── config
├── database
├── public
├── resources
├── storage
├── tests
├── vendor
├── artisan
├── composer.json
├── composer.lock
├── gulpfile.js
├── package.json
├── phpunit.xml
├── readme.md
└── server.php
</pre>

<br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br /><br />

We can quickly notice that many of the folders that previously were in "app/" are now moved to the base folder. So, let's move "app/config", "app/database", "app/storage" and "app/tests" to their new home. The next thing to do is to create a new "resources" folder in the base folder and move "app/lang" and "app/views" there. In the resources we can also create an "assets" folder where we’ll later move the css and js files.

What we have left in the "app/"" folder is: the "commands", "controllers" and "start" folders, and the "filters.php" and "routes.php" files. I also have a "platform" folder where I put my application’s logic and from where I'm autoloading it using PSR-4. We’ll talk more about it later, but for now just know that everything in the "app/"" folder is now namespaced, including the controllers. Now, in "app/" create a folders called "Http", "Console", "Exceptions" and "Providers".

#### Http

As the name says, the logic based here will be in charge of everything related to the HTTP requests. For now, inside this one create three new folders called "Controllers" (and move the controller files here), "Middleware" (the mechanism that now replaces the old filters, we’ll talk about them later) and "Requests" (will talk later about them too). Another thing we need to do is to move the "routes.php" file here and also create a new one called "Kernel.php". The source for the new file can be taken from [here](https://github.com/laravel/laravel/blob/develop/app/Http/Kernel.php).

#### Console

If you have any custom developed Artisan commands (I don’t) move them from the commands them here.

#### Exceptions

In this namespace we'll have our exception handlers.

#### Providers

This is the place where our service providers will live. For now, enter the folder and create those three files: "AppServiceProvider.php", "EventServiceProvider.php" and "RouteServiceProvider.php". Feel free to check them on GitHub and grab their default source from there.

In the "app/start" folder I used to have an "ioc.php" file which was used to bind some interfaces to their implementations using the IoC container. I was later including this file at the end of the "global.php" file. If you also have some custom code in the "global.php" or "local.php" files, move it to the proper service providers and delete the said folder.

For now, I will also delete my "platform" folder and I will think what to do with those files later.

**Important**: After making the changes in the structure, be sure to check the ".gitignore" file so no sensitive data gets to the repository.

Default base namespace
----

As I already mentioned before, all the classes in the "app/" folder are now namespaced. While you were getting the default source code for some of the new files, you probably noticed that they’re placed in the "App\\" base namespace. Now, put the remaining of the classes in this namespace (We’ll change it in a moment, I promise).

To automatically change that base namespace we can now use the new Artisan command called “app:name“.

Oops! We still can’t use Artisan because of the previously mentioned error, the “App\Console\Kernel” class was not found.

That’s because I missed that this file is new and we should create it. So, create new "Kernel.php" file in the "app/Console" folder and get its content from [here](https://github.com/laravel/laravel/blob/develop/app/Console/Kernel.php). Notice that the `$commands` attribute contains the "App\Console\Commands\InspireCommand" command. I didn’t grab this one because I don’t need it so I removed it from the array.

{% highlight php startinline %}
/**
 * The Artisan commands provided by your application.
 *
 * @var array
 */
protected $commands = [];
{% endhighlight %}

If you have your own Artisan commands, don't forget to register them here.

However, we still haven't changed Composer to load the classes properly.

Fixing the autoloading
---

Lets open the "composer.json" file and see what we have there. The autoload part of the mine composer file looks like this:

{% highlight javascript %}
"autoload": {
    "classmap": [
        "app/commands",
        "app/controllers",
        "app/database/migrations",
        "app/database/seeds",
        "app/tests/TestCase.php"
    ],
    "psr-4": {
        "Angelov\\Eestec\\Platform\\": "app/platform"
    }
},
{% endhighlight %}

Since now everything is namespaced, we don’t need this class-mapping stuff anymore (except for the database folder). The new autoload block is:

{% highlight javascript %}
"autoload": {
    "classmap": [
        "database",
        "tests/TestCase.php"
    ],
    "psr-4": {
        "App": "app/"
    }
},
{% endhighlight %}

Hit `composer dump-autoload` and everything will work as it should. Not. Not yet :). We’re now getting another error that says "Call to undefined method Illuminate\Foundation\Bootstrap\ConfigureLogging::configureHandler()".

Fixing the configuration files
---

If we take a look at the "/vendor/laravel/framework/src/Illuminate/Foundation/Bootstrap/ConfigureLogging.php file", we can see this block of code:

{% highlight php startinline %}
/**
* Configure the Monolog handlers for the application.
*
* @param  \Illuminate\Contracts\Foundation\Application  $app
* @param  \Illuminate\Log\Writer  $log
* @return void
*/
protected function configureHandlers(Application $app, Writer $log)
{
    $method = "configure".ucfirst($app['config']['app.log'])."Handler";
    $this->{$method}($app, $log);
}
{% endhighlight %}

Here we can see that this is dynamically constructing the method's name according to the value set in `$app['config']['app.log']`. If we take a look in the "app.php" file in the config folder, we can see that we don't have a "log" property there. So, add it.

{% highlight php startinline %}
/*
|--------------------------------------------------------------------------
| Logging Configuration
|--------------------------------------------------------------------------
|
| Here you may configure the log settings for your application. Out of
| the box, Laravel uses the Monolog PHP logging library. This gives
| you a variety of powerful log handlers / formatters to utilize.
|
| Available Settings: "single", "daily", "syslog"
|
*/
'log' => 'daily',
{% endhighlight %}

While we're here, let's also fix the list of registered service providers. There are a few new default service providers used by the framework and a few of the existing are now removed (or renamed, i guees). Open the file on [GitHub](https://github.com/laravel/laravel/blob/develop/config/app.php), navigate to the "Autoloaded Service Providers" section, compare it with your file and do the changes. Also, be careful not to remove some of the other vendor libraries or your service providers.

Since there are also some changes in the "Class Aliases" section, make sure you check it as well.

And the last thing that we need to change here is the "manifest" property from `storage_path() . '/meta'` to `storage_path().'/framework'`.

The "app.php" config file is now okay, but let's also check what else in this folder needs to be changed. There are two new files: "filesystems.php" and "services.php". Grab their default content and move on.

Next, check the existing files for any needed modifications. From what I've seen there, almost nothing has been changed in the structure of the configuration files. Remember that the views folder was moved to the base directory? Make sure you don't forget to change that in the view.php file.

The storage folder
---

On the mission to be able to use Artisan again, I faced a new error: No such file/directory as "storage/framework/services.json". The structure of the "storage" folder has also changed. From what we can see [here](https://github.com/laravel/laravel/tree/develop/storage), some folders were deleted and some moved. Do the same changes to your folder, and note that some of the ".gitignore" files inside them were changed too.

Change the default base namespace
---

I hit `php artisan` and I successfuly got the Artisan commands list. It had been a long time since I had seen a nice Artisan output :)
Remember that I promised that we'll change the default base namespace? Let's do it now.

As mentioned before, to change the default namespace we can simply use the "app:name" Artisan command. Hit `php artisan app:name your-namespace-here` and it will change the namespace across the many files in the application (including the config ones).

I had a little problem here with the "\\" character in my "Angelov\\Eestec\\Platform" namespace. At first I wrote the namespace in the command like this `php artisan app:name "Angelov\\\\Eestec\\\\Platform"` because I knew that I need to have double backslashes in composer.json. It did end well in this file, but not in the other ones. There I got something like 

{% highlight php startinline %}
namespace Angelov\\Eestec\\Platform\Console;
{% endhighlight %}

I just simply wrote `php artisan app:name "Angelov\Eestec\Platform"` and I only had to change the namespace in "composer.json" and everywhere else it was okay.

Now just hit `composer dump-autoload` and we're done for now.

Configure the Twig bridge
---

One of the first things I did when I started the upgrading was changing the Twig bridge library I was using, but I wasn't able to configure it because Artisan didn't work. What I noticed here is that the old "config:publish" command is now renamed to "publish:config".

First sights of working state
---

Since our application seems to work, let's get our main code located in the "platform" folder (the one I deleted before). I also deleted the [file](https://github.com/angelov/eestec-platform/blob/f0fdb0e4838f6d74f03c2232576e4b6ba588a070/app/start/ioc.php) in which I was binding some interfaces to their implementation, so I created two new ServiceProviders in which I put the binding logic.

Missing Illuminate\Auth\UserInterface
---

The next error I was faced with was that the "Illuminate\Auth\UserInterface" is not around anymore. I was using that interface in my Member entity. I checked the default User class that comes with Laravel and saw that the "UserInterface" and "RemindableInterface" are renamed to "Authenticatable" and "CanResetPassword" respectfully, and are now part of the new "illuminate/contracts" package. The traits that were used for implementing those interfaces are also renamed to "Authenticatable" and "CanResetPassword". So, here's my new Member class:

{% highlight php startinline %}
// ...
use Illuminate\Auth\Authenticatable;
use Illuminate\Auth\Passwords\CanResetPassword;
use Illuminate\Contracts\Auth\Authenticatable as AuthenticatableInterface;
use Illuminate\Contracts\Auth\CanResetPassword as CanResetPasswordInterface;

class Member extends Model implements AuthenticatableInterface, CanResetPasswordInterface
{
    use Authenticatable, CanResetPassword;
    // ...
}
{% endhighlight %}

Making the facades work
---

Although I'm not a fan of the Laravel's facades, I'm still using them in some places in the controllers. For example, I'm rendering the views using `View::make()`. Since our controllers are now namespaced, the system thinks that the facades are in the same namespace too. For now, let's just import them by adding

{% highlight php startinline %}
use View;
{% endhighlight %}

(or similar).

Speaking of the View facade, if you use the same Twig bridge and have some "Unresolvable dependency resolving" problem, check out this [issue](https://github.com/rcrowe/TwigBridge/issues/150) on GitHub.


Filters went Middlewares
---

The concept of filters in Laravel was changed few time in the development process of the new framework's new version. At first, they were moved from the "filters.php" file to separate classes and later they were replaced with middlewares. [This article](http://mattstauffer.co/blog/laravel-5.0-middleware-replacing-filters) explains how the middlewares work and how can we use them as we were using "before" and "after" filters in our applications.

The "filters.php" file used to contain multiple default filters such as "auth" and "guest". Those filters are now replaced with the "[Authenticate](https://github.com/laravel/laravel/blob/develop/app/Http/Middleware/Authenticate.php)" and "[RedirectIfAuthenticated](https://github.com/laravel/laravel/blob/develop/app/Http/Middleware/RedirectIfAuthenticated.php)" middlewares that we should put in the "app/Http/Middleware" folder.

I also had two custom filters stored in their own classes, "BoardMembersFilter" and "AjaxFilter". To create a new middleware we can use the "make:middleware" Artisan command, for example: `php artisan make:middleware AjaxOnlyMiddleware`.

So, my "AjaxFilter" class became "AjaxOnlyMiddleware" and instead of 

{% highlight php startinline %}
public function filter()
{
    if (!$this->request->ajax()) {
        throw new NotAllowedException();
    }
}
{% endhighlight %}

I now have

{% highlight php startinline %}
public function handle($request, Closure $next) 
{
    if (!$request->ajax()) {
        throw new NotAllowedException();
    }

    return $next($request);
}
{% endhighlight %}

Note that after our actions in the middleware we should pass the request further by calling `$next($request)`.

If we open the "Http\Kernel" class we can see that it contains a "$routeMiddleware" attribute where we can register our middlewares. The three default middlewares are already registered for us so we just need to add the custom ones.

{% highlight php startinline %}
protected $middleware = [
    'auth' => 'Angelov\Eestec\Platform\Http\Middleware\Authenticate',
    'auth.basic' => 'Illuminate\Auth\Middleware\AuthenticateWithBasicAuth',
    'guest' => 'Angelov\Eestec\Platform\Http\Middleware\RedirectIfAuthenticated',

    'boardMember' => 'Angelov\Eestec\Platform\Http\Middleware\BoardMembersOnlyMiddleware',
    'ajax' => 'Angelov\Eestec\Platform\Http\Middleware\AjaxOnlyMiddleware',
];
{% endhighlight %}

To filter an route in the "routes.php" file, we could set an "before" or "after" parameter with the name of the filter. Now, we don't need to specify when the middleware will be used and just add it as "middleware" parameter. For example:

{% highlight php startinline %}
Route::get('/',
    ['as' => 'auth',
     'uses' => 'AuthController@index',
     'middleware' => 'guest']
);
{% endhighlight %}	

I was combining multiple filters on the routes by concatenating them and adding a "\|" separator. This doesnt work with the middlewares and in order to have multiple middlewares we should put them in an array.

{% highlight php startinline %}
Route::group(['prefix' => 'fees', 'middleware' => ['auth', 'boardMember']], function () {
    // ...
}
{% endhighlight %}  

Environment Configuration
---

Laravel gives us an option to have different configuration files for different environments. From this version, it ships with an ".env.example" file which we can use to tell the framework on which environment we are running the application. So, go get that new file. You can read more about the environment detection and the ".env" files in [this article](http://mattstauffer.co/blog/laravel-5.0-environment-detection-and-environment-variables).

TokenMismatchException
---

I managed to run my application in the browser, but when I firstly submitted a form i got an error for unhandled "TokenMismatchException" exception. That happens because in the "Http\Kernel" class, we're inserting "Illuminate\Foundation\Http\Middleware\VerifyCsrfToken" to the application's HTTP middleware stack. Because of that, every form has to include a CSRF token.

If you want to exclude this middleware, just remove it from the array. Otherwise, you should edit your views and add the missing field to the forms.

{% highlight html %}
<!-- remove the backslashes from the value field -->
<input type="hidden" name="_token" value="{\{ csrf_token() }\}">
{% endhighlight %}

If you're sending requests via ajax, you should also include the CSRF tokens there. For example:

{% highlight javascript %}
$(document).on('click', '.btn-delete-meeting', function() {
    // ...
    var token = $("#csrf-token").val();

    $.ajax({
        // ...
        data: {
            '_token': token
        },
        success: function(data){
            // ...
        }
    });

    return false;
});
{% endhighlight %}

Pagination
---

The pagination mechanism is also changed in Laravel 5. Previously, to use the paginator we could inject the "Illuminate\Pagination\Factory" class and use the `Factory::make(...)` method to create it, but this class no longer exists in the framework's core. Currently, there's no documentation on how to use the new pagination so what I did as a temporary solution is to replace the "Illuminate\Pagination\Factory" class with a new one. This is what I have now:

{% highlight php startinline %}
namespace Angelov\Eestec\Platform\Paginator;

use Illuminate\Pagination\LengthAwarePaginator;
use Illuminate\Pagination\Paginator;

class Factory
{
    public static function make(array $items, $totalItems, $itemsPerPage) 
    {
        $page = Paginator::resolveCurrentPage();

        return new LengthAwarePaginator($items, $totalItems, $itemsPerPage, $page, [
            'path' => Paginator::resolveCurrentPath()
        ]);
    }
}
{% endhighlight %}

Routing
---

By now, the only way to configure the application's endpoints was to list them in the "routes.php" file. From this version, there are multiple ways to achieve the same result. For example, we can use annotations in the controllers' methods to specify on which endpoind the method should be called.

My opinion is that the "routes.php" file is the best place for listing the routes, as I like to have them in one place and quickly take a first look on how the application handles my requests. We can consider the file as some kind of a Table of Contents.

In this file, we no longer have to use the "Route" facade and instead we can use something like this:

{% highlight php startinline %}
use Illuminate\Routing\Router;

/** @var Router $router */

$router->group(['prefix' => 'members'], function (Router $router) {

    $router->group(['middleware' => 'guest'], function(Router $router) {
        $router->get('/register',
            ['as' => 'members.register',
             'uses' => 'MembersController@register']
        );
        $router->post('/register',
            ['as' => 'members.postRegister',
             'uses' => 'MembersController@postRegister']
        );
    });

    // ...
}

{% endhighlight %}

With this, we also get rid of the annoying "Class not found" warnings in our IDE.

DatabaseSeeder
---

In order to make our database seeders work, we should fix the class in "database/seeds/DatabaseSeeder.php". The default content of the file in 4.2 was:

{% highlight php startinline %}
class DatabaseSeeder extends Seeder {

    /**
     * Run the database seeds.
     *
     * @return void
     */
    public function run()
    {
        Eloquent::unguard();

        // calling the seeders
        // $this->call('UserTableSeeder');
    }

}
{% endhighlight %}

The system can't find the Eloquent class anymore so we should change the `Eloquent::unguard()` part to

{% highlight php startinline %}
\Illuminate\Database\Eloquent\Model::unguard();
{% endhighlight %}

Unit tests
---

If you have unit tests for your application and you use the Laravel's "TestCase" class you should also make some changes there.
As you know, we already changed the way the application is created, so we need to apply the changes here, and instead of

{% highlight php startinline %}
/**
 * Creates the application.
 *
 * @return \Symfony\Component\HttpKernel\HttpKernelInterface
 */
public function createApplication()
{
    $unitTesting = true;

    $testEnvironment = 'testing';

    return require __DIR__.'/../../bootstrap/start.php';
}
{% endhighlight %}

we are creating the application with

{% highlight php startinline %}
/**
 * Creates the application.
 *
 * @return \Illuminate\Foundation\Application
 */
public function createApplication()
{
    $app = require __DIR__.'/../bootstrap/app.php';

    $app->make('Illuminate\Contracts\Console\Kernel')->bootstrap();

    return $app;
}
{% endhighlight %}

Because the tests are also moved, we need to change the properties in the "phpunit.xml" file too.
We need to change the "testsuites" directive from 

{% highlight xml %}
<testsuites>
    <testsuite name="Application Test Suite">
        <directory>./app/tests/</directory>
    </testsuite>
</testsuites>
{% endhighlight %}

to

{% highlight xml %}
<testsuites>
    <testsuite name="Application Test Suite">
        <directory>./tests/</directory>
    </testsuite>
</testsuites>
{% endhighlight %}

and after it, we also need to add

{% highlight xml %}
<php>
    <env name="APP_ENV" value="testing"/>
</php>
{% endhighlight %} 

You should also check your test to see if you need to make some modifications there.

Its working!
---

Finally, I came to a point where everything in the application works as it worked with Laravel 4.2. However, there is more awesomness that comes with the new version and I haven't made use of it yet. But first, lets clean the project from some unnecessary files. As we don't need them anymore, we can delete "app/filters.php" and "bootstrap/paths.php". Since I now use middlewares, I also deleted the classes in the "Filter" namespace. 

Method Injection
---

One of the things I really liked when I once had to develop something in Silex was that the dependencies can be autoresolved not only when they're injected in the constructorsm but in the methods as well. Having the autoresolving working only for the constructors we could end up injecting many things for all the methods when we needed them only for a couple of them. The alternatives were to use `App::make(...)` or to use the facades, but I didn't liked that.

Here's an example of what I've changed:

{% highlight php startinline %}
/**
 * Display the specified member.
 *
 * @param  int $id
 * @return Response
 */
public function show($id)
{
    $member = $this->members->get($id);

    /** @var MembershipService $membershipService */
    $membershipService = App::make('MembershipService');

    /** @var MeetingsService $meetingsService */
    $meetingsService = App::make('MeetingsService');

    $attendance = $meetingsService->calculateAttendanceDetailsForMember($member);
    $joinedDate = $membershipService->getJoinedDate($member);

    //...   
}
{% endhighlight %}

is now 

{% highlight php startinline %}
/**
 * Display the specified member.
 *
 * @param MembershipService $membershipService
 * @param MeetingsService $meetingsService
 * @param int $id
 * @return Response
 */
public function show(MembershipService $membershipService, MeetingsService $meetingsService, $id)
{
    $member = $this->members->get($id);

    $attendance = $meetingsService->calculateAttendanceDetailsForMember($member);
    $joinedDate = $membershipService->getJoinedDate($member);

    // ...
}
{% endhighlight %}

Form requests and validation
---

To validate the data sent using the forms in the views, I was using a few different validators. To check if the data is valid, in many methods in the controllers I had something like this

{% highlight php startinline %}
public function store()
{
    if (!$this->validator->validate($this->request->all())) {
        $errorMessages = $this->validator->getMessages();
        $this->session->flash('errorMessages', $errorMessages);

        return $this->redirector->back()->withInput();
    }

    //...
}
{% endhighlight %}

Using the new FormRequests feature, I got rid of this duplicated code by creating different classes for the requests that needed validation.

I created an abstract class for all the Requests that extends the "Illuminate\Foundation\Http\FormRequest" class and that has a basic (usually) duplicated behavior for the invalid requests.

{% highlight php startinline %}
abstract class Request extends FormRequest
{
    //...
    protected $rules = [];

    public function rules()
    {
        return $this->rules;
    }

    public function response(array $errors)
    {
        $messages = $this->parseErrors($errors);

        $this->session->flash('errorMessages', $messages);
        return $this->redirector->back()->withInput();
    }

    //...
}
{% endhighlight %}

Later, the other custom request classes extends this class and can specify their own validation rules or other behavior if needed.

{% highlight php startinline %}
class LoginFormRequest extends Request
{
    protected $rules = [
        'email' => 'required|email',
        'password' => 'required|min:6'
    ];

    public function response(array $errors)
    {
        $this->session->flash('auth-error', 'Please insert valid information.');
        return $this->redirector->back()->withInput();
    }
}
{% endhighlight %}

In the base controller class, we should specify that we use the "Illuminate\Foundation\Validation\ValidatesRequests" trait.

{% highlight php startinline %}
class BaseController extends Controller
{
    use ValidatesRequests;
    //..
}
{% endhighlight %}

And now when I want to use a request, I inject an object of the needed request:

{% highlight php startinline %}
class MembersController extends BaseController
{
    /**
     * Update the specified member in storage.
     *
     * @param MembersPopulator $populator
     * @param UpdateMemberRequest $request
     * @param  int $id
     * @return Response
     */
    public function update(MembersPopulator $populator, UpdateMemberRequest $request, $id)
    {
        //...
    }

    //...
}
{% endhighlight %}

Conclusion
---

It was fun making all these changes to my application. I really like how the new version of Laravel works but according to my plans, I hope that the first version of "EESTEC Platform for Local Committees" will hit stable before the end of this year so I guess what I have to do now is `git revert`. If you decide not to leave what you've done here, you should be very careful and keep up with the framework's changes that happen every day.

If you notice something wrong or have a queston or something, feel free to get in touch with me on [Twitter](http://twitter.com/angelovdejan).