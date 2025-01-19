---
title: "Decoupling your application's User Model from Symfony's Security System"
seoTitle: "Decoupling your application's User Model from Symfony's Security Syste"
seoDescription: "This article explores how to decouple your application's user model from Symfony's security system"
datePublished: Sun Sep 15 2024 04:22:11 GMT+0000 (Coordinated Universal Time)
cuid: cm132l1eu000009mrfszxf0py
slug: decoupling-your-applications-user-model-from-symfonys-security-system
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1726367363564/eefdcf6f-20af-4497-a3b1-2a854c018458.jpeg
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1726374026864/066edab8-1f42-4bd0-8915-31f84c7b2969.jpeg
tags: security, symfony

---

## Introduction

When building Symfony applications with advanced architectural patterns like hexagonal architecture, the primary goal is often to decouple the domain (business logic) from the infrastructure (technical choices). This separation allows for a clear focus on the business side of the application without getting overly entangled in specific technical implementations.

In a traditional Symfony setup :

> permissions are always linked to a user object. If you need to secure (parts of) your application, you need to create a user class. This is a class that implements [`UserInterface`](https://github.com/symfony/symfony/blob/7.2/src/Symfony/Component/Security/Core/User/UserInterface.php). Often, this is a [Doctrine](https://www.doctrine-project.org/projects/doctrine-orm/en/3.2/tutorials/getting-started.html#entity-repositories) entity \[4\].

While this approach is straightforward, it introduces tight coupling between the domain and the infrastructure layer—in this case, Symfony itself.

Some experts within the Symfony community have explored various strategies for decoupling, particularly in the context of security. One of them is to create a class dedicated solely to security \[1, 2, 3\].

However the last significant technical exploration on this topic was published six years ago, and Symfony’s security system has since undergone considerable changes. In this article, I want to revisit this problem and provide an updated approach.

## **How to separate them ?**

1. **Create a separate security user class** (`SecurityUser.php`)
    
2. **Create a security user provider** (`SecurityUserProvider.php`)
    
3. **Configure Symfony to use the new security user and provider** in the `config/packages/security.yaml`
    

**SecurityUser :** In this class, we’ll define a user entity strictly for security purposes, decoupled from our domain `User` entity. The class will implement `UserInterface` and possibly other interfaces required by Symfony’s security system.

```php
<?php

declare(strict_types=1);

namespace Infrastructure\Framework\Symfony\Security;

use Domain\Model\User\Entity\User;
use Symfony\Component\Security\Core\User\{
    UserInterface,
    PasswordAuthenticatedUserInterface
};
use Symfony\Component\Uid\Uuid;

final readonly class SecurityUser implements 
    UserInterface, 
    PasswordAuthenticatedUserInterface
{
    private function __construct(
        private Uuid $id,
        private string $email,
        private string $password,
        private array $roles
    ) {
    }

    public static function create(User $user): self
    {
        return new self(
            $user->getId(),
            $user->getEmail(),
            $user->getPassword(),
            $user->getRoles()
        );
    }

    #[\Override]
    public function getPassword(): ?string
    {
        return $this->password;
    }

    #[\Override]
    public function getRoles(): array
    {
        return $this->roles;
    }

    #[\Override]
    public function getUserIdentifier(): string
    {
        return $this->email;
    }

     #[\Override]
    public function eraseCredentials(): void
    {
    }
}
```

**SecurityUserProvider** : The [user provider](https://symfony.com/doc/current/security/user_providers.html#creating-a-custom-user-provider) will be responsible for loading the `SecurityUser` based on an identifier, such as email. It can load the user from a database or any other source, typically using your application's domain model (the actual `User` entity).

```php
<?php

declare(strict_types=1);

namespace Infrastructure\Framework\Symfony\Security;

use Domain\Model\User\Repository\UserRepository;
use Symfony\Component\Security\Core\{
    Exception\UserNotFoundException,
    User\UserInterface,
    User\UserProviderInterface
};

final readonly class SecurityUserProvider implements UserProviderInterface
{
    public function __construct(
        private UserRepository $userRepository
    ) {
    }

    #[\Override]
    public function refreshUser(UserInterface $user): UserInterface
    {
        return $this->loadUserByIdentifier($user->getUserIdentifier());
    }

    #[\Override]
    public function loadUserByIdentifier(string $identifier): UserInterface
    {
        $user = $this->userRepository->getByEmail($email);
        if ($user === null) {
            throw new UserNotFoundException();
        }

        return SecurityUser::create($user);
    }

    #[\Override]
    public function supportsClass(string $class): bool
    {
        return $class === SecurityUser::class;
    }
}
```

Done, Let’s tell Symfony about the user provider by adding it in `config/packages/security.yaml`

```yaml
security:
    # ...
    providers:
        app_user_provider:
            id: Infrastructure\Framework\Symfony\Security\SecurityUserProvider

    firewalls:
        dev:
            pattern: ^/(_(profiler|wdt)|css|images|js)/
            security: false
        main:
            lazy: true
            provider: app_user_provider
            form_login:
                login_path: app_login
                check_path: app_login
                enable_csrf: true
            logout:
                path: app_logout
                target: app_login

    access_control:
        - { path: ^/login$, roles: PUBLIC_ACCESS }
        - { path: ^/, roles: ROLE_USER }
```

## Conclusion

Symfony offers several built-in [authenticators](https://symfony.com/doc/7.2/security.html#authenticating-users), such as `form_login`, which simplifies the implementation of user authentication. However, if your application requires more complex authentication logic, you can create a custom authenticator and leverage the custom `SecurityUserProvider` we’ve implemented to load users during the authentication process.

```php
public function authenticate(Request $request): Passport 
{
    $token = $request->request->get('_csrf_token');
	$email = $request->request->get('email');
	$password = $request->request->get('password');

	$request
		->getSession()
		->set(SecurityRequestAttributes::LAST_USERNAME, $email);

	$passport = new Passport(
		 new UserBadge($email, $this->securityUserProvider->loadUserByIdentifier(...)),
		 new PasswordCredentials($password), 
		 [
		    new CsrfTokenBadge('authenticate', $token),
		    new RememberMeBadge(),
		 ]
	);

	return $passport;
}
```

It’s important to note that when [fetching](https://symfony.com/doc/7.2/security.html#fetching-the-user-object) the currently logged-in user (for example, through the `getUser()` method or using `app.user` in Twig), Symfony will return the `SecurityUser` object we created earlier, not your domain's `User`. This distinction is key: `SecurityUser` serves only the purpose of authentication and authorization, keeping our domain logic decoupled from Symfony’s infrastructure.

If you need additional details from the actual `User` entity—such as user profile information or other domain-specific data—it's recommended to use a view model. Which would act as a bridge, fetching the necessary data from the `User` entity and exposing it in a form that your application can use without violating the separation between the domain and infrastructure layers.

## References

1. [https://simshaun.medium.com/decoupling-your-application-user-from-symfonys-security-user-60fa31b4f7f2](https://simshaun.medium.com/decoupling-your-application-user-from-symfonys-security-user-60fa31b4f7f2)
    
2. [https://matthiasnoback.nl/2022/07/decoupling-your-security-user-from-your-user-model/](https://matthiasnoback.nl/2022/07/decoupling-your-security-user-from-your-user-model/)
    
3. [https://stovepipe.systems/post/decoupling-your-security-user](https://stovepipe.systems/post/decoupling-your-security-user)
    
4. [https://symfony.com/doc/7.2/security.html](https://symfony.com/doc/7.2/security.html)