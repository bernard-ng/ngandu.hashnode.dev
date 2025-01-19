---
title: "Sending GitHub Notifications to Telegram, A Symfony Webhook Guide"
seoTitle: "Sending GitHub Notifications to Telegram, A Symfony Webhook Guide"
seoDescription: "Guide to sending GitHub notifications to Telegram using Symfony webhooks, including setup, coding, and configuration for seamless integration"
datePublished: Mon Jul 01 2024 04:26:33 GMT+0000 (Coordinated Universal Time)
cuid: cly2h9wj5000108mh0mmk6ssw
slug: sending-github-notifications-to-telegram-a-symfony-webhook-guide
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1720113977792/e8421d8d-4231-4aaa-8e1a-139beb54b3e7.jpeg
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1719807943626/52fd0476-6892-46b8-838c-eeccc88277cf.jpeg
tags: github, webhooks, symfony, telegram

---

Me and my team use Telegram to communicate. Not very conventional, yes, I admit it, but Telegram remains the tool we're used to. If you didn't know, it's possible to create a bot on Telegram. As we wanted to receive GitHub notifications directly in Telegram, like when a colleague makes a push on a project, we explored this possibility.

## What is a webhook ?

A webhook is a way for one application to provide real-time information to another. Unlike traditional APIs, where one application has to query the other for data, webhooks send data as soon as a specific event occurs. This is particularly useful for receiving instant notifications of events such as commits or pull requests on GitHub.

#### Setting up everything

To receive GitHub notifications via webhook, we need to set up webhooks on our GitHub repository or organization. Here's [how to do it](https://docs.github.com/en/webhooks/using-webhooks/creating-webhooks), DM [https://telegram.me/BotFather](https://telegram.me/BotFather) to create a Bot and get and API token.

## Let's code !

Let's assume you've already setup your Symfony project. we need to install Symfony's **Webhook** and **RemoteEvent** component as well as Telegram's client API for PHP. (Note that I'm a contributor to this client API).

```bash
composer require symfony/webhook symfony/remote-event telegram-bot/api
```

Now that our libraries are installed, let's move on to configuring our environment variables. These variables are essential for storing our API keys. For added security, I recommend using [Symfony's secret management system](https://symfony.com/doc/current/configuration/secrets.html). This allows us to protect our sensitive information and sleep soundly!

```bash
# .env.local
TELEGRAM_API_TOKEN=xxxxxxxx:xxxxxxxxxxxxxxxxxxx
GITHUB_WEbHOOK_SECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

To simplify the injection of these values, we can configure them directly as parameters in our service configuration so we can easily access these values in any Class, thanks to Autowiring.

By configuring our parameters in this way, Symfony will automatically inject the necessary values where we need them. Here's how to do it:

```yaml
# config/service.yaml
parameters:
    telegram_api_token: '%env(TELEGRAM_API_TOKEN)%'
    github_webhook_secret: '%env(GITHUB_WEbHOOK_SECRET)%'
```

Our API client needs a token to make requests to Telegram. Here's how we can configure the injection of this token once and for all, using the parameter defined above:

```yaml
# config/service.yaml
services:
    TelegramBot\Api\BotApi:
        arguments:
            - '%telegram_api_token%'
```

Now let's create the classes needed to handle webhook requests from GitHub. We'll need two classes:

* **RequestParser**: This class will intercept POST requests in JSON from GitHub and return a **RemoteEvent** object.
    
* **WebhookConsumer**: This class will receive a **RemoteEvent** and handle the corresponding logic.
    

Thanks to symfony's maker component, we don't need to create them manually, we can use the following command to generate them :

```bash
php bin/console make:webhook github
```

`src/RemoteEvent/GithubWebhookConsumer.php` and `src/Webhook/GithubRequestParser.php` will be generated, before going any further, I'll take this opportunity to configure the webhook, noting that this class also acts as a controller and the webhook component creates a `/webhook/{type}` route associated, here `http://localhost:8000/webhook/github`, which is precisely the one you need to specify on Github. You can use [ngrok](https://ngrok.com/) to test on your local environment.

```yaml
# config/packages/webhook.yaml
framework:
    webhook:
        routing:
            github:
                service: App\Webhook\GithubRequestParser
                secret: '%github_webhook_secret%'
```

`github_webhook_secret` ?? Yes I didn't mention it but to be sure that the requests come from Github, we can add a secret that Github will use to generate a `sha1` and `sha256` signature that will allow us to validate the authenticity of the requests.

### GithubRequestParser

this class has three methods,

* `getRequestMatcher` : which ensures that the request we receive corresponds to the one we expect. For example, we can expect all requests from github to be in json format and method POST, and that the host is "[github.com](http://github.com)".
    
* `doParse` : which takes a request and transforms it into an instance of **RemoteEvent**
    
* `validateSignature` **:** exceptional here we validate the signature using our secret key
    

```php
use Symfony\Component\HttpFoundation\HeaderBag;
use Symfony\Component\HttpFoundation\ChainRequestMatcher;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\RequestMatcherInterface;
use Symfony\Component\RemoteEvent\RemoteEvent;
use Symfony\Component\Webhook\Client\AbstractRequestParser;
use Symfony\Component\Webhook\Exception\RejectWebhookException;
use Symfony\Component\HttpFoundation\RequestMatcher\{
    MethodRequestMatcher,
    IsJsonRequestMatcher
};


final class GithubRequestParser extends AbstractRequestParser
{
    protected function getRequestMatcher(): RequestMatcherInterface
    {
        return new ChainRequestMatcher([
            new MethodRequestMatcher(Request::METHOD_POST),
            new IsJsonRequestMatcher()
        ]);
    }

    protected function doParse(
        Request $request, 
        #[\SensitiveParameter] string $secret
    ): ?RemoteEvent {
        $this->validateSignature(
            headers: $request->headers, 
            body: $request->getContent(), 
            secret: $secret
        );
        
        return new RemoteEvent(
            name: $request->headers->get('X-GitHub-Event'),
            id: $request->headers->get('X-GitHub-Hook-ID'),
            payload: $request->getPayload()->all()
        );
    }

    private function validateSignature(
        HeaderBag $headers, string $body, 
        #[\SensitiveParameter] string $secret
    ): void {
        $signature = hash_hmac('sha256', $body, $secret);

        if (!hash_equals($signature, $headers->get('X-Hub-Signature-256'))) {
            throw new RejectWebhookException(406, 'Invalid signature.');
        }
    }
}
```

### GithubWebhookConsumer

Think of this class as a service that encapsulates logic, in our case it's simply checking the type of event that Github sends us and using our telegram client to send the notification back to the group chat, obviously I added the bot to the group, to get the **ChatId** of your group [read this](https://stackoverflow.com/questions/32423837/telegram-bot-how-to-get-a-group-chat-id)

```php
use TelegramBot\Api\BotApi;
use Psr\Log\LoggerInterface;
use Symfony\Component\RemoteEvent\Attribute\AsRemoteEventConsumer;
use Symfony\Component\RemoteEvent\Consumer\ConsumerInterface;
use Symfony\Component\RemoteEvent\RemoteEvent;


#[AsRemoteEventConsumer('github')]
final readonly class GithubWebhookConsumer implements ConsumerInterface
{
    public function __construct(
        private BotApi $api,
        private LoggerInterface $logger
    ) {
    }

    public function consume(RemoteEvent $event): void
    {
        $name = $event->getName();

       try {
           match (true) {
               $name === 'push' => $this->handlePushEvent($event),
               $name === 'ping' => $this->handlePingEvent($event),
               default => null,
           };
       } catch (\Throwable $e) {
              $this->logger->error($e->getMessage());
       }
    }

    private function handlePushEvent(RemoteEvent $event): void
    {
        $data = $event->getPayload();
        $project = $data['repository']['full_name'];
        $pusher = $data['pusher']['name'];
        $description = $data['head_commit']['message'];
        $ref = str_replace('refs/heads/', '', $data['ref']);
        $commit = substr(strval($data['after']), 0, 8);

        $message = vsprintf(
            format: $commit === '00000000' ?
                '🔥 %s deleted %s on %s' :
                '🔥 %s pushed %s on %s : %s',
            values: [$pusher, $ref, $project, $description]
        );

        $this->sendMessage($message);
    }

    private function handlePingEvent(RemoteEvent $event): void
    {
        $data = $event->getPayload();
        $message = sprintf('👉 Github ping : %s', $data['zen']);
        $this->sendMessage($message);
    }

    private function sendMessage(?string $message = null): void
    {
        if ($message !== null) {
            $this->api->sendMessage(
                chatId: 'your group id',
                text: $message,
                disablePreview: true,
                messageThreadId: 'your topic id if any'
            );
        }
    }
}
```

## Conclusion

And now we can receive a notification every time a colleague pushes something on a project. Don't worry, we're a small team, so this won't be a nuisance.

That said, the use of **Webhook** and **RemoteEvent** components has not yet been sufficiently documented. I hope this post will help you to clarify things.