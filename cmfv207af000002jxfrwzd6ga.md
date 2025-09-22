---
title: "How to Change Algorithms in Symfony without Code Modifications: The Strategy Pattern"
seoTitle: "Symfony: Using Strategy Pattern Guide"
seoDescription: "Use the Strategy design pattern in Symfony for flexible behavior switching, enhancing maintainability and scalability without altering client code"
datePublished: Mon Sep 22 2025 11:39:25 GMT+0000 (Coordinated Universal Time)
cuid: cmfv207af000002jxfrwzd6ga
slug: symfony-strategy-design-pattern
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1758535611552/cd340803-decb-4846-ad47-2b92bd8e6585.jpeg
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1758541096159/42bc79fc-617c-4ad2-8ba0-ec1f9dde0754.jpeg
tags: design-patterns, php, software-architecture, symfony

---

Ever been in a situation where you had to switch between multiple behaviors in your application, only to end up with endless `if/else` or `switch` blocks? Not only does it look ugly, but it also becomes harder to maintain as the application grows.

Think about a system like **Google Maps**. To the user, there’s just one interface: *find a route*. But internally, the logic varies depending on whether you’re traveling by car, bike, or foot. That’s the **Strategy design pattern** in action. In this post, I’ll show you how we can apply it elegantly in Symfony to keep our code flexible, extensible, and clean.

## The Strategy Pattern in a Nutshell

The [**Strategy design pattern**](https://en.wikipedia.org/wiki/Strategy_pattern) is a classic behavioral pattern. It allows you to:

* Define a family of algorithms.
    
* Encapsulate each one separately.
    
* Make them interchangeable at runtime.
    

The client code doesn’t need to change when the algorithm changes. It just uses the interface. This separation of concerns is what makes the Strategy pattern so powerful.

## Dependency Injection is Your Friend

In Symfony, the backbone of this pattern is the **Dependency Injection (DI) container**. While DI is often thought of as just a way to manage services, its real strength lies in how it lets you dynamically inject the right implementation depending on the context.

Let’s look at a practical example: **publishing an article**. We want to publish posts, but the output format may vary—JSON feed, RSS feed, or plain HTML. Instead of hardcoding multiple conditions, we’ll use strategies.

### Step 1: Define the Contract

We start with a common interface. This contract guarantees consistent behavior across all publishers.

```php
interface PublisherInterface
{
    public function publish(Post $post): void;
}
```

### Step 2: Create Concrete Implementations

Now, we create our strategy classes: `JsonPublisher` and `RssPublisher`. Each implements the same interface, but handles the logic for its own format.

```php
final readonly class JSONPublisher implements PublisherInterface
{
    public function __construct(private LoggerInterface $logger) {}

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
final readonly class RSSPublisher implements PublisherInterface
{
    public function __construct(private LoggerInterface $logger) {}

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

Each class is independent, self-contained, and replaceable.

### Method 1: The Auto-wire Iterator Approach

Here’s the first challenge: when Symfony tries to inject `PublisherInterface`, it doesn’t know which implementation to use—`JsonPublisher` or `RssPublisher`.

The fix? We tell Symfony to **collect all services implementing our interface** and inject them as an **iterable**. This way, we can loop through them and decide at runtime.

First, we tag the interface so Symfony knows to collect its implementations:

```php
#[AutoconfigureTag]
interface PublisherInterface
{
    public function publish(Post $post): void;
    public function supports(string $channel): bool;
}
```

Then, we create a central `Publisher` service that receives them all:

```php
final readonly class Publisher
{
    /** @param iterable<PublisherInterface> */
    public function __construct(
        #[AutowireIterator(PublisherInterface::class)]
        private iterable $publishers,
    ) {}

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

Now, the controller only needs to inject the `Publisher` service. It doesn’t care about the individual publishers—it just delegates. That’s the Strategy pattern in its purest form: **behavior switching without touching the client code.**

```php
final class PostController extends AbstractController
{
    public function __construct(
        private readonly ClockInterface $clock,
        private readonly Publisher $publisher, // Ideally, should use interface instead
    ) {}

    #[Route('/{id}/{channel}', name: 'app_post_show', requirements: [
        "channel" => "html|json|rss"
    ], methods: ['GET'])]
    public function show(Post $post, string $channel = "html"): Response
    {
        $this->publisher->publish($post, $channel);
        return $this->render('post/show.html.twig', ['post' => $post]);
    }
}
```

One refinement: since `Publisher` acts like a strategy itself, we should have it implement `PublisherInterface` directly. That way, we keep everything consistent and predictable.

```php
final readonly class Publisher implements PublisherInterface
{
    public function __construct(
        #[AutowireIterator(PublisherInterface::class, excludeSelf: true)]
        private iterable $publishers,
    ) {}

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

Symfony is smart enough not to inject the `Publisher` itself even if it’s tagged—thanks to the `excludeSelf` flag , which is `true` be default.

### Method 2: The Optimized Auto-wire Locator

The iterator method works fine, but there’s one inefficiency: we must **instantiate all publishers just to check support**. That’s unnecessary overhead.

Instead, we can flip the design. Instead of each strategy telling us whether it supports a channel, each one can simply *declare* its channel key upfront.

This aligns with the [Interface Segregation Principle](https://en.wikipedia.org/wiki/Interface_segregation_principle): we split our contract into two:

```php
#[AutoconfigureTag]
interface StrategyPublisherInterface extends PublisherInterface
{
    public static function supports(): string;
}
```

Our publishers now implement `StrategyPublisherInterface`, with `supports()` returning their channel:

```php
interface PublisherInterface
{
    public function publish(Post $post, string $channel = "html"): void;
}
```

Then we wire up the central `Publisher` using Symfony’s `ServiceLocator`. This gives us [**O(1)**](https://en.wikipedia.org/wiki/Big_O_notation) **lookups**:

```php
#[AsAlias(PublisherInterface::class)]
final readonly class Publisher implements PublisherInterface
{
    public function __construct(
        #[AutowireLocator(StrategyPublisherInterface::class, defaultIndexMethod: "supports")]
        private ServiceLocator $publishers
    ) {}

    public function publish(Post $post, string $channel = "html"): void
    {
        if ($this->publishers->has($channel)) {
            $this->publishers->get($channel)->publish($post, $channel);
        }
    }
}
```

This way, Symfony handles the mapping automatically. We don’t instantiate every strategy—only the one we need. Cleaner, faster, and easier to maintain.

Controller usage becomes even simpler:

```php
final class PostController extends AbstractController
{
    public function __construct(
        private readonly ClockInterface $clock,
        private readonly PublisherInterface $publisher,
    ) {}

    #[Route('/{id}/{channel}', name: 'app_post_show', requirements: [
        "channel" => "html|json|rss"
    ], methods: ['GET'])]
    public function show(Post $post, string $channel = "html"): Response
    {
        $this->publisher->publish($post, $channel);
        return $this->render('post/show.html.twig', ['post' => $post]);
    }
}
```

## Wrapping Up

Both approaches—iterator and locator—faithfully implement the **Strategy pattern** in Symfony.

* The **iterator method** is straightforward and intuitive, great for smaller sets of strategies.
    
* The **locator method** is more efficient and scales better, avoiding unnecessary instantiations.
    

Either way, you eliminate bloated conditionals, keep your codebase clean, and make it effortless to extend behavior in the future. That’s the real strength of the Strategy pattern: **flexibility without compromise.**