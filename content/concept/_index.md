---
title: "concept"
description: "The concept of Lucid Architecture"
date: 2020-11-30T17:02:27Z
draft: false
---

this architecture did not just drop out of nowhere. It is in fact an amalgamation of best practices, design patterns and proven methods
that we decided to incorporate at the forefront of our code.

Lucid's built-in patterns are **Command Bus** and **Domain Driven Design**, upon which it builds a stack of directories and units
to organize business logic. It also derives from **SOA (Service Oriented Architecture)** the notion of encapsulating functionality
within a "service" and enriches the concept with more than the service being a class.

- **Command Bus**: to dispatch units of work. In Lucid terminology these units will be a `Feature`, `Job` or `Operation`.
- **Domain Driven Design**: to organize the units of work by categorizing them according to the topic they belong to.
- **Service Oriented Architecture**: to encapsulate and manage functionalities of the same purpose with their required resources (routes, controllers, views, datatbase migrations etc.)

## position

in a typical MVC application, Lucid will be the bond between the application's entrypoints and the units that do the work,
securing code form meandring in drastic directions:

![Lucid MVC Position](/images/concept/mvc-position.png)

## the stack

at a glance...

![Lucid Stack](/images/concept/stack.png)

### framework

provides the "kernel" to do the heavy lifting of the tedious stuff such as request/response lifecycle, dependency
injection, and other core functionalities.

### foundation

extends the framework to provide higher level abstractions that are custom to the application and can be shared
across the entire stack rather than being case-specific.

examples of what could go into foundation are:
- `DateTime` a support class for common date and time functions
- `JsonSerializableInterface` that is used to identify an object to be serializable from and to JSON format

### domains

provide separation to categorize jobs and corresponding classes that belong to the same topic. A domain operates in isolation
from other domains and exposes its functionalities to features and operations through Lucid jobs only.

consider the structure below for an example on what a domain may look like:

```
app/Domains/GitHub
├── GitHubClient
├── Jobs
│   ├── FetchGitHubRepoInfoJob
│   └── LoginWithGitHubJob
├── Exceptions
│   ├── InvalidTokenException
│   └── RepositoryNotFoundException
└── Tests
    └── GitHubClientTest
    └── Jobs
        ├── FetchGitHubReposJobTest
        └── LoginWithGitHubJobTest
```

[documentation](https://docs.lucidarch.dev/domains/) contains more details on working with domains.

### services

are directories rich in functionality, used to separate a [Monolith](https://docs.lucidarch.dev/micro-vs-monolith/#monolith) into
areas of focus in a multi-purpose application.

consider the example of an application where we enter food recipes and would want our members to have discussions in a forum,
we would have two services: *1) Kitchen, 2) Forum* where the kitchen would manage all that's related to recipes, and forum is obvious:

```
app/Services
├── Forum
└── Kitchen
```

and following is a single service's structure, highlighted are the Lucid specific directories:

<pre>
app/Services/Forum
├── Console
│   └── Commands
├── <strong>Features</strong>
├── <strong>Operations</strong>
├── Http
│   ├── Controllers
│   └── Middleware
├── Providers
│   ├── KitchenServiceProvider
│   ├── BroadcastServiceProvider
│   └── RouteServiceProvider
├── <strong>Tests</strong>
│   └── <strong>Features</strong>
│   └── <strong>Operations</strong>
├── database
│   ├── factories
│   ├── migrations
│   └── seeds
├── resources
│   ├── lang
│   └── views
└── routes
    ├── api
    ├── channels
    ├── console
    └── web
</pre>

[documentation](https://docs.lucidarch.dev/services/) has more examples of services and their contents.

### features

represent a human-readable application feature in a class. It contains the logic that implements the feature but with the least
amount of detail, by running jobs from domains and operations at the application or service level.

serving the Feature class will be the only line in a controller's method (in MVC), consequently achieving the thinnest form of controllers.

```php
class AddRecipeFeature extends Feature
{
    public function handle(AddRecipe $request)
    {
        $price = $this->run(CalculateRecipePriceOperation::class, [
            'ingredients' => $request->input('ingredients'),
        ]);

        $this->run(SaveRecipeJob::class, [
            'price' => $price,
            'user' => Auth::user(),
            'title' => $request->input('title'),
            'ingredients' => $request->input('ingredients'),
            'instructions' => $request->input('instructions'),
        ]);

        return $this->run(RedirectBackJob::class);
    }
}
```

[documentation](https://docs.lucidarch.dev/features/) about features expands on how to serve them as classes from anywhere.

### operations

their purpose is to increase the degree of code reusability by piecing jobs together to provide composite functionalities from across domains.

```php
class NotifySubscribersOperation extends Operation
{
    private int $authorId;

    public function __construct(int $authorId)
    {
        $this->authorId = $authorId;
    }

    /**
     * Sends notifications to subscribers.
     *
     * @return int Number of notification jobs enqueued.
     */
    public function handle(): int
    {
        $author = $this->run(GetAuthorByIDJob::class, [
            'id' => $this->authorId,
        ]);

        do {

            $result = $this->run(PaginateSubscribersJob::class, [
                'authorId' => $this->authorId,
            ]);

            if ($result->subscribers->isNotEmpty()) {
                // it's a queueable job so it will be enqueued, no waiting time
                $this->run(SendNotificationJob::class, [
                    'from' => $author,
                    'to' => $result->subscribers,
                    'notification' => 'article.published',
                ]);
            }

        } while ($result->hasMorePages());

        return $result->total;
    }
}
```

[documentation](https://docs.lucidarch.dev/operations/) goes over this simple yet powerful concept.

### data

for a scalable set of interconnected data elements, we've created a place for them in `app/Data`,
because most likely over time writing the application there could develop a need for more than Models in data,
such as Repositories, Value Objects, Collections and more.

```
app/Data
├── Models
├── Values
├── Collections
└── Repositories
```

## benefits

there are valuable advantages to what may seem as overengineering.

### <i class="ti-layers" style="font-size: large"></i>&nbsp;organization

- predictable impact of changes on the system when reviewing code
- reduced debugging time since we’re dividing our application into isolated areas of focus (divide and conquer)
- with Monolith, each of our services can have their own versioning system (e.g. Api service is at v1 while Chat is at v2.3 yet reside)
yet reside in the same codebase

### <i class="ti-loop" style="font-size: large"></i>&nbsp;reuse & replace

by dissecting our application into small building blocks of code - a.k.a units - we've instantly opened the door for a high
degree of code sharing across the application with Data and Domains, as well as replaceability with the least amount of friction
and technical debt.

### <i class="ti-layout-slider" style="font-size: large"></i>&nbsp;boundaries

by setting boundaries you would've taken a step towards proetcting application code from growing unbearably large
and made it easier for new devs to onboard. Most importantly, that you've reduced technical debt to the minimum so that you don't
have to pay with bugs and sleepless nights; code doesn't run on good intentions nor wishes.

### <i class="ti-server" style="font-size: large"></i>&nbsp;multitenancy

when our application scales we'd typically have a bunch of instances of it running in different locations,
at some point we would want to activate certain parts of our codebase in some areas and shut off others.

here’s a humble example of running *Api*, *Back Office* and *Web App* instances of the same application, which in Lucid terminology
are *services* that share functionality through *data* and *domains*:

![Lucid multitenancy](/images/multitenancy.jpeg)
