---
title: "Passkey Authentication Guide for Symfony"
seoTitle: "Passkey Authentication Guide for Symfony"
seoDescription: "Implement secure, passwordless login using passkeys in Symfony with WebAuthn. Comprehensive guide with code examples and setup instructions"
datePublished: Sun Sep 22 2024 05:49:40 GMT+0000 (Coordinated Universal Time)
cuid: cm1d5shu7000f09l1eb0ke5y7
slug: passkey-authentication-guide-for-symfony
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1726969763309/9eedb3f5-5fce-4712-adc1-3d873595ef3d.jpeg
tags: authentication, security, php, symfony, webauthn, passkeys

---

## Introduction

Passkeys, also known as [WebAuthn](https://www.w3.org/TR/webauthn-2/) credentials, represent a modern approach to passwordless authentication. As an evolution in web security, they aim to replace traditional password-based logins by utilizing a combination of public-key cryptography and hardware-based authentication. With passkeys, users authenticate themselves using an authenticator (like their phone or security key) rather than relying on passwords, which are often vulnerable to breaches, phishing attacks, and other security risks.

For this blog post, we'll focus on implementing passkeys registration and login using [**Symfony 7.1**](https://symfony.com/releases/7.1) and [**PHP 8.3**](https://www.php.net/releases/8.3/en.php) **with an open source bundle**. Rather than relying on third-party SaaS providers.

Thanks to the work of [Florent Morselli](https://x.com/FlorentMorselli), we have access to a production-ready library broken down into three key components:

1. [**webauthn-lib**](https://github.com/web-auth/webauthn-lib): The core library responsible for handling WebAuthn protocol logic.
    
2. [**webauthn-stimulus-bundle**](https://github.com/web-auth/webauthn-symfony-bundle): interaction between user devices and the server.
    
3. [**webauthn-symfony-bundle**](https://github.com/web-auth/webauthn-symfony-bundle): Integration with the Symfony framework.
    

Now, let’s dive into the code and see how to set everything up in Symfony!

## Setting up a Symfony project

Thanks to Symfony’s [**Maker Bundle**](https://github.com/symfony/maker-bundle), we can easily generate essential features such as user registration and login. Once the foundation is in place, we’ll enhance the project by incorporating passkeys for both **login** and **registration**.

```bash
symfony new passkey-auth --webapp
php bin/console make:user
php bin/console make:security:form-login
php bin/console make:registration-form
```

```bash
docker compose up -d && symfony serve --no-tls
```

### User Entity

For this demo, the default user class generated is sufficient. It already includes the essential fields such as `email`, `password`, and `roles`, which is a good starting point.

```php
<?php

namespace App\Entity;

// importations...

#[ORM\Table(name: '`user`')]
#[ORM\Entity(repositoryClass: UserRepository::class)]
#[ORM\UniqueConstraint(name: 'UNIQ_IDENTIFIER_EMAIL', fields: ['email'])]
#[UniqueEntity(fields: ['email'], message: 'something went wrong !')]
class User implements 
    UserInterface, 
    PasswordAuthenticatedUserInterface
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 180)]
    private ?string $email = null;

    #[ORM\Column]
    private array $roles = [];

    #[ORM\Column(nullable: true)]
    private ?string $password = null;

    // methods...
}
```

Since you’re using PostgreSQL with Docker, let’s make sure to generate the migration files and apply them to your PostgreSQL database.

```bash
symfony console make:migration
symfony console doctrine:migrations:migrate
```

To make entity persistence easier, we’ll add two helper methods to the `UserRepository` class

```php
<?php

namespace App\Repository;

// importations...

class UserRepository extends ServiceEntityRepository implements 
    PasswordUpgraderInterface
{
    // constructor...

    public function save(User $user): void
    {
        $this->getEntityManager()->persist($user);
        $this->getEntityManager()->flush();
    }

    public function remove(User $user): void
    {
        $this->getEntityManager()->remove($user);
        $this->getEntityManager()->flush();
    }

    // interface implementation...
}
```

### Restricted area

Let's create a **MainController** that serves as a restricted area, accessible only to authenticated users.

```bash
symfony console make:controller MainController
```

Symfony offers two ways to enforce access control for specific routes:

1. Using the `IS_GRANTED` attribute directly in the controller.
    
2. Defining access control rules in `config/packages/security.yaml`.
    

```php
<?php

namespace App\Controller;

// importations...

#[IsGranted('IS_AUTHENTICATED_FULLY')]
class MainController extends AbstractController
{
    #[Route('/main', name: 'app_main')]
    public function index(): Response
    {
        return $this->render('main/index.html.twig', [
            'controller_name' => 'MainController',
        ]);
    }
}
```

## Setting up passkeys authentication

it’s time to implement **passkey authentication**. We'll handle this natively using the libraries provided by **Florent**. As mentioned earlier

```bash
composer require web-auth/webauthn-lib
composer require web-auth/webauthn-symfony-bundle 
composer require web-auth/webauthn-stimulus
```

### **The Relying Party**

In the context of **passkey** and **WebAuthn** authentication, the [**Relying Party**](https://webauthn-doc.spomky-labs.com/prerequisites/the-relying-party) (often referred to as "RP") is the application or service that interacts with the user and their **authenticator**. The [authenticator](https://webauthn-doc.spomky-labs.com/webauthn-in-a-nutshell/authenticators) is a device or system that securely stores passkeys and responds to authentication requests (e.g., a smartphone, hardware key, or biometric device).

1. **RP Name**: This is a human-readable name for the application, which the user will recognize when interacting with the authentication process. For example, "My Application"
    
2. [**RP ID**](https://webauthn-doc.spomky-labs.com/prerequisites/the-relying-party#how-to-determine-the-relying-party-id): This is usually the domain name of the application (e.g., [`localhost`](http://localhost), [`myapp.com`](http://myapp.com)). The RP ID is important because it binds the authentication request to the domain, ensuring that credentials registered for one RP cannot be used by another.
    

You can use **environment variables** to configure the values for the **Relying Party (RP)** dynamically

```bash
# .env

###> web-auth/webauthn-symfony-bundle ###
RELYING_PARTY_ID=localhost
RELYING_PARTY_NAME="My Application"
###< web-auth/webauthn-symfony-bundle ###
```

### Credential Source

After a user registers an authenticator, your application receives a **Public Key Credential Source** object. This object stores all the credential information required for authenticating the user in future login attempts.

The **Public Key Credential Source** not only includes the data necessary for authentication but also provides detailed information about the authenticator itself, helping your application manage user credentials more effectively.

```php
<?php

namespace App\Entity;

use Symfony\Component\Uid\Uuid;
use App\Repository\WebauthnCredentialSourceRepository;
use Doctrine\ORM\Mapping as ORM;
use Webauthn\PublicKeyCredentialSource;
use Webauthn\TrustPath\TrustPath;

#[ORM\Table(name: 'webauthn_credentials')]
#[ORM\Entity(repositoryClass: WebauthnCredentialSourceRepository::class)]
class WebauthnCredentialSource extends PublicKeyCredentialSource
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    public function __construct(
        string $publicKeyCredentialId,
        string $type,
        array $transports,
        string $attestationType,
        TrustPath $trustPath,
        Uuid $aaguid,
        string $credentialPublicKey,
        string $userHandle,
        int $counter
    ) {
        parent::__construct(
            $publicKeyCredentialId, $type, $transports, 
            $attestationType,$trustPath, 
            $aaguid, $credentialPublicKey, 
            $userHandle, $counter
        );
    }
}
```

```php
<?php

namespace App\Repository;

use App\Entity\User;
use App\Entity\WebauthnCredentialSource;
use Doctrine\Persistence\ManagerRegistry;
use Webauthn\{
    Bundle\Repository\DoctrineCredentialSourceRepository,
    PublicKeyCredentialSource
};

final class WebauthnCredentialSourceRepository extends DoctrineCredentialSourceRepository
{
    public function __construct(ManagerRegistry $registry)
    {
        parent::__construct($registry, WebauthnCredentialSource::class);
    }

    public function saveCredentialSource(PublicKeyCredentialSource $publicKeyCredentialSource): void
    {
        if (!$publicKeyCredentialSource instanceof WebauthnCredentialSource) {
            $publicKeyCredentialSource = new WebauthnCredentialSource(
                $publicKeyCredentialSource->publicKeyCredentialId,
                $publicKeyCredentialSource->type,
                $publicKeyCredentialSource->transports,
                $publicKeyCredentialSource->attestationType,
                $publicKeyCredentialSource->trustPath,
                $publicKeyCredentialSource->aaguid,
                $publicKeyCredentialSource->credentialPublicKey,
                $publicKeyCredentialSource->userHandle,
                $publicKeyCredentialSource->counter
            );
        }
        parent::saveCredentialSource($publicKeyCredentialSource);
    }
}
```

### Credential User

Represents a user who interacts with your application and their authenticators. It encapsulates essential user data while adhering to specific constraints required by the WebAuthn specification.

To prevent conflicts and ensure that each user can be uniquely identified :

1. Each user must have a unique identifier.
    
2. The username (email in our case) must also be unique.
    

```php
<?php

namespace App\Repository;

use App\Entity\User;
use LogicException;
use Random\RandomException;
use Doctrine\DBAL\Exception;
use Doctrine\DBAL\Connection;
use ParagonIE\ConstantTime\Base64UrlSafe;
use Webauthn\{
    Exception\InvalidDataException,
    Bundle\Repository\CanGenerateUserEntity,
    Bundle\Repository\CanRegisterUserEntity,
    PublicKeyCredentialUserEntity,
    Bundle\Repository\PublicKeyCredentialUserEntityRepositoryInterface
};

final readonly class WebauthnCredentialUserRepository implements
    PublicKeyCredentialUserEntityRepositoryInterface,
    CanRegisterUserEntity,
    CanGenerateUserEntity
{
    public function __construct(
        private UserRepository $userRepository,
        private Connection $connection
    ) {
    }

    /**
     * @see https://dba.stackexchange.com/q/253090
     * @see https://dba.stackexchange.com/a/253098
     * @todo using UUIDs would be a better idea as they are decoupled from the database
     */
    public function generateNextUserEntityId(): string
    {
        return (string) $this->connection
            ->executeQuery('SELECT last_value + 1 FROM user_id_seq;')
            ->fetchOne();
    }

    public function saveUserEntity(PublicKeyCredentialUserEntity $userEntity): void
    {
        /** @var User|null $user */
        $user = $this->userRepository->findOneBy(['id' => $userEntity->id]);

        if ($user === null) {
            $user = (new User())
                ->setEmail($userEntity->name)
                ->setRoles(['ROLE_USER']);
        }

        $this->userRepository->save($user);
    }

    public function findOneByUsername(string $username): ?PublicKeyCredentialUserEntity
    {
        $user = $this->userRepository->findOneBy(['email' => $username]);
        return $this->getUserEntity($user);
    }

    public function findOneByUserHandle(string $userHandle): ?PublicKeyCredentialUserEntity
    {
        $user = $this->userRepository->findOneBy(['id' => $userHandle]);
        return $this->getUserEntity($user);
    }

    public function generateUserEntity(?string $username, ?string $displayName): PublicKeyCredentialUserEntity
    {
        $randomUserData = Base64UrlSafe::encodeUnpadded(random_bytes(32));

        return PublicKeyCredentialUserEntity::create(
            $username ?? $randomUserData,
            $this->generateNextUserEntityId(),
            $displayName ?? $username ?? $randomUserData,
            null
        );
    }

    private function getUserEntity(null|User $user): ?PublicKeyCredentialUserEntity
    {
        if ($user === null) {
            return null;
        }

        return new PublicKeyCredentialUserEntity(
            $user->getUserIdentifier(),
            (string) $user->getId(),
            $user->getDisplayName(),
            null
        );
    }
}
```

Once done, you'll need to configure the **webauthn** bundle properly. This includes specifying custom repositories for credential sources and user entities, as well as defining creation and request profiles for the authentication process.

```yaml
# config/packages/webauthn.yaml
webauthn:
    credential_repository: 'App\Repository\WebauthnCredentialSourceRepository'
    user_repository: 'App\Repository\WebauthnCredentialUserRepository'
    creation_profiles:
        default:
            rp:
                name: '%env(RELYING_PARTY_NAME)%'
                id: '%env(RELYING_PARTY_ID)%'
    request_profiles:
        default:
            rp_id: '%env(RELYING_PARTY_ID)%'
```

To enable the user authentication, you just have to declare the webauthn authenticator in the appropriate firewall (here `main`).

```yaml
# config/packages/security.yaml
security:
    firewalls:
        main:
            # ...
            webauthn:
                registration:
                    enabled: true
                    profile: default
                    routes:
                        options_path: '/passkeys/attestation/options'
                        result_path: '/passkeys/attestation/result'
                authentication:
                    enabled: true
                    profile: default
                    routes:
                        options_path: '/passkeys/assertion/options'
                        result_path: '/passkeys/assertion/result'
```

### Handling “localhost”

In a development environment, you might not have HTTPS enabled, which can lead to challenges when implementing **WebAuthn**. While HTTPS is crucial for secure communication, you can configure your application to treat certain contexts as secure even without it.

You can bypass the scheme verification by defining a list of **Relying Party IDs** that your application considers secure. This approach lets you test and develop your passkey authentication features without the need for a secure connection.

```yaml
parameters:
    # Do not use this in production - for testing purposes only
    webauthn.secured_rp_ids: ['localhost']
```

### Registration and Login with passkeys

Now that everything is set up, you can leverage the **Stimulus Controller** provided to transform your basic login form into a fully functional WebAuthn-compatible.

#### Registration (creation\_profiles, **attestation ceremony**)

* Users can register their authenticators during the initial setup. When they fill out the registration form and opt for passkeys, the Stimulus Controller will handle the WebAuthn registration process.
    
* If users prefer not to register with passkeys right away, they can still create an account using traditional password-based authentication. You can also provide an option for users to add passkeys later in their account settings.
    

```xml
{{ form_start(registrationForm, {
    attr: {
         ...stimulus_controller('@web-auth/webauthn-stimulus', {
                usernameField: registrationForm.email.vars.full_name,
                creationSuccessRedirectUri: path('app_main'),
                creationResultUrl: path('webauthn.controller.security.main.creation.result'),
                creationOptionsUrl: path('webauthn.controller.security.main.creation.options'),
            }).toArray
     }
}) }}

     {{ form_row(registrationForm.email) }}
     {{ form_row(registrationForm.plainPassword, {label: 'Password'}) }}

     <button type="submit">Register</button>
     <button {{ stimulus_action('@web-auth/webauthn-stimulus', 'signup') }}>
         Register with passkey
     </button>
{{ form_end(registrationForm) }}
```

#### Login (request\_profiles, **assertion ceremony**)

* Once registered, users can log in using their passkeys. appropriate challenges will be sent to the user's authenticator.
    
* After successful authentication, users are granted access to the application.
    

```xml

<form method="post" {{ stimulus_controller('@web-auth/webauthn-stimulus',
    {
       useBrowserAutofill: true,
       usernameField: '_username',
       requestSuccessRedirectUri: path('app_main'),
       requestResultUrl: path('webauthn.controller.security.main.request.result'),
       requestOptionsUrl: path('webauthn.controller.security.main.request.options')
    }
) }}>
     
    // input[name=_username]
    // input[name=_password]
    // input[name=_csrf_token, type=hidden]            

     <button type="submit">Connect</button> 
     <button {{ stimulus_action('@web-auth/webauthn-stimulus', 'signin') }}>
        Connect with passkey
     </button>
  </div>
</form>
```

## Live demo

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1726976979571/d2b79f46-29c6-4067-9b89-85e2c7390b9a.jpeg align="center")

* [https://youtube.com/shorts/geEV7Uk-nJU?si=sYClpaUZYWUdvLS7](https://youtube.com/shorts/geEV7Uk-nJU?si=sYClpaUZYWUdvLS7)
    
* [https://github.com/devscast-youtube/symfony-webauthn-passkeys](https://github.com/devscast-youtube/symfony-webauthn-passkeys)
    
* [https://github.com/web-auth/symfony-webauthn-demo](https://github.com/web-auth/symfony-webauthn-demo)
    

## Conclusion

Incorporating passkey authentication via WebAuthn into your Symfony application marks a significant step towards enhancing user security while improving the overall user experience, thanks for reading and happy coding !

## References

* [https://developer.mozilla.org/en-US/docs/Web/API/Web\_Authentication\_API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API)
    
* [https://fidoalliance.org/passkeys/](https://fidoalliance.org/passkeys/)
    
* [https://github.com/web-auth/webauthn-framework](https://github.com/web-auth/webauthn-framework)
    
* [https://web.dev/articles/passkey-form-autofill](https://web.dev/articles/passkey-form-autofill)
    
* [https://web.dev/articles/passkey-registration](https://web.dev/articles/passkey-registration)
    
* [https://developers.google.com/identity/passkeys](https://developers.google.com/identity/passkeys)