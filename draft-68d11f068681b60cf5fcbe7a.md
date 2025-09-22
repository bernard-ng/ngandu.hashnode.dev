---
title: "How to Change Algorithms in Symfony without Code Modifications: The Strategy Pattern"
slug: how-to-change-algorithms-in-symfony-without-code-modifications-the-strategy-pattern
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1758535611552/cd340803-decb-4846-ad47-2b92bd8e6585.jpeg

---

Ever been in a situation where you needed to switch between different behaviors in your application without writing a bunch of ugly `if/else` statements? Imagine a system like Google Maps. It offers a single interface—finding a route—but the actual calculation changes based on your mode of transport: car, bike, or on foot. That's the **Strategy design pattern** in action, and in this post, I'll show you how we can implement it beautifully in Symfony.

The [**Strategy design pattern**](https://en.wikipedia.org/wiki/Strategy_pattern) is a powerful behavioral pattern that allows you to define a family of algorithms, encapsulate each one, and make them interchangeable. This means you can change the algorithm used at runtime without altering the client code that uses it.

## Dependency Injection is Your Friend

The core of this pattern in a Symfony application is the **Dependency Injection (DI) container**. It's the unsung hero that gives us the power to manage services and, more importantly, to inject the correct implementation of a service when we need it.

Let's start with a simple example: publishing an article. We want to publish a post, but the format might change—maybe it's a JSON feed, an RSS feed, or just a simple HTML view.

First, we define a common interface that all our "publishers" will implement. It's the contract that guarantees a unified behavior.

```php
<?php

namespace App\Service\Publisher;

use App\Entity\Post;

interface PublisherInterface
{
    public function publish(Post $post): void;
}
```

Now, we create our concrete implementations, like `RssPublisher` and `JsonPublisher`. Each of them will contain the specific logic for its respective format.

```php
<?php

namespace App\Service\Publisher;

final readonly class JSONPublisher implements PublisherInterface
{

    public function __construct(
        private LoggerInterface $logger,
    ) {
    }

    #[\Override]
    public function publish(Post $post): void
    {
        $fqcn = $this::class;
        $this->logger->critical("Strategy {$fqcn}");
    }

    #[\Override]
    public function supports(string $channel): bool
    {
        return $channel === "json";
    }
}
```

```php
<?php

namespace App\Service\Publisher;

final readonly class RSSPublisher implements PublisherInterface
{

    public function __construct(
        private LoggerInterface $logger,
    ) {
    }

    #[\Override]
    public function publish(Post $post): void
    {
        $fqcn = $this::class;
        $this->logger->critical("Strategy {$fqcn}");
    }

    #[\Override]
    public function supports(string $channel): bool
    {
        return $channel === "rss";
    }
}
```

### Method 1: The Auto-wire Iterator Approach

Our initial challenge is that when we try to inject `PublisherInterface` into our controller, Symfony gets confused. It sees two different implementations (`RssPublisher` and `JsonPublisher`) and doesn't know which one to pick.

The solution? We can tell Symfony to collect all services that implement our interface and inject them as an **iterable**. This allows us to loop through all available publishers and select the correct one at runtime.

To do this, we add a `#[AutoconfigureTag]` attribute to our interface. This ensures that any class that implements this interface will be "tagged" by the container. We also add a `supports()` method to our `PublisherInterface` so each implementation can tell us which "channel" it supports.

```php
<?php

namespace App\Service\Publisher;

use App\Entity\Post;

#[AutoconfigureTag]
interface PublisherInterface
{
    public function publish(Post $post): void;

    public function supports(string $channel): bool;
}
```

Then, we create a central `Publisher` service that collects these tagged services using the `#[AutoWireIterator]` attribute.

```php
<?php

// In our central Publisher service
use App\Service\Publisher\PublisherInterface;
use Symfony\Component\DependencyInjection\Attribute\AutowireIterator;

final readonly class Publisher
{
    /** @param iterable<PublisherInterface> */
    public function __construct(
        #[AutowireIterator(PublisherInterface::class)]
        private iterable $publishers,
    ) {
    }

    public function publish(Post $post, string $channel): void
    {
        foreach ($this->publishers as $publisher) {
            if ($publisher->supports($channel)) {
                $publisher->publish($post);
                return;
            }
        }
    }
}
```

Now, our controller only needs to inject this single `Publisher` service. The controller doesn't need to know anything about the individual publishers; it just calls the `publish` method, passing the post and the desired channel. This is the essence of the Strategy pattern: **dynamic behavior without changing the client code.**

```php
<?php

namespace App\Controller;

#[Route('/post')]
final class PostController extends AbstractController
{
    public function __construct(
        private readonly ClockInterface $clock,
        private readonly Publisher $publisher,    // hummmm weird, we should use interface instead
    ) {
    }

    #[Route('/{id}/{channel}', name: 'app_post_show', requirements: [
        "channel" => "html|json|rss"
    ], methods: ['GET'])]
    public function show(Post $post, string $channel = "html"): Response
    {
        $this->publisher->publish($post, $channel);
        return $this->render('post/show.html.twig', [
            'post' => $post,
        ]);
    }
}
```

We still have an issue, at the moment we are injecting a concrete class that doesn’t implements our `PublisherInterface` explicitly, even if the signature is correct we should be able to keep a clean and predictable contract. to do so we will implements the `PublisherInterface` in our `Publisher` class

```php
<?php

use App\Service\Publisher\PublisherInterface;
use Symfony\Component\DependencyInjection\Attribute\AutowireIterator;

final readonly class Publisher implements PublisherInterface
{
    public function __construct(
        #[AutowireIterator(PublisherInterface::class, excludeSelf: true)]
        private iterable $publishers,
    ) {
    }

    #[\Override]
    public function publish(Post $post, string $channel): void
    {
        foreach ($this->publishers as $publisher) {
            if ($publisher->supports($channel)) {
                $publisher->publish($post);
                return;
            }
        }
    }

    #[\Override]
    public function supports(string $channel): bool
    {
        return true;
    }
}
```

Don’t worry symfony wont inject our publisher even it’s tagged as PublisherInterface, by the default the `AutowireIterator` exclude the class in which is used on injection

### Method 2: The Optimized Auto-wire Locator

While this will work correctly having this support method on the instance can be problematic : we have to instanciate the publisher in order know if it supports a specific channel, think about it a moment is this comparison even necessary at the publisher level ? we could simply return the supported channel and make the method `static` also this behaviour is only need when selecting a strategy, applying the [interface segregation principle](https://en.wikipedia.org/wiki/Interface_segregation_principle) we can split our current contract in two distinct interfaces `StrategyPublisherInterface` and `PublisherInterface`

```php
<?php

namespace App\Service\Publisher;

use Symfony\Component\DependencyInjection\Attribute\AutoconfigureTag;

#[AutoconfigureTag]
interface StrategyPublisherInterface extends PublisherInterface
{
    public static function supports(): string;
}
```

our `RssPublisher` and `JsonPublisher` should implement this new interface and we add the channel to publish method as parameter to simply strategy selection even if this wont be used as part of the publish method itself (let’s be pragmatic :))

```php
<?php

namespace App\Service\Publisher;

use App\Entity\Post;

interface PublisherInterface
{
    public function publish(Post $post, string $channel = "html"): void;
}
```

We can now update our `Publiser` to be the default implementation of the `PublisherInterface` using `#[AsAlias]` and inject only `StrategyPublisherInterface` with `#[AutowireLocator]`, the main difference here is our strategies are index and can be directly accessed using the key, in our case the $channel in term of complexity this is 0(1) where with the previous implementation was 0(n). plus we don’t have to instantiate the strategy every time only when it needed (selected).

```php
<?php

namespace App\Service\Publisher;

use App\Entity\Post;
use Symfony\Component\DependencyInjection\Attribute\AsAlias;
use Symfony\Component\DependencyInjection\Attribute\AutowireLocator;
use Symfony\Component\DependencyInjection\ServiceLocator;

#[AsAlias(PublisherInterface::class)]
final readonly class Publisher implements PublisherInterface
{

    public function __construct(
        #[AutowireLocator(StrategyPublisherInterface::class, defaultIndexMethod: "supports")]
        private ServiceLocator $publishers
    ) {
    }

    public function publish(Post $post, string $channel = "html"): void
    {
        if ($this->publishers->has($channel)) {
            $publisher = $this->publishers->get($channel);
            $publisher->publish($post, $channel);
        }
    }
}
```

This is so much cleaner! The framework now handles the mapping for us and usage in our controller is much simple and maintenable

```php
<?php

namespace App\Controller;

#[Route('/post')]
final class PostController extends AbstractController
{
    public function __construct(
        private readonly ClockInterface $clock,
        private readonly PublisherInterface $publisher,
    ) {
    }

    #[Route('/{id}/{channel}', name: 'app_post_show', requirements: [
        "channel" => "html|json|rss"
    ], methods: ['GET'])]
    public function show(Post $post, string $channel = "html"): Response
    {
        $this->publisher->publish($post, $channel);
        return $this->render('post/show.html.twig', [
            'post' => $post,
        ]);
    }
}
```

### Wrapping Up

Whether you use the iterator or the locator, the **Strategy pattern** is a fantastic way to write code that's both flexible and easy to maintain. It prevents bloated conditional logic and makes your application's behavior transparent and interchangeable.