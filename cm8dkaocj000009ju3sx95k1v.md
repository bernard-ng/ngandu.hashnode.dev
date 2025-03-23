---
title: "Getting Started with Value Objects in Symfony"
seoTitle: "Getting Started with Value Objects in Symfony"
seoDescription: "Value Objects in Symfony enhance clarity and maintainability by encapsulating business logic and ensuring data integrity in PHP applications"
datePublished: Mon Mar 17 2025 21:13:39 GMT+0000 (Coordinated Universal Time)
cuid: cm8dkaocj000009ju3sx95k1v
slug: getting-started-with-value-objects-in-symfony
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1742245985204/7bcb78ca-a2bc-4ee6-98a9-89587b0fed31.jpeg
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1742245923542/fdd5b069-9a3a-482d-97fb-a839e09683b8.jpeg
tags: symfony, ddd, value-objects

---

In the realm of modern PHP development, where clarity and precision are paramount, relying solely on primitive types like strings and integers to represent crucial business data can introduce a cascade of problems. As applications evolve and become more intricate, these primitive types often fall short in capturing the nuances and rules inherent in domain-specific data. This can lead to subtle bugs, code duplication, and a maintenance nightmare. In this expanded exploration, we'll delve deeper into how Value Objects in Symfony provide a robust solution by encapsulating business logic, ensuring data integrity, and promoting a cleaner, more maintainable codebase.

## The Pitfalls of Primitive Types

Consider a typical `Student` class :

```php
class Student {
    public __construct(
        public readonly int $id,
        private(set) string $email,
        private(set) string $username,
        private(set) string $city,
        private(set) string $country,
        private(set) string $addressLine1
        private(set) string $addressLine2
        private(set) string $birthdate
    ) {
    }
}
```

When you use primitive types to represent complex data, you often lose the context that defines that data. For instance, a simple string might be used to represent an email address, a username, or even a raw piece of text. This ambiguity can lead to:

* **Ambiguity and Lack of Context:** Strings like 'email' or 'birthdate' lack inherent meaning without additional context. This can lead to misinterpretations and errors.
    
* **Validation Scattered and Duplicated:** Ensuring the validity of email addresses, usernames, and other data often results in validation logic scattered throughout the application, leading to duplication and potential inconsistencies.
    
* **Data Integrity at Risk:** Without built-in safeguards, invalid data can easily infiltrate the system, causing unexpected behavior and bugs.
    

By acknowledging these pitfalls, developers are encouraged to adopt better solutions for handling domain-specific data.

## What Are Value Objects?

[Value Objects](https://en.wikipedia.org/wiki/Value_object) focus on the **value** of data rather than its identity. Unlike entities, which are identified by an ID, Value Objects encapsulate a set of data along with rules that govern it.

* **Encapsulation of Business Rules:** Each Value Object is responsible for validating and managing its data. For example, an `Email` Value Object wouldn’t just store an email address—it would verify that the email conforms to expected standards.
    
* **Immutability:** Once created, Value Objects cannot be modified. This ensures data consistency throughout the application.
    
* **Comparison by Value:** Instead of comparing references (like two objects), Value Objects are compared based on the actual data they hold. This makes equality checks straightforward and predictable.
    

Common examples in PHP include classes like `DateTimeImmutable` and `SplFileInfo`, which embody these principles.

## **Crafting Value Objects in Symfony**

Transitioning to Value Objects in Symfony involves a few key steps:

### 1\. Creating the Value Object Class

A well-designed Value Object should adhere to these principles:

* **Validate Input:** Upon creation, a Value Object should rigorously validate the provided data, ensuring it adheres to the domain's rules.
    
* **Encapsulate Behavior:** Any logic related to the Value Object's data, such as formatting or transformation, should be encapsulated within the Value Object itself.
    

#### Email Value Object:

```php
namespace App\Entity\ValueObject;

use Webmozart\Assert\Assert;

final readonly class Email implements \Stringable
{
    public string $email;

    public function __construct(string $value)
    {
        Assert::notEmpty($value);
        Assert::email($value);

        $this->value = $value;
    }

    #[\Override]
    public function __toString(): string
    {
        return $this->email;
    }
}
```

#### Username Value Object:

```php
namespace App\Entity\ValueObject;

use Webmozart\Assert\Assert;

final readonly class Username implements \Stringable
{
    private const int MIN_LENGTH = 3;
    private const int MAX_LENGTH = 30;
    private const string PATTERN = 'some complex regex';

    private string $username;

    private function __construct(string $username)
    {
        Assert::notEmpty($username);
        Assert::minLength($username, self::MIN_LENGTH);
        Assert::maxLength($username, self::MAX_LENGTH);
        Assert::pattern($username, self::PATTERN);

        $this->username = $username;
    }

    #[\Override]
    public function __toString(): string
    {
        return $this->username;
    }
}
```

### **2\. Taming Complexity with Factories**

Validating an Address Value Object can be challenging due to its inherent complexity. For example, how can we ensure that the specified country or city actually exists? In such cases, an Object Factory can be used to instantiate the Value Object. This factory can leverage external services or even query your database to validate the provided data.

#### Address Value Object:

```php
namespace App\Entity\ValueObject;

final readonly class Address
{
    public function __construct(
        public ?string $city = null,
        public ?string $country = null,
        public ?string $addressLine1 = null,
        public ?string $addressLine2 = null
    ) {
    }
}
```

Whenever validation logic becomes too intricate to be handled within the Value Object itself, consider using object factories. They can be injected into your forms or any other part of your application where Value Objects are created. Here’s a simple example:

#### Address Factory:

```php
namespace App\Factory;

use App\Entity\ValueObject\Address;
use Symfony\Component\Intl\Countries;
use Webmozart\Assert\Assert;

final readonly class AddressFactory
{
    public function create(
        ?string $city, 
        ?string $country, 
        ?string $line1, 
        ?string $line2
    ): Address {
        Assert::notEmpty($city, 'City cannot be empty');
        Assert::notEmpty($country, 'Country cannot be empty');
        Assert::notEmpty($addressLine1, 'Address line 1 cannot be empty');
        Assert::nullOrNotEmpty($addressLine2, 'Address line 2 cannot be empty');

        // or any data source like a repository etc...
        if (!Countries::alpha3CodeExists($country) || !Countries::exists($country)) {
            throw new \InvalidArgumentException('Invalid Country');
        }

        return new Address($city, $country, $addressLine1, $addressLine2);
    }
}
```

With this approach, having an instance of a Value Object guarantees that its value is valid, eliminating the need for further checks or validations. This not only simplifies our reasoning but also proves highly beneficial in large-scale applications.

### 3\. **Persisting Value Objects with Doctrine**

Symfony’s integration with Doctrine makes it easy to persist Value Objects to the database. Using Doctrine's [**embeddable**](https://www.doctrine-project.org/projects/doctrine-orm/en/3.3/tutorials/embeddables.html) feature, you can treat a Value Object as part of a larger entity.

#### Email Value Object:

```php
namespace App\Entity\ValueObject;

use Doctrine\ORM\Mapping\Column;
use Doctrine\ORM\Mapping\Embeddable;
use Webmozart\Assert\Assert;

#[Embeddable]
final readonly class Email implements \Stringable
{
    #[Column(type: "string")]
    public string $email;
}
```

#### Username Value Object:

```php
namespace App\Entity\ValueObject;

use Doctrine\ORM\Mapping\Column;
use Doctrine\ORM\Mapping\Embeddable;
use Webmozart\Assert\Assert;

#[Embeddable]
final readonly class Username implements \Stringable
{
    #[Column(type: "string")] 
    public string $username;
}
```

#### Address Value Object:

```php
namespace App\Entity\ValueObject;

use Doctrine\ORM\Mapping\Column;
use Doctrine\ORM\Mapping\Embeddable;

#[Embeddable]
final class Address
{
    public function __construct(
        #[Column(length: 255)] public ?string $city = null,
        #[Column(length: 255)] public ?string $country = null,
        #[Column(length: 255)] public ?string $addressLine1 = null,
        #[Column(length: 255, nullable: true)] public ?string $addressLine2 = null
    ) {
    }
}
```

This is the simplest approach, but you can also define a custom mapping type for more specific use cases.

### 4\. Handling Form Types

Working with forms in Symfony often involves converting user input into custom Value Objects, which requires creating custom form types. This process includes implementing the `DataMapperInterface` to handle the conversion between raw form data and the Value Object, as well as ensuring that the data is properly transformed and validated according to the rules defined in the Value Object class.

#### Email Type:

```php
namespace App\Form\Types;

use App\Entity\ValueObject\Email;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\DataMapperInterface;
use Symfony\Component\Form\Extension\Core\Type\EmailType as SymfonyEmailType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\Form\FormError;
use Symfony\Component\OptionsResolver\OptionsResolver;

final class EmailType extends AbstractType implements DataMapperInterface
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder->add('email', SymfonyEmailType::class, [
            'label' => "email",
            'attr' => [
                'placeholder' => 'exemple bernard@devscast.tech'
            ]
        ])->setDataMapper($this);
    }

    /**
     * @see https://github.com/symfony/symfony/issues/59950
     */
    public function getBlockPrefix(): string
    {
        return '';
    }

    public function configureOptions(OptionsResolver $resolver): OptionsResolver
    {
        parent::configureOptions($resolver);
        $resolver->setDefaults([
            'data_class' => Email::class, /** value object */
            'empty_data' => null
        ]);

        return $resolver;
    }

    public function mapDataToForms(mixed $viewData, \Traversable $forms): void
    {
        $forms = iterator_to_array($forms);
        $forms['email']->setData((string) $viewData);
    }

    public function mapFormsToData(\Traversable $forms, mixed &$viewData): void
    {
        $forms = iterator_to_array($forms);
        try {
            $viewData = new Email($forms['email']->getData());
        } catch (\InvalidArgumentException $e) {
            $forms['email']->addError(new FormError($e->getMessage()));
        }
    }
}
```

Specifically for the EmailType, as it may cause confusion with Symfony's native EmailType, you should override the `getBlockPrefix` method. You can learn more about this here.

**Username Type:**

```php
namespace App\Form\Types;

use App\Entity\ValueObject\Username;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\DataMapperInterface;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\Form\FormError;
use Symfony\Component\OptionsResolver\OptionsResolver;

final class UsernameType extends AbstractType implements DataMapperInterface
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder->add('username', TextType::class, [
            'label' => "nom d'utilisateur",
            'attr' => [
                'placeholder' => 'exemple @_bernard.ng_'
            ]
        ])->setDataMapper($this);
    }

    public function configureOptions(OptionsResolver $resolver): OptionsResolver
    {
        parent::configureOptions($resolver);
        $resolver->setDefaults([
            'data_class' => Username::class,
            'empty_data' => null
        ]);

        return $resolver;
    }

    public function mapDataToForms(mixed $viewData, \Traversable $forms): void
    {
        $forms = iterator_to_array($forms);
        $forms['username']->setData((string) $viewData);
    }

    public function mapFormsToData(\Traversable $forms, mixed &$viewData): void
    {
        $forms = iterator_to_array($forms);
        try {
            $viewData = new Username($forms['username']->getData());
        } catch (\InvalidArgumentException $e) {
            $forms['username']->addError(new FormError($e->getMessage()));
        }
    }
}
```

**Address Type using Address Factory:**

Adding errors to the correct field using this approach can be challenging. If you need more control over this, you can create an `AddressFormFactory` that handles the entire form validation process and adds errors to the appropriate fields. However, to keep things simple, let's stick with the basic `AddressFactory` as defined earlier.

```php
namespace App\Form\Types;

use App\Entity\ValueObject\Address;
use Symfony\Component\Form\AbstractType;
use Symfony\Component\Form\DataMapperInterface;
use Symfony\Component\Form\Extension\Core\Type\CountryType;
use Symfony\Component\Form\Extension\Core\Type\TextType;
use Symfony\Component\Form\FormBuilderInterface;
use Symfony\Component\OptionsResolver\OptionsResolver;

final class AddressType extends AbstractType implements DataMapperInterface
{
     public function __construct(
        private readonly AddressFactory $addressFactory
    ) {
    }

    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder
            ->add('city', TextType::class)
            ->add('country', CountryType::class)
            ->add('addressLine1', TextType::class)
            ->add('addressLine2', TextType::class, [
                'required' => false
            ])
            ->setDataMapper($this);
    }

    public function configureOptions(OptionsResolver $resolver): OptionsResolver
    {
        parent::configureOptions($resolver);
        $resolver->setDefaults([
            'data_class' => Address::class,
            'empty_data' => null
        ]);

        return $resolver;
    }

    public function mapDataToForms(mixed $viewData, \Traversable $forms): void
    {
        $forms = iterator_to_array($forms);
        $forms['city']->setData($viewData?->city);
        $forms['country']->setData($viewData?->country);
        $forms['addressLine1']->setData($viewData?->addressLine1);
        $forms['addressLine2']->setData($viewData?->addressLine2);
    }

    public function mapFormsToData(\Traversable $forms, mixed &$viewData): void
    {
        $forms = iterator_to_array($forms);
        try {
            // encapsulate heavy validation logic
            $viewData = $this->addressFactory->create(
                $forms['city']->getData(),
                $forms['country']->getData(),
                $forms['addressLine1']->getData(),
                $forms['addressLine2']->getData()
            );
        } catch (\InvalidArgumentException $e) {
            // you can create custom exception for each field
            // and map it to the right field
            $forms['city']->addError(new FormError($e->getMessage()));
        }
    }
}
```

Now, handling the form should work seamlessly, with no changes needed in our controller or views. By defining our value objects and creating a form type for each one, our entity can utilize them and eliminate the use of primitive types.

```php
#[ORM\Entity(repositoryClass: StudentRepository::class)]
class Student
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private(set) ?int $id = null;

    public function __construct(
        #[ORM\Embedded(class: Email::class)]
        private(set) Email $email,

        #[ORM\Embedded(class: Username::class)]
        private(set) Username $username,

        #[ORM\Embedded(class: Address::class, columnPrefix: false)] 
        private(set) Address $address,

        #[ORM\Column]
        private(set) \DateTimeImmutable $birthdate,
    ) {
    }
}
```

## Conclusion

Embracing Value Objects in Symfony transforms the way you manage domain-specific data. By encapsulating business rules, ensuring immutability, and enhancing clarity, Value Objects help you avoid the pitfalls associated with primitive types. Whether you’re validating an email address or embedding complex types into your entities, this approach leads to safer, more predictable, and more expressive code.

Happy coding!