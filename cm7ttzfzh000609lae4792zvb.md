---
title: "Streamline Symfony error tracking with GlitchTip"
seoTitle: "Streamline Symfony error tracking with GlitchTip"
seoDescription: "Use GlitchTip for Symfony app monitoring: gain insights, troubleshoot, and simplify error tracking for high performance"
datePublished: Tue Mar 04 2025 01:49:28 GMT+0000 (Coordinated Universal Time)
cuid: cm7ttzfzh000609lae4792zvb
slug: streamline-symfony-error-tracking-with-glitchtip
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1740078193297/022aeacc-7611-470a-a260-53183122ae7c.jpeg
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1741052870364/b096f1d3-af2e-4441-96b1-0be3537f4c43.jpeg
tags: monitoring, symfony, observability, sentry, glitchtip

---

In today's fast-paced development environment, ensuring that your Symfony application is running smoothly in production is crucial. One way to achieve this is through effective error monitoring and observability. But what exactly does observability mean, and why is it so important in production? Let's dive in.

## What is Monitoring and Observability ?

**Monitoring** refers to the processes and tools used to collect and analyze performance metrics and application logs. It helps developers and operations teams detect and address issues in their systems. Monitoring focuses on predefined metrics and alerts that signal when something goes wrong.

**Observability**, on the other hand, takes a broader approach. It's a measure of how well you can understand what's happening inside your application based on the data it generates. Observability encompasses monitoring but also includes tools and practices to help you:

* Gain insight into application behavior.
    
* Diagnose and troubleshoot complex issues.
    
* Understand the root cause of unexpected behavior.
    

With observability, you’re not just reacting to problems but proactively gaining visibility into your system’s inner workings.

In production environments, even minor issues can snowball into significant problems, such as application downtime, performance degradation, or user dissatisfaction. Observability plays a critical role in:

1. **Early Issue Detection:** Catching bugs, errors, or anomalies before they impact users.
    
2. **Performance Optimization:** Identifying bottlenecks and improving the overall user experience.
    
3. **Root Cause Analysis:** Quickly pinpointing the source of issues to minimize downtime.
    
4. **Continuous Improvement:** Using insights from monitoring data to make informed decisions about your application.
    

## GlitchTip Integration

[GlitchTip](https://glitchtip.com/) is an open-source alternative to Sentry that provides error monitoring and performance tracking. It is even compatible with the Sentry SDK, making it a great choice for Symfony developers who want a self-hosted monitoring solution.

### Self hosting GlitchTip

For those who prefer full control over their monitoring infrastructure, self-hosting GlitchTip is a viable option. Below is a basic `docker-compose` configuration that sets up GlitchTip with PostgreSQL and Redis. This setup includes:

* A PostgreSQL database for storing events.
    
* Redis for caching and job queue management.
    
* GlitchTip’s web service, worker, and migration jobs.
    

```yaml
x-environment: &default-environment
  DATABASE_URL: postgres://postgres:postgres@postgres:5432/postgres
  SECRET_KEY: "you random secret key"
  PORT: 8000
  EMAIL_URL: "your smpt server url"
  GLITCHTIP_DOMAIN: https://glitchtip.yourdomain.com
  DEFAULT_FROM_EMAIL: contact@yourdomain.ocm
  CELERY_WORKER_AUTOSCALE: "1,3"
  ENABLE_USER_REGISTRATION: 'false'
  ENABLE_ORGANIZATION_CREATION: 'true'

x-depends_on: &default-depends_on
  - postgres
  - redis

services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_HOST_AUTH_METHOD: "trust" # Consider removing this and setting a password
    restart: unless-stopped
    volumes:
      - pg-data:/var/lib/postgresql/data
  redis:
    image: valkey/valkey
    restart: unless-stopped
  web:
    image: glitchtip/glitchtip
    depends_on: *default-depends_on
    ports:
      - "127.0.0.1:8000:8000"
    environment: *default-environment
    restart: unless-stopped
    volumes:
      - uploads:/code/uploads
  worker:
    image: glitchtip/glitchtip
    command: ./bin/run-celery-with-beat.sh
    depends_on: *default-depends_on
    environment: *default-environment
    restart: unless-stopped
    volumes:
      - uploads:/code/uploads
  migrate:
    image: glitchtip/glitchtip
    depends_on: *default-depends_on
    command: ./bin/run-migrate.sh
    environment: *default-environment

volumes:
  pg-data:
  uploads:
```

This is a basic setup that can be enhanced with additional security measures, persistent storage configurations, and scaling options if needed. By self-hosting, you maintain full control over your monitoring data and can tailor the setup to your specific requirements.

### Connecting your Symfony project to GlitchTip

First, install the `sentry/sentry-symfony` package:

```bash
composer require sentry/sentry-symfony
```

Set the environment variable `SENTRY_DSN` to your GlitchTip DSN. Alternatively, configure it directly in `config/packages/sentry.yaml`:

```yaml
when@prod:
    sentry:
        dsn: "%env(SENTRY_DSN)%"
        register_error_listener: false
        register_error_handler: false
        options:
            traces_sample_rate: 0.1
            ignore_exceptions:
                - 'Symfony\Component\ErrorHandler\Error\FatalError'
                - 'Symfony\Component\Debug\Exception\FatalErrorException'
    monolog:
        handlers:
            sentry_fingers_crossed:
                type: fingers_crossed
                action_level: error
                handler: sentry
                excluded_http_codes: [404, 405]
                buffer_size: 50
            sentry:
                type: sentry
                level: !php/const Monolog\Logger::ERROR
                hub_id: Sentry\State\HubInterface
                fill_extra_context: true
                process_psr_3_messages: false
```

This configuration sets up GlitchTip to capture errors in a production Symfony application, but with careful filtering to reduce noise and overhead. It only sends errors and above to sentry, and uses a fingers crossed handler to buffer errors until an actual error occurs. It also ignores 404 and 405 errors, and certain fatal errors. Tracing is enabled at a 10% sample rate.

### Logging an Error

Here’s a simple example of logging an error and an unhandled exception in a Symfony controller:

```php
class HomeController extends AbstractController
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {
    }

    #[Route("/", name: "home", methods: ["GET"])]
    public function home(): Response
    {
        // will be available on glitchtip
        $this->logger->error('My custom logged error.');
        
        if (time() % 2 == 0) {
            // will be available on glitchtip
            throw new \RuntimeException('Example exception.');
        }

        return new JsonResponse([], status: 200);
    }
}
```

## Reviewing Issues in GlitchTip

Once everything is set up, you can log into your GlitchTip instance and start reviewing issues. The interface provides several useful details, such as:

* The environment where the error occurred (e.g., `prod` or `dev`).
    
* The exact release version (`dev-main@69e2435`).
    
* The route where the error was triggered (`content_management_content_show`).
    
* The server name and OS details.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1741051766411/59eb86a9-1877-4334-8e7c-5d3e410446fd.png align="center")

### Analyzing Errors and Stack Traces

GlitchTip allows you to view the full backtrace of an error, including the specific line in the code where the exception was triggered. If the error is not handled, you can trace it back to the originating file and line number.

Additionally, you can add extra context when logging messages to help debug issues more efficiently.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1741052181415/0a31df08-f63d-4a9f-9939-37d3bdeff9cf.png align="center")

## Conclusion

Error monitoring and observability are essential for maintaining high-quality Symfony applications in production. GlitchTip offers a powerful yet straightforward solution to keep your application running smoothly. By integrating GlitchTip, you can gain real-time insights, quickly resolve issues, and provide a better experience for your users.

Happy coding !